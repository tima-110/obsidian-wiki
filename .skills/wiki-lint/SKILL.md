---
name: wiki-lint
description: >
  Audit and maintain the health of the Obsidian wiki. Use this skill when the user wants to check their
  wiki for issues, find orphaned pages, detect contradictions, identify stale content, fix broken wikilinks,
  or perform general maintenance on their knowledge base. Also triggers on "clean up the wiki",
  "what needs fixing", "audit my notes", or "wiki health check". Add --consolidate to switch from
  report-only to act-and-report mode (the "dream cycle"): fixes broken links, adds missing cross-references
  for orphans, normalizes tag aliases, and adds contradiction callouts — all with a dry-run preview and
  explicit user confirmation before any writes.
---

# Wiki Lint — Health Audit

You are performing a health check on an Obsidian wiki. Your goal is to find and fix structural issues that degrade the wiki's value over time.

**Before scanning anything:** follow the Retrieval Primitives table in `llm-wiki/SKILL.md`. Prefer frontmatter-scoped greps and section-anchored reads over full-page reads. On a large vault, blindly reading every page to lint it is exactly what this framework is built to avoid.

## Before You Start

1. Read `.env` to get `OBSIDIAN_VAULT_PATH`
2. Read `index.md` for the full page inventory
3. Read `log.md` for recent activity context

## Lint Checks

Run these checks in order. Report findings as you go.

### 1. Orphaned Pages

Find pages with zero incoming wikilinks. These are knowledge islands that nothing connects to.

**How to check:**
- Glob all `.md` files in the vault
- For each page, Grep the rest of the vault for `[[page-name]]` references
- Pages with zero incoming links (except `index.md` and `log.md`) are orphans

**How to fix:**
- Identify which existing pages should link to the orphan
- Add wikilinks in appropriate sections

### 2. Broken Wikilinks

Find `[[wikilinks]]` that point to pages that don't exist.

**How to check:**
- Grep for `\[\[.*?\]\]` across all pages
- Extract the link targets
- Check if a corresponding `.md` file exists

**How to fix:**
- If the target was renamed, update the link
- If the target should exist, create it
- If the link is wrong, remove or correct it

### 3. Missing Frontmatter

Every page should have: title, category, tags, sources, created, updated.

**How to check:**
- Grep frontmatter blocks (scope to `^---` at file heads) instead of reading every page in full
- Flag pages missing required fields

**How to fix:**
- Add missing fields with reasonable defaults

### 3a. Missing Summary (soft warning)

Every page *should* have a `summary:` frontmatter field — 1–2 sentences, ≤200 chars. This is what cheap retrieval (e.g. `wiki-query`'s index-only mode) reads to avoid opening page bodies.

**How to check:**
- Grep frontmatter for `^summary:` across the vault
- Flag pages without it, **but as a soft warning, not an error** — older pages predating this field are fine; the check exists to nudge ingest skills into filling it on new writes.
- Also flag pages whose summary exceeds 200 chars.

**How to fix:**
- Re-ingest the page, or manually write a short summary (1–2 sentences of the page's content).

### 3b. Missing Relationship Fields (soft warning)

`people/` pages *should* have an `entity:` frontmatter field (wikilink to the org/dept this person belongs to), and `programs/` pages *should* have an `entities:` field (list of wikilinks to entities the program involves). These power Obsidian Bases queries and help ingest skills disambiguate people.

**How to check:**
- Grep `people/` frontmatter for `^entity:` — flag pages without it as a soft warning
- Grep `programs/` frontmatter for `^entities:` — flag pages without it as a soft warning
- Older pages predating these fields are fine; treat as soft warnings only

**How to fix:**
- Check `_meta/stakeholders.md` to determine the correct entity for a people page, then add `entity: "[[entities/<name>]]"` to frontmatter
- For program pages, add `entities: ["[[entities/<name>]]"]` based on program context

### 4. Stale Content

Pages whose `updated` timestamp is old relative to their sources.

**How to check:**
- Compare page `updated` timestamps to source file modification times
- Flag pages where sources have been modified after the page was last updated

### 5. Contradictions

Claims that conflict across pages.

**How to check:**
- This requires reading related pages and comparing claims
- Focus on pages that share tags or are heavily cross-referenced
- Look for phrases like "however", "in contrast", "despite" that may signal existing acknowledged contradictions vs. unacknowledged ones

**How to fix:**
- Add an "Open Questions" section noting the contradiction
- Reference both sources and their claims

### 6. Index Consistency

Verify `index.md` matches the actual page inventory.

**How to check:**
- Compare pages listed in `index.md` to actual files on disk
- Check that summaries in `index.md` still match page content

### 7. Provenance Drift

Check whether pages are being honest about how much of their content is inferred vs extracted. See the Provenance Markers section in `llm-wiki` for the convention.

**How to check:**
- For each page with a `provenance:` block or any `^[inferred]`/`^[ambiguous]` markers, count sentences/bullets and how many end with each marker
- Compute rough fractions (`extracted`, `inferred`, `ambiguous`)
- Apply these thresholds:
  - **AMBIGUOUS > 15%**: flag as "speculation-heavy" — even 1-in-7 claims being genuinely uncertain is a signal the page needs tighter sourcing or should be moved to `synthesis/`
  - **INFERRED > 40% with no `sources:` in frontmatter**: flag as "unsourced synthesis" — the page is making connections but has nothing to cite
  - **Hub pages** (top 10 by incoming wikilink count) with INFERRED > 20%: flag as "high-traffic page with questionable provenance" — errors on hub pages propagate to every page that links to them
  - **Drift**: if the page has a `provenance:` frontmatter block, flag it when any field is more than 0.20 off from the recomputed value
- **Skip** pages with no `provenance:` frontmatter and no markers — treated as fully extracted by convention

**How to fix:**
- For ambiguous-heavy: re-ingest from sources, resolve the uncertain claims, or split speculative content into a `synthesis/` page
- For unsourced synthesis: add `sources:` to frontmatter or clearly label the page as synthesis
- For hub pages with INFERRED > 20%: prioritize for re-ingestion — errors here have the widest blast radius
- For drift: update the `provenance:` frontmatter to match the recomputed values

### 8. Fragmented Tag Clusters

Checks whether pages that share a tag are actually linked to each other. Tags imply a topic cluster; if those pages don't reference each other, the cluster is fragmented — knowledge islands that should be woven together.

**How to check:**
- For each tag that appears on ≥ 5 pages:
  - `n` = count of pages with this tag
  - `actual_links` = count of wikilinks between any two pages in this tag group (check both directions)
  - `cohesion = actual_links / (n × (n−1) / 2)`
- Flag any tag group where cohesion < 0.15 and n ≥ 5

**How to fix:**
- Run the `cross-linker` skill targeted at the fragmented tag — it will surface and insert the missing links
- If a tag group is large (n > 15) and still fragmented, consider splitting it into more specific sub-tags

### 9. Visibility Tag Consistency

Checks that `visibility/` tags are applied correctly and aren't silently missing where they matter.

**How to check:**

- **Untagged PII patterns:** Grep page bodies for patterns that commonly indicate sensitive data — lines containing `password`, `api_key`, `secret`, `token`, `ssn`, `email:`, `phone:` followed by an actual value (not a field description). If a page matches and lacks `visibility/pii` or `visibility/internal`, flag it as a likely mis-classification.
- **`visibility/pii` without `sources:`:** A page tagged `visibility/pii` should always have a `sources:` frontmatter field — if there's no provenance, there's no way to verify the classification. Flag any `visibility/pii` page missing `sources:`.
- **Visibility tags in taxonomy:** `visibility/` tags are system tags and must **not** appear in `_meta/taxonomy.md`. If found there, flag as misconfigured — they'd be counted toward the 5-tag limit on pages that include them.

**How to fix:**
- For untagged PII patterns: add `visibility/pii` (or `visibility/internal` if it's team-context rather than personal data) to the page's frontmatter tags
- For missing `sources:`: add provenance or escalate to the user — don't auto-fill
- For taxonomy contamination: remove the `visibility/` entries from `_meta/taxonomy.md`

### 10. Synthesis Gaps

Identify high-value synthesis opportunities the wiki is missing — concept pairs that co-occur across many pages but have no `synthesis/` page connecting them.

**How to check:**
- List all pages in `synthesis/` — collect the concept pairs each one already covers (from its `[[wikilinks]]` or title)
- Pick 10-15 frequently linked concepts from `concepts/`, `entities/`, and `programs/`
- For each pair, run a quick grep to count pages that link to both:
  ```bash
  grep -rl "\[\[ConceptA\]\]" "$OBSIDIAN_VAULT_PATH" --include="*.md" > /tmp/a.txt
  grep -rl "\[\[ConceptB\]\]" "$OBSIDIAN_VAULT_PATH" --include="*.md" > /tmp/b.txt
  comm -12 <(sort /tmp/a.txt) <(sort /tmp/b.txt) | wc -l
  ```
- Flag pairs with co-occurrence ≥ 3 that have no existing synthesis page

**How to fix:**
- Run `/wiki-synthesize` to automatically discover and fill the top gaps

## Output Format

Report findings as a structured list:

```markdown
## Wiki Health Report

### Orphaned Pages (N found)
- `concepts/foo.md` — no incoming links

### Broken Wikilinks (N found)
- `entities/bar.md:15` — links to [[nonexistent-page]]

### Missing Frontmatter (N found)
- `skills/baz.md` — missing: tags, sources

### Stale Content (N found)
- `references/paper-x.md` — source modified 2024-03-10, page last updated 2024-01-05

### Contradictions (N found)
- `concepts/scaling.md` claims "X" but `synthesis/efficiency.md` claims "not X"

### Index Issues (N found)
- `concepts/new-page.md` exists on disk but not in index.md

### Missing Summary (N found — soft)
- `concepts/foo.md` — no `summary:` field
- `entities/bar.md` — summary exceeds 200 chars

### Missing Relationship Fields (N found — soft)
- `people/jane.md` — no `entity:` field
- `programs/dac.md` — no `entities:` field

### Synthesis Gaps (N found)
Concept pairs that co-occur frequently but have no synthesis page:

| Pair | Co-occurrence | Suggested Action |
|---|---|---|
| [[entities/ssl-labs]] × [[programs/dac]] | 5 pages | Run `/wiki-synthesize` |

### Provenance Issues (N found)
- `concepts/scaling.md` — AMBIGUOUS > 15%: 22% of claims are ambiguous (re-source or move to synthesis/)
- `entities/some-tool.md` — drift: frontmatter says inferred=0.10, recomputed=0.45
- `concepts/transformers.md` — hub page (31 incoming links) with INFERRED=28%: errors here propagate widely
- `synthesis/speculation.md` — unsourced synthesis: no `sources:` field, 55% inferred

### Fragmented Tag Clusters (N found)
- **#systems** — 7 pages, cohesion=0.06 ⚠️ — run cross-linker on this tag
- **#databases** — 5 pages, cohesion=0.10 ⚠️

### Visibility Issues (N found)
- `entities/user-records.md` — contains `email:` value pattern but no `visibility/pii` tag
- `concepts/auth-flow.md` — tagged `visibility/pii` but missing `sources:` frontmatter
- `_meta/taxonomy.md` — contains `visibility/internal` entry (system tag must not be in taxonomy)
```

## After Linting

Append to `log.md`:
```
- [TIMESTAMP] LINT issues_found=N orphans=X broken_links=Y stale=Z contradictions=W prov_issues=P missing_summary=S missing_rel_fields=R fragmented_clusters=F visibility_issues=V synthesis_gaps=G
```

Offer to fix issues automatically or let the user decide which to address.

---

## Consolidate Mode (`--consolidate`)

Triggered by `wiki-lint --consolidate`. Switches from report-only to **act-and-report** — the "dream cycle" that runs periodically so the wiki self-heals.

### Safety protocol

**Always run in dry-run first.** Before writing anything:

1. Run all lint checks (Checks 1–10 above).
2. Print the planned consolidation actions as a structured list (see Dry-Run Output below).
3. Ask the user: `"Apply these N changes? [yes / no / select]"`.
4. Only proceed with writes after explicit confirmation. If the user selects individual actions, apply only those.
5. Never merge pages — use `wiki-dedup` for that. Only link, fix, and flag.

### Consolidation actions (in order, after confirmation)

#### Action 1: Fix broken wikilinks

For each broken `[[Target]]` found in Check 2:
- Search the vault for a page whose title or filename is the closest fuzzy match (use `Grep` across `index.md` titles)
- If a unique best match exists (edit distance ≤ 2 characters or same root word): rewrite the link. Note the rewrite: `[[Original]] → [[corrected-page]]`.
- If no match or ambiguous: convert to plain text and add a comment `<!-- broken link: no match found -->`.
- Never create a new page just to satisfy a broken link.

#### Action 2: Add missing cross-references for orphans

For each orphan page found in Check 1 (zero incoming links):
- Grep the vault body text for mentions of the page's title or aliases (case-insensitive).
- For each mention found in another page, add a `[[wikilink]]` replacing the plain-text mention.
- Limit to 3 insertions per orphan — don't flood pages with links.
- This is scoped to orphans only (different from `cross-linker` which runs broadly).

#### Action 3: Correct lifecycle states

**Skip silently if no pages have a `lifecycle:` frontmatter field.** Only act when the field is present:
- For pages where `lifecycle: draft` AND `created` > 30 days ago: set `lifecycle: reviewed`, `lifecycle_changed: <today>`, `lifecycle_reason: "auto-promoted by wiki-lint --consolidate: age>30d"`.
- For pages where `is_stale = (today − updated) > 180 days` AND `lifecycle: verified`: add a callout at the top of the page body: `> ⚠️ **Stale**: This page was last updated <date>. Verify before relying on it.` Only add if not already present.
- Do not change `reviewed` → `verified` — that is human-only.

#### Action 4: Tier demotion

**Skip silently if no pages have an explicit `tier: supporting` frontmatter field.** Only demote pages that already carry `tier: supporting` AND have 0 incoming links AND haven't been updated in 90+ days. Set `tier: peripheral` and emit a list for the user to review. Never demote `tier: core`.

#### Action 5: Tag normalization

Read `_meta/taxonomy.md` for the alias mapping. For each page, replace known alias tags with their canonical form in the `tags:` frontmatter field. This is a subset of `tag-taxonomy`'s work — only alias fixes, no full audit.

#### Action 6: Contradiction callouts

For each contradiction flagged in Check 5:
- Check whether a `> ⚠️ Contradiction flagged with [[Other Page]]` callout already exists near the relevant claim.
- If not, add it at the end of the "Key Ideas" section (or before "Open Questions" if no "Key Ideas" section). One line only.
- Do not resolve the contradiction; only flag it visually.

### Action 7: Write consolidation report

After all actions, write a report to `synthesis/consolidation-<YYYY-MM-DD>.md`:

```markdown
---
title: Consolidation Report <YYYY-MM-DD>
category: synthesis
tags: [maintenance, consolidation]
sources: []
summary: Auto-generated consolidation report from wiki-lint --consolidate run on <date>.
created: <ISO timestamp>
updated: <ISO timestamp>
---

# Consolidation Report — <YYYY-MM-DD>

## Summary
- Broken links fixed: N
- Cross-references added: M
- Lifecycle states updated: K
- Tier demotions: D
- Tags normalized: T
- Contradiction callouts added: C

## Broken Link Fixes
- `concepts/foo.md:12` — [[OldTarget]] → [[correct-target]]

## Cross-References Added (orphan rescue)
- `concepts/baz.md` — now linked from: [[concepts/alpha]], [[skills/beta]]

## Lifecycle Updates
- `synthesis/stale-verified.md` — stale callout added (last updated <date>)

## Tag Normalizations
- `entities/some-tool.md` — `it-ops` → `it-operations`

## Contradiction Callouts
- `concepts/foo.md` — flagged contradiction with [[synthesis/bar]]
```

### Dry-Run Output (shown before any writes)

```
wiki-lint --consolidate — Dry Run

Planned actions (N total):
[1] Fix broken link: concepts/foo.md:12 [[OldTarget]] → [[correct-target]]
[2] Add cross-ref: concepts/baz.md ← [[concepts/alpha]] (orphan rescue)
[3] Tag alias: entities/some-tool.md: it-ops → it-operations
[4] Contradiction callout: concepts/foo.md ↔ [[synthesis/bar]]

Apply these 4 changes? [yes / no / select by number]
```

### Log entry for consolidate mode

```
- [TIMESTAMP] LINT_CONSOLIDATE links_fixed=N orphans_rescued=M tag_fixes=T contradiction_callouts=C report=synthesis/consolidation-YYYY-MM-DD.md
```

---

## QMD Refresh After Vault Writes

QMD is a search index, not the source of truth. **If `$QMD_WIKI_COLLECTION` is empty or unset, skip this step entirely.** Run only after this skill has written or rewritten vault markdown (i.e., after `--consolidate` writes). If QMD refresh fails, do not roll back the vault changes; report the QMD status separately.

Use `$QMD_CLI` if set; otherwise use `qmd`.

```bash
${QMD_CLI:-qmd} update
```

If the output says vectors are needed or embeddings may be stale, run:

```bash
${QMD_CLI:-qmd} embed
```

Verify with:

```bash
${QMD_CLI:-qmd} ls "$QMD_WIKI_COLLECTION"
```

Record one of:
- `QMD refreshed: update + embed + verified`
- `QMD refreshed: update only + verified`
- `QMD skipped: QMD_WIKI_COLLECTION unset`
- `QMD skipped: qmd CLI unavailable`
- `QMD failed: <short error summary>`
