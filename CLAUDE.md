# CLAUDE.md — fisiontv-cert-contract

## Role

The **source of truth** for the FisionTV+ Network Certifier wire format. Two artifacts:

- **`openapi.yaml`** — OpenAPI 3.1 spec. Authoritative shape of every request/response on `/v1/*`. Consumers vendor this and validate against it at runtime (the backend via `kin-openapi`).
- **`SPEC.md`** — narrative companion. Endpoint-by-endpoint behavior, gating rules, error semantics, integrity model, threading/timing constraints, retry policies. Read it when openapi alone doesn't pin down behavior.

This repo ships **only spec**. No code, no Docker images, no CI besides spec validation (`spectral` lint).

## Neighbors

This repo is the contract for one logical system. It's vendored as a git submodule into two consumers, both checked out under `/Users/lukasz.stepniowski/Development/`:

- **backend** — `NetworkQualityHWCBackend` — implements every endpoint defined here. Validates incoming requests against `openapi.yaml` at runtime.
- **android** — `NetworkQualityHWC` — Kotlin STB client. Generates no code from openapi (parsers are hand-written in `update/AppVersionManifestParser.kt`, `cert/CertificationPayload.kt`, etc.) but matches the schemas line-by-line.
- **dashboard** — `NetworkQualityHWCDashboard` — does not consume this contract directly; reads `/admin/*` from the backend.

**Pin discipline:** backend and android pin their `contract/` submodule to an **exact tag** (no `-N-gSHA` suffix). The two pins must match. Both repos' `build.yml` runs `git -C contract describe --tags --exact-match` as a CI gate.

## Conventions

- **PR-only.** Sandbox blocks direct push to `main`. Workflow: `git checkout -b spec/<short-name>` → commit → push → `gh pr create` → wait for CI green → `gh pr merge --merge --delete-branch` → fast-forward main → tag if releasing.
- **Conventional Commits.** `spec:` for spec changes (schema or narrative), `release:` for the commit that bumps openapi `info.version`, `chore:` for tooling/CI/docs not in `SPEC.md`.
- **Tag = `vMAJOR.MINOR.PATCH` matching openapi `info.version`.** Bump the openapi version in the same commit (or PR) that tags. Never tag a SHA that has a stale `info.version`.

## When to bump

- **PATCH** (`v1.2.1`): narrative clarifications in `SPEC.md`, openapi description-only edits, fixing examples. No client/server code change required.
- **MINOR** (`v1.3.0`): new endpoint, new optional field, new error code. Existing consumers keep working without updates; consumers can opt in.
- **MAJOR** (`v2.0.0`): breaking change (removed endpoint, required field added, response shape change). Avoid; sequence with coordinated client + server cuts.

## Release flow

1. Branch + spec change.
2. Bump `info.version` in `openapi.yaml`.
3. PR + CI green.
4. Merge.
5. `git tag -a vX.Y.Z -m "release: X.Y.Z — <one-line>" HEAD && git push origin vX.Y.Z`.
6. Open paired submodule-bump PRs in **both consumers** (backend + android) updating `contract/` to the new tag.

Don't tag a feature branch HEAD before it merges — the tag will then sit on a SHA that isn't reachable from `main`, which is the state v1.2.1 was briefly in on 2026-05-11 (recoverable: the SHA is still fetched directly by tag ref, but it's a smell).

## Operational notes

- **`spectral` lint is strict-warn.** Even a `Warning` from `info-license-strict` (or similar) fails CI. Look at the workflow at `.github/workflows/validate.yml` before adding new spec entries that might trip a built-in rule.
- **No client code generation.** Both consumers parse JSON by hand. If a schema changes, the parsers in `internal/api/*.go` (backend) and `*/cert/CertificationPayload.kt` + `update/AppVersionManifestParser.kt` (android) must follow.
- **PII fields are marked in `SPEC.md` §5 (Privacy).** When adding a new field that might contain PII (MAC, IP, SSID, etc.), update the privacy section and add the field to the backend's `internal/pii/hash.go` `piiPaths` list in the matching PR.
