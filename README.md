# fisiontv-cert-contract

The API contract between the FisionTV+ Network Certifier Android app and
the backend that ingests its results. **Source of truth** for both
projects: any payload-shape change goes here first, then propagates to the
client and server via submodule bumps.

## Files

- **`openapi.yaml`** — OpenAPI 3.1 spec. Defines every endpoint, request,
  response, and schema. Both client and server validate against this.
- **`SPEC.md`** — Human-readable narrative: why each field exists, what
  the policy decisions were, what's deliberately deferred. Read this
  before changing the YAML.
- **`fixtures/`** — Example JSON payloads. CI validates them against the
  OpenAPI schemas to catch drift.
- **`CHANGELOG.md`** — What changed in each tagged release.

## Versioning

Tagged with [SemVer](https://semver.org):

- **MAJOR** — payload shape change clients must update for. Coordinate
  client + server roll-out.
- **MINOR** — field additions; clients ignore unknown fields, so old
  clients keep working against a new spec.
- **PATCH** — wording / clarifications / fixture updates without payload
  changes.

Both consumer projects pin to a tag via git submodule:

```bash
git submodule add https://github.com/lstepnio/fisiontv-cert-contract.git contract
cd contract && git checkout v1.0.0
```

Bump the pin by editing the submodule pointer:

```bash
cd contract && git fetch && git checkout v1.1.0 && cd ..
git add contract && git commit -m "chore: bump contract to v1.1.0"
```

## Validating the spec

```bash
# Lint the OpenAPI spec
npx --yes @redocly/cli@latest lint openapi.yaml

# Validate fixtures against their schemas
npx --yes ajv-cli validate \
  -s openapi.yaml \
  -d fixtures/cert-config.example.json \
  --spec=draft2020 --strict=false
```

CI (`.github/workflows/validate.yml`) runs both on every PR.

## Generating clients

This repo intentionally does not publish generated SDKs; both consumer
projects validate hand-written types against the spec in CI rather than
codegen them. If you need a quick scratch client (curl-style):

```bash
npx --yes @openapitools/openapi-generator-cli generate \
  -i openapi.yaml -g python -o /tmp/fisiontv-py
```

## Contributing

1. Edit `openapi.yaml`. Update `SPEC.md` if intent or policy changed.
2. Update / add `fixtures/` so CI validates the new shape.
3. Add a `## [Unreleased]` entry in `CHANGELOG.md` describing the change.
4. PR. CI must pass (lint + fixture validation).
5. After merge, tag the new version and update CHANGELOG.

## License

Proprietary — Hotwire Communications. See `LICENSE`.
