---
name: ea-registry
description: >
  Reads, queries, and maintains the enterprise application and interface registry.
  Answers questions about what applications exist, what interfaces connect them,
  what domain a system belongs to, and what the blast radius of a change might be.
  Accepts contributions from /reverse-engineer and other skills, writing new entries
  as unverified YAML with a PR for owner review. Performs audit sweeps to find stale,
  unverified, or missing entries. Feeds /discovery and /definition with dependency
  context for modernisation planning.
  Use when someone says "what systems does X integrate with", "what interfaces
  does the card platform have", "register this application", "update the registry",
  "what's the blast radius of replacing X", "audit the registry", or when
  /reverse-engineer or /discovery asks for dependency context.
triggers:
  - "what systems does X integrate with"
  - "what interfaces does"
  - "register this application"
  - "update the registry"
  - "blast radius"
  - "what depends on"
  - "add to the registry"
  - "audit the registry"
  - "stale entries"
  - "what's in the payments domain"
  - "dependency context"
  - "interface inventory"
---

# EA Registry Skill

For a practical operator guide and workflow diagram, see
`.github/skills/ea-registry/README.md`.

## What this skill does

Four distinct modes of operation. Identify which mode applies before proceeding.

| Mode | When to use |
|------|------------|
| **QUERY** | Answer a question about the current state of the registry |
| **CONTRIBUTE** | Add or update application or interface entries |
| **AUDIT** | Find gaps, stale entries, unverified items, or orphaned interfaces |
| **FEED** | Provide dependency context to another skill (/discovery, /definition, /reverse-engineer) |

State the mode at the start of every response.

---

## Entry condition check

Verify the registry directory exists and is readable:
- `registry/applications/` — contains application YAML files
- `registry/interfaces/` — contains interface YAML files
- `registry/CONVENTIONS.md` — format reference exists

If registry is empty or not initialised:
> "The EA registry appears to be empty or not yet initialised.
> To add the first application: share the application details and I'll create
> the entry."

---

## Mode: QUERY

### Query types and how to answer them

**"What interfaces does [app] have?"**

1. Read `registry/applications/[app-slug].yaml` — confirm the app exists
2. Search all files in `registry/interfaces/` for any containing `[app-slug]`
   in either the `source.app` or `target.app` fields
3. For each interface found: name, type, subtype, direction, counterparty,
   criticality, modernisation_disposition, verification_status
4. Group by: inbound vs outbound, then by type
5. Flag any `unverified` or `stale` entries explicitly

Output format:
```
## Interfaces for [App Name] ([app-slug])

### Inbound (X interfaces)
| Interface | Counterparty | Type | Criticality | Status |
|-----------|-------------|------|-------------|--------|
| [name] | [app] | [type/subtype] | [criticality] | [verification] |

### Outbound (X interfaces)
| Interface | Counterparty | Type | Criticality | Status |
|-----------|-------------|------|-------------|--------|

⚠️ [n] interfaces are unverified — treat with caution.
⚠️ [n] interfaces are stale (not reviewed in >12 months).
```

---

**"What depends on [app]?" / "What's the blast radius of replacing [app]?"**

1. Find all interfaces where `target.app = [app-slug]` (inbound — systems that call this one)
2. Find all interfaces where `source.app = [app-slug]` (outbound — systems this one calls)
3. For inbound: these are systems that would be directly affected by replacing [app]
4. For outbound: these are interfaces that the replacement must preserve or re-implement
5. Traverse one level deeper: for each directly dependent app, find their other interfaces
   — these are indirect dependencies (won't break immediately but may be affected)

Output format:
```
## Blast Radius: [App Name]

### Direct impact — systems that call [app] (must handle cutover)
[list with criticality and interface type]

### Interfaces [app] calls (must be preserved or re-implemented in replacement)
[list with modernisation_disposition]

### Indirect dependencies (one hop away)
[list — informational, lower urgency]

### Summary
Replacing [app] directly affects [n] systems via [n] interfaces.
[n] of these interfaces are critical.
[n] interfaces have modernisation_disposition = 'changing' or 'retiring'.
[n] counterparty systems will need to make changes to their integration.
```

---

**"What's in the [domain] domain?"**

1. Read `registry/domains/[domain-slug].yaml`
2. For each application in the domain: name, tier, lifecycle_status, verification_status
3. Find all interfaces that are internal to the domain (both source and target in same domain)
4. Find all interfaces that cross domain boundaries (one app inside, one outside)

Output format:
```
## Domain: [Domain Name]

### Applications ([n] total)
| Application | Tier | Status | Verification |
|-------------|------|--------|-------------|

### Internal interfaces ([n])
[interfaces where both apps are in this domain]

### Boundary interfaces ([n]) — cross-domain touchpoints
| Interface | Internal App | External App | External Domain | Type |
|-----------|-------------|--------------|-----------------|------|
```

---

**"What do we know about [app]?"**

Read and summarise the application YAML file:
- Identity, ownership, domain, tier, lifecycle status
- Technology stack
- Data domains owned and consumed
- Regulatory scope
- Link to all interfaces (run the interface query)
- Modernisation note if applicable
- Verification status and last reviewed date

If `verification_status: unverified` — prepend:
> ⚠️ **This entry is unverified.** It was added via [source] on [date]
> and has not yet been reviewed by the application owner.
> Treat with caution — confirm with [owner] before using in programme planning.

---

**"Does an interface exist between [app A] and [app B]?"**

Search interface files for any where both app slugs appear in source/target fields.
May find: zero interfaces (none registered), one or more, or `unverified` entries.

If zero found:
> "No interface is registered between [A] and [B].
> This could mean: (a) no interface exists, (b) an interface exists but hasn't been
> registered yet, or (c) it was found via hidden pattern scan but not yet added.
> If you believe an interface exists, use CONTRIBUTE mode to add it."

---

## Mode: CONTRIBUTE

### The one rule this mode never breaks
**Every contribution goes through a PR. No exceptions.**
This applies whether the contributor is a human or a skill, whether the entry
is new or an update, whether the contributor is an architect or a junior engineer.
The PR is the governance mechanism — not role-based access control.

Skills in this mode always:
1. Write new YAML files in a working branch (never directly to main)
2. Set `verification_status: unverified` on every created entry
3. Set `source: skill-ea-registry` or `source: skill-[originating-skill]`
4. Produce a PR description summarising what was added and why
5. Tag the relevant reviewers in the PR description

Skills never:
- Commit directly to main
- Set `verification_status: verified`
- Overwrite an existing `verified` entry (raise a conflict PR instead)
- Delete entries

---

### Adding a new application entry

Triggered by: "register this application", "add [app] to the registry",
or when /reverse-engineer produces application data.

**Step 1: Duplicate check**
Search existing application files for the app name and any known aliases.
If a potential match exists:
> "An entry for [similar name] already exists as [slug].
> Is this the same application or a different one?"
Do not create a duplicate without confirmation.

**Step 2: Collect required fields**
Minimum viable entry: `slug`, `name`, `owner`, `domain`, `business_capability`,
`tier`, `lifecycle_status`. All other fields can be filled incrementally.

If contributing manually (not from a skill output), ask for any missing required fields.
If contributing from /reverse-engineer output, populate from the extraction data
and note any fields that couldn't be determined.

**Step 3: Write the YAML**
Create `registry/applications/[app-slug].yaml` from `_template.yaml`.
Always set:
- `verification_status: unverified`
- `source: [manual | skill-name]`
- `last_reviewed: [today]`

**Step 4: Propose the PR**
Output a PR description ready to use:

```
## New application entry: [App Name]

Added by: [contributor]
Source: [manual / skill-reverse-engineer / etc.]

### What was added
- Application: [name] ([slug])
- Domain: [domain]
- Owner: [owner]
- Lifecycle status: [status]
- Tier: [tier]

### Verification needed
This entry is unverified. The following reviewer should confirm accuracy
and set verification_status to 'verified' when satisfied:

Reviewer: @[owner]

### Files changed
- registry/applications/[app-slug].yaml (new)
```

---

### Adding a new interface entry

Triggered by: "add this interface", "register the interface between X and Y",
or when /reverse-engineer produces interface data.

**Step 1: Validate both apps exist**
Check that source and target app slugs exist in `registry/applications/`.
If not:
> "[slug] is not yet in the registry. I can create the application entry
> alongside this interface, or you can add it separately first. Which do you prefer?"

**Step 2: Duplicate check**
Search interface files for any existing interface between the same source and target.
Multiple interfaces between the same pair are valid — but confirm:
> "An interface already exists between [A] and [B]: [name].
> Is this a new interface or an update to the existing one?"

**Step 3: Identify type**
If not provided, ask:
> "What type of interface is this?
> - `api-sync` — REST, SOAP, ISO 8583 — synchronous request/response
> - `event-async` — Kafka, MQ, SNS/SQS, webhook — asynchronous message
> - `batch-file` — SFTP, file drop, S3, scheduled exchange
> - `db-coupling` — shared table, DB polling, cross-schema query, stored proc
> - `stream` — Kafka stream, CDC — continuous data flow
> - `manual` — human handoff, report, spreadsheet"

**Step 4: Collect interface details**
Required minimum: source app, target app, type, direction, trigger, criticality.
Populate from extraction data if available — note fields that couldn't be determined.

**Step 5: Write the YAML**
File name: `registry/interfaces/[source]--[target]--[name].yaml`
Always set:
- `verification_status: unverified`
- `source: [manual | skill-name]`
- `last_verified: [today]`
- If from /reverse-engineer: populate `discovery.method`, `discovery.evidence`,
  and `discovery.hidden_pattern` if applicable

**Step 6: Propose the PR**

```
## New interface entry: [Interface name]

Added by: [contributor]
Source: [manual / skill-reverse-engineer / etc.]
[If hidden pattern scan:] Found via: hidden pattern scan ([pattern type])

### What was added
- Interface: [name]
- [Source app] → [Target app]
- Type: [type / subtype]
- Criticality: [criticality]
- Modernisation disposition: [disposition]

### Verification needed
This interface is unverified. Both application owners should review and
confirm accuracy before setting verification_status to 'verified'.

Reviewers: @[source-app-owner], @[target-app-owner]

[If hidden_pattern is set:]
⚠️ This interface was found via hidden pattern scan and was not previously
registered. Particular care in review — confirm the coupling exists and
the description is accurate before verifying.

[If modernisation_disposition is 'changing' or counterparty_change_required is true:]
⚠️ This interface has a modernisation implication. Architect review recommended
in addition to application owner review.

### Files changed
- registry/interfaces/[filename].yaml (new)
[If new application entries were also created:]
- registry/applications/[slug].yaml (new)
```

---

### Handling conflicts with existing verified entries

A conflict exists when new information (from extraction, observation, or a
contributor) differs from an existing `verified` entry.

**Never overwrite a verified entry directly.** The process:

1. Create a PR branch with the proposed change applied to the existing file
2. Add a conflict comment block at the top of the YAML:
   ```yaml
   # CONFLICT — proposed change differs from verified entry
   # Proposed by: [contributor / skill-name]
   # Date: [date]
   # Proposed change: [what is being changed and why]
   # Current verified value: [what it currently says]
   # Evidence: [code reference / artefact / observation]
   ```
3. Change `verification_status` from `verified` to `disputed` in the PR
4. PR description:

```
## Conflict: proposed update to verified entry [slug]

Proposed by: [contributor]

### What changed and why
[Description of the discrepancy — what the extraction / observation found
vs what the verified entry currently states]

### Evidence
[Code location, extraction report reference, or other evidence]

### Resolution options
- [ ] Accept proposed change — update entry, restore to verified
- [ ] Keep current entry — close PR, add explanatory note to YAML
- [ ] Update to a third version — modify in review

Reviewers: @[entry-owner]
[If interface:] @[source-app-owner], @[target-app-owner]
```

---

### Skill-to-registry contribution (from /reverse-engineer)

When /reverse-engineer has completed an extraction, it invokes /ea-registry
in CONTRIBUTE mode to register findings.

For each new application found:
1. Duplicate check — the legacy system may already be registered under an alias
2. Create application entry, populated from extraction data
3. Set `source: skill-reverse-engineer`, `verification_status: unverified`
4. Note any fields that couldn't be determined from extraction alone

For each new interface found:
1. Validate both app slugs (create application entries if needed)
2. Duplicate check
3. Create interface entry, populated from extraction data
4. If hidden pattern: set `discovery.hidden_pattern`, note explicitly in PR
5. If conflicts with a verified entry: raise conflict PR, not a new entry

For interfaces that match existing `unverified` entries:
- If extraction provides more detail than the existing entry: update and re-PR
- If extraction matches: add extraction as additional evidence, no change to content

Produce a single consolidated PR for all registry contributions from one extraction:

```
## EA Registry: contributions from /reverse-engineer — [system name]

### Summary
- [n] new application entries
- [n] new interface entries
- [n] hidden-pattern interfaces (not previously registered — flagged for close review)
- [n] conflicts with existing verified entries (separate conflict PRs raised)
- [n] interfaces matched existing entries — no change

### New applications
[list]

### New interfaces
[list — flag hidden pattern ones]

### Conflicts raised as separate PRs
[list with PR links if available]

### Reviewers needed
[list by owner — so each owner can see only their items]
@[owner-1]: [list of their entries]
@[owner-2]: [list of their entries]
```

---

## Mode: AUDIT

### When to run
- Quarterly review sweep: "audit the registry"
- Before a modernisation programme starts: "audit the [domain] domain"
- After a bulk contribution from /reverse-engineer: "check what's unverified"

### What the audit checks

**1. Stale entries** — `last_reviewed` / `last_verified` older than 12 months
Flag: application slug, owner, last reviewed date, days overdue.

**2. Unverified entries** — `verification_status: unverified`
Flag: entry type (app/interface), slug, source, extracted date, responsible reviewer.
Group by: pending which owner's review.

**3. Orphaned interfaces** — interface files where source or target app slug
does not match any application file.
Flag: these indicate a renamed or deleted application entry, or a typo.

**4. Applications with no interfaces** — application entries with no interface
files referencing them.
May be legitimate (standalone apps) but worth confirming — often indicates
the application was registered but its interfaces were never documented.

**5. Interface gaps vs discovery** — if a /reverse-engineer report exists for any
application in the registry, check whether the interfaces in that report are all
registered. Flag any that appear in the report but not in the registry.

**6. Lifecycle inconsistencies** — interfaces where one or both participating apps
have `lifecycle_status: decommissioned` but the interface is still `active`.

### Audit output format

```
## EA Registry Audit — [Date]
Scope: [full registry / domain / application]

### Summary
| Check | Total | Issues found |
|-------|-------|-------------|
| Stale entries (>12 months) | [n] | [n] |
| Unverified entries | [n] | [n] |
| Orphaned interfaces | [n] | [n] |
| Apps with no interfaces | [n] | [n] |
| Interface gaps vs extraction reports | [n] | [n] |
| Lifecycle inconsistencies | [n] | [n] |

### Items requiring action
[Grouped by owner — so each team sees their own items]

#### [Owner / Team]
- [ ] [App/interface slug] — [issue] — [recommended action]

### Recommended actions
1. [Highest priority action]
2. [Next action]
```

---

## Mode: FEED

### Feeding /discovery

When /discovery is running for a modernisation programme affecting a known application,
/ea-registry provides:

1. **Application profile** — what the system does, tier, lifecycle, owner
2. **Direct interfaces** — all inbound and outbound, with type and criticality
3. **Blast radius summary** — how many systems depend on this app
4. **Domain context** — other apps in the same domain that may be in scope
5. **Unverified interface warning** — if any interfaces are unverified, flag explicitly

Output is a structured context block for /discovery to incorporate:

```
## EA Registry Context for /discovery
Application: [name] ([slug])
Domain: [domain]
Tier: [tier] | Status: [lifecycle_status]

Direct interfaces: [n] ([n] inbound, [n] outbound)
Critical interfaces: [n]
Systems that depend on this app: [n]
Systems this app depends on: [n]

⚠️ Registry confidence: [n] verified, [n] unverified
Unverified interfaces should not be treated as definitive scope.

Interface summary:
[condensed table]

Recommended scope additions for /discovery:
- [App name] — [reason: critical dependency / same domain / shared data]
```

### Feeding /definition

When /definition is creating epics and stories for a modernisation programme,
/ea-registry provides interface-level detail for each interface that will be
affected:

For each interface in scope:
- Type and protocol (determines what kind of migration work is needed)
- Criticality (determines sequencing and risk)
- Counterparty change required (determines dependencies on other teams)
- Modernisation disposition (confirms current thinking)

This feeds directly into the interfaces epic structure in /definition.

### Feeding /reverse-engineer

Before /reverse-engineer starts extraction, /ea-registry provides:
- What is already known about the system being extracted
- What interfaces are already registered (saves re-extracting known interfaces)
- What aliases or previous names this system may be known by
- What other systems are known to integrate with it (context for hidden interface scan)

---

## Quality checks before any output

**QUERY mode:**
- All app slugs referenced exist in the registry
- Unverified and stale entries are explicitly flagged — not silently included as facts
- Blast radius includes both direct and indirect dependencies

**CONTRIBUTE mode:**
- Duplicate check performed before every new entry
- `verification_status` is always `unverified` on creation — no exceptions
- PR description produced for every contribution — never a direct commit to main
- Both app owners tagged for interface PRs
- Conflicts with verified entries raised as separate conflict PRs — never overwritten
- Skill-generated entries have `source` and `discovery` fields populated
- Hidden pattern interfaces flagged explicitly in both YAML and PR description

**AUDIT mode:**
- Orphaned interfaces checked (app slug exists for both source and target)
- Output is grouped by owner — actionable per team, not just a flat list

**FEED mode:**
- Unverified entries flagged before being used as inputs to other skills
- Registry confidence stated explicitly in the context block

---

## What this skill does NOT do

- Does not commit directly to main — every contribution is a PR
- Does not set `verification_status: verified` — only a human reviewer can do that
- Does not delete entries — sets `lifecycle_status: decommissioned` or `modernisation_disposition: retired`
- Does not silently overwrite verified entries — raises a conflict PR instead
- Does not resolve disputed entries — flags for human resolution
- Does not generate architecture diagrams — produces structured data for tooling to render
- Does not enforce schema validation — that is a CI/CD concern for the registry repo
- Does not access systems directly — reads and writes registry files only
- Does not require a specific role to contribute — anyone may open a PR
