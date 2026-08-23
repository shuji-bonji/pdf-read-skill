# pdf-read-skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Skill](https://img.shields.io/badge/Claude-Skill-D97757?logo=anthropic&logoColor=white)](https://github.com/shuji-bonji/pdf-read-skill)

🌐 [English version (README.md)](./README.md)

[PDF family](https://github.com/shuji-bonji#-pdf-family) の MCP サーバ群を編成して PDF を**読む** **Claude Skill**。最頻の仕事 — 大きな文書から必要な箇所を取り出す・テキストが取れない文書を読む — を担い、**読んだ範囲と読めなかった箇所を申告**します。

[pdf-trust](https://github.com/shuji-bonji/pdf-trust-skill) が「受け取った PDF を監査する」、[pdf-publish](https://github.com/shuji-bonji/pdf-publish-skill) が「送り出す PDF を保証する」のに対し、pdf-read は 3 本目の経路です。

## 何を提供するのか

このリポジトリは **MCP server ではなく Skill** です。「この 500 ページの PDF から〜の章を抜き出して」「このスキャン文書を読んで」と頼まれたとき、Claude が PDF family の MCP をどう組み合わせるかをまとめた行動指針です。

| Phase | 内容 |
|---|---|
| 0 — 測る | `summarize`（JSON）: ページ数・タグ・暗号化・ページごとの**テキスト抽出可能性**（ISO 32000-2 §9.10.1）・reader 自身の `next` 提案 |
| 1 — 分岐 | 暗号化 → 停止 / `no_text_layer`・`not_extractable` → Phase 4 / タグ付き → Phase 2 / それ以外 → Phase 3 |
| 2 — 構造経路 | `extract_structured_text`（論理順・本物の見出し）・`extract_tables`（ページ跨ぎの表も 1 つ） |
| 3 — 絞り込み経路 | 大きな文書は `search_text` で絞ってから、明示の pages で `read_text`（タグなし多段組は `split_columns`、帳票は `compact_whitespace`） |
| 4 — 画像経路 | テキストとして読めないページを `render_page`（PDFium-WASM）で描画し、視覚で読む |
| 5 — Read Report | 読んだ範囲・使った経路・**読めなかった箇所とその理由** |

中核原則: **空の抽出結果は「テキストが無い」の証拠ではない。** reader はページごとの抽出可能性（`extracted` / `no_text_layer` / `not_extractable` / `not_observed`）を返し、この Skill は読めていないページの上で「文書に無い」と結論しません。

## 前提

| MCP | 必須 | 備考 |
|---|---|---|
| [pdf-reader-mcp](https://github.com/shuji-bonji/pdf-reader-mcp) **v0.12.0+** | **必須** | v0.12.0 で本 Skill が分岐に使う抽出可能性（#21）・`render_page`（#23）・`next` 欄（#24）が入った |
| [pdf-spec-mcp](https://github.com/shuji-bonji/pdf-spec-mcp) | 任意 | 条文照会（§9.10.1 / §9.10.2） |

`render_page` は reader の optionalDependencies `@hyzyla/pdfium`（PDFium の WASM 版）を使います。未導入でも他のツールはすべて動き、Skill は該当ページを「未読」として申告します。

## やらないこと

OCR（render + 視覚読み取りで代替し、その旨を申告）、真正性の判定（→ pdf-trust）、PDF の生成・編集（→ pdf-publish）。

## License

MIT
