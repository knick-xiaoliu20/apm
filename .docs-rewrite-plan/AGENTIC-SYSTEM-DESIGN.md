# Docs Full-Rewrite -- Agentic System Design (genesis handoff)

Source of truth for every subagent. RELOAD before each spawn.

## 1. Intent + scope

Re-architect `docs/src/content/docs/` (53 source pages -> ~45 new pages
across 8 sections) per the CDO synthesis, applying 12 P0 fixes,
splitting 8 monolith pages, building 26 new pages, retiring 4 stale
pages, and modernizing Astro 6.2 Starlight feature usage. Output: a
fully-rewritten user-facing docs corpus on a single branch
`docs/full-rewrite-persona-ramps`, opened as one PR.

Boundary: does NOT touch product code (CLI, schemas, policy engine);
does NOT add product features; does NOT alter contributor docs except
to MOVE them out of user nav.

## 2. Locked decisions (from user, this session)

- D1: Tool list canonical = ALL supported INCLUDING Windsurf.
- D2: Retire AI-Native framing this cycle (replace with 3-promise
  spine).
- D3: npm collisions are INTENTIONAL ergonomics; never propose
  rename. Mitigate purely via doc callouts.
- D4: Split `getting-started/authentication.md` -> 60-line consumer
  page + full enterprise page.
- D5: Lock canonical 3-promise strings (growth-lens G4) verbatim
  across README, landing, intro, release notes.

## 3. Personas (the staffing chart)

| Role | Identity | Core responsibility | Loop position |
|---|---|---|---|
| Orchestrator | main thread | wave dispatch, PR mgmt, plan stewardship | always |
| **doc-writer** | existing `.agent.md` | drafts page from CDO disposition | step 1 of per-page loop |
| **python-architect** (verifier) | existing `.agent.md` | reads draft + verifies every code-truth claim against `src/`; flags drift; runs `apm --help` etc as needed | step 2 of per-page loop |
| **editorial-owner** (NEW) | inline persona below | tone, voice, concision, no-bloat, human, progressive disclosure | step 3 of per-page loop |
| **CDO** (chapter gate) | general-purpose synthesizer | per-chapter narrative + bridge coherence; veto power | chapter checkpoint |
| **growth-hacker** | existing `.agent.md` | corpus-wide funnel + ramp coherence, landing copy, hooks | wave H final pass |
| **astro-starlight-expert** (NEW) | inline persona below | Astro 6.2 feature audit + implementation | wave I final pass |

### NEW persona: editorial-owner

Voice: warm but technically precise; second person ("you"); active
voice; short paragraphs; one-idea-per-sentence preferred. No marketing
adjectives, no "powerful / seamless / robust / cutting-edge". Code-
forward: every claim is paired with a runnable example or a code
citation. Pragmatic: if a paragraph does not change reader behavior,
delete it. Progressive disclosure: top of page = one-line answer, then
the how, then the why, then edge cases. Length: aim for 50-200 lines
per page; flag anything over 300 for split. ASCII-only per repo
encoding rules.

### NEW persona: astro-starlight-expert

Knows Astro 6.2 + Starlight 0.38 idioms cold. Audits current
`docs/astro.config.mjs` + sidebar config + content collections + per-
page frontmatter for missed leverage:
- Starlight content-loaders / collections schema with Zod
- LinkCard + CardGrid + Card + Tabs + Aside + Steps + FileTree
  components
- `<Code>` for syntax-highlighted referenced files
- Page badges, banners, draft frontmatter
- `<TabItem>` for runtime/target switching
- Sidebar groups with `collapsed` and `autogenerate`
- `pagefind` search optimization
- Optimized SEO + OG image generation
- llms.txt + llms-full.txt updates
- Heading anchors + `Steps` over manual `1. 2. 3.`
- `lastUpdated`, `editLink`, `prev/next` overrides
- Custom 404 + redirects audit

## 4. Component diagram

```
                            +----------------------+
                            |    Orchestrator      |
                            |  (main thread; PR;   |
                            |   plan steward)      |
                            +----------+-----------+
                                       |
        +------+------+------+---------+--------+------+------+------+
        v      v      v      v        v        v      v      v      v
     [WaveA][WaveB][WaveC][WaveD]  [WaveE]  [WaveF][WaveG][WaveH][WaveI]
       P0   Found Cons  Prod      Enter    Ref   Int+  Growth Astro
      fix   ation umer  ucer      prise         Trbl   pass   pass

   Each Wave A-G runs the per-page loop:

       +---------------+  draft  +----------------+
       |  doc-writer   |-------->| python-arch    |  verify
       |  (drafter)    |         | (verifier)     |  vs src/
       +-------^-------+         +--------+-------+
               |                          |
         drift back                       v
               |                  +-----------------+
               +------------------|  doc-writer     |
                                  |  (revise)       |
                                  +--------+--------+
                                           |
                                           v
                                  +-----------------+
                                  | editorial-owner |  voice/tone/
                                  | (refine)        |  concision
                                  +--------+--------+
                                           |
                                  ALL PAGES IN CHAPTER DONE
                                           v
                                  +-----------------+
                                  |  CDO (chapter)  |  narrative +
                                  |  checkpoint     |  bridge
                                  +--------+--------+
                                       accept|reject
                                           |
                            reject --> back to writers (max 2 cycles)
                            accept --> chapter sealed; next wave proceeds
```

## 5. Sequence diagram (per-page loop)

```
  Orchestrator     doc-writer       python-arch      editorial-owner    CDO
       |               |                 |                  |             |
       |--draft prompt->|                 |                  |             |
       |               |---draft v1------>|                  |             |
       |               |                 |--verify (read src,|             |
       |               |                 |  run --help)     |             |
       |               |<--findings------|                  |             |
       |               |---draft v2 if   |                  |             |
       |               |   drift found-->|                  |             |
       |               |<--clean--------|                  |             |
       |               |---verified v   |                  |             |
       |               |   draft------>>>>>>>>>>>>>>>>>>>>>>|             |
       |               |                 |                  |--refine---->|
       |               |                 |                  |  (voice)    |
       |               |                 |                  |             |
       |<--page final--|                 |                  |<--polished--|
       |    (verified+refined)           |                  |             |
       |                                                                  |
       | --- repeat for every page in chapter ---                         |
       |                                                                  |
       |---chapter assembly (all pages + bridge text) -------------------->
       |                                                                  |
       |                                       <--accept|reject + diff----|
       |---if reject: writers + verifiers + editor revise flagged pages
       |    (max 2 CDO cycles per chapter; escalate to user on cycle 3)   |
       |---if accept: seal chapter, commit, advance wave                  |
```

## 6. Wave dependency graph

```
  Wave A  (P0 fixes -- 12 surgical edits, no rewrite loop)
    |
    | (independent of B; can run in parallel)
    v
  Wave B  (Foundation: index + quickstart + concepts/*)
    |     ^------- locks vocabulary, primitives table, glossary, 3-promise strings
    |
    +------+------+--------+
    v      v      v        v
  Wave C  Wave D  Wave E   Wave F   (parallelizable after B sealed)
  Cons    Prod    Enter    Reference
    |      |       |        |
    +------+-------+--------+
                   v
                Wave G  (Integrations + Troubleshooting; depend on C/D/E/F)
                   |
                   v
                Wave H  (growth-hacker corpus pass; landing copy lock-in)
                   |
                   v
                Wave I  (astro-starlight-expert audit + implementation)
                   |
                   v
                Open PR, request review
```

## 7. Wave staffing + page list

| Wave | Pages (in NEW TOC) | Writer threads | Loop |
|---|---|---|---|
| A: P0 fixes | 12 surgical edits across existing files | 1 | direct (no editorial loop) |
| B: Foundation | index.mdx, quickstart, concepts/{what-is-apm, three-promises, primitives-and-targets, package-anatomy, lifecycle, glossary} | 7 | full loop + CDO |
| C: Consumer | consumer/{install-packages, install-mcp-servers, deploy-a-bundle, run-scripts, update-and-refresh, manage-dependencies, authentication, private-and-org-packages, drift-and-secure-by-default, governance-on-the-consumer-ramp} | 10 | full loop + CDO |
| D: Producer | producer/{author-primitives/{skills, prompts, instructions-and-agents, hooks-and-commands, mcp-as-primitive}, compile, preview-and-validate, pack-a-bundle, publish-to-a-marketplace, package-relative-links} | 10 | full loop + CDO |
| E: Enterprise | enterprise/{governance-overview, apm-policy-getting-started, policy-pilot, enforce-in-ci, drift-detection, registry-proxy, security-and-supply-chain, adoption-playbook, github-rulesets} | 9 | full loop + CDO |
| F: Reference | reference/cli/{init,install,update,view,list,prune,outdated,cache,config,compile,run,preview,audit,targets,runtime,pack,policy,marketplace,marketplace-package,mcp,experimental} + reference/schemas/{apm-yml,apm-policy-yml,apm-lock-yaml,marketplace-yml} + reference/{targets-matrix, policy-baseline-checks, environment-variables, examples} | 25 | full loop, lighter editorial, CDO checkpoints by sub-section (cli/, schemas/, top-level) |
| G: Integrations + Troubleshooting | integrations/{ide/vscode, ide/cursor, ide/claude-code, ide/copilot-cli, ide/codex, ide/opencode, ide/llm-cli, ide/windsurf, ci-cd, gh-aw, copilot-cowork} + troubleshooting/{common-errors, install-failures, compile-zero-output-warning, ssl-issues, policy-debugging, migration} | 17 | full loop + CDO |
| H: Growth | corpus-wide pass: landing first-viewport copy, persona ramp cards, README<->landing continuity, hook lock-in | 1 (growth-hacker) reads everything, edits landing + ramp entry pages | direct |
| I: Astro | audit + implement Starlight 0.38 features, sidebar restructure, components, search, llms.txt regen | 1 (astro-expert) | direct + verify build |

Total writer agent runs: ~78 page-drafts + ~78 verifications + ~78
editorial refines + 7 chapter CDO checkpoints + growth + astro = ~242
agent invocations. Run as background, batched per wave, parallelism
limited to ~10 concurrent.

Cost-control gates: after Wave B (foundation), STOP and report to user
with sample chapter for sign-off before committing C-G resources. Wave
A is safe to run pre-checkpoint (surgical, low-risk).

## 8. Single-writer rule + worktree discipline

- All file writes happen in `/Users/danielmeppiel/Repos/awd-cli-docs-rewrite`
  on branch `docs/full-rewrite-persona-ramps`.
- Subagents may READ but must NOT write directly to disk; they return
  page content as agent output. Orchestrator is the only writer.
- This avoids race conditions and keeps subagents stateless.
- Commits: one commit per wave on green CDO seal. Conventional commit
  prefixes: `docs(wave-a): apply P0 fixes`, `docs(wave-b): foundation
  -- index, quickstart, concepts/`, etc.

## 9. Pattern selections (genesis tier-3/2)

- Architectural (tier 3): WAVE EXECUTION (sequential waves with intra-
  wave fan-out), STAFFED PLAN (fixed staffing chart per page), PANEL
  realized as FAN-OUT + SYNTHESIZER (chapter CDO synthesizes verified+
  refined pages).
- Design (tier 2): A8 ALIGNMENT LOOP (CDO checkpoint with iteration),
  B5 ARBITER (CDO single-arbiter with veto), B4 PLAN MEMENTO (this
  doc, reloaded by every spawn), B8 ATTENTION ANCHOR (each spawn gets
  the 3-promise spine + locked decisions verbatim).

## 10. Anti-patterns guarded against

- **Phantom dependency**: docs that reference commands/flags that do
  not exist. Mitigation: python-architect verifier MUST run the cmd
  or grep src/ for the symbol before the page passes.
- **Bundle leakage**: contributor docs in user nav. Mitigation:
  CDO disposition retires `reference/primitive-types.md` and similar.
- **Goal drift**: writers solve different problems than the disposition
  asked. Mitigation: each writer prompt includes the EXACT D1 gap row,
  D3 disposition, and target audience persona.
- **Editorial bloat**: editor adds prose. Mitigation: editor's prompt
  measures success in lines REMOVED + reading-grade-level reduced, not
  added.
- **CDO infinite loop**: max 2 CDO cycles per chapter; cycle 3 escalates
  to user with diff.

## 11. Evals (pre-flight + post-flight)

Pre-flight per page: writer prompt MUST quote the D1 evidence row + the
ground-truth source path. If the prompt cannot cite a source path, the
page is not ready to write -- return to CDO disposition for clarification.

Post-flight per chapter: CDO acceptance test:
1. Does this chapter open with the persona's goal in one sentence?
2. Does each page answer "what do I type next" in the first viewport?
3. Do bridge sentences carry the reader from this page to the next?
4. Does the chapter end with a clear handoff (to another chapter or to
   reference)?
5. Are all 3 promises visible at least once across the chapter?

Build gate (always): `npm run build` in `docs/` succeeds; link
validator passes; no broken anchors.

CI lint gate: not applicable (docs only); but markdown front-matter
must remain valid; ASCII-only enforced.

## 12. Reload checkpoints

Every subagent prompt MUST include:

1. The 3-promise canonical strings (locked, see decisions D5).
2. The encoding rule (ASCII only, no emoji/unicode).
3. The locked harness list (D1).
4. The page's D1 row + D3 disposition + D5 new-page entry (whichever
   apply).
5. A relative path to read this AGENTIC-SYSTEM-DESIGN.md if needed.
6. Output format spec (full file content + a SOURCE TABLE listing
   every code-truth claim with src/ citation).

## 13. PR plan

- Title: `docs: full corpus rewrite -- persona ramps + verified-against-code`
- Base: main
- Body: link to this design doc + CDO synthesis + per-wave commit list
  + screenshots of new landing + persona-ramp cards.
- Reviewers: maintainer + (post-merge) docs-readers from the broader
  team.
- Draft until Wave I completes and `npm run build` passes locally.

## 14. Open todos (SQL)

Tracked in session SQL `todos` table; statuses updated wave-by-wave.
