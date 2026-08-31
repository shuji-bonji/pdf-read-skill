# pdf-read-skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Skill](https://img.shields.io/badge/Claude-Skill-D97757?logo=anthropic&logoColor=white)](https://github.com/shuji-bonji/pdf-read-skill)

🌐 [日本語版 (README.ja.md)](./README.ja.md)

A **Claude Skill** that orchestrates the [PDF family](https://github.com/shuji-bonji#-pdf-family) MCP servers to **read PDFs** — the most frequent job of all: pulling what you need out of a large document, and reading documents whose text cannot be extracted as text.

Where [pdf-trust](https://github.com/shuji-bonji/pdf-trust-skill) audits a PDF you *received* and [pdf-publish](https://github.com/shuji-bonji/pdf-publish-skill) guarantees a PDF you *ship*, pdf-read covers the third path: **get the content out, and say what could not be read.**

## What it does

This repository is a **Skill, not an MCP server**: a Markdown playbook telling Claude how to combine the PDF family MCP tools when a user asks "extract the section about X from this 500-page PDF" or "read this scanned document".

The phases:

| Phase | What happens |
|---|---|
| 0 — Measure | `summarize` (JSON): page count, tagged?, encrypted?, per-page **text extractability** (ISO 32000-2 §9.10.1), and the reader's own `next` suggestions |
| 1 — Branch | encrypted → stop; `no_text_layer` / `not_extractable` pages → Phase 4; tagged → Phase 2; else → Phase 3 |
| 2 — Structure route | `extract_structured_text` (logical content order, real headings), `extract_tables` (page-spanning tables stay whole) |
| 3 — Narrow & extract | `search_text` first on large documents, then `read_text` with an explicit page range (`split_columns` for untagged multi-column, `compact_whitespace` for forms) |
| 4 — Image route | `render_page` (PDFium-WASM) for pages that cannot be read as text; a vision model reads the pixels |
| 5 — Read Report | which pages were read, by which route, and **what could not be read and why** |

Core rule: **an empty extraction result is not evidence that a page has no text.** The reader reports per-page extractability (`extracted` / `no_text_layer` / `not_extractable` / `not_observed`), and this Skill refuses to conclude "not in the document" over pages it could not read.

## Prerequisites

| MCP | Required | Notes |
|---|---|---|
| [pdf-reader-mcp](https://github.com/shuji-bonji/pdf-reader-mcp) **v0.14.0+** | **Yes** | v0.12.0 adds text extractability (#21), `render_page` (#23) and the `next` field (#24) this Skill branches on. v0.14.0 adds `scope` — which of the readings behind an answer were done — and makes a field whose reading did not happen `null`; Phase 0 branches on that. It still runs on earlier versions, but **the stop for an encrypted document does not fire on a password-protected file** there |
| [pdf-spec-mcp](https://github.com/shuji-bonji/pdf-spec-mcp) | Optional | Clause lookups (§9.10.1 / §9.10.2) |

`render_page` needs the reader's optional dependency `@hyzyla/pdfium` (PDFium compiled to WebAssembly). Without it, every other tool works and the Skill reports the affected pages as unread.

## Out of scope

OCR (replaced by render + vision, and declared as such), authenticity auditing (→ pdf-trust), PDF generation and editing (→ pdf-publish).

## License

MIT
