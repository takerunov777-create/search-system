# 03-search-strategy — 検索戦略・多言語・並列実行

## 役割
各サブ質問に対して検索クエリを設計し、WebSearch + WebFetch で情報を収集する。
独立したサブ質問は並列エージェントで同時実行する。

## 参照パターン
- gpt-researcher の Planner-Executor（クエリ分解・並列投入）
- STORM の Adaptive Query Expansion（クエリ拡張）

## ツール
- `WebSearch` — 一般検索（Claude Code 組み込み）
- `WebFetch` — 特定 URL のコンテンツ取得（Claude Code 組み込み）

---

## 実行手順

### Step 1: 入力を読み込む

`data/{セッション}/questions.json` から統合サブ質問リストを読み込む。

### Step 2: 各サブ質問のクエリセットを設計する

各サブ質問に対して **最低 3 クエリ** を生成する:

```
Q01「主要競合企業は何社か」の場合:
  クエリ1（日本語・一般）: "{対象} 競合 一覧 2024 2025"
  クエリ2（英語・一般）: "{target} competitors list market 2025"
  クエリ3（業界用語）: "{target} alternative products comparison"
  クエリ4（ニュース系）: "{target} 競合 新規参入 2025"
  クエリ5（公式情報）: site:{competitor_domain} about
```

**クエリ設計のルール:**
- 調査テーマの固有名詞は必ず含める
- 日付を含める（直近情報を優先）
- 情報タイプごとにクエリを変える（`site:`, `filetype:pdf`, `"決算"` 等）
- 英語クエリは必ず含める（海外情報をカバー）

### Step 3: 並列実行で検索する

**High 優先度のサブ質問** は Agent tool を使って並列実行する:
- 各エージェントは独立した 1 つのサブ質問を担当する
- エージェントへの指示には以下を含める:
  - 担当サブ質問の ID・テキスト
  - 実行するクエリリスト（Step 2 で設計したもの）
  - 出力先パス: `data/{セッション}/raw/Q{N}.json`
  - 収集ルール（04-data-collection.md の基準を要約して渡す）

**Medium / Low 優先度** は直列で実行する。

### Step 4: 各クエリを実行する（エージェント内の処理）

1. `WebSearch` でクエリを実行し、上位 5〜10 件の URL を取得する
2. 各 URL に対して `WebFetch` でコンテンツを取得する
3. コンテンツが取得できない場合はスキップし、エラーを記録する
4. 取得結果を `data/{セッション}/raw/Q{N}.json` に保存する

### Step 5: Adaptive Query Expansion（クエリ拡張）

初回検索の結果を分析し、不足情報を補うクエリを追加する:

1. 検索結果に頻出するキーワードを抽出する
2. それらを使った追加クエリを生成する（最大 3 クエリ追加）
3. 追加クエリを実行して結果を既存 JSON に追記する
4. **最大 3 ラウンドまで** 繰り返す（無限ループ防止）

**拡張を止める条件:**
- 新規情報が前ラウンドと 30% 以下しか変わらない
- 収集ソース数が 15 件を超えた
- 3 ラウンド完了した

### Step 6: 鮮度フィルタ

取得コンテンツに対して:
- 公開日が確認できる場合: `prompt.md` の時間スコープ外はフラグを立てる（削除しない）
- 公開日不明の場合: `published_date: "unknown"` としてそのまま保存する

---

## 検索対象サイトの優先順位

| 優先度 | サイト種別 | 例 |
|---|---|---|
| 最高 | 公式サイト・IR・プレスリリース | 企業ドメイン, prnewswire.com |
| 高 | 信頼性の高いニュース | nikkei.com, reuters.com, techcrunch.com |
| 高 | 業界データベース | statista.com, crunchbase.com, cb-insights.com |
| 中 | 専門メディア | industry-specific media |
| 中 | レビューサイト | g2.com, capterra.com, producthunt.com |
| 低 | 個人ブログ・SNS | note, medium, twitter/x |

---

## 出力フォーマット

各サブ質問の結果を `data/{セッション}/raw/Q{N}.json` に保存:

```json
{
  "question_id": "Q01",
  "question_text": "...",
  "executed_at": "2026-03-25T10:00:00Z",
  "queries": [
    {
      "query": "...",
      "language": "ja",
      "round": 1,
      "results": [
        { "url": "...", "title": "...", "snippet": "..." }
      ]
    }
  ],
  "fetched_pages": [
    {
      "url": "...",
      "title": "...",
      "content": "（全文）",
      "published_date": "...",
      "retrieved_at": "...",
      "source_type": "news",
      "fetch_status": "success"
    }
  ],
  "total_sources": 12,
  "rounds_executed": 2
}
```

全サブ質問の検索完了後「検索完了: {N}問 / 合計{M}ソース収集」と表示する。
