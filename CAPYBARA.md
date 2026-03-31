# Capybara

## The short version

`Capybara` is not a random mascot name that happened to leak into Claude Code.

In the recovered Claude Code 2.1.88 source, Capybara shows up in exactly the places you would expect a **real internal model track** to show up:

- Anthropic-only model alias resolution
- context-window and max-token overrides
- thinking / effort defaults
- model-specific prompt tuning
- model-specific stop-sequence bugs and mitigations
- codename masking and anti-leak logic

The code reads like Anthropic had a serious internal model family called `Capybara`, with variants like `capybara-fast` and `capybara-v2-fast[1m]`, and the team spent time hardening Claude Code around its behavior.

Then, in late March 2026, public reporting starts surfacing a new unreleased Anthropic name: `Mythos`.

The interesting read is:

> Capybara looks like a scrubbed internal codename for Anthropic's next model line, and Mythos looks very plausibly like the public-facing name, successor name, or adjacent tier name that surfaced later.

That exact mapping is not proven by the code alone. But the code absolutely does show that Capybara was real, important, and sensitive enough that Anthropic built leak-prevention machinery around it.

## Why this is interesting

If Capybara were just a harmless internal joke, you would expect to see it once or twice in comments.

Instead, the code shows all of this:

- Anthropic-only aliases like `capybara-fast`
- a `capybara-v2-fast[1m]` model string shape
- Capybara-specific prompt counterweights
- Capybara-specific false-claim metrics
- Capybara-specific stop-sequence bugs
- UI masking of internal model codenames
- attribution fallback logic to avoid leaking hidden model names
- even a buddy sprite workaround because `capybara` was apparently on a codename canary list

That is not background noise. That is the footprint of a model that mattered internally.

## The smoking guns in the code

### 1. Undercover mode explicitly treats Capybara as sensitive internal model intel

`src/utils/undercover.ts` is the first giveaway.

Undercover mode tells Anthropic employees not to leak:

- internal model codenames
- unreleased model versions
- internal repo names
- internal tools and Slack channels

And it uses `Capybara` as the example.

It literally includes a bad example:

> "Fix bug found while testing with Claude Capybara"

That is not abstract policy language. That is the sort of thing you write when people have actually been using that internal model name.

### 2. Capybara had Anthropic-only model aliases

The strongest operational evidence is in the model plumbing.

Relevant files:

- `src/utils/model/antModels.ts`
- `src/utils/model/model.ts`
- `src/main.tsx`
- `src/utils/permissions/permissionSetup.ts`

The code references these shapes:

- `capybara-fast`
- `capybara-v2-fast`
- `capybara-v2-fast[1m]`

Those are not public Claude API IDs like `claude-opus-4-6`. They belong to Anthropic's internal `antModels` system, which is loaded from the GrowthBook config `tengu_ant_model_override`.

That internal model config supports:

- alias
- backing model string
- label
- description
- default effort
- context window
- default max tokens
- upper max tokens limit
- always-on thinking

So Capybara was not just a hidden string. Claude Code had a whole data model for it.

### 3. Capybara had model-specific prompt counterweights

`src/constants/prompts.ts` is full of launch-scarring comments:

- "Update comment writing for Capybara"
- "capy v8 thoroughness counterweight"
- "capy v8 assertiveness counterweight"
- "False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)"
- "Remove this section when we launch numbat"

This is the most revealing section in the entire repo.

It says:

- there was a `Capybara v8`
- Capybara had known behavior issues
- Anthropic was actively bending the Claude Code system prompt to compensate
- those issues included:
    - over-commenting
    - not being assertive enough
    - not being thorough enough
    - a much higher false-claims rate than `v4`

This is not a hypothetical future placeholder. It reads like live launch support for a real model deployment.

## What kind of model was Capybara?

The code strongly suggests Capybara was not a toy internal checkpoint. It looks like a serious model family used in dogfooding and maybe in internal production environments.

Why:

- it had fast variants
- it had 1M-context variants
- it had dedicated effort / thinking / token limit hooks
- it had launch-specific prompt tuning
- it had enough usage to generate A/B measurements and failure-rate commentary

My read:

> Capybara was almost certainly an internal Claude model family or launch track, probably one aimed at higher-end reasoning / coding / agent behavior than the public line at the time.

The code cannot tell us the final product marketing name. But it absolutely tells us Capybara was treated like a real model line.

## The runtime scars are even better than the aliases

If you want the juiciest evidence, ignore the alias strings and read the bugs.

### 4. Capybara had stop-sequence / turn-boundary problems

Relevant files:

- `src/utils/messages.ts`
- `src/utils/toolResultStorage.ts`
- `src/query.ts`

The comments say things like:

- Capybara sampled an unwanted stop sequence when `tool_reference` expansions appeared at the prompt tail
- repeated `Human:` boundaries could teach "capy" to emit `Human:` and terminate early
- empty `tool_result` tails could make Capybara end its turn with zero output
- replaying protected-thinking signatures from a model like Capybara to an Opus fallback could 400

Those comments include A/B notes:

- `21/200 vs 0/200 on v3-prod`
- `92% -> 0%`

This is exactly what real model integration looks like: weird, ugly prompt-format paper cuts that only appear after enough internal usage.

### 5. Capybara affected real runtime behavior

Relevant files:

- `src/utils/context.ts`
- `src/utils/effort.ts`
- `src/utils/thinking.ts`
- `src/utils/model/modelOptions.ts`

Resolved ant-only models can override:

- context window
- default max tokens
- upper max tokens limit
- default effort
- always-on thinking

Anthropic-only users also get picker options built from the `antModels` payload.

So Capybara was plugged into the operational guts of Claude Code, not just the UI.

## Anthropic clearly did not want this codename leaking

This is where the story gets especially good.

### 6. The code masks internal codenames in UI

`src/utils/model/model.ts` has `maskModelCodename()`.

Example comment:

- `capybara-v2-fast -> cap*****-v2-fast`

If Anthropic users are on an internal model without a public display name, Claude Code masks the codename rather than showing it raw.

That is direct evidence of active codename hygiene.

### 7. Public attribution falls back to Opus to avoid leaking hidden names

`src/utils/attribution.ts` says that if the model is not recognized as public, external attribution falls back to:

- `Claude Opus 4.6`

The comment explicitly says this is to avoid leaking codenames.

That means Anthropic expected Claude Code to sometimes know about models it could not safely name in public.

### 8. Even the buddy sprite system had to tiptoe around Capybara

`src/buddy/types.ts` is one of the funniest clues in the repo.

It says one species name collides with a "model-codename canary" in `excluded-strings.txt`, so the code constructs `capybara` via `String.fromCharCode(...)` to keep the literal out of the bundle.

That means:

- `capybara` was sensitive enough to be on a leak-detection list
- that list was checking built artifacts
- and a completely unrelated mascot feature had to work around it

This is the kind of detail you only get when a codename has become operationally sensitive.

## The best theory

If I had to state it in interesting, readable terms:

> Capybara was Anthropic's hidden next-model codename inside Claude Code, complete with fast variants, 1M-context variants, prompt patches, and behavior-specific mitigations. Claude Code's source is effectively full of Capybara fingerprints, while the public-facing layers are doing everything they can not to say the name out loud.

More speculative, but plausible:

> Capybara was likely the internal name for a model line that later surfaced publicly as, or alongside, `Mythos`.

I would treat that second sentence as a strong read, not a proven fact.

## Mythos: the newer public leak

`Mythos` does **not** appear in this recovered code snapshot at all.

Codebase result:

- `rg -n -i "mythos" /home/peron/dev/claude-code/src /home/peron/dev/claude-code/vendor`
- no matches

But the late-March 2026 reporting makes the Capybara story more interesting, not less.

### Axios, March 29, 2026

Source:

- https://www.axios.com/2026/03/29/claude-mythos-anthropic-cyberattack-ai-agents

What matters:

- says Anthropic is privately warning officials about a not-yet-released model "currently branded `Mythos`"
- says Fortune obtained an unpublished Anthropic blog post describing Mythos as far ahead in cyber capabilities
- explicitly ties this to Anthropic's earlier public cyberattack disclosures

### The Information briefing, crawled March 28, 2026

Source:

- https://www.theinformation.com/briefings/anthropic-discusses-q4-ipo-preps-advanced-claude-mythos-capybara-ai/

What matters:

- the headline itself pairs `Claude Mythos` and `Capybara`
- the available summary says a leaked unreleased post described a new tier called `Capybara`, larger and more intelligent than Opus
- it also says the relationship between `Mythos` and `Capybara` is unclear

That ambiguity is actually the interesting part.

It suggests at least one of these is true:

- `Capybara` was the internal codename and `Mythos` is the external branding
- `Capybara` was the tier name and `Mythos` was a newer internal/public rebrand
- `Capybara` and `Mythos` refer to closely related efforts in the same launch wave

## The official Anthropic context that makes the leak believable

Anthropic's official post from **November 13, 2025** is here:

- https://www.anthropic.com/news/disrupting-AI-espionage

It does **not** mention Capybara or Mythos.

But it does matter because it shows Anthropic had already gone public months earlier about:

- Claude Code being used in a sophisticated cyber espionage campaign
- AI handling 80-90% of tactical operations
- agentic cyber misuse becoming a real-world problem

That makes the later `Mythos` reporting feel directionally consistent with Anthropic's own public cyber framing, rather than totally out of left field.

## What I think happened

If you want the most interesting version that still fits the evidence:

1. Anthropic had an internal model track called `Capybara`.
2. Claude Code integrated it deeply enough that the repo accumulated visible Capybara-specific launch scars.
3. Anthropic simultaneously built extensive anti-leak defenses so external builds would not expose the codename.
4. By late March 2026, a newer public leak starts surfacing `Mythos`.
5. The simplest read is that `Mythos` is the newer outward-facing name, successor name, or adjacent label for the same next-generation model push that internally showed up as `Capybara`.

## The punchline

The interesting story is not just "Anthropic had a codename."

It is this:

> Claude Code's recovered source contains the residue of an unreleased internal model line called Capybara: its aliases, its bugs, its prompt patches, its masking logic, and the company's attempts to keep the name out of public builds. Then, just as the code stops, the outside world starts hearing a different unreleased name: Mythos.

That is the kind of trail you only get when a codename was real.
