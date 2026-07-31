# proto-registry — shared gRPC contracts (mirror)

Single place to see the current `.proto` contracts for all Research Ambit
services. This repo holds no logic and does no releases/versioning/lint —
it's a plain mirror, kept in sync automatically by CI in each owning
service's repo.

| Package | Owned by (source of truth) | Consumers |
|---------|------|--------|
| `auth.v1` | `auth-service` | `api-gateway` |
| `chat.v1` | `chatbot-agent` | `api-gateway` |
| `directory.v1` | `research-ambit-main` | `api-gateway` |
| `embedding.v1` | `SEO-Backend-iitd/services/embedding` | `search-api`, `chatbot-agent` |
| `search.v1` | `SEO-Backend-iitd` (search-api) | `chatbot-agent` |

## How sync works

- Each owning service edits its own `.proto` file **in its own repo**, not
  here.
- That service's CI pushes the current file into this repo (`proto/<pkg>/v1/<pkg>.proto`)
  whenever it changes on `main` — a plain copy + commit + push, no build
  step, no validation.
- Services that consume proto files but don't own them get their own copy
  either via `vm-infra`'s `protos/` submodule (pointing at this repo — for
  services that build from the `vm-infra` workspace root) or via their own
  CI pulling the current content of this repo into their own committed
  `protos/`/generated-stub files (for services that build from their own
  repo as Docker context).

## Rules

- Edit the `.proto` file in the owning service's repo, not here — changes
  made directly in this repo will be overwritten by the next sync.
- No breaking-change tooling is enforced. Be careful with in-place field
  changes; prefer adding a new field/RPC over changing an existing one.
