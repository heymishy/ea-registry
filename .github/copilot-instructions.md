# Copilot Instructions — EA Registry

## What this repository is

This is the enterprise application and interface registry for Westpac NZ.
It is the authoritative source of truth for all applications, the interfaces
between them, and the business domains they belong to.

It is not a product codebase. There is no application code here.
Every file in `registry/` is a YAML data entry or a reference document.

**Registry repo:** https://bitbucket.westpac.co.nz/projects/[PROJECT-KEY]/repos/ea-registry

---

## How to work in this repo

The only skill you need here is `/ea-registry`.

```
/ea-registry    Query, contribute, or audit the registry
```

All other pipeline skills (discovery, definition, reverse-engineer, etc.) live
in product repos. If you have arrived here from a product repo skill, you are
in the right place — use `/ea-registry` in CONTRIBUTE or QUERY mode.

---

## The one rule

**Every change goes through a PR. No direct commits to main.**

This applies to humans and skills equally. Branch protection is enforced.
See `registry/CONVENTIONS.md` for full contribution and review rules.

---

## Repository structure

```
registry/
  CONVENTIONS.md              ← Read this before contributing anything
  applications/
    _template.yaml            ← Copy to create a new application entry
    [app-slug].yaml           ← One file per application
  interfaces/
    _template.yaml            ← Copy to create a new interface entry
    [source]--[target]--[name].yaml
  domains/
    _template.yaml            ← Copy to create a new domain entry
    [domain-slug].yaml
```

---

## Coding standards

This repo contains YAML and Markdown only. No application code.

**YAML conventions:**
- 2-space indentation
- Quoted strings for all free-text fields
- `null` for unpopulated optional fields — do not leave fields blank or delete them
- Comments are encouraged — use `#` to explain non-obvious values
- Slugs: lowercase, hyphen-separated, stable after first commit

**Markdown conventions:**
- CONVENTIONS.md is the only prose document in `registry/`
- Headings use `##` and `###` — no `#` (reserved for file title)

**Validation:**
- Schema validation runs in CI on every PR — check pipeline status before merging
- A PR with failing schema validation must not be merged

---

## Linking to this registry from other repos

In any product repo's `.github/copilot-instructions.md`, reference this registry as:

```markdown
## EA Registry
EA Registry repo: https://bitbucket.westpac.co.nz/projects/[PROJECT-KEY]/repos/ea-registry
Raw file base URL: https://bitbucket.westpac.co.nz/projects/[PROJECT-KEY]/repos/ea-registry/raw

Skills that interact with the registry (/discovery, /reverse-engineer, /definition)
should read application and interface entries from this repo when identifying
dependencies, blast radius, or interface scope.
```

---

## Placeholders to fill in before committing

- `[PROJECT-KEY]` — replace with your Bitbucket project key (e.g. `ARCH`, `EA`, `PLAT`)
  in all three URL references above and in `registry/CONVENTIONS.md`
