# 07-report — HTML レポート生成

## 役割
分析結果を単一 HTML ファイルとして出力する。外部リソース参照なし・完全自己完結。

## 参照パターン
- ui-ux-pro-max スキル: アクセシビリティ・パフォーマンス・レスポンシブ
- article-writing スキル: 引用・論拠付き文章・フィラー排除
- theme-factory スキル: テーマ適用（カラー・フォント）

---

## 出力仕様（絶対条件）

| 項目 | 仕様 |
|---|---|
| ファイル形式 | 単一 HTML（`.html`）— CSS・JS はすべて inline |
| 外部 CDN | **禁止**（Chart.js も inline で埋め込む） |
| ベースカラー | 白（`#FFFFFF`）ベース |
| ダークモード | `@media (prefers-color-scheme: dark)` で自動切替 |
| レスポンシブ | PC・タブレット・モバイル対応 |
| 印刷 | `@media print` で印刷最適化 |
| 文字コード | UTF-8 |

---

## 実行手順

### Step 1: 入力を読み込む

- `data/{セッション}/analysis/competitor_scores.json`
- `data/{セッション}/analysis/market.json`
- `data/{セッション}/analysis/swot.json`
- `data/{セッション}/analysis/positioning.json`
- `data/{セッション}/verified/verified_facts.json`
- `prompt.md`

### Step 2: テーマを選択する（theme-factory）

以下の 10 テーマから 1 つを選ぶ（デフォルト: **Modern Minimalist**）。
`prompt.md` に `report_theme:` の指定があればそれを使用する。

| テーマ名 | 主色 | フォント（見出し / 本文） | 適した用途 |
|---|---|---|---|
| Modern Minimalist | `#2C3E50` | Helvetica Neue / Inter | 汎用・ビジネス |
| Ocean Depths | `#1B4F72` | Merriweather / Source Sans Pro | 調査・分析レポート |
| Tech Innovation | `#58A6FF` | JetBrains Mono / Inter | テック・IT系 |
| Midnight Galaxy | `#66FCF1` | Orbitron / Rajdhani | スタートアップ・先端技術 |
| Golden Hour | `#E67E22` | Playfair Display / Lato | 投資家向け・高級感 |
| Arctic Frost | `#AED6F1` | Roboto Slab / Open Sans | 医療・金融 |
| Forest Canopy | `#27AE60` | Raleway / Nunito | サステナ・環境 |
| Sunset Boulevard | `#E74C3C` | Oswald / Roboto | マーケティング・消費財 |
| Desert Rose | `#C0392B` | Georgia / Garamond | コンサル・伝統産業 |
| Botanical Garden | `#1ABC9C` | Quicksand / Poppins | ヘルスケア・ライフスタイル |

### Step 3: HTML 構造を構築する

以下のセクション順で HTML を生成する:

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>{テーマ名} 競合調査レポート — {調査日}</title>
  <style>
    /* === CSS変数（テーマ色）=== */
    :root { --primary: ...; --secondary: ...; --accent: ...; --bg: #fff; --text: ...; }
    @media (prefers-color-scheme: dark) { :root { --bg: #1a1a2e; --text: #e0e0e0; ... } }
    /* === レイアウト・タイポグラフィ === */
    /* === コンポーネント（カード・テーブル・バッジ） === */
    /* === チャートコンテナ === */
    /* === 印刷スタイル === */
    @media print { ... }
  </style>
</head>
<body>
  <!-- ナビゲーション（PC: サイドバー / モバイル: トップ） -->
  <!-- 1. エグゼクティブサマリー -->
  <!-- 2. 市場概況（Chart.js: 棒グラフ・折れ線） -->
  <!-- 3. 競合マップ（Chart.js: 散布図・レーダー） -->
  <!-- 4. 個社分析 -->
  <!-- 5. SWOT 分析 -->
  <!-- 6. 予測・推奨アクション -->
  <!-- 7. 情報源一覧 -->
  <!-- 8. 付録: 検証ステータス -->
  <script>
    /* Chart.js ライブラリ（minified inline） */
    /* チャート初期化コード */
  </script>
</body>
</html>
```

### Step 4: 各セクションを執筆する（article-writing ルール）

**文章品質ルール（article-writing スキルより）:**
- 事実を先に書き、理由・背景は後に書く
- 1 文に 1 情報（複文を避ける）
- 「〜と考えられる」「〜の可能性がある」は根拠とセットで書く
- フィラーワード禁止（「非常に」「かなり」「様々な」等）
- 数値は必ず出典付き（`<sup>[1]</sup>` 形式）

**信頼度バッジ（ui-ux-pro-max ルールより）:**
```html
<!-- confirmed データ -->
<span class="badge badge-confirmed">✅ 確認済み</span>
<!-- unverified データ -->
<span class="badge badge-unverified">⚠️ 未確認</span>
<!-- conflicted データ -->
<span class="badge badge-conflicted">❌ 矛盾あり</span>
```

### Step 5: Chart.js グラフを生成する

**市場規模グラフ（棒グラフ）:**
- X軸: 年次 / Y軸: 市場規模（億円）
- TAM / SAM / SOM を色分け

**成長率グラフ（折れ線グラフ）:**
- X軸: 年次 / Y軸: 成長率（%）

**競合スコアレーダーチャート:**
- 各競合を重ね合わせ（最大 5 社）
- 評価軸: 製品力・市場シェア・資金力・ブランド・技術力・脅威度

**ポジショニングマップ（散布図）:**
- 2 軸は analysis/positioning.json の設定に従う
- 各競合を点でプロット、ラベル付き

### Step 6: アクセシビリティチェック（ui-ux-pro-max ルールより）

- グラフには `aria-label` を付ける
- カラーのみに依存しない（バッジにはアイコンも使用）
- コントラスト比 4.5:1 以上を確保する
- キーボードナビゲーション対応

### Step 7: フッターを生成する

```html
<footer>
  <p>本レポートは Claude Code による自律的調査に基づき生成されました。</p>
  <p>生成日時: {ISO 8601} / モデル: Claude Sonnet 4.6</p>
  <p>⚠️ 未確認情報・矛盾情報が含まれる場合があります。重要な意思決定の前に一次情報を確認してください。</p>
</footer>
```

---

## 出力

`report/YYYY-MM-DD_{テーマ名}.html` に保存する。

同名ファイルが存在する場合は `_v2`, `_v3` のサフィックスを付ける。

完了後「レポート生成完了: report/YYYY-MM-DD_{テーマ名}.html ({ファイルサイズ})」と表示する。
