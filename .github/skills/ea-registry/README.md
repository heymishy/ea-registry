# EA Registry Skill README

This guide explains how to use the `/ea-registry` skill in this repository.

## Purpose

`/ea-registry` is the single skill used to query, contribute to, and audit the enterprise application and interface registry.

It supports four operating modes:
- `QUERY`: Answer questions about current registry contents
- `CONTRIBUTE`: Add or update registry entries (always via PR)
- `AUDIT`: Find stale, unverified, orphaned, or inconsistent entries
- `FEED`: Provide dependency context to other skills (`/discovery`, `/definition`, `/reverse-engineer`)

## Mermaid Diagram: How The Skill Works

```mermaid
flowchart TD
    A[User asks registry question or change] --> B{Classify intent}

    B -->|State of registry| Q[QUERY mode]
    B -->|Add or update entries| C[CONTRIBUTE mode]
    B -->|Quality sweep| U[AUDIT mode]
    B -->|Input for other skills| F[FEED mode]

    Q --> Q1[Check registry paths exist]
    Q1 --> Q2[Load apps, interfaces, domains]
    Q2 --> Q3[Compute answer: interfaces, blast radius, domain, app profile]
    Q3 --> Q4[Flag unverified or stale entries]
    Q4 --> Z[Return structured result]

    C --> C1[Duplicate checks]
    C1 --> C2[Validate required fields]
    C2 --> C3[Write YAML from template]
    C3 --> C4[Set verification_status to unverified]
    C4 --> C5[Set source field and dates]
    C5 --> C6{Conflict with verified entry?}
    C6 -->|Yes| C7[Raise conflict PR and set disputed]
    C6 -->|No| C8[Prepare standard PR description]
    C7 --> Z
    C8 --> Z

    U --> U1[Check stale by date]
    U1 --> U2[Check unverified entries]
    U2 --> U3[Check orphaned interfaces]
    U3 --> U4[Check apps with no interfaces]
    U4 --> U5[Check lifecycle inconsistencies]
    U5 --> U6[Group actions by owner]
    U6 --> Z

    F --> F1[Build app and interface context]
    F1 --> F2[Include blast radius and domain context]
    F2 --> F3[State registry confidence verified vs unverified]
    F3 --> Z

    G[Guardrails apply in all modes] --> G1[Never commit direct to main]
    G --> G2[Never set verified in skill-generated changes]
    G --> G3[Never silently overwrite verified entries]
    G --> G4[Never delete entries for retirement]
    G4 --> G5[Use lifecycle_status or disposition instead]

    G1 -.-> C
    G2 -.-> C
    G3 -.-> C
    G4 -.-> C
```

## Quick Start

Use plain-language prompts; the skill infers mode from intent.

Example prompts:
- `What interfaces does card-issuing-platform have?`
- `What depends on card-issuing-platform?`
- `What is the blast radius of replacing card-issuing-platform?`
- `Audit the registry and show stale or unverified entries`
- `Register this application: [details]`
- `Add an interface from [source] to [target]`

## Mode Behavior

### QUERY
Use for read-only analysis.

Typical outputs:
- Interface inventory (inbound/outbound)
- Blast radius (direct and one-hop dependencies)
- Domain view (apps and boundary interfaces)
- Application profile summary

Quality rules:
- Flag `unverified` entries explicitly
- Flag `stale` entries explicitly
- Avoid presenting `disputed` entries as confirmed fact

### CONTRIBUTE
Use for adding or updating YAML entries.

Required behavior:
- Always work through PRs
- Always create entries as `verification_status: unverified`
- Always include `source` and review metadata
- Never overwrite verified data silently; raise conflict PR if needed

For new application entries:
- Start from `registry/applications/_template.yaml`
- Minimum fields: `slug`, `name`, `owner`, `domain`, `business_capability`, `tier`, `lifecycle_status`

For new interface entries:
- Start from `registry/interfaces/_template.yaml`
- Validate both applications exist first
- Filename pattern: `[source]--[target]--[name].yaml`

### AUDIT
Use for governance and hygiene checks.

Expected checks:
- Stale entries by `last_reviewed`/`last_verified`
- Unverified entries pending review
- Orphaned interfaces (source or target app missing)
- Applications with no interfaces
- Lifecycle inconsistencies

### FEED
Use to support other planning skills.

Provide:
- Application profile and lifecycle
- Direct dependencies and criticality
- Domain context
- Confidence summary (`verified` vs `unverified`)

## Repository Paths

- `registry/applications/`: application entries
- `registry/interfaces/`: interface entries
- `registry/domains/`: domain entries
- `registry/CONVENTIONS.md`: naming, taxonomy, and governance rules
- `.github/skills/ea-registry/SKILL.md`: operational instructions for the skill

## Contribution Workflow (Human + Skill)

1. Confirm intent and mode (`QUERY`, `CONTRIBUTE`, `AUDIT`, `FEED`).
2. For changes, perform duplicate and consistency checks.
3. Create or update YAML using templates and conventions.
4. Keep skill-generated additions as `unverified`.
5. Prepare PR description with affected files and requested reviewers.
6. Let owners/EA reviewers validate and set final verification status.

## Common Pitfalls

- Creating interfaces where source/target applications are not yet registered.
- Marking entries `verified` before reviewer confirmation.
- Updating verified entries directly without conflict handling.
- Treating unverified entries as definitive planning truth.

## Definition of Done

A `/ea-registry` task is complete when:
- Mode was correctly selected and stated.
- Output is structured and decision-ready.
- Any unverified/stale/disputed data is clearly called out.
- For contributions, PR-ready content is produced with correct metadata.
