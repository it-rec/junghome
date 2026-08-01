# AGENTS.md

Guidance for AI coding agents working in this repository.

This is a Home Assistant **custom integration** for the **JUNG HOME** gateway,
distributed via HACS. It talks to a gateway on the local network over its REST
API and WebSocket (`iot_class: local_push`, `quality_scale: platinum`).

There is no vendor SDK: the gateway protocol was reverse-engineered and is
documented in [`docs/`](docs/README.md). **Read the relevant doc before
inferring protocol behaviour from the code** — most surprising-looking code
here encodes a hard-won fact about the gateway.

## Setup and commands

Python 3.14 in CI (`.ruff.toml` targets `py313` syntax). `homeassistant` is
pinned in `requirements.txt` — install the pinned version, tests need it.

```bash
python3 -m pip install -r requirements_test.txt   # runtime + test deps
./scripts/setup                                   # runtime deps only

python3 -m pytest -q                              # tests
python3 -m pytest -q --cov=custom_components/junghome --cov-report=term-missing
python3 -m mypy custom_components/junghome --strict  # must pass; CI gates on it
python3 -m ruff check .                           # lint (CI: no --fix)
python3 -m ruff format . --check                  # format check
./scripts/lint                                    # ruff format + check --fix

./scripts/develop     # run a local HA against ./config with this integration
docker compose up     # local test harness
```

Run a single test file or test with the usual pytest selectors, e.g.
`python3 -m pytest tests/test_cover.py -q` or `... -k tilt`.

CI (`.github/workflows/`): `lint.yml` (ruff check + format), `test.yml` (mypy
`--strict`, then pytest with coverage), `validate.yml` (hassfest + HACS).
Coverage is gated at `fail_under = 95` with `branch = true` (`.coveragerc`) —
new code needs tests covering **both** sides of its guards.

Before pushing, expect to run at minimum: `ruff format .`, `ruff check .`,
`mypy custom_components/junghome --strict`, `pytest`.

## Layout

- `custom_components/junghome/` — the integration.
  - `__init__.py` — entry setup/unload, one-time registry migration to stable
    IDs, area suggestion from gateway groups, and `_register_capability_reload`
    (reloads the entry when a device's datapoint-type set changes).
  - `coordinator.py` — REST polling (60 s fallback) + WebSocket connection,
    commands, and the `junghome_scene_recalled` bus event.
  - `config_flow.py` — zeroconf + manual setup, options flow, reconfigure,
    reauth.
  - `const.py` — `DOMAIN` and the stable-ID / datapoint helpers
    (`device_slug`, `datapoint_suffix`, `stable_unique_id`,
    `is_presence_quantity`, `datapoint_value`, `gateway_device_info`).
  - `models.py` — `TypedDict`s for gateway payloads (`Device`, `Datapoint`).
    The data is still untrusted JSON: keep defensive `.get(...)` at call sites.
  - `entity.py` — shared `CoordinatorEntity` base (`device_info`, `available`,
    lookups) plus `claim_new_entity` for duplicate-safe discovery. Scenes are
    intentionally *not* based on it (no backing device).
  - Platforms: `light.py`, `switch.py`, `sensor.py`, `binary_sensor.py`,
    `event.py`, `cover.py`, `climate.py`, `scene.py`. Each does live discovery
    of devices added at runtime.
  - `diagnostics.py`, `logbook.py`, `quality_scale.yaml`, `icons.json`,
    `strings.json`, `translations/`.
- `tests/` — pytest suite, one file per platform plus `test_init.py`
  (setup/entity lifecycle, the largest), `test_const.py`, `test_config_flow.py`,
  `test_coordinator.py`, `test_websocket.py`, `test_logbook.py`.
- `docs/` — reverse-engineered gateway reference (start at
  [`docs/README.md`](docs/README.md)) plus `example-button-automation.md`.
- `blueprints/automation/junghome/button_gestures.yaml` — shipped HA blueprint
  deriving single/double/hold from raw button edges. Users import it **by URL**;
  HACS only installs `custom_components/`, so it is not distributed by HACS.
- `config/`, `docker-compose.yml`, `scripts/` — local test harness.
- `tools/bt-mesh-direct/` — gateway-free BT-Mesh prototypes (not shipped).
- `disk_dump/` — gateway microSD image, **gitignored**: contains tokens and mesh
  keys. Never commit it, never paste its contents into code, docs, or a PR.

## Invariants — do not break these

- **Stable identity.** The gateway regenerates device/datapoint `id`s on
  firmware updates. Entity `unique_id`s and device identifiers derive from the
  device **label** + datapoint **suffix** via `stable_unique_id`, never the raw
  gateway id. Do not reintroduce id-based identifiers.
- **Entity naming.** Entities set `_attr_has_entity_name = True` and a short
  `_attr_name` (or `None` for a device's main feature, e.g. light/socket). The
  **device** carries the label; never bake the label into the entity name — HA
  would compose it twice (the old `event.<label>_<label>_..._event` bug).
  Naming changes only affect new entities; existing `entity_id`s are sticky.
- **Capabilities follow datapoint *presence*, not the function-type name.**
  Each platform freezes supported features at construction from the datapoints
  present (cover tilt ← `angle`, light brightness/colour ← `brightness` /
  `color_temperature`), and `_discover_*` is add-only, so a live entity never
  re-derives them. The gateway *can* add or drop a datapoint at runtime, which
  is why `_register_capability_reload` reloads the entry on a datapoint-type-set
  change. Keep both halves of that mechanism intact.
- **A `Thermostat`'s `switch` datapoint is not an on/off.** The gateway
  re-labels the RTR's `automatic_mode` as type `switch`; in the field it tracks
  the regulator's momentary heating output and flips several times an hour. A
  room regulator has no on/off, so the climate entity is permanently
  `HVACMode.HEAT` (`hvac_modes = [HEAT]`) and that datapoint feeds **only**
  `hvac_action` (heating/idle). Never map it to `hvac_mode` again — issue #121,
  evidence in [`docs/gateway-websocket.md`](docs/gateway-websocket.md).
- **Cover position is percent-*closed*.** Confirmed against firmware: a *close*
  maps to BT-Mesh "down" (`0x7FFF`, drives `level`→100%) and an *open* to "up"
  (`0x8000`, →0%), so HA position = `100 - level` — correct for roller
  shutters/blinds. **Awnings are mounted inverted**; users flag them in the
  options flow (`CONF_INVERTED_COVERS`), which switches that cover to an
  identity mapping. The single inversion point is `_to_ha` / `_to_device` in
  `cover.py` (both take an `inverted` flag) — keep it that way. Changing the
  inverted set reloads the entry (`async_reload_entry`, gated on an options
  snapshot in the coordinator).
- **Presence vs. numeric split.** A `quantity` datapoint whose label denotes
  presence/occupancy (empty unit, 0/1 value — e.g. a BWM detector's
  `Presence Detected`) becomes an occupancy **binary_sensor**; other `quantity`
  datapoints become numeric **sensors**. `is_presence_quantity` in `const.py` is
  the single split point: `binary_sensor.py` claims those labels and `sensor.py`
  skips them. Don't add a second split rule elsewhere.
- **Buttons emit raw edges.** `event.py` exposes RockerSwitch buttons; the
  gateway reports only `pressed` / `depressed` (no native single/double/hold)
  and alternates a button between its `up_request` and `down_request` events on
  consecutive presses. Gesture derivation belongs in the blueprint, not here.
- **Registration.** Tokens come from `POST /api/junghome/register`
  (`{"user_name": ...}`), which blocks up to 180 s until the user approves the
  request in the JUNG HOME app (Settings → Gateway → Access Permissions → Open
  Requests). Don't shorten that timeout expecting a fast reply.

Function-type → platform map: `OnOff` / `DimmerLight` / `ColorLight` → light;
`Socket` → switch + sensor; `Measurement` → sensor + binary_sensor;
`Position` / `PositionAndAngle` → cover; `Thermostat` → climate;
`RockerSwitch` → event + switch (status LED). Scenes arrive on the WebSocket
`scenes` broadcast and recall over REST (`POST /scenes/{id}`; the WebSocket
`scene` command is unimplemented).

## Conventions

- Follow Home Assistant integration patterns; when unsure, compare against a
  mature core integration (`homeassistant/components/shelly` is the reference
  used for this codebase's review notes).
- Full type annotations — `mypy --strict` must pass on
  `custom_components/junghome`.
- Ruff with `select = ["ALL"]` and a curated ignore list in `.ruff.toml`. Don't
  widen the ignore list to make new code pass; fix the code. Docstrings are
  required in `custom_components/` (relaxed for `tests/` and `tools/`).
- Reuse the shared aiohttp session via
  `async_get_clientsession(hass, verify_ssl=False)` — the gateway's cert is
  self-signed. Never create per-request `ClientSession`s and never build SSL
  contexts on the event loop.
- Keep `strings.json` and `translations/en.json` in sync. No `<...>` in
  translation text — it breaks the translation parser.
- User-visible strings go through translations; don't hardcode English in
  entity or flow code.
- Bump `version` in `custom_components/junghome/manifest.json` for a release;
  `hacs.json` pins the minimum HA version.
- Tests use `pytest_homeassistant_custom_component`'s `hass` fixture,
  `MockConfigEntry` and `aioclient_mock` (`asyncio_mode = auto`, so no
  `@pytest.mark.asyncio`). Add platform tests to the matching
  `tests/test_<platform>.py`.

## Working agreements

- Don't commit `disk_dump/`, gateway tokens, mesh keys, or real gateway
  hostnames/IPs. Scrub captures before adding them to `docs/` or tests.
- Protocol findings belong in `docs/`, not only in code comments — the next
  agent reads `docs/` first.
- `CLAUDE.md` holds additional context plus a review backlog of ideas
  (**not** commitments — evaluate cost/value before implementing any of them).
  Parts of it lag the code; trust the code and this file where they disagree.
- Commit messages in this repo are short, imperative, and subject-only (e.g.
  "Thermostat: read the RTR switch datapoint as hvac_action, not hvac_mode").
- Open a pull request only when explicitly asked.
