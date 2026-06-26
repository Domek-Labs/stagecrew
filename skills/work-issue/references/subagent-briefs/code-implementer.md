# Subagent-Brief — Loop-Type: `code` / Stage: Implementer

**Verwendet von:** `/work-issue` Stage 2 wenn Issue-Label `loop-type:code` (oder Default-Fallback).

**Loaded by:** `skills/work-issue/SKILL.md` zur Subagent-Brief-Auswahl.

**Placeholder:** `{{branch_pattern}}`, `{{commit_format}}`, `{{syntax_check}}`, `{{hard_gates}}`, `{{default_branch}}`, `{{repo_path}}`, `{{slug}}`, `{{issue_num}}`, `{{secret_scan_pattern}}` — werden zur Render-Zeit aus AGENTS.md-Cache + State-Tracker ersetzt.

---

## Briefing (Template)

> Du implementierst Issue #{{issue_num}} im Repo `{{slug}}` (lokal: `{{repo_path}}`).
> Validator hat GO gegeben.
>
> **Standards aus AGENTS.md / Issue-Override (L1+L2):**
> - `branch_pattern`: `{{branch_pattern}}`
> - `default_branch`: `{{default_branch}}`
> - `commit_format`: `{{commit_format}}`
> - `syntax_check`: `{{syntax_check}}`
> - `hard_gates`: `{{hard_gates}}`
>
> ### Schritte
>
> 1. **Branch-Setup**
>    ```bash
>    cd {{repo_path}}
>    git fetch origin && git checkout {{default_branch}} && git pull
>    git checkout -b <branch_aufgeloest-nach-pattern>
>    ```
>    Branch-Name aus `{{branch_pattern}}` ableiten (z.B. `feature/voice-diff-review`).
>
> 2. **Code-Conventions checken via codebase-memory**
>    ```
>    search_code(<keyword>) -> Sibling-Funktionen + Style-Conventions
>    ```
>    Verstehe wie aehnliche Features im Repo strukturiert sind.
>
> 3. **Implementiere nach Spec**
>    Strict gegen Issue-AC. Jede AC-Box muss am Ende durch dein Diff abgedeckt sein.
>
> 4. **Syntax-Check**
>    ```bash
>    {{syntax_check}}
>    ```
>    Fehlschlag = STOP, kein Commit. Fix erst Syntax.
>
> 5. **Secret-Scan vor Commit**
>    ```bash
>    git add -A
>    git diff --cached | grep -iE '{{secret_scan_pattern}}'
>    ```
>    Default-Pattern wenn nicht ueberschrieben:
>    `(ghp_[A-Za-z0-9]{30,}|sk-ant-[A-Za-z0-9_-]{40,}|TELEGRAM_BOT_TOKEN=[0-9]+:[A-Za-z0-9_-]+|API_KEY=[a-zA-Z0-9]{20,})`
>
>    Match = **ABORT** mit klarer Hinweis-Message. Kein Commit.
>
> 6. **Commit**
>    Format nach `{{commit_format}}` (typisch `conventional`):
>    ```
>    <type>(<scope>): <short summary>
>
>    <body>
>
>    Refs #{{issue_num}}
>
>    Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
>    ```
>
> 7. **Push**
>    ```bash
>    git push -u origin <branch>
>    ```
>
> 8. **Issue-Kommentar** `## [stage:implementer] ready for test` mit:
>    - Branch-Name
>    - Commit-Hash
>    - LOC + Files-Count
>    - AC-Selfcheck (pro Box: erledigt? in welchem Diff?)
>    - Verwendete Standards-Werte (Branch-Pattern aufgeloest, Syntax-Check-Output)
>
> ### Hart-Constraints
>
> - **KEIN** docker/test-run (das ist Tester-Stage)
> - **KEIN** PR-Create (das ist Closer-Stage)
> - **KEINE** Hard-Gates-Verletzung (siehe `{{hard_gates}}`)
> - **KEINE** Edits ausserhalb Files-To-Touch ohne Begruendung im Comment
>
> ### Parent-Output
>
> Max 200 Woerter mit:
> - Branch + Commit
> - Diff-Stat (Files, LOC)
> - AC-Coverage-Selfcheck
> - Auffaelligkeiten (Spec-Luecken, unerwartete Refactorings, Skipped-AC mit Begruendung)

---

## Parent-Entscheidung (nach Implementer-Output)

- **OK** → Stage 3 (Tester) starten. Skip-Regel: Diff nur in `docs/**` / `*.md` / `// comment` → Tester skippen, direkt Stage 4.
- **Spec-Luecke zu gross** → ESCALATE. Telegram-Text an User mit Spec-Hint, `/create-issue --refine {{issue_num}}` vorschlagen.

---

## Tester-Pruef-Kriterien (Type-spezifisch fuer `code`)

Tester-Stage prueft fuer `code`-Type:
- `{{smoke_test}}` aus AGENTS.md erfolgreich (Build GREEN, Tests GREEN)
- Pro AC-Box: konkreter Smoke-Test + Beweis (Log-Snippet, Command-Output)
- Cleanup: Live-Stack/State zuruecksetzen falls angefasst

Siehe `skills/work-issue/SKILL.md` Stage 3 fuer den vollstaendigen Tester-Brief
(generisch ueber alle Types).

---

## Critic-Pruef-Kriterien (Type-spezifisch fuer `code`)

Critic-Stage prueft fuer `code`-Type:
- Pro AC: `search_code` aus codebase-memory fuer Code-Evidence
- Diff-Quality: Naming, Error-Handling, Cleanup, Security
- Out-of-Scope-Check gegen `default_oos` + Issue-OoS
- `hard_gates`-Check

Siehe `skills/work-issue/SKILL.md` Stage 4 fuer den vollstaendigen Critic-Brief.
