# Source bundle

| File | Contents |
|---|---|
| `chat-v2.zip` | chat app, `develop` @ `bcbaa6dab` (2026-09-23) |
| `code-interpreter.zip` | code-interpreter `develop` + latest upstream merged (`bf31b41`) |
| `agents-v2.zip` | agents library (`@librechat/agents` fork, published as `@illuma-ai/agents`) `develop` + latest upstream merged (`f6b10969`) |
| `CHAT-V2-CHANGES.md` | chat-v2 functional & design changes with exact file:line references |

Source only: no `node_modules`, `.git` history, `.env` or build output. Run `npm ci` after unzipping.

Download everything: `curl -L -o bundle.zip https://api.github.com/repos/illuma-ai/kestrel-drift-4821/zipball/main`
