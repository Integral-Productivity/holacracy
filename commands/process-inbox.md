---
description: Walk through unprocessed tensions on the actor's roles and decide how to process each. Routes to tactical or governance meetings via meeting_type, archives false positives, or defers. Surfaces supersession candidates inline.
argument-hint: [circle name, optional]
---

# /holacracy:process-inbox

Review and triage the unprocessed tensions currently in your GlassFrog tension inbox across the roles you fill. This command does not *resolve* tensions -- resolution happens in tactical and governance meetings. It triages them: routes each to the right meeting type, archives the ones that no longer apply, and flags supersession candidates.

## What this command does

1. **Resolve actor + role roster.** Follow `skills/shared/actor-and-role-resolution.md` Steps 1–2: `glassfrog_get_me` for the actor, `glassfrog_list_my_roles` for their full role roster.
2. **Filter to a circle if $ARGUMENTS provided.** If the user named a circle, narrow the roster to roles in that circle. Otherwise, work across all the actor's roles.
3. **Fetch unprocessed tensions per role.** For each role in scope, call `glassfrog_list_role_tensions(role_id, status: "unprocessed")`. Aggregate the results into a single working list, annotated with the sensing role + circle.
4. **Run a quick supersession pre-scan.** Before walking each tension, apply `skills/shared/tension-triage.md` Step 3 across the working list to detect candidate pairs where one tension may be subsumed by another. Note these as supersession candidates to be raised when their primary surfaces.
5. **Walk each tension with the user.** For each tension in the working list, present a triage block:

   ```
   Tension [N of M] on [Role name] of [Circle name]
   Body:    [tension body]
   Filed:   [created_at, if available]

   [If a supersession candidate is flagged: "May overlap with: [other tension excerpt] (ten_yyy). Apply S.5.5.1d test?"]

   Process as:
     [g] route to governance  -> update_tension(meeting_type: "governance")
     [t] route to tactical    -> update_tension(meeting_type: "tactical")
     [a] archive              -> update_tension(status: "archived")
     [s] mark processed       -> update_tension(status: "processed") (only if already resolved outside the inbox)
     [d] defer / leave        -> no action
     [e] edit body            -> update_tension(body: ...)
   ```

6. **For each user decision, call the appropriate `glassfrog_update_tension`.** Apply `skills/shared/tension-triage.md` Step 2 (governance vs. tactical) as the decision guide when the user asks for a recommendation.
7. **At the end, summarize the session.** Number routed to governance, number routed to tactical, number archived, number deferred. Surface any supersession candidates the user did not act on (offer to run `/holacracy:supersession-sweep`).

## Behaviour

- This command operates on **filed** tensions. To capture a *new* tension, use `/holacracy:capture-tension`.
- Per-tension decision -- never batched. The user can quit at any point (`q`) and the remaining tensions stay unprocessed.
- The "mark processed" option (`s`) is for catch-up only: tensions that were resolved in a meeting but never marked processed in GlassFrog. Do not use it as a way to clear the inbox; that would lie about whether the tension was actually worked.
- The "archive" option (`a`) is the right path for false positives, no-longer-relevant tensions, and superseded ones. Archive is reversible (the tension still exists with `status: "archived"`); deletion is permanent and this command does not offer it.
- If `glassfrog_list_role_tensions` is unavailable (older MCP server), this command names the constraint and exits gracefully: *"Your GlassFrog MCP server doesn't expose tension listing yet -- you'll need to triage the inbox in the GlassFrog UI."*

## What this command does NOT do

- It does not resolve tensions on the user's behalf. Resolution is a meeting activity.
- It does not file proposals or make governance changes.
- It does not delete tensions (use the GlassFrog UI if a tension must be permanently removed).
- It does not assume `meeting_type` for the user -- it suggests, the user decides.

## Why this command exists

The GlassFrog tension inbox tends to grow when the practice of processing tensions falls behind the practice of sensing them. The fastest way for a busy role-filler to keep the inbox useful is to *triage* regularly: route each tension to the right meeting type so it appears on the right agenda, archive what no longer applies, and surface supersession before the inbox bloats with overlapping items.

This command is the operational surface for that practice.
