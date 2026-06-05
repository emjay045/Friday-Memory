# Friday Memory System

A hierarchical long-term memory system with semantic search, deduplication, aging, conflict resolution, and full audit trails. Designed for AI assistant persistence.

## Features

- **Hybrid Search** — TF-IDF + sentence-transformers semantic embeddings for retrieval
- **Structured Schema** — typed facts with subject/predicate/object, tags, confidence, stability
- **Deduplication** — exact match, semantic similarity (cosine ≥ 0.75), and fuzzy Jaccard matching
- **Memory Aging** — confidence decays over time; under-referenced facts eventually archive
- **Conflict Resolution** — contradictory facts are merged, archived, or downgraded
- **Audit Trail** — every create/update/merge/archive/delete is logged
- **Write Locking** — file-based locking for concurrent access safety
- **Automatic Promotion** — frequently confirmed facts promote from temporary → evolving → stable → permanent

## Memory Schema

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

## CLI Usage

```
# Save a fact
python memory.py remember "User prefers dark mode" --type preference --subject user --predicate prefers --object "dark mode" --tags "preference,editor" --confidence 0.9 --stability stable

# Quick save (positional maps to summary)
python memory.py remember "User prefers dark mode" --tags "preference,editor"

# Search (hybrid TF-IDF + semantic)
python memory.py recall "what editor does user use"
python memory.py recall "what editor does user use" --strict      # 0.7 confidence gate
python memory.py recall "what editor does user use" --include-archived --include-stale

# Delete a memory
python memory.py forget fact_1717000000

# View audit trail
python memory.py lineage fact_1717000000

# List all facts (applies aging decay + stability promotion)
python memory.py list
python memory.py list --tag project

# Preload embedding model (faster first recall)
python memory.py warm
```

## Installation

```bash
pip install -r requirements.txt
```

The first `warm` or `recall` command will download the `all-MiniLM-L6-v2` model (~80MB).

## How It Works

### Storage
Facts are stored as JSON in `data/facts.json`. Embeddings are cached in `data/embeddings.json`. Conversations are logged in `data/conversations.json`. All mutations are audited to `data/audit.json`.

### Search
Queries are scored by a hybrid of:
1. **TF-IDF cosine similarity** — on summary and detail fields
2. **Semantic embedding cosine similarity** — via sentence-transformers (`all-MiniLM-L6-v2`)
3. **Combined score** — weighted average of both

### Dedup
On `remember`, duplicates are checked in order:
1. Exact match on `(subject, predicate, object)` — same fact
2. Semantic match — embedding cosine ≥ 0.75
3. Fuzzy match — Jaccard similarity ≥ 0.80 on token sets

If a duplicate is found, the existing fact is **merged** (confidence bumped, tags unioned, stability promoted).

### Aging
Confidence decays 0.2% per day for temporary/evolving facts. Facts older than 180 days without confirmation are deprioritized. At 365 days without reference, they're archived. Permanent facts are exempt.

### Conflict Resolution
Contradictions on `(subject, predicate)` cannot coexist. The system merges (one subsumes the other), archives (outdated version), or downgrades (keeper with lower confidence gets `temporary` stability with a cross-reference).

## License

MIT
