---
work-issue:
  branch_pattern: "feature/<short>"
  default_branch: main
  pr_base: main
  commit_format: conventional
  syntax_check: "true"
  smoke_test: "true"
  deploy_command: ""
  linter: ""
  hard_gates:
    - "Kein direkter Edit auf main"
    - "Keine neuen Secrets im Repo"
    - "SKILL.md braucht YAML-Frontmatter (name/description/model)"
    - "Plugin-Cache reload (claude restart) nach SKILL.md-Aenderung sonst alte Version aktiv"
    - "Bei Version-Bump muss .claude-plugin/plugin.json synchron mit Plugin-Cache-Pfad sein"
  default_oos:
    - "Keine Code-Generation (Repo ist Markdown-only)"
    - "Keine Migration alter Skill-Versionen ohne explizite User-Freigabe"
  ac_templates:
    - "CLAUDE.md am Repo-Root reflektiert die Aenderung"
    - "SKILL.md hat valides YAML-Frontmatter"
    - ".claude-plugin/plugin.json ist konsistent (Version, Name)"
loop_types:
  enabled: [code, research]
  default: code
  type_overrides:
    research:
      ac_templates: []
      smoke_test: "true"
    # Roadmap (v3.4.0+):
    # text:
    #   ac_templates: ["markdownlint passt", "linkcheck gruen"]
    # decision:
    #   ac_templates: ["ADR-Format korrekt", "Constraint-Coverage vollstaendig"]
    # diagnostic:
    #   ac_templates: ["Hypothesen reproduzierbar", "Fix-Spec klar"]
---

# Agent Instructions for loop-engineering-workflow

Per-Repo Coding-Standards-Spec fuer AI-gestuetzte Workflows. Single-Source-of-Truth fuer dieses Plugin-Repo selbst (Dogfooding — wir nutzen `/work-issue` um an `/work-issue` zu entwickeln).

## Architektur

Single-Plugin-Repo fuer das `loop-engineering-workflow` Claude-Code-Plugin.

- `.claude-plugin/plugin.json` — Plugin-Manifest (Name, Version)
- `CLAUDE.md` — Plugin-Doku
- `README.md` — Public-Audience-Tagline + Quickstart
- `skills/`
  - `loop/` — Umbrella-Skill (Router zu init-agents / create-issue / work-issue)
  - `init-agents/` — Bootstrap AGENTS.md fuer einen Repo
  - `create-issue/` — Idee → GitHub-Issue mit Spec-Standard
  - `work-issue/` — Issue → Merge-Loop bis Merge (5-Stage Spec→Build-Loop)

## Code-Style / Conventions

- Repo ist **Markdown-only**. Keine TS/JS/Python.
- SKILL.md **muss** YAML-Frontmatter haben mit Pflicht-Feldern `name`, `description`, optional `model`, `allowed-tools`.
- Description-Field: lowercase, kommagetrennte Trigger-Wörter (keine inline-`Trigger:`-Praefixe — das bricht strict YAML).
- Commit-Format: conventional (`feat(skill): ...`, `fix(create-issue): ...`, `docs: ...`).
- Branch-Pattern: `feature/<short>` oder `fix/<short>`.

## Loop-Types (v3.3.0 Multi-Type-System)

Dieser Skill unterstuetzt seit v3.3.0 mehrere Issue-Types:
- **`code`** (default) — Software-Implementation-Tasks
- **`research`** — Erkenntnis-Generierung mit Test-Matrix

Erprobt in:
- Code-Loops: 13 PRs in `dscheinecker-at7media/personal-ai-bot` (2026-06-26)
- Research-Loop: `dscheinecker-at7media/personal-ai-bot#51` (2026-06-26)

Roadmap fuer `text`, `decision`, `diagnostic` Types — siehe Repo-Issues mit Label `roadmap` + `loop-type`.

## Hard-Gates Detail

1. **Kein direkter Edit auf main** — PR-Flow Pflicht, branch-protection sollte aktiv sein (wenn nicht: einrichten).
2. **Keine neuen Secrets im Repo** — kritisch da spaeter Public-Flip geplant ist.
3. **SKILL.md YAML-Frontmatter** — Loader-tolerant aber strict-YAML compliance ist Pflicht (sonst CI-Brueche bei kuenftigen Validator-Tools).
4. **Plugin-Cache reload nach Skill-Aenderungen** — `claude restart` Pflicht in jedem PR-Body.
5. **plugin.json Version-Bump synchron** — Plugin-Cache-Pfad nutzt Version, asynchroner Bump = stale Cache.

## Plugin-Loader-Gotchas

- Cache-Pfad: `~/.claude/plugins/cache/loop-engineering-workflow/<version>/skills/<name>/SKILL.md`
- Marketplace-Source: kann lokaler Pfad sein (Development) oder GitHub-URL (Production-Public)
- Bei Version-Bump in `plugin.json`: alter Cache-Pfad bleibt, neuer Pfad wird erstellt — `claude restart` zwingend
