# Native Client Attestation

## Why this matters

Claude Code does not just send normal API requests. The recovered source shows a custom attribution-and-attestation mechanism designed to let Anthropic distinguish a real Claude Code client from something that is merely replaying its HTTP shape.

The interesting part is not just that there is a header. It is that the header is deliberately built so a native Bun layer can **rewrite the serialized request body in place** without changing `Content-Length`.

That is a very specific implementation choice, and it strongly suggests Anthropic wanted a client-integrity signal that is harder to fake from a generic wrapper.

## The core mechanism

The main implementation sits in `src/constants/system.ts`.

`getAttributionHeader(fingerprint)` builds a string like:

```text
x-anthropic-billing-header: cc_version=<version>.<fingerprint>; cc_entrypoint=<entrypoint>; cch=00000; cc_workload=<optional>;
```

The important fields are:

- `cc_version`
- `cc_entrypoint`
- optional `cc_workload`
- `cch=00000`

The code comment says:

- when `NATIVE_CLIENT_ATTESTATION` is enabled, the header includes a `cch=00000` placeholder
- Bun's native HTTP stack finds that placeholder in the serialized request body
- it overwrites the zeroes with a computed hash
- the server verifies that token to confirm the request came from a real Claude Code client
- the underlying implementation is in `bun-anthropic/src/http/Attestation.zig`

The explicit reason for the placeholder approach:

- same-length replacement avoids `Content-Length` changes
- same-length replacement avoids buffer reallocation

That is the key design detail.

## The request flow

### 1. A tiny fingerprint is computed from the first user message

In `src/utils/fingerprint.ts`, Claude Code computes a 3-hex-character fingerprint using:

- hardcoded salt: `59cf53e54c78`
- characters from the first user message at indices `4`, `7`, and `20`
- current CLI version

Algorithm:

```text
SHA256(SALT + msg[4] + msg[7] + msg[20] + version)[:3]
```

This is not the attestation token itself. It is a lightweight attribution fingerprint used in `cc_version=<version>.<fingerprint>`.

### 2. The fingerprint is inserted before synthetic system noise is added

In `src/services/api/claude.ts`, the code explicitly computes the fingerprint:

- after message normalization
- before deferred tool metadata and other synthetic system additions

The comment says this is so the fingerprint reflects the **actual user input**.

Then `getAttributionHeader(fingerprint)` is prepended as its own system block.

### 3. Side queries use the same mechanism

In `src/utils/sideQuery.ts`, side queries:

- extract the first user message text
- compute the same fingerprint
- build the same attribution header
- keep it in its own system block

The comment there says this separation prevents server-side parsing from accidentally treating later system text as part of `cc_entrypoint`.

So this is not just main-loop behavior. Anthropic threaded the same mechanism into side queries too.

## Why the system prompt carries a fake header-looking line

This is one of the odder implementation details.

The "header" is actually injected into the **system prompt blocks**, not attached as a normal HTTP header in the application code path shown here.

That means the string:

```text
x-anthropic-billing-header: ...
```

is part of request content that survives into serialization, where the Bun-native layer can patch the `cch=00000` substring in-place.

This is why the same-length placeholder matters so much.

It also explains the comments in:

- `src/utils/api.ts`

which have special logic for prompt blocks that start with `x-anthropic-billing-header`.

## What gets gated

This mechanism is not unconditional.

### Feature gate

`src/constants/system.ts` uses:

- `feature('NATIVE_CLIENT_ATTESTATION')`

So the placeholder only appears when that build flag is enabled.

### Killswitches

The broader attribution block can also be disabled:

- env var: `CLAUDE_CODE_ATTRIBUTION_HEADER`
- GrowthBook flag: `tengu_attribution_header`

That means Anthropic can remotely suppress the whole attribution block, including the `cch` placeholder, if needed.

## The extra fields matter too

The attestation field is the flashy part, but the surrounding fields are informative.

### `cc_version`

Built from:

- `MACRO.VERSION`
- the 3-character message-derived fingerprint

So it looks like:

```text
cc_version=2.1.88.abc
```

This gives Anthropic a cheap way to associate requests with:

- client version
- a tiny stable signal derived from the user's first message

### `cc_entrypoint`

Derived from:

- `process.env.CLAUDE_CODE_ENTRYPOINT ?? 'unknown'`

This lets the server distinguish how the client was invoked.

### `cc_workload`

Derived from:

- `getWorkload()`

The comments say this is a turn-scoped hint so Anthropic can route things like cron-initiated work to different QoS pools.

This field is referenced in:

- `src/types/textInputTypes.ts`
- `src/cli/print.ts`
- `src/hooks/useScheduledTasks.ts`

So the billing-header block is doing more than anti-spoofing. It is also carrying internal routing metadata.

## The strongest implication

The strongest read from this code is:

> Anthropic wanted a signal that a request came from the real Claude Code native client, and they implemented it below the JavaScript layer so the final token is produced by their custom Bun HTTP stack rather than by normal app code.

That is a very different posture from "we add a user-agent and hope for the best."

It suggests Claude Code requests are expected to carry provenance that:

- includes app-level metadata
- includes a lightweight prompt fingerprint
- and, when enabled, includes a native attestation token that JS never directly computes

## What the code does not show

The recovered source gives the client-side contract, but not the native implementation.

It does **not** show:

- the Zig attestation code
- the exact hash algorithm used for `cch`
- what bytes are covered by the attestation
- what keying material or trust root the server uses
- whether this is advisory telemetry or a hard enforcement path

The code comment names `bun-anthropic/src/http/Attestation.zig`, but that file is not in the recovered tree.

So the interesting confirmed fact is the **shape** of the protocol:

- JS emits a placeholder
- native Bun rewrites it in-place
- server verifies it

The exact cryptographic implementation is outside the recovered package.

## The weirdest detail

The weirdest and most revealing detail is that the placeholder is exactly `00000`.

Not empty.
Not variable-length.
Not appended later.

A fixed-width five-character slot.

That tells you the transport-level patching behavior was designed first, and the JS-side string format was written to fit it.

That is the sort of scar you only get when product code and a custom runtime are co-designed.
