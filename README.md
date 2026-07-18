# proto-registry — shared gRPC contracts

Single source of truth for all east–west (service-to-service) proto contracts
for the Research Ambit services. Every microservice call goes through Envoy
(`envoy:10000`) using these contracts; no ad-hoc REST between services.

Migrated out of the `vm-infra` monorepo's `protos/` directory, which is kept
as a frozen reference copy until all consumers below have cut over (see that
repo's `protos/README.md` for migration status).

| Package | RPCs | Server | Callers |
|---------|------|--------|---------|
| `auth.v1` | `VerifyToken`, `GetUser` | auth-service (`:50051`) | api-gateway |
| `embedding.v1` | `Embed`, `Rerank` | embedding (`:50052`) | search-api, chatbot-agent |
| `search.v1` | `FacultyForQuery` | search-api (`:50053`) | chatbot-agent |
| `chat.v1` | `CheckQuota` | chatbot-agent (reserved) | api-gateway |
| `directory.v1` | directory/KG/content/user/suggestions | research-ambit-main backend (`:50055`) | api-gateway |

## Tooling

- `buf lint` — style/consistency checks (`buf.yaml`, `BASIC` ruleset — the
  stricter `DEFAULT` category flags RPC request/response naming conventions
  that predate this repo; renaming those message types would break every
  existing consumer's generated code, so it's intentionally not enabled).
- `buf breaking --against '.git#branch=main'` — fails on any wire-incompatible
  change to an existing package. Requires no `buf.build` account; runs purely
  against local git history.
- Python stubs are generated with `grpc_tools.protoc` directly (not `buf
  generate` — `grpc_tools.protoc` bundles python+grpc codegen in one
  self-contained invocation, not the separable-plugin model `buf generate`
  expects). CI runs this on tagged releases only, committing output to a
  `release/vX.Y.Z` branch (see `.github/workflows/ci.yml`) so the immutable
  tag itself only ever contains source `.proto` files.

## Consumption

This repo is **public**. That makes plain `git+https` installs (Python)
anonymous — no credentials needed. It does NOT make the npm side
credential-free: GitHub Packages requires authentication for every install,
public or private, so Node consumers still need a `GITHUB_TOKEN`/PAT with at
least `read:packages` even though the source repo itself is open.

- **Node** (api-gateway, auth-service, research-ambit-main, search-api):
  install the raw `.proto` tree as `@iitd-tech-ambit/protos` from GitHub
  Packages, load at runtime with `@grpc/proto-loader` + `@grpc/grpc-js` — no
  codegen step, same model as before migration. Still needs an auth token
  for `npm install` (see above).
- **Python** (embedding, chatbot-agent): pin
  `git+https://github.com/IITD-Tech-Ambit/proto-registry.git@release/vX.Y.Z#subdirectory=gen-python`
  in `requirements.txt`/`pyproject.toml`, then import as
  `from iitd_ambit_protos.embedding.v1 import embedding_pb2` (etc.) — stubs
  are nested under the `iitd_ambit_protos` top-level package deliberately,
  since a consumer service named after its own domain (an `embedding`
  service, a `search-api`) would otherwise collide with and silently shadow
  a top-level package of the same name. Tags themselves stay immutable
  (source only); each tag's generated stubs live on the matching
  `release/vX.Y.Z` branch, built with `grpcio-tools==1.62.3` deliberately, so
  output stays importable on both older (`protobuf>=4.25.0`) and newer
  (`grpcio>=1.66.0`) consumer pins.

## Rules

- Packages are versioned (`*.v1`). Breaking changes require a new version
  package (e.g. `directory.v2`), never in-place field renumbering — enforced
  by `buf breaking` in CI.
- Fields use proto3 defaults; `0`/empty string mean "unset" where noted in
  comments.
- Releases are semver git tags on `main` (`vX.Y.Z`): patch for non-wire
  changes, minor for additive changes, major reserved for anything
  `buf breaking` would otherwise flag (should be rare — prefer a new version
  package instead).
