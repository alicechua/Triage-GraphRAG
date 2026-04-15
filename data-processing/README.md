# data-processing

This folder contains three scripts for mapping raw patient diagnoses (a `dx` column in a CSV) to knowledge-graph nodes and then traversing the graph to surface recommended diagnostic tests.

Three approaches are provided, each with different trade-offs between speed, accuracy, and infrastructure requirements:

| Script | Matching approach | Vector store | Embedding |
|--------|-------------------|--------------|-----------|
| `kg_substring_csv_mapping.py` | Substring (no API needed) | None | None |
| `graphrag_semantic_csv_mapping.py` | Semantic similarity | Qdrant Cloud | Nebius e5-mistral-7b-instruct |
| `graphrag_semantic_csv_mapping_chroma.py` | Semantic similarity | ChromaDB (local) | MedEmbed / MedEIR (local) |

All three scripts expect the input CSV to contain a column named **`dx`** with raw patient diagnosis strings and produce an enriched output CSV with two new columns:

| Column | Description |
|--------|-------------|
| `matched_graph_nodes` | KG nodes matched to each diagnosis, with node type and confidence |
| `potential_tests` | Recommended diagnostic tests linked from those nodes via `REQUIRES_TEST` edges |

---

## Files

### `kg_substring_csv_mapping.py`

**Purpose:** Match patient diagnoses to KG nodes using case-insensitive substring matching — no API keys or network calls required.

**What it does:**
1. Loads the serialised knowledge graph PKL.
2. Reads unique diagnosis strings from the `dx` column.
3. For each diagnosis (and each `;`-separated term within it), scans every KG node for a substring match (needle ∈ label or label ∈ needle).
4. Traverses `REQUIRES_TEST` and `DIRECTLY_INDICATES_TEST` edges (and 2-hop `INDICATES_CONDITION → REQUIRES_TEST` paths) to collect recommended tests.
5. Writes an enriched CSV.

Also provides the shared helpers imported by both semantic scripts: `get_tests_for_node`, `_run_validation`, `TEST_COLUMN_MAP`.

**Usage:**
```bash
python kg_substring_csv_mapping.py \
    --input-csv /path/to/patient_data.csv \
    --output-csv patient_diagnoses_with_tests.csv
```

**All CLI options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--input-csv` | *(required)* | Input CSV with a `dx` column |
| `--output-csv` | `patient_diagnoses_with_tests.csv` | Enriched output CSV |
| `--graph-pkl` | `knowledge-graphs/triage_knowledge_graph_enriched.pkl` | Serialised graph PKL |
| `--limit N` | *(all rows)* | Process only the first N rows |
| `--resume` | `false` | Skip diagnoses already present in the output CSV |
| `--min-nodes N` | `1` | Min matched nodes recommending a test before it is included |

**No environment variables required.**

---

### `graphrag_semantic_csv_mapping.py`

**Purpose:** Match patient diagnoses to KG nodes using semantic vector search against a Qdrant Cloud `kg_nodes` collection (built by `knowledge-graphs/vector_db/embed_kg_nodes.py`).

**What it does:**
1. Loads the KG PKL for graph traversal.
2. For each unique diagnosis, embeds the text using the Nebius `e5-mistral-7b-instruct` model and queries Qdrant for the top-K most similar KG nodes above a cosine similarity threshold.
3. Traverses test edges to collect recommended tests.
4. Writes an enriched CSV with incremental checkpointing (resumable with `--resume`).

**Usage:**
```bash
python graphrag_semantic_csv_mapping.py \
    --input-csv /path/to/patient_data.csv \
    --output-csv patient_diagnoses_semantic.csv
```

**All CLI options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--input-csv` | *(required)* | Input CSV with a `dx` column |
| `--output-csv` | `patient_diagnoses_semantic.csv` | Enriched output CSV |
| `--graph-pkl` | from `KG_PKL_FILE` env | Serialised graph PKL |
| `--limit N` | *(all rows)* | Process only the first N rows |
| `--resume` | `false` | Skip diagnoses already in the output CSV |
| `--min-nodes N` | `1` | Min matched nodes recommending a test |
| `--semantic-top-k K` | `5` | Max KG nodes retrieved per query |
| `--semantic-threshold F` | `0.50` | Min cosine similarity (0–1) to accept a match |
| `--save-every N` | `10` | Checkpoint to disk every N diagnoses |

**Environment variables required** (in `knowledge-graphs/.env`):
```
NEBIUS_API_KEY=<Nebius AI Studio key>
QDRANT_URL=<Qdrant Cloud cluster URL>
QDRANT_API_KEY=<Qdrant Cloud API key>
```

---

### `graphrag_semantic_csv_mapping_chroma.py`

**Purpose:** Match patient diagnoses to KG nodes using semantic vector search against a local ChromaDB `kg_nodes` collection (built by `knowledge-graphs/vector_db/build_vector_db_chroma.py`). No cloud credentials required for embeddings.

**Additional features vs. the Qdrant version:**
- **Negation handling** — nodes whose label appears after a negation word (`no`, `not`, `denies`, `without`, etc.) within a 6-token window are excluded (e.g. `"no fever"` does not match the Fever node).
- **Score-weighted test ranking** — tests are ranked by the sum of cosine similarities across all matched nodes that recommend them, not by raw vote count.
- **Extra output columns** — `match_scores` (per-node cosine similarity), `test_scores` (per-test aggregate score), and `matches_json` (full match data for downstream use).

**Usage:**
```bash
python graphrag_semantic_csv_mapping_chroma.py \
    --input-csv /path/to/patient_data.csv \
    --output-csv patient_diagnoses_chroma.csv
```

**All CLI options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--input-csv` | *(required)* | Input CSV with a `dx` column |
| `--output-csv` | `patient_diagnoses_chroma.csv` | Enriched output CSV |
| `--graph-pkl` | from `KG_PKL_FILE` env | Serialised graph PKL |
| `--limit N` | *(all rows)* | Process only the first N rows |
| `--resume` | `false` | Skip diagnoses already in the output CSV |
| `--min-score-sum F` | `0.5` | Min aggregate cosine score for a test to appear in output |
| `--semantic-top-k K` | `5` | Max KG nodes retrieved per query |
| `--semantic-threshold F` | `0.50` | Min cosine similarity (0–1) to accept a match |
| `--save-every N` | `10` | Checkpoint to disk every N diagnoses |

**Environment variables required** (in `knowledge-graphs/.env`):
```
MEDEMBED_MODEL=abhinand/MedEmbed-large-v0.1   # or another MedEmbed variant
CHROMA_PATH=./knowledge-graphs/vector_db/chroma_db
KG_PKL_FILE=triage_knowledge_graph_enriched.pkl
```

No Qdrant or Nebius credentials needed. Models are downloaded from HuggingFace on first run.

---

## Output columns

| Column | Scripts | Description |
|--------|---------|-------------|
| `matched_graph_nodes` | all | Pipe-separated matched KG nodes with type and confidence |
| `potential_tests` | all | Comma-separated recommended diagnostic tests |
| `match_scores` | chroma only | Pipe-separated `Label: score` pairs (raw cosine similarity) |
| `test_scores` | chroma only | Pipe-separated `Test: aggregate_score` pairs |
| `matches_json` | chroma only | Full match data as a JSON array for downstream use |
