# 04-data-collection — データ収集・記録ルール

## 役割
03-search-strategy.md で取得したコンテンツを、検証・分析で使える形式に構造化して記録する。
**省略・要約・圧縮禁止。全文を保存する。**

---

## 絶対ルール（違反禁止）

1. **全文保存** — `content` フィールドは取得した本文をそのまま保存する。要約・省略・「...」での省略は不可。
2. **ソース必須** — URL・タイトル・取得日時のないデータは記録しない。
3. **重複排除** — 同一 URL は 1 件のみ記録する（初出を残す）。
4. **エラー記録** — アクセス不能な URL は削除せず `fetch_status: "error"` で記録する。
5. **生データ保持** — 判断・フィルタリング・評価はこのフェーズでは行わない（Phase 5 の役割）。

---

## 実行手順

### Step 1: raw データを読み込む

`data/{セッション}/raw/` 以下の全 `Q{N}.json` を読み込む。

### Step 2: 各エントリを標準フォーマットに変換する

各 fetched_page に対して以下のフィールドを確定する:

| フィールド | 取得方法 |
|---|---|
| `url` | WebFetch の取得 URL そのまま |
| `title` | HTML の `<title>` タグ / WebSearch 結果のタイトル |
| `published_date` | `<meta property="article:published_time">` / 本文中の日付 / `"unknown"` |
| `retrieved_date` | 取得実行時の ISO 8601 日時 |
| `source_type` | URL のドメインから判定（下記ルール参照） |
| `language` | 本文の言語を自動判定（`ja` / `en` / その他） |
| `content` | 本文全文（HTML タグを除去したプレーンテキスト） |
| `question_ids` | この情報が関連するサブ質問 ID のリスト |

**source_type 判定ルール:**
- 企業公式ドメイン → `official`
- nikkei.com / reuters.com / techcrunch.com 等 → `news`
- statista.com / crunchbase.com 等 → `database`
- g2.com / capterra.com 等 → `review`
- arxiv.org / researchgate.net 等 → `academic`
- その他 → `other`

### Step 3: 重複を排除する

同一 URL が複数のサブ質問から収集されている場合:
- 最初に出現したものを残す
- `question_ids` に両方のサブ質問 ID を追加する

### Step 4: 統合データを保存する

```json
// data/{セッション}/raw/all_entries.json
{
  "session": "...",
  "collected_at": "...",
  "total_entries": 87,
  "entries": [
    {
      "entry_id": "E001",
      "url": "https://...",
      "title": "...",
      "published_date": "2026-01-15",
      "retrieved_date": "2026-03-25T10:01:23Z",
      "source_type": "news",
      "language": "en",
      "question_ids": ["Q01", "Q03"],
      "content": "（全文）",
      "fetch_status": "success"
    }
  ],
  "error_entries": [
    {
      "entry_id": "E_ERR001",
      "url": "https://...",
      "fetch_status": "error",
      "error_message": "403 Forbidden",
      "question_ids": ["Q02"]
    }
  ]
}
```

### Step 5: 収集統計を出力する

```
収集完了:
  総エントリ数:    {N} 件
  成功:            {M} 件
  エラー:          {K} 件
  言語分布:        日本語 {J} 件 / 英語 {E} 件 / その他 {O} 件
  ソースタイプ分布: official {N} / news {N} / database {N} / review {N} / other {N}
  期間分布:        直近1年 {N} / 1〜3年 {N} / 3年以上 {N} / 不明 {N}
```
