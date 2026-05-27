---
description: Capture a Holacratic tension to GlassFrog via the draft-and-confirm flow. Resolves sensing role, applies role-vs-person triage, drafts the body, and files with a single per-tension confirmation.
argument-hint: [tension text, optional]
---

# /holacracy:capture-tension

On-demand entry to the canonical tension capture flow. Dispatches the `tension-capture` subagent, which runs the full B-flow specified in `skills/shared/tension-capture-flow.md`.

## What this command does

1. **Parse $ARGUMENTS.** If the user passed tension text inline, use it as the seed. If not, ask: *"What tension would you like to capture?"* and wait for input.
2. **Dispatch the `tension-capture` subagent** with the tension text and the dispatch source (`explicit command`). Let the subagent handle Steps 2–8 of `skills/shared/tension-capture-flow.md`:
   - Resolve sensing role via `glassfrog_get_me` + `glassfrog_list_my_roles` (with circle narrowing per conversation context).
   - Apply the role-vs-person triage gate from `skills/shared/tension-triage.md` Step 1.
   - Draft the body, preserving the user's own words.
   - Suggest a `meeting_type` (governance vs. tactical) per `skills/shared/tension-triage.md` Step 2.
   - Present the per-tension confirmation block.
   - On approval, call `glassfrog_create_tension(role_id, body, status: "unprocessed")` and follow up with `glassfrog_update_tension` if a `meeting_type` or `label` was specified.
3. **Surface the subagent's structured result** (tension ID, role + circle, meeting type) and return to the original conversation.

## Behaviour

- This command captures **one tension per invocation**. If the user wants to capture multiple, run it multiple times.
- If the role-vs-person triage gate refuses to file (person tension), this command reports the refusal and the suggested IDR route -- it does not silently swallow the request.
- If GlassFrog is not connected, the subagent will offer to draft a plain-text version for manual entry. This command honours that fallback.
- The constitutional safeguard from `skills/shared/tension-capture-flow.md` applies: no file without explicit per-tension confirmation.

## When to use this command vs. ambient capture

- **Use this command** when the user is explicitly ready to file a tension and wants the structured flow.
- **Use ambient detection** (in `holacratic-ai-governance`) when tension language surfaces during other work and Claude should offer to capture without the user having to remember a slash command.

Both paths dispatch the same `tension-capture` subagent, so the behaviour is identical from Step 2 onward.

## What this command does NOT do

- It does not process tensions (no `meeting_type: processed` writes).
- It does not delete tensions.
- It does not file proposals or make governance changes.
- It does not batch multiple tensions into one confirmation.

For inbox review and routing of already-filed tensions, use `/holacracy:process-inbox`. For end-of-session deduplication, use `/holacracy:supersession-sweep`.
