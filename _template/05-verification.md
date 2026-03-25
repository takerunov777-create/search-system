# 05-verification — 交差検証・コンセンサス判定・反証チェック

## 役割
収集したデータの信頼性を評価し、各主張・数値を `confirmed` / `unverified` / `conflicted` に分類する。

## 参照パターン
gpt-researcher の Source Tracking & Filtering

---

## 前提原則

- **1 ソースの断言は認めない** — 重要な主張は必ず複数ソースで確認する
- **反証を能動的に探す** — 「〜である」が確認できたら「〜でない可能性」を必ず検索する
- **矛盾を消去しない** — 相反するデータは両論を残す
- **不確実性を正直に示す** — 「わからない」は `unverified` と記録する

---

## 実行手順

### Step 1: 入力を読み込む

`data/{セッション}/raw/all_entries.json` を読み込む。

### Step 2: 主要クレームを抽出する

全エントリから、レポートに記載する可能性のある **主張・数値・評価** を抽出する。

抽出対象:
- 数値（市場規模・シェア・調達額・売上・ユーザー数 等）
- 事実（製品の機能・価格・リリース日・提携先 等）
- 評価（強み・弱み・顧客評価 等）
- 予測・見通し（アナリスト予測・企業ガイダンス 等）

各クレームに以下を付与する:
```json
{
  "claim_id": "C001",
  "text": "A社の2024年売上高は100億円",
  "source_entry_ids": ["E003"],
  "claim_type": "数値",
  "question_ids": ["Q05"]
}
```

### Step 3: 交差検証を実行する

各クレームに対して:

1. 同じ内容を報告しているエントリが他にあるか確認する
2. 相反する数値・主張が存在するか確認する
3. 必要に応じて **追加 WebSearch** を実行して裏付けを取る（最大 3 クエリ）

**信頼度の判定:**

| 状態 | 条件 | ラベル |
|---|---|---|
| `confirmed` | 2 ソース以上が一致、矛盾なし | ✅ 確認済み |
| `unverified` | 1 ソースのみ、または裏取り不能 | ⚠️ 未確認 |
| `conflicted` | ソース間で矛盾・数値が大きく異なる | ❌ 矛盾あり |

**確認済み率の計算:**
```
confirmed_rate = confirmed 件数 / 全クレーム件数
```

### Step 4: コンセンサス判定

| confirmed_rate | 信頼レベル |
|---|---|
| 70% 以上 | 高信頼（High confidence） |
| 40〜70% | 中信頼（Medium confidence）— レポートに注記 |
| 40% 未満 | 低信頼（Low confidence）— 仮説として扱う |

### Step 5: 反証チェック

主要な `confirmed` クレームに対して、以下の反証クエリを実行する:

```
クレーム: "A社は市場シェア40%"
反証クエリ:
  - "A社 市場シェア 誇張 批判"
  - "A社 market share criticism inaccurate"
  - "{competitor} A社比較 シェア 異なる"
```

反証が見つかった場合:
- 元のクレームを `conflicted` に変更する
- 反証内容を `counter_evidence` フィールドに記録する

### Step 6: conflicted クレームの対照表を作成する

```json
// data/{セッション}/verified/conflicts.json
{
  "conflicts": [
    {
      "claim_id": "C012",
      "text": "A社の市場シェア",
      "versions": [
        { "value": "40%", "source": "E003", "source_url": "...", "date": "2025-06-01" },
        { "value": "28%", "source": "E017", "source_url": "...", "date": "2025-11-15" }
      ],
      "note": "調査機関の定義・対象市場が異なる可能性"
    }
  ]
}
```

---

## 出力

```json
// data/{セッション}/verified/verified_facts.json
{
  "session": "...",
  "verified_at": "...",
  "summary": {
    "total_claims": 87,
    "confirmed": 61,
    "unverified": 19,
    "conflicted": 7,
    "confirmed_rate": 0.70,
    "confidence_level": "High"
  },
  "claims": [
    {
      "claim_id": "C001",
      "text": "...",
      "status": "confirmed",
      "source_count": 3,
      "source_entry_ids": ["E003", "E011", "E024"],
      "counter_evidence": null
    }
  ]
}
```

完了後「検証完了: {N}件のクレーム / 確認済み {M}件（{R}%）/ 矛盾 {K}件」と表示する。
