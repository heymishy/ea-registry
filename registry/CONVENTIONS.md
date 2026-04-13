# EA Registry — Conventions and Format Reference

**Registry repo:** https://github.com/heymishy/ea-registry.git
**Raw file base URL:** https://raw.githubusercontent.com/heymishy/ea-registry/refs/heads/main/README.md

## What this registry is

A living, Git-native record of every application in the organisation and the
interfaces between them. It is the authoritative source of truth for:

- What applications exist, who owns them, and their lifecycle status
- What interfaces connect them — type, direction, protocol, and criticality
- What data domains each application owns or touches
- Which applications are candidates for or affected by modernisation programmes

It is **not** a deep technical specification for any individual system.
It is a **map** — accurate enough to reason about dependencies, plan
modernisation, and identify blast radius, but not a replacement for
per-system design documentation.

---

## Repository structure

```
registry/
  applications/
    _template.yaml          ← copy this to create a new application entry
    [app-slug].yaml         ← one file per application
  interfaces/
    _template.yaml          ← copy this to create a new interface entry
    [source]--[target]--[name].yaml  ← one file per interface
  domains/
    _template.yaml          ← copy this to create a new domain entry
    [domain-slug].yaml      ← optional: group applications by business domain
  CONVENTIONS.md            ← this file
```

---

## Naming conventions

### Application slugs
Lowercase, hyphen-separated, stable. Once set, do not change — other files reference it.

```
card-issuing-platform
fraud-engine
payment-gateway
customer-profile-service
```

### Interface file naming
`[source-app-slug]--[target-app-slug]--[interface-name].yaml`

The interface name should describe what flows, not how:
```
card-issuing-platform--fraud-engine--auth-request.yaml
payment-gateway--settlement-service--daily-settlement-file.yaml
core-banking--customer-profile-service--customer-event-stream.yaml
```

If direction is bidirectional, source = the system that initiates the exchange.

### Domain slugs
Business domain names, not technical:
```
payments
lending
identity-and-access
customer-data
risk-and-compliance
```

---

## Lifecycle status values

Used in application entries. Controls how skills interpret and flag the application.

| Status | Meaning |
|--------|---------|
| `active` | In production, maintained |
| `active-under-review` | In production but under modernisation assessment |
| `active-modernising` | In production, actively being replaced or re-platformed |
| `active-legacy` | In production, known end-of-life, no active modernisation programme yet |
| `decommissioning` | Replacement in progress, cutover date known |
| `decommissioned` | No longer in production — retained for historical interface traceability |
| `planned` | Not yet in production — referenced by in-flight programmes |

---

## Interface type taxonomy

Every interface must have exactly one `type`. Sub-types provide additional detail.

| Type | Sub-types | Description |
|------|-----------|-------------|
| `api-sync` | `rest`, `soap`, `graphql`, `grpc`, `iso8583` | Synchronous request/response |
| `event-async` | `kafka`, `mq`, `sns-sqs`, `webhook`, `proprietary` | Asynchronous event or message |
| `batch-file` | `sftp`, `s3`, `shared-filesystem`, `email`, `proprietary` | File-based periodic exchange |
| `db-coupling` | `shared-table`, `db-polling`, `cross-schema-query`, `stored-proc` | Direct database integration |
| `stream` | `kafka-stream`, `cdc`, `proprietary` | Continuous data stream |
| `manual` | `human-handoff`, `report`, `spreadsheet` | Human-mediated process |

---

## Verification status

Applied to both application entries and interfaces. Determines how skills treat the entry.

| Status | Meaning | Who can set |
|--------|---------|------------|
| `verified` | Confirmed accurate by the owning team or architect | App owner or EA team |
| `unverified` | Added via skill extraction or automated means — not yet reviewed | Any contributor / skill |
| `disputed` | Someone has raised a question about accuracy — see notes | Any contributor |
| `stale` | Not reviewed in >12 months — may be inaccurate | Automated staleness check |

Skills may read `unverified` entries but must flag them explicitly in their output.
Skills must not treat `disputed` entries as facts.

---

## Contribution rules

### Who can contribute
Anyone — architects, engineers, BAs, or skills — can contribute to any part
of the registry. There are no ownership gatekeepers for who may *propose* a change.
The PR review process is the governance mechanism, not role-based access control.

### The one rule: everything goes through a PR
**No entry is ever written directly to the main branch.** Every addition or
modification — whether made by a human or a skill — must go through a PR.
This applies without exception, including to the contributor's own team's applications.

This rule exists because:
- The registry is shared infrastructure — unreviewed changes affect everyone
- Git history provides a complete audit trail only if changes are deliberate PRs
- Conflicts (skill-found interfaces that differ from existing entries) must be
  visible and resolved by a human, not silently overwritten

### PR review requirements

| Change type | Required reviewers |
|-------------|-------------------|
| New application entry | `owner` field in the new entry (or a nominated architect if owner is a team) |
| New interface entry | `owner` of source app AND `owner` of target app |
| Modify a `verified` application entry | Current `owner` in the entry |
| Modify a `verified` interface entry | Owner of both source and target apps |
| Modify an `unverified` entry | Any single reviewer familiar with the system |
| Domain entry (new or modified) | EA team representative |
| Skill-generated entries (any) | Same as above — no reduced bar because a skill created it |

If an application has no `owner` set, tag the EA team as reviewer and add
a note to resolve the ownership gap.

### Verification

`verification_status: verified` may only be set by the PR reviewer when they
are satisfied the entry accurately reflects reality. It must never be set by:
- The contributor who opened the PR
- A skill (skills always create entries as `unverified`)
- Anyone who hasn't reviewed the substance of the entry

### Conflicts

A conflict exists when a skill (or human contributor) finds an interface or
application detail that differs from an existing `verified` entry.

**Do not overwrite the verified entry.** Instead:
1. Open a PR with the proposed change
2. Add a comment block to the YAML:
   ```yaml
   # CONFLICT — proposed change differs from verified entry
   # Proposed by: [contributor / skill-name]
   # Date: [date]
   # Proposed change: [what is being changed and why]
   # Current verified value: [what it currently says]
   # Evidence: [code reference / extraction report / observation]
   ```
3. Set `verification_status: disputed` in the PR
4. Tag both the entry's current `owner` and the contributor as reviewers
5. The reviewer resolves the conflict — accepts the new value, keeps the old, or
   updates to a third version — and sets `verification_status` appropriately

### Skill contributions
Skills (reverse-engineer, ea-registry, and others) must:
1. Create new YAML files with `verification_status: unverified`
2. Set `source: skill-[skill-name]` and populate `discovery` fields
3. **Propose a PR** — never commit directly to main
4. Tag relevant owners as reviewers in the PR description
5. Include a summary of what was found and how in the PR body

Skills must NOT:
- Commit directly to main branch
- Change `verification_status` from `unverified` to `verified`
- Overwrite existing `verified` entries — raise a conflict PR instead
- Delete entries — set `lifecycle_status: decommissioned` or `modernisation_disposition: retired`

---

## Staleness management

Every application entry has a `last_reviewed` date.
Every interface entry has a `last_verified` date.

Entries not reviewed in 12 months should be flagged as `stale`.
The `/ea-registry` skill will flag stale entries when queried.

The EA team should run a quarterly review sweep using:
> "Run /ea-registry audit — flag all entries not reviewed since [date]"

---

## Design decisions

**Why YAML, not Markdown?**
Skills need to read this programmatically. YAML is human-writable with comments,
produces clean PR diffs, and can be schema-validated. A markdown table cannot be
queried reliably by a skill across hundreds of entries.

**Why one file per application and one file per interface?**
One file per entity means:
- A PR for one application change is scoped and reviewable
- Git blame gives clear ownership history per application
- Interface files can be found by searching for either app slug in the filename
- No merge conflicts from two teams updating the same file simultaneously

**Why not store interfaces inside the application file?**
Interfaces are co-owned by two applications. Storing them in one application's
file creates an asymmetry — one owner "holds" the record. Separate files with
both slugs in the filename makes co-ownership explicit and searchable.

**Why a separate domains directory?**
Applications belong to domains. Domains can be queried by skills to find all
applications in a modernisation scope ("what else lives in the payments domain?").
Domains also provide a grouping for blast-radius analysis.
