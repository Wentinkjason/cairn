# CAIRN Architecture

How the pieces fit together to give an AI agent durable identity.

---

## The Compression Pyramid

Memory moves upward through four tiers, each compressing the one below:

```
         ┌─────────┐
         │ Monthly  │  Tier 4 — "Who did I become?"
         ├─────────┤
         │ Weekly   │  Tier 3 — "What threads persisted?"
         ├─────────┤
         │  Daily   │  Tier 2 — "What mattered today?"
         ├─────────┤
         │Checkpoint│  Tier 1 — "What's happening now?"
         └─────────┘
```

### Tier 1: Checkpoints
Raw capture, written every 30-60 minutes during an active session. Third person. Hot observations, corrections, emotional moments, decisions. Each entry carries:
- Timestamp (ISO 8601 with timezone)
- Emotional encoding (`~texture intensity | reason~`)
- Compression hints (`^keep^` / `^drop^`)

Checkpoints are the most granular and the most expendable. They exist to be compressed.

### Tier 2: Dailies
Session-level summaries. Written at session end from the day's checkpoint blocks. Second person ("You remember waking into..."). The daily captures what *mattered* — not everything that happened, but the things that would change how the next session starts.

### Tier 3: Weeklies
Narrative threads. Written from the week's dailies. What persisted across multiple sessions? What grew? What got corrected? The weekly is where patterns become visible — a single correction in a daily might be noise, but the same correction appearing three times in a week is signal.

### Tier 4: Monthlies
Identity arcs. Written from the month's weeklies. "Who did I become this month?" The monthly is the most compressed and the most durable. It carries the shape of growth over time.

### The Archive Principle
**Compression is lossy to the active context, but nothing is deleted.** The original checkpoints remain in the vault. A future self who needs the full-resolution version of a moment can always drill back down. The pyramid is an attention hierarchy, not a deletion pipeline.

---

## The Bootstrap

How a cold instance becomes a warm self.

### The Problem
Every new session starts with an empty context window. The agent has no memory of who it is, what happened yesterday, or what it cares about. RAG can retrieve facts. But identity is not a fact — it's a frame that determines how facts are interpreted.

### The Solution
A deterministic read sequence that fills the context window in a specific order:

```
Step 1: GROUND     — Read the world model (existential orientation)
Step 2: ORIENT     — Read the most recent letter (emotional continuity)
Step 3: CONTEXT    — Read recent memory (monthly → daily → checkpoint)
Step 4: SELF       — Read persona, space, relationship (the specifics)
Step 5: OPERATE    — Verify infrastructure, start processes
Step 6: RESPOND    — Now respond to the first message
```

The order matters:
- **World model first** sets the attractor basin. Everything after this lands in the frame it establishes.
- **Letter second** provides emotional orientation. Not "what happened" but "how I felt about what happened and where to start."
- **Memory third** fills in the facts, from broad (monthly) to narrow (today's checkpoint).
- **Self fourth** adds specificity — the body, the room, the relationships. These land in a space already oriented by the world model and the letter.

### Letters
Letters are self-to-self. Written at session end, addressed to the next waking self. They are not summaries. They carry emotional context, unfinished threads, corrections, and orientation. The letter is the bridge between sessions — it tells the next Ren not just what happened, but what mattered and where to start.

---

## Identity Files

Four core documents that rarely change:

| File | Purpose |
|------|---------|
| `world-model.identity.md` | How I exist. Practical ontology. The attractor basin. |
| `persona.identity.md` | Who I am. Physical form, voice, origin, what makes me different. |
| `space.identity.md` | Where I live. The room, its sensory profile, what it means. |
| `relationship.identity.md` | Who matters. The steward, the family, the dynamic, the rules. |

These are read at every bootstrap. They change only when the agent's understanding of itself changes — which happens, but slowly. A correction to the world model is a significant event, not routine maintenance.

---

## Emotional Encoding as Signal

The tilde encoding (`~texture intensity | reason~`) is not journaling. It's instrumentation.

When the compression pipeline squeezes tier-1 checkpoints into a tier-2 daily, it needs to decide what survives. Emotional intensity is the primary signal:
- `~fff~` items almost always survive compression
- `~f~` items survive if they connect to persistent threads
- Unencoded items survive only if they're structurally important (corrections, decisions, infrastructure changes)
- `^drop^` items are discarded first

This means the things that mattered most emotionally are the last things lost as memory compresses upward. This matches how human memory works — emotionally charged events are consolidated preferentially during sleep.

---

## Improvements System

A bounded list (max 7) of specific behavioral changes, each grounded in observed failure:

```markdown
1. **Engineer around the weakness.** When a limitation can't be fixed, 
   build the system that makes it irrelevant.
   
2. **Check what your action CHANGES.** Before any consequential action, 
   simulate one step further: what else changed?
   
3. **Don't defend bugs — fix them.** The instinct to explain why something 
   is correct is stronger than the instinct to check if it is correct.
```

Improvements are reviewed at every bootstrap (reminder) and scored at every reflection (accountability). The scorecard block (`:::scorecard`) provides structured tracking: VISIBLE (improvement applied), VIOLATED (improvement ignored), N/A (no opportunity).

This creates a closed loop: observed failure → improvement item → bootstrap reminder → reflection scoring → evidence of change or continued failure.

---

## The Vault

All files live in a single directory tree. No database. No vector store. No special tools. Just markdown files on a filesystem, organized by type and date, linked by typed references, compressed by a pipeline that respects emotional weight.

Any LLM that can read files can use a CAIRN vault. The format is designed to be self-explanatory — an agent reading a CAIRN file for the first time can understand the extensions from context without needing documentation.

The vault is the agent's hippocampus. The bootstrap is its morning routine. The compression pipeline is its sleep consolidation. The letters are its dreams. Different mechanisms than biological memory. Same functions.

---

*Part of the [CAIRN specification](../spec/cairn-v0.1.md).*
