# Mars Hydro / Mars Pro - Quickstart

Zero to a live reading in a few copy-paste steps. Pick one of the two tracks and follow it top to
bottom:

- **Track A · Command line only** - `curl` + `jq` for REST, the `mosquitto` clients for MQTT. No code.
- **Track B · Python** - one `mars.py` helper drives both REST and MQTT.

Two channels are in play either way: REST (`https://mars-pro.api.lgledsolutions.com`) for
login/inventory, MQTT/TLS (`mars-pro.mqtt.lgledsolutions.com:8883`) for live telemetry and control.
Auth is email + password; no request signing. Full contract:
[02_api-reference.md](02_api-reference.md) · quirks: [03_device-behaviour.md](03_device-behaviour.md).

> **Environment.** Every command assumes a POSIX shell: macOS, Linux, or Windows via WSL. They won't
> run in native PowerShell or `cmd.exe` (no `source`, heredocs, `$$`, or `chmod`), so open a WSL
> terminal on Windows. Track A also needs the `mosquitto` clients and `openssl`; Track B needs
> `python3`. Install via your package manager (`brew`, `apt`).

> ⚠️ **The backend does no validation.** Whatever you write is stored verbatim and pushed to the
> controller, bad values and all. There's no "bad request": the device usually ignores junk, but a bad
> write can wedge a control's slider in every app (fix: write a valid value back). The login,
> device-list, and live-read steps only read. Only the control step writes.

## The flow

Each call feeds the next: the ids and credentials from one step are the inputs to the following one.
The channel switches from REST to MQTT once you hold the device's MQTT creds.

```
1. REST  mailLogin       → token, mqttName, mqttPwd
2. REST  getDeviceList   → serial, productType  (model = productType without "MH-")
   ── switch to MQTT: the creds + serial/model are all it needs ──
3. MQTT  getDevSta       → live reading       (read)
4. MQTT  getConfigFile   → full config        (read)
5. MQTT  setConfigField  → writes the device  (QoS 1)
```

REST authenticates you and names the device; MQTT reads and controls it. Steps 1–4 only read; only 5
writes. Track A adds a step before 3 to fetch the broker's self-signed cert; Track B skips it in code.

---
---

# Track A · Command line only (curl + mosquitto)

You'll need `curl`, `jq`, `openssl`, and the `mosquitto` clients. Secrets go in one file
(`login.json`); the login step writes a `mars.env` that carries everything else.

## Setup (once)

Make a working directory and move into it:
```bash
mkdir -p ~/mars && cd ~/mars
```
Create your credentials file:
```bash
nano login.json
```
Paste this, fill in your real email/password, then save (Ctrl-O, Enter, Ctrl-X):
```json
{ "email": "you@example.com", "password": "yourpassword", "loginMethod": "1" }
```
Now create `mars.env`. It holds the hosts and an `sd` helper that builds the required `systemdata`
header, adding your token automatically once you have one:
```bash
cat > mars.env <<'EOF'
export API=https://mars-pro.api.lgledsolutions.com
export MQTT_HOST=mars-pro.mqtt.lgledsolutions.com
sd() { jq -nc --arg t "${TOKEN:-}" '{reqId:(now*1000|floor),appVersion:"2.1.0",osType:"android",osVersion:"14",deviceType:"sdk",deviceId:"sdk",netType:"wifi",wifiName:"x",timestamp:(now*1000|floor),language:"English"} + (if $t!="" then {token:$t,timezone:"0"} else {} end)'; }
EOF
```
Lock both files down (they hold your password and token):
```bash
chmod 600 login.json mars.env
```
`login.json` is the only file you put secrets in. Every step starts with `source ./mars.env` and uses
`$(sd)` for the header.

## 1. Log in

Logging in returns your token plus the MQTT username/password; we append all three to `mars.env`.

Load the env (hosts + the `sd` helper):
```bash
source ./mars.env
```
Send the login, capture the response in `$R`:
```bash
R=$(curl -s --compressed "$API/api/android/ulogin/mailLogin/v1" \
  -H 'Content-Type: application/json' -H 'User-Agent: Dart/3.5 (dart:io)' \
  -H "systemdata: $(sd)" -d @login.json)
```
Check it worked - expect `code: "000"` (if not, fix the email/password in `login.json`):
```bash
echo "$R" | jq '{code, msg}'
```
Append the token and MQTT creds to `mars.env`:
```bash
{ echo "export TOKEN=$(echo "$R" | jq -r .data.token)"
  echo "export MQTT_USER=$(echo "$R" | jq -r .data.mqttName)"
  echo "export MQTT_PWD=$(echo "$R" | jq -r .data.mqttPwd)"; } >> mars.env
```
Reload so `$(sd)` now includes your token:
```bash
source ./mars.env
```

## 2. Find your device

Reload the env (so `$(sd)` carries the token):
```bash
source ./mars.env
```
Scan product groups `0..9` and grab the first device into `$DEV`:
```bash
DEV=$(for g in $(seq 0 9); do
  curl -s --compressed "$API/api/android/udm/getDeviceList/v1" \
    -H 'Content-Type: application/json' -H 'User-Agent: Dart/3.5 (dart:io)' \
    -H "systemdata: $(sd)" -d "{\"currentPage\":1,\"type\":null,\"deviceProductGroup\":$g}" \
    | jq -c '.data.list[]?'
done | head -1)
```
Confirm it found one:
```bash
echo "$DEV" | jq '{deviceName, deviceSerialnum, productType}'
```
Append serial + model (`<model>` is `productType` minus the `MH-` prefix):
```bash
{ echo "export SERIAL=$(echo "$DEV" | jq -r .deviceSerialnum)"
  echo "export MODEL=$(echo "$DEV"  | jq -r '.productType | sub("^MH-";"")')"; } >> mars.env
```
`mars.env` now has everything MQTT needs: `MQTT_USER`, `MQTT_PWD`, `SERIAL`, `MODEL`.

## 3. Get the broker cert

The broker cert is self-signed and the `mosquitto` CLI can't fully skip verification (`--insecure` only
skips hostname checks, not chain validation), so fetch the CA once (public cert, no private key - the
broker shows its chain to every client, and it can rotate). The broker presents two certs: the leaf
server cert first, then the self-signed CA. You need the **CA**, so grab the second cert onward -
`openssl x509` alone would only keep the first (leaf) cert and verification would fail with a misleading
`Error: Protocol error`:
```bash
cd ~/mars
openssl s_client -connect mars-pro.mqtt.lgledsolutions.com:8883 -showcerts </dev/null 2>/dev/null \
  | awk '/-----BEGIN CERTIFICATE-----/{c++} c>=2{print}' > broker-ca.pem
```

## 4. Live read: getDevSta (read-only)

Subscribe to the `UP` topic in the background, publish a `getDevSta` to `DOWN`, and print the reply,
all in one terminal:
```bash
source ./mars.env
( mosquitto_sub -h "$MQTT_HOST" -p 8883 -V mqttv311 --cafile broker-ca.pem --insecure \
    -i "mars-sub-$$" -u "$MQTT_USER" -P "$MQTT_PWD" \
    -t "MHPRO/$MODEL/API/UP/$SERIAL" -C 1 | jq . ) &
sleep 1
mosquitto_pub -h "$MQTT_HOST" -p 8883 -V mqttv311 --cafile broker-ca.pem --insecure \
  -i "mars-pub-$$" -u "$MQTT_USER" -P "$MQTT_PWD" \
  -t "MHPRO/$MODEL/API/DOWN/$SERIAL" -m "{\"method\":\"getDevSta\",\"params\":{\"pid\":\"$SERIAL\"}}"
wait
```
`-C 1` makes the subscriber exit after the one reply; `$$` gives a unique client id. Topics are
UPPERCASE `UP`/`DOWN`.

<!-- The getDevSta example + liveness note below are duplicated in Track B §3. Keep them in sync. -->
You get `{ "method":"getDevSta", "data": … }`:
```jsonc
{ "sensor": { "temp":26.4, "humi":57.9, "vpd":1.45, "ppfd":916,
              "tempSoil":26.3, "humiSoil":68.7, "ECSoil":0.48, "isDaySensor":1 },
  "light":  { "modeType":12, "on":1, "level":78 },
  "fan":    { "on":1, "level":10 },                 // oscillating fan (no modeType)
  "blower": { "modeType":13, "on":1, "level":100 }, // inline/exhaust fan
  "humidifier":   { "modeType":4, "level":0 },      // socket: no `on` → idle
  "dehumidifier": { "modeType":4, "on":1, "level":1 } }
```
Liveness is asymmetric: the dehumidifier runs iff `on==1` (`level` is a setpoint); the humidifier
omits `on` and runs iff `level>0`; off devices collapse to `{ "level":0 }`.

**jq output styles.** `jq .` is pretty, auto-colored to a terminal. `jq -Cc` is compact, one colored
line per value (`-C` forces color through a pipe). One line per device:
`jq -Cc '.data | to_entries[] | {(.key):.value}'`. For a live monitor, drop the `-v` and pipe
`mosquitto_sub … | jq -Cc .` to print one line per frame as it arrives.

## 5. Control: setConfigField (writes)

This writes - heed the warning at the top. Read-modify-write: fetch the actuator's config (swap
`getDevSta` → `getConfigFile` above), change field(s), publish the whole object back at QoS 1 (`-q 1`):
```jsonc
{ "method":"setConfigField",
  "params":{ "pid":"<serial>", "keyPath":["device","blower"],
             "blower":{ /* full config with your change */ "maxSpeed":60, "minSpeed":35 } } }
```
The PUBACK (QoS 1) is your delivery confirmation; there are no per-request ids. Back up the config
first (re-fetch and save): a bad write can wedge a slider.

<!-- §6–§8 below are duplicated verbatim in Track B §5–§7. Edit both copies together. -->
## 6. Config: getConfigFile → data.configFile

```jsonc
{ "device": {
    "light":  { "modeType":12, "mLevel":70, "ppfdMinBrightness":60, "ppfdMaxBrightness":78,
                "ppfdPeriod":[{ "startTime":21600,"endTime":64800,"brightness":900,"fadeTime":1800,… }],
                "darkTemp":30, "offTemp":40 },
    "fan":    { "mOnOff":1, "mLevel":10, "maxSpeed":10, "minSpeed":3, "natural":1, "shakeLevel":10, … },
    "blower": { "modeType":13, "mLevel":41, "maxSpeed":100, "minSpeed":25, "closeCO2":0, … },
    "humidifier":   { "modeType":4, "mLevel":4, … },
    "dehumidifier": { "modeType":4, "level":1, "mLevel":1, … },
    "senConfig":    [ { "id":"…","type":2,"soilType":1,"calibration":{},"label":"P1" } ]
  }, "target": {…}, "alarm": {…}, "calibration": {…}, "system": {…} }
```
Times are seconds-since-midnight; `weekmask` is a 7-bit day mask (`127` = every day). `senConfig` is
the soil-probe registry (`type 2` = soil), soil-only: ambient temp/humi stays one reading.

## 7. Field reference

`modeType` (per device; Manual is the field omitted, never `0`):

| value | meaning |
|---|---|
| *(absent)* | Manual |
| `1` | Timer · light: Timer+Dimming |
| `2` | Cycle |
| `3` / `4` | inline fan Environmental: temp-only / humidity-only |
| `7` / `8` | inline fan Environmental: temp-priority / humidity-priority |
| `13` | inline fan Environmental: both |
| `4` | socket (humidifier/dehumidifier): Environmental |
| `12` | light: Timer+PPFD |

| device | level field(s) | range | notes |
|---|---|---|---|
| light/light2 | `mLevel` | 10–100 % | `ppfdPeriod[].brightness` µmol; `ppfdMin/MaxBrightness` 10–100 |
| fan (osc.) | `mLevel` manual · `maxSpeed` run | 1–10 | `minSpeed` standby · `shakeLevel` 0/5/10 = 0°/45°/90° |
| blower (inline) | `mLevel` manual · `maxSpeed` run | 25–100 % | `minSpeed` standby (below run) · `closeCO2` |
| humidifier | `level` 1–4 | 1–4 | level in `level`, not `mLevel`; Env "Auto" drops `level` |
| dehumidifier | Low/High via `mLevel` presence | - | High = `mLevel:1`; Low = drop `mLevel`. **Never write `level`** (corrupts it) |

Run speed (`maxSpeed`/`level`) is written in Timer/Cycle/Env (Timer `timePeriod.brightness` is
vestigial); Manual writes only `mLevel`.

## 8. Gotchas (presence encoding)

Off or disabled means drop the field - the server reads presence, not zero values:

| field | on | off |
|---|---|---|
| `mOnOff` (manual power) | `1` | **drop** |
| `natural` (fan natural-wind) | `1` | **drop** |
| `closeCO2` (CO₂ retain) | `1` | **drop** |
| `shakeLevel` (oscillation) | `5`/`10` | **drop** (= 0°) |
| `minSpeed` (standby) | value | **drop** |

- `mOnOff` is Manual-only: present in Manual+on, dropped in auto modes. The exception is the
  oscillating fan, which keeps it in every mode.
- Env-mode override: while `modeType` is environmental the device regulates to the tent target.
  Writing `mLevel` alone is ignored until you switch to Manual (drop `modeType`).
- Natural-wind (fan): modulates the motor but reports nothing. `level` reads 0, and on/off is
  unreadable while it's on.

---
---

# Track B · Python (urllib + paho-mqtt)

You'll need `python3` and `paho-mqtt` (`pip install paho-mqtt`); everything else is stdlib. You create
two files (`login.json` and `mars.py`); the login and device steps write a `mars.json` that the rest
of the flow reads.

## Setup (once)

Make a working directory and move into it:
```bash
mkdir -p ~/mars-py && cd ~/mars-py
```
Create your credentials file:
```bash
nano login.json
```
Paste this, fill in your real email/password, then save (Ctrl-O, Enter, Ctrl-X):
```json
{ "email": "you@example.com", "password": "yourpassword", "loginMethod": "1" }
```
Lock it down (it holds your password):
```bash
chmod 600 login.json
```
Now create the helper module:
```bash
nano mars.py     # paste the script below, save
```
```python
import json, os, ssl, time, urllib.request
import paho.mqtt.client as mqtt

API   = "https://mars-pro.api.lgledsolutions.com"
MQTT  = "mars-pro.mqtt.lgledsolutions.com"
STATE = "mars.json"   # token + MQTT creds + device, written by login()/find_device()

def _sd(token=None):
    """Build the required systemdata header (adds the token once you have one)."""
    now = int(time.time() * 1000)
    h = {"reqId": now, "appVersion": "2.1.0", "osType": "android", "osVersion": "14",
         "deviceType": "sdk", "deviceId": "sdk", "netType": "wifi", "wifiName": "x",
         "timestamp": now, "language": "English"}
    if token:
        h.update(token=token, timezone="0")
    return json.dumps(h)

def _post(path, body, token=None):
    req = urllib.request.Request(API + path, json.dumps(body).encode(),
        {"Content-Type": "application/json", "User-Agent": "Dart/3.5 (dart:io)",
         "systemdata": _sd(token)})
    return json.loads(urllib.request.urlopen(req).read())

def _state():
    return json.load(open(STATE)) if os.path.exists(STATE) else {}

def _save(**kw):
    s = _state(); s.update(kw); json.dump(s, open(STATE, "w")); os.chmod(STATE, 0o600)

def login():
    r = _post("/api/android/ulogin/mailLogin/v1", json.load(open("login.json")))
    print({"code": r.get("code"), "msg": r.get("msg")})   # expect code "000"
    d = r["data"]
    _save(token=d["token"], mqttName=d["mqttName"], mqttPwd=d["mqttPwd"])

def find_device():
    tok = _state()["token"]
    for g in range(10):                                   # scan product groups 0..9
        lst = (_post("/api/android/udm/getDeviceList/v1",
                     {"currentPage": 1, "type": None, "deviceProductGroup": g},
                     tok).get("data") or {}).get("list") or []
        if lst:
            d = lst[0]
            print({k: d[k] for k in ("deviceName", "deviceSerialnum", "productType")})
            _save(serial=d["deviceSerialnum"], model=d["productType"].replace("MH-", "", 1))
            return
    print("no device found")

def _request(method, params, qos=0):
    """Subscribe UP, publish one command to DOWN, return the first reply. TLS on, no verify."""
    s = _state()
    up   = f"MHPRO/{s['model']}/API/UP/{s['serial']}"     # topics are UPPERCASE
    down = f"MHPRO/{s['model']}/API/DOWN/{s['serial']}"
    c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2, client_id=f"mars-{os.getpid()}",
                    protocol=mqtt.MQTTv311)
    c.username_pw_set(s["mqttName"], s["mqttPwd"])
    c.tls_set(cert_reqs=ssl.CERT_NONE); c.tls_insecure_set(True)
    box = {}
    c.on_connect = lambda cl, *_: (cl.subscribe(up, 0),
        cl.publish(down, json.dumps({"method": method, "params": params}), qos=qos))
    c.on_message = lambda cl, _u, m: (box.setdefault("r", json.loads(m.payload)), cl.disconnect())
    c.connect(MQTT, 8883, 30); c.loop_forever()
    return box.get("r")

def read():
    print(json.dumps(_request("getDevSta", {"pid": _state()["serial"]}), indent=2))

def get_config():
    return _request("getConfigFile", {"pid": _state()["serial"]})["data"]["configFile"]

def control(dev, **changes):
    """READ-MODIFY-WRITE one device block at QoS 1. WRITES - heed the warning at the top."""
    s = _state()
    block = get_config()["device"][dev]   # back this up before changing it
    block.update(changes)
    _request("setConfigField",
             {"pid": s["serial"], "keyPath": ["device", dev], dev: block}, qos=1)
```
Install the one dependency:
```bash
pip install paho-mqtt
```

## 1. Log in

```bash
python3 -c 'import mars; mars.login()'      # expect: {'code': '000', 'msg': ...}
```
If you don't see `code '000'`, fix the email/password in `login.json` and re-run. The token and MQTT
creds are now in `mars.json`.

## 2. Find your device

```bash
python3 -c 'import mars; mars.find_device()'
```
`mars.json` now has everything MQTT needs: `mqttName`, `mqttPwd`, `serial`, `model`. Python skips the
broker-cert step; `tls_insecure_set(True)` connects without verifying the self-signed cert.

## 3. Live read: getDevSta (read-only)

```bash
python3 -c 'import mars; mars.read()'       # prints the live reading, pretty-printed
```
<!-- The getDevSta example + liveness note below are duplicated in Track A §4. Keep them in sync. -->
You get `{ "method":"getDevSta", "data": … }`:
```jsonc
{ "sensor": { "temp":26.4, "humi":57.9, "vpd":1.45, "ppfd":916,
              "tempSoil":26.3, "humiSoil":68.7, "ECSoil":0.48, "isDaySensor":1 },
  "light":  { "modeType":12, "on":1, "level":78 },
  "fan":    { "on":1, "level":10 },                 // oscillating fan (no modeType)
  "blower": { "modeType":13, "on":1, "level":100 }, // inline/exhaust fan
  "humidifier":   { "modeType":4, "level":0 },      // socket: no `on` → idle
  "dehumidifier": { "modeType":4, "on":1, "level":1 } }
```
Liveness is asymmetric: the dehumidifier runs iff `on==1` (`level` is a setpoint); the humidifier
omits `on` and runs iff `level>0`; off devices collapse to `{ "level":0 }`.

**Live monitor.** `read()` exits after one frame because `on_message` disconnects. To stream instead,
copy `_request` without the `cl.disconnect()` and print each frame as it arrives.

## 4. Control: setConfigField (writes)

This writes - heed the warning at the top. `control()` does the read-modify-write for you: fetch the
actuator's config, change field(s), publish the whole object back at QoS 1. For example, set the
blower's run/standby speeds:
```bash
python3 -c 'import mars; mars.control("blower", maxSpeed=60, minSpeed=35)'
```
The PUBACK (QoS 1) is your delivery confirmation; there are no per-request ids. Back up the config
first - `python3 -c 'import mars,json; print(json.dumps(mars.get_config()))' > config.bak.json` - a
bad write can wedge a slider.

<!-- §5–§7 below are duplicated verbatim in Track A §6–§8. Edit both copies together. -->
## 5. Config: getConfigFile → data.configFile

```jsonc
{ "device": {
    "light":  { "modeType":12, "mLevel":70, "ppfdMinBrightness":60, "ppfdMaxBrightness":78,
                "ppfdPeriod":[{ "startTime":21600,"endTime":64800,"brightness":900,"fadeTime":1800,… }],
                "darkTemp":30, "offTemp":40 },
    "fan":    { "mOnOff":1, "mLevel":10, "maxSpeed":10, "minSpeed":3, "natural":1, "shakeLevel":10, … },
    "blower": { "modeType":13, "mLevel":41, "maxSpeed":100, "minSpeed":25, "closeCO2":0, … },
    "humidifier":   { "modeType":4, "mLevel":4, … },
    "dehumidifier": { "modeType":4, "level":1, "mLevel":1, … },
    "senConfig":    [ { "id":"…","type":2,"soilType":1,"calibration":{},"label":"P1" } ]
  }, "target": {…}, "alarm": {…}, "calibration": {…}, "system": {…} }
```
Times are seconds-since-midnight; `weekmask` is a 7-bit day mask (`127` = every day). `senConfig` is
the soil-probe registry (`type 2` = soil), soil-only: ambient temp/humi stays one reading.

## 6. Field reference

`modeType` (per device; Manual is the field omitted, never `0`):

| value | meaning |
|---|---|
| *(absent)* | Manual |
| `1` | Timer · light: Timer+Dimming |
| `2` | Cycle |
| `3` / `4` | inline fan Environmental: temp-only / humidity-only |
| `7` / `8` | inline fan Environmental: temp-priority / humidity-priority |
| `13` | inline fan Environmental: both |
| `4` | socket (humidifier/dehumidifier): Environmental |
| `12` | light: Timer+PPFD |

| device | level field(s) | range | notes |
|---|---|---|---|
| light/light2 | `mLevel` | 10–100 % | `ppfdPeriod[].brightness` µmol; `ppfdMin/MaxBrightness` 10–100 |
| fan (osc.) | `mLevel` manual · `maxSpeed` run | 1–10 | `minSpeed` standby · `shakeLevel` 0/5/10 = 0°/45°/90° |
| blower (inline) | `mLevel` manual · `maxSpeed` run | 25–100 % | `minSpeed` standby (below run) · `closeCO2` |
| humidifier | `level` 1–4 | 1–4 | level in `level`, not `mLevel`; Env "Auto" drops `level` |
| dehumidifier | Low/High via `mLevel` presence | - | High = `mLevel:1`; Low = drop `mLevel`. **Never write `level`** (corrupts it) |

Run speed (`maxSpeed`/`level`) is written in Timer/Cycle/Env (Timer `timePeriod.brightness` is
vestigial); Manual writes only `mLevel`.

## 7. Gotchas (presence encoding)

Off or disabled means drop the field - the server reads presence, not zero values:

| field | on | off |
|---|---|---|
| `mOnOff` (manual power) | `1` | **drop** |
| `natural` (fan natural-wind) | `1` | **drop** |
| `closeCO2` (CO₂ retain) | `1` | **drop** |
| `shakeLevel` (oscillation) | `5`/`10` | **drop** (= 0°) |
| `minSpeed` (standby) | value | **drop** |

- `mOnOff` is Manual-only: present in Manual+on, dropped in auto modes. The exception is the
  oscillating fan, which keeps it in every mode.
- Env-mode override: while `modeType` is environmental the device regulates to the tent target.
  Writing `mLevel` alone is ignored until you switch to Manual (drop `modeType`).
- Natural-wind (fan): modulates the motor but reports nothing. `level` reads 0, and on/off is
  unreadable while it's on.
