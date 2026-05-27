# Tension Capture Flow -- Shared Reference

This document specifies the canonical **B-flow** used by every tension capture in this plugin: the on-demand `/holacracy:capture-tension` command, the ambient detection in `holacratic-ai-governance`, and the `tension-capture` subagent.

The B-flow is **draft-and-confirm**: Claude proposes a complete tension (sensing role + body + meeting_type), the human confirms or edits per-tension, and only on explicit confirmation does the API call fire. No auto-file. No silent file. Per-tension confirmation, never batched.

This is the v0.3 contract. Option D (auto-file from explicit human statements) and AI-agent self-filing are deferred to v0.4 with their own ADRs.

---

## The Constitutional Safeguard

> **Draft and confirm only.** Do not call `glassfrog_create_tension` without explicit human confirmation. Do not call `glassfrog_update_tension` or `glassfrog_delete_tension` without explicit human confirmation. Do not process tensions on the human's behalf -- processing happens in human governance and tactical meetings.

This is non-negotiable across every code path in this plugin. The principle: Claude is a tension *sensor* and *capture assistant*; humans are tension *processors*. The new GlassFrog tension write endpoints do not change that boundary. They change *how cheaply* the human can capture what they're sensing.

---

## The Capture Flow

### Step 1 -- Determine intent

The flow can begin in three ways:

1. **Explicit command.** User runs `/holacracy:capture-tension` (optionally with the tension text as `$ARGUMENTS`). Intent is clear; proceed.
2. **Ambient detection.** `holacratic-ai-governance` is loaded and Claude observes tension language in conversation ("I keep hitting...", "no one owns...", "we can't get...", "I'm frustrated by..."). Claude **pauses** and offers to capture: *"That sounds like a tension worth filing -- want me to draft one?"* If the user assents, proceed. If they deflect, drop it and continue the original work.
3. **Pattern 3 follow-up.** A tension surfaced by data-mining tension sensing (`holacratic-ai-governance/references/engagement-patterns.md` Pattern 3) is offered for capture. User confirms or skips; proceed for those they confirm.

Never proceed past Step 1 without the user's clear assent to capture.

### Step 2 -- Resolve sensing role

Follow the procedure in `./actor-and-role-resolution.md`, **but with the target = any role the actor fills in the relevant circle** (not a single named target role).

1. Confirm actor identity via `glassfrog_get_me`.
2. Load actor's role roster via `glassfrog_list_my_roles`.
3. Narrow to the circle implied by the tension's content (use conversation context, or ask).
4. Within that circle, identify the role(s) the actor fills that the tension's content most plausibly attaches to.
5. If exactly one plausible sensing role -> use silently.
6. If multiple -> ask: *"This could be filed on [Role A] or [Role B] -- which feels closer to the gap you're sensing?"*
7. If zero (the actor fills no role in that circle) -> name the constraint per `./tension-triage.md` Step 4.

### Step 3 -- Run triage (Step 1 of `./tension-triage.md`)

Apply the role-vs-person gate. If person tension, **refuse to draft `create_tension`** and surface the IDR / direct-conversation route. Do not proceed.

This gate runs *before* drafting the body, because drafting a person tension and then refusing to file is wasted work and confuses the user about what the system is willing to do.

### Step 4 -- Draft the tension body

Preserve the user's own words as much as possible. Tensions are lived experiences; the user's framing is part of the data.

Three guidelines:

- **Keep it concrete.** "The IT Governance role hasn't responded to our approval request in 3 weeks, blocking the rollout of Tool X" is better than "we have an IT problem."
- **Name the gap, not the desired fix.** Tensions describe what *is* vs. what *could be*. Proposed solutions belong in the governance proposal that processes the tension, not in the tension body itself.
- **Attribution-on-behalf-of.** If the tension is being carried for someone else (per `./tension-triage.md` Step 1, "Carrying tensions on behalf of others"), prepend the body with `Sensed by [name], carried as [role]:`.

The body must be 1–5000 characters (API constraint). Most tensions fit in 1–3 sentences.

### Step 5 -- Route to meeting type (Step 2 of `./tension-triage.md`)

Suggest a `meeting_type`:

- **governance** if the tension implies a structural change.
- **tactical** if the tension implies operational coordination or unblocking.
- **omit** if genuinely ambiguous; surface the ambiguity to the user.

The user can override. If they pick neither and the field is omitted, the tension is still filed -- it just routes to neither agenda until later triaged.

### Step 6 -- Present the per-tension confirmation

Show the user a single, compact confirmation block. The format:

```
Sensing role:  [Role name] of [Circle name]    (role_id: role_xxx...)
Body:          [Tension body, with attribution preamble if applicable]
Meeting type:  [governance | tactical | (none)]

File this? [y] yes  [e] edit  [n] no
```

The user can:

- **y / yes** -> Proceed to Step 7.
- **e / edit** -> Allow editing any of the three fields, then re-present.
- **n / no** -> Abort. Confirm the abort and return to the original conversation.

Per-tension confirmation. Never batched. This is the v0.3 contract.

### Step 7 -- File the tension

On confirmation, two-call sequence:

1. `glassfrog_create_tension(role_id, body, status: "unprocessed")` -- returns the new tension. **`label` and `meeting_type` are rejected on `create_tension`** (see `holacratic-ai-governance/references/glassfrog-api-constraints.md`).
2. If the user specified a `meeting_type` (or a label), follow up with `glassfrog_update_tension(tension_id, meeting_type: ..., label: ...)`.

If `create_tension` fails (network, permissions, role no longer exists), surface the error honestly: *"I couldn't file the tension -- GlassFrog returned [error]. The draft is still here; want me to retry or adjust?"*

### Step 8 -- Acknowledge and return

Return to the user a compact confirmation:

```
Filed: [body excerpt or label]
Tension ID: ten_xxx...
On role: [Role name] of [Circle name]
Routed to: [meeting type]
```

Then continue the original conversation. Do not interrogate the user about the next step; they will process the tension in a meeting on their own time.

---

## What the Flow Does NOT Do (v0.3)

- **No auto-file.** Every tension passes through Step 6 confirmation, every time.
- **No batched confirmation.** Even if Claude detects three tensions in one conversation, each gets its own confirmation.
- **No AI-agent self-filing.** Scheduled routines that fire as AI agents collect candidate tensions for human review in the next session; they do not call `create_tension` themselves.
- **No processing.** Claude does not mark tensions `processed` on the human's behalf. That happens when a human runs a governance or tactical meeting and processes the tension. (`update_tension(status: "processed")` is available to the human via `/holacracy:process-inbox` for meeting-day catch-up, but only under explicit instruction.)
- **No deletion.** `glassfrog_delete_tension` is permanent. Prefer `status: "archived"` for false positives or superseded tensions. Deletion is reserved for cases where the user explicitly chooses it.

---

## How This Flow Composes with Other Surfaces

| Surface | How it uses this flow |
|---|---|
| `/holacracy:capture-tension` | Entry at Step 1.1 (explicit command). Runs Steps 2–8 via the subagent. |
| `holacratic-ai-governance` (ambient detection) | Entry at Step 1.2 (ambient). On user assent, dispatches the `tension-capture` subagent to run Steps 2–8. |
| `/holacracy:process-inbox` | Skips Steps 1–6 (the tension is already filed). Runs `update_tension` for routing/archiving, using `./tension-triage.md` Steps 2–3 as the decision guide. |
| `/holacracy:supersession-sweep` | Runs `./tension-triage.md` Step 3 (supersession) across recent or session-filed tensions. On confirmation, archives subsumed tensions via `update_tension(status: "archived")`. |
| `holacratic-ai-governance` Pattern 3 (Tension Sensing) | Detects candidate tensions from data patterns. For each, can route into this flow at Step 1.3 (Pattern 3 follow-up). |
