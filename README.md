# Source bundle

| File | Contents |
|---|---|
| `chat-v2.zip` | chat app, `develop` @ `bcbaa6dab` (2026-09-23) |
| `code-interpreter.zip` | code-interpreter `develop` + latest upstream merged (`bf31b41`) |
| `agents-v2.zip` | agents library (`@librechat/agents` fork, published as `@illuma-ai/agents`) `develop` + latest upstream merged (`f6b10969`) |
| `CHAT-V2-CHANGES.md` | chat-v2 functional & design changes with exact file:line references |
| **`rag-api.zip`** (release asset, ~1.5 GB) | RAG API, Ranger-branded: source **plus both models** (`bge-base-en-v1.5` embeddings, `bge-reranker-base` reranker), ready to build and host with no model downloads. Release `rag-api-v1.0.0` |

Source only for the three zips above: no `node_modules`, `.git` history, `.env` or build output. Run `npm ci` after unzipping.

Download the repository: `curl -L -o bundle.zip https://api.github.com/repos/illuma-ai/kestrel-drift-4821/zipball/main`

## RAG API

`rag-api.zip` is too large to commit (git hosts reject files over 100 MB), so it is
attached to the release instead:

```bash
URL=$(curl -s https://api.github.com/repos/illuma-ai/kestrel-drift-4821/releases/tags/rag-api-v1.0.0 \
  | grep -o '"url": *"https://api.github.com/repos/[^"]*/releases/assets/[0-9]*"' | head -1 | cut -d'"' -f4)
curl -L -H "Accept: application/octet-stream" -o rag-api.zip "$URL"
unzip rag-api.zip && cd rag-api
```

Then follow `docs/deployment/eks-production.md` (build the images, deploy on EKS)
and `docs/deployment/chat-v2-integration.md` (set `RAG_API_URL` and the shared
`JWT_SECRET` in chat-v2). Production settings: `.env.prod.example` (local models,
AWS Textract only, no API keys).
