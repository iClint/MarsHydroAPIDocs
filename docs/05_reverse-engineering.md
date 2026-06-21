# Reverse‑engineering the Mars Hydro / Mars Pro cloud API (for local control)

*A technical writeup for the curious — how the Mars Hydro "Mars Pro" app talks to its cloud, so
you can build your own dashboards / Home Assistant integrations. Interop research on my own account
and my own gear. No credentials, serials, or certs are included.*

---

## TL;DR

- The **Mars Pro** app (package `com.marspro.meizhi` — also covers **Meizhi**, a Mars sub‑brand) talks
  to an **LG‑LED Solutions** cloud over two channels:
  - **REST** (`mars-pro.api.lgledsolutions.com`) — login, device list, history.
  - **MQTT** (`mars-pro.mqtt.lgledsolutions.com:8883`, an **EMQX** broker) — **live telemetry + control**.
- **Auth is just email + password.** The REST login returns a token *and* a static **account UUID**.
  The MQTT broker authenticates with **username = your email, password = that UUID**.
- The app bundles a "mutual‑TLS" **client certificate — but it's vestigial**: neither the API nor the
  broker actually requires it.
- You can build a **local live dashboard with no cloud token at all** — just the MQTT channel.
- **Security:** the broker enforces proper **per‑account ACLs** (you can't touch devices you don't own
  — verified). The one real weakness is that the **MQTT server cert isn't verified** by clients, so the
  live channel is **MITM‑able on a hostile network**.

---

## How it's put together

The app is **Flutter (Dart)**. The real logic lives in the compiled `libapp.so`, which is only in the
**arm64 split APK** of the app bundle (the base split has none of it). Workflow that worked:

1. Pull the split APKs, merge them (APKEditor), and extract `lib/arm64-v8a/libapp.so`.
2. Recover strings/structure with **Blutter** (Dart snapshot tooling). The app is obfuscated, but
   string literals survive in the object pool — that's where the endpoint paths and topic templates live.
3. Confirm everything against live traffic with a TLS MITM (rooted emulator + mitmproxy + a relay for
   the MQTT broker).

Hosts:

| Purpose | Endpoint |
|---|---|
| REST API | `https://mars-pro.api.lgledsolutions.com` |
| MQTT broker | `mars-pro.mqtt.lgledsolutions.com:8883` (TLS) |
| Static files | `mars-pro.file.lgledsolutions.com` |

---

## Channel 1 — REST (login / inventory / history)

Path scheme: `/api/{android|ios}/<module>/<action>/vN`. Every request carries a **`systemdata`** JSON
header (app/OS info, a millisecond timestamp, and — once logged in — the bearer `token`). **There is no
HMAC / request signature** — the token in `systemdata` is the whole gate.

**Login:**

```
POST /api/android/ulogin/mailLogin/v1
systemdata: {... no token yet ...}
body: { "email": "...", "password": "...", "loginMethod": "1" }

→ { "code":"000", "msg":"success",
    "data": { "token":"…", "mqttName":"…", "mqttPwd":"…", … } }
```

`mqttPwd` is your **account UUID** — a stable value you'll reuse as the MQTT password. Other endpoints:
`udm/getDeviceList/v1`, `udm/getDeviceDetail/v1`, `dr/getDeviceTHPData/v1` (history), `mine/info/v1`, etc.

**Versioning gotcha:** paths are versioned **per endpoint** (`/vN`), but a live probe shows that only
**`/v1` actually resolves** for the device endpoints — `v2…v20` return `105 "no.method"`. A `/v18` that
shows up in the decompiled strings is a Dart **string‑pool artifact**, not a real route. Build against `/v1`.

---

## Channel 2 — MQTT (the interesting part)

This is the real‑time channel — the app's dashboard is **cloud MQTT pub/sub**, not REST polling.

- Broker: EMQX on `:8883` over TLS. **Auth: username = email, password = account UUID.**
- Topics: `MHPRO/<model>/API/{UP|DOWN}/<serial>`
  - `DOWN` = app → device (requests/commands)
  - `UP` = device → app (replies/telemetry)
- It's **JSON‑RPC**. You publish a request to `DOWN`, the device answers on `UP`:

```
publish → MHPRO/<model>/API/DOWN/<serial>
  { "method": "getDevSta", "params": { "pid": "<serial>" } }

receive ← MHPRO/<model>/API/UP/<serial>
  { "method":"getDevSta", "code":200, "data": {
      "sensor": { "temp":…, "humi":…, "vpd":…, "ppfd":…,
                  "tempSoil":…, "humiSoil":…, "ECSoil":…,
                  "isDaySensor":0/1 },
      "light":{…}, "fan":{…}, "blower":{…}, "humidifier":{…}, "dehumidifier":{…}
  }}
```

Methods seen:

| Method | Returns / does |
|---|---|
| `getDevSta` | live sensors + actuator on/off/level (read) |
| `getConfigFile` | full config — incl. **alarm bands** (`vmin/vmax` per metric) and **day/night targets** |
| `getSysSta` | firmware / WiFi / timezone |
| `setConfigField` | **control** — change an actuator setting (write) |

`getDevSta` is a pure status request (changes nothing), so a **read‑only** monitor is trivial and safe.

---

## The bundled certificate is a red herring

The app ships an EMQX client cert + key and appears to do "mutual TLS." In practice it's **not enforced**:

- Connecting to the broker with **no client cert at all** (just username + password) → **`CONNACK: Success`**.
- The REST API **never uses** the client cert — it's plain HTTPS with a normal CA‑issued server cert.

So the cert is vestigial. The only things that actually authenticate you anywhere are **email + password**
(REST) and **email + account UUID** (MQTT).

---

## Security assessment

The interesting question for anyone building on this: *can someone mess with my tent, or read it?*

- ✅ **Per‑account ACLs are enforced (verified).** Subscribing (with valid creds) to a topic for a serial
  that **isn't on your account** returns a **SUBACK denial** — no data flows. The broker maps the
  authenticated user → their own device serials. **Knowing someone else's serial does not grant access.**
  No cross‑tenant / IDOR problem.
- ⚠️ **MQTT TLS is encrypted but *not authenticated*.** The broker presents a **self‑signed** cert and
  clients connect with verification **disabled**. Passive eavesdropping is blocked (it's still TLS), but
  an **active man‑in‑the‑middle** on the path (rogue AP, hostile router, DNS hijack) can present any cert,
  the client accepts it, and the plaintext‑JSON payload — including your email + UUID — is exposed. On a
  trusted home network you're fine; on untrusted networks this is a genuine weakness.
- ⚠️ **The account UUID is a static, non‑rotating "password."** Poor hygiene — but its blast radius is
  bounded to your own devices by the ACLs above.
- ✅ **REST is properly secured** — real CA‑issued cert, verified end‑to‑end.

Net: the authorization model is actually sound; the one legitimate finding is the **disabled MQTT server‑cert
verification**.

---

## What you can build with this

- A **local read‑only dashboard** — connect to the broker, subscribe to your `UP` topic, poll `getDevSta`.
  No cloud token needed; it's the same data the official app shows, in real time.
- A **Home Assistant / MQTT bridge** for temp / humidity / VPD / PPFD / soil moisture+EC and equipment state.
- **History graphs** and **feed/EC calculators** (history needs the REST `getDeviceTHPData` endpoint).
- Controls are possible (`setConfigField`) but obviously do that carefully — it writes to live hardware.

---

## Caveats / honesty

- **Unofficial & undocumented.** The vendor can change it. The MQTT/firmware contract is sticky (changing
  it means firmware updates to every controller), but the **REST login is their easy lever** — they could
  add request signing with just a backend + app update and break new sign‑ins until it's re‑derived.
- **It's against their ToS.** Mars Hydro's app agreement prohibits reverse engineering and third‑party
  clients. This is **personal, non‑commercial interop research on my own account and my own device**, shared
  as knowledge — not a product, and not a tool to access anything that isn't yours.
- **Don't redistribute** the bundled cert/key or anybody's credentials. There's nothing sensitive *to*
  redistribute here anyway — the cert is unused and every account uses its own email/UUID.
- The cross‑tenant ACLs are solid, so this isn't a "here's how to break into other people's grows" post —
  it's "here's the contract, go build your own local control."

*Build local, keep your data, and verify everything yourself.*
