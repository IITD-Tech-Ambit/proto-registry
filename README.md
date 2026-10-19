# proto-registry

Source of truth for Research Ambit cross-service gRPC contracts. The contract files are `proto/<package>/v1/<package>.proto`.

api-gateway, research-ambit-main, chatbot-agent, and the SEO backend each keep their own copy of these files. Copy the `.proto` from this repository when a consumer needs the update. This repository does not run a sync or a byte-identity check of those copies.

## Packages

### `auth.v1`

`AuthService` (`proto/auth/v1/auth.proto`) is the identity plane, served by auth-service and called by api-gateway through Envoy. `VerifyToken` validates an application session JWT issued by auth-service (never an IITD access token) and returns the identity claims embedded in it. `GetUser` loads the persisted user profile from the auth-service users store.

### `directory.v1`

`DirectoryService`, `KnowledgeGraphService`, `ContentService`, `UserService`, and `SuggestionService` (`proto/directory/v1/directory.proto`) share the research-ambit-main backend listener (Docker service `backend`, gRPC `:50055`, Envoy prefix `/directory.v1.`). They freeze the current `/api/*` surface except `/api/auth/*` and `/health`: faculty directory reads, knowledge-graph and atlas reads, CMS magazine reads and writes, CMS account reads and writes, and suggestion intake. Requests are typed. Responses are typed DTOs, or an opaque UTF-8 JSON string (`data_json`, `detail_json`, `body_json`) where proto3 cannot round-trip the REST payload. The gateway rebuilds the `{ success, message, data, timestamp }` envelope from the gRPC data payload; `GetAtlas` is the exception and returns the raw atlas body.

### `search.v1`

`SearchService` and `TaxonomyService` (`proto/search/v1/search.proto`) share the search-api listener (gRPC `:50053`, Envoy prefix `/search.v1.`) and freeze `/search/*`, whose Fastify routes live under `/api/v1`. `SearchService` covers search, author scope, suggest, faculty-for-query, documents, author collaborators, and the intellectual-property routes. `TaxonomyService` covers taxonomy reads. Suggest, taxonomy, faculty-for-query, and collaborator payloads are typed; document-bearing and polymorphic payloads are `*_json` strings the gateway splices verbatim. These RPCs are public and read-only.

### `embedding.v1`

`EmbeddingService` (`proto/embedding/v1/embedding.proto`) is the BGE embedding and cross-encoder reranker, served by `services/embedding` in SEO-Backend-iitd and called by search-api and chatbot-agent through Envoy. `Embed` returns one vector per input text. `Rerank` returns document scores sorted descending, truncated to `top_n` when `top_n > 0`; `0` means no truncation.

### `chat.v1`

`ChatService` (`proto/chat/v1/chat.proto`) is an internal helper served by chatbot-agent. `CheckQuota` reads the per-user daily message quota and does not consume it, so other services can see quota state. The public answer stream stays SSE at api-gateway.

## Compatibility

Published field numbers stay as assigned. `search.v1` records that `FacultyForQuery` is an unchanged wire contract and that its field numbers are preserved. No file declares `reserved` numbers or names.

Comments on fields record the JSON name the gateway emits when it is not the proto name (`citation_count` as `citationCount`, and the same pattern elsewhere). There is no `json_name` option. `optional` marks presence so an unset field is omitted; directory comments also say when a missing value must be JSON `null` instead. Payloads proto3 cannot round-trip stay opaque JSON strings, and the gateway splices that text into the REST body unchanged.

`auth.v1.User.role` is field 8. auth-service computes it, and the value is never stored in the JWT.

## Making a change

Edit the `.proto` in this repository first. Then update the vendored copy in each consumer that ships that file: api-gateway, research-ambit-main, chatbot-agent, and the SEO backend.
