# Changelog

All notable changes to the FisionTV+ Network Certifier API contract.

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/);
versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

A **major** bump means clients pinned to the previous major must be updated
before they can produce or consume payloads against the new version.
**Minor** bumps add fields backward-compatibly (clients ignore unknown
fields). **Patch** bumps clarify the spec without changing payload shape.

## [Unreleased]

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
