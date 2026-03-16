# EA Registry

Enterprise application and interface registry for Westpac NZ.

This repository stores registry data as YAML in `registry/`.

## Key Paths

- `registry/CONVENTIONS.md`: contribution and format rules
- `registry/applications/`: application entries
- `registry/interfaces/`: interface entries
- `registry/domains/`: domain entries
- `.github/skills/ea-registry/SKILL.md`: skill behavior and operating modes
- `.github/skills/ea-registry/README.md`: usage guide and Mermaid workflow diagram

## How To Use The Skill

Use `/ea-registry` in one of four modes inferred from intent:
- `QUERY`
- `CONTRIBUTE`
- `AUDIT`
- `FEED`

See `.github/skills/ea-registry/README.md` for detailed examples, guardrails, and workflow.

## Contribution Rule

All changes go through pull request. No direct commits to main.
