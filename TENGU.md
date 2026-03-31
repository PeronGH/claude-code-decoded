# Tengu

## The simple version

`Tengu` is not just the analytics prefix in Claude Code.

In the recovered source, `tengu_*` names show up across:

- telemetry events
- GrowthBook flags
- dynamic configs
- sink killswitches
- model overrides
- auto-dream scheduling
- Remote Control entitlement
- cron / scheduled task rollout
- settings sync
- attribution headers

The interesting read is:

> `Tengu` looks like the internal umbrella namespace for Claude Code's product-control plane: analytics, experiments, rollout gates, and operational knobs all live under the same naming scheme.

It is bigger than "Datadog event names start with `tengu_`."

## Why this matters

If `Tengu` were only a telemetry codename, you would expect to see it mostly in `logEvent(...)` calls.

Instead, the codebase uses `tengu_*` names to decide:

- who gets Remote Control
- which internal models Anthropic employees can resolve
- whether background memory consolidation runs
- whether cron scheduling is enabled and how much jitter it gets
- whether attribution headers stay on
- whether settings sync uploads
- whether verification agents are required
- whether individual analytics sinks are killed

That makes `Tengu` look less like a single subsystem and more like Claude Code's internal product namespace.

## 1. Yes, Tengu is the telemetry codename

The obvious part is real.

The telemetry stack uses `tengu_*` everywhere:

- `src/services/analytics/datadog.ts`
- `src/services/analytics/firstPartyEventLogger.ts`
- `src/services/analytics/index.ts`

Examples:

- `tengu_api_success`
- `tengu_query_error`
- `tengu_started`
- `tengu_tool_use_success`
- `tengu_team_mem_sync_push`
- `tengu_model_fallback_triggered`

There is also event sampling via:

- `tengu_event_sampling_config`

So the README's current claim that Tengu is the internal telemetry codename is definitely correct.

## 2. But Tengu also names the experiment and rollout layer

This is the more interesting part.

`src/services/analytics/growthbook.ts` is the central GrowthBook integration, and `tengu_*` is all over it:

- feature overrides
- cached feature values
- experiment exposure logging
- refresh listeners for long-lived config consumers

The code is effectively built to consume a stream of `tengu_*` gates and configs.

That means `Tengu` is not just the name of emitted events. It is also the namespace of the values that shape runtime behavior.

## 3. Tengu controls internal model resolution

One of the strongest signals is:

- `tengu_ant_model_override`

Relevant files:

- `src/utils/model/antModels.ts`
- `src/main.tsx`
- `src/utils/effort.ts`
- `src/utils/permissions/yoloClassifier.ts`

This config controls Anthropic-only model aliases and model metadata. That includes:

- internal aliases
- labels
- descriptions
- context windows
- default effort
- always-on thinking

This is the same mechanism that surfaces internal model names like `capybara-fast`.

So `Tengu` is not merely watching the product. It is actively deciding what model Anthropic employees can run.

## 4. Tengu gates major product features

Some of the highest-signal examples:

### Remote Control

- `tengu_ccr_bridge`

Used in:

- `src/bridge/bridgeEnabled.ts`

This gate controls whether a claude.ai subscriber is entitled to Remote Control / bridge mode.

There are related bridge gates too, like:

- `tengu_ccr_bridge_multi_session`

### Verification agent requirement

- `tengu_hive_evidence`

Used in:

- `src/constants/prompts.ts`
- `src/tools/AgentTool/builtInAgents.ts`
- `src/tools/TodoWriteTool/TodoWriteTool.ts`
- `src/tools/TaskUpdateTool/TaskUpdateTool.ts`

This is the gate that turns on the hard "independent adversarial verification before reporting completion" contract.

### Auto-dream / memory consolidation

- `tengu_onyx_plover`

Used in:

- `src/services/autoDream/config.ts`
- `src/services/autoDream/autoDream.ts`

This one controls whether background "dream" memory consolidation runs and what thresholds it uses.

### Settings sync

- `tengu_enable_settings_sync_push`

Used in:

- `src/services/settingsSync/index.ts`

So whether settings get uploaded in the background is also a Tengu knob.

### Attribution header

- `tengu_attribution_header`

Used in:

- `src/constants/system.ts`

This controls whether Claude Code emits the attribution / attestation header block at all.

## 5. Tengu also carries ops knobs, not just user-facing features

The scheduled-task system is a good example.

Relevant files:

- `src/utils/cronScheduler.ts`
- `src/utils/cronJitterConfig.ts`
- `src/tools/ScheduleCronTool/prompt.ts`

Important config names:

- `tengu_kairos_cron`
- `tengu_kairos_cron_durable`
- `tengu_kairos_cron_config`

These do not just enable cron features. They also control:

- whether running schedulers stop mid-session
- jitter windows
- rollout behavior
- durable scheduling behavior

That is classic ops-control-plane material.

## 6. Tengu can even kill its own analytics sinks

The funniest example is probably:

- `tengu_frond_boric`

Used in:

- `src/services/analytics/sinkKillswitch.ts`

This is a GrowthBook JSON config that can disable individual analytics sinks:

- `datadog`
- `firstParty`

So the Tengu namespace includes the switch that can partially turn Tengu itself off.

That kind of self-referential control is usually a sign of a mature internal experimentation / reliability setup.

## 7. First-party event logging is a serious subsystem

If you only read the Datadog path, you miss the bigger story.

Relevant files:

- `src/services/analytics/firstPartyEventLogger.ts`
- `src/services/analytics/firstPartyEventLoggingExporter.ts`

The first-party path includes:

- OpenTelemetry-based batching
- retry with backoff
- failed-event append-only storage on disk
- auth-aware export
- GrowthBook experiment logging
- dynamic batch config:
  - `tengu_1p_event_batch_config`

This is not "throw JSON at a logging endpoint." It is a fairly deliberate internal logging pipeline.

## 8. Tengu is broad enough that it probably names the project, not just one subsystem

This is the main conclusion.

From the code alone, the strongest interpretation is:

> `Tengu` is the internal namespace for Claude Code's experimentation, telemetry, and operational feature-control system.

Why that read holds up:

- event names are `tengu_*`
- dynamic configs are `tengu_*`
- rollout gates are `tengu_*`
- model override configs are `tengu_*`
- bridge entitlements are `tengu_*`
- memory features are `tengu_*`
- cron controls are `tengu_*`
- sink killswitches are `tengu_*`

That is too broad to be accidental naming drift.

## Good examples to remember

If you want the shortest list of "wow, Tengu really is everywhere" examples:

- `tengu_ant_model_override`
  - internal Anthropic model aliases and runtime defaults
- `tengu_ccr_bridge`
  - Remote Control entitlement
- `tengu_onyx_plover`
  - dream / memory-consolidation rollout and thresholds
- `tengu_kairos_cron`
  - scheduled-task rollout
- `tengu_enable_settings_sync_push`
  - settings-sync upload
- `tengu_attribution_header`
  - billing / attribution header enablement
- `tengu_frond_boric`
  - analytics sink killswitch
- `tengu_event_sampling_config`
  - event sampling

That is a product-control namespace, not just a telemetry label.

## The punchline

The interesting update to the README is not "Tengu is telemetry."

It is this:

> Tengu appears to be Claude Code's internal control-plane codename: the namespace Anthropic uses for telemetry, experiments, rollout gates, entitlement checks, operational tuning, and internal model overrides.

That is much more revealing than the current one-line summary.
