
<p align="center">
  <img src="assets-for-github/logo.svg" alt="OpenIngest logo" width="100">
</p>

<h1 align="center">OpenIngest</h1>

<p align="center">
  <a href="./pyproject.toml">
    <img src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white" alt="Python 3.11+">
  </a>
  <a href="./OpenIngest/serve.py">
    <img src="https://img.shields.io/badge/FastAPI-API-009688?logo=fastapi&logoColor=white" alt="FastAPI">
  </a>
  <a href="./LICENSE">
    <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="MIT License">
  </a>
</p>

OpenIngest is a simple, modular builder of **in-depth AI/RAG knowledge datasets** from ordinary, messy documentation.

I built it while working on a low-level, security-critical local assistant for a finance-software firm. Many GitHub projects focus on novelty, but this project came from a practical need: I couldn't find a tool that would have saved me as much time as this one could have. If it saves even a single developer time, the effort is worthwhile. Building it also helped me understand RAG architecture much more deeply.

It converts DOCX/PDF files into structured records that include source context, hierarchy, optional generated Q&A, embeddings, and traceability fields—so the output is retrieval-ready, not just raw text.

## Table of Contents

- [Why OpenIngest](#why-openingest)
- [Workflow](#workflow)
- [Extreme Configurability](#extreme-configurability)
  - [Custom Summarizer Fields](#custom-summarizer-fields)
- [Output Architecture (Oracle Is an Example)](#output-architecture-oracle-is-an-example)
- [Create a Custom Writer in 5 Minutes](#create-a-custom-writer-in-5-minutes)
- [Quick Start](#quick-start)

## Why OpenIngest

Real documentation is usually not RAG-ready:

- Inconsistent structure
- Long procedural blocks
- Critical meaning hidden in screenshots
- Mixed tables, notes, steps, and troubleshooting

OpenIngest transforms that into structured knowledge with:

- `KONTEKST` — grounded chunk text
- `PITANJE` / `ODGOVOR` — retrieval-friendly intent and concise answers
- Breadcrumbs and page ranges
- Embeddings and model provenance
- Stable IDs (`DOC_ID`, `SECTION_ID`, `CHUNK_ID`)

## Workflow

1. Extract blocks from DOCX/PDF.
2. Extract images and optionally caption them with vision models.
3. Merge image descriptions inline.
4. Compute heading breadcrumbs.
5. Detect parent sections.
6. Split into child chunks using token- and overlap-aware strategies.
7. Generate questions, summaries, and keywords.
8. Embed chunks.
9. Write records through the selected writer.

## Extreme Configurability

OpenIngest is intentionally configurable at every stage.

- **Config sources:** Defaults, JSON/YAML configuration files, and environment variables.
- **Chunking:** Mode (`procedure`, `structure`, `window`, `semantic`), token targets, overlap, heading and keyword heuristics, and OCR language.
- **Enrichment:** Language strategy, image captions, summaries, synthetic questions, and prompts.
- **Custom summarizer fields:** Declare dynamic chunk-level fields (`enum`, `freelist`, `freeparagraph`) and generate them during chunk enrichment.
- **Embedding:** Model, dimensions, batch size, and input template.
- **Pipeline control:** Per-stage enablement (`extract`, `chunk`, `enrich_text`, `embed`, `write`, etc.).
- **Writer mapping:** Configure destination field mappings through `writer.mapping`.

The same core pipeline can be adapted to different documentation styles, data contracts, and storage backends.

### Custom Summarizer Fields

You can declare custom fields in configuration and inject them dynamically into chunk summarization prompts.

Each field declaration supports:

- `name`: Output field name using letters, numbers, and underscores; must start with a letter or underscore.
- `type`: `enum`, `freelist`, or `freeparagraph`.
- `description`: Instructions describing what the model should extract or generate.
- `required`: Optional; defaults to `false`.
- `options`: Required for `enum` and forbidden for other types.

Example:

```yaml
enrichment:
  custom_fields:
    - name: app_id
      type: enum
      description:
        - Classify the application identifier used by this chunk.
        - Use the identifier that best matches the source application or business context.
        - Return exactly one of the configured enum values.
      required: true
      options: ["1", "2", "3", "10"]

    - name: impact_level
      type: enum
      description:
        - Classify the operational impact severity of this chunk.
        - Prefer the highest severity directly supported by the chunk content.
      required: true
      options: [low, medium, high]

    - name: affected_modules
      type: freelist
      description:
        - List all modules or systems explicitly mentioned as impacted.
        - Use concise names only.

    - name: operator_note
      type: freeparagraph
      description:
        - Write a short operator-facing paragraph with the key caution.
        - Keep it direct and actionable.
```

Generated values are stored in `text.custom_fields` in each output record, allowing writer mappings to target paths such as:

- `text.custom_fields.impact_level`
- `text.custom_fields.affected_modules`
- `text.custom_fields.operator_note`

## Output Architecture (Oracle Is an Example)

OpenIngest uses a **writer-driven output architecture**.

- Current built-in writers: `jsonl`, `oracle23ai`.
- The Oracle writer reflects a specific schema used by one application at my current firm.
- It is provided as a **reference implementation**, not a platform limitation.

You can target alternative database architectures by implementing another writer using the same `Writer` interface and selecting it through `writer.kind`.

Oracle is one adapter example; the architecture is extensible by design.

## Create a Custom Writer in 5 Minutes

1. Create a file in `OpenIngest/writers/`, for example `mydb_writer.py`.
2. Implement the `Writer` interface from `OpenIngest.writers.base`.
3. Convert each `ChunkRecord` into your database or API payload.
4. Return a `WriteResult` containing counts and the destination.
5. Register and select your writer through `writer.kind` in configuration.

Minimal example:

```python
from OpenIngest.writers.base import Writer, WriteResult
from OpenIngest.models import ChunkRecord


class MyDbWriter(Writer):
    def write(self, records: list[ChunkRecord]) -> WriteResult:
        # Map records to your storage schema, then insert.
        inserted = len(records)
        return WriteResult(written=inserted, destination="mydb")
```

Configuration example:

```yaml
writer:
  kind: mydb
```

Tip: Start by copying `OpenIngest/writers/oracle23ai.py` or `OpenIngest/writers/jsonl.py` and replacing the mapping and write logic.

## Quick Start

### Backend

```bash
pip install -e .
uvicorn OpenIngest.serve:app --reload
```

### UI

```bash
cd ui
npm install
npm run dev
```

### CLI

```bash
openingest-ingest /path/to/file.pdf --metadata '{"app_id":"10"}'
```

Accepted sources: `.pdf`, `.docx`.

### Common Environment Variables

```bash
OPENAI_API_KEY=...
ORACLE_USER=...
ORACLE_PASSWORD=...
ORACLE_DSN=host/service_name

OPENINGEST_VISION_MODEL=gpt-4.1-mini
OPENINGEST_SUMMARIZE_MODEL=gpt-4.1-mini
OPENINGEST_EMBEDDING_MODEL=text-embedding-3-small
OPENINGEST_VISION_MAX_WORKERS=4
OPENINGEST_OPENAI_MAX_RETRIES=4
OPENINGEST_OPENAI_BACKOFF=1.0
ORACLE_TABLE=RAG_CHUNKS
ORACLE_BATCH_SIZE=50
```
