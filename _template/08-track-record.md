# 08-track-record — 予測トラックレコード

## 役割
Phase 6 で生成した予測を累積記録し、検証期日が来た予測の的中率を評価する。
分析の質を継続的に改善するためのフィードバックループ。

---

## 実行モード

このファイルは 2 つのモードで実行される:

1. **記録モード** — Phase 6 完了直後に自動実行。新規予測を `predictions.json` に追記する。
2. **検証モード** — 手動呼び出し時に実行。期日到来の予測を検証する。

---

## 記録モード（Phase 6 直後に実行）

### Step 1: 新規予測を読み込む

`data/{セッション}/analysis/predictions.json` を読み込む。

### Step 2: `track-record/predictions.json` に追記する

ファイルが存在しない場合は新規作成する。
存在する場合は既存データに追記する（上書きしない）。

```json
// track-record/predictions.json
{
  "last_updated": "2026-03-25T12:00:00Z",
  "total_predictions": 3,
  "predictions": [
    {
      "pred_id": "PRED_001",
      "session": "2026-03-25_NotionVSConcept",
      "theme": "Notion vs 競合",
      "created_at": "2026-03-25",
      "prediction": "A社は2026年9月末までに国内シェアを35%超に拡大する",
      "basis_claim_ids": ["C008", "C015"],
      "confidence": 65,
      "verification_date": "2026-09-30",
      "category": "市場シェア",
      "status": "pending",
      "actual_outcome": null,
      "verified_at": null,
      "accuracy_score": null,
      "notes": null
    }
  ]
}
```

完了後「トラックレコード更新: {N}件の予測を記録しました」と表示する。

---

## 検証モード（手動呼び出し）

### Step 1: 期日到来の予測を抽出する

`track-record/predictions.json` を読み込み、以下の条件の予測を抽出する:
- `status: "pending"`
- `verification_date` が本日以前

### Step 2: 各予測を検証する

抽出された予測ごとに:

1. 予測内容と関連する最新情報を **WebSearch** で検索する
2. 結果を評価して以下のステータスを付与する:

| ステータス | 条件 |
|---|---|
| `correct` | 予測が実現した（数値なら ±10% 以内） |
| `partial` | 方向性は正しいが数値・条件が一部外れた |
| `incorrect` | 予測が外れた |
| `unverifiable` | 検証に必要なデータが公開されていない |

3. `actual_outcome` に実際の結果を記録する
4. 検証に使用したソース URL を `verification_sources` に記録する

### Step 3: 的中率を計算する

検証済み予測（`unverifiable` を除く）に対して:

```
correct_rate = (correct + partial * 0.5) / (correct + partial + incorrect)
```

テーマ別・信頼度帯別に集計する:

| 信頼度帯 | 件数 | correct | partial | incorrect | 的中率 |
|---|---|---|---|---|---|
| 60-70% | N | N | N | N | XX% |
| 70-80% | N | N | N | N | XX% |
| 80-90% | N | N | N | N | XX% |

### Step 4: バイアスを検出する

以下のパターンを確認し、発見されたバイアスを記録する:

- **過信バイアス**: 実際の的中率 < 平均信頼度（例: 信頼度70%平均なのに的中率40%）
- **楽観バイアス**: ポジティブ予測の的中率 < ネガティブ予測の的中率
- **カテゴリバイアス**: 特定カテゴリ（市場シェア・財務等）の的中率が低い
- **時間バイアス**: 短期予測 vs 長期予測の的中率差

### Step 5: 改善提言を生成する

検出されたバイアスに基づき、次回調査の信頼度設定への反映を提案する。

例:
```
【バイアス検出】市場シェア予測の的中率が34%（信頼度平均65%）
→ 次回から市場シェアカテゴリの信頼度を -15% 補正することを推奨
```

---

## 出力

`track-record/predictions.json` を更新する（追記・上書き選択）。

検証モード実行後:
```
検証完了:
  対象予測: {N} 件
  correct: {A} 件 / partial: {B} 件 / incorrect: {C} 件 / unverifiable: {D} 件
  的中率: {R}%
  検出バイアス: {バイアス名 or なし}
```
