# Skill Development Standards

When building or modifying any NanoClaw skill for the dm-with-andreas agent group:

## Agent Group
- ID: `ag-1778621651804-pya2np`
- Folder: `groups/dm-with-andreas/`
- Skills folder: `groups/dm-with-andreas/skills/`

## Required Steps for Every Skill

### 1. Build skill files
- `SKILL.md` in `groups/dm-with-andreas/skills/<name>/`
- Scripts in `groups/dm-with-andreas/skills/<name>/scripts/` (if any)
- Make all scripts executable: `chmod +x groups/dm-with-andreas/skills/<name>/scripts/*`

### 2. Update CLAUDE.local.md
- Add an explicit instruction telling Pepe how to use the skill
- Reference the skill path and any key commands

### 3. Install container dependencies

**apt/npm packages** (tracked in DB, no Dockerfile change needed):
```bash
ncl groups config add-package --id ag-1778621651804-pya2np --apt <pkg>
ncl groups config add-package --id ag-1778621651804-pya2np --npm <pkg>
```
Requires `--rebuild` on restart (see step 6).

**Python packages** (edit `container/Dockerfile`):

The container base is `node:22-trixie-slim` (Debian 13) + a `python:3.14-slim` build stage that copies Python 3.14 and its shared library into the image. Adding a new Python tool is just one more `pip install` line in the build stage:

```dockerfile
# In the python-tools build stage:
ARG MY_TOOL_VERSION=x.y.z
RUN pip install --no-cache-dir "aula==${AULA_VERSION}" "mytool==${MY_TOOL_VERSION}"

# Then copy its entrypoint into the main stage:
COPY --from=python-tools /usr/local/bin/mytool /usr/local/bin/mytool
```

⚠️ Both the `python-tools` stage and the main stage must use the same Debian version (both Trixie/Debian 13) or the glibc versions won't match and Python will crash with `GLIBC_x.xx not found`.

### 4. Add required mounts

Two places to update:

**a) Mount allowlist** (`~/.config/nanoclaw/mount-allowlist.json`):
```json
{
  "path": "~/.config/mytool",
  "allowReadWrite": true,
  "description": "mytool config and token cache"
}
```
Use `allowReadWrite: true` if the tool needs to write (e.g. refresh tokens). The allowlist is reloaded on each spawn — no service restart needed.

**b) Container config DB**:
```bash
pnpm exec tsx scripts/q.ts data/v2.db \
  "UPDATE container_configs SET additional_mounts = json_insert(additional_mounts, '\$[#]', json('{\"hostPath\":\"/home/andreas/.config/mytool\",\"containerPath\":\"/home/node/.config/mytool\",\"readonly\":false}')), updated_at = datetime('now') WHERE agent_group_id = 'ag-1778621651804-pya2np'"
```
Use an absolute `containerPath` (starts with `/`) — the mount security module supports this and will not prepend `/workspace/extra/`.

### 5. Set environment variables

Stored as a JSON object in the `env` column:
```bash
# Set (overwrites all env vars for this group):
pnpm exec tsx scripts/q.ts data/v2.db \
  "UPDATE container_configs SET env = '{\"MY_VAR\":\"value\"}', updated_at = datetime('now') WHERE agent_group_id = 'ag-1778621651804-pya2np'"

# Read current value:
pnpm exec tsx scripts/q.ts data/v2.db \
  "SELECT env FROM container_configs WHERE agent_group_id = 'ag-1778621651804-pya2np'"
```

### 6. Rebuild container image

Required when:
- You changed `container/Dockerfile` (new packages, new tools)
- You added an apt/npm package via `ncl groups config add-package`

Not required when:
- Only mount or env var changes (those are read at spawn time)
- Only skill file changes (skills are symlinked from the host at spawn time)

```bash
./container/build.sh
```

### 7. Restart agent

```bash
# Without rebuild (mount/env/skill changes only):
ncl groups restart --id ag-1778621651804-pya2np

# With rebuild (Dockerfile or package changes):
ncl groups restart --id ag-1778621651804-pya2np --rebuild

# With a wake message (agent acts on it immediately after restart):
ncl groups restart --id ag-1778621651804-pya2np --message "Verify aula works: run aula --output json profile"
```

### 8. Test end-to-end via CLI channel

```bash
pnpm run chat "<test message>"
```

⚠️ **Cold-start latency**: The first message after a restart takes ~2 minutes — the container has to start, Claude has to process, and the tool may have its own startup time. The chat script times out at 120s. If it times out, check the outbound DB for what Pepe actually replied:
```bash
pnpm exec tsx scripts/q.ts \
  data/v2-sessions/ag-1778621651804-pya2np/sess-*/outbound.db \
  "SELECT * FROM messages_out ORDER BY seq DESC LIMIT 1"
```

Verify the response is correct before considering the skill done. Fix any issues before finishing.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `command not found` | Binary not in image | Check Dockerfile, rebuild |
| `Permission denied` on binary | uv dirs under `/root` | Use `UV_TOOL_DIR=/opt/uv-tools UV_PYTHON_INSTALL_DIR=/opt/uv-python` |
| Auth/token errors | Token file not mounted or mount is readonly | Check mount allowlist + container config; set `allowReadWrite: true` |
| Reply goes to WhatsApp, not CLI | CLI wiring uses `agent-shared` or no CLI destination | Set wiring `session_mode: shared`; add CLI destination via `ncl destinations add` |
| Timeout on first message | Normal cold-start delay | Check outbound DB for response; message arrived after timeout |
| Service crash-loops after code change | Migration conflict | Check error log; if column already exists, insert a row into `schema_version` to mark migration as applied |
