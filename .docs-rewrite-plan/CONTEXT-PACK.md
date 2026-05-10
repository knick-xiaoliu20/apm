# Docs Rewrite -- Shared Context Pack

Every Wave B+ subagent (writer, verifier, editor, CDO) MUST read this
file before starting. It is the source-of-truth for terminology, voice,
locked decisions, and ground-truth source paths.

## 1. The 3 promises (canonical strings -- LOCKED, never paraphrase)

### Promise 1: Portable by manifest

- **Hook (10 words):** "One `apm.yml`. Seven harnesses. Reproducible AI
  agent setup."
- **30-second proof (56 words):** Every developer who clones the repo
  runs `apm install` and gets the same skills, prompts, instructions,
  hooks, and MCP servers wired into Copilot, Claude, Cursor, OpenCode,
  Codex, Gemini, and Windsurf. The lockfile pins exact versions and
  content hashes. New contributor onboarding for AI context goes from
  "follow this 12-step README" to one command.
- **The 10-second demo:**
  ```bash
  git clone <repo> && cd <repo> && apm install
  ```

### Promise 2: Secure by default

- **Hook (11 words):** "Every `apm install` scans for hidden Unicode
  before agents read it."
- **30-second proof (55 words):** Agent context is executable -- a
  prompt is a program for an LLM. APM treats it that way. Each install
  scans for invisible Unicode that can hijack agent behavior, pins
  content hashes in the lockfile, and gates transitive MCP servers
  behind explicit trust prompts. `apm audit` rebuilds context in scratch
  and diffs against your working tree to catch hand-edits before they
  ship.
- **The 10-second demo:**
  ```bash
  apm audit
  ```

### Promise 3: Governed by policy

- **Hook (10 words):** "Org policy enforced at install time, before MCP
  touches disk."
- **30-second proof (57 words):** `apm-policy.yml` lets a security team
  allow-list sources, scopes, and primitives. Every `apm install` runs
  the policy *before* writing to disk -- including transitive MCP
  servers shipped by deep dependencies. Tighten-only inheritance flows
  enterprise -> org -> repo. `apm audit --ci` wires the same checks
  into branch protection. This is the supply-chain check npm and pip
  cannot do.
- **The 10-second demo:**
  ```bash
  apm install --dry-run <package>
  ```

## 2. Canonical harness list (LOCKED)

All supported INCLUDING Windsurf. The full canonical list, in this
order when listed alphabetically or as prose:

- GitHub Copilot (CLI + IDE)
- Claude Code
- Cursor
- Codex
- Gemini
- OpenCode
- llm CLI
- VS Code (with Copilot)
- Windsurf

When listing in a "supports N harnesses" claim, use 7 (the README's
canonical count for end-user agent runtimes; covers Copilot, Claude,
Cursor, Codex, Gemini, OpenCode, Windsurf). The llm CLI and VS Code are
delivery surfaces, not separate harnesses.

Verify against `src/apm_cli/integration/targets.py` -- if a harness is
not registered there, do NOT name it.

## 3. Locked decisions

- **D1 (harness list):** above. Windsurf in.
- **D2 (AI-Native framing):** RETIRED this cycle. Replace with the
  3-promise spine. Demote the awesome-ai-native link to a "Prior art"
  footnote in CONTRIBUTING/community pages only.
- **D3 (npm collisions):** INTENTIONAL ergonomics. `apm install`,
  `apm update`, `apm list`, `apm prune` deliberately mirror npm verbs
  even when semantics diverge. Mitigate via plain-language callouts in
  docs; never propose rename. Add a "Coming from npm?" callout where
  collisions would surprise (especially `apm update` = updates the CLI
  binary, not deps).
- **D4 (auth split):** Consumer-side auth is short and friendly (60
  lines max, "public packages just work; set GITHUB_APM_PAT for
  private"). Enterprise-side auth is the deep-dive
  (token policy, hosts, EMU, Microsoft Entra). Both pages exist; both
  are linked from their persona's ramp.
- **D5 (canonical strings):** Section 1 above is LOCKED. Quote
  verbatim. Do not paraphrase.

## 4. Persona vocabulary

Three personas. Use these names consistently in TOC, sidebar, and
internal references. User-facing copy may use friendlier phrases ("Use
a package", "Author and publish", "Govern at fleet scale") but the
sidebar slugs are stable: `consumer/`, `producer/`, `enterprise/`.

| Persona | Goal | Entry point | Ramp-end success |
|---|---|---|---|
| Consumer | Run someone's package on my agent harness | `/quickstart` | `apm run my-script` succeeds |
| Producer | Author + publish a primitive others can install | `/concepts/primitives-and-targets` -> `/producer/` | a packed plugin in a marketplace, installable by another developer |
| Enterprise | Gate org installs on policy + audit in CI | `/enterprise/governance-overview` | `apm-policy.yml` in `<org>/.github`, CI audit gating merges, registry pinned |

## 5. Editorial voice (the editorial-owner enforces this)

- Second person ("you"), active voice.
- One idea per sentence; short paragraphs; one-line answers up top.
- Code-forward: every claim is paired with a runnable example or a code
  citation.
- Pragmatic: if a paragraph does not change reader behavior, delete it.
- Progressive disclosure: page top = one-line answer; then the how;
  then the why; then edge cases. Reference pages may invert this for
  table-first layouts.
- No marketing adjectives: "powerful, seamless, robust, cutting-edge,
  best-in-class, next-generation" are banned.
- Length: aim 50-200 lines per page; flag anything over 300 for split
  unless it is a reference table.

## 6. Encoding rule (HARD)

ASCII only (printable U+0020-U+007E). No emoji, no unicode dashes
(em-dash, en-dash), no curly quotes, no box-drawing characters. Status
symbols use ASCII brackets: `[+]` success, `[!]` warning, `[x]` error,
`[i]` info, `[*]` action, `[>]` running.

## 7. Ground-truth source paths

Verifiers MUST cite these for every code-truth claim:

- CLI command surface: `src/apm_cli/commands/*.py` and
  `src/apm_cli/cli.py` (Click groups + commands)
- Manifest schema: `src/apm_cli/models/apm_package.py`
- Lockfile schema: `src/apm_cli/deps/lockfile.py`
- Auth + tokens: `src/apm_cli/core/auth.py` and `src/apm_cli/utils/github_host.py`
- Policy engine: `src/apm_cli/policy/*.py` (especially `ci_checks.py` for the 8 baseline checks)
- Targets / harnesses: `src/apm_cli/integration/targets.py`
- Cache: `src/apm_cli/commands/cache.py`
- Marketplace: `src/apm_cli/commands/marketplace/`
- Integrators: `src/apm_cli/integration/`

To check a CLI surface live, run from the repo root:
```bash
uv run python -m apm_cli --help
uv run python -m apm_cli <subcommand> --help
```

## 8. New TOC slug map (Wave B foundation)

| Old path | New path | Disposition |
|---|---|---|
| `index.mdx` | `index.mdx` | REWRITE |
| `getting-started/quick-start.md` | `quickstart.mdx` (or `quickstart/index.md`) | REWRITE + MOVE |
| `introduction/what-is-apm.md` | `concepts/what-is-apm.md` | REWRITE |
| (NEW) | `concepts/the-three-promises.md` | NEW |
| `introduction/key-concepts.md` (split) | `concepts/primitives-and-targets.md` | NEW (absorbs) |
| `introduction/anatomy-of-an-apm-package.md` | `concepts/package-anatomy.md` | KEEP+EDIT |
| `introduction/how-it-works.md` | `concepts/lifecycle.md` | REWRITE |
| (NEW) | `concepts/glossary.md` | NEW |

For Wave B, write the NEW pages directly at the NEW paths. Wave I
(Astro expert) will handle sidebar restructure + redirects. Old paths
remain in place during Wave B; sidebar updates wait for I.

## 9. Inline canonical references

- README spine: `/README.md` lines 1-100 (3 promises + how-to-install)
- CDO synthesis (full corpus disposition + P0/P1/P2 backlog):
  session file
  `/var/folders/1d/s61glgts17lgfr6_0j0mcrjh0000gn/T/1778436665889-copilot-tool-output-vdeff3.txt`
- Growth-lens advisory (G1-G6 including locked strings + landing copy):
  session file
  `/var/folders/1d/s61glgts17lgfr6_0j0mcrjh0000gn/T/1778436594668-copilot-tool-output-izqc1i.txt`
- Architecture: `.docs-rewrite-plan/AGENTIC-SYSTEM-DESIGN.md`

## 10. Output format spec (for writer agents)

Return a SINGLE markdown response containing:

```
=== FILE START: <relative path under docs/src/content/docs/> ===
<the full file content, including frontmatter>
=== FILE END ===

=== SOURCE TABLE ===
| Claim in page | Source path:line | Verified? |
|---|---|---|
| (every code-truth claim, command, flag, env var, schema field) | `src/apm_cli/...` | yes/no/run-output |

=== NOTES ===
<anything the verifier or editor needs to know: tradeoffs, open
questions, skipped scope, etc.>
```

The orchestrator parses `=== FILE START ===` blocks and writes them to
disk verbatim.
