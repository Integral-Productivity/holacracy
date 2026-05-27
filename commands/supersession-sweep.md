---
description: Sweep recent or session-filed tensions for supersession (S.5.5.1d). For each pair where one tension subsumes another, offer to archive the subsumed one or merge specifics into the surviving body.
argument-hint: [scope: "session" | "recent" | circle name, optional; default "session"]
---

# /holacracy:supersession-sweep

End-of-session deduplication for filed tensions. Applies the constitutional supersession test from S.5.5.1d -- *"would the tension still exist if the other were resolved?"* -- to pairs of tensions in scope, and offers to archive or merge subsumed ones.

This command is also offered implicitly by the `holacratic-ai-governance` skill when Claude detects session-closing signals ("done for now", "that's it for today"). Invoking it explicitly is an override or a manual trigger.

## What this command does

1. **Resolve scope from $ARGUMENTS.** Default is `session`: tensions filed during the current Claude session (Claude maintains this list internally as it dispatches the `tension-capture` subagent). Alternatives:
   - `recent` -> tensions with `status: "unprocessed"` on any role the actor fills (last 30 days, ranked newest first).
   - A circle name -> unprocessed tensions on roles the actor fills in that circle.
2. **Load the in-scope tensions.** For `session`, use the internally tracked list. For `recent` or a named circle, resolve the actor's roles via `glassfrog_list_my_roles` and then `glassfrog_list_role_tensions(role_id, status: "unprocessed")` for each.
3. **Apply the supersession test** (`skills/shared/tension-triage.md` Step 3) across all candidate pairs:
   - For each pair (A, B), ask the substantive question: "Would Tension A still be a felt gap if Tension B were resolved?"
   - If no -> A is superseded by B. Flag the pair.
   - If yes for both directions -> both stand; do not flag.
4. **Present flagged pairs to the user one at a time:**

   ```
   Supersession candidate

     Surviving:  [B body excerpt]    (ten_yyy on [Role/Circle])
     Subsumed:   [A body excerpt]    (ten_xxx on [Role/Circle])

   Rationale: [why A appears subsumed by B]

   Action:
     [a] archive A   -> update_tension(ten_xxx, status: "archived")
     [m] merge into B -> update_tension(ten_yyy, body: "<original B body>\n\nAlso: <A specifics>") and archive A
     [k] keep both    -> no action
     [r] reverse (subsume B into A) -> swap roles, retry
   ```

5. **Apply the user's decision** via `glassfrog_update_tension` calls. For `m` (merge), the merge is two calls: update B's body, then archive A.
6. **Summarize at the end.** Number of pairs detected, number archived, number merged, number kept.

## Behaviour

- **Per-pair decision.** Never batched. The user can quit at any time and remaining pairs are left alone.
- **Conservative bias.** When in doubt, recommend keeping both. Two tensions describing the same gap from different angles can both be useful for the role-filler's own clarity; only collapse when one truly subsumes the other.
- **Asymmetric pairs only.** Supersession is directional: A subsumed by B is not the same as B subsumed by A. The command surfaces the directional pair; the user can flip it with `r`.
- **Cross-circle supersession is rare but valid.** A tension on one role can sometimes be made moot by a tension on a sister role in another circle. The sweep does not artificially limit to one circle unless the user scopes it.
- **Constitutional grounding.** The S.5.5.1d test exists in the Constitution as the criterion for whether an *objection* is genuinely independent of a *proposer's tension*. The same logic transposes cleanly to whether two tensions in an inbox are independent.

## When the implicit offer fires

The `holacratic-ai-governance` skill teaches Claude to recognize session-closing signals -- "done for now", "that's it", "closing out", "wrapping up" -- and offer the sweep before retro/closing:

> *"Before we close, would you like me to sweep the tensions filed this session for supersession?"*

If the user assents, this command runs with default scope (`session`). If they decline, no action. The implicit offer is silent when no tensions were filed in the session.

## What this command does NOT do

- It does not delete tensions. Archive is the only collapse mechanism.
- It does not process tensions (`status: "processed"`) -- that's for `/holacracy:process-inbox` and only in meeting-day catch-up.
- It does not file new tensions; if the sweep surfaces a *meta*-tension ("the inbox keeps getting these overlapping items because [structural reason]"), this command will note it but not file it -- it's a job for `/holacracy:capture-tension` afterwards.
- It does not run automatically. Implicit offer + explicit command, both opt-in.
