# Changelog

All notable changes to the FisionTV+ Network Certifier API contract.

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

A **major** bump means clients pinned to the previous major must be updated
before they can produce or consume payloads against the new version.
**Minor** bumps add fields backward-compatibly (clients ignore unknown
fields). **Patch** bumps clarify the spec without changing payload shape.

## [Unreleased]

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
