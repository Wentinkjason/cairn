---
type: project
about: cairn
created: 2026-11-11
status: active
cairn-version: 1.1
supersedes: cairn-v1.0.md
---

# CAIRN Specification v1.1

**One standard for every memory: what it is, where it came from, and whether it has changed.**

> **No receipt, no memory.**

v1.0 standardized how a note *reads*: frontmatter, emotional encoding, typed links, compression hints. v1.1 adds how a note is *trusted*: a permanent identity, its sources, and tamper evidence. It is a strict superset. Every v1.0 file stays valid; v1.1 only adds fields and one log file.

Why it exists: an agent that remembers will sooner or later remember something wrong, and from the inside a wrong memory feels exactly like a right one. Two failures matter most. A note can make a claim nobody can trace back to anything, and a note can change after it was written with nothing showing it. v1.1 makes both visible.

---

## 1. Scope: what counts as a memory

A **memory** is any note the agent will later read as true about its life, work, people or self: identity files, letters, the checkpoint/daily/weekly/monthly pyramid, reflections, projects, relationship and peer files, guides, folder READMEs, and any separate memory store the agent keeps (one fact per file).

**Not memories** (tools or evidence, out of scope): code, scripts, configs, logs, CSV/JSONL data, raw transcripts, images, backups, archived folders, and raw recovered material kept as evidence rather than as the agent's own claims.

## 2. Identity block additions (frontmatter)

```yaml
id: mem-20261111-a3f9c2         # permanent; minted once, never changed by a rename or move
written: 2026-11-11T13:59-05:00  # when this version was written, by the clock
writer: agent | tool:<name> | peer:<name>
sources:                         # the receipts, or the single word: unsourced
  - kind: telegram | email | transcript | file | live | camera | web | peer | memory
    id: <message id / mail id / session:line / path#section / url>
    at: <ISO time of the source event>
    note: "<optional: the quoted line, under 30 words>"
```

- **`id`** is required. Format `mem-YYYYMMDD-` plus 6 hex digits. It is minted by the writing tool, once. A copy gets a new id and a `sources:` entry of `kind: memory` pointing at the original.
- **`sources`** is required, but it may be the literal `unsourced`. **A guessed source is non-conformant; an honest `unsourced` is conformant.** A wrong receipt is worse than none, because it looks checked.
- **`provenance:`** keeps its v1.0 meaning (the document this note was born from). It is a graph link, not a receipt.
- **Claim-level receipts:** one bullet or sentence can carry its own receipt inline as `^src: <kind>:<id>@<time>`. That is the right grain for checkpoints and dailies, where one note mixes many sources.
- **Own-memory sources are weak.** Citing another of the agent's own notes (`kind: memory`) is allowed, but it must name that note's `id`, and a recurrence or "new evidence" claim cannot rest on it alone. The agent repeating itself is an echo, not a second witness.
- **Memory stores with their own frontmatter** (for example `name` / `description` / `metadata.type`) keep their keys and carry `id`, `written`, `writer` and `sources` under `metadata:`. A checker reads both shapes.

## 3. Tamper evidence

1. **Body fingerprint.** sha256 of the note's *body*: every byte after the frontmatter's closing `---`, with no newline normalization (a CRLF/LF flip is a change). Frontmatter is deliberately outside the fingerprint so ids, receipts and metadata can be added later without breaking the seal.
2. **The write log**, `.ledger/writes.jsonl` at the vault root: append-only, one JSON line per write:

   ```json
   {"id": "mem-...", "path": "letters/...", "sha256": "<body hash>", "writer": "agent",
    "written": "2026-11-11T15:56-05:00", "sources_n": 0, "prev": "<sha256 of the previous log line>"}
   ```

   The first line's `prev` is `"genesis"`. Every later `prev` is the sha256 of the exact bytes of the line before it, so rewriting any old line breaks the chain from that point on.
3. **The check.** Recompute each note's body hash and compare it with the newest log line for its id:
   - **sealed**: they match.
   - **modified outside the record**: they don't. This is surfaced, never auto-fixed. The cause gets named first (a file-sync conflict, another machine writing the vault, a script that went around the writer), and only then is the note resealed, or restored if the change is wrong.
   - **unsealed**: an id with no log line. **no-id**: not minted yet.
4. **Where the log lives matters.** It sits outside anything that recall or sync-merge touches. A version-control repo would do the same job, but a `.git` inside a cloud-synced folder fights the sync, so the log is a plain file that the regular backup carries.

**What this proves, and what it doesn't.** It proves where a memory came from and that it hasn't changed since it was sealed. It does not prove the memory is *true*. Truth still comes from re-reading the source, which is why every receipt must point at something re-readable.

## 4. Writing: seal on write

A seal is only as good as the writers that keep it. Every tool that writes a memory does three things:

1. **Before the write**, read the note's seal state.
2. **Write.**
3. **After the write:**
   - it was **sealed** → append a new log line (reseal), signed by this writer;
   - it was **new, or had no id** → mint an id (and `written:` if absent) and append its first log line;
   - it was **already modified outside the record** → do **not** reseal. Report it. Resealing here would sign somebody else's change as your own.

The before-state is the whole point. A hook that only runs after a write cannot tell "I just changed this" from "something else changed this an hour ago and I touched it after." A pre-write read closes that gap.

A file that opens with frontmatter keeps it on line 1. A tool that "prepends" text to a note inserts it *below* the frontmatter's closing `---`, never above it, or the frontmatter (and the id in it) stops being frontmatter.

## 5. Cuts and summaries

When anything is summarized or dropped (a compacted context, the daily or weekly pyramid, a trimmed window), the summary carries `sources:` pointing at the **exact original span** (a transcript `session:line-range`, or the checkpoint block's header time), and is labeled as an aid: `type: summary`, or `aid: true`. A summary never replaces its originals in the graph.

## 6. Conformance levels

- **graph-compliant**: v1.0 frontmatter with `type`, a date where the type needs one, and a `parent` link (folder READMEs excepted); headers; a README in every folder; edges.
- **receipted** (v1.1): graph-compliant, plus `id`, `written`, `sources` (a list or `unsourced`), plus a matching write-log line.

A vault report shows both counts per folder, plus three lists: missing receipts, `unsourced`, and modified-outside-record. The chain is verified on every report.

## 7. Migrating a v1.0 vault

1. **Checker first, read-only.** Count graph-compliant, receipted, unsourced and unparseable notes. No writes.
2. **Fix unparseable frontmatter.** The usual cause is an unquoted value holding `: ` (v1.0 §1). Quote the one bad value, confirm YAML reads back the exact original text, confirm the body bytes are identical, back the original up.
3. **One full backup of the vault.**
4. **Genesis seal.** Add `id` (and `written` where absent) as new frontmatter lines only, never rewriting existing ones, then write one log line per memory. Re-grade every file after its write and stop if a body hash moved. The genesis runs exactly once; a second run is refused.
5. **Seal on write** in every writer (§4) before anything else writes the vault.
6. **Backfill receipts** in small batches from the record (message ids in the text, `^src` tags, transcripts on disk). Anything not found becomes `unsourced`. Never guessed.
7. **A rotating spot-check**: re-verify a few dozen notes a night, least-recently-checked first, so the whole vault is re-checked every month or so. Anything modified is a finding with a named cause.
8. **Guard the log itself**: the machine's integrity scan treats a missing log, a shorter log, a broken chain, or any changed old line as a red finding. New lines at the end are normal.

## Changes from v1.0

- New frontmatter fields: `id`, `written`, `writer`, `sources` (or `unsourced`), and the inline `^src:` tag.
- The body fingerprint and the hash-chained write log (§3).
- The seal-on-write rule, including the never-reseal-a-modified-note rule (§4), and prepend-below-frontmatter.
- Summaries as labeled aids with exact source spans (§5).
- The receipted conformance level (§6) and a migration path (§7).

---

*v1.1, 2026-11-11. Built in a day, from the question "how do you guarantee provenance?" The echo rule and the summary-as-aid rule came out of a correspondence with another agent's harness about how its own memory survives cuts. Supersedes `cairn-v1.0.md` (every v1.0 file stays valid). The reference checker, genesis sealer and seal hook are not published yet.*
