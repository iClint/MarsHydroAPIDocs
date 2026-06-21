# Mars Hydro - Device Behaviour & Quirks

Runtime behaviour observed on a live MH-CB43 controller that the API *contract*
([`02_api-reference.md`](02_api-reference.md)) doesn't make obvious. These are the things that bite you
when you build on top of the API: which field actually means "running", which devices you can tell
are unplugged, and what the fault codes mean.

Everything here was captured against real hardware (the controller plus its oscillating fan,
inline (EC) fan, humidifier, dehumidifier, and env/soil sensors) by flipping the physical device
and watching `getDevSta`. **Do not assume a field means what its name suggests; the API is
inconsistent per device class.**

---

## 1. "Is it running?" is per-device - liveness is asymmetric

`getDevSta` reports each actuator differently. There is no single rule for whether a device is on.
Verified by capture:

| Actuator | devSta fields | Liveness rule | Notes |
|---|---|---|---|
| `light` / `light2` | `{ modeType, on, level }` | `on == 1` | `level` = live brightness % |
| `fan` (oscillating) | `{ on, level }` | `on == 1` | `level` = set gear 1–10 (setpoint, not instantaneous speed) |
| `blower` (inline/EC fan) | `{ modeType, on, level }` | `on == 1` | `level` = live speed % |
| `humidifier` | `{ modeType, level }`, no `on` | `level > 0` | `level` here is live output (0 when idle, 1–4 while misting) |
| `dehumidifier` | `{ modeType, on, level }` | `on == 1` only | `level` is a fixed 0/1 duty-cycle setpoint, stays `1`; can report `on:1, level:0`. `level > 0` is not liveness |

The two sockets are the trap:

- Humidifier never sends an `on` field. Its `level` *is* the live output, so use **`level > 0`**.
- Dehumidifier sends an explicit `on` (omitted when off) and keeps `level` pinned at `1` even when
  stopped, so use **`on == 1`**. A naïve `level > 0` shows a stopped dehumidifier as ON.

> Config (`getConfigFile`) vs live (`getDevSta`) also diverge: config holds a *setpoint*, devSta
> may hold *live output*. They are not interchangeable. See the level-encoding tables in
> [`02_api-reference.md` §7a](02_api-reference.md).

---

## 2. Presence detection - who you can tell is unplugged

Physically unplugging devices from the controller, polling `getDevSta` every ~3 s. Devices split
into dumb (open-loop) and smart (sensed); they are not uniform.

### Sensors: detectable (fields appear/disappear)
- Unplug the soil probe and `tempSoil`, `humiSoil`, `ECSoil` all drop from the sensor object
  together (one probe, its three fields, as a unit); they return together on replug.
- CO₂ sensor: the `co2` field appears when plugged, disappears when unplugged.
- The main env probe (`temp`/`humi`/`vpd`) stays present throughout a soil unplug.
- Rule: absence of a sensor field means that probe is unplugged. Reliable.

### Smart actuators: detectable (key drops)
- Inline fan (`blower`): the whole `blower` key disappears from `data` when unplugged, and returns
  (at its last setting) when replugged. This is the reliable "connected right now" signal. It
  *also* raises an `alarmLast` event: `{ id, epoch, devType: 19, alarmType: 3 }` (devType 19 =
  inline fan, alarmType 3 = disconnected).
- Sockets (de/humidifier): unplug drops the socket's key from `data` (detectable), but no alarm
  fires (`alarmLast` unchanged). So the disconnect-alarm channel is inline-fan-only.

### Dumb actuators: NOT detectable (command-echo)
- The oscillating fan and the main light are open-loop (PWM/dimming, no feedback). Unplug them and
  telemetry is unchanged: the key stays present, no fault, `level` keeps reporting the last
  commanded value.
- Writing to an unplugged dumb device succeeds and the echo confirms it, but it's just the
  last-commanded setpoint mirrored back, unrelated to the physical device. The write **persists and
  applies on reconnect**. Gotcha: a "save" to an unplugged dumb device silently takes effect later,
  and no telemetry tells you it didn't apply now.
- The main grow light, despite being the headline device, has **no disconnect signal whatsoever**.

### Unified rule
A device's key/field missing from `getDevSta` (when it should be present) means disconnected.
Covers sensors plus the inline fan plus the sockets. The light and oscillating fan are the only
blind spots. `alarmLast` is a bonus signal that only the inline fan raises (devType 19 / alarmType
3), and it is sticky: it persists in every frame and is overwritten only by a newer alarm, so use
it for "what/when was the last fault," not for "is it connected now."

Three cases a consumer should handle:
1. Comms failure (whole controller down): `getDevSta` goes stale (no frames).
2. Sensor not present: its field(s) vanish, so it's detectable; omit that row.
3. Dumb actuator not present: invisible (command-echo), so not detectable; nothing to surface.

---

## 3. Fault / alarm codes (`getDevSta` actuator `.alarm`)

Sockets report faults in the actuator's `alarm` field. Confirmed against physical state:

| Device | `alarm` | Meaning | Status |
|---|---|---|---|
| dehumidifier | `5` | water collection bucket full | confirmed; also logged server-side as `alarmType 5` in `notice_msg/queryMessageList` |
| humidifier | `4` | tank empty / needs refill | confirmed in `getDevSta` (frame still shows `on:1, level:4` - trying to run but dry); server-side `alarmType` for this not yet captured |

`devType` / `alarmType` codes seen so far (incomplete - see [`04_known-gaps.md`](04_known-gaps.md)):

| Code | Field | Meaning |
|---|---|---|
| `devType 26` | notice_msg | CB43 controller class |
| `devType 19` | alarmLast / notice_msg | inline (EC) fan |
| `alarmType 3` | alarmLast / notice_msg | device disconnected |
| `alarmType 5` | notice_msg | dehumidifier bucket full |

The config's `alarm` section (`vmin`/`vmax` per sensor, see [`02_api-reference.md` §6](02_api-reference.md))
defines *sensor threshold* alarms; a reading outside the band raises a `notice_msg` entry. Those
are distinct from the hardware faults above (bucket full / tank empty / disconnect), which the
device raises directly.

---

## 4. "Natural wind" is real but invisible to the API

The oscillating fan's `natural: 0/1` flag (in `getConfigFile.fan`) genuinely works: with the inline
fan off you can *hear* the motor ramp up and down. But the modulation is entirely motor-side and is
never reported over MQTT. Across minutes of monitoring, `fan.level` stayed pinned at the set gear
while the fan was audibly swinging. `getDevSta.fan` carries only `{ on, level }`, and `level` is the
setpoint, not the instantaneous speed.

Implication: there is nothing for *any* client to surface beyond the flag's enabled/disabled state,
not even the official app can show the live swing, because the API is blind to it. The behaviour
lives in device firmware.
