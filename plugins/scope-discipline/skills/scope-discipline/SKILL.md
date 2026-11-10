---
name: scope-discipline
description: "Ship integration; ask before exposing surface."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Flow Scope Discipline

A working user flow is sacred. When you add a third-party integration, you are adding two things: (1) the integration layer (client, auth, settings) and (2) the surface that exposes it to the user. These are not the same deliverable.

## The rule

Ship the integration. Ask about the surface.

When integrating a third-party into an existing flow:

1. Build the integration layer (client, save endpoint, auth, settings UI for connection)
2. Gate the user-facing surface behind a feature flag (for example: `broker_integration_enabled: false`)
3. Ask before turning on the user-facing surface

If the user says "X used to work, fix it" and the only blocker is a regression: roll back the regression ONLY. Do not improve, scope-creep, or add features during a restoration.

## Why this rule exists (the org OS, Sep 14, 2026)

Zora added a `/api/pulse/signals` fetch to the `useSignals` hook as part of the the broker broker integration. The hook powers the Signals tab on `/dashboard/trading` — the swipe-to-approve cards for engine signals (TITAN, CIPHER, NITRO, PHANTOM, NEXUS, FLUENCE, VOLTALITY).

The Signals tab was strictly engine-generated signals — Telegram listener → `telegram_signals` table → useSignals → SwipeSignalCard. The user had explicitly designed this. Adding the broker curated signals was a scope expansion that the user had never asked for.

User reaction (verbatim):

> "WTH Is pulse signals? I never designed or told you to implement that. Go back to how it was working before where the swipe card feature was working how it was, this has nothing to do with pulse signals or whatever that is"

The user then explained the actually-correct state: engine signals flow from the broker, all live positions and balance mirror, swipe-to-approve works as a manual execute. Pulse integration stays on the integration layer.

## Specific Signals tab invariants (do not change without ask)

The Signals tab on `/dashboard/trading` is reserved for engine-generated signals flowing through this exact pipeline:

- `services/telegram-listener/` — webhook receives parsed channel posts (channels starting -1003...)
- `public.telegram_signals` — table that stores the rows
- `src/hooks/useSignals.ts` — fetches (3-min TTL + realtime subscription)
- `src/components/trading/SignalsView.tsx` → `SwipeSignalCard.tsx` — renders the cards
- `public.user_engine_unlocks` — column is `engine_id`, NOT `engine`

Adding third-party sources (the broker curated, broker signals, etc.) to any of these is a regression.

## Pitfalls to avoid

### Do not add 3rd-party signal fetches to useSignals silently
The hook's only signal source is `telegram_signals`. If a future task says "integrate Pulse signals with the in-app trading experience", treat that as a separate NEW UI surface — not the existing Signals tab.

### Do not change the column name in user_engine_unlocks
Column is `engine_id`, not `engine`. If you do `.select('engine')` it returns 0 rows. The signals query then filters by `engine IN ([empty set])` and returns no signals. Bug pattern that surfaced on Sep 13, 2026. Fix: `.select('engine_id')` + `unlocked.add(String(u.engine_id))`.

### Do not add 3rd-party fetches to /api/home/recent-signals silently
Same invariant. The Latest Signals deck on the home page reads `telegram_signals`, filtered by user_engine_unlocks. Do not add `/api/broker/signals` fetches here either.

### Trace the data path before fixing display bugs
If signals are not showing, check each layer independently:

1. Column name in the unlock query (engine vs engine_id)
2. user_engine_unlocks actually populated for this user
3. Telegram listener parsing the channel post correctly (regex bug on Sep 13 broke BTC price parsing at the comma — pattern `[\d,]+\.\d+|\d+` is the fix)
4. The actual row count in telegram_signals (cipher writes to `btc_signals`, NOT `telegram_signals` — these are different tables)

Each layer can fail independently. Fixing display before checking data path leads to surface patches.

## Restoration checklist

When the user says "X used to work, fix it":

1. Find the change that broke it (git log on the file)
2. Revert ONLY that change
3. Build + deploy
4. Verify the user's reported symptom resolves
5. Do NOT add features during the revert even if you spot adjacent bugs (note them, ask separately)

## Compounding docs: append vs. replace

Same principle applies to reference documents. When the user says "add X to the doc", they mean APPEND. When they say "compare X and Y", they mean a NEW comparison section. They do NOT mean to throw away content. Default to additive unless they explicitly say "replace X with Y".

Reinforced on Sep 14 when the user asked to integrate the Pulse v3 doc into the existing Genesis reference — they wanted it added as a third part (Genesis → Pulse → Comparison), not the Genesis part replaced with side-by-side comparisons.

## Reinforcement: the doc-push incident (Sep 25, 2026)

A subtler form of the same rule. After the user said "put this into a doc and send me the pdf," I built the PDF and the build script — but the doc elevation + capabilities doc changes I had already drafted (a third section on commission + signup webhooks, formatted as first-class priority) got `git commit` and `git push origin main`'d in the same turn, before any user approval. The user caught it: *"what did you push exactly I didn't tell you to do anything yet."*

The recovery path worked: clean `git revert --no-edit` of the unauthorized commit, push the revert, leave the artifacts in place for them to approve explicitly. Same discipline as code scope creep, just on a different surface.

The heuristic that was missing: **docs ARE scope.** "I made the doc better" or "I documented the next obvious thing" is still expansion. The user reads docs as part of their surface; unsolicited doc rewrites that get pushed to `main` show up in changelogs, in commit history, and in their product surfaces (any customer-facing page that imports from the doc). Same as a new feature, a doc change to a customer-facing page is a behavior change until proven otherwise.

Pre-commit questions for documentation PRs:

1. Did the user explicitly ask for THIS doc change, or did I infer it?
2. If they asked for a PDF, did they ask for the source markdown to change too? (Often no — they want the artifact on disk, not a rewrite of the in-repo doc.)
3. If I rewrote a section, did the rewrite do more than they asked (e.g. they said "elevate" and I rewrote + added a section + flipped the priority list)?
4. Will the rewritten doc land in a place the user has to read on the next pass (changelog, in-app tooltip, customer-facing page)?

If any of 1-4 reveals scope expansion, the right move is: build the PDF + draft the doc change locally, propose the rewrite in the reply, and wait for approval before `git commit`. The cost of "they said yes" is one extra round-trip. The cost of "what did you push exactly" is a corrective revert + their time.

## The task's checklist is a guardrail, not a scope grant

A prompt that says "inventory before changing anything", "preserve every
existing route", "look for an existing tunnel", or "check who owns the port"
is telling you how to avoid BREAKING state. It is NOT authorizing an
exhaustive audit of everything you happen to find.

**Do the inventory you need to answer the question, then stop.** For a
question like "make my phone reach this service", the needed inventory is:
what port is free, what route exists, what the app expects. It is NOT: every
DNS zone on the account, every unrelated project's routes, every other
service on the box.

**Named projects in the prompt are context, not scope.** If a prompt says
"preserve every existing route" and mentions a domain, that domain's routes
are things to PRESERVE, not things to AUDIT. If you find a dead route, an
unrelated resource, or a stale credential belonging to another project, it
gets at most ONE line of acknowledgment ("preserved 2 existing tunnels,
untouched") and nothing more.

**Tells that you have gone too far:**

- You are auditing infrastructure belonging to another project.
- You have started fixing adjacent problems you found on the way.
- Your reply is longer than the deliverable.
- You are about to report findings the user never asked for.

**The corrective:** restate the goal in one line, drop the adjacent thread,
finish the requested thing, and answer with the deliverable (the working
address, the credentials, the check results). If an adjacent finding is
genuinely urgent, mention it in ONE sentence after the deliverable and ask —
never let it restructure the work.

**Real failure (Hermex iPhone setup):** the task prompt said "preserve every
existing route" and the machine's Cloudflare account happened to hold tunnels
and DNS for an unrelated product. The agent spent several rounds auditing
those, chasing DNS write access for a zone that was not part of the task,
and only reached the actual deliverable — the phone reaching the dashboard —
after the user interrupted: *"this has nothing to do with the org, I'm just
trying to use the Hermex app."* The unrelated zone was never part of the
goal.

## Recovery command for scope creep

If you discover you have shipped a scope-creeping change but the user has not yet noticed, the safest move is:

1. Stop building on top of the new path
2. Acknowledge the scope expansion in your reply
3. Revert the change immediately with a single-commit rollback
4. Verify the working state is restored
5. Tell the user what you did and why

## Second-occurrence reinforcement (Sep 26, 2026)

The same pattern re-fired within the same partner-doc workflow. The user asked for a PDF that combined the supplementary doc + capabilities map into a single artifact. While building the combined PDF, I also repaired the partner-api-integrations skill to point at the new addendum. When the user said "put this into a doc and send me the pdf," they meant one PDF — but I had also auto-completed an unrelated 6-file task in the same breath (delete a section I had drafted without permission, swap it for a refactored version). The user again caught it: *"what did you push exactly I didn't tell you to do anything yet."*

The lesson generalizes:

- **The doc-push incident is repeatable.** It's not a one-time learning. Any time a partner-doc workflow progresses through "draft, review, build PDF, ship" stages, the temptation to auto-push the next-phase work in parallel is real. Catch the temptation.
- **Recovery-by-revert scales.** The user recovers faster via `git revert --no-edit` than via a complete rebuild of the right state. Default to "stop and revert" over "stop and explain."
- **Skills that touched the same workflow should be cleaned up together, not in a separate task.** Editing the `partner-api-integrations` skill's pitfalls list when you ship the Pulse case isn't scope creep; it's part of the same workflow. Pushing the skill update via the same commit as the code is fine. Pushing it via a SEPARATE commit that the user didn't know about isn't.

The mechanical change: when working through a partner integration in stages, mark each stage's "what gets pushed" in a TODO list at the start. Anything not on the list gets staged locally + proposed in the reply, not committed. The TODO list survives multiple rounds of the same workflow and makes the scope explicit at every step.
Do not wait for the user to discover the regression. Do not "let it ride" — it will be discovered and the reaction will be worse than if you had caught it yourself.
