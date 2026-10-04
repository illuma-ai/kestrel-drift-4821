# chat-v2 — Functional & Design Changes (fork delta)

**Snapshot:** `illuma-ai/chat-v2` branch **`develop` @ `bcbaa6dab` (2026-09-23)**, which includes
the upstream rc4 merge `8b6a6b778`. The `main` branch is stale (2026-08-03), so work from
`develop`.

**Scope:** chat-v2 only. The agents library and code-interpreter folders in this bundle are already
updated to the latest upstream plus their fork deltas. They are mentioned below only where chat-v2
calls into them.

**How to read references:** every reference is `path:line`, relative to the `chat-v2/` folder.
Line numbers are exact at `bcbaa6dab`. The master ledger of all fork divergences is
**`docs/fork-ui-deltas.md`**, cited below as `§NN`.

---

## 1. Document preview (Excel, Word, PowerPoint, PDF, CSV)

### What users get
- Word, PowerPoint, ODF and PDF files open **page-perfect** in three places: the Artifacts side
  panel, a modal popup, or fullscreen. This works for **uploaded** and **agent-generated** files.
  Text is selectable, the modal and fullscreen views have a thumbnail rail, and an outline button
  lists the sections.
- **Spreadsheets (xlsx, xls, ods, csv) open in a native grid**, not as a PDF. The grid keeps cell
  styles, formulas, embedded charts and sheet tabs.
- Previews are rendered **lazily, when opened, from the stored original**, so they never expire.
  If rendering fails, the server returns 404 and the UI falls back to download-only.
- Legacy `.doc` files are converted on upload instead of being rejected (commit `fe575023d`).

### Data flow
```
GET /api/files/:id/preview/pdf  → chat-v2 reads the original (S3/local storage strategy)
      ├─ native PDF   → streamed back as-is
      └─ office file  → POST {LIBRECHAT_CODE_BASEURL}/documents/preview (multipart)
                         code-interpreter converts it to PDF with LibreOffice
                         → chat-v2 downloads the PDF → client  (Cache-Control: private, max-age=300)
GET /api/files/:id/preview/original → raw bytes, used by the spreadsheet grid viewer
```

### Exact files
| What | File:line |
|---|---|
| Preview status poll route | `api/server/routes/files/files.js:604` |
| PDF preview route | `api/server/routes/files/files.js:664` |
| Original-bytes route | `api/server/routes/files/files.js:692` |
| Read original from storage | `api/server/services/Files/documentPreview.js:56` (`readOriginalDocument`) |
| Render file → PDF | `api/server/services/Files/documentPreview.js:84` (`renderFilePreviewPdf`) |
| Render buffer → PDF | `api/server/services/Files/documentPreview.js:133` (`renderBufferToPdfBytes`) |
| Upload stores the preview locator (`text` = preview URL, `textFormat:'pdf'`) | `api/server/services/Files/process.js:1618-1690` |
| Executor endpoint constants | `packages/api/src/files/documents/endpoints.ts:16` (`DOCUMENT_PREVIEW_PATH`), `:19` (`DOCUMENT_PAGES_PATH`) |
| Office type detection | `packages/api/src/files/documents/office.ts:42` `officeBucket`, `:78` `isOfficeDocument`, `:82` `isNativePdf`, `:92` `isPdfPreviewable`, `:105` `buildPreviewPdfRoute` |
| Executor render calls | `packages/api/src/files/documents/render.ts:99` `renderCodeDocumentToPdf`, `:138` `renderUploadedDocumentToPdf` |
| Viewer router (spreadsheet → grid, else PDF) | `client/src/components/Artifacts/DocViewer.tsx` |
| Modal/fullscreen portal | `client/src/components/Artifacts/DocViewerShell.tsx` |
| PDF viewer (pdf.js, text layer, thumbnails) | `client/src/components/Artifacts/PdfViewer.tsx`, `client/src/components/Artifacts/pdf/render.ts` |
| Excel grid viewer | `client/src/components/Artifacts/ExcelViewer.tsx` (uses the `@illuma-ai/doc-viewer` package) |
| Artifact tabs | `client/src/components/Artifacts/ArtifactTabs.tsx` |
| Spreadsheet detection | `client/src/utils/artifacts.ts:613` (`isSpreadsheetArtifact`) |
| Preview URL → original URL | `client/src/utils/artifacts.ts:650` (`originalFileUrlFromPreview`) |
| Popup ↔ side-panel handoff | `client/src/hooks/Artifacts/useOpenDocViewer.ts` |
| File chip preview dialog | `client/src/components/Chat/Messages/Content/FilePreviewDialog.tsx` |
| Lazy chunk split (vendor bundle 6.79 MB → 3.55 MB) | `client/vite.config.ts` (`manualChunks`) |
| Dependencies | `client/package.json:47` `@illuma-ai/doc-viewer ^1.0.0`, `client/package.json:111` `pdfjs-dist ^5.4.624`, `package.json:279` |
| Design docs | `docs/architecture-document-rendering.md`, `docs/attachments-option-a-rendered-native.md`, `docs/fork-ui-deltas.md` §14e, §14f, §60, §67 |

**Configuration:** `LIBRECHAT_CODE_BASEURL` must point at code-interpreter's **`/v1`** root
(JWT auth).

**Commits:**
- `a64a84f2c` (06-03): first viewer
- `c12335c8a` (06-08)
- `54164df16` (06-18)
- `95572b4b7` (07-04)
- `d36729eb6` and `07584f20e` (07-25): moved to the `/documents/*` endpoints
- `6f0662146` (08-14)
- `fe575023d` (08-28): `.doc` conversion

---

## 2. Icon library — where the icons live

Icons come from three layers, described below.

### 2a. `@illuma-ai/icons` npm package — the primary library, including the **Microsoft 365 icons**
- **Version pins.** `client/package.json:48` uses `^2.7.0`. Four packages still pin `^2.6.2`:
  - `packages/api/package.json:179`
  - `packages/client/package.json:130`
  - `packages/data-provider/package.json:46`
  - `packages/data-schemas/package.json:99`
- **Microsoft 365 file icons** (Fluent colours): `word`, `excel`, `powerpoint`, `outlook`,
  `onedrive`, `sharepoint`, `office`.
- **Other file icons:** csv, html, image, java, javascript, json, markdown, pdf, postgresql,
  python, react, text, typescript, yaml.
- **There is no Teams icon.**
- **Subpath exports:**
  - `@illuma-ai/icons/files`: `FileIcon`, `ArtifactIcon`, `AnimatedFileIcon`,
    `resolveFileIcon`, `resolveArtifactIcon`.
  - `@illuma-ai/icons/files/animations.css`
  - `@illuma-ai/icons/animated`
  - `@illuma-ai/icons/brand`: `TerminalIcon`, `AnimatedTerminalIcon`, `ThinkingOrb`.
- **Source repo:** `illuma-ai/icons`. It is **not checked out locally**; only the built
  `node_modules/@illuma-ai/icons/dist` exists.
- **To add an icon:**
  1. Edit the icons repo, then build, bump the version and publish.
  2. Bump the dependency in chat-v2.
  3. Clear `client/node_modules/.vite`.
  4. Restart Vite.
- **Consumers include:**
  - `client/src/components/Chat/Messages/Content/ToolOutput/ToolIcon.tsx`
  - `client/src/components/Chat/Messages/Content/Parts/Attachment.tsx`
  - `client/src/components/Artifacts/ArtifactButton.tsx`
  - `client/src/components/Chat/Input/Files/FilePreview.tsx`
  - `client/src/components/Files/FilesGridView.tsx`

### 2b. Self-hosted logos in `client/public/assets/`, served at `/assets/*`
- **Provider, model and connector logos.** Examples: `anthropic.svg`, `openai.svg`, `google.svg`,
  `gmail.svg`, `slack.svg`, `github.svg`, `jira.svg`, `notion.png`.
- **Microsoft logos here are only `outlook.svg`, `sharepoint.svg` and `microsoft.svg`.**
  OneDrive operations use `microsoft.svg`.
- `client/public/assets/capabilities/` holds the capability tiles: agents, connectors, data,
  documents, presentations, skills, spreadsheets, websearch.

### 2c. Where icons are resolved in code
| Kind | File:line |
|---|---|
| Tool/operation icon base path | `api/app/clients/tools/operations/manifest.js:5` (`ICON_BASE = '/assets'`) |
| Tool/operation icon picker | `api/app/clients/tools/operations/manifest.js:45` (`iconFor(op)`). It checks the key prefix first, then a provider map. |
| Client pluginKey → icon map | `client/src/hooks/Plugins/useToolIconMap.ts:14`. Used by `ToolCall.tsx`, `ToolCallGroup.tsx` and `StackedToolIcons.tsx` (§32). |
| Model vendor icons | `packages/data-provider/src/modelIcons.ts:39` (`MODEL_VENDORS`) |
| Tool-type glyph map | `client/src/components/Chat/Messages/Content/ToolOutput/ToolIcon.tsx:104` (`ICON_MAP`) |
| Inline SVG components | `packages/client/src/svgs/SharePointIcon.tsx`, `client/src/components/Workflows/OutlookIcon.tsx` |

---

## 3. Questions & answers (ask-user-question): chat-v2 side

- **What comes from upstream.** The question tool (`ask_user_question` and the batched
  `ask_user_questions`) and most of the question UI are upstream code.
- **What the fork adds: answer resume.** When the user answers, the server rebuilds the agent
  graph. chat-v2 then calls the agents-library fork API `rehydrateFromContentParts(seedContent)`,
  which gives three fixes:
  - Earlier tool steps keep their original order.
  - The answered tool shows its output instead of "Cancelled".
  - Multi-question batches resume without crashing.
- **Feature probe:** `api/server/controllers/agents/client.js:5533`
  (`typeof StandardGraph.prototype.rehydrateFromContentParts === 'function'`)
- **Call site:** `api/server/controllers/agents/client.js:5645-5652`
- **Why it's documented:** `api/server/controllers/agents/client.js:5519`
- ⚠️ **Silent-regression risk.** If an agents build **without** the fork patch is installed, the
  probe fails quietly and resume breaks again. Always consume the agents library through the
  `@illuma-ai/agents` alias in `branding.config.json`.
- **Question UI files** (mostly upstream):
  - `client/src/components/Chat/Messages/Content/AskUserQuestion.tsx`
  - `client/src/components/Chat/Messages/Content/AskUserQuestions.tsx`
  - `client/src/components/Chat/Messages/Content/AskUserQuestionCall.tsx`
  - `client/src/components/Chat/Messages/Content/AskUserQuestionProgress.tsx`
  - `client/src/components/Chat/Input/AskUserQuestionPopover.tsx`
  - `client/src/hooks/Input/useAskAnswerMode.ts`
  - `client/src/hooks/Input/useAskQuestionsForm.ts`
  - `client/src/components/Chat/ask/options.tsx`
- **Fork-only UI commit:** `56a0ccf21` (09-14). The pause surfaces (question and approval cards)
  share one enforced fill/colour vocabulary.

---

## 4. Tool grouping & tool-chain animations

### 4a. Provider tool grouping: ⚠️ **parked, NOT on `develop`**
- **Branch:** `parked/tool-grouping-collections-mcp`, HEAD `282e338c7` (2026-07-22). It has not
  been merged, and `develop` is about 1,237 commits ahead of it. The paths below exist **on that
  branch only**.
- **What it does:** operation tools collapse into one row per app, for example
  "Outlook (12 tools)". This applies in every tool picker. Rows are collapsed by default, and a
  search expands them.
- **Files on the branch:**

| What | File (on the parked branch) |
|---|---|
| Group registry: 14 groups, `GROUP_META` (label + icon), `groupForKey` (longest prefix wins) | `api/app/clients/tools/operations/groups.js` |
| Coverage test (fails if any operation is ungrouped) | `api/app/clients/tools/operations/groups.spec.js` |
| Wire field `TPlugin.group {id,label,icon}` | `packages/data-provider/src/schemas.ts` |
| Generic partition util | `client/src/utils/toolGrouping.ts` (`partitionByGroup`) |
| Shared accordion | `client/src/components/SidePanel/Agents/Tools/ToolGroupSection.tsx` |
| Pickers | `client/src/components/SidePanel/Agents/Tools/MarketplaceCatalog.tsx`, `Builder/cards/ToolConnectorDialog.tsx`, `Workflows/canvas/NodePalette.tsx`, `Workflows/inspector/ToolNodeForm.tsx` |
| How-to guide and skill | `docs/adding-a-grouped-tool.md`, `.claude/skills/adding-grouped-tool/` |

- **Also on the branch:**
  - **Collections:** reusable tool bundles. Attaching one expands it into `agent.tools`.
  - A collection exposed as an **MCP server**, with API-key auth.
- **Commits:** `617afb574`, `431aa60c8`, `595693305`, `6ad070c94`, `3b6d73d68`, `231c77d73`,
  `5bb978fda`, `282e338c7`.

### 4b. Tool-call chain animations (on `develop`)
These are pure CSS plus inline transitions, with no framer-motion. Every animation respects
`prefers-reduced-motion`.

| What | File:line | Technique |
|---|---|---|
| Expand/collapse of a tool group | `client/src/hooks/Messages/useExpandCollapse.ts:6`, `:47` | `grid-template-rows 0fr→1fr` + opacity, **0.3s `cubic-bezier(0.16,1,0.3,1)`**. Content is `inert` while collapsed. |
| Running-step pulse | `client/src/components/Chat/Messages/Content/ToolCallGroup.tsx:213` | `animate-pulse` while the step runs |
| Rail icon | `client/src/components/Chat/Messages/Content/ToolCallGroup.tsx:236` | `StepRailIcon` |
| Live group header | `client/src/components/Chat/Messages/Content/ToolCallGroup.tsx:1049` | `animate-pulse` while the group is live |
| Chevron | `client/src/components/Chat/Messages/Content/ToolCallGroup.tsx:1112` | `rotate-180`, 200ms ease-out, shown on hover/focus |
| Live label shimmer | `client/src/components/Chat/Messages/Content/ProgressText.tsx:172` | `.shimmer` class |
| Streamed-word fade | `client/src/components/Chat/Messages/Content/animate.tsx:7-9` | `FADE_DURATION_MS=250`, `FADE_STAGGER_MS=25`, `FADE_STAGGER_MAX_MS=250` |
| Collapsible keyframes | `client/tailwind.config.cjs:61`, `:65`, `:108-109` | down 0.3s `cubic-bezier(0,0,0.2,1)` / up 0.2s `cubic-bezier(0.4,0,1,1)` |
| Other live components | `Content/ActivityPhaseGroup.tsx`, `Content/Parts/RunActivityLine.tsx`, `Content/Parts/Thinking.tsx`, `Content/thinkingGlyph.tsx`, `Content/ToolStepContext.ts`, `client/src/components/ui/ToolIconWaterFill.tsx`, `client/src/components/ui/BrandLoader.tsx` | — |

CSS keyframes in `client/src/style.css`:

| Keyframe | Line | Purpose | Timing |
|---|---|---|---|
| `stepper-flow-move` | 40 | A band flows down the rail into the running step | 1.5s linear infinite |
| `tool-activity` | 61 | Running icon "breathes" (opacity .5→1, scale .9→1) | 1.4s ease-in-out |
| `icon-water-fill` | 90 | Tool icon fills bottom-up (clip-path) | 2.2s ease-in-out |
| `brand-loader-window` | 283 | Brand loader | 5s ease-in-out |
| `brand-loader-glyph` | 292 | Brand loader | 5s ease-in-out |
| `brand-loader-dot` | 319 | Loader dots | 1.4s, staggered 0.2s / 0.4s |
| `card-shimmer-sweep` | 347 | Loading sweep on attachment cards | 1.4s ease-in-out |
| `lc-fade-in` | 2689 | Streamed words fade in (`[data-lc-fade]`) | 250ms ease-out |
| `shimmer` | 3261 | Text-clip gradient (`--shimmer-base` / `--shimmer-dip` tokens) | 4s linear |

**Ledger sections:** §13j, §13k, §24 (tool-call chain), §32 (provider logos), §46 (monochrome
stepper), §47 (BrandLoader and water-fill), §52 (auto-collapse), §55a.

**Commits:**
- `9b6c6dd3e` (07-18): monochrome stepper
- `5d3edeb38` (08-14): phase transitions
- `f29c3d3a7` (09-08): rail product mark
- `1f66e9a8e` and `ddd8d78d1` (09-20): live activity row

---

## 5. Document upload pipeline: grouping, native send, caching

**Design docs:**
- `docs/attachments-unified-routing.md`: §3 design, §5f caching, §5i collective budget, §8 native
  lane
- `docs/attachments-option-a-rendered-native.md`
- `docs/fork-ui-deltas.md` §14, §66

### 5a. Auto-routing: the client proposes a lane, the server decides
- The client picks a lane for each file. The lanes are inline, native document, rendered pages,
  `file_search`, and the code sandbox.
  - Routing table: `packages/data-provider/src/attachments.ts:428` (`classifyAttachment`) and
    `:601` (`planAutoUpload`)
- The server re-derives the lane and clamps it, so the client can never force one (`45055ba31`).
  - Server function: `packages/api/src/files/autoroute.ts:58` (`resolveAutoToolResource`)
  - Called at: `api/server/services/Files/process.js:861`
- **Feature flag:** `interface.attachmentAutoRouting`, at `packages/data-provider/src/config.ts:2245`.
- **Paste** auto-routes the same way drag-drop does (§66).

### 5b. Grouping: shared budgets across the whole upload batch
| Rule | File:line | Commit |
|---|---|---|
| The batch shares one inline budget of **0.6 MB**. Files past it go to `file_search`; the first file is exempt. | `packages/data-provider/src/attachments.ts:150` (`ATTACHMENT_INLINE_CONVERSATION_MAX_BYTES`) | `b39e43593` |
| Office files count at an estimated text size (**0.2×**), so one deck doesn't evict the rest. | `packages/data-provider/src/attachments.ts:166` (`ATTACHMENT_OFFICE_CONTAINER_TEXT_RATIO`) | `1be5c699a` |
| At most **15 rendered pages per document**. | `packages/data-provider/src/attachments.ts:217` (`ATTACHMENT_RENDERED_NATIVE_MAX_PAGES`) | `24ae9c6f0` |
| At most **60 rendered pages across all documents**. | `packages/data-provider/src/attachments.ts:230` (`ATTACHMENT_RENDERED_NATIVE_MAX_TOTAL_PAGES`) | `eddeb51bb` |
| Native PDFs share a page budget at encode time (the provider allows 100 pages in total). Overflow is sent as text. | `packages/api/src/files/encode/document.ts` | `3640719cf` |

### 5c. Native document send vs text extraction
- **Which providers get native PDF blocks:** only providers that can read PDFs. The list is
  `packages/data-provider/src/schemas.ts:54` (`documentSupportedProviders`); everyone else gets
  extracted text (`0eb21109a`). Non-document models such as DeepSeek, Qwen and GLM degrade to
  text, and scanned files go to retrieval for OCR (`218700c07`).
- **Rendered-native is the primary PDF lane** (`690cc29da`, `2b51c5993`):
  - **When:** each PDF is rasterized **once at upload**, through code-interpreter's
    `/documents/pages` endpoint.
  - **Code:** `packages/api/src/files/documents/pages.ts:145` (`renderUploadedPdfPages`) and
    `:213` (`fetchRenderedPages`).
  - **Storage:** `<file>.pages.json`, with its locator at `file.metadata.renderedPages`.
  - **Each turn:** the model gets page images plus page text, never the raw PDF bytes again.
  - **Fallbacks:** if there is no executor and the file is ≤ 10 MB, it goes as a native block.
    If rendering fails, it goes to retrieval.
- **PDF passthrough:** PDFs produced by the code tool skip preview conversion and open straight in
  pdf.js. See `packages/api/src/files/code/process.ts:513-517` (§66e).

### 5d. Caching
- **Prompt cache.** The agents library puts a tail cache breakpoint over the whole prefix, which
  includes documents. Measured: turn 1 wrote 107,554 cached tokens and turn 2 read 107,554.
  chat-v2's job is to keep that prefix byte-stable:
  - **The fix (`d96573063`):** files are re-sent in a stable order. They used to be sorted by
    `updatedAt`, which changes every turn and broke the cache; they are now sorted by `createdAt`
    then `_id`.
  - **Where:** `packages/data-schemas/src/methods/file.ts:557`
    (`{ createdAt: 1, _id: 1 }`; also `:255`, `:635`, `:761`).
- **Rendered-pages LRU.** This is a performance cache, not a prompt cache.
  - `packages/api/src/files/documents/pages.ts:50`
    (`RENDERED_PAGES_CACHE_MAX_BYTES = 64 MB`), with `:55` `clearRenderedPagesCache`.
  - Failures are never cached, and rendered PDFs are no longer re-downloaded and base64-encoded on
    every turn (`eddeb51bb`).
  - It is per-task: a miss on another replica re-reads from S3, so results stay correct.

### 5e. Parked: 50-document hardening (PR #12)
- **Branch:** `park/doc-pipeline-hardening` (head `2c059b927`). Its 23 commits are not on
  `develop`.
- **What it adds:**
  - RAG timeouts (`RAG_*_TIMEOUT_MS`)
  - a cross-turn inline budget
  - `RAG_CONTEXT_MAX_CHARS` = 400K
  - higher upload rate limits (250 per user / 500 per IP)
  - a retention-sweeper fix (`9a0d5d9fb`)
- **Blocker:** `develop` carries reverts of this work (`230fbca3b`, `7ba1002e9`), so landing it
  means reverting those reverts.

---

## 6. Other notable chat-v2 deltas
| Delta | Where |
|---|---|
| Upstream merge procedure (the fork stays cleanly pullable; latest is the 09-23 rc4, 94 commits) | `.claude/skills/upstream-merge-pilot/SKILL.md`, `docs/fork-ui-deltas.md` (merge checklist at the bottom) |
| Build-time branding and the agents library alias | `branding.config.json` |
| Sandbox `edit_file` runs `str_replace` inside `/exec` (the executor has no `/files/edit`) | chat-v2 PR #16 |

## Known gaps
1. There is no Teams icon anywhere, and the Microsoft logos are split between `@illuma-ai/icons`
   and `client/public/assets/`.
2. The `@illuma-ai/icons` source repo isn't checked out locally, and `packages/*` still pin
   `^2.6.2`.
3. Tool grouping and Collections are parked on a branch that's about 1,237 commits behind
   `develop`.
4. The 50-document hardening (PR #12) is parked behind reverts.
5. Tool image attachments use presigned S3 URLs with a 120 s TTL that are never re-signed, so
   transcript images return 403 after a few minutes.
