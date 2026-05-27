# 3. Adopt the GlassFrog tension API via a draft-and-confirm contract

Date: 2026-05-27

## Status

Accepted

## Context

The plugin's v0.2.0 docs explicitly stated that the GlassFrog API did not
support filing, reading, or processing tensions, and defended this as a
*principled boundary* -- not just a technical limitation. Specifically,
`skills/holacratic-ai-governance/SKILL.md` (line 41) listed "No tension
filing" under Critical API Constraints, and
`references/glassfrog-api-constraints.md` (lines 149-153, 178-182) gave
the architectural rationale: tensions are lived experiences, AI-filed
tensions would lack embodied context, and human governance must own
processing.

The GlassFrog MCP server now exposes a full tension CRUD:

- `glassfrog_create_tension(role_id, body, status)` -- with the
  asymmetric quirk that `label` and `meeting_type` are rejected on
  create and must be set with a follow-up update.
- `glassfrog_list_role_tensions(role_id, status, q, per_page, cursor)`
  -- returns Page<Tension>.
- `glassfrog_get_tension(tension_id)` -- single tension.
- `glassfrog_update_tension(tension_id, body?, label?, status?,
  meeting_type?)` -- routes (`tactical` | `governance`), archives,
  marks processed, edits body.
- `glassfrog_delete_tension(tension_id)` -- permanent.

The status enum is `unprocessed | processed | archived`.

This is a material capability shift. Continuing to claim "the API
doesn't support this" is no longer honest, and the existing Pattern 3
(Tension Sensing) -- which stops at drafting a text report -- now
*undershoots* what the API can do.

The question this ADR resolves is: how does the plugin adopt the
write capability without abandoning the principled boundary the v0.2
docs defended?

## Decision

The plugin adopts the GlassFrog tension API via a **draft-and-confirm
contract** with three distinct write privileges, ordered by autonomy:

1. **AI drafts; human confirms per-tension; AI files** as
   `status: "unprocessed"`. Routed to `meeting_type` (tactical or
   governance) via a follow-up update. This is the **v0.3 contract**
   and the only autonomous file path allowed.
2. **Human directs; AI executes** routing, archiving, or processing
   updates via `/holacracy:process-inbox` (one tension at a time, no
   batching).
3. **Human directs; AI executes** supersession-driven archiving via
   `/holacracy:supersession-sweep` (constitutional grounding: the
   S.5.5.1d objection-independence test transposes to inbox
   deduplication).

The full B-flow is specified in
`skills/shared/tension-capture-flow.md` and runs through the new
`agents/tension-capture.md` subagent. The role-vs-person triage gate
(`skills/shared/tension-triage.md` Step 1) is non-negotiable -- the
subagent refuses to file person tensions and surfaces the IDR /
direct-conversation route instead.

The constitutional safeguard:

> Draft and confirm only. Do not call `glassfrog_create_tension`,
> `glassfrog_update_tension`, or `glassfrog_delete_tension` without
> explicit human confirmation. Do not process tensions on the human's
> behalf.

### What is NOT adopted in v0.3

These were considered and deferred:

- **Option D: auto-file from explicit human tension statements.** When
  a user makes an unambiguous tension statement ("file this as a
  tension: ..."), v0.3 still requires the per-tension confirmation
  block before filing. Option D collapses that to a single
  acknowledgment. Deferred until v0.3 produces a calibration baseline
  -- the cost of a false-positive auto-file (wrong role attribution,
  pollutes inbox) is meaningful, and the user's tolerance for
  confirmation friction in v0.3 will tell us whether Option D is
  worth its risk.
- **AI-agent self-filing.** When a scheduled routine fires as an AI
  agent that is itself a role-filler (per the actor model in
  `skills/shared/actor-and-role-resolution.md`), the agent could
  in principle file tensions on its own role without human
  confirmation. The constitutional question -- whether an
  AI-agent role-filler has the same "lived experience" basis for
  filing that a human role-filler has -- has not been worked
  through with sufficient care for v0.3. Agent-detected tensions
  in v0.3 queue up for human confirmation in the next interactive
  session.
- **Tension-detection hook (UserPromptSubmit).** A shell hook could
  regex-scan user prompts for tension language. v0.3 uses skill-driven
  attention in the main conversation thread instead, because (a) hooks
  cannot call MCP and so a hook can only inject a hint, not act, and
  (b) skill-driven attention preserves the conversational quality of
  the "tension worth filing -- want me to draft one?" offer. We will
  revisit only if skill-driven attention proves insufficient in
  practice.
- **Read tension history during context resolution.**
  `glassfrog_list_role_tensions` could become part of the output of
  `/holacracy:context`. We deferred this to keep `/holacracy:context`
  cheap and focused on identity + role roster; users who want their
  inbox surfaced can run `/holacracy:process-inbox` directly.

Each of these will get its own follow-up ADR if and when graduated.

## Consequences

### Positive

- The plugin's docs become honest about what the API does. The v0.2
  claim that "tensions cannot be filed" is corrected wherever it
  appears.
- Capturing a felt tension drops from "remember to log into GlassFrog
  later" to "one confirmation in flow." The friction reduction is the
  point: tensions get captured when they are sensed, not when the
  user finds time to triage.
- The plugin remains a *tension sensor and capture assistant*, not a
  *tension processor*. The constitutional boundary is preserved
  through the confirmation contract, not through technical
  unavailability.
- The capability is cross-cutting: any role-flavored session (Core
  Four or otherwise) can capture tensions on any role the actor
  fills. The five-skill bundle gains a shared capability without
  requiring a new role-flavored skill.

### Negative / risks

- **Per-tension confirmation friction.** Users who file many tensions
  per session will feel the confirmation prompt. This is the cost
  of the safeguard; Option D is the relief valve if it proves too
  high. We're choosing to start strict and relax later, not the
  reverse, because relaxing is one ADR away and tightening after
  bad inbox data lands is much harder.
- **Wrong-role attribution.** The subagent infers the sensing role
  from conversation context and the actor's role roster. Inference
  is sometimes wrong. The confirmation block surfaces the inferred
  role so the user can correct it; we expect a calibration period
  where the user overrides ~10-20% of inferences.
- **Older GlassFrog MCP servers** without the tension endpoints will
  see commands degrade gracefully (the subagent falls back to
  drafting plain-text tensions for manual entry). This is documented
  in `glassfrog-api-constraints.md`. The plugin announces the
  fallback rather than silently failing.
- **The v0.2 docs were emphatic about the principled boundary.** Users
  who internalized "AI cannot file tensions, by design" may
  experience the change as a reversal. The ADR text and the
  corrected docs both explain that the principle holds -- the
  boundary moved from "no file" to "no file without explicit human
  authorship" -- but communication of the shift matters.

### Operational implications

- New artifacts shipped in v0.3.0: `agents/tension-capture.md`,
  `commands/capture-tension.md`, `commands/process-inbox.md`,
  `commands/supersession-sweep.md`, `skills/shared/tension-triage.md`,
  `skills/shared/tension-capture-flow.md`.
- Modified artifacts: `holacratic-ai-governance/SKILL.md`,
  `holacratic-ai-governance/references/glassfrog-api-constraints.md`,
  `holacratic-ai-governance/references/engagement-patterns.md` (adds
  Pattern 5), `holacracy-rep-link/references/tension-triage-guide.md`
  (cross-reference to the new shared/triage), `.claude-plugin/plugin.json`
  (bump to 0.3.0), `README.md` (slash command docs + MCP requirement).
- Three follow-up issues to be filed at PR time (per the user's
  global CLAUDE.md "Proposing Future Work" rule):
  1. Option D: auto-file for explicit tension statements
  2. AI-agent self-filing during scheduled routines
  3. Tension history surfacing in `/holacracy:context`

## Notes

The constitutional grounding for the supersession sweep is S.5.5.1d
(objection-independence test). The same logic that determines
whether an objection is genuinely separate from a proposer's
tension determines whether two tensions in an inbox are separate.
This is documented in `skills/shared/tension-triage.md` Step 3 and
referenced from `commands/supersession-sweep.md`.

The hook-cannot-call-MCP constraint -- documented in
`hooks-handlers/session-start.sh` -- is the reason the supersession
sweep is Claude-driven rather than fired automatically by a Stop
hook. A Stop hook could read a local session-tension ledger and
emit a hint, but it cannot itself archive tensions; the human-in-
the-loop write must happen in the main thread or a subagent. The
implicit Claude-driven offer plus the explicit slash command cover
the use case without needing the ledger machinery.
