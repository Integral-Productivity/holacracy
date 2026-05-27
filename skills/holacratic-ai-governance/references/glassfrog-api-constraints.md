# GlassFrog API Constraints -- Comprehensive Reference

This reference documents the complete capabilities and limitations of the GlassFrog API as exposed through a standard MCP server integration. Understanding these constraints is essential for operating the skill's engagement patterns correctly and for communicating honestly with users about what AI can and cannot do within their governance system.

---

## Table of Contents

1. [Available Tool Inventory](#available-tool-inventory)
2. [Read Capabilities](#read-capabilities)
3. [Write Capabilities](#write-capabilities)
4. [Hard Boundaries -- What Cannot Be Done](#hard-boundaries)
5. [Architectural Rationale](#architectural-rationale)
6. [Workarounds and Proxies](#workarounds-and-proxies)
7. [Known Data Quirks](#known-data-quirks)

---

## Available Tool Inventory

A standard GlassFrog MCP server exposes tools across these categories:

### Read-Only Structure (6 tools)
| Tool | Input | Returns |
|---|---|---|
| `list_circles` | (none) | All circles with IDs, names, strategies, role_ids, people_ids |
| `get_circle` | circle_id | Full circle detail: strategy, all roles with purpose/accountabilities/domains, all people |
| `list_roles` | circle_id (optional), person_id (optional) | Role summaries: id, name, purpose, is_circle, is_core_role |
| `get_role` | role_id | Full role detail: purpose, accountabilities, domains, people, parent_circle_id, sub_circle_id |
| `list_people` | circle_id (optional), role (optional: lead_link\|rep_link\|facilitator\|secretary) | All people: id, name, email |
| `get_person` | person_id | Single person: id, name, email, external_id |

### Read-Only Operations (3 tools)
| Tool | Input | Returns |
|---|---|---|
| `list_checklist_items` | circle_id (optional), global (optional boolean) | Items: id, description, frequency, global, role_id, circle_id |
| `list_metrics` | circle_id (optional), global (optional boolean) | Metrics: id, description, frequency, global, role_id, circle_id |
| `list_projects` | circle_id (optional) | Projects: id, description, status, value, effort, roi, private, created_at, role_id, circle_id, person_id |

### Tension Read Operations (2 tools)
| Tool | Input | Returns |
|---|---|---|
| `list_role_tensions` | role_id, status (optional: `unprocessed` \| `processed` \| `archived`), q (optional), per_page, cursor | Page<Tension>: id, body, status, role, label, meeting_type |
| `get_tension` | tension_id | Single Tension: id, body, status, role, label, meeting_type |

### Definition Update Operations (4 tools)
| Tool | Input | Modifies |
|---|---|---|
| `update_checklist_item` | item_id, description?, frequency? | Checklist item definition (not completion status) |
| `update_metric` | metric_id, description?, frequency? | Metric definition (not reported values) |
| `update_project` | project_id, description?, status?, value?, effort?, roi?, private? | Project metadata |
| `update_person` | person_id, name?, email?, external_id? | Person name, email, or external ID |

### Item Creation Operations (3 tools)
| Tool | Input | Creates |
|---|---|---|
| `create_checklist_item` | description, frequency, circle_id, role_id | New checklist item tied to a role in a circle |
| `create_metric` | description, frequency, circle_id, role_id | New metric tied to a role in a circle |
| `create_project` | description, circle_id, role_id | New project tied to a role in a circle |

### Item Deletion Operations (3 tools)
| Tool | Input | Deletes |
|---|---|---|
| `delete_checklist_item` | item_id | Permanently removes a checklist item |
| `delete_metric` | metric_id | Permanently removes a metric |
| `delete_project` | project_id | Permanently removes a project |

### Tension Write Operations (3 tools)
| Tool | Input | Action |
|---|---|---|
| `create_tension` | role_id, body (optional), status (optional, default `unprocessed`) | Creates a tension on the sensing role. **Asymmetric constraint:** rejects `label` and `meeting_type` on create -- to set either, follow up with `update_tension` on the returned tension ID. |
| `update_tension` | tension_id, body? label? status? meeting_type? (`tactical` \| `governance`) | Updates body, label, status, or routes to a meeting type. Symmetric -- all four fields are accepted. |
| `delete_tension` | tension_id | Permanently removes a tension. Prefer `update_tension(status: "archived")` for soft-collapse; deletion is permanent and irrecoverable. |

### People Management Operations (2 tools)
| Tool | Input | Action |
|---|---|---|
| `create_person` | name, email, external_id? | Adds a new member to the organization (triggers welcome email) |
| `delete_person` | person_id | Permanently removes a person from the organization |

### Role Assignment Operations (2 tools)
| Tool | Input | Action |
|---|---|---|
| `assign_person_to_role` | role_id, person_id | Assigns a person to a role (replaces current filler for Lead Link roles) |
| `unassign_person_from_role` | role_id, person_id | Removes a person from a role |

### Reference (1 tool)
| Tool | Input | Returns |
|---|---|---|
| `list_frequencies` | (none) | Array of frequency strings. May omit custom frequencies not yet assigned to any item -- see Known Data Quirks. |

---

## Read Capabilities

### What Can Be Read

**Organizational structure:**
- Complete circle hierarchy (all circles, their roles, people)
- Role details: purpose, accountabilities (text), domains (text), assigned people
- Circle strategies
- Core role identification (Lead Link, Rep Link, Facilitator, Secretary)
- Whether a role is itself a circle (sub-circles)

**Operational tracking:**
- Checklist items: description, frequency, assigned role, parent circle
- Metrics: description, frequency, assigned role, parent circle
- Projects: description, status, value/effort/ROI annotations, privacy flag, creation date, assigned role and person

**People:**
- Name, email, external ID
- Role assignments (derived by cross-referencing roles)

### What Cannot Be Read

- **Policies**: GlassFrog policies are not exposed through the standard API. This is a significant gap -- policies constrain how roles operate, and without them, the AI's governance understanding is incomplete. When relevant, ask the user for policy context directly.
- **Meeting history**: Past governance and tactical meeting records are not available via API.
- **Checklist completion records**: Whether a checklist item was marked done/not-done in a particular meeting.
- **Metric reported values**: The actual numbers reported for metrics. Only the metric definition (what to track, how often) is available.
- **Governance change history**: When a role was created or modified, by whom, or what changed.
- **Cross-link relationships**: Super-circle/sub-circle relationships can be inferred from role data (via `is_circle` and `sub_circle_id`), but the API does not provide a dedicated hierarchy endpoint.

*Tensions* are now readable per-role via `list_role_tensions` and `get_tension` (see Tension Read Operations above). Older notes in this plugin claimed tensions were inaccessible -- that was true before the GlassFrog MCP exposed the tension endpoints. It is no longer true.

---

## Write Capabilities

### What Can Be Written

**Checklist items**: Can create new items (with description, frequency, circle, and role), update description and frequency, and delete items permanently. Cannot mark items as complete in a meeting context.

**Metrics**: Can create new metrics (with description, frequency, circle, and role), update description and frequency, and delete metrics permanently. Cannot report metric values in a meeting context.

**Projects**: Can create new projects (with description, circle, and role), update description/status/value/effort/ROI/privacy, and delete projects permanently.

**People**: Can create new organization members (triggers a welcome email with password setup), update name/email/external ID, and delete members permanently.

**Role assignments**: Can assign a person to a role and unassign a person from a role. For Lead Link roles, assigning a new person replaces the current filler rather than adding additively.

### Important Distinctions

- **Updating a checklist description** != **completing a checklist item**. The former changes what the item says; the latter records whether it was done in a given tactical meeting. Only the former is possible via API.
- **Updating a metric description** != **reporting a metric value**. The former changes what the metric tracks; the latter records the actual number. Only the former is possible via API.
- **Updating a project status** is a metadata annotation, not a formal workflow transition. GlassFrog projects do not have a formal status machine -- the status field is freeform text.
- **Deleting items is permanent**. There is no soft-delete or trash. For projects, prefer setting status to "Archived" unless the intent is true removal.

### Tension write asymmetry

`create_tension` rejects `label` and `meeting_type` on create, even though `update_tension` accepts both. To file a tension with a label or a routed meeting type, use the two-call sequence:

```
1. create_tension(role_id, body, status: "unprocessed") -> returns ten_xxx
2. update_tension(ten_xxx, label: "...", meeting_type: "governance" | "tactical")
```

The `tension-capture` subagent and the `/holacracy:*` slash commands implement this sequence; downstream callers do not need to handle it manually.

---

## Hard Boundaries

These are actions that **cannot be performed via the GlassFrog API under any circumstances**. Do not attempt them, do not suggest they are possible, and do not present workarounds as equivalents.

### Governance Changes (Cannot Be Made via API)
- Creating, modifying, or deleting roles
- Adding, changing, or removing accountabilities
- Creating, modifying, or deleting domains
- Creating, modifying, or deleting policies
- Creating or modifying circle strategies
- Moving roles between circles
- Converting a role to a circle or vice versa

### Tension Processing (Constitutional Boundary, Not Technical)

Filing, reading, archiving, and routing tensions are now possible via API (`create_tension`, `list_role_tensions`, `get_tension`, `update_tension`, `delete_tension`). What remains off-limits is **processing** -- the work of resolving a tension into a governance proposal or a tactical action. That happens in human meetings, and the plugin's tension-capture flow enforces this distinction:

- AI may **draft** and, on explicit per-tension human confirmation, **file** tensions (status `unprocessed`).
- AI may **route** filed tensions to `tactical` or `governance` meeting types, on user direction (`/holacracy:process-inbox`).
- AI may **archive** filed tensions on user direction (false positives, superseded items).
- AI does **not** mark tensions `processed` autonomously -- the `processed` status reflects that a human governance or tactical meeting actually worked the tension.

This is a *principled* boundary, not a technical limitation. See `skills/shared/tension-capture-flow.md` for the canonical capture contract, including the constitutional safeguard.

### Meeting Operations (Cannot Be Done via API)
- Starting, running, or recording meetings
- Marking checklist items as complete in a meeting context
- Reporting metric values in a meeting context
- Requesting projects in tactical meetings

---

## Architectural Rationale

These constraints are not merely technical limitations -- they reflect a principled design boundary.

### Why Governance is Read-Only

Holacracy's governance process is a human-centered practice of distributed authority. The governance meeting has a specific format (proposal -> objection -> integration) designed to ensure that structural changes reflect the lived experience of role-fillers and are tested against the organization's collective wisdom.

Allowing API-driven governance changes would:
- Bypass the objection process, potentially creating structure that harms the organization
- Undermine the developmental practice of learning to sense and articulate tensions
- Create a two-track governance system (human meetings + API changes) that could produce contradictions
- Remove the consent mechanism that protects against premature or harmful structural changes

The read-only constraint preserves governance as a human developmental practice. The AI's role is to support that practice -- by surfacing relevant context, sensing potential tensions, and drafting proposals -- not to replace it.

### Why Tension Filing Is Now Supported -- With a Constitutional Safeguard

Tensions in Holacracy are *lived experiences* of a gap between what is and what could be. They are inherently personal and require the tension-holder to confirm them as their own. The historical concern about API-driven tension filing was that an AI-filed tension would lack the embodied context that makes the tension real and processable.

The plugin resolves this concern with the **draft-and-confirm contract** specified in `skills/shared/tension-capture-flow.md`: the AI may *draft* a tension with full attribution and routing, but cannot file it without the human's per-tension explicit confirmation. The act of confirmation is what makes the tension the human's own. The API write is then a low-friction mechanism for what the human has already affirmed.

What remains exclusively human:

- *Processing* a tension into a governance proposal or a tactical action (the meeting work).
- *Marking a tension processed* (the API write is available, but only under explicit human direction in `/holacracy:process-inbox`).
- *Sensing-on-behalf-of-an-AI-agent that fills its own role* (deferred to v0.4 with its own ADR).

The result: AI accelerates *capture*; humans retain authorship of *processing*.

### The Healthy Boundary

These constraints create a clear division of labor:
- **AI**: Reads structure, senses patterns, holds perspectives, produces work artifacts
- **Humans**: Evolve structure, process tensions, make governance decisions, embody roles

This division supports both operational efficiency (AI handles data-intensive pattern detection) and developmental integrity (humans retain authority over meaning-making and structural evolution).

---

## Workarounds and Proxies

For capabilities the API does not support, these proxies can partially close the gap:

### For governance proposals
- Draft the proposal text following Holacratic format
- Present it to the user for review
- The user brings it to a governance meeting manually

### For tension filing (when the GlassFrog MCP does not expose tension endpoints)

If a user's GlassFrog MCP server is older and does not expose `create_tension` / `list_role_tensions` / `update_tension`, the plugin falls back to:

- Format detected tensions as tension statements with the resolved sensing role labelled.
- Organize them by circle and suggested meeting type (governance vs. tactical).
- Present a "tension report" the user can copy into the GlassFrog UI manually.

This is the same artifact Pattern 3 (Tension Sensing) produced before the tension endpoints existed. The capture flow degrades to this gracefully when the API is unavailable.

### For checklist completion tracking
- If the user verbally reports completion status, record it in the conversation context
- Suggest the user update GlassFrog directly during or after the tactical meeting
- Note: checklist item creation and deletion are now API-supported; only meeting-context completion remains UI-only

### For metric reporting
- Help the user calculate or prepare metric values
- The user enters them in GlassFrog during the tactical meeting
- Note: metric creation and deletion are now API-supported; only in-meeting value reporting remains UI-only

---

## Known Data Quirks

Issues observed in production GlassFrog instances that the skill should handle gracefully:

### Null Frequencies
Some checklist items and metrics may have null or empty frequency values. This typically indicates either:
- An item created before frequency was a required field
- A data migration artifact
- A governance gap (the frequency was never explicitly decided)
- The intended frequency is a custom frequency (e.g., "Daily") that was never actually assigned to the item

Handle by flagging these during tension scanning and, if the user wants, using the update tools to set an appropriate frequency. When items appear to be daily operational indicators (real-time dashboards, daily task lists), ask the user whether a custom frequency like "Daily" is configured in their GlassFrog admin before assuming it does not exist.

### Invisible Custom Frequencies
`list_frequencies` only returns frequencies that are currently assigned to at least one checklist item or metric. If a GlassFrog administrator has configured a custom frequency (e.g., "Daily", "Fortnightly", "Annual") but no items currently use it, the frequency will not appear in tool results. This creates a blind spot: the AI cannot discover frequencies that exist but are unused.

**Workaround**: If the user confirms a custom frequency exists, use it directly in `update_checklist_item` or `update_metric` calls -- the API will accept it even though `list_frequencies` did not return it.

### Orphaned Role Assignments
Checklist items, metrics, or projects may reference role_ids that no longer exist in the current governance structure. This occurs when a role is removed or restructured in governance but the operational items are not cleaned up.

Handle by detecting the orphan in tension sensing and presenting it to the user for resolution.

### Core Roles Without People
In some circles, core roles (especially Rep Link and Secretary) may not have anyone assigned. This is technically a governance gap -- these roles should be filled. Flag it as a structural tension.

### People Without Roles
Some people in the GlassFrog instance may not be assigned to any roles. This could mean:
- They are new and not yet assigned
- They have been removed from roles but not from the organization
- They serve in an informal capacity not captured in governance

Do not assume this is an error. Note it if relevant and let the user decide.
