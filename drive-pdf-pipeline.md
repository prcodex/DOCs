# Drive PDF Ingestion Pipeline

> How a research-intelligence RAG platform turns PDFs dropped into a cloud-storage folder into searchable, vectorized rows in its main retrieval corpus — within an hour, with a single Vision-Language Model call per document.
>
> Last verified against running code: **2026-06-08**.

This is a runbook-grade walkthrough of a working production pipeline. Names, IPs, folder IDs, and source institutions are anonymized to placeholders; the architecture, code shape, and operational numbers are real.

## What this subsystem does

Lets a small team dump bank / sell-side / think-tank research PDFs into a shared cloud-storage folder and have them appear — fully extracted, summarized, vectorized, and source-tagged — in the platform's RAG corpus within an hour. No manual ingestion. The corpus is queryable from a chat bot and from internal HTTP endpoints.

There are **two** source folders, classified by folder (not by content analysis):

| Folder placeholder | Classification | Language | Typical sources |
|---|---|---|---|
| `<FOLDER_INTL>` | `international` | English | A dozen global investment banks and macro research houses |
| `<FOLDER_LOCAL>` | `local` | Portuguese | A handful of regional banks/brokers and central-bank analyses |

The folder-as-classifier choice is a deliberate simplification: an upstream filter (which folder you upload to) replaces an expensive downstream classifier (LLM-deciding which corpus the document belongs to). Misclassification is the analyst's responsibility, not the system's.

As of the last verification date, the `<FOLDER_LOCAL>` folder is largely dormant (a handful of PDFs total, none recent) — an open question about whether the regional upload workflow is still in use.

## The two parallel pipelines

This is the most important thing to know about the architecture: there are **two scripts** reading the same `<FOLDER_INTL>`, with different models and different destinations. They don't share dedup state.

| | **Pipeline A (RAG corpus)** | **Pipeline B (Review UI)** |
|---|---|---|
| Script | `drive_ingest.py` (canonical) | `drive_monitor_<port>.py` |
| Runs on | Ingest node (`<GATEWAY_IP>`) | UI node (`<UI_IP>`) |
| Reads | Both folders | Intl only |
| VLM | Single call, cheap-tier model | Two calls: cheap-tier (metadata) + mid-tier (analysis brief) |
| Page cap | **40 pages** (lifted from 8 on 2026-06-08) | 20 pages |
| Vectors | Yes — 2048-dim embedding | No |
| Output table | `unified_feed` (RAG corpus) | `email_analysis` (review UI table) |
| Consumer | RAG bot queries | Human review UI |
| Cron | Hourly (`8 * * * *`) | Every 20 min (`*/20 * * * *`) |
| Cost / PDF | ~$0.30–0.80 (post-lift); ~$0.08–0.16 pre-lift | ~$0.50–1.00 |
| LLM calls / PDF | 1 | 2 |

The split is intentional: Pipeline A feeds the **RAG corpus** that bot queries hit, optimized for breadth and cost. Pipeline B produces a higher-fidelity analyst brief for human review and runs on the same source data through a more expensive model.

## Anatomy of `drive_ingest.py`

The canonical script is ~390 lines. Pure stdlib + an Anthropic SDK + an embedding-provider client + a cloud-storage API client (service-account auth).

### Run cadence

Triggered every hour at `:08` by cron via a thin wrapper. Each run:

1. Iterates both folders (`international`, `local`).
2. Lists PDFs from cloud storage, ordered by `modifiedTime desc`, top 100.
3. Skips anything already in a tracker JSON (`processed_pdfs.json`) by file ID.
4. Skips anything older than **14 days** (`MAX_AGE_DAYS=14`) and marks it `skipped_old` in the tracker so it's not re-checked next hour.
5. Skips duplicates by filename (handles re-uploads with new IDs).
6. Processes up to 5 PDFs per run (`MAX_FILES=5`).

### Per-PDF processing

For each PDF that survives the dedup gauntlet:

1. **Download** via the cloud-storage SDK's media-download into memory (no disk write).
2. **Render pages** with PyMuPDF at **1.5× matrix** → list of base64-encoded PNGs, capped at **`MAX_PAGES=40`** (lifted from 20 on 2026-06-08).
3. **Single VLM call** with all extracted pages as image blocks plus a prompt that asks for:
   - Source institution (from logo/header)
   - Author(s)
   - Title
   - Full content as markdown — tables, data, analysis
   - Explicit exclusions: legal disclaimers, compliance text, copyright, contact info, "professional investors only" warnings, distribution restrictions, risk boilerplate, footers
4. **Parse metadata** from the response with a regex/heuristic pass that handles both `Key: Value` and markdown-header (`## Title`) formats.
5. **Embed** the first 3000 chars of content at 2048 dimensions via the embedding provider.
6. **Write** a `unified_feed` row to a local LanceDB shim on the ingest node.
7. **Sync** to the RAG corpus by POSTing the row to `http://<RAG_IP>:<port>/api/gateway-insert`.
8. **Auto-register** the new `*_drv` source key in the platform's source-classification registry.

### Schema written to `unified_feed`

| Field | Value |
|---|---|
| `id` | `drv_{SourceKey}_drv_{YYYYMMDD_HHMMSS}_{file_id[:8]}` |
| `username` | `{SourceKey}_drv` (e.g. symbolic: `BankA_drv`, `ResearchHouseA_drv`, `AnalystA_drv`) |
| `display_name` | Raw source string from VLM |
| `title` | VLM-extracted title (capped 150 chars), falls back to filename |
| `content_text` | Full VLM markdown output |
| `created_at` / `scraped_at` | Processing time (so items appear at top of feed) |
| `source` | `Drive - {SourceName}` |
| `source_type` | `drive_pdf` |
| `author` | VLM-extracted author |
| `sender` | Same as `display_name` |
| `ai_score` | **8.0** (drive PDFs are trusted institutional research) |
| `hashtags` | `#{SourceKey} #research` |
| `feed_type` | `international` or `local`, from folder config |
| `has_vector` | **1.0** |
| `status` | `enriched` |
| `content_vector` | 2048-dim embedding of first 3000 chars |

### Source-key convention

The institution string is reduced to a single first-word identifier by a tiny helper. Symbolic examples:

- `Some Bank Asset Management` → `Bank`
- `Some Research Group` → `Research`
- `XYZ Securities` → `XYZ` (acronym detection: ≤5 chars, all-upper → preserved)
- `ABC Holdings LLC` → `ABC`

Everything ends in `_drv` so the corpus can identify drive-origin items at a glance.

## The 2026-06-08 page-cap lift

Before this date, the script extracted up to 20 pages from each PDF but **only sent the first 8 to the VLM** (a hidden `images[:8]` slice in `analyze_vlm()`). Long bank reports were silently truncated: a 30-page strategy piece would be summarized from its first 8 pages — usually cover + intro + first chart — missing the meat.

The change lifted both caps:

```python
# before                              # after
MAX_PAGES = 20                        MAX_PAGES = 40
content = [... for img in images[:8]]  content = [... for img in images]
```

The cheap-tier VLM has the context to absorb 40 pages of image content comfortably. The expected cost impact is ~5× per PDF (since image tokens dominate input cost):

| Pages sent | Observed cost / PDF | Daily ceiling (5 PDFs × 24 runs) |
|---|---|---|
| 8 (old) | $0.08 – $0.16 | ~$20/day |
| 40 (new) | ~$0.40 – $0.80 (projected) | ~$100/day |
| Realistic (1–6 new PDFs/run, mostly idle) | — | $2 – $10/day |

## How we run the retrieval — step by step

This section is the runbook: what actually happens between "an analyst uploads a PDF to the shared folder" and "a chat query returns a passage from it." Two halves: **ingestion** (the cron run that puts the PDF into the corpus) and **query-time retrieval** (the RAG path that surfaces it later).

### Half 1: ingestion — the hourly run

The cron fires at `:08` past every hour and walks both folders. The wall-clock anatomy of one PDF being processed (symbolic example, taken from real log output, source name redacted):

```
[11:08:03] DRIVE INGEST - start                    ← script start
[11:08:04]    Found 100 PDFs                       ← cloud-storage list (intl folder)
[11:08:04]    New (recent): 6                      ← survived tracker + age filter
[11:08:07]    ✅ Downloaded                        ← ~3s for service-account fetch
[11:08:08]    ✅ 20 pages                          ← PyMuPDF render at 1.5×
[11:08:08]    🤖 Analyzing 20 pages with VLM...    ← VLM call begins
[11:09:37]    ✅ VLM: 25052 chars, $0.125          ← 1m29s VLM latency
[11:09:37]    Inserted 1 records (RAG corpus)      ← POST to /api/gateway-insert
[11:09:37]    ✅ Stored as ExampleA_drv            ← username pattern
[11:09:37]    📋 Auto-registered ExampleA_drv as international
```

End-to-end per PDF: **~1m34s**, dominated by the VLM call.

### What the VLM call actually looks like

A single `messages.create` call to the cheap-tier VLM per PDF. The payload is one `user` message with a list of image blocks followed by one text block:

```python
content = [
    {"type": "image",
     "source": {"type": "base64", "media_type": "image/png", "data": img}}
    for img in images          # all extracted pages, capped by MAX_PAGES=40
]
content.append({"type": "text", "text": prompt})

response = client.messages.create(
    model="<cheap-tier-vlm>",
    max_tokens=16000,
    messages=[{"role": "user", "content": content}],
)
```

The prompt frames the document as a research piece and asks for **institution, authors, title, full content as markdown** with an explicit *exclude* list (legal disclaimers, compliance text, copyright, contact info, "professional investors only" boilerplate, distribution restrictions, risk warnings, footers/headers). Language is switched at runtime — English for the intl folder, Portuguese for the local folder.

`max_tokens=16000` gives the model enough room for a long-form extraction of dense research; observed outputs range from ~8.5K chars (short note) to ~25K chars (long-form weekly).

### Metadata parsing from the VLM response

The same response carries both the markdown body and the metadata. A lightweight parser scans the first 30 lines for labels — `Company:`, `Institution:`, `Author:`, `Title:` — and also handles markdown-header form (`## Title \n Short hiring note...`). The "next non-empty line" trick lets it survive either layout. Title falls back to filename if the model produced a `#`-prefixed header or no useful title at all.

After parsing, `get_simple_name()` reduces multi-word company strings to a single source key and preserves acronyms. This becomes the username stem appended with `_drv`.

### Embedding step

Content (truncated to first 3000 chars) is embedded at `output_dimension=2048`:

```python
embed_client.embed([text[:3000]], model="<embedding-model>", output_dimension=2048)
```

The 3000-char truncation is a cost/quality trade-off: the embedding model has plenty of context, but the first 3K chars almost always cover the abstract + headline takeaways, which is what semantic queries usually match against. The full markdown still lives in `content_text` and is what gets returned to the user once a row is retrieved.

If the embedding client isn't configured, the script returns a zero-vector — the row still goes in but `has_vector=1.0` would be misleading. In practice the key is always present in production; this is a degraded-mode fallback.

### Storage: local shim → RAG corpus

The row is first written to a **local LanceDB shim** at `<gateway_root>/lancedb_shim/unified_feed`. The "shim" pattern lets the script use the same `db.open_table("unified_feed").add([entry])` API as if it were writing to the canonical corpus — but the local table is empty, used only for schema enforcement. The actual durability comes from the next step.

After the local add, a custom LanceDB wrapper POSTs the row to the RAG host at `http://<RAG_IP>:<port>/api/gateway-insert`. The RAG host performs its own SQLite-backed dedup on insert. Success returns `Inserted 1 records`; the tracker is then updated and the next PDF starts.

### Half 2: query-time retrieval — how a chat query finds these PDFs

Once the row lands in `unified_feed` on the RAG host, it's just another item in the corpus. Retrieval is the same vector-hybrid path every other ingestion goes through:

1. **Query enters** the RAG server via chat bot or HTTP. A cheap-tier classifier reads the query and emits `(international_pct, regional_pct, source_importance, detected_entity)`.
2. **Entity resolution** — if a known source is detected (a bank, a research house), a 5-layer resolver (classifier → DB → display_name → fuzzy pairs → content fallback) finds the matching `*_drv` username because we auto-registered it in the source-classification registry at ingest time.
3. **Hybrid search** is issued against LanceDB — BM25 + dense vector with Reciprocal Rank Fusion. The drive PDF row participates in both channels: `content_vector` is the 2048-dim semantic side, `content_text` is the BM25 side.
4. **Scoring v3** weights the candidates: `semantic * (0.70 - si*0.55) + source * (0.15 + si*0.55) + freshness * 0.15`. Drive PDFs benefit on `source` (`ai_score=8.0` is high and the `*_drv` username is a recognized institutional source). `si` (source_importance) shifts weight toward source vs. semantic — institutional-research queries trigger more source weight.
5. **Institution time override** — when an entity is detected (e.g. "what does <SourceA> say about CPI"), the time window is forced to 7 days. Recent drive PDFs dominate the candidate set for those queries.
6. **Soul + synthesis** — top candidates are passed to a soul-driven mid-tier synthesis. The Drive PDF's `display_name` and `author` fields show up as the source attribution in the final response.

The practical consequence: if a long-form research weekly lands at 14:08 UTC, it's discoverable through a `"what is <SourceA> saying about X"` query within the next hour, with the actual markdown content quoted in the response.

### Running it manually

When you want to force a run (e.g. after dropping a brand-new PDF):

```bash
ssh ingest_node
cd <gateway_root>/scripts
python3 drive_ingest.py    # foreground, with logs to stdout
```

To re-process a PDF that's already in the tracker, remove its entry first:

```bash
python3 -c "
import json
p = '<gateway_root>/data/processed_pdfs.json'
with open(p) as f: t = json.load(f)
t['processed_files'].pop('<file_id>', None)
with open(p, 'w') as f: json.dump(t, f, indent=2)
"
```

To watch the next cron fire:

```bash
tail -f <gateway_root>/logs/drive_ingest.log
```

### Monitoring + debugging

| Symptom | What to check | Where |
|---|---|---|
| New PDF not appearing in bot queries | Did the cron fire? | `tail` the ingest log |
| Cron fired, no PDFs processed | Tracker says "skipped_old"? Folder mtime older than 14d? | tracker JSON + cloud-storage UI mtime |
| Stuck on a PDF | VLM timeout / rate limit | Missing `✅ VLM:` line after `🤖 Analyzing` |
| Row in shim, not in RAG corpus | `/api/gateway-insert` HTTP error | Log line `Inserted 0 records` or HTTP status |
| Bot finds row but content truncated | Old 8-page cap (pre-2026-06-08) | Re-process from tracker |
| Bot can't find by author | Author missing from VLM output | Inspect `content_text` first 200 chars |

## Operations

### Cron entry

```
8 * * * * <gateway_root>/run_script.sh normal drive_ingest drive_ingest.py >> <gateway_root>/logs/drive_ingest.log 2>&1
```

### Logs

`<gateway_root>/logs/drive_ingest.log` — per-PDF lines with cost, page count, char count, and destination. Look for `Inserted 1 records` to confirm the write reached the RAG host, and `Auto-registered X_drv` to confirm the source registration.

### Tracker & dedup

- `<gateway_root>/data/processed_pdfs.json` — JSON map of `file_id → {name, processed, status}`. Three terminal states: normal (no `status`), `skipped_old`, `skipped_duplicate`.
- RAG-side dedup is done by `/api/gateway-insert` against SQLite.

### Credentials

- Cloud-storage service-account JSON at `<gateway_root>/config/service_account.json` (read-only scope).
- VLM and embedding API keys in `<gateway_root>/api_vault.env`.

## Known issues / open questions

- **Local folder is essentially empty.** A handful of PDFs total, none recent. Either the regional upload workflow stopped, or it never started in earnest. Worth asking the team or hiding the folder loop until it's used.
- **No quality alarm on VLM output.** A degraded VLM run (timeout, partial output) writes the truncated `content_text` to `unified_feed` with no flag. Pipeline B has a similar weakness. Worth a min-char-count check before insert.
- **`processed_pdfs.json` grows unbounded.** Mostly cheap (file_id → small dict) but should be rotated/compacted eventually.
- **The two pipelines don't share dedup.** Every PDF gets two independent VLM passes (cheap + mid-tier). That's by design but worth re-confirming whenever cost climbs.

## Changelog

- **2026-06-08** — Page cap lifted from 8 → 40 (`MAX_PAGES` 20 → 40, `images[:8]` slice removed). This page published.
- **Earlier 2026** — Last meaningful edit to `drive_ingest.py` before the cap lift.
- **Late 2025/early 2026** — Initial deployment of the dual-pipeline architecture.
