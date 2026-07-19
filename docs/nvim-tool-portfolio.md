# Neovim tool portfolio

Decisions about which Neovim-ecosystem developer tools to build, in what order, in which language, and for whom. This page is the source of truth for the trajectory that neospec sits inside — expand it as new tools are scoped and as decisions land.

Companion rule: [`rules/tool-language-selection.md`](https://github.com/ocrosby/claude-config/blob/main/rules/tool-language-selection.md) in the personal claude-config repo captures the language-selection matrix in the reusable form. This doc applies that matrix to the specific portfolio.

## Target audience — mental model

Three archetypal Neovim plugin authors anchor every tool decision. Each represents a distinct audience segment; each has a distinct taste; each will endorse or ignore a tool for legible reasons. Naming them explicitly prevents the drift toward "everyone will like this."

| Author | Design center | Who they serve | Taste in tools |
|---|---|---|---|
| **tpope** | Vim-native mechanism, user autonomy — "the plugin gives you a lever, you decide what to lift" | End-users via primitives | No dependencies, no `setup()`, `<Plug>`-first mappings, hand-written vimdoc, deprecation shims that never break silently. Ships zero CI, zero tests. Endorses tools that codify his patterns; will not adopt tools that add plumbing to his own workflow. |
| **tjdevries** | Composable infrastructure, developer ergonomics — "here's the primitive; extend it by writing plain tables that conform to a small contract" | Plugin authors via shared libraries | Composition-first, extension-protocol tables (`{setup, exports, health}`), docs as build artifacts (LuaDoc → vimdoc via docgen), DI at integration boundaries. Builds the missing utility once and puts it in plenary. Politically will not adopt a competing runner; will adopt a coverage-only companion. |
| **theprimeagen** | Tight workflow, teaching artifact — "one workflow, done right; the README teaches you why" | End-users + aspiring plugin authors | Small opinionated surface, user owns keymaps, plugin owns behavior. Iterates aggressively (harpoon v1 → v2). Ships teaching scaffolds (`first-nvim-plugin`, `your-first-plugin`). Willing to try new tooling on smaller repos before flagship. |

### The three questions

Every tool proposal is scored against three questions derived from the archetypes:

1. **Whose day does it improve?** End-user (tpope + prime), plugin author (tj), or aspiring plugin author (prime).
2. **What does the design center on?** A Vim-native mechanism the user composes (tpope), an infrastructure primitive other tools build on (tj), or a single tight workflow the user adopts wholesale (prime).
3. **Is CI/plumbing part of the value?** Yes (tj, prime — both ship CI); no (tpope — never has).

Ideas that hit **two of these axes in the shared region** attract two or more authors. Ideas in a single quadrant attract one. Prioritize the former.

## Candidate tool portfolio

Each tool is scored 0–3 per author on likely-would-use / likely-would-endorse. Ideas favored by 2+ authors sort to the top. Language pick applies the claude-config rule with the driving signal named.

| # | Tool | tpope | tj | prime | Total | Language | Signal |
|---|---|---|---|---|---|---|---|
| 1 | **`plug-audit`** — static analyzer for Neovim plugin repos | 3 | 3 | 2 | **8** | Rust | Joins the stylua/selene shelf; tree-sitter Lua parsing is idiomatic in Rust; performance profile matters when scanning hundreds of files |
| 2 | **`keymap-forensics`** — `:WhyKey <lhs>` runtime diagnosis | 3 | 2 | 3 | **8** | Lua | Runtime editor-inspection work — must run inside Neovim |
| 3 | **`operator.nvim`** — generic operator-func library for Lua-era plugins | 2 | 3 | 2 | **7** | Lua | In-runtime editor API; ideal to merge into plenary later |
| 4 | **`vimhelp.nvim`** + indexer — offline searchable `:h` UX | 3 | 2 | 2 | **7** | Lua + Rust indexer | Full-text search over vimdoc favors `tantivy`; the plugin side is Lua |
| 5 | **`plug-scaffold`** — Neovim plugin equivalent of `cargo new` | 1 | 2 | 3 | **6** | Go | Templating problem; path-dependence favors Go if it lives near neospec's distribution pipeline |
| 6 | **`neospec exec`** — N-Neovim-version matrix runner (subcommand) | 0 | 3 | 3 | **6** | Go | Extends neospec; CI plumbing dominates (HTTP download, subprocess orchestration) |

Score cutoff: build only tools with total ≥ 6. Ideas below that threshold serve fewer than half the audience by design and represent per-audience niches — record them as deferred rather than in the active portfolio.

## Language decisions per tool

Applying [`rules/tool-language-selection.md`](https://github.com/ocrosby/claude-config/blob/main/rules/tool-language-selection.md):

### `plug-audit` → **Rust**

- **First-principles pick.** Tree-sitter Lua parsing is central. Performance-sensitive scans over many files (this scales with plugin repo size). Joins the stylua/selene shelf — ecosystem-consistency is a first-class trust signal to the tj/prime audience.
- **Path-dependence would say:** Go, to share neospec's release pipeline.
- **Resolution:** first-principles wins here. The audience-signal advantage of shipping this in Rust — landing on the same shelf as stylua/selene — is worth the cost of a second release pipeline. This is the tool where the language choice *is part of the value proposition*.

### `neospec exec` → **Go**

- **First-principles pick and path-dependent pick agree.** CI plumbing dominates: HTTP downloads, subprocess orchestration, cross-platform binary distribution — all Go's sweet spot. Shares neospec's existing neovim-download cache. Rewriting neospec in Rust would be pure cost with no first-principles advantage.
- **Resolution:** ship as a `neospec exec <cmd>` subcommand rather than a separate binary. Zero new distribution surface.

### `plug-scaffold` → **Go**

- **First-principles pick: either.** Templating problem, both languages equally capable.
- **Path-dependent tie-breaker:** Go, because it can share the neospec release pipeline and matches the "CLI you can `brew install`" story already established.

### `keymap-forensics`, `operator.nvim` → **Lua**

- Both run inside Neovim, not adjacent to it. Language is fixed by the runtime — no decision to make.
- `operator.nvim` ships as an independent standalone library. Do not design it toward upstream merge into plenary or any other repo — Omar does not maintain authority in those upstreams, and building toward a merge target he can't guarantee introduces cross-repo dependency risk. The library stands on its own; downstream plugins that want it install it directly.

### `vimhelp.nvim` → **Lua plugin + Rust indexer**

- Plugin side is Lua (runtime inspection). Full-text indexer is a separate CLI where Rust's `tantivy` outclasses Go's `bleve` on the specific workload. Two binaries, each in its right language.

## Neospec's placement

Neospec is the anchor of this portfolio. Its role and language choice are already committed and documented in [`neospec/CLAUDE.md`](https://github.com/jedi-knights/neospec/blob/main/CLAUDE.md) and [`neospec/docs/adoption-strategy.md`](https://github.com/jedi-knights/neospec/blob/main/docs/adoption-strategy.md). Summary:

- **Language:** Go. CI plumbing (neovim-download cache, sandbox, GoReleaser distribution) is Go's sweet spot. First-principles pick and current implementation agree.
- **Compatibility invariant:** any file that runs under `PlenaryBustedDirectory` must run under `neospec run` unchanged. This is the wedge that makes adoption possible.
- **Adoption strategy:** primary target is the theprimeagen audience (plenary.busted users with manual Neovim-install CI); secondary target is the tjdevries audience via a coverage-only companion mode; tpope audience is unreachable and not chased.

Neospec's `exec` subcommand (portfolio row 6) is the extension that lets the same binary serve the "N-version matrix runner" audience without splitting distribution.

## Formal scoring — `plug-audit`

The #1-ranked tool in the portfolio, expanded to full architectural-decision-record shape. This section is the working that produced the top-line score; when the tool ships, most of this content migrates to `docs/plug-audit.md` and this section shrinks to a reference.

### Per-author scoring (0–3)

**tpope: 3.** His entire design philosophy — `<Plug>`-first mappings guarded by `hasmapto()`/`maparg()`, `augroup foo | autocmd!` idiom, no `setup()`, `<line1>`/`<count>`/`<bang>`/`<mods>` command-argument protocol, `User` autocmd events as extension points, deprecation shims that `echoerr` the new name — needs to be codified for Lua-era authors who never wrote Vimscript and never picked up the patterns by osmosis. He wouldn't run the linter himself (his workflow doesn't include linters), but he would endorse it because it teaches his taste to the audience he can't reach directly. Endorsement without adoption is a full 3 on this scale — the score measures interest, not personal use.

**tj: 3.** Composition-first authors need mechanical rules for extension-point discipline: `{setup, exports, health}` protocol-table shape, `nvim_create_augroup(name, {clear=true})` lifecycle, `lua/<name>/health.lua` presence, LuaDoc completeness on exported functions, `pcall(require, ...)` around optional peers. These are exactly the invariants tj enforces informally across plenary/telescope/colorbuddy; a linter that enforces them mechanically is his taste made portable. He would both use it *and* champion it — the maximum score condition.

**prime: 2.** Willing to iterate on tooling (harpoon v1→v2 rewrite; refactoring.nvim moved off plenary to mini.test). A "you forgot health.lua" catch is genuinely useful for the small plugins he ships fast. But he's less structural-taste-driven than tj — his interest is workflow, not code shape. He would try it once; if it passes without noise he'd keep it in CI; if it's noisy he'd disable it. Likely-to-use, not likely-to-champion — the definition of a 2.

**Total: 8. Clears the 6-point cutoff for active-portfolio inclusion.**

### Three-question mental model

1. **Whose day does it improve?** All three audiences of plugin author (tpope's users, tj's downstream authors, prime's teaching audience) — one of the rare tools that lands in every quadrant because plugin quality is a shared concern regardless of design philosophy.
2. **What does the design center on?** A Vim-native mechanism (rules codify Vim/Neovim best-practice), implemented as infrastructure (static-analysis primitive) other tools and CIs build on. Hits two of the three centers, misses only "single tight workflow" — appropriate, since a linter *is* infrastructure, not a workflow.
3. **Is CI/plumbing part of the value?** Yes, load-bearing. The tool's primary delivery is a GitHub Action; local CLI is the secondary interface. This matches tj and prime's CI-centric distribution and is the reason tpope endorses without adopting (he doesn't ship CI).

Overlap in the "shared region" across all three questions — the strongest scoring position possible in this framework.

### Language pick

**Rust.** Justification is fully captured in the "Language decisions per tool" section above. Recap of the signal citations from [`rules/tool-language-selection.md`](https://github.com/ocrosby/claude-config/blob/main/rules/tool-language-selection.md):

- **Tree-sitter parsing is central.** Every rule inspects the Lua AST; `tree-sitter-lua` bindings are idiomatic in Rust and native in the target ecosystem.
- **Performance-sensitive scans over many files.** Real plugin repos have hundreds to thousands of Lua files (telescope: ~180k LOC; plenary: ~15 modules × 500 LOC per). Rust's zero-cost abstractions matter at that scale; a Go implementation with `go-tree-sitter` would be 3–5× slower on the same corpus.
- **Ecosystem-shelf consistency.** stylua (Rust), selene (Rust), oxc/biome (Rust for JS/TS analogues). Shipping the Neovim-plugin linter in Rust lands it on the shelf the target audience already trusts — "author read the room" is a first-class trust signal.

Path-dependent argument (Go, to share neospec's release pipeline) is explicitly overridden by first-principles fit. Recorded here so future readers see the tradeoff was considered, not overlooked.

### Initial rule set (v0.1.0)

Five rules, each mapped to a documented pattern from the three-author consensus. Deliberately small — expand only after the first release survives contact with 5+ real plugins.

| Rule ID | Fires on | Source pattern | Severity | Auto-fixable? |
|---|---|---|---|---|
| `augroup-clear` | `vim.api.nvim_create_augroup(name)` or `nvim_create_augroup(name, {})` without `clear = true` | tj / yoda idiom — augroups leak on `:source` without explicit clear | Should Fix | Yes |
| `plug-mapping` | Default `nmap`/`vmap`/`imap` in `plugin/*.lua` bound to a bare `<leader>` or literal key without a `<Plug>` indirection | tpope's core idiom — users must be able to remap | Should Fix | No |
| `health-present` | Plugin repo (has `plugin/*.lua` or `lua/<name>/init.lua`) with no `lua/<name>/health.lua` | tj universal pattern; universal gap in Omar's own jedi-knights suite | Must Fix | No (scaffold provided) |
| `optional-peer-pcall` | `require("<peer>")` where `<peer>` is not `vim`, `plenary`, or a first-party module, without `pcall` guard | tpope's `silent!` idiom translated to Lua era | Should Fix | Yes (wraps in `pcall`) |
| `setup-optional` | Commands defined in `plugin/*.lua` that require `require("<name>").setup(...)` to have been called | tpope invariant — plugin works without `setup()` being called | Must Fix | No |

Each rule maps to a finding shape per [`rules/findings-format.md`](https://github.com/ocrosby/claude-config/blob/main/rules/findings-format.md): file:line, one-sentence what, one-sentence why, fix suggestion. Machine-readable JSON option (`--format=json`) for CI consumption in addition to the human console format.

### Design constraints

- **Zero Neovim install required.** Single Rust binary with `tree-sitter-lua` embedded. This is the tool's parity claim against neospec — both must work in a bare CI environment.
- **Exit-code discipline:** `0` on findings-only, `1` on errors (rule-config invalid, unparseable Lua, I/O failure), `2` on Must-Fix findings when `--strict` is passed. Clippy semantics — do not block CI by default; opt-in strictness for repos ready to gate on it.
- **Config file: `.plug-audit.toml`** at repo root. Precedence: CLI flags > env > TOML > built-in defaults (matches neospec).
- **Rule enable/disable at three levels:** global (`.plug-audit.toml`), file (`-- plug-audit: disable-next-line augroup-clear`), and inline (same syntax as `selene`). Every suppression requires an inline reason per [`rules/lint-suppression.md`](https://github.com/ocrosby/claude-config/blob/main/rules/lint-suppression.md).
- **Findings are stable across runs.** No non-deterministic rule output — no timestamps, no random ordering, no hash-of-input keys in output.
- **Ships as GitHub Action** in the same shape as `jedi-knights/neospec@v0`: composite action, downloads the right binary for `os/arch`, runs the check, exports `passed` and `findings-count` outputs.

### First-release adoption plan

Same shape as neospec's adoption strategy — compatibility first, then case study, then platform.

1. **Compatibility claim in README:** "runs against any Neovim plugin repo, zero config required. Reports issues; does not require you to change your code style unless you opt in with `--strict`."
2. **First test corpus** (order matters — start with lowest-risk to validate rule accuracy, escalate to highest-audience-reach):
   - **Round 1 — internal validation:** Omar's six jedi-knights plugins (yoda, go, python, pytest, invoke, go-task). Known ground truth from the earlier audit — every plugin should produce specific expected findings. If any rule produces surprising findings on this corpus, fix the rule before proceeding.
   - **Round 2 — external validation:** harpoon (prime), git-worktree (prime), plenary (tj), colorbuddy (tj). Run privately; capture findings; verify each finding is genuine (not a false positive) against the author's known style.
   - **Round 3 — case study PR:** pick one plugin from round 2 with a small number of high-confidence findings. Open a PR: "here's what plug-audit found, here are the fixes, here's the resulting CI diff." Link that PR from the README prominently.
3. **Second test corpus** (once round 3 lands): telescope, refactoring.nvim, harpoon2. Broader coverage; each finding shipped through the same case-study path.

### Blockers / open questions

- **Rust proficiency.** Omar's stack is heavy Go, sparse-to-absent Rust (per memory `user_neovim_plugin_work.md` and org survey). First-principles-correct pick for `plug-audit` requires a proficiency ramp before shipping — call this out as a real cost, not a rounding error. Options: (a) accept the ramp as an investment in a second language that also unlocks `vimhelp.nvim` indexer and future portfolio items; (b) build v0 in Go as a path-dependent pick and rewrite in Rust once the rule ontology is stable; (c) partner or contract the initial Rust implementation. **Recommendation: option (a) — the ramp cost amortizes across `plug-audit`, `vimhelp.nvim`, and any future AST-touching tool.**
- **Tree-sitter grammar edge cases.** `tree-sitter-lua` is mature but Neovim uses Lua 5.1 with LuaJIT extensions; some Neovim-specific syntax (`vim.` API dot chains, `---@class` LuaDoc comments) needs verification against the grammar. Test suite for the parser must include real Neovim plugin files, not just synthetic examples.
- **Rule-name ontology stability.** Once a rule name ships, renaming it breaks every downstream `.plug-audit.toml` and inline suppression. Ontology must be right at v0.1.0. Adopt the `selene`/`clippy` convention: kebab-case, category-prefixed (`nvim/augroup-clear`, `nvim/plug-mapping`, `nvim/health-present`) so future categories (e.g., `lua/*` for pure-Lua idioms) fit cleanly.
- **False-positive tolerance.** Every false positive on round-2 corpus is a trust hit; the rule set must be tuned for zero false positives on rules marked Must Fix, and < 5% on Should Fix. If the Round 1 pass shows higher rates, drop the rule to Consider or hold it until the detection improves.

### Ship criteria — what must be true to consider `plug-audit` worth building

All of the following, or the tool goes to the deferred list:

- [ ] Rust proficiency plan in place (option a/b/c above, chosen explicitly)
- [ ] Rule ontology finalized (5 initial rule names + category prefix scheme, documented)
- [ ] Round-1 test corpus scores expected findings on every plugin without surprises
- [ ] Round-2 test corpus produces < 5% false-positive rate on Should Fix rules, 0% on Must Fix
- [ ] Distribution model chosen (`cargo dist` vs GoReleaser-equivalent) and matches neospec's brew-tap/GitHub-Action shape
- [ ] One round-3 case-study PR is scoped (which plugin, which findings, expected diff) before the v0.1.0 release lands

Explicit non-goals for v0.1.0 (record here so scope doesn't creep during implementation):

- No autofix for `plug-mapping` — the `<Plug>` translation is too context-dependent to safely rewrite mechanically.
- No LSP mode. The tool is a batch analyzer; on-save inline diagnostics can come in v0.2.0 as a Neovim plugin wrapping the binary via `lint.nvim` or `nvim-lint`.
- No custom-rule authoring API. All rules ship in-binary at v0.1.0; a plugin API for third-party rules is a v1.0 concern.

## Portfolio evolution rules

- **Score before scoping.** Any new tool idea gets scored against the three-question mental model and the 0–3 per-author matrix *before* implementation planning. Ideas below the 6-point cutoff go to the deferred list rather than the roadmap.
- **Name the language signal.** Every tool in the active portfolio must cite a specific signal from `rules/tool-language-selection.md` for its language pick. "Author's usual language" is not a signal — it's an admission of path-dependence, and if that's the real reason, say so explicitly.
- **When path-dependence and first-principles disagree, name both.** Do not pretend one is the other. The honest form is: "first-principles pick: X (signal Y). Path-dependent pick: Z (existing tool W). Recommendation: Z because [reason]." Both arguments preserved for future readers.
- **Audience-signal counts as a first-class reason.** Shipping a Lua tool in Rust to sit on the stylua/selene shelf is not vanity — it is a legitimate trust-and-adoption signal that shortens the path from "author read the room" to "author is one of us." Rank it alongside performance and ecosystem fit.
- **Update this doc when a tool ships or a target audience shifts.** New named authors get new mental-model rows. Shipped tools get an implementation-notes section below. Cancelled tools stay recorded with the reason, so the same idea doesn't get re-proposed under a different name.

## Deferred / rejected ideas

Record here to prevent re-proposal. Include the reason and the score bar the idea failed to clear.

_None yet — populate as ideas are triaged._
