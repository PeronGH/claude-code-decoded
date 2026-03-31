# Buddy

## The short version

Claude Code contains a hidden companion system built around `/buddy`, and it looks very much like an April Fool's rollout that turned into a permanent feature.

The strongest evidence is in `src/buddy/useBuddyNotification.tsx`, which hardcodes:

- a **teaser window: April 1-7, 2026**
- a **local-time rollout**, not UTC
- explicit comments about:
  - "24h rolling wave across timezones"
  - "sustained Twitter buzz"
  - "gentler on soul-gen load"

That is not a generic UI experiment. It is an Easter egg launch plan.

## Why this is interesting

The obvious reading from the README would be "Claude Code has a mascot sprite."

The code says something much more specific:

- there was a date-gated teaser campaign
- it was tuned for social buzz
- it involved "soul generation"
- it stores a model-generated personality
- it animates in the footer and fullscreen UI
- it reacts to user interaction like petting

This is one of the clearest examples of Claude Code having a hidden playful feature that sits far outside the normal coding-assistant story.

## 1. The April 1 rollout is explicit

The giveaway is `src/buddy/useBuddyNotification.tsx`.

Two functions matter:

- `isBuddyTeaserWindow()`
- `isBuddyLive()`

The comments say:

- local date, not UTC
- the teaser window is **April 1-7, 2026 only**
- the command stays live forever after

The teaser logic:

- if `feature('BUDDY')` is on
- and the user has no companion yet
- and the date is in the teaser window
- show a rainbow `/buddy` startup notification

The code literally renders a rainbow `/buddy` teaser.

That is almost certainly the "fool's day easter egg" you were thinking of.

## 2. It is not just a teaser, it is a full companion system

The `src/buddy/` directory is substantial:

- `src/buddy/companion.ts`
- `src/buddy/types.ts`
- `src/buddy/sprites.ts`
- `src/buddy/CompanionSprite.tsx`
- `src/buddy/prompt.ts`
- `src/buddy/useBuddyNotification.tsx`

This is not throwaway novelty code. It has persistent state, deterministic generation, animation, prompt integration, and UI behaviors.

## 3. The companion has "bones" and a "soul"

`src/buddy/types.ts` is the funniest file in the feature.

It splits the companion into:

- deterministic **bones**
- stored **soul**

The comments say:

- bones are deterministic and derived from the user
- the soul is **model-generated**
- the soul is stored after the first hatch

The type split is:

- `CompanionBones`
- `CompanionSoul`
- `Companion`
- `StoredCompanion`

So the pet is not purely random and not purely persistent:

- the structural traits come from deterministic user-derived generation
- the personality/name layer is generated and stored

That is much more elaborate than a static mascot.

## 4. The pet is generated from user identity

In `src/buddy/companion.ts`:

- the user identity comes from account UUID or fallback user ID
- a deterministic seeded PRNG generates rarity, species, eyes, hat, shininess, and stats
- the salt is:

```text
friend-2026-401
```

That `401` looks like a wink at April 1.

The file also exposes an `inspirationSeed`, suggesting the deterministic roll was probably used to condition the model-generated "soul."

## 5. It has collectible-game logic

`src/buddy/types.ts` and `src/buddy/companion.ts` show the companion system is built like a lightweight collectible:

- rarity tiers:
  - common
  - uncommon
  - rare
  - epic
  - legendary
- species list
- eyes
- hats
- shiny chance
- stat spread with peak/dump stat logic

This is game design, not mere branding polish.

## 6. The sprite is animated and pettable

`src/buddy/CompanionSprite.tsx` shows real UI behavior:

- idle animation frames
- reaction bubbles
- fullscreen floating bubble mode
- footer mode
- pet burst hearts after `/buddy pet`
- auto-clearing reactions
- narrow-terminal collapse behavior

The code tracks:

- `companionReaction`
- `companionPetAt`

in app state.

So the companion is a living UI surface, not just a hatch-once joke.

## 7. The companion also reaches the model

`src/buddy/prompt.ts` injects a `companion_intro` attachment into model context when a companion exists and has not already been introduced.

So Buddy is not only frontend art. Claude gets informed about the companion too.

That makes the system feel intentionally "in-universe" rather than purely decorative.

## 8. Prompt input explicitly knows about `/buddy`

The teaser is not isolated.

There is `/buddy` handling spread across the UI:

- `src/components/PromptInput/PromptInput.tsx`
  - rainbow highlighting for `/buddy`
  - a footer item that can submit `/buddy`
- `src/screens/REPL.tsx`
  - companion rendering in the footer/fullscreen layout
- `src/state/AppStateStore.ts`
  - dedicated companion state

So the command is woven into the main terminal UX.

## 9. One especially funny side effect: Capybara collision

The buddy species list includes `capybara`, and `src/buddy/types.ts` has to hide that literal behind `String.fromCharCode(...)` because it collides with the internal model-codename canary list.

So the repo contains this bizarre overlap:

- `Capybara` the internal hidden model codename
- `capybara` the hidden buddy species

And the mascot implementation had to tiptoe around the model leak detector.

That is exactly the kind of weird residue you only get in a real internal codebase.

## Best read

The most interesting interpretation is:

> Anthropic planned `/buddy` as a playful April 1, 2026 teaser campaign, but the codebase behind it is big enough that it looks like a genuine long-lived hidden companion feature rather than a one-day prank.

The date-gating is the Easter egg.
The surrounding code says the joke had budget.

## The punchline

Claude Code does not just have an April Fool's comment.

It has a full companion system with:

- an April 1-7 teaser window
- social-rollout comments
- deterministic pet generation
- a model-generated personality layer
- animated sprites
- pet reactions
- model-context integration

That is one of the clearest "this product has a secret life" artifacts in the recovered source.
