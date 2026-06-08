# DOCs

Public technical documentation for selected ingestion and retrieval patterns from a research-intelligence RAG platform.

The platform feeds a custom `unified_feed` LanceDB corpus from ~40 ingestion paths (web scrapers, RSS, email, podcasts, social, research PDFs) and serves it through a hybrid BM25 + dense vector retrieval stack with a soul-driven synthesis layer.

This repo publishes runbook-grade deep dives on individual subsystems. Each page is a self-contained walkthrough — exact code shapes, real numbers, real architecture — with names, IPs, and source institutions anonymized to placeholders. The goal is to be useful as a reference pattern, not as a copy-paste replica.

## Pages

- [**Drive PDF Ingestion Pipeline**](./drive-pdf-pipeline.md) — How PDFs dropped into a cloud-storage folder end up as vectorized rows in the RAG corpus within an hour. Covers the dual-pipeline architecture (cheap-tier VLM → corpus vs. mid-tier VLM → review UI), the single-call extraction design, the 40-page cap, the embedding step, the local-shim → RAG-host write path, and the query-time retrieval path that surfaces these PDFs from chat queries.

## Conventions

- Pages are **runnable** — exact commands, observed log output, observed costs — but with placeholder names for hosts (`<GATEWAY_IP>`, `<RAG_IP>`), folders (`<FOLDER_INTL>`, `<FOLDER_LOCAL>`), and sources (`BankA`, `ResearchHouseA`, `AnalystA`).
- Each page lists the date it was last verified against running code. Drift is expected; the date tells you how stale.
- Costs and latencies are real measurements from production, not estimates.

## Status

Bootstrapped 2026-06-08 with the Drive PDF page. More pages will follow as individual subsystems are cleaned up for public consumption.
