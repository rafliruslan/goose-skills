# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Goose Skills is an NPM-distributed registry of 125 GTM (Go-To-Market) skills for Claude Code, Codex, and Cursor, built by [Gooseworks](https://gooseworks.sh). Skills are installed via `npx goose-skills install <slug>` and fetched from GitHub raw CDN (`https://raw.githubusercontent.com/athina-ai/goose-skills/main/`). Published to npm as `goose-skills`. Requires Node.js 20+.

## Commands

```bash
npm run validate:skills   # Validate all skill metadata and structure
npm run build:index       # Regenerate skills-index.json from skill directories
npm test                  # Run tests (node:test + node:assert/strict)
npm run ci                # Full pipeline: validate + build:index + test

# Run CLI locally
node bin/goose-skills.js list
node bin/goose-skills.js install <slug>
node bin/goose-skills.js info <slug>
```

Run a single test with `node --test --test-name-pattern="<pattern>" test/goose-skills-targets.test.js`.

## Architecture

### Skill Structure

Every skill lives in `skills/{capabilities|composites|playbooks}/<slug>/` and requires:

- **SKILL.md** — YAML frontmatter (`name`, `description`) followed by markdown instructions
- **skill.meta.json** — Machine-readable metadata: `slug`, `category`, `tags`, `installation`

Optional: `scripts/` (Python/Node executables), `references/`, `templates/`

#### SKILL.md Frontmatter

```yaml
---
name: {slug}
description: >
  Multi-line description
---
```

Playbooks additionally include `type: playbook`, `graph` (provides/requires/connects_to), and `skills_used` fields.

#### skill.meta.json

```json
{
  "slug": "skill-slug-name",
  "category": "capabilities|composites|playbooks",
  "tags": ["tag1", "tag2"],
  "installation": {
    "base_command": "npx goose-skills install skill-slug-name",
    "supports": ["claude", "cursor", "codex"]
  }
}
```

**Valid tags:** All, ads, brand, competitive-intel, content, design, lead-generation, monitoring, outreach, research, seo

**Slug format:** lowercase with hyphens only (`^[a-z0-9-]+$`)

### Skill Categories

- **capabilities** (55) — Atomic single-purpose tools (scrapers, API integrations, analyzers)
- **composites** (61) — Multi-skill chains combining capabilities into workflows
- **playbooks** (9) — End-to-end orchestrated workflows

### CLI (`bin/goose-skills.js`)

Fetches `skills-index.json` from GitHub raw CDN, downloads skill files to platform-specific locations:
- Claude Code: `~/.claude/skills/<slug>/`
- Codex (`--codex`): `~/.codex/skills/<slug>/`
- Cursor (`--cursor --project-dir <path>`): `.cursor/rules/goose-<slug>.mdc`

Platform-specific install logic is in `bin/lib/targets.js`.

### Index Generation (`scripts/build-index.js`)

Scans all three category directories, parses SKILL.md frontmatter and skill.meta.json, collects all files recursively, and outputs `skills-index.json`. The index is sorted by slug.

### Validation (`scripts/validate-skills.js`)

Enforces: slug format, SKILL.md and skill.meta.json presence, JSON validity, slug/category consistency between directory name and metadata, required fields, no duplicate slugs. **Note:** currently only validates `capabilities` and `composites` — not `playbooks`.

### Schema (`schemas/skill-meta.schema.json`)

Defines the skill.meta.json contract using JSON Schema (Draft 2020-12).

## Key Conventions

- **Module system:** CommonJS (`require`/`module.exports`)
- **No runtime dependencies** — pure Node.js for the CLI; only `@changesets/cli` as a dev dependency
- **skills-index.json must be committed** — CI verifies it matches what `build-index.js` would generate via `git diff --exit-code`
- Directory name must exactly match `meta.slug` and belong under the matching `meta.category` directory
- The `description` field in the index comes from SKILL.md frontmatter, not from skill.meta.json
- **Versioning:** Managed via `@changesets/cli` — do not bump versions manually
- The GitHub repo is `athina-ai/goose-skills` — the CDN base URL and `REPO` constant in `bin/goose-skills.js` depend on this

## Adding a New Skill

1. Create `skills/{category}/<slug>/` with SKILL.md and skill.meta.json
2. Optionally add `scripts/` and `references/` subdirectories
3. Run `npm run validate:skills`
4. Run `npm run build:index`
5. Run `npm test`
6. Commit the skill files **and** the regenerated `skills-index.json`

## CI/CD

- **ci.yml:** Runs on PRs and pushes to main — validates skills, builds index, checks for uncommitted index changes, runs tests
- **release.yml:** On push to main — uses changesets to create release PRs or publish to npm
- **dispatch-private-sync.yml:** Triggers sync to the gooseworks private repo on push to main
