# Harbor Dataset: RS-SDK Woodcutting Task

## Context

**Goal:** Get the Harbor dataset task at `datasets/rs-sdk/woodcutting-10/` working end-to-end: Docker container boots all RS-SDK services, a headless bot client connects and completes tutorial, the MCP tools work for an agent, and the verification script correctly reads the bot's woodcutting level.

**Why it matters:** This is the first Harbor benchmark task for RS-SDK. Once this container works, we can run `harbor run` against it to evaluate how well AI agents can play RuneScape autonomously.

**Key resources:**
- Dataset files: `datasets/rs-sdk/woodcutting-10/` (task.toml, instruction.md, environment/, tests/)
- RS-SDK codebase: this repo (engine/, gateway/, webclient/, sdk/, mcp/)
- Existing test utils: `test/utils/browser.ts` (reference for puppeteer + SDK integration)
- Learnings: `learnings/woodcutting.md`
- Docker image `rs-sdk-woodcutting-10` is already built and loaded locally

**Harbor framework reference (READ THESE when verifying task format correctness):**
The Harbor source is at `~/projects/harbor-explore/downloads/harbor`. Key files to consult:
- `src/harbor/models/task/config.py` — TaskConfig schema (task.toml format, MCPServerConfig fields)
- `src/harbor/verifier/verifier.py` — How verification works (reads /logs/verifier/reward.txt or reward.json)
- `src/harbor/agents/installed/claude_code.py` — How Claude Code agent is installed + MCP server setup
- `src/harbor/agents/installed/base.py` — BaseInstalledAgent.setup() — how MCP .mcp.json is created
- `src/harbor/environments/docker/docker.py` — How Docker environments start, mount logs, exec commands
- `examples/tasks/hello-world/` — Simplest working task example
- `examples/tasks/hello-mcp/` — MCP server task example (closest to ours)
Validate our task.toml, test.sh, and check_level.ts against these sources to make sure the format is correct.

**Architecture:**
```
entrypoint.sh starts:
  1. Game engine (port 8888) — serves game + bot client page
  2. Gateway (port 7780) — WebSocket relay between SDK and bot client
  3. launch-bot.ts — headless puppeteer loads /bot page, connects bot, skips tutorial

Then Harbor installs agent (Claude Code) which gets MCP tools:
  - execute_code(bot_name, code) — runs TypeScript with bot/sdk globals
  - list_bots() / disconnect_bot()

After agent finishes, verifier runs:
  - tests/test.sh → tests/check_level.ts
  - Connects via SDK observer, reads woodcutting level
  - Writes reward = min(level/10, 1.0) to /logs/verifier/reward.txt
```

---

## State

**Progress:** ALL CORE TASKS COMPLETE. Container works end-to-end.

**What's verified (2026-02-11):**
- Engine starts and logs "World ready" within ~2s
- Gateway starts on port 7780
- Puppeteer launches chromium headless, navigates to bot page, and reaches `ingame` state
- Tutorial is auto-skipped, bot lands in Lumbridge (3222, 3222)
- SDK control mode works: can chop trees, drop items, read inventory/skills
- MCP bot manager (`botManager.connect("agent")`) works: auto-discovers bot.env, connects to gateway
- Verification script (`check_level.ts`) correctly reads woodcutting level via observe mode and writes reward.txt
- Full chopping + verification loop works: chop trees → level goes up → verifier reads correct level

**Only fix needed:** `check_level.ts` had `wc?.xp` instead of `wc?.experience` — fixed (display-only, didn't affect reward calculation).

---

## Predictions

- [x] The game client probably works in headless chromium — the existing test suite uses puppeteer headless (`HEADLESS=true` in test/utils/browser.ts) so this should be supported
- [x] The main issue is likely a timing or initialization problem, not a fundamental incompatibility
- [x] The MCP stdio transport will work once the bot is connected — the MCP server just needs the gateway at ws://localhost:7780
- [x] The verification script (check_level.ts) will work on first try since it's just SDK observer mode

---

## Prediction Outcomes

All predictions were correct:
1. **Game client works in headless chromium** — YES. The container starts chromium headless, loads /bot page, and reaches ingame within seconds.
2. **Timing/init problem** — The issue from the previous session appears to have been resolved already. The current Docker image works on first try. The key was the `cp -r webclient/out/bot engine/public/bot` Dockerfile step that copies the built webclient JS to the engine's public dir.
3. **MCP stdio transport works** — YES. `botManager.connect("agent")` correctly loads `bots/agent/bot.env`, connects to `ws://localhost:7780`, and provides `bot`/`sdk` globals for code execution.
4. **Verification script works on first try** — YES. Connects in observe mode, reads woodcutting level, writes reward to `/logs/verifier/reward.txt`.

---

## Discoveries

1. **The container was already working** — The previous session's debugging notes said the bot failed to reach ingame state, but the current Docker image works fine. The `cp -r webclient/out/bot engine/public/bot` line was already in the Dockerfile.
2. **`check_level.ts` had wrong field name** — Used `wc?.xp` but the SDK type is `wc?.experience`. Only affected display, not reward calculation.
3. **Container startup is fast** — Engine starts in ~2s, gateway instantly, bot login + tutorial skip takes ~10s. Total: well under 30s.
4. **Inventory starts full from tutorial** — The bot has 18 tutorial items. Agents need to drop items before chopping or they'll fill up after ~10 trees.
5. **Harbor uses deprecated `memory`/`storage` in hello-world example** — But the `EnvironmentConfig` schema prefers `memory_mb`/`storage_mb`. We use the new fields correctly.
6. **Harbor mounts `/logs/verifier` and `/logs/agent` as volumes** — via `docker-compose-base.yaml`. So `reward.txt` written inside the container is visible to the host verifier. The verifier reads it as `float(content)`.
7. **Harbor's Claude Code agent writes MCP config to user-scope `.claude.json`** — not project-scope `.mcp.json`. This means the MCP server is loaded without a trust dialog, which is necessary for headless benchmark runs.
8. **stdio MCP is simpler for single-container tasks** — The hello-mcp example uses `streamable-http` with a separate docker service. Our stdio approach avoids the extra service, healthcheck, and docker-compose overlay. Both work, but stdio is less moving parts.

---

## Tasks

### Core (all done)
- [x] Debug why headless bot client doesn't reach `ingame` state — it works, no fix needed
- [x] Fix the container so the bot successfully connects and reaches ingame state — already working
- [x] Fix the container so tutorial is skipped and bot is in Lumbridge ready to play — works
- [x] Verify all services start correctly
- [x] Verify MCP tools work inside the container (botManager.connect, execute_code flow)
- [x] Verify the test/verification script works (check_level.ts reads level, writes reward.txt)
- [x] Full end-to-end: chop trees → verify level goes up → reward increases

### Harbor Format Validation (all done)
- [x] Read Harbor's TaskConfig schema and validate our task.toml matches — all fields correct, uses new `memory_mb`/`storage_mb` (not deprecated `memory`/`storage`)
- [x] Read Harbor's MCPServerConfig — stdio transport with `command`+`args` is valid, produces correct `.claude.json` format
- [x] Read Harbor's verifier — test.sh is copied to `/tests/`, runs via `bash /tests/test.sh`, reads `/logs/verifier/reward.txt` as `float()`. Our format matches.
- [x] Read Harbor's hello-mcp example — compared; our stdio approach is simpler (no separate docker service needed), both valid
- [x] Read how Claude Code agent sets up MCP — `_build_claude_json_mcp_servers()` produces `{"type":"stdio","command":"bash","args":[...]}` which is exactly what Claude Code expects

### Later
- [ ] Optimize Docker image size (currently probably 2GB+)
- [ ] Consider pre-packing the game cache during build to speed up container startup
- [ ] Add more tasks to the dataset (level 20, different skills, combat, quests)

---

## Instructions

1. **Read context** — This file, progress-harbor.txt if it exists, and the dataset files at `datasets/rs-sdk/woodcutting-10/`
2. **Pick the most important unchecked task** (not necessarily in order)
3. **Implement it fully** — edit the Dockerfile, entrypoint.sh, launch-bot.ts etc. as needed
4. **Run and verify** — rebuild with `cd datasets/rs-sdk/woodcutting-10/environment && docker build -t rs-sdk-woodcutting-10 .` and test with `docker run`
5. **Update** — Check off tasks, update State section
6. **Commit** — `git add -A && git commit -m "feat: <description>"`

**Important:** When debugging, `docker run --rm -it rs-sdk-woodcutting-10 bash` lets you manually start services and test interactively. This is faster than rebuilding the image for each change.

**Rebuild shortcut:** For changes to entrypoint.sh or launch-bot.ts (COPY'd files), you must rebuild. For changes to rs-sdk code itself, you'd need to either rebuild or `docker cp` files in.

---

## Success Criteria

- Container starts all services (engine, gateway, headless bot) without errors
- Bot reaches ingame state and completes tutorial automatically
- MCP `execute_code` tool works to control the bot
- Verification script reads skill levels and writes reward correctly
- A manual test of "chop trees → check level" works end-to-end

---

## Termination

When all tasks complete OR blocked:
- All done: `<promise>COMPLETE</promise>`
- Blocked: `<promise>BLOCKED</promise>`

---

## If Stuck

1. Compare launch-bot.ts with test/utils/browser.ts line-by-line — the test utils work locally with headless puppeteer
2. Try running the engine locally and connecting via puppeteer to isolate container vs code issues
3. Check if the game engine needs specific environment variables or ports beyond 8888/7780
4. If truly stuck: `<promise>BLOCKED</promise>`
