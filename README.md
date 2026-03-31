# Claude Code 2.1.88 Source Analysis

Recovered from `@anthropic-ai/claude-code@2.1.88` npm package via embedded source map (`cli.js.map`, 60MB, source map v3 with `sourcesContent`). 1902 TypeScript/TSX source files, 516K lines of code.

## Architecture Overview

The codebase is a Bun-bundled TypeScript CLI built with React/Ink for the terminal UI. Key directories:

| Directory             | Purpose                                                               |
| --------------------- | --------------------------------------------------------------------- |
| `src/tools/`          | 40+ tool implementations (Bash, FileEdit, FileRead, Agent, MCP, etc.) |
| `src/services/`       | API, analytics, MCP, compaction, voice, memory, rate limits           |
| `src/components/`     | React/Ink UI components (permissions, prompt input, status line)      |
| `src/skills/bundled/` | Built-in slash commands (/commit, /simplify, /stuck, /debug, etc.)    |
| `src/utils/`          | Model configs, permissions, sandbox, git, settings                    |
| `src/constants/`      | System prompts, pricing, beta headers, OAuth                          |
| `src/bridge/`         | Remote Control / bridge mode for IDE extensions                       |
| `src/memdir/`         | Persistent memory system (auto-memory, dream consolidation)           |

---

## Interesting Findings

### 1. "Undercover Mode" for Anthropic Employees

**`src/utils/undercover.ts`** — When Anthropic employees (`USER_TYPE=ant`) work on public/open-source repos, Claude Code activates "undercover mode" that:

- Strips ALL attribution (no `Co-Authored-By`, no "Generated with Claude Code")
- Removes model name/ID from the system prompt entirely
- Injects strict instructions to never reveal internal codenames, model versions, Slack channels, or that an AI is being used
- Auto-activates unless the repo remote matches a hardcoded internal allowlist
- There is **no force-OFF** — "if we're not confident we're in an internal repo, we stay undercover"

Source refs: `src/utils/undercover.ts` (`isUndercover`, `getUndercoverInstructions`, `shouldShowUndercoverAutoNotice`), `src/utils/commitAttribution.ts` (`getRepoClassCached`, `INTERNAL_MODEL_REPOS`).

Additional details from the code:

- This entire feature is ant-only; the file comment says external builds dead-code-eliminate the Anthropic-only branches.
- Auto mode treats repo classes `'external'`, `'none'`, and even `null` (classification not run yet) as undercover ON. Only a positively identified `'internal'` repo disables it.
- There is a force-ON env var (`CLAUDE_CODE_UNDERCOVER=1`) and a one-time auto-undercover notice, but still no force-OFF path.

The undercover instructions include explicit examples:

> **GOOD**: "Fix race condition in file watcher initialization"
> **BAD**: "1-shotted by claude-opus-4-6", "Generated with Claude Code"

### 2. Internal Repo Allowlist Leak

**`src/utils/commitAttribution.ts`** — Hardcoded list of Anthropic's private repos:

Source refs: `src/utils/commitAttribution.ts` (`INTERNAL_MODEL_REPOS`, `isInternalModelRepo`).

- `anthropics/claude-cli-internal` (the internal version of this codebase)
- `anthropics/anthropic`
- `anthropics/apps`
- `anthropics/casino`
- `anthropics/dbt`
- `anthropics/dotfiles`
- `anthropics/terraform-config`
- `anthropics/hex-export`
- `anthropics/feedback-v2`
- `anthropics/labs`
- `anthropics/argo-rollouts`
- `anthropics/starling-configs`
- `anthropics/ts-tools`, `anthropics/ts-capsules`
- `anthropics/feldspar-testing`
- `anthropics/trellis`
- `anthropics/claude-for-hiring`
- `anthropics/forge-web`
- `anthropics/infra-manifests`
- `anthropics/mycro_manifests`, `anthropics/mycro_configs`
- `anthropics/mobile-apps`

The source comment explicitly says this is a repo-level allowlist, not an org-level allowlist, because the `anthropics` and `anthropic-experimental` orgs also contain public repos like `anthropics/claude-code`. The list stores both SSH and HTTPS remote URL forms.

### 3. Full Model Lineup and Pricing

**`src/utils/model/configs.ts`** — Complete model registry with date-stamped IDs:

Source refs: `src/utils/model/configs.ts` (`ALL_MODEL_CONFIGS`), `src/utils/model/model.ts` (`getDefaultMainLoopModelSetting`), `src/utils/modelCost.ts` (`MODEL_COSTS`, `getOpus46CostTier`).

| Model      | First-Party ID               | Launch Date               |
| ---------- | ---------------------------- | ------------------------- |
| Opus 4.6   | `claude-opus-4-6`            | (no date suffix = latest) |
| Opus 4.5   | `claude-opus-4-5-20251101`   | Nov 1, 2025               |
| Opus 4.1   | `claude-opus-4-1-20250805`   | Aug 5, 2025               |
| Opus 4     | `claude-opus-4-20250514`     | May 14, 2025              |
| Sonnet 4.6 | `claude-sonnet-4-6`          | (no date suffix)          |
| Sonnet 4.5 | `claude-sonnet-4-5-20250929` | Sep 29, 2025              |
| Sonnet 4   | `claude-sonnet-4-20250514`   | May 14, 2025              |
| Haiku 4.5  | `claude-haiku-4-5-20251001`  | Oct 1, 2025               |

The registry also carries Bedrock, Vertex, and Foundry IDs for each model family, not just first-party API names.

**`src/utils/modelCost.ts`** — API pricing per million tokens:

| Tier     | Input | Output | Cache Write | Cache Read | Models                              |
| -------- | ----- | ------ | ----------- | ---------- | ----------------------------------- |
| $3/$15   | $3    | $15    | $3.75       | $0.30      | All Sonnets (3.5, 3.7, 4, 4.5, 4.6) |
| $5/$25   | $5    | $25    | $6.25       | $0.50      | Opus 4.5, Opus 4.6 (normal)         |
| $15/$75  | $15   | $75    | $18.75      | $1.50      | Opus 4, Opus 4.1                    |
| $30/$150 | $30   | $150   | $37.50      | $3.00      | Opus 4.6 (fast mode)                |
| $1/$5    | $1    | $5     | $1.25       | $0.10      | Haiku 4.5                           |
| $0.80/$4 | $0.80 | $4     | $1.00       | $0.08      | Haiku 3.5                           |

Default model selection:

- **Max/Team Premium subscribers**: Opus by default, with `[1m]` appended only when the `isOpus1mMergeEnabled()` path is active
- **Anthropic employees**: `getAntModelOverrideConfig()?.defaultModel`, otherwise Opus with 1M context
- **Everyone else**: `getDefaultSonnetModel()`; the source comment explicitly notes that 3P PAYG can lag first-party and may resolve to an older Sonnet variant

### 4. Internal Feature Flags (`bun:bundle` feature gates)

The codebase uses `feature('FLAG_NAME')` for build-time dead code elimination. Flags found:

Source refs: `src/entrypoints/cli.tsx`, `src/tools.ts`, `src/query.ts`, `src/screens/REPL.tsx`, `src/skills/bundled/index.ts`.

- `KAIROS` — "Assistant mode" / proactive autonomous agent mode
- `KAIROS_BRIEF` — Brief mode (background work with checkpoint notifications)
- `KAIROS_CHANNELS` — Channel-based messaging for Kairos
- `PROACTIVE` — Proactive autonomous agent
- `TRANSCRIPT_CLASSIFIER` — Auto-permission classification
- `BRIDGE_MODE` — Remote Control / IDE extension bridge
- `DIRECT_CONNECT` — Direct connection mode
- `SSH_REMOTE` — SSH remote sessions
- `COORDINATOR_MODE` — Multi-agent coordinator
- `EXTRACT_MEMORIES` — Background memory extraction
- `COMMIT_ATTRIBUTION` — Enhanced commit attribution with trailers
- `NATIVE_CLIENT_ATTESTATION` — Bun-native attestation token in HTTP requests
- `EXPERIMENTAL_SKILL_SEARCH` — DiscoverSkills tool
- `VERIFICATION_AGENT` — Adversarial verification subagent
- `TOKEN_BUDGET` — Token budget mode ("+500k", "spend 2M tokens")
- `MONITOR_TOOL` — Stream events from background processes
- `WEB_BROWSER_TOOL` — Bun WebView browser tool
- `CACHED_MICROCOMPACT` — Time-based micro-compaction
- `CONTEXT_COLLAPSE` — Context collapsing
- `CHICAGO_MCP` — MCP integration
- `BG_SESSIONS` — Background sessions
- `UDS_INBOX` — Unix domain socket inbox
- `CCR_MIRROR` — Remote control mirror
- `LODESTONE` — Unknown purpose
- `CONNECTOR_TEXT` — Summarize connector text
- `TEAMMEM` — Team memory sharing
- `TEMPLATES` — Job classification templates
- `NEW_INIT` — New init flow

These are not passive config toggles. The source uses them to gate lazy `require()`s and entire CLI entrypoints, tools, skills, and UI branches so Bun can tree-shake whole features out of external builds.

### 5. Native Client Attestation

Dedicated deep-dive: [ATTESTATION.md](ATTESTATION.md)

**`src/constants/system.ts`** — Claude Code includes a native attestation mechanism:

Source refs: `src/constants/system.ts` (`getAttributionHeader`).

Important correction: the source uses a `cch=00000` placeholder, not a fixed hash literal. The comment says Bun's native HTTP stack overwrites those zeroes in the serialized request body with a computed attestation token, specifically to avoid changing `Content-Length`.

The same header also carries:

- `cc_version=<MACRO.VERSION>.<fingerprint>`
- `cc_entrypoint=<entrypoint>`
- optional `cc_workload=<workload>`

This is Anthropic's way of verifying requests come from genuine Claude Code binaries — implemented in Zig within a custom Bun fork.

### 6. "Dream" — Autonomous Memory Consolidation

**`src/services/autoDream/`** — Claude Code has a "dream" system that:

Source refs: `src/services/autoDream/autoDream.ts`, `src/services/autoDream/config.ts`, `src/services/autoDream/consolidationPrompt.ts`, `src/tasks/DreamTask/DreamTask.ts`.

- Fires automatically after 24+ hours and 5+ sessions since last consolidation
- Runs as a forked subagent in the background
- Reviews past session transcripts
- Consolidates, merges, and prunes persistent memory files
- Has a lock mechanism to prevent concurrent dreams
- Restricted to read-only Bash (no writes)
- Tracks "dream tasks" visible in the UI

Additional details from the code:

- The defaults are `minHours: 24` and `minSessions: 5`, but both come from the GrowthBook config `tengu_onyx_plover`.
- There is also a `SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000` throttle so once the time gate opens it does not rescan transcripts every turn.
- Auto-dream is disabled entirely in KAIROS mode and remote mode.
- Killing the dream task rolls back the consolidation-lock mtime so the next session can retry.
- `DreamTask` keeps only the most recent 30 assistant turns, and its `filesTouched` list is explicitly documented as incomplete best-effort tracking.

The consolidation prompt tells Claude to act like it's dreaming:

> "You are performing a dream — a reflective pass over your memory files. Synthesize what you've learned recently into durable, well-organized memories."

### 7. Verification Agent — Adversarial Self-Testing

**`src/tools/AgentTool/built-in/verificationAgent.ts`** — A built-in subagent designed to _try to break_ implementations before reporting them as complete. Key aspects:

Source refs: `src/tools/AgentTool/built-in/verificationAgent.ts` (`VERIFICATION_SYSTEM_PROMPT`, `VERIFICATION_AGENT`).

- Explicitly told about its own failure patterns: "verification avoidance" and "being seduced by the first 80%"
- Cannot modify project files (read-only + temp scripts only)
- Required to produce actual command output for every check
- Must include at least one adversarial probe (concurrency, boundary values, idempotency, orphan operations)
- Issues PASS/FAIL/PARTIAL verdicts with evidence
- Warns: "If you catch yourself writing an explanation instead of a command, stop. Run the command."

The implementation details are unusually strict:

- The built-in agent disallows `AgentTool`, `ExitPlanModeTool`, `FileEditTool`, `FileWriteTool`, and `NotebookEditTool`.
- It explicitly permits temp-only harnesses under `/tmp` or `$TMPDIR`.
- Its report format is machine-parseable: every check needs `Command run`, `Output observed`, and `Result`, and the final line must be literal `VERDICT: PASS`, `VERDICT: FAIL`, or `VERDICT: PARTIAL`.

### 8. Ant-Only Prompt Differences

Anthropic employees get different system prompts:

Source refs: `src/constants/prompts.ts` (Anthropic-only prompt branches and `@[MODEL LAUNCH]` comments).

- **Numeric length anchors**: "Keep text between tool calls to ≤25 words. Keep final responses to ≤100 words"
- **Stronger writing guidance**: Long prose section about "communicating with the user" vs external users' terse "Go straight to the point"
- **False-claims mitigation**: "Never claim 'all tests pass' when output shows failures"
- **Assertiveness**: "If you notice the user's request is based on a misconception... say so"
- **Comment policy**: "Default to writing no comments" — more aggressive than external

### 9. OAuth and Infrastructure Details

**`src/constants/oauth.ts`**:

Source refs: `src/constants/oauth.ts`, `src/constants/keys.ts`.

- Production OAuth client ID: `9d1c250a-e61b-44d9-88ed-5944d1962f5e`
- Staging client ID: `22422756-60c9-4084-8eb7-27705fd5cf9a`
- Authorized OAuth redirect through `claude.com/cai/*` for attribution
- FedStart/PubSec deployments: `claude.fedstart.com`, `claude-staging.fedstart.com`
- MCP proxy: `mcp-proxy.anthropic.com`
- Custom OAuth URL allowlist restricted to prevent token leakage

Other notable details:

- MCP OAuth uses a hosted client metadata URL: `https://claude.ai/oauth/claude-code-client-metadata`
- Local dev OAuth reuses the staging client ID and swaps the MCP proxy path to `/v1/toolbox/shttp/mcp/{server_id}`
- Invalid `CLAUDE_CODE_CUSTOM_OAUTH_URL` values throw immediately; the override is hard-allowlisted to approved FedStart-style endpoints only

**`src/constants/keys.ts`** — GrowthBook SDK keys:

- External: `sdk-zAZezfDKGoZuXXKe`
- Internal prod: `sdk-xRVcrliHIlrg4og4`
- Internal dev: `sdk-yZQvlplybuXjYh6L`

### 10. Beta Headers Timeline

**`src/constants/betas.ts`** reveals API feature rollout dates:

Source refs: `src/constants/betas.ts`.

- `claude-code-20250219` — Original Claude Code beta
- `interleaved-thinking-2025-05-14` — Interleaved thinking
- `context-1m-2025-08-07` — 1M context window
- `context-management-2025-06-27` — Context management
- `structured-outputs-2025-12-15` — Structured outputs
- `web-search-2025-03-05` — Web search
- `effort-2025-11-24` — Effort levels
- `fast-mode-2026-02-01` — Fast mode
- `task-budgets-2026-03-13` — Task budgets
- `prompt-caching-scope-2026-01-05` — Prompt caching scope
- `token-efficient-tools-2026-03-28` — Token-efficient tools
- `advisor-tool-2026-03-01` — Advisor tool
- `afk-mode-2026-01-31` — AFK mode
- `redact-thinking-2026-02-12` — Redact thinking

Interesting extras in the same file:

- Tool search uses different beta headers depending on provider:
  - first-party / Foundry: `advanced-tool-use-2025-11-20`
  - Vertex / Bedrock: `tool-search-tool-2025-10-19`
- There is an ant-only internal header: `cli-internal-2026-02-09`
- Connector-text summarization has its own gated header: `summarize-connector-text-2026-03-13`

### 11. Internal Analytics ("Tengu")

Dedicated deep-dive: [TENGU.md](TENGU.md)

All analytics events are prefixed with `tengu_` — presumably an internal codename for the Claude Code telemetry system. GrowthBook feature flags also follow this pattern (e.g., `tengu_attribution_header`, `tengu_onyx_plover`, `tengu_cobalt_lantern`, `tengu_hive_evidence`, `tengu_terminal_panel`).

Source refs: `src/services/analytics/datadog.ts`, `src/services/analytics/firstPartyEventLogger.ts`, `src/services/analytics/sinkKillswitch.ts`, `src/services/analytics/growthbook.ts`, `src/constants/keys.ts`.

The naming runs deep: sink killswitch is `tengu_frond_boric`, event sampling is `tengu_event_sampling_config`, and first-party batching uses `tengu_1p_event_batch_config`.

### 12. Bundled Skills

Internal-only (`ant`) skills not available to external users:

Source refs: `src/skills/bundled/index.ts`, `src/skills/bundled/stuck.ts`, `src/skills/bundled/loremIpsum.ts`, `src/skills/bundled/simplify.ts`, `src/skills/bundled/scheduleRemoteAgents.ts`.

- `/stuck` — Diagnoses frozen Claude Code sessions, posts to `#claude-code-feedback` Slack channel (ID: `C07VBSHV7EV`)
- `/lorem-ipsum` — Generates filler text for context testing (up to 500K tokens)
- `/debug` — Debug utilities

External skills available to all:

- `/commit` — Git commit with safety protocols
- `/simplify` — Launches 3 parallel review agents (code reuse, quality, efficiency)
- `/loop` — Recurring task execution
- `/schedule` — Cron-scheduled remote agents
- `/claude-api` — Help building with the Anthropic SDK

Notable prompt details:

- `/simplify` literally instructs Claude to launch three review agents concurrently in a single message: reuse, quality, and efficiency.
- `/stuck` tells Claude to inspect `ps`, `pgrep -lP`, `~/.claude/debug/<session-id>.txt`, and optionally `sample <pid> 3`, then post a one-line top-level Slack message plus a threaded diagnostic dump.
- `/schedule` explicitly cannot delete triggers from the CLI and instead directs users to `https://claude.ai/code/scheduled`.

### 13. Sleep Prevention

**`src/services/preventSleep.ts`** — On macOS, Claude Code spawns `caffeinate -i` to prevent the system from sleeping during long operations. Uses a reference-counting pattern and auto-restarts every 4 minutes (5-minute timeout as self-healing if the Node process is killed).

Source refs: `src/services/preventSleep.ts`.

The implementation also `unref()`s both the `caffeinate` child process and the restart interval, and uses `SIGKILL` for immediate teardown.

### 14. Sandbox Security

**`src/tools/BashTool/bashSecurity.ts`** — Extensive shell command security:

Source refs: `src/tools/BashTool/bashSecurity.ts`.

- Blocks Zsh-specific attacks: `zmodload` (gateway to dangerous modules), `emulate -c` (eval equivalent), `sysopen`/`syswrite`, `zpty`, `ztcp`
- Blocks command substitution patterns: `$()`, `${}`, process substitution, Zsh equals expansion
- Blocks PowerShell comment syntax as defense-in-depth
- Validates heredocs in substitutions
- Detailed allow/deny filesystem and network configs for sandboxed execution

### 15. The "Capybara" Model Codename

Dedicated deep-dive: [CAPYBARA.md](CAPYBARA.md)

Comments throughout reference model launches with `@[MODEL LAUNCH]` markers. Comments mention:

Source refs: `src/constants/prompts.ts`, `src/query.ts`.

- "capy v8 thoroughness counterweight" and "capy v8 assertiveness counterweight" — suggesting "Capybara" is the internal codename for a model version
- "False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)" — internal quality metrics
- "Remove this section when we launch numbat" — another internal codename
- "Tengu" — the Claude Code project's internal codename (used in all analytics events)

### 16. `/buddy` — April 1 Easter Egg Turned Full Companion System

Dedicated deep-dive: [BUDDY.md](BUDDY.md)

**`src/buddy/useBuddyNotification.tsx`** — Claude Code contains a hidden `/buddy` companion feature with an explicit teaser window:

- Hardcoded teaser dates: **April 1-7, 2026**
- Uses **local time, not UTC** for a rolling timezone-based launch
- Comments explicitly mention **"sustained Twitter buzz"** and **"gentler on soul-gen load"**
- The teaser is a rainbow `/buddy` startup notification for users without a companion yet
- The command stays live after the teaser window

The surrounding `src/buddy/` code shows this was not a one-line joke:

- deterministic pet generation from user identity
- rarity/species/hat/shiny/stats game logic
- a stored **model-generated soul**
- animated sprites, reactions, and `/buddy pet` hearts
- prompt/model-context integration via `companion_intro`
