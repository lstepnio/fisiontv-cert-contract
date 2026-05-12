# Changelog

All notable changes to the FisionTV+ Network Certifier API contract.

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

A **major** bump means clients pinned to the previous major must be updated
before they can produce or consume payloads against the new version.
**Minor** bumps add fields backward-compatibly (clients ignore unknown
fields). **Patch** bumps clarify the spec without changing payload shape.

## [Unreleased]

## [2.2.0] — 2026-05-12

### Added

- **Per-device cert-config targeting.** Operators can now ship
  different configs to different STB hardware without app changes.
  Three optional selector fields land on `CertConfig`:
  - `targetManufacturer` (Android `Build.MANUFACTURER`)
  - `targetModel` (Android `Build.MODEL`)
  - `targetBuildFingerprint` (Android `Build.FINGERPRINT`)

  Each nullable; an all-null row is the default.

- **Three new optional request headers on `GET /v1/cert-config`:**
  `X-Device-Manufacturer`, `X-Device-Model`, `X-Device-Build-Fingerprint`.
  Pre-v2.2.0 clients send none of these and resolve to the default,
  preserving existing behavior under a v2.2.0 server.

- **Resolution algorithm** (§ 4.1.1): the server picks the row whose
  selectors most-specifically match the request — build-fingerprint
  match wins over model, model wins over manufacturer, manufacturer
  wins over default. Deterministic; no priority list.

- **Per-resolved-config `ETag`.** The `ETag` returned now varies by
  the resolved config (`sha256(resolved-bytes)`), not by a global
  active-row identifier. This keeps the 304 fast-path correct when
  different devices resolve to different rows.

### Migration

Schema additions are fully optional. Pre-2.2.0 configs already in the
DB act as the default. Pre-2.2.0 clients send no targeting headers
and continue to receive the default config. No coordinated rollout
required; backend, client, and admin tooling can be bumped
independently.

## [2.1.1] — 2026-05-12

### Changed (clarification only — schema + payload shape unchanged)

- SPEC §4.4 + openapi.yaml `getAppVersion` description now describe the
  client's actual behavior since v0.8.7: auto-update runs in the
  background, **never blocks the cert**. The previous three-state
  "Run/Update banner/Update & run" gating model was replaced by a
  client-side decision (see `MainViewModel.preCertUpdate`) and the
  spec had drifted from reality.

- `minRequiredVersionCode` description reframed from "cert-blocking
  floor" to **advisory floor** — the version the server would prefer
  results to come from. The client never gates the cert on this; the
  server is the right place to flag/dim/discard results below the
  floor.

- `releaseNotes` description marked as reserved/future-use. The
  current client auto-updates silently and surfaces nothing in any
  UI. Field is still accepted; consumers should not rely on it being
  rendered.

- `Response 426` paragraph: the client logs the response and still
  runs the cert; 426 is a signal to escalate auto-update urgency, not
  a cert-blocking gate.

### Migration

No payload-shape change. Backend implementations and clients pinned
to v2.1.0 continue to interop unchanged with v2.1.1 clients/servers.
This is a documentation-accuracy patch.

## [2.1.0] — 2026-05-12

### Added

- `CertConfig.killswitch` — true killswitch. When `enabled: true`, the
  Android client exits cleanly on receipt. Recovery is automatic on
  next app launch (fetches active config, proceeds if killswitch flipped
  off, exits again otherwise). No UI surface, no shared-prefs cache,
  no in-process polling — kept deliberately minimal.

  Distinct from `uploadResults.enabled=false`, which only suppresses
  publishing; under that flag certs still run locally.

  Two fields: `enabled` (required boolean) and `reason` (optional
  string, informational only — round-tripped into logcat at exit time
  so techs can correlate).

### Changed

- §6.1 prose clarifies `uploadResults.enabled` is *not* a hard stop —
  it suppresses POSTs but certs still run.
- §6.1.1 knobs table gains the `killswitch.*` rows and a new
  "Killswitch behavior" subsection.

## [2.0.0] — 2026-05-12

### Removed (BREAKING)

Nine cert-config fields removed entirely from the schema. v1.4.0
marked them `deprecated: true`; v2.0.0 drops them outright. None of
them were ever consumed by the Ookla code path:

- `CertConfig.servers[]` (the field) and the `OoklaServer` schema is
  no longer referenced by `CertConfig`. The `OoklaServer` schema
  itself is retained because `CertificationResult.selectedServer`
  still uses it to describe the server Ookla *picked* per run.
- `tests.download.{durationSec,perRequestBytes,warmupFraction}`
- `tests.upload.{durationSec,perRequestBytes,warmupFraction}`
- `tests.latency` section (and the `LatencyPhaseConfig` schema)

Phase durations and ping counts live in the Ookla embed-config and
must be adjusted at speedtest.net's account dashboard, not via this
contract.

### Migration

- Clients (Android `RuntimeConfigParser`) silently ignore unknown
  JSON keys, so a v2.0.0-pinned client still parses a v1.x cert-config
  document. No client-side migration needed.
- Backend stops validating these fields (the per-field range checks
  are removed). Existing pre-2.0.0 configs already in the DB are
  served unchanged.
- Anyone tooling against the OpenAPI schema directly will need to
  regenerate.

## [1.4.0] — 2026-05-12

### Deprecated

Nine cert-config fields are now marked `deprecated: true` and no
longer in `required:` lists. They remain in the schema so configs
created before v1.4.0 still deserialize, but new configs should omit
them entirely. The Android client and backend ignore them.

- `servers[]` (and the `OoklaServer` schema) — server selection is
  delegated to the Ookla embed-config response; the cert-config list
  was never passed to the binary.
- `tests.download.durationSec`, `tests.download.perRequestBytes`,
  `tests.download.warmupFraction` — phase length, chunk sizing, and
  IQM warmup are all determined by the Ookla embed-config response
  (`suite.testStage.download.testDurationSeconds` etc.) or by the
  binary's internal logic.
- `tests.upload.{durationSec,perRequestBytes,warmupFraction}` — same.
- `tests.latency` (the whole section, including `samples` and
  `timeoutMs`) — embed-config's `suite.testStage.latency.pingCount`
  controls ping count; per-ping timeout is internal to the binary.

To actually change phase durations or sample counts, reconfigure the
Ookla embed at speedtest.net's dashboard (account holder action) — no
code change required.

### Changed

- SPEC §6.1.1 split into "Live knobs" and "Deprecated" tables so an
  operator can see at a glance what's actually consumed.
- SPEC §6.1 example and `fixtures/cert-config.example.json` no longer
  include the dead fields.

## [1.3.0] — 2026-05-12

### Added

- `CertConfig.wifiLinkQuality` (optional) — operator-tunable RSSI bands +
  rate-adaptation cutoff that produce the STRONG / GOOD / MARGINAL / WEAK
  WiFi badge. Four fields: `excellentRssiMin`, `strongRssiMin`,
  `goodRssiMin`, `rateAdaptationDegradedThreshold`.
- `CertConfig.healthAssessment` (optional) — operator-tunable headroom
  buckets + top-tier stretch factors that produce the
  EXCELLENT / STRONG / GOOD / MARGINAL `health.rating` field. Five
  fields: `excellentMin`, `strongMin`, `goodMin`,
  `topTierStretchUpFactor`, `topTierStretchDownFactor`.
- SPEC §6.1.1 "Server-tunable knobs reference" — a single table listing
  every cert-config field with type, range, default, and operational
  effect. Distinguishes server-tunable from in-code (e.g. probe
  implementations, the Tier enum). Includes the invariants the backend
  enforces (`excellentRssiMin > strongRssiMin > goodRssiMin`, etc.).

### Notes

- Both new sections are **additive and optional**. Clients on the
  bundled defaults keep working unchanged. Per-field parser tolerance:
  a missing or out-of-range value falls back to the bundled default
  rather than rejecting the whole config.
- No schemaVersion bump (still 1) — the change is fully backward-compat.

## [1.2.1] — 2026-05-12

### Fixed

- Internal OpenAPI consistency: declared `openapi: 3.0.3` (the document
  was using 3.0-only `nullable: true`), rewrote 4 `oneOf:[$ref, null]`
  blocks to `allOf + nullable`, added explicit `type: object` for the
  nullable-type-sibling lint rule.

## [1.2.0] — 2026-05-11

### Added

- `GET /v1/app/version` — the latest-app manifest the client polls on
  every launch. Carries the latest `versionCode` + `versionName`, the
  `minRequiredVersionCode` floor below which cert runs are blocked,
  the absolute `apkUrl`, and integrity pins (`apkSha256` and
  `signingCertSha256`). Supports 304 via ETag and a 426 hard-floor
  response. SPEC.md grows a corresponding "Self-update flow" section.
- `AppVersionManifest` schema + `X-App-Version-Code` header parameter.

Clients that don't call the new endpoint continue to work unchanged
(it's an addition, not a behavior change to any existing path).

## [1.1.0] — 2026-05-11

### Added

- Two optional POST-time timestamps on `CertificationResult`:
  - `enqueuedAt` — when the row first entered the client publish queue
  - `submittedAt` — when this specific POST attempt left the device
- "Timestamp semantics" guidance in `SPEC.md` clarifying that backends
  should key storage off `completedAt` (cert run time) rather than
  `received_at` (request arrival), so reports remain accurate when a
  payload sat queued for hours or days before draining.

Both fields are optional; clients pinned to 1.0.x continue to produce
valid payloads without them. Servers must accept payloads with or
without the new fields.

## [1.0.0] — 2026-05-06

### Added

- Initial spec covering `GET /v1/cert-config`, `POST /v1/certifications`,
  and `GET /v1/certifications/{id}`.
- Schemas for runtime config (servers, test phases, tier thresholds) and
  for the full certification result payload (device / identity /
  capabilities / network / wifi / result / metrics blocks).
- Bearer auth scheme; `X-Device-Id`, `X-App-Version`, `X-Schema-Version`
  required headers across all endpoints.
- Example fixtures under `fixtures/` for both the config response and a
  representative certification POST body.
