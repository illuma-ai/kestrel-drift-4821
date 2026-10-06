# Source bundle

| File | Contents |
|---|---|
| `chat-v2.zip` | chat app, `develop` @ `bcbaa6dab` (2026-09-23) |
| `code-interpreter.zip` | code-interpreter `develop` + latest upstream merged (`bf31b41`) |
| `agents-v2.zip` | agents library (`@librechat/agents` fork, published as `@illuma-ai/agents`) `develop` + latest upstream merged (`f6b10969`) |
| `rag-api.zip` | RAG API (Ranger-branded), upstream `c4e5cbf` + retrieval enhancements @ `919c9cd` (2026-10-06): structured extraction, Textract OCR, hybrid search, self-hosted embeddings, EKS deployment |
| `rag-api-models/bge-base-en-v1.5/` | embedding model for the RAG API, split into files under 100 MB; baked into the embedding server image so nothing downloads at runtime |
| `CHAT-V2-CHANGES.md` | chat-v2 functional & design changes with exact file:line references |

Source only: no `node_modules`, `.git` history, `.env` or build output. Run `npm ci` after unzipping.

Download everything: `curl -L -o bundle.zip https://api.github.com/repos/illuma-ai/kestrel-drift-4821/zipball/main`

## RAG API quick start

```bash
unzip rag-api.zip
mkdir -p rag-api/models && cp -r rag-api-models/bge-base-en-v1.5 rag-api/models/
cd rag-api
```

Then follow `docs/deployment/eks-production.md` (build the two images, deploy on
EKS) and `docs/deployment/chat-v2-integration.md` (set `RAG_API_URL` and the
shared `JWT_SECRET` in chat-v2). Production settings: `.env.prod.example`
(local embedding model, AWS Textract only, no API keys).
