# Mars Hydro / Mars Pro - API Reference

How to consume the Mars Hydro backend (app package `com.marspro.meizhi`, vendor LG-LED
Solutions). This is a consumer contract: hosts, auth, endpoints, message formats, and data
shapes.

> Verified against a single live MH-CB43 controller (firmware 4.4) on the author's own
> account. Everything here is empirical. For what is *not* yet verified, see
> [`04_known-gaps.md`](04_known-gaps.md); for observed runtime quirks (presence, alarms, liveness), see
> [`03_device-behaviour.md`](03_device-behaviour.md).

Two channels:
- REST - login, device inventory, historical data.
- MQTT (cloud EMQX broker) - live telemetry and device control.

---

## 1. Hosts

| Purpose | Endpoint |
|---|---|
| REST API | `https://mars-pro.api.lgledsolutions.com` |
| MQTT broker | `mars-pro.mqtt.lgledsolutions.com:8883` (TLS + mutual-TLS) |
| Static files | `http://mars-pro.file.lgledsolutions.com` (images, ToS) |

---

## 2. Authentication (REST)

Every REST request carries a single JSON header, `systemdata`. Requests without a valid one are
rejected by the gateway with `{"code":"100","msg":"Request illegal."}`.

```
systemdata: {
  "reqId":      <int, per-request, ~10-digit>,
  "appVersion": "2.1.0",
  "osType":     "android",
  "osVersion":  "14",
  "deviceType": "sdk_gphone64_arm64",
  "deviceId":   "<any build-id string>",
  "netType":    "wifi",
  "wifiName":   "unknown",
  "timestamp":  <epoch ms>,
  "token":      "<login token>",     // omit on the login call itself
  "timezone":   "<tz, e.g. 34>",     // present once logged in
  "language":   "English"
}
```

- No HMAC or request signing. Auth is the opaque bearer `token` carried inside `systemdata`.
- Other headers: `content-type: application/json`, `user-agent: Dart/3.5 (dart:io)`,
  `accept-encoding: gzip`.
- Path prefix is `/api/<client>/...` where `<client>` (`android`, `ios`, `h5`, …) is a free-form
  label the gateway does not validate; use `android`.

### Login - `POST /api/android/ulogin/mailLogin/v1`
Request body (no `token` in `systemdata` yet):
```json
{ "email": "<email>", "password": "<password>", "loginMethod": "1" }
```
Response `data` (the token + MQTT creds you need for everything else):
```json
{
  "token":      "<REST bearer>",      // -> systemdata.token on all later REST calls
  "userId":     10000,
  "nickName":   "Mars Pro",
  "userTimeZone":"34",
  "mqttName":   "<account email>",    // -> MQTT username
  "mqttPwd":    "<account uuid>",      // -> MQTT password
  "userHeadImg":"https://mars-pro.file.lgledsolutions.com/image/...",
  "timeFormat": 0
}
```

---

## 3. Response envelope (REST)

```json
{ "code": "000", "subCode": null, "msg": "success", "data": <object|array|null> }
```
| `code` | Meaning |
|---|---|
| `"000"` | success |
| `"100"` | gateway reject ("Request illegal") or operation failed ("fail") |

---

## 4. REST endpoints

All `POST`, JSON body, `systemdata` header.

### `udm/getDeviceList/v1` - list devices
Request: `{ "currentPage": 1, "type": null, "deviceProductGroup": <int> }`
Note: devices are bucketed by `deviceProductGroup` (the test controller is group 7). Iterate groups
`0..9` to find all devices.
Response `data`: `{ "list": [ <device>, ... ], "hasNext": false }` where `<device>`:
```json
{
  "id": 100001, "userId": 10000, "productType": "MH-CB43",
  "deviceSerialnum": "AABBCCDDEEFF", "deviceName": "SmokeyCoder's tent",
  "connectStatus": 1, "status": true, "deviceProductGroup": 7,
  "deviceInfo": "<JSON string: sys/wifi/mqtt/bluetooth status>"
}
```

### `udm/getDeviceDetail/v1` - device detail
Request: `{ "deviceId": <id> }`
Response `data`: `id, deviceName, deviceSerialnum, deviceVersion, productType, connectStatus,
isClose, isStart, fanModel, temperature, humidity, …`
`temperature`/`humidity` here are a cache and are usually `null`; get live values from MQTT.
Also present (observed 2026-06-14): `deviceVersion` (firmware, e.g. `"4.4"`),
`isNeedUp` (bool - firmware update available), `deviceBluetooth` (= the serial),
`deviceLightRate`, `groupId`/`groupName` (device grouping, null when ungrouped),
`isNetDevice`, `deviceImg`.

> The lighter `udm/getDeviceList/v1` row additionally embeds the entire `getSysSta` `sys` block
> (see §6) as a JSON string in `deviceInfo`, so firmware/wifi/uptime are reachable over REST without
> an MQTT round-trip.

### `dr/getDeviceTHPData/v1` - historical sensor data
Request:
```json
{ "type": "day", "deviceSerialnum": "<serial>",
  "startDate": "YYYY-MM-DD HH:MM:SS", "endDate": "YYYY-MM-DD HH:MM:SS" }
```
- `type` ∈ `"day" | "week" | "month" | "year"` (`"day"` = full 5-minute resolution).
- Backend retains roughly the last 2 weeks. For long history, request per day and dedupe by row
  `id`. (This is the data the broken "email history" feature wraps.)

Response `data`: array of rows:
```json
{
  "id": 914799, "deviceSerialnum": "AABBCCDDEEFF", "addTime": "06-06-2026 14:00:01",
  "temp": "20.2", "tempF": "68.36", "humi": "58.8", "vpd": "0.98", "co2": null, "ppfd": "0",
  "tempSoil": "19.6", "tempSoilF": "67.28", "humiSoil": "65.3", "ecSoil": "0.31",
  "tempUnit": "0", "tempUnitSoil": "0", "power": null
}
```
(Values are strings; `addTime` is `MM-dd-yyyy HH:mm:ss`.)

This endpoint is **average / device-wide only**. The soil columns (`tempSoil`/`humiSoil`/`ecSoil`)
are the average across all probes; a `sensorId` here is **silently ignored**. For a single probe's
history use `getDeviceTHPData_Industry` below.

### `dr/getDeviceTHPData_Industry/v1` - per-probe (multi-soil) history  *(captured 2026-06-22)*
The history call the app fires when you select a specific soil probe (rather than the average).
Request:
```json
{ "type": "day", "deviceSerialnum": "<serial>",
  "startDate": "YYYY-MM-DD HH:MM:SS", "endDate": "YYYY-MM-DD HH:MM:SS",
  "sensorId": "<probe-hwid>" }
```
- `sensorId` is **required** and is the probe's hardware id - the same value as `senConfig.id` and
  `getDevSta.sensors[].id`.

Response `data`: **soil-only** rows (every air metric is `null`):
```json
{
  "id": 205292, "deviceSerialnum": "AABBCCDDEEFF", "roomId": 10000,
  "sensorId": "363035390F303F42", "sensorType": 0, "tempUnit": "0",
  "temp": null, "humi": null, "vpd": null, "tempF": null, "co2": null, "ppfd": null,
  "tempSoil": "21.1", "tempSoilF": "70.0", "humiSoil": "59.4", "ecSoil": "0.41",
  "tempUnitSoil": "0", "addTime": "06-21-2026 14:00:00"
}
```
(Values are strings; `ecSoil` is lowercase; same UTC window/`addTime` rules as `getDeviceTHPData`.
The `Industry` suffix is just the vendor's name for the multi-sensor variant, not a product tier.)

### `mine/info/v1` - profile  *(probed 2026-06-13)*
Request: empty body (`{}`). Response `data`: `userName, uuid, nickName, userHeadImg, timezone`.
`timezone` here is a numeric id (e.g. `"34"`), not an IANA string; the IANA name comes from
`getSysSta` (`sys.timezone`).

### `dr/getDevicePowerData/v1` - power / energy history  *(probed 2026-06-13)*
Request (same shape as `getDeviceTHPData`):
```json
{ "type": "day", "deviceSerialnum": "<serial>",
  "startDate": "YYYY-MM-DD HH:MM:SS", "endDate": "YYYY-MM-DD HH:MM:SS" }
```
Response `data`: `{ "sumPower": <float>, "data": [ <per-interval rows> ] }`.
Note: a CB43 sensor/socket controller returns `sumPower: 0.0` and an empty `data` array for every
`type`/range tried (day/week/month/hour, by serial or deviceId); energy metering only populates for
metered grow-light fixtures. The endpoint is valid (`code 000`) regardless.

### `notice_msg/queryMessageList/v1` - notifications / alerts feed  *(probed 2026-06-13; alarms captured 2026-06-14)*
Request: `{ "currentPage": 1 }` (paginated). Response `data` is an array of message objects
(empty `[]` when the account has no notifications). This is the server-side alarm log: a threshold
breach in the device config's `alarm` bands (see §6) is persisted here.

Each record (observed for the dehumidifier water-bucket alarm):
```json
{ "id": 2650004, "msgType": 1, "deviceSerialnum": "AABBCCDDEEFF",
  "alarmLogId": 152, "alarmTime": 1781394821, "alarmType": 5, "devType": 26,
  "userId": 10000, "seen": 0, "idDelete": 0,
  "addTime": "06-13-2026 23:54:06", "updateTime": "06-13-2026 23:53:52",
  "deviceImg": "https://…", "deviceName": null, "alarmDescribe": null }
```
- `alarmType` - the alarm reason. `5` = dehumidifier water bucket full (matches the `alarm:5` flag
  on the live `getDevSta` actuator entry, §6).
- `alarmTime`/`addTime` - epoch s / display time of the event; `alarmLogId` increments per event.
- `seen` - `0` = unread; `devType 26` = the CB43 controller class.
- The live `getDevSta` payload also carries `alarmLast: { id, epoch, devType }` pointing at the
  most recent alarm.

### `dict/getTimeZone/v1` - timezone dictionary  *(probed 2026-06-13: needs an unknown param)*
Returns `code 100 "param error"` for an empty body and for `{type}`/`{currentPage}`/`{language}`/
`{countryCode}`; the required param isn't in our captures. Low value anyway: the timezone is
already available from `getSysSta` (`sys.timezone`, IANA) and `mine/info` (numeric id).

### Other (auth/account) - documented, not probed (mutating/destructive, deliberately left alone)
`ulogin/mailRegister/v1`, `ulogin/getEmailCode/v1`, `ulogin/forgotPassword/v1`,
`ulogin/updatePassword/v1`, `tlogin/unionLogin/v1` (Google), `mine/editNickName/v1`,
`mine/editTHUnits/v1`, `udm/addDevice/v1`, `udm/deleteDevice/v1`.

---

## 5. MQTT (live telemetry + control)

### Connection
- Broker `mars-pro.mqtt.lgledsolutions.com:8883`, TLS. The app ships a client cert + key
  (`ca-marspro.pem`, `emqx-marspro.pem`, `emqx-marspro.key`) for nominal mutual-TLS, but the broker
  does not actually require it (see [`05_reverse-engineering.md`](05_reverse-engineering.md), "the bundled
  certificate is a red herring"). The cert/key are not redistributed in this repo; extract them from
  the app if you want to mirror its exact handshake, or just connect without one.
- Username = `mqttName` (account email), password = `mqttPwd` (account UUID); both from the REST
  login response.
- Use a unique client id.

### Topics - `MHPRO/<model>/API/<dir>/<serial>`
- `<model>` = `productType` without the `MH-` prefix (e.g. `CB43` for `MH-CB43`).
- `<dir>` = `DOWN` (app to device, requests/commands) or `UP` (device to app, replies/telemetry).
- e.g. publish to `MHPRO/CB43/API/DOWN/AABBCCDDEEFF`, subscribe to `MHPRO/CB43/API/UP/AABBCCDDEEFF`.
- The broker ACL scopes you to your own device's UP/DOWN topics (wildcards denied).

### Message envelope
Request (publish to DOWN):
```json
{ "method": "<method>", "params": { "pid": "<serial>", ... } }
```
Reply / telemetry (received on UP):
```json
{ "method": "<method>", "pid": "<serial>", "uid": "<userId>", "UTC": <epoch s>,
  "code": 200, "msg": "ok", "data": { ... } }
```
The controller also pushes `getDevSta`/`getSysSta` every few seconds while a session is active.

### Methods
| Method | Purpose |
|---|---|
| `getDevSta` | live sensors + actuator states (see §6) |
| `getSysSta` | firmware/uptime/wifi/mqtt/bluetooth status |
| `getConfigFile` | full device configuration (modes, levels, schedules) |
| `setConfigField` | write one actuator's config (see §7) |

---

## 6. Data shapes

### Live sensors - `getDevSta` -> `data.sensor`
```json
{ "temp": 27.6, "humi": 58.4, "vpd": 1.54, "ppfd": 771,
  "tempSoil": 23.3, "humiSoil": 62.3, "ECSoil": 0.31,
  "isDayEnvTarget": 1, "isDaySensor": 1 }
```
| Field | Meaning | Unit |
|---|---|---|
| `temp` / `humi` | air temperature / humidity | °C / % |
| `vpd` | vapour-pressure deficit | kPa |
| `ppfd` | light intensity | µmol/m²/s |
| `tempSoil` / `humiSoil` / `ECSoil` | substrate temp / moisture / EC | °C / % / mS/cm |

(Per-probe detail in `data.sensors[]`: `{ id, type, tempSoil, humiSoil, ECSoil, typeCode }`;
`id:"avg"` is the average. Note: live uses `ECSoil`, history uses `ecSoil`.)

### Actuator state - `getDevSta` -> `data.<actuator>`
```json
"light":        { "modeType": 12, "on": 1, "level": 78 },
"light2":       { "level": 0 },
"fan":          { "on": 1, "level": 7 },
"blower":       { "modeType": 7, "on": 1, "level": 50 },
"humidifier":   { "modeType": 4, "level": 0 },
"dehumidifier": { "modeType": 4, "on": 1, "level": 1 }
```
Liveness is asymmetric (verified by capture; encoded in the MCP `spec.decode_state`): the humidifier
omits `on` and reports live output in `level` (0 when idle), so `level>0` = running. The dehumidifier
sends an explicit `on` and keeps `level` as a fixed setpoint (can report `on:1,level:0`), so **only
`on==1`** = running; `level` is not liveness. The dehumidifier also raises `alarm:5` when its water
bucket is full.

### System status - `getSysSta` -> `data.sys`  *(captured 2026-06-13)*
```json
{ "ver": "3.7", "hwcode": 1, "buildTime": "Aug 18 2025 16:49:30",
  "localtime": "2026-06-13 18:53:58 Saturday AEST", "timezone": "Australia/Sydney",
  "TZ": "AEST-10AEDT,M10.1.0,M4.1.0/3", "gmtoff": 0, "tzoff": 36000,
  "upCount": 4, "upTime": 299338, "mem": 1319, "UTC": 1781340838,
  "verUpdateNum": 13, "verUpdateTime": 1780901452, "verUpdateWho": "local",
  "wifi": { "isConnect": 1, "rssi": -43, "channel": 1, "ssid"/"bssid": "…",
            "mac": "aa:bb:cc:dd:ee:ff", "ip": "192.168.4.215", "gw": "…", "mask": "…" },
  "eth": { "isConnect": 0 }, "mqtt": { "isConnect": 1, "connectTime": 299330 },
  "bluetooth": { "isConnect": 0 } }
```
`ver` = firmware; `upTime` ms since boot, `upCount` boots; `mem` free KB; connectivity blocks for
wifi/eth/mqtt/bluetooth. Surfaced by the MCP `get_sys_status` tool. (Capture predates the 4.4 update;
`ver` reads the firmware live.)

### Device config - `getConfigFile` -> `data.configFile.device`
```json
"light":        { "modeType":12, "mLevel":34, "darkTemp":30, "offTemp":32,
                  "ppfdMinBrightness":10, "ppfdMaxBrightness":100,
                  "timePeriod":[...], "ppfdPeriod":[{ "enabled":1,"startTime":21600,
                  "endTime":64800,"weekmask":127,"brightness":750,"fadeTime":1800 }] },
"fan":          { "maxSpeed":6, "shakeLevel":10, "natural":1, "mOnOff":1, "mLevel":7,
                  "timePeriod":[...] },
"blower":       { "modeType":7, "minSpeed":25, "maxSpeed":50, "mOnOff":1, "mLevel":25 },
"humidifier":   { "modeType":4, "mOnOff":1, "mLevel":1 },
"dehumidifier": { "modeType":4, "level":1, "mOnOff":1, "mLevel":1 },
"senConfig":    [ { "id":"363035390F303F42", "type":2, "soilType":1, "calibration":{}, "label":"P1" } ]
```
`modeType` (per-device; **Manual = field omitted, never 0**). Fans/sockets: `1`=Timer, `2`=Cycle;
the inline fan's Environmental has five sub-modes - `3` temp-only · `4` humidity-only · `7`
temp-priority · `8` humidity-priority · `13` both (sockets use `4` for Environmental). Lights:
`1`=Timer·Dimming, `12`=Timer·PPFD. `mLevel` = manual level. `mOnOff` = the manual power switch:
present(=1) only in Manual+on, dropped in auto modes and when off, except the oscillating `fan`,
which keeps it in every mode. `senConfig` = soil-probe registry (`type 2` = soil, `label` = user's
name); soil-only (a 2nd ambient temp/humi sensor adds nothing). Times are seconds-since-midnight;
`weekmask` is a 7-bit day mask (`127` = every day). *(Confirmed 2026-06-18 across every device & mode.)*

`configFile` has five sibling sections - `device` (above), plus `target`, `alarm`, `calibration`,
`system` (all captured 2026-06-14):

#### `configFile.target` - day/night climate setpoints (what env mode regulates to)
```json
{ "dayTime": { "startTime": 21600, "endTime": 68400 },
  "temp": { "targetDay": 26, "targetNight": 22, "deadband": 2 },
  "humi": { "targetDay": 60, "targetNight": 65, "deadband": 3 },
  "co2":  { "targetDay": 300, "targetNight": null, "deadband": 10 } }
```
- `dayTime.start/endTime` = the day window (seconds-since-midnight; here 06:00–19:00); outside it
  the `*Night` targets apply.
- `deadband` = the ± hysteresis band around each target before the actuator reacts.

#### `configFile.alarm` - per-sensor threshold bands (breach raises a `notice_msg` alarm, §4)
```json
{ "temp": {"vmin":17,"vmax":29}, "humi": {"vmin":45,"vmax":80},
  "co2": {"vmin":350,"vmax":1600}, "tempSoil": {"vmin":18,"vmax":26},
  "humiSoil": {"vmin":30,"vmax":50}, "ECSoil": {"vmin":0.3,"vmax":1.5},
  "vpd": {"vmin":0.3,"vmax":1.5}, "ppfd": {"vmax":4000} }
```
A reading outside `[vmin, vmax]` raises an alarm logged via `notice_msg/queryMessageList`. (Sensor
alarms are distinct from the hardware `alarmType 5` bucket alarm, which is device-raised.)

#### `configFile.calibration` - sensor offsets applied to raw readings
```json
{ "humi": -1.5 }
```
Per-sensor additive offset (here humidity is trimmed −1.5%). Only calibrated sensors appear.

#### `configFile.system` - device UI / housekeeping
```json
{ "scroff": 60, "alarmTone": 1, "hour": 24,
  "verUpdateNum": 13, "verUpdateTime": 1780901452, "verUpdateWho": "local" }
```
`scroff` = screen-off seconds; `alarmTone` = audible alarm on/off; `hour` = 12/24h clock;
`verUpdate*` = firmware revision bookkeeping (mirrors `getSysSta.sys`).

---

## 7. Control (write)

One method controls every actuator. Read-modify-write: fetch the current actuator config, change a
field, send the whole object back.

```json
// publish to DOWN
{ "method": "setConfigField",
  "params": { "pid": "<serial>", "keyPath": ["device", "<actuator>"],
              "<actuator>": { ...full actuator config with your change... } } }
```
- `<actuator>` ∈ `light, light2, fan, blower, humidifier, dehumidifier`.
- Common edits: `fan.mLevel` (1–10), `light.ppfdPeriod[0].brightness` (PPFD setpoint),
  `light.mLevel` (manual %), `<actuator>.mOnOff` (0/1).
- Mode nuance: env-mode actuators (blower/humidifier/dehumidifier) keep auto-regulating to their
  target until `modeType` is switched to manual; setting `mLevel` alone may be overridden.
- Safety: always read and back up the config before writing; enforce sane bounds; verify with a
  follow-up `getDevSta`. The reference client auto-backs-up and bounds-checks.

---

## 7a. Per-device fields, ranges & quirks

Source of truth for a full API user doc. Verified against a live MH-CB43 + the official app
(2026-06-11). The biggest risk: writing a field out of range, or the *wrong field*, silently
corrupts the stored config and can **break the slider in both the official and unofficial apps**
(observed: writing `dehumidifier.level:2`). Stay within the ranges below.

### Cross-cutting rules
- **Read-modify-write.** `setConfigField` replaces the whole actuator object - fetch current, change
  one field, send it all back. Unknown fields must be preserved.
- **Drop-field idiom.** A field's *absence* encodes its off/floor/default state, e.g. Manual = no
  `modeType`; fan oscillation 0° = no `shakeLevel`; standby off = no `minSpeed`; humidifier Auto =
  no `level`; dehumidifier Low = no `mLevel`. Don't write a "zero" value where the device expects
  the field dropped.
- **Env-mode override.** While `modeType` is an environmental value, the device auto-regulates to
  the tent target; writing `mLevel`/`level` alone may be ignored until you switch `modeType` to
  Manual (drop it).
- **`modeType` is per-device** (see each table). Manual = field omitted.
- **Times** = seconds-since-midnight; **`weekmask`** = 7-bit day mask (127 = every day).
- **devSta ≠ config.** Some devices report live OUTPUT in devSta `level` while config holds a
  separate SETPOINT; they are not interchangeable (see humidifier/dehumidifier).

### light / light2
| field | range / values | meaning |
|---|---|---|
| `mOnOff` | 0–1 (omit = off) | manual power |
| `mLevel` | 10–100 | manual brightness % (min 10) |
| `modeType` | omit=manual, `1`=timer·dimming, `12`=timer·PPFD | operating mode |
| `darkTemp` / `offTemp` | °C (omit = disabled) | heat-protection dim / cutoff thresholds |
| `ppfdPeriod[]` | `brightness` µmol/m²/s | PPFD schedule; `ppfdMin/MaxBrightness` 10/100 |
| `timePeriod[]` | - | timer window (enabled, startTime, endTime, weekmask, brightness, fadeTime) |
- devSta: `{ modeType, on, level }`; `level` = live brightness %.

### fan (oscillating)
| field | range / values | meaning |
|---|---|---|
| `mLevel` | 1–10 | manual speed |
| `maxSpeed` | 1–10 | run speed in Timer/Cycle/Env |
| `minSpeed` | 1–10, omit = no standby | standby/idle speed |
| `shakeLevel` | omit=0°, 5=45°, 10=90° | oscillation angle |
| `natural` | 0–1 (omit = off) | natural-wind mode |
| `mOnOff` | 0–1 (**master switch, kept in every mode**, unlike other devices) | manual power |
| `modeType` | omit=manual, `1`=timer, `2`=cycle, env: `3`temp-only/`4`humidity-only/`7`temp-priority/`8`humidity-priority/`13`both | mode/priority |
- devSta: `{ on, level }`; `level` = live speed 1–10. Natural-wind blanks `on`/`level` while on.

### blower (inline / exhaust fan)
| field | range / values | meaning |
|---|---|---|
| `mOnOff` | 0–1 | manual power |
| `mLevel` | 25–100 | manual speed % (min 25) |
| `maxSpeed` | 25–100 | run speed in Timer/Cycle/Env |
| `minSpeed` | 25–100, omit = no standby | standby/idle speed (kept below run) |
| `closeCO2` | 0–1 (omit = off) | ease exhaust to retain CO₂ (only with CO₂ control) |
| `modeType` | omit=manual, `1`=timer, `2`=cycle, env: `3`temp-only/`4`humidity-only/`7`temp-priority/`8`humidity-priority/`13`both | mode/priority |
- devSta: `{ modeType, on, level }`; `level` = live speed %. `mOnOff` dropped in auto modes.

### humidifier (level lives in `level`, not `mLevel`)
| field | range / values | meaning |
|---|---|---|
| `mOnOff` | 0–1 | manual power |
| `level` | 1–4, omit = Auto | the user-facing level. Auto drops the field |
| `mLevel` | 1–4 | separate manual baseline (seen pinned at 1) |
| `modeType` | `4` = environmental (auto) | mode |
- devSta: `{ modeType, level }`, **no `on` field**. `level` here = live output (0 when idle, 1–4
  while misting). So "is it running?" = `level > 0`.
- Quirk: the 1–4 selection is written to `level`; `mLevel` is a baseline that stays 1.

### dehumidifier (Low/High = presence of `mLevel`; never write `level`)
| field | range / values | meaning |
|---|---|---|
| `mOnOff` | 0–1 | manual power |
| `mLevel` | present `1` = High; omit = Low | the only level control (2 states, by presence) |
| `level` | **device-managed, always `1`, DO NOT WRITE** | writing `2` corrupts config + breaks both apps' sliders |
| `alarm` | `5` = water bucket full | fault |
| `modeType` | `4` = environmental (auto) | mode |
- devSta: `{ modeType, on, level }`. `on` = running flag (omitted when off); `level` = a 0/1
  duty-cycle, not the setpoint. So "is it running?" = **`on == 1`** (never `level > 0`).
- Quirk: only Low/High exist. High = include `mLevel:1`; Low = drop `mLevel`. Leave `level` alone.

### Safe-write summary (don't break the device)
| actuator.field | safe range | notes |
|---|---|---|
| `fan.mLevel` | 1–10 | |
| `blower.mLevel` | 25–100 | |
| `light(2).mLevel` | 0–100 | |
| `humidifier.level` | 1–4 (or omit for Auto) | the real level; not `mLevel` |
| `humidifier.mLevel` | 1–4 | baseline; normally 1 |
| `dehumidifier.mLevel` | omit=Low / `1`=High | presence-encoded; never write `2`+ |
| `dehumidifier.level` | **leave at 1, never write** | out-of-range here is the known corrupter |
| `*.mOnOff` | 0–1 | |

---

## 8. Minimal consumer recipe

```
data   = POST mailLogin {email,password,loginMethod:"1"}          # -> token, mqttName, mqttPwd
serial = (POST getDeviceList) device.deviceSerialnum              # group 7 for CB43
model  = productType without "MH-"                                # CB43

# live readings (MQTT, mutual-TLS, user=mqttName pwd=mqttPwd)
sub  MHPRO/<model>/API/UP/<serial>
pub  MHPRO/<model>/API/DOWN/<serial>  {"method":"getDevSta","params":{"pid":serial}}
# -> reply.data.sensor = { temp, humi, vpd, ppfd, tempSoil, humiSoil, ECSoil }

# history (REST)
POST getDeviceTHPData {type:"day", deviceSerialnum:serial, startDate, endDate}  # per-day, dedupe by id

# control (MQTT) - read-modify-write
cfg = pub getConfigFile -> data.configFile.device.fan
cfg.mLevel = 6
pub setConfigField {pid:serial, keyPath:["device","fan"], fan:cfg}
```

---

## 9. Notes & limits
- Unofficial API. Treat it as best-effort, isolate it behind one adapter, and handle token expiry by
  re-logging in. The MQTT layer is a firmware JSON-RPC contract, so it's stable in practice.
- The shared client cert is embedded in the app; the vendor could rotate it. (The broker doesn't even
  *require* it - auth is `username=email` + `password=account-uuid` at MQTT CONNECT.)
- Personal/own-device use. Don't hammer the API; mirror the `systemdata`/User-Agent shape.

### Endpoint versioning (per-endpoint `/vN`, verified by live probe 2026-06)
Every REST path ends in its own `/vN`; there is no single global API version, and the number is that
endpoint's own revision. The app binary (v2.1.0) references mostly `/v1` plus a couple of higher
numbers (`udm/getDeviceDetail/v18`, `data/dtd/v11`), but a live probe of the *server* shows:

| Endpoint | Live result |
|---|---|
| `udm/getDeviceDetail/v1` | `code 000`, 37 fields (android and ios) |
| `udm/getDeviceDetail/v2…v20` | `code 105 "no.method"` - not registered |
| `udm/getDeviceList/v1` | `code 000` |
| `udm/getDeviceList/v2,v3` | `105 "no.method"` |
| `data/dtd/v11` | `code 100 "fail"` (route may exist but rejects our params) |

So only `/v1` actually resolves for the device endpoints; the `/v18` from the string dump is almost
certainly a Dart string-pool artifact (adjacent fragments), not a callable route. Build against
`/v1`. Codes: `105 "no.method"` = route not found; `100 "fail"`/`"Request illegal."` =
gateway/validation reject (reached Spring Boot, params/signature wrong); `000` = success.
