# Friday - An AI Memory System

**Atomic, file-based long-term memory for AI assistants. One fact, one entry. No database, no daemon, no cloud.**

Friday Memory stores what an assistant knows as structured, single-concept facts in a plain JSON file. Every fact gets a confidence score, a stability level, an audit trail, and a lifecycle: it ages, it decays, it gets confirmed, merged, or archived. Retrieval is hybrid TF-IDF + semantic embeddings, filtered by confidence and freshness. Everything is auditable. Everything is a file you can read, backup, and own.

It is the lightweight alternative to memory servers that turn "remember what I said" into a distributed system.

## Why this exists

Most memory systems do two things differently:

- **They are servers.** A daemon, a port, a database, hooks, an SDK, a cloud. For a single user with a single assistant, that is a lot of machinery to remember what color your editor theme is.
- **They store blobs.** Entire sessions get captured and distilled later, which means facts are fuzzy, redundant, and hard to audit.

Friday Memory is the opposite: explicit, atomic, and self-contained. Each `remember` call creates exactly one fact. Each fact has a subject, predicate, object, confidence, stability, origin, and a full lineage. You can read the entire memory store with a text editor.

## Quick start

```bash
pip install -r requirements.txt
python memory.py warm          # preload embedding model (first recall is faster)

# Save a fact (structured)
python memory.py remember "Alex prefers dark mode" \
  --type preference --subject Alex --predicate prefers --object "dark mode" \
  --tags "preference,editor" --confidence 0.9 --stability stable

# Or quick, positional (maps to summary)
python memory.py remember "Alex prefers dark mode" --tags "preference,editor"

# Search (hybrid TF-IDF + semantic)
python memory.py recall "what editor does Alex use"
python memory.py recall "what editor does Alex use" --strict   # confidence >= 0.7
python memory.py recall "what editor does Alex use" --tag editor

# Inspect anything
python memory.py list                       # all facts (applies aging + promotion)
python memory.py list --tag project
python memory.py lineage fact_1717000000    # full audit trail for one fact
python memory.py forget fact_1717000000     # delete a memory

# Keep the store healthy
python memory.py integrity check            # find orphans, dupes, broken links
python memory.py consolidate --cluster      # synthesize concept facts from clusters
python memory.py backup                     # timestamped snapshot
python memory.py restore --list
```

## What makes it different

### Atomic facts, not blobs
One concept = one entry. A fact is a first-class object with typed fields, not a paragraph your assistant may or may not parse correctly. Plural facts (`user has 3 dogs`) coexist correctly with singular ones; contradictory facts are resolved, never silently stacked.

### A real lifecycle
- **Confidence** decays 0.2%/day for temporary and evolving facts. Stale facts are deprioritized at 180 days, archived at 365.
- **Promotion**: a fact confirmed enough times upgrades temporary → evolving → stable → permanent. Nothing stays a guess forever, and nothing gets to claim permanence without evidence.
- **Quarantine** exists for low-quality or contradictory inputs before they pollute the store.
- **Conflict resolution**: contradictions on the same `(subject, predicate)` are merged, archived, or downgraded. No silent coexistence.

### Everything is audited
Every create, update, merge, archive, and delete is logged with a reason and source IDs. `lineage` shows the full history of a single fact. You can prove where any belief came from.

### Dedup that actually works
`remember` checks, in order: exact `(subject, predicate, object)` match, semantic cosine >= 0.75, and Jaccard >= 0.80. A match means **merge**, never a second copy. Confidence bumps, tags union, stability promotes.

### Working memory, not just long-term
`focus` maintains a separate active-context layer: topics you're actively working on, which decay when ignored and promote into long-term facts when they stick. Useful for assistants that need to know what you're doing *right now* without polluting the permanent store.

### Own your data
Everything lives in `~/.config/friday/memory/data/`:
- `facts.json` — the memory store
- `conversations.json` — session summaries (separate from facts, by design)
- `audit.json` — every mutation
- `embeddings.json`, `tfidf_cache.json` — retrieval caches

Copy the folder, and the assistant's memory moves with it. No export API needed.

## Schema

```json
{
  "id": "fact_1717000000",
  "type": "preference | project | relationship | workflow | event | identity | goal | habit",
  "category": "broad_grouping",
  "subject": "entity",
  "predicate": "relationship/action",
  "object": "target_value",
  "summary": "human-readable compressed summary",
  "details": { "optional_context": "additional nuance" },
  "source": {
    "origin": "conversation | system | user_import | inferred",
    "timestamp": "2026-06-05T12:00:00+00:00"
  },
  "memory_properties": {
    "confidence": 0.0,
    "importance": 0.0,
    "stability": "temporary | evolving | stable | permanent"
  },
  "retrieval": {
    "tags": ["tag1", "tag2"],
    "embedding_ref": "optional_vector_reference"
  },
  "last_updated": "ISO-8601 timestamp",
  "update_count": 1
}
```

## Commands

| Command | What it does |
|---|---|
| `remember` | Save a fact (structured or quick), with dedup + conflict resolution |
| `recall` | Hybrid TF-IDF + semantic search, confidence/freshness filtered |
| `forget` | Delete a memory by ID |
| `list` | List facts/conversations, with aging decay + promotion applied |
| `save-conv` | Log a conversation summary (kept separate from facts) |
| `consolidate` | Compression pipeline: cluster facts, extract candidates from conversations |
| `lineage` | Full audit trail for a single memory |
| `focus` | Working-memory context: set a topic, list, decay, clear |
| `integrity` | Check or auto-repair orphaned embeddings, duplicates, broken audit links |
| `backup` / `restore` | Timestamped snapshots of the entire store |
| `warm` | Preload the embedding model |

## How retrieval works

Queries are scored by a weighted hybrid:

1. **TF-IDF cosine similarity** over summary and detail fields (cached, zero-cost)
2. **Semantic embedding cosine similarity** via sentence-transformers (`all-MiniLM-L6-v2`, ~80MB, local)
3. Combined score, then filtered by confidence threshold (0.5 general, 0.7 strict), staleness, and archive status

Searching is semantic: `recall "database performance problem"` can surface a memory saved as "N+1 query fix". No keyword matching required.

## Requirements

- Python 3.10+
- `sentence-transformers` (see `requirements.txt`; model downloads on first `warm`/`recall`)

## Design notes / tradeoffs

- **Single-user by design.** The write lock is file-based and short-lived. If you need multi-process concurrent writes at scale, this is not the tool.
- **Inferred facts are capped.** Anything derived rather than stated starts at confidence <= 0.69 and can never promote without explicit user confirmation. The system does not let guesses masquerade as facts.
- **Privacy-first.** No telemetry, no cloud, no network calls beyond the model download.

## License

MIT
