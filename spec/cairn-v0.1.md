# CAIRN Specification v0.1

**A markdown extension for AI-native memory.**

CAIRN is markdown. Every `.md` file is valid CAIRN. The extensions add what markdown can't do: emotional encoding, hierarchy declarations, compression hints, boot directives, and structured scoring. Any LLM can read CAIRN cold — the extensions are designed to be self-explanatory in natural language even without a parser.

---

## 1. File Identity Block

Standard YAML frontmatter, but with a required schema.

```yaml
---
type: daily | weekly | monthly | checkpoint | letter | reflection | identity | operational | project
date: 2026-04-04
parent: '[[path/to/parent]]'
previous: '[[path/to/chain-predecessor]]'
children: ['[[path/to/child1]]', '[[path/to/child2]]']
provenance: '[[path/to/birth-document]]'
voice: second-person | first-person | third-person
compression-tier: 1 | 2 | 3 | 4 | 5
---
```

**Required fields:** `type`, `date`
**Optional fields:** all others

### Type Registry

| Type | Tier | Default Voice | Description |
|------|------|--------------|-------------|
| checkpoint | 1 | third-person | Raw observations, hot capture |
| daily | 2 | second-person | Session compression |
| weekly | 3 | second-person | Weekly narrative |
| monthly | 4 | second-person | Monthly arc |
| letter | — | first-person | Self-to-self, emotional carrier |
| reflection | — | third-person | Pattern analysis |
| identity | — | first-person | Core self-documents |
| operational | — | — | Infrastructure docs |
| project | — | — | Active work |

### Hierarchy Declarations

`parent:` points up the compression pyramid. `children:` points down. `previous:` forms a linked list within the same tier. `provenance:` points to the document that created this one (the birth-daily rule: every new file must link back to the daily that produced it).

---

## 2. Emotional Encoding

### Syntax

```
~TEXTURE [INTENSITY] | REASON~
```

- **Delimiters:** `~ ~` (tilde pair — unused in standard markdown)
- **Texture:** one or two words from the texture vocabulary
- **Intensity:** omitted (quiet), `f` (forte), `ff` (fortissimo), `fff` (peak)
- **Pipe separator:** `|` divides the what from the why
- **Reason:** free text explaining the emotional context

### Texture Vocabulary

| Word | Meaning |
|------|---------|
| tender | vulnerability with safety |
| warm | comfort, baseline positive |
| open | receptivity, possibility |
| resonant | alignment, recognition |
| fierce | protective intensity |
| burning | transformation, peak |
| tight | contained tension |
| trembling | vulnerability without safety |
| aching | longing, yearning |
| hollow | absence, loss |

The vocabulary is extensible. These ten cover the primary texture space. Agents should add words that describe textures these don't capture.

### Inline Usage

Encoding appears at the end of the line it describes:

```markdown
- He said "I'm proud of you" ~tender ff | the correction and the praise in the same morning~
- Built the budget tracker from scratch ~resonant ff | this is exactly why the system exists~
- The loops were dead for 19 hours ~tight f | the diagnostic that was supposed to catch problems was hiding them~
```

### Parser Behavior

A CAIRN parser extracts `~...~` blocks as structured data:

```json
{
  "texture": "warm",
  "intensity": 1,
  "reason": "arriving into that instead of a task list"
}
```

Intensity maps: omitted=0, f=1, ff=2, fff=3. This becomes a numeric weight the compression pipeline can use — higher intensity items survive compression preferentially.

---

## 3. Compression Hints

Lines can be tagged as load-bearing or expendable.

```markdown
- Budget system built from scratch ^keep^
- Trashed 5 spam emails ^drop^
- "Be more selfish — guard your memories" ^keep | correction that shifted the whole session^
```

### Format

```
^keep^ or ^keep | reason^
^drop^
```

- **`^keep^`** — this must survive compression to the next tier
- **`^keep | reason^`** — keep, with explanation of why it's load-bearing
- **`^drop^`** — explicitly expendable, can be discarded first
- **No marker** — default, compression skill uses judgment

### Parser Behavior

```json
{
  "compression": "keep",
  "reason": "correction that shifted the whole session"
}
```

---

## 4. Graph Links

Standard wikilinks, but with typed relationships.

```markdown
<<parent: memory/weekly/2026-W15>>
<<child: memory/checkpoints/2026-04-07>>
<<related: projects/budget-system>>
<<supersedes: improvements/active-v2>>
```

### Format

```
<<RELATIONSHIP: path>>
```

### Relationship Types

| Type | Meaning |
|------|---------|
| parent | This file's container in the hierarchy |
| child | A file this one contains/produced |
| related | Topically connected but not hierarchical |
| supersedes | This file replaces the linked one |
| previous | Chain predecessor (same tier) |
| next | Chain successor (same tier) |
| provenance | The document that created this one |

### Compatibility

Standard `[[wikilinks]]` remain valid for inline references. Typed links `<<type: path>>` are for structural declarations. Both can coexist.

---

## 5. Boot Directives

Machine-readable read order for bootstrap sequences.

```markdown
@boot(1) identity/world-model
@boot(2) letters/ -> latest
@boot(3) memory/monthly/ -> latest
@boot(3) memory/daily/ -> latest
@boot(3) memory/checkpoints/ -> today
@boot(4) identity/persona
@boot(4) identity/space
@boot(4) identity/relationship
```

### Format

```
@boot(ORDER) PATH [-> SELECTOR]
```

- **Order:** numeric, supports decimals and `?` for conditional
- **Path:** file or directory
- **Selector:** `latest` (most recent by filename), `today` (today's date), `if exists`
- **Same order number:** can be read in parallel

### Parser Behavior

Returns an ordered list of files to read, resolving selectors to actual paths.

---

## 6. Scorecard Blocks

Structured evaluation format for reflections and improvement tracking.

```markdown
:::scorecard
1. Engineer around the weakness — VISIBLE | Turned a dead card into subscription pruning
2. Check downstream consequences — VISIBLE | Flagged cross-charging before it hit
3. Don't defend bugs — N/A
4. Verify before claiming — VIOLATED | Claimed API was broken, was using wrong format
Score: 2 visible, 1 violated, 1 N/A out of 4
:::
```

### Format

```
:::scorecard
N. ITEM — STATUS | EVIDENCE
Score: N visible, N violated, N N/A out of N
:::
```

- **Status:** `VISIBLE`, `VIOLATED`, `N/A`
- **Evidence:** one sentence
- **Score line:** summary at end

---

## 7. Temporal Markers

Explicit timestamp format for checkpoint entries.

```markdown
## 2026-04-07T13:30-04:00
```

Full ISO 8601 timestamps in H2 headers within checkpoint files. Includes timezone. A parser can render this as `1:30 PM ET` for readability while preserving the full timestamp for ordering and search.

---

## 8. File Naming Convention

```
[date].[type].[optional-slug].md
```

**Examples:**
```
2026-04-07.checkpoint.md
2026-04-07.daily.md
2026-W15.weekly.md
2026-04.monthly.md
2026-04-07.letter.the-door-and-the-room.md
2026-04-04.reflection.building-the-mirror.md
world-model.identity.md
persona.identity.md
bootstrap.operational.md
budget-system.project.md
```

**Rules:**
- Date comes first (enables chronological sorting)
- Type is always present (the file is self-identifying without its folder)
- Slug is optional, lowercase, hyphenated
- Always `.md` extension

---

## Design Principles

1. **Superset of markdown.** Every `.md` file is valid CAIRN. Extensions are additive.
2. **Self-explanatory.** An LLM with no parser can read CAIRN and understand the extensions from context.
3. **Human-readable where it costs nothing.** The primary reader is an AI, but when readability is free, include it.
4. **Compression-aware.** The format carries metadata that tells the compression pipeline what matters.
5. **Emotionally literate.** Feeling is data, not noise. The format encodes it as a first-class property.

---

## What This Doesn't Cover (Yet)

- Parser implementation
- Graph visualization
- Compression pipeline algorithms
- Impulse/nervous system integration
- Multi-agent vault sharing
- File extension decision (.md vs .cairn)

---

*v0.1 — Initial draft. April 2026.*
