# FisionTV+ Network Certifier — Backend API Spec

**Hand this document to a fresh Claude session as the brief for building the backend.**
It is intentionally self-contained; the reader has no prior context.

---

## 1. Product context

Hotwire Communications (HWC) is a regional ISP whose TV product is **FisionTV+**.
We're building a small Android TV app called the **FisionTV+ Network Certifier**
that runs on a customer's set-top box (STB) and certifies whether the connection
can sustain a given streaming quality tier (SD / HD / Full HD / 4K / 4K HDR).

The on-device app runs a sequence of tests (server selection, DNS, latency,
parallel HTTPS download, parallel HTTPS upload, DASH playback) against HWC's
own Ookla speedtest infrastructure and produces a **CertificationResult**:
the achieved tier plus the underlying metrics.

This backend has two responsibilities:

1. **Serve runtime config to the app** so we can change server lists,
   tier thresholds, test durations, and the playback manifest URL **without
   shipping a new APK**. STBs are slow to update in the field — config-driven
   behavior is critical.
2. **Receive certification results** from the field for support, fleet
   analytics, and capacity planning.

Everything else (a dashboard, alerting, BI integration) is out of scope here —
keep the surface small and focused on those two endpoints.

---

## 2. Non-functional requirements

- **Idempotent writes.** STBs are on flaky networks. The same certification
  may be POSTed multiple times. Dedupe on a client-supplied UUID.
- **Forward-compatible schema.** The app will be in the field for years and may
  run against a backend that's been schema-bumped. App sends `schemaVersion`;
  server must accept any version it knows or returns a clear `426 Upgrade Required`.
- **Backward-compatible config.** When you add new fields to `cert-config`, old
  apps must keep working. Never remove or rename fields; only add.
- **Zero customer PII unless explicitly approved.** `bssid`, `ssid`, `publicIp`,
  and `gatewayIp` are identifying. Hash or drop on ingest unless legal/privacy
  has signed off. The app will send them; the server is the policy enforcement
  point.
- **Low latency for config GET.** This is on the app's startup path; aim for
  < 200 ms p95 from US edges.
- **Auth that works for unattended STBs.** No human will be present to log in.
  See § 5.
- **Observability.** Each ingest should be queryable by `certificationId`,
  `deviceId`, and date. Expose at minimum a `GET /v1/certifications/{id}` for
  support to pull a single result by UUID.

---

## 3. Stack — recommended, not prescribed

Pick whatever your team owns. Sensible defaults if you have no opinion:

- **Language**: Go or TypeScript (Node + Fastify). Both fit the request shape
  cleanly.
- **DB**: Postgres. The result payload is mostly fixed-shape with a few JSONB
  blobs for samples.
- **Deployment**: anything that gives you a stable HTTPS endpoint with sane TLS
  (the STB pins nothing — standard public CA chain is fine).
- **Schema migrations**: own them from day one. `golang-migrate`, Prisma
  migrations, or whatever your team uses.

If you build in TypeScript, use `zod` (or equivalent) for request validation
against the schemas in § 6 — they are the contract.

---

## 4. Endpoints

### 4.1 `GET /v1/cert-config`

Called by the app on launch and on each "Run again". Cached locally with a
1-hour TTL; the app falls back to bundled defaults if this 5xx's or the
returned `schemaVersion` is higher than the app understands.

**Request headers:**

| Header                    | Required | Notes                                  |
|---------------------------|----------|----------------------------------------|
| `Authorization: Bearer …` | yes      | See § 5                                |
| `X-Device-Id: <uuid>`     | yes      | App-generated, persistent per install  |
| `X-App-Version: 0.1.0`    | yes      | App build version                      |
| `X-Schema-Version: 1`     | yes      | Highest schema the client understands  |

**Response 200:** body = `CertConfig` (§ 6.1).

**Response 304:** if client sends `If-None-Match` and ETag matches.

**Response 426:** if `X-Schema-Version` is older than minimum supported.

### 4.2 `POST /v1/certifications`

Called by the app at the end of each successful run. Idempotent on
`certificationId`.

**Request headers:** same as above plus `Content-Type: application/json`.

**Request body:** `CertificationResult` (§ 6.2).

**Responses:**

- **201 Created** — first time we've seen this `certificationId`.
- **200 OK** — duplicate POST (same `certificationId`, same payload hash); body
  echoes the stored record. App treats both as success.
- **409 Conflict** — same `certificationId` but different payload hash. Almost
  always a client bug; log loudly and reject.
- **400** — schema validation failed. Body: `{ error, details: [{ path, msg }] }`.
- **401** — bad/missing token.
- **413** — payload too large. Cap at 256 KB; samples arrays are the offender.

### 4.3 `GET /v1/certifications/{id}`

For HWC support to pull a single result by UUID. Returns the stored record or
404. Same auth as above. Not called by the app.

### 4.4 `GET /v1/app/version`

Called by the app on launch in parallel with the cert-config fetch. The
response describes the latest released APK plus the minimum versionCode
the client is allowed to run a certification at. The client uses the
manifest to gate its "Run certification" button:

- `installed >= latestVersionCode` → button reads "Run certification"
- `minRequiredVersionCode <= installed < latestVersionCode` → button still
  works, but a banner above offers an update
- `installed < minRequiredVersionCode` → button flips to "Update & run";
  the cert is **blocked** until the update is downloaded, verified,
  installed, and the app relaunches

**Why this exists.** Cert correctness changes (tier thresholds, server
lists, probe revisions) need to ship in hours. Firmware OTAs ship in
weeks. This endpoint plus the APK download is how the app bumps itself
between OTA cycles. Firmware OTA remains the floor — every STB ships
with a baseline working app.

**Request headers:**

| Header                       | Required | Notes                                        |
|------------------------------|----------|----------------------------------------------|
| `Authorization: Bearer …`    | yes      | See § 5                                      |
| `X-Device-Id: <uuid>`        | yes      | App-generated, persistent per install        |
| `X-App-Version: 0.5.0`       | yes      | Installed app's `versionName`                |
| `X-App-Version-Code: 50`     | yes      | Installed app's `versionCode`                |
| `If-None-Match: <etag>`      | no       | ETag from a prior 200; server returns 304 if unchanged |

**Response 200:** body = `AppVersionManifest` (§ 6.4).

**Response 304:** if `If-None-Match` matches the current manifest's ETag.

**Response 426:** if `X-App-Version-Code` is below a hard server-enforced
floor (e.g., a known-broken release has been pulled). Client treats this
the same as `installed < minRequiredVersionCode` — block, surface, force
update.

**Integrity model (client-side).** The server's job is to return an
accurate manifest. The client enforces three invariants before installing
any APK it has downloaded:

1. The downloaded bytes hash to `apkSha256`. (Catches MITM and corrupted
   downloads.)
2. The APK's signing certificate SHA-256 matches the manifest's
   `signingCertSha256` AND the compile-time pin in
   `BuildConfig.APP_SIGNING_CERT_SHA256` (set from the keystore at
   firmware-build time). (Catches a compromised manifest endpoint — the
   attacker would also need the production signing key.)
3. The APK's `versionCode` is strictly greater than the installed
   `versionCode`. (Catches downgrade attacks and stale manifests.)

A failure of any check refuses the install and surfaces the reason to the
field tech. The downloaded APK is deleted from disk.

**APK hosting.** The `apkUrl` in the manifest is absolute. It may be the
API host itself or a CDN. The client treats it as opaque — no auth is
required to download (the integrity model above is what protects the
client). Recommendation: serve from a CDN behind a long cache lifetime
keyed on the APK content hash.

---

## 5. Auth — pick one of these and write it down

The STB is unattended; there is no user identity. Two viable approaches:

**Option A: Per-install bearer token issued at first launch.**
- App, on first launch, calls `POST /v1/devices/register` with a generated
  `deviceId` (UUID) and gets back a long-lived bearer token. Token is stored
  in app private storage.
- Subsequent calls present the token. Server can revoke per-device.
- **Recommended** for anything beyond the first prototype. Slightly more
  backend work; much better operational story.

**Option B: HMAC-signed requests with a shared secret baked into the APK.**
- Faster to ship. Secret is one rotation away from being on every customer's
  STB forever. Acceptable for a closed pilot, not for a fleet rollout.

Whichever you pick: document the choice in the README of the backend repo and
match it on the client. Don't half-build both.

---

## 6. Schemas

All timestamps are RFC 3339 UTC strings. All durations are explicit
(`Sec` / `Ms` suffix). Never assume units.

### 6.1 `CertConfig` (response of `GET /v1/cert-config`)

```json
{
  "schemaVersion": 1,
  "configVersion": "2026-05-06.1",
  "tests": {
    "download":  { "parallel": 8 },
    "upload":    { "parallel": 16 },
    "playback":  { "manifestUrl": "https://dash.akamaized.net/akamai/bbb_30fps/bbb_30fps.mpd", "durationSec": 20 }
  },
  "tiers": [
    { "id": "sd",      "displayName": "SD",      "minDownloadMbps": 5,  "minUploadMbps": 1, "maxLatencyMs": 200, "maxJitterMs": 50, "minPlaybackHeight": 480,  "minPlaybackBitrateKbps": 1500 },
    { "id": "hd",      "displayName": "HD",      "minDownloadMbps": 12, "minUploadMbps": 3, "maxLatencyMs": 120, "maxJitterMs": 30, "minPlaybackHeight": 1080, "minPlaybackBitrateKbps": 4000 },
    { "id": "uhd",     "displayName": "4K",      "minDownloadMbps": 40, "minUploadMbps": 5, "maxLatencyMs": 80,  "maxJitterMs": 20, "minPlaybackHeight": 2160, "minPlaybackBitrateKbps": 16000 },
    { "id": "uhd_hdr", "displayName": "4K HDR",  "minDownloadMbps": 55, "minUploadMbps": 5, "maxLatencyMs": 50,  "maxJitterMs": 15, "minPlaybackHeight": 2160, "minPlaybackBitrateKbps": 25000, "requiresHdr": true }
  ],
  "uploadResults": { "enabled": true, "endpoint": "/v1/certifications" },

  "wifiLinkQuality": {
    "excellentRssiMin": -55,
    "strongRssiMin": -65,
    "goodRssiMin": -75,
    "rateAdaptationDegradedThreshold": 0.5
  },
  "healthAssessment": {
    "excellentMin": 80,
    "strongMin": 55,
    "goodMin": 30,
    "topTierStretchUpFactor": 1.5,
    "topTierStretchDownFactor": 0.66
  }
}
```

**Field rules:**

- `schemaVersion` is a single integer. Bump it when the *shape* of this
  document changes in a non-additive way. Apps refuse to use a config whose
  `schemaVersion` is greater than they understand.
- `configVersion` is opaque (string). The app round-trips it in the result POST
  so support can correlate which config a given run used.
- `servers[].weight` defaults to 1.0. Use it to drain a server from selection
  (set to 0) without removing it from the list (which we do want auditable).
- `uploadResults.enabled = false` is the kill switch if the result pipeline
  breaks; the app will skip the POST.
- `wifiLink` is `null` when transport is not Wi-Fi. **It is advisory and does
  not influence `achievedTier` or `health`.** It exists so a tech (and the
  fleet dashboard) can see "the connection certified at HD today, but the
  Wi-Fi link is marginal so this customer is one microwave away from
  buffering." Treat the `rating` enum the same as `health.rating`
  (`EXCELLENT` / `STRONG` / `GOOD` / `MARGINAL`).
- **The `tiers[].min*` / `max*` values are the *certification* thresholds,
  not the bare streaming spec.** They are intentionally buffered above the
  minimum bitrate / latency a codec can technically sustain (throughput
  ×1.5, latency ×0.75, jitter ×0.6 vs. raw streaming floors). A connection
  that just clears the codec spec but not these thresholds is *not* certified
  for that tier — the headroom keeps the customer working when a second
  device joins, evening peak hits, or the Wi-Fi link degrades after install.
- `wifiLinkQuality` and `healthAssessment` are **optional**; when missing,
  the app falls back to bundled defaults per-field (same per-field
  tolerance as the rest of `tests.*`). See §6.1.1 for the full tunables
  reference.

#### 6.1.1 Server-tunable knobs reference

This table is the single source of truth for what an operator can change
server-side (no APK push) versus what's hard-coded in the app or
delegated to the Ookla binary. Every row here corresponds to a field in
`CertConfig`; everything not listed is in code.

| Path | Type | Range | Default | What it controls |
|---|---|---|---|---|
| `tests.download.parallel` | int | 1–16 | 8 | TCP streams the Ookla binary opens for download. Passed via `--download-conn-range`. |
| `tests.upload.parallel` | int | 1–16 | 16 | Same for upload (`--upload-conn-range`). |
| `tests.playback.manifestUrl` | string | non-empty | bbb_30fps.mpd | DASH manifest the playback probe loads. |
| `tests.playback.durationSec` | int | 5–120 | 20 | Length of the playback probe — controls our own Media3 player. |
| `tiers[].minDownloadMbps` etc. | per Tier schema | per tier | see §6.1 | Certification thresholds — buffered above streaming spec. |
| `uploadResults.enabled` | bool | – | true | **Kill switch.** false ⇒ STB drops results on the floor. |
| `uploadResults.endpoint` | url | – | – | Where to POST results. |
| `wifiLinkQuality.excellentRssiMin` | int dBm | -100–0 | -55 | RSSI ≥ this → STRONG badge. |
| `wifiLinkQuality.strongRssiMin` | int dBm | -100–0 | -65 | [strongRssiMin, excellentRssiMin) → STRONG. |
| `wifiLinkQuality.goodRssiMin` | int dBm | -100–0 | -75 | [goodRssiMin, strongRssiMin) → GOOD. Below → MARGINAL/WEAK. |
| `wifiLinkQuality.rateAdaptationDegradedThreshold` | num | 0.0–1.0 | 0.5 | linkRate/maxSupported below this ⇒ link flagged as degraded regardless of RSSI. |
| `healthAssessment.excellentMin` | int % | 1–100 | 80 | headroom% ≥ this → `health.rating=EXCELLENT`. |
| `healthAssessment.strongMin` | int % | 1–100 | 55 | [strongMin, excellentMin) → STRONG. |
| `healthAssessment.goodMin` | int % | 1–100 | 30 | [goodMin, strongMin) → GOOD. Below → MARGINAL. |
| `healthAssessment.topTierStretchUpFactor` | num | >1.0 | 1.5 | Cap headroom% at this × top-tier minimum so 4K HDR doesn't read 1000%. |
| `healthAssessment.topTierStretchDownFactor` | num | 0.0–1.0 | 0.66 | Top-tier MARGINAL floor as a fraction of the tier minimum. |

To change phase durations, sample counts, or which servers the binary
picks from, reconfigure the Ookla embed at speedtest.net's dashboard
(account holder action) — those are not surfaced in `CertConfig`.

##### Invariants enforced server-side

`POST /admin/cert-configs` returns 400 if any of these fail:
- All numeric ranges in the live-knobs table above.
- `wifiLinkQuality`: `excellentRssiMin > strongRssiMin > goodRssiMin`.
- `healthAssessment`: `excellentMin > strongMin > goodMin`.

##### Fallback semantics

Any optional section that's missing, malformed, or out-of-range falls
back to bundled defaults **per-field** on the client. A single bad
value can't poison the rest of the config — and `uploadResults`
specifically is parsed first, in isolation, so the kill switch always
survives unrelated failures.

##### Things deliberately NOT tunable (would require an APK release)

- Probe implementations (DNS, Ookla CLI, ExoPlayer playback).
- The `Tier` enum values themselves (`sd | hd | uhd | uhd_hdr | none`).
- Diagnostics fields collected (thermal, CPU, wifi raw).
- `dnsProbeHosts` list (technically optional in the wire format but
  parsing-tolerant; treat as in-code for operational purposes).
- The cert-config schema itself (bumping fields ⇒ contract bump).

### 6.2 `CertificationResult` (body of `POST /v1/certifications`)

```json
{
  "schemaVersion": 1,
  "configVersion": "2026-05-06.1",
  "certificationId": "550e8400-e29b-41d4-a716-446655440000",
  "deviceId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "startedAt": "2026-05-06T14:32:11Z",
  "completedAt": "2026-05-06T14:33:08Z",
  "enqueuedAt": "2026-05-06T14:33:09Z",
  "submittedAt": "2026-05-06T14:33:09Z",

  "device": {
    "model": "FRC1-Hotwire",
    "manufacturer": "ONN",
    "androidVersion": "12",
    "apiLevel": 31,
    "buildFingerprint": "...",
    "appVersion": "0.1.0",
    "appVersionCode": 1,
    "uptimeMs": 432000000
  },

  "identity": {
    "deviceId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "hsn": "E44AW3251919440",
    "hardwareSerial": "E44AW3251919440",
    "ethernetMac": "aa:bb:cc:dd:ee:ff",
    "wifiMac": null
  },

  "network": {
    "transport": "WIFI",
    "vpnActive": false,
    "metered": false,
    "validated": true,
    "linkDownstreamKbps": 300000,
    "linkUpstreamKbps": 300000,
    "privateIp": "192.168.10.189",
    "gatewayIp": "192.168.10.1",
    "dnsServers": ["192.168.10.1", "1.1.1.1"],
    "publicIp": "203.0.113.5",
    "dhcp": {
      "serverAddress": "192.168.10.1",
      "gateway": "192.168.10.1",
      "netmask": "255.255.255.0",
      "ipAddress": "192.168.10.189",
      "leaseSec": 86400,
      "dns1": "192.168.10.1",
      "dns2": "1.1.1.1"
    }
  },

  "capabilities": {
    "drm": {
      "widevineSecurityLevel": "L1",
      "widevineSystemId": "4464",
      "widevineHdcpLevel": "HDCP-V2.2",
      "widevineMaxHdcpLevel": "HDCP-V2.2",
      "widevineVersion": "16.1.0"
    },
    "display": {
      "widthPx": 3840,
      "heightPx": 2160,
      "refreshRateHz": 60.0,
      "densityDpi": 320,
      "supportedModes": [
        { "widthPx": 3840, "heightPx": 2160, "refreshRateHz": 60.0 },
        { "widthPx": 1920, "heightPx": 1080, "refreshRateHz": 60.0 }
      ],
      "hdrTypes": ["HDR10", "DOLBY_VISION", "HLG"]
    },
    "videoCodecs": [
      { "name": "c2.android.hevc.decoder", "mimeType": "video/hevc", "isHardwareAccelerated": true, "maxWidth": 3840, "maxHeight": 2160, "maxFrameRate": 60 },
      { "name": "c2.android.avc.decoder",  "mimeType": "video/avc",  "isHardwareAccelerated": true, "maxWidth": 1920, "maxHeight": 1088, "maxFrameRate": 60 }
    ],
    "audio": {
      "supportedEncodings": ["PCM_16BIT", "AC3", "EAC3", "EAC3_JOC"],
      "hdmiConnected": true
    },
    "thermal": { "status": "NONE" },
    "memory": { "totalMb": 1840, "availableMb": 720, "thresholdMb": 184, "lowMemory": false },
    "storage": { "dataPartitionFreeMb": 4200, "dataPartitionTotalMb": 12000 },
    "locale": { "locale": "en-US", "timezoneId": "America/New_York", "timezoneOffsetMin": -240 },
    "power": { "acPlugged": true, "batteryPresent": false, "batteryLevelPct": null },
    "system": { "developerOptionsEnabled": true, "adbEnabled": true, "isDebuggable": true },
    "wifiSupport": {
      "isWifiEnabled": true,
      "fiveGhzSupported": true,
      "sixGhzSupported": false,
      "sixtyGhzSupported": false,
      "wpa3SaeSupported": true,
      "wpa3SuiteBSupported": false,
      "easyConnectSupported": true,
      "enhancedOpenSupported": true,
      "staApConcurrencySupported": false,
      "multiStaConcurrencySupported": false,
      "tdlsSupported": true
    },
    "bootReason": "reboot",
    "bootTimeEpochMs": 1746230400000
  },

  "wifi": {
    "rssiDbm": -54,
    "signalLevel": 4,
    "linkSpeedMbps": 433,
    "txLinkSpeedMbps": 433,
    "rxLinkSpeedMbps": 433,
    "maxSupportedTxLinkSpeedMbps": 866,
    "maxSupportedRxLinkSpeedMbps": 866,
    "frequencyMhz": 5180,
    "band": "5GHZ",
    "channelWidthMhz": null,
    "standard": "11AC",
    "security": "WPA2_PSK",
    "ssid": null,
    "bssid": null,
    "supplicantState": "COMPLETED",
    "hiddenSsid": false
  },

  "result": {
    "achievedTier": "hd",
    "marginalMetric": "downloadMbps",
    "tierBreakdown": [
      { "tierId": "sd",  "passed": true,  "failedChecks": [] },
      { "tierId": "hd",  "passed": true,  "failedChecks": [] },
      { "tierId": "fhd", "passed": false, "failedChecks": ["minDownloadMbps"] }
    ],
    "health": {
      "headroomPct": 42,
      "rating": "GOOD",
      "limitingMetric": "Download",
      "nextTier": "uhd",
      "perMetric": { "Download": 42, "Latency": 88, "Jitter": 75 }
    },
    "wifiLink": {
      "rating": "STRONG",
      "rssiDbm": -63,
      "band": "5GHZ",
      "linkSpeedMbps": 433,
      "maxSupportedMbps": 866,
      "rateAdaptationDegraded": false,
      "advice": "−63 dBm on 5 GHz, 433/866 Mbps. Healthy link with margin."
    }
  },

  "metrics": {
    "selectedServer": { "id": "dfw", "rttMs": 56 },
    "serverProbes": [
      { "id": "mia", "rttMs": 38, "selected": false },
      { "id": "dfw", "rttMs": 56, "selected": true  }
    ],
    "download": { "steadyMbps": 371.2, "peakMbps": 395.1, "durationSec": 10, "samples": [120.1, 285.4, 360.0] },
    "upload":   { "steadyMbps": 184.5, "peakMbps": 210.0, "durationSec": 5,  "samples": [88.0, 180.5, 195.0] },
    "latency":  { "medianMs": 12, "p95Ms": 18, "jitterMs": 3, "lossPct": 0.0, "samples": 20 },
    "dns":      {
      "medianMs": 14,
      "maxMs": 38,
      "failureCount": 0,
      "samples": [
        { "host": "gethotwired.com", "resolveMs": 12, "success": true,  "resolvedIps": ["..."] },
        { "host": "google.com",      "resolveMs": 9,  "success": true,  "resolvedIps": ["..."] },
        { "host": "netflix.com",     "resolveMs": 38, "success": true,  "resolvedIps": ["..."] },
        { "host": "youtube.com",     "resolveMs": 14, "success": true,  "resolvedIps": ["..."] },
        { "host": "akamaized.net",   "resolveMs": 22, "success": true,  "resolvedIps": ["..."] },
        { "host": "cloudflare.com",  "resolveMs": 11, "success": true,  "resolvedIps": ["..."] }
      ]
    },
    "playback": { "peakHeight": 1080, "peakBitrateKbps": 6000, "rebufferCount": 0, "startupMs": 1240, "durationSec": 20 }
  }
}
```

**Field rules:**

- `certificationId` is the idempotency key — required, UUID v4.
- `deviceId` (top-level *and* duplicated inside `identity`) is stable per
  install — generated client-side as a UUID, persisted in app private storage.
  This is the analytics primary key and survives factory reset of the app's
  data only if the STB image preserves it; otherwise a new UUID is minted.
- `identity.hsn` is read from the system property `ro.product.hsnt` on HWC
  STB images. May be `null` on devices that don't expose it (e.g. emulators
  or non-HWC hardware used for testing).
- `identity.hardwareSerial` is read from `ro.serialno` (or `Build.getSerial()`
  if available). Often equal to `hsn` on HWC hardware; expose both because
  that equality is not contractual.
- `identity.ethernetMac` and `identity.wifiMac` are read from
  `/sys/class/net/{eth0,wlan0}/address`. **Both are usually `null` unless
  the app is signed into the system image** — Android blocks MAC reads from
  ordinary apps. Wi-Fi MAC is additionally randomized per-network from
  Android 10+, so even when accessible it may not match the hardware burn-in.
- `wifi` block is `null` when `network.transport != "WIFI"` (Ethernet STBs are
  common; don't fabricate Wi-Fi fields).
- `network.transport` is one of `WIFI | ETHERNET | CELLULAR | VPN | OTHER`.
- `wifi.band` is one of `2_4GHZ | 5GHZ | 6GHZ | UNKNOWN`.
- `wifi.standard` is one of `11A | 11B | 11G | 11N | 11AC | 11AX | 11BE | UNKNOWN`.
- `wifi.security` is one of `OPEN | WEP | WPA | WPA2_PSK | WPA2_EAP | WPA3 | UNKNOWN`.
- `result.achievedTier` matches a `tiers[].id` from the config the run used,
  or the literal string `"none"` if no tier passed.
- `result.marginalMetric` names the *single* metric that prevented the next-up
  tier; useful for support triage. `null` if achievedTier is the top tier.
- `metrics.*.samples` are raw per-sample values for debugging. Server stores
  them as JSONB; do not index. Cap array length at 200 server-side.

**`capabilities` block — what each subfield means and why it matters:**

- `drm.widevineSecurityLevel` — `L1` / `L2` / `L3`. **Premium 4K studio
  content (Netflix UHD, Disney+ 4K) requires L1.** A device reporting `L3`
  cannot stream 4K regardless of bandwidth. This is the single highest-value
  capability for explaining "your network passed but streaming fails."
- `drm.widevineHdcpLevel` / `widevineMaxHdcpLevel` — current and max HDCP
  versions on the HDMI link. HDCP 2.2+ required for premium 4K.
- `display.hdrTypes` — empty array means no HDR support; 4K HDR tier is
  unreachable regardless of network.
- `display.supportedModes` — if no `2160p` mode at 60 Hz, 4K tier unreachable.
- `videoCodecs` — list of *decoders* available. Filter to mime
  `video/hevc`, `video/av01`, `video/x-vnd.on2.vp9` and check
  `maxWidth >= 3840` to confirm hardware 4K decode for the relevant codec.
  `isHardwareAccelerated = false` is a strong signal the device cannot
  sustain 4K decode in real time.
- `audio.supportedEncodings` — passthrough capabilities of the connected
  HDMI sink. `EAC3_JOC` indicates Atmos passthrough.
- `thermal.status` — `LIGHT`/`MODERATE`/`SEVERE`/`CRITICAL` indicates the
  device is actively throttling and explains anomalously low throughput on
  an otherwise-good link.
- `memory.lowMemory = true` or `availableMb < 200` correlates with rebuffer
  events on tight-budget STBs.
- `bootReason` — why the device last rebooted (`reboot`, `kernel_panic`,
  `power_off`, etc.). Frequent unexpected reasons across the fleet flags
  unstable hardware/firmware.
- `bootTimeEpochMs` — absolute wall-clock timestamp of the last boot.
  Useful in support tickets ("this STB has been up since X").

### 6.4 `AppVersionManifest` (response of `GET /v1/app/version`)

```json
{
  "schemaVersion": 1,
  "latestVersionName": "0.7.1",
  "latestVersionCode": 71,
  "minRequiredVersionCode": 68,
  "apkUrl": "https://certifier-api.gethotwired.com/v1/app/download/0.7.1.apk",
  "apkSizeBytes": 12345678,
  "apkSha256": "ab12cd34ef560000000000000000000000000000000000000000000000000000",
  "signingCertSha256": "ff009911000000000000000000000000000000000000000000000000000000ff",
  "releaseNotes": "Adjusted 4K-HDR tier thresholds; added ATL-3 server.",
  "publishedAt": "2026-05-09T18:00:00Z"
}
```

**Field notes:**

- `latestVersionCode` — monotonic. Each release bumps it by ≥1. The client
  treats `installed >= latestVersionCode` as "no update available".
- `minRequiredVersionCode` — the floor. Must satisfy
  `1 <= minRequiredVersionCode <= latestVersionCode`. Use this to enforce
  a cert-blocking update when the installed app has bugs that produce
  incorrect measurements.
- `apkUrl` — absolute. HTTPS in production. The client downloads with no
  auth header — integrity is enforced via the SHA-256 + signing-cert
  pinning, not via transport auth.
- `apkSizeBytes` — exact size. The client refuses to install if the
  downloaded length differs.
- `apkSha256` — 64-char lowercase hex SHA-256 of the APK contents.
- `signingCertSha256` — 64-char lowercase hex SHA-256 of the APK's
  signing certificate. Must equal the actual certificate of the
  downloaded APK AND match `BuildConfig.APP_SIGNING_CERT_SHA256` in
  the client build.
- `releaseNotes` — optional, shown verbatim in the "Update available"
  banner. Keep short (≤120 chars renders cleanly on a TV).
- `publishedAt` — informational only.

---

## 7. Storage shape (Postgres reference)

Two tables is enough.

```sql
create table cert_config (
  config_version    text primary key,
  schema_version    int  not null,
  document          jsonb not null,           -- the full CertConfig
  is_active         bool not null default false,
  created_at        timestamptz not null default now()
);
-- exactly one row has is_active = true at a time; enforce via partial unique index.
create unique index cert_config_active on cert_config ((true)) where is_active;

create table certifications (
  certification_id  uuid primary key,
  device_id         uuid not null,
  hsn               text,                     -- HWC serial number, may be null
  hardware_serial   text,
  ethernet_mac      text,
  schema_version    int  not null,
  config_version    text references cert_config(config_version),
  started_at        timestamptz not null,
  completed_at      timestamptz not null,
  achieved_tier     text not null,
  marginal_metric   text,
  transport         text not null,
  -- Capability hot-path fields for fleet queries:
  widevine_level    text,                     -- L1/L2/L3
  hdr_types         text[],                   -- e.g. {HDR10,DOLBY_VISION}
  display_max_height int,
  thermal_status    text,
  -- Denormalized hot-path fields for cheap queries:
  download_steady_mbps  numeric,
  upload_steady_mbps    numeric,
  latency_median_ms     int,
  -- Full payload for everything else:
  payload           jsonb not null,
  payload_hash      text not null,
  received_at       timestamptz not null default now()
);
create index on certifications (device_id, completed_at desc);
create index on certifications (hsn, completed_at desc);
create index on certifications (achieved_tier, completed_at desc);
```

**Timestamp semantics.** Four payload timestamps describe a single run:

| Field | Set by | Meaning |
|---|---|---|
| `startedAt` | client (cert engine) | When the certification began on the STB. |
| `completedAt` | client (cert engine) | When the certification finished on the STB. **This is the moment the cert measured the network — store cert records keyed off this, not `received_at`.** |
| `enqueuedAt` | client (publish queue) | When the result first entered the local publish queue. Usually within 1 s of `completedAt`. |
| `submittedAt` | client (publish queue) | When *this* POST attempt was made. Refreshed on every retry. |

`received_at` (server-only) is the wall-clock moment the row arrived at
the API. For runs that POSTed on the first try, all four client-side
timestamps land within a couple of seconds of each other. For runs that
queued because the API was unavailable, the gap between `completedAt`
and `submittedAt` can be hours or days — that gap is the queue-delay
metric.

**Do not** use `received_at` as the certification time. A run from
yesterday that finally drained today would otherwise be indistinguishable
from a fresh run, breaking trend queries.

`GET /v1/cert-config` reads the active row. To roll out a new config: insert
a new row, then in a transaction flip `is_active`. Old configs stay in the
table for audit / config-version round-tripping.

---

## 8. Test cases the implementation must pass

1. **Config GET, happy path.** Returns 200 with a valid `CertConfig`.
2. **Config GET, app schema too old.** Server's active config has
   `schemaVersion = 2`; client sends `X-Schema-Version: 1`. Server returns the
   newest config the client *can* understand, or 426 if none exists.
3. **Result POST, first time.** Returns 201, row inserted.
4. **Result POST, exact duplicate.** Same `certificationId`, byte-identical
   body. Returns 200, no second row.
5. **Result POST, conflicting duplicate.** Same `certificationId`, different
   `payload_hash`. Returns 409, no row mutated.
6. **Result POST, wifi block null on Ethernet.** Validates fine.
7. **Result POST, payload over 256 KB.** Returns 413.
8. **Result GET by id.** Returns the stored record.
9. **Auth missing/bad.** All three endpoints return 401.

---

## 9. What is explicitly out of scope here

- A web dashboard / UI — not this team's deliverable.
- Aggregations or rollups across STBs — downstream BI job.
- Alerting on degraded servers — separate concern; we may build it on top of
  this dataset later.
- Push from server → app (the app polls config; that's enough for v1).

---

## 10. Open questions to resolve before merging

These need decisions from HWC, not from the backend implementer:

- **Auth approach** (Option A vs B in § 5).
- **PII policy** for `ssid`, `bssid`, `publicIp`, `gatewayIp` — store, hash,
  or drop?
- **Retention** — how long do we keep certification records? Default to 18
  months unless told otherwise.
- **Hostname** — what domain will this live under? (`certifier-api.gethotwired.com`?)

When in doubt, build the smallest correct thing for the schema in § 6, leave
the questions above marked `TODO` in code, and ask before guessing.
