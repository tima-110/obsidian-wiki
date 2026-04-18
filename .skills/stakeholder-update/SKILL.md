---
name: stakeholder-update
description: >
  Update the stakeholder registry with information extracted from wiki pages and reMarkable notes.
  Fills in Objectives and Notes for existing stakeholders, and suggests newly discovered people
  in a Discovered section. Use this skill after running remarkable-ingest or wiki-ingest, or when
  the user says "update stakeholders", "sync stakeholder file", "who did we discover",
  "fill in stakeholder info", or "what do we know about my stakeholders now".
---

# Stakeholder Update — Registry Enrichment

You are enriching a stakeholder registry file with information distilled from wiki pages and source notes. The stakeholder file is used by multiple AI tools — respect its format precisely.

## Before You Start

1. Read `_meta/stakeholders.md` (symlinked from the actual file) — learn the current state of every stakeholder entry
2. Read `index.md` to know what wiki pages exist
3. Read `log.md` to see what was recently ingested (focus on new content)
4. Note the `structure_version` in the stakeholder file frontmatter — preserve it exactly

## Critical Rules

1. **Preserve the file format exactly.** The stakeholder file has a specific indentation-based structure used by other tools. Every stakeholder entry follows this pattern:
   ```
   - <Full Name>
     Email: <email>
     Role: <role>
     Engagement Type: <type>
     Objectives:
       - <objective>
     Notes:
       - <note>
   ```
   Do not change indentation, field order, or field names.

2. **Preserve `structure_version` and all metadata fields.** Update `last_edited` to today's date.

3. **Never remove or reorder existing entries.** Only add to Objectives and Notes fields.

4. **Allowed engagement types** (from the file's own rules): Manager, Peer, Partner, Internal Customer, External Customer, Vendor, Regulator, Executive Sponsor, Direct Report, Other.

5. **Department sensitivity** (from the file's own guidance):
   - PO&T → frame in terms of manufacturing modernization, patient supply, COGM
   - IT → frame in terms of delivery reliability, stakeholder management, service continuity
   - External → frame in terms of scope, deliverables, approvals

## Step 1: Gather Intelligence from Wiki

For each stakeholder in the registry, search the wiki for mentions:

1. **Check `people/` pages** — if a `people/<name>.md` exists, read it for role, decisions, hot topics
2. **Grep the vault** for their name (case-insensitive) across all wiki pages — note which pages mention them
3. **Check `decisions/` pages** — are they listed as a decision-maker?
4. **Check `action-items/` pages** — are they listed as an owner?

Build a profile of what the wiki knows about each stakeholder.

## Step 2: Update Existing Stakeholders

For each stakeholder with new information:

### Objectives
Replace `(none noted)` with concrete objectives discovered in wiki pages. Format as bullet points. Only add objectives that are explicitly stated or strongly supported by multiple sources. Mark inferred objectives with "(inferred)" suffix.

Example:
```
  Objectives:
    - Drive AMS transition to TCS across enterprise apps and labs
    - Establish MSP governance framework (inferred)
```

### Notes
Replace `(none noted)` with relevant context discovered in wiki pages. Keep notes concise — one line per fact. Include the wiki page source in parentheses for traceability.

Example:
```
  Notes:
    - Key decision-maker in TCS vendor selection (from: processes/AMS-transition)
    - Mentioned in GETS Topics re: cyber assessment prioritization (from: entities/GETS)
    - Sponsors the Shift Left program metrics (from: concepts/shift-left)
```

### Rules for updating
- **Don't overwrite existing non-empty Objectives or Notes** — append new items
- **Don't add duplicate information** — check what's already there
- **Keep each note to one line** — this file is a registry, not a narrative
- **Include source attribution** in parentheses — `(from: <wiki-page>)` — so the user can verify
- **Respect department sensitivity framing** when writing notes

## Step 3: Suggest New People

For people discovered in wiki pages who are NOT in the stakeholder registry, append a `## Discovered` section at the **bottom** of the file (before any closing content). Do NOT add them to the department sections — the user promotes them manually.

Format:
```
## Discovered
*People found in wiki pages not yet in the stakeholder registry. Review and promote to the appropriate department section if relevant.*

- <Full Name>
  Context: <what role/department, from which wiki pages>
  Suggested Department: <PO&T | IT | External>
  Suggested Engagement Type: <type>
  First seen: <date>

- <Full Name>
  Context: <context>
  Suggested Department: <department>
  Suggested Engagement Type: <type>
  First seen: <date>
```

**Only suggest people who appear substantively** — mentioned by name with a role, decision, or action attached. Don't suggest every name that appears once in passing.

**If a `## Discovered` section already exists**, merge new entries into it. Don't duplicate people already listed there.

## Step 4: Report

Present a summary to the user:

```markdown
## Stakeholder Update Report

### Updated: N stakeholders
| Stakeholder | Fields Updated | Sources |
|---|---|---|
| Derek Krow | +2 objectives, +3 notes | AMS-transition, MSP-governance |
| ... | | |

### Discovered: M new people
| Name | Suggested Dept | Context |
|---|---|---|
| Hector | IT | GETS leader, cyber/CMMS owner (from: entities/GETS) |
| ... | | |

### No changes: K stakeholders
- <names with no new information found>
```

## Step 5: Update Log

Append to `log.md`:
```
- [TIMESTAMP] STAKEHOLDER_UPDATE stakeholders_updated=N fields_added=F people_discovered=M
```

## Tips

- **Run after every remarkable-ingest.** New notes often mention people in passing who become important later.
- **The Discovered section accumulates.** Review it periodically and promote or dismiss entries.
- **Cross-reference with `people/` pages.** If a `people/` page exists but the person isn't in the stakeholder file, they belong in Discovered.
- **Don't guess email addresses.** Only add emails if explicitly found in source notes.
