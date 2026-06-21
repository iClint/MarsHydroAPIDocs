# Mars Hydro / Mars Pro - Reverse-Engineered API Docs

Community documentation of how the Mars Hydro "Mars Pro" app (package `com.marspro.meizhi`, also
covering the Meizhi sub-brand, vendor LG-LED Solutions) talks to its cloud backend, so you can build
your own dashboards, Home Assistant bridges, or local control.

This is interop research on the author's own account and own hardware, shared as knowledge. It
documents both what is known and, just as importantly, the precise gaps in that knowledge; nothing
here is official or guaranteed.

## The two channels

| Channel | Host | What it carries |
|---|---|---|
| REST | `https://mars-pro.api.lgledsolutions.com` | login, device inventory, history, notifications |
| MQTT (EMQX) | `mars-pro.mqtt.lgledsolutions.com:8883` | live telemetry + device control |
| Static files | `mars-pro.file.lgledsolutions.com` | images, ToS/privacy HTML |

Auth is just email + password (REST returns a bearer token and an account UUID; the MQTT broker
authenticates with username = email, password = that UUID). There is no request signing.

## Documentation

| Doc | What's in it |
|---|---|
| [docs/01_quickstart.md](docs/01_quickstart.md) | Shapes + examples to get running fast: login, MQTT connect, read/write recipes, field & `modeType` tables, gotchas. **Start here to build something.** |
| [docs/02_api-reference.md](docs/02_api-reference.md) | The exhaustive consumer contract: hosts, auth, REST endpoints, MQTT topics & methods, every data shape, and the per-device control/level encodings. |
| [docs/03_device-behaviour.md](docs/03_device-behaviour.md) | Empirical runtime quirks the contract doesn't make obvious: per-device "is it running?" rules, which devices you can detect as unplugged, fault/alarm codes, and the natural-wind flag. |
| [docs/04_known-gaps.md](docs/04_known-gaps.md) | The honest boundary: what is *not* known, stated precisely. Unverified device types, unprobed endpoints, partial code tables, the untouched BLE channel, and the security posture. |
| [docs/05_reverse-engineering.md](docs/05_reverse-engineering.md) | The narrative: how the app was taken apart (Flutter/Dart, Blutter, TLS MITM), the cloud architecture, why the bundled mutual-TLS cert is vestigial, and a security assessment. |

## Scope & honesty

- Verified against a single MH-CB43 controller (firmware 4.4) on one account. The API differs per
  device class; treat other hardware as unverified until captured. See
  [04_known-gaps.md](docs/04_known-gaps.md).
- No credentials, serials, certs, or keys are included in this repo. Example serials / IDs / MACs in
  the docs are placeholders.
- The bundled "mutual-TLS" client cert is not redistributed, and isn't needed; the broker doesn't
  require it.

## Ethics & terms

This is personal, non-commercial interop research on my own account and my own device, published as
documentation. Mars Hydro's app terms prohibit reverse engineering and third-party clients, so:

- This is not a product and not a way to access anything that isn't yours. The broker enforces
  per-account ACLs (verified), so knowing a serial grants no access.
- The vendor can change or break this at any time; it's best-effort, undocumented, and unofficial.
- **Don't hammer the API.** Isolate it behind one adapter, mirror the app's request shape, and handle
  token expiry by re-logging in.

*Build local, keep your data, and verify everything yourself.*
