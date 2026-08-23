---
name: pdf-read
description: 大きな PDF・読めない PDF から必要な箇所を取り出す読み取りオーケストレーション。pdf-reader-mcp を軸に、summarize で文書の性質（ページ数・タグ・暗号化・テキスト抽出可能性）を測ってから、構造経路（extract_structured_text / extract_tables）・絞り込み経路（search_text → pages 指定の read_text）・画像経路（render_page）のいずれかで読む。読んだ範囲と読めなかった箇所を Read Report で申告する。ユーザーが「この PDF から〜を抜き出して」「PDF の内容を要約して」「この資料の第3章だけ読んで」「スキャン PDF を読んで」「500 ページあるから必要なとこだけ」「PDF の表をデータにして」などに言及したら、単発の read_text で済ませず必ずこの Skill を使う。真正性の監査は pdf-trust、生成・納品は pdf-publish。
---

# pdf-read — 読んだ範囲を申告する PDF 読み取りパイプライン

PDF family の「読む」を担う Skill。pdf-trust が「受け取った PDF を監査する」、
pdf-publish が「送り出す PDF を保証する」のに対し、こちらは最頻の仕事 —
「**この PDF から必要なものを取り出す**」— を、読み落としを黙らせずに行う。

中核原則（family 共通 + read 固有）:

1. **空の抽出結果は「テキストが無い」の証拠ではない。** ISO 32000-2 §9.10.1 は
   「表示はできるが Unicode に変換できない」状態を定義しており、reader は
   ページごとの抽出可能性（extracted / no_text_layer / not_extractable /
   not_observed）を返してくる。**この状態を読まずに本文だけ読まない**
2. **読んだ範囲を申告する。** 全ページを読んでいないなら、どのページを読み、
   どのページを読んでいないかを Read Report に書く。切り詰め（応答上限）に
   当たったら、当たったことを書く
3. **観測と判定を混同しない。** reader の出力は観測。内容の真偽・文書の真正性は
   この Skill の射程外（真正性は pdf-trust へ）
4. 大きな文書で pages を省略しない。read_text / read_images の pages 省略は
   全ページを意味し、応答上限を 1 呼び出しで使い切る

## 前提 MCP

| MCP | 必須/任意 | 役割 |
|---|---|---|
| pdf-reader-mcp（**v0.12.0+ 推奨**） | **必須** | 全経路。**v0.12.0 でテキスト抽出可能性の 3 値化（#21）・render_page（#23）・summarize の next 欄（#24）が入った**。それ未満では Phase 0 の分岐材料が無いため、後述の縮退手順で読む |
| pdf-spec-mcp | 任意 | 抽出可能性の根拠条文（§9.10.1 / §9.10.2）の照会 |

pdf-reader-mcp が未接続なら成立しない。`npx @shuji-bonji/pdf-reader-mcp@latest` の
接続を案内して停止する。

`render_page` は optionalDependencies の `@hyzyla/pdfium`（PDFium の WASM 版）を使う。
未インストールならツールがその旨とインストール方法を返すので、**そのメッセージを
そのままユーザーに伝える**（Phase 4 が使えないことを Read Report に明記する）。

## 手順

### Phase 0 — 測る（summarize）

`summarize` を `response_format: "json"` で 1 回呼ぶ。読み取りに使う観測は:

| フィールド | 使い方 |
|---|---|
| `metadata.pageCount` | 50 超なら Phase 3 で必ず search_text から入る |
| `metadata.isEncrypted` | true なら Phase 1 で停止 |
| `metadata.isTagged` | true なら Phase 2（構造経路）を第一候補にする |
| `textExtractability` | 文書全体の畳み込み。extracted 以外なら `unreadablePages` を見る |
| `unreadablePages` | 状態と原因（フォント名・条項）付き。Phase 4 の対象ページ一覧になる |
| `next` | 観測から機械的に決まる次の一手。**各行が前提の観測名を名乗る**ので、前提を確認してから従う |

v0.12.0 未満の reader には `textExtractability` / `next` が無い。その場合は
`hasText` と `imageCount` だけで推定することになるが、**空ページとスキャンページを
区別できない**旨を Read Report に明記する（推定を観測のように書かない）。

### Phase 1 — 分岐

Phase 0 の観測で経路を選ぶ。複数該当なら該当ページごとに経路を分ける:

- `isEncrypted: true` → **停止**。reader は復号しない（§7.6.2 の暗号文は
  どのツールでも過小報告になる）。パスワードを知っているなら qpdf 等での復号を
  案内し、復号後のファイルで最初からやり直す
- `textExtractability` が `no_text_layer` / `not_extractable` → 該当ページは
  **Phase 4（画像経路）**。extracted のページが混在するなら、そちらは Phase 2/3 で読む
- `isTagged: true` → **Phase 2（構造経路）**
- それ以外 → **Phase 3（絞り込み経路）**

### Phase 2 — 構造経路（タグ付き文書）

1. 文書の骨格が要るなら `inspect_structure`（カタログ・オブジェクト統計）
2. 本文は `extract_structured_text` — 論理コンテンツ順（§14.8.2.5）で、
   見出しレベルは構造ツリー由来の**本物**。座標ソートの read_text より常に優先
3. 表は `extract_tables` — ページを跨ぐ表も 1 つの表として返る
4. 特定の話題だけが要るなら、先に `search_text` でページを絞ってから
   `extract_structured_text` に `pages` を渡す

構造経路でも抽出可能性の状態は付いてくる。`not_extractable` のページが
混ざっていたら、そのページは Phase 4 に回す。

### Phase 3 — 絞り込み経路（タグなし・大きい文書）

1. `pageCount` > 50、または目的が「〜について書いてある箇所」なら、
   **必ず `search_text` から入る**。ヒットしたページ番号が pages 指定になる
2. `read_text` に**明示の `pages`** を渡して読む。タグなし多段組は
   `split_columns: 2 | 3`、帳票・様式は `compact_whitespace: true`
3. `search_text` が 0 件のときは、結果の unsearchablePages / note を読む。
   **読めないページがある場合の 0 件は「読めた範囲に無い」であって
   「文書に無い」ではない** — 該当ページを Phase 4 に回してから結論を出す
4. 応答が切り詰められたら、範囲を分割して読み直す（切り詰めを黙って
   受け入れて「全部読んだ」と書かない）

### Phase 4 — 画像経路（テキストとして読めないページ）

1. `render_page` に **`pages` を明示**して呼ぶ（必須引数）。スキャンは
   `format: "jpeg"`、図面・小さい文字は `dpi: 300`
2. 返った image コンテンツブロックを視覚で読む。読み取った内容は
   「ページ画像からの読み取り」であることを Read Report に明記する
   （テキスト抽出と同じ確度で書かない）
3. 埋め込み画像そのものが目的なら `read_images`（ページの絵ではなく
   画像 XObject を取り出す。用途が違う）
4. `render_page` が使えない（optionalDependencies 未導入）なら、その旨と
   インストール方法（`npm install @hyzyla/pdfium`）を伝えて、該当ページを
   「未読」として Read Report に載せる

### Phase 5 — Read Report

読み取り結果の冒頭または末尾に、次の申告を付ける。1 ページの単純な文書で
全文が読めたなら 1 行で足りる（過剰な様式で本文を埋もれさせない）:

```markdown
## Read Report

- 対象: <ファイル名>（<総ページ数> ページ）
- 読んだ範囲: <ページ列挙 or 全ページ> / 経路: <構造 | 絞り込み | 画像 | 混在>
- 使ったツール: <ツール名の列挙（バージョンが得られれば併記）>
- テキスト抽出可能性: <extracted N / no_text_layer N / ...（reader の申告を転記）>
- 読めなかった箇所: <ページと理由（原因フォント・暗号化・render 不可など）。無ければ「なし」>
- 切り詰め: <あり（対処: 分割読み） | なし>
```

**「読めなかった箇所」を空にしない** — 読めなかった箇所が本当に無いときだけ
「なし」と書く。未確認のまま空欄にすると「確認して問題なし」と誤読される。

## この Skill がやらないこと

- OCR（reader は OCR を持たない。render_page + 視覚モデルの読み取りで代替し、
  その旨を申告する）
- 真正性・改ざんの判定（pdf-trust へ）
- PDF の生成・変換・編集（pdf-publish へ）
- URL 上の PDF への直接の構造検査 — read_url はテキストだけを返す。
  他のツールを使うには先にローカルへダウンロードして file_path で渡す（#25）
