# bifrost-api

**The frozen OpenAPI contract for [Bifrost](https://github.com/brandonrc/bifrost),
plus generated client SDKs. Generated — do not hand-edit.**

Bifrost is a Go port of the mobula control plane. Its API surface starts as a
one-time freeze of mobula's spec rather than an ongoing bot-pushed feed —
`openapi.json` in this repo is a point-in-time snapshot, not a live mirror.
Once Bifrost's Go server (`github.com/brandonrc/bifrost`) has its own
generator producing this file, this repo's role switches to hosting that
generated artifact and its SDKs, the same pattern mobula-api uses for mobula.

## Freeze provenance

- **Source:** `mobula@84d12bb` (branch `fix/openapi-complete-registry`),
  `openapi.json`, OpenAPI 3.1.0, emitted by utoipa v5.
- **Verified complete:** 47 operations across 36 paths — no operations
  silently dropped (see `docs/adr/0002-openapi-codegen-result.md` in the
  `bifrost` repo for the independent oapi-codegen spot-check that confirms
  the same counts against the strict-server interface).
- **Normalization:** the frozen file was run through `jq -S .` once for
  stable, deterministic key ordering:

  ```
  jq -S . < mobula/openapi.json > openapi.json
  ```

- **YAML companion:** `openapi.yaml` is the same spec re-serialized from the
  frozen JSON, for tools that prefer YAML. Generated with:

  ```
  ruby -rjson -ryaml -e 'File.write("openapi.yaml", JSON.parse(File.read("openapi.json")).to_yaml)'
  ```

  (Ruby's stdlib `json`/`yaml` were used in place of PyYAML, which wasn't
  available in the environment this freeze was produced in; any JSON→YAML
  conversion that preserves key order and doesn't rewrite values is
  equivalent.)

## Go server generation

Bifrost's Go server generates directly from this frozen file with
oapi-codegen v2.8.0 (confirmed in `bifrost`'s ADR-0002 — direct OpenAPI 3.1
generation works with no 3.0 downgrade fallback needed):

```
go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest \
  -generate types,std-http,strict-server -package api openapi.json
```

## Layout

```
openapi.json         # the frozen spec (authoritative artifact)
openapi.yaml          # same spec, YAML — for tools that prefer it
.spectral.yaml        # Spectral lint ruleset
redocly.yaml          # Redocly lint/docs config
sdk/
  typescript/         # config.yaml + templates/  (openapi-generator: typescript-fetch)
  rust/               # config.yaml + templates/  (openapi-generator: rust/reqwest)
  python/              # config.yaml + templates/  (openapi-generator: python)
```

Each `sdk/<lang>/` is **config-only**: openapi-generator emits the whole
package (including package.json/Cargo.toml/pyproject) from `config.yaml`;
drop `.mustache` files in `templates/` to override specific generated files.
Nothing generated is committed.

This layout, plus the pipeline files (`redocly.yaml`, `.spectral.yaml`,
`sdk/`, `.github/workflows/`), was bootstrapped from
[`mobula-api`](https://github.com/brandonrc/mobula-api) and renamed
`mobula` → `bifrost` throughout (package names, workflow env, the
`redocly.yaml` API id).

## SDK packages

| Language   | Package                     | Generator config              |
|------------|------------------------------|--------------------------------|
| TypeScript | `@brandonrc/bifrost-client`  | `sdk/typescript/config.yaml`   |
| Python     | `bifrost_client`             | `sdk/python/config.yaml`       |
| Rust       | `bifrost-client`              | `sdk/rust/config.yaml`         |

SDKs are generated in CI (`.github/workflows/generate.yml`) with
**openapi-generator v7.12.0** (Mustache templates, one config per language
under `sdk/<lang>/`), on every push to `main` that touches `openapi.json`,
`sdk/**`, or the workflow itself. Dev versioning is loose: SDKs publish
`0.1.<run_number>` and consumers track `latest`.

- TypeScript publishes to GitHub Packages npm (`@brandonrc` scope) on every
  qualifying `main` push.
- Rust and Python build and validate (`cargo build`, `python -m build`) on
  every run, but only *publish* (to crates.io / PyPI) when
  `CARGO_REGISTRY_TOKEN` / `PYPI_API_TOKEN` are set. On a `main`-branch push,
  a missing publish secret is a **hard failure** (`exit 1`), not a silent
  skip — a `main` push is expected to be publishable, so a missing secret
  there indicates a misconfigured repo rather than an intentional dry run.
  Non-`main` runs (e.g. `workflow_dispatch` off a branch) still build without
  publishing.
- Each SDK build also produces a CycloneDX SBOM
  (`anchore/sbom-action@v0`) as a workflow artifact.

## Pipelines

- **`validate.yml`** — Spectral + Redocly lint the spec on every push/PR.
- **`generate.yml`** — on spec/SDK change (or manual dispatch), regenerate
  each SDK, produce its SBOM, and publish where secrets allow.

## TODO

- Swap `bifrost-ui` from whatever client it uses today to
  `@brandonrc/bifrost-client` once the first SDK publish lands.
- Once Bifrost's Go server generates its own `openapi.json` (rather than
  consuming this one-time freeze of mobula's), wire that generation into
  this repo's CI the way `mobula-api` consumes mobula's bot-pushed spec.
