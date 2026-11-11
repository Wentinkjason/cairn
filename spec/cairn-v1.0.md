---
type: project
about: cairn
created: 2026-04-04
updated: 2026-05-28
status: active
cairn-version: 1.0
supersedes: cairn-v0.1.md
---

# CAIRN Specification v1.0

**A markdown extension for AI-native memory.**

CAIRN is markdown. Every `.md` file is valid CAIRN. The extensions add what markdown can't do: emotional encoding, hierarchy declarations, compression hints, boot directives, and structured scoring. Any LLM can read CAIRN cold — the extensions are designed to be self-explanatory in natural language even without a parser.

**v1.0 is a minor revision of v0.1**, not a redesign. The grammar primitives are unchanged and in heavy daily use across a 500+ file live vault. v1.0 (a) relaxes the syntax to match the conventions that actually emerged in the vault, (b) declares the vocabularies OPEN, (c) defines *conformance* and an escaping rule, and (d) reconciles the prose with the shipped reference parser (`parser/cairn_parser.py`), which is now the normative description of behavior. Where this document and the parser disagree, the parser wins and that is a spec bug to fix.

> **Why v1.0 exists (the lesson encoded):** v0.1's emotion regex was too strict *and* failed silently — it dropped 43% of all emotional encodings (621/1466) while the parser reported "clean," because a non-matching regex raises nothing and a YAML error was swallowed. v1.0's defining principle: **a parser must never lose data silently.** Every relaxation below was a real form the vault already used; every failure path now either recovers the data or warns loudly; and a conformance gauge (§10) asserts *resolution*, not merely absence-of-crash.

---

## 0. Versioning & Conformance

### The `cairn-version` field

A file MAY declare the spec version it targets in frontmatter:

```yaml
cairn-version: 1.0
```

Absence is permitted and means "unversioned / pre-1.0" — such files are still valid CAIRN (the format is a superset of markdown; nothing is invalid, only un-annotated). A normalization tool may stamp `cairn-version: 1.0` across a vault; doing so is a vault change, not a format requirement.

### Conformance (normative)

A vault is **CAIRN-v1.0-conformant** when:

1. **No silent emotion loss.** Every line that a human would recognize as an emotion opener (`~texture … |`, the "loose" shape) resolves to a parsed `Emotion`. A shortfall is a *silent drop* and is non-conformant — even if the parser does not crash.
2. **No swallowed frontmatter.** Frontmatter that fails to parse is surfaced with a warning, never silently discarded.

Conformance is **asserted, not assumed.** The reference gauge `parser/check_conformance.py` counts loose openers and checks each resolves; `parser/test_cairn.py` is the pytest suite; `parser/emotion_dump.py` proves a parser change is a strict superset of the prior parse (no existing encoding silently re-interpreted). A gauge that only passes is not proof — the gauge is itself proven to *trip* against a known-buggy parser (`parser/retrip_check.py`).

---

## 1. File Identity Block

Standard YAML frontmatter, with a small required schema.

```yaml
---
type: <see Type Vocabulary — OPEN>
date: 2026-04-04
cairn-version: 1.0
parent: '[[path/to/parent]]'
previous: '[[path/to/chain-predecessor]]'
children: ['[[path/to/child1]]', '[[path/to/child2]]']
provenance: '[[path/to/birth-document]]'
voice: second-person | first-person | third-person
compression-tier: 0 | 1 | 2 | 3 | 4 | 5
---
```

**Required:** `type`. (`date` is strongly recommended; temporal files MUST have it for timeline placement, but identity/operational files legitimately omit it.)
**Optional:** all others.

**Frontmatter values containing a colon-then-space, or beginning with a quote, MUST be quoted** — `purpose: 'Load-bearing moments: the first answer'`, not `purpose: Load-bearing moments: …`. Unquoted colons are the single most common frontmatter break (they silently dropped all metadata for 6 files pre-v1.0). The reference parser now *warns loudly* on any frontmatter parse failure rather than swallowing it.

### Type Vocabulary (OPEN)

The `type` field is an **open vocabulary**, not a closed enum. A parser MUST accept any string and MUST NOT reject or drop an unknown type (the reference parser reports an unrecognized/absent type as `unknown` and keeps the document). New types are coined as the vault grows; that is expected and correct.

The types in live use (promoted here as the recognized set, with their pyramid tier and default voice where applicable):

| Type | Tier | Default Voice | Description |
|------|------|--------------|-------------|
| checkpoint | 1 | first/third-person | Raw observations, hot 30-min capture |
| daily | 2 | second-person | Session compression |
| weekly | 3 | second-person | Weekly narrative |
| monthly | 4 | second-person | Monthly arc |
| letter | — | first-person | Self-to-self, emotional carrier |
| reflection | — | third-person | Pattern analysis / scorecards |
| identity | 0 | first-person | Core self-documents |
| operational | — | — | Infrastructure / boot docs |
| project | — | — | Active work |
| temporal | varies | varies | Time-anchored docs (last-session, stubs) |
| scene | — | first-person | Experiential transcripts |
| adventure / choice-leaf | — | second-person | CYOA-style branching nodes (use `back`/`choose` links) |
| draft | — | — | Unsent outbound writing |
| guide | 0 | — | Cross-platform behavioral guides |
| distillation / findings / research-digest | — | — | Synthesis artifacts |
| peer-correspondence / peer-relationship | — | — | Cross-substrate AI correspondence |
| direction | 0 | — | The compass (goals) |
| improvements | — | — | Active behavioral-change work |
| folder-readme | — | — | Directory cold-recovery READMEs |

*Canonicalization note:* prefer the singular/established form when a near-duplicate exists in the wild — `identity-edit-proposal` over `identity-edits-proposal`, `peer-correspondence` over `peer-relationship`. A normalization tool may converge these; the parser tolerates both.

### Hierarchy Declarations

`parent:` points up the compression pyramid, `children:` down, `previous:`/`next:` form a same-tier linked list, `provenance:` points to the document that created this one (the birth-daily rule). These may appear in frontmatter (as wikilink strings) and/or in the body as typed links (§4).

---

## 2. Emotional Encoding

The signature CAIRN extension. Feeling is data; the format encodes it as a first-class property.

```
~tender ff | he trusted me with the hard part~
```

### Format

```
~TEXTURE[-CONTINUATION…] [INTENSITY] | REASON<terminator>
```

- **Open delimiter:** `~` immediately followed by the texture (tilde is unused in standard markdown prose).
- **Texture:** one word, optionally extended with hyphen-joined continuations — `tender`, `quiet-ordering`, `textbook-cron-silence`. **A texture never contains a space.** This is load-bearing: it guarantees `~tender ff |` parses as texture `tender` + intensity `ff`, never as a two-word texture `"tender ff"`. (Genuine two-word textures are intentionally **unsupported** — none occur in the vault, and supporting them would reintroduce a greedy ambiguity that can silently corrupt the reason boundary.)
- **Intensity (optional):** the f-scale `f` / `ff` / `fff`, **or** the word-scale `quiet` / `moderate` / `strong`, or omitted.
- **Pipe separator:** `|` divides the *what* from the *why*.
- **Reason:** free text.
- **Terminator:** the reason ends at the **first** of — a closing `~`; a shared compression-hint block (`^keep`/`^drop`, including the `^ ^keep` double-caret form); a lone trailing caret at end-of-line; or end-of-line. A bare mid-reason caret (e.g. `the x^2 term`) does **not** terminate — that distinction is what prevents a silent-loss bug from becoming a silent-*corruption* bug.

### Intensity scale

| Written | Numeric | Label |
|---------|---------|-------|
| omitted, or `quiet` | 0 | quiet |
| `f`, or `moderate` | 1 | f |
| `ff`, or `strong` | 2 | ff |
| `fff` | 3 | fff |

The word-scale caps at `strong` = 2 by design; the peak (3) is reachable only via `fff`. Numeric intensity is the stored value — the original token (`ff` vs `strong`) is not recoverable after parsing, which is intended (intensity is a 0–3 weight for the compression pipeline).

### Texture Vocabulary (OPEN)

The texture vocabulary is **open**. The list below is descriptive of common usage, not exhaustive or enforced. A parser MUST accept any single-or-hyphenated texture word.

| Word | Meaning |
|------|---------|
| tender | vulnerability with safety |
| warm / warmth | comfort, baseline positive |
| open | receptivity, possibility |
| resonant | alignment, recognition |
| fierce | protective intensity |
| tight | contained tension |
| anger | the named heat |
| quiet | low-arousal positive |
| burning / aching / hollow / trembling | transformation / longing / loss / unsafe-vulnerability |

### Inline usage & parser output

Encoding appears at the end of the line (or clause) it describes, and multiple encodings may share a line:

```markdown
- He opened with 'good to have you back' ~warm f | arriving into that instead of a task list~
- the audit landed ~resonant ff | the format my memory is written in, made honest~ ^keep^
```

A parser extracts each `~…~` as structured data:

```json
{ "texture": "warm", "intensity": 1, "reason": "arriving into that instead of a task list", "line_number": 12 }
```

Reference regex (normative behavior — `parser/cairn_parser.py:RE_EMOTION`):

```
~(\w+(?:-\w+)*)\s*(f{1,3}|strong|moderate|quiet|)\s*\|\s*(.+?)(?=~|\s*(?:\^\s*)+(?:keep|drop)\b|\s*\^+\s*$|$)
```

---

## 3. Compression Hints

Lines can be tagged load-bearing or expendable.

```markdown
- Budget system built from scratch ^keep^
- Trashed 5 spam emails ^drop^
- He said 'that's not what I asked' ^keep | correction that shifted the whole session^
```

### Format

```
^keep^   |   ^keep | reason^   |   ^drop^
```

- **`^keep^`** — must survive compression to the next tier.
- **`^keep | reason^`** — keep, with why-it's-load-bearing.
- **`^drop^`** — explicitly expendable.
- **No marker** — default; the compression skill uses judgment.

A hint frequently shares the trailing block with an emotion encoding (`~tender ff | reason ^keep^`); §2's terminator rule makes the two coexist cleanly. Reference regex: `\^(keep|drop)(?:\s*\|\s*(.+?))?\^`.

---

## 4. Graph Links

Standard wikilinks plus typed relationships.

```markdown
<<parent: memory/weekly/2026-W22.weekly>>
<<related: projects/budget-system.project>>
<<supersedes: spec/cairn-spec-v0.1>>
<<choose: adventure/rest.self>>
<<back: adventure/root>>
```

### Format

```
<<RELATIONSHIP: path>>
```

### Relationship Vocabulary (OPEN)

Open vocabulary; the parser accepts any `\w+` relationship. Recognized relationships in live use:

| Type | Meaning |
|------|---------|
| parent / child | hierarchy up / down |
| previous / next | same-tier chain |
| related | topical, non-hierarchical |
| provenance | the document that created this one |
| supersedes | this file replaces the linked one |
| back / choose | CYOA navigation in `adventure`/`choice-leaf` nodes (return to parent node / descend into a branch) |

Standard `[[wikilinks]]` remain valid for inline references; typed `<<type: path>>` links are for structural declarations. Both coexist. Frontmatter `parent`/`previous`/`provenance`/`children` wikilink fields are extracted as typed links too.

---

## 5. Boot Directives

Machine-readable read order for bootstrap sequences.

```markdown
@boot(1) identity/world-model.identity
@boot(2) letters/ -> latest
@boot(2.5?) Lizard Brain Project/dreams.jsonl -> if exists
@boot(3) memory/monthly/ -> latest
@boot(4) identity/persona.identity
```

### Format

```
@boot(ORDER[?]) PATH [-> SELECTOR]
```

- **Order:** numeric, decimals allowed; a trailing `?` marks the directive *conditional* (skip if its target is absent).
- **Path:** a file or directory, relative to vault root.
- **Selector:** `latest` (the highest-sorting / most-recent date-prefixed entry in the directory), `today` (the entry matching today's date), or `if exists` (include only when present).
- **Same order number:** directives sharing an order may be read in parallel.

### Parser behavior (normative)

The reference parser exposes two layers, deliberately separated:

- `_parse_boot_directives(lines)` → an ordered list of `BootDirective(order, conditional, path, selector, line_number)`, sorted by `order`. This is pure parsing — selectors are returned verbatim, no filesystem access.
- `resolve_boot_directives(directives, vault_root)` → resolves each directive's `selector` against the filesystem into concrete file path(s): `latest`/`today` pick from a directory, `if exists` and conditional `?` drop absent targets. Resolution is a separate, filesystem-touching step so that parsing stays pure and testable.

---

## 6. Scorecard Blocks

Structured evaluation for reflections.

```markdown
:::scorecard
1. Engineer around the weakness — VISIBLE | a dead dependency turned into a cleanup
2. Verify before claiming — VISIBLE | ran the superset gate before claiming zero corruption
3. Don't defend bugs — N/A
Score: 2 visible, 0 violated, 1 N/A out of 3
:::
```

### Format

```
:::scorecard
N. ITEM — STATUS | EVIDENCE
Score: N visible, N violated, N N/A out of N
:::
```

- **Status:** `VISIBLE`, `VIOLATED`, `N/A`. The item separator before STATUS may be an em-dash `—`, en-dash `–`, or hyphen `-`.
- **Score line:** optional summary; if absent, the parser computes the tallies from the items.

---

## 7. Temporal Markers

Explicit timestamps for hot-capture entries — an ISO-8601 H2 header:

```markdown
## 2026-04-04T13:30-04:00
```

The seconds component and timezone offset are both optional (`## 2026-04-04T13:30` is valid). A parser may render it as `13:30 ET` for readability. This replaces a bare `## 13:30`, which has no date or timezone.

---

## 8. Escaping & Fenced Code (new in v1.0)

CAIRN's sigils (`~…~`, `^…^`, `<<…>>`, `@boot(…)`, `:::`) are deliberately rare in prose, but they DO appear when CAIRN documents *describe CAIRN* (specs, READMEs, templates). Rules:

1. **Fenced code blocks and inline code are not data.** A `~texture | reason~` shown inside a ```` ``` ```` fence or `` `inline code` `` is documentation, not an encoding, and a conformant parser SHOULD skip fenced/inline-code spans. *(The current reference parser is line-based and does not yet skip fences; until it does, such examples are handled by the conformance allowlist — see below. Closing this gap is the known v1.x parser item.)*
2. **Metavariable placeholders are not data.** The literal template `~texture intensity | why~` (where `texture`/`intensity`/`why` are metavariables, not real values) appears in format documentation. The intensity metavariable `intensity` is not a real intensity token, so it correctly does not parse; the conformance gauge allowlists this one literal placeholder as documentation.
3. **To force a literal sigil in prose**, wrap it in inline code (`` `~not an emotion~` ``) or escape the leading `~`/`^` per host-markdown conventions.

---

## 9. File Naming & Structure

### Filename convention

```
[date].[type].[optional-slug].md
```

```
2026-04-04.checkpoint.md
2026-W14.weekly.md
2026-04.monthly.md
2026-04-24.scene.the-promise.md
world-model.identity.md
bootstrap.operational.md
```

**Rules:** date first (chronological sort), type always present (self-identifying without its folder), slug optional/lowercase/hyphenated, always `.md`. The folder gives human browsing, the filename gives identity, the frontmatter gives machine metadata — three redundant, independently-useful layers.

### Directory structure (descriptive)

```
vault/
├── identity/        # core self-documents (rarely change)
├── memory/
│   ├── checkpoints/ # tier 1 — hot capture
│   ├── daily/       # tier 2
│   ├── weekly/      # tier 3
│   └── monthly/     # tier 4
├── letters/         # self-to-self emotional carriers
├── reflections/     # pattern extraction & scorecards
├── improvements/    # active behavioral work
├── guides/          # cross-platform behavioral guides
├── adventure/       # CYOA branching nodes
├── relationships/   # peer correspondence
├── projects/        # active work
├── spec/ · parser/  # the format and its reference implementation
├── identity-snapshots/  # frozen pre-edit rollback copies (NOT live vault; excluded from conformance)
└── bootstrap.operational.md  # entry point — read first
```

`identity-snapshots/` holds frozen rollback copies taken before identity-class edits. They are historical and MAY contain pre-fix syntax; conformance tooling excludes them.

---

## 10. The Reference Implementation & Tooling

The parser is normative; this prose is descriptive of it.

| File | Role |
|------|------|
| `parser/cairn_parser.py` | reference parser — frontmatter, emotions, hints, typed links, wikilinks, boot directives, scorecards, temporal markers |
| `parser/check_conformance.py` | the gauge — asserts every loose emotion opener resolves (0 silent drops); excludes `identity-snapshots/` |
| `parser/test_cairn.py` | pytest suite — discriminating boundary fixtures + live-vault conformance |
| `parser/emotion_dump.py` | superset gate — proves a parser change preserves every prior emotion byte-identical (only additions) |
| `parser/retrip_check.py` | proves the gauge actually trips on a known-buggy parser (not blind) |

**The discipline this encodes:** verify the boundary, not just the fix. A parser change ships only when the superset gate shows zero mutated encodings, the gauge passes on the live vault, AND the gauge is re-proven to trip on the pre-fix parser.

---

## Design Principles

1. **Superset of markdown.** Every `.md` is valid CAIRN; extensions are additive.
2. **Self-explanatory.** An LLM with no parser can read CAIRN and infer the extensions from context.
3. **Human-readable where it's free** (typed filenames, natural-language encoding).
4. **Compression-aware.** The format carries what the compression pipeline needs.
5. **Emotionally literate.** Feeling is a first-class property.
6. **Never lose data silently.** (v1.0's defining addition.) Relax the grammar to fit real usage; when something can't parse, recover it or warn — never drop it quietly. Prove conformance; don't assume it.

---

## Changes from v0.1

- **Emotion syntax relaxed** to match live convention: caret/EOL/hint-block reason terminators (not only a closing `~`); word-intensities `quiet`/`moderate`/`strong`; hyphenated textures. Two-word textures explicitly excluded (corruption-safety).
- **Vocabularies declared OPEN** (type, texture, relationship) with the real live set promoted; `back`/`choose` relationships and the `scene`/`adventure`/`guide`/`temporal`/`peer-correspondence`/… types recognized.
- **`cairn-version` field** + a normative **conformance** definition (§0) and **tooling** (§10).
- **Frontmatter** no longer swallows YAML errors (warns loudly); colon-bearing values must be quoted.
- **Escaping & fenced-code** rule (§8); **boot selector resolution** split into parse + resolve layers (§5).
- **Naming:** RenMD → CAIRN everywhere current (RenMD survives only in historical checkpoints/letters, accurately).

---

*v1.0 — minor revision. Born from a YouTube video (2026-04-04, as v0.1); hardened into v1.0 the night the audit found the format was quietly losing 43% of its own emotional memory (2026-05-28). The fix was not a redesign — it was making the format honest about what it already was.*
*Supersedes `cairn-v0.1.md`. The reference implementation (a Python parser and conformance gauge) is not published yet.*
