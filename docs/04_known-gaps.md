# Mars Hydro — Known Gaps & Unknowns

What we **don't** know, stated precisely. The contract in [`02_api-reference.md`](02_api-reference.md)
is empirical: it was derived from one device and one account. This page is the honest boundary of
that work — read it before you rely on anything as "documented."

Legend: 🔴 unmapped (no data) · 🟡 partial / inferred · ⚪ deliberately not probed · 🔵 known
limitation (verified, can't be fixed client‑side).

---

## Coverage of the work

| | |
|---|---|
| **Devices verified** | A single **MH‑CB43** controller, firmware **3.7**, `deviceProductGroup 7`, with env + soil sensors, oscillating fan, inline (EC) fan, humidifier, dehumidifier. |
| **Account** | One personal account. |
| **Channels** | REST (login/inventory/history) + MQTT (live telemetry/control) fully exercised. BLE and WiFi‑provisioning channels **not touched at all**. |
| **App version** | Mars Pro `com.marspro.meizhi` v2.1.0 (Flutter/Dart). |

Because the API is known to differ **per device class** (field names, modes, and "off" encodings
vary by actuator), nothing here should be assumed to transfer to other hardware without a capture.

---

## 1. Device & hardware coverage 🔴🟡

- 🔴 **Other product types unverified.** Only MH‑CB43 was tested. Grow‑light fixtures (`MH-*`
  lights), other controllers, and Meizhi‑branded units are unmapped. Their actuator schemas, modes
  and sensor sets are unknown.
- 🟡 **`deviceProductGroup` mapping.** CB43 = group 7. The full group‑number → device‑class table
  is unknown; the working heuristic is to iterate groups `0..9` to enumerate devices.
- 🟡 **`light2` (second light channel).** Only ever observed as `{ level: 0 }`. Its full schema and
  behaviour when actually driven are unobserved.
- 🟡 **CO₂ and PPFD with real hardware.** The config and history carry `co2`/`ppfd` fields, but the
  test CB43 had no CO₂ sensor (field absent) and `ppfd` in history was always `"0"`. Behaviour with
  a real CO₂ sensor or a metered light fixture is unverified, including `blower.closeCO2` (documented
  but only meaningful under CO₂ control).

## 2. REST endpoints 🟡⚪

- 🟡 **`dr/getDevicePowerData/v1` (energy history).** Valid (`code 000`) but returns `sumPower: 0.0`
  + empty `data[]` for the CB43 across every type/range. We *believe* metered light fixtures
  populate it — **unverified** (no metered fixture to test). The shape of a populated `data[]` row
  is unknown.
- 🟡 **`dict/getTimeZone/v1`.** Returns `code 100 "param error"` for every body tried; the required
  parameter is not in any capture. (Low value — timezone is already available from `getSysSta` and
  `mine/info`.)
- 🟡 **`data/dtd/v11`.** Returns `code 100 "fail"` — the route may exist but rejects our params. Its
  purpose and request shape are unknown.
- 🟡 **Per‑endpoint versioning.** Only `/v1` resolves on the device endpoints (`v2…v20` →
  `105 "no.method"`); the `/v18` seen in the binary is believed a Dart string‑pool artifact. Whether
  *any* endpoint exposes a real `v2+` is unconfirmed.
- ⚪ **Mutating / account endpoints — deliberately NOT probed** (request/response shapes unverified):
  `ulogin/mailRegister`, `ulogin/getEmailCode`, `ulogin/forgotPassword`, `ulogin/updatePassword`,
  `tlogin/unionLogin` (Google), `mine/editNickName`, `mine/editTHUnits`, `udm/addDevice`,
  `udm/deleteDevice`.
- 🔴 **`/device/*` REST controller group** (`setting`, `timer`, `auto`, `cycle`, `sensorSetting`,
  `alarmSetting`, `outletSetting`, `wifiNearby`, `deviceUpgrade`) — seen in the string dump, never
  probed. Relationship to the MQTT `getConfigFile`/`setConfigField` config is unknown (possibly a
  REST mirror of the same settings).
- 🔴 **Other modules seen in strings, unmapped:** `usm`/`ums` (scene/stage templates), `community`
  (social feed), `product`, `dua` (device upgrade/OTA), `export` (history‑to‑email).

## 3. Config & control semantics 🟡

- 🟡 **`modeType` enumeration is incomplete and per‑actuator.** Known values: light `12`
  (timer/PPFD); fan `1`/`2`/`7`/`8`/`13`; blower `1`/`2`/`7`/`8`/`13`; sockets `4` (environmental).
  Manual is encoded by **omitting** `modeType`. The full set per device and the exact meaning of
  some values are inferred, not exhaustively confirmed.
- 🟡 **"Drop‑field" / presence encodings** (Manual = no `modeType`, fan 0° = no `shakeLevel`,
  standby‑off = no `minSpeed`, humidifier Auto = no `level`, dehumidifier Low = no `mLevel`) are
  verified for the tested devices only. The known config‑corruption hazard — writing
  `dehumidifier.level: 2` wedges the slider in *both* apps — means untested writes are genuinely
  risky. See [`02_api-reference.md` §7a](02_api-reference.md).
- 🟡 **Socket level ranges.** Humidifier = `level` 1–4 (verified); dehumidifier = Low/High only,
  encoded by presence of `mLevel` (verified). Whether other socket‑class devices have wider ranges
  is unknown.
- 🟡 **`getConfigFile` completeness.** Five sections captured (`device`, `target`, `alarm`,
  `calibration`, `system`) for one device in a few modes. Coverage across all modes/devices is
  unverified.

## 4. Alarms, fault & device‑type codes 🟡🔴

- 🟡 **`alarmType` table is partial.** Known: `3` = disconnected (inline fan), `5` = dehumidifier
  bucket full. Sensor threshold‑breach types and the humidifier "tank empty" server‑side type are
  unmapped.
- 🟡 **Humidifier empty (`alarm 4` in `getDevSta`)** is confirmed live, but its corresponding
  `notice_msg/queryMessageList` `alarmType` has **not** been captured.
- 🔴 **`devType` table is partial.** Known: `26` = CB43 controller, `19` = inline fan. The rest is
  unmapped.
- 🟡 **Socket unplug behaviour** is inferred to be symmetric per socket (key drops, no alarm) but was
  only observed with both sockets unplugged together.

## 5. Auth, sessions & lifetime 🟡

- 🟡 **Token TTL unknown.** No documented expiry; handled defensively by re‑login on a non‑`000`
  envelope. How long a token actually lives is unmeasured.
- 🟡 **Firebase relationship.** The app embeds Firebase Auth (project `mars-pro-930a4`) and Google
  OAuth, but the primary email/password path goes through the vendor's own `ulogin` REST. How (or
  whether) Firebase issues the REST token vs. it being minted server‑side is not fully traced.
- 🟡 **Login signature.** Confirmed there is **no HMAC/request signing** — the `systemdata` bearer
  token is the whole gate. The earlier-suspected "signature" was a dead end. (Stated as a finding,
  not a gap, but: the vendor could *add* signing at any time and break new sign‑ins.)

## 6. Provisioning channels 🔴

- 🔴 **BLE — completely unmapped.** The app ships `flutter_reactive_ble` + a `bledata.proto` for
  local device provisioning/control. Not reverse‑engineered at all.
- 🔴 **WiFi provisioning / `wifiNearby` / `deviceUpgrade` (OTA)** — permissions and string
  references exist; the flows are unmapped.

## 7. Misc 🔵🟡

- 🔵 **Natural‑wind fan is invisible to telemetry** (verified). The flag works physically but the
  swing is never reported over MQTT — no client can surface it. Not fixable; see
  [`03_device-behaviour.md` §4](03_device-behaviour.md).
- 🔵 **Dumb‑actuator presence is undetectable** (light + oscillating fan are command‑echo). Verified
  limitation, not a gap to close.
- 🟡 **History retention** is "roughly the last 2 weeks" at 5‑minute resolution — the exact retention
  window is unconfirmed.

---

## Security posture (verified, for completeness)

These are **findings**, not gaps — included so consumers don't have to re‑derive them. Full
write‑up in [`05_reverse-engineering.md`](05_reverse-engineering.md).

- ✅ **Per‑account broker ACLs are enforced.** Subscribing to a serial not on your account → SUBACK
  denial. Knowing someone else's serial grants no access — no cross‑tenant / IDOR issue.
- ⚠️ **MQTT server‑cert verification is disabled** in the client (self‑signed broker cert, accepted
  without verification). Passive eavesdropping is blocked, but an **active MITM** on a hostile
  network can intercept the plaintext‑JSON payload (incl. email + account UUID). Fine on a trusted
  network; a real weakness on untrusted ones.
- ⚠️ **The account UUID is a static, non‑rotating MQTT password.** Poor hygiene, but blast radius is
  bounded to your own devices by the ACLs above.
- ✅ **REST is properly secured** — CA‑issued cert, verified end‑to‑end.

---

## How to close a gap

The project method is **capture, don't assume**: run a read‑only logger that dumps `getConfigFile`
(per actuator, on change) and `getDevSta` (+ the raw UP topic), flip the setting in the official app
with a *distinctive* value (set a level to `3`, not `1`, so the moved field is unambiguous), and
watch which field actually changes. A plausible assumption carried over from another device
produces a control that looks right but does nothing — or corrupts the stored config. If something
*must* be assumed, it belongs on this page as an open item, not buried in the contract.
