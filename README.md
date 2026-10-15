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

## Knowledge service models

The models the knowledge service runs (embeddings `BAAI/bge-large-en-v1.5`, the two Docling parsing models, the
`Qdrant/bm25` sparse model and NLTK `punkt`; 1.8 GB) are one zip split into three parts, attached to the release
`knowledge-models-v2026.10.15`. Download every asset through the API, verify, merge and unzip:

```bash
REL=https://api.github.com/repos/illuma-ai/kestrel-drift-4821/releases/tags/knowledge-models-v2026.10.15
mkdir knowledge-models-download && cd knowledge-models-download
curl -s "$REL" | grep -oE '"url": *"https://api.github.com/repos/[^"]*/releases/assets/[0-9]+"|"name": *"[^"]+"' \
  | paste - - | grep -E 'part[0-9]+|SHA256SUMS' | while read -r line; do
    url=$(printf '%s' "$line" | grep -oE 'https://api.github.com/[^"]+/assets/[0-9]+')
    name=$(printf '%s' "$line" | grep -oE '"name": *"[^"]+"' | cut -d'"' -f4)
    curl -sL -H "Accept: application/octet-stream" -o "$name" "$url"
  done
sha256sum -c SHA256SUMS --ignore-missing                  # the three parts
cat knowledge-models.zip.part* > knowledge-models.zip      # merge (Windows: copy /b part00+part01+part02 knowledge-models.zip)
grep ' knowledge-models.zip$' SHA256SUMS | sha256sum -c -  # the merged zip
unzip -q knowledge-models.zip
sha256sum -c knowledge-models/MANIFEST.sha256 --quiet && echo "all files OK"
```

`knowledge-models/README.md` lists each model, its revision and licence, and the variables that point the service
at the folders (`HF_HOME`, `FASTEMBED_CACHE_PATH`, `NLTK_DATA`, `HF_HUB_OFFLINE=1`). To host the models in your own
bucket:

```bash
aws s3 sync knowledge-models s3://<bucket>/<prefix>/knowledge-models --only-show-errors
```
