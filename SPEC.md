# 競合調査システム 仕様書

**バージョン:** 1.0
**作成日:** 2026-03-25
**対象:** Claude Code による自律的競合調査・市場調査・レポート生成システム

---

## 1. システム概要

### 目的
特定の企業・製品・市場に関する競合調査を、Claude Code が自律的に実行し、引用付きの HTML レポートとして出力するシステム。

### 設計原則
- **全文保存・省略禁止** — 収集データは圧縮・省略せず記録する
- **ソース完全追跡** — 全主張にソース URL・日付・信頼度を紐づける
- **多角的検証** — 単一ソースの断言を排除し、交差検証を義務化する
- **並列実行優先** — 独立したフェーズは並列で実行し、速度を最大化する
- **予測の検証可能性** — 分析結果に予測を記録し、的中率を蓄積する

### 入力
- `prompt.md` — 調査テーマ・対象・目的・制約条件を記載したドメイン設定ファイル
- （新規テーマの場合）`00a-generate-prompt.md` が `prompt.md` を自動生成

### 出力
- `report/YYYY-MM-DD_{テーマ名}.html` — 単一HTMLファイル（白ベース・ダークモード自動対応）
- `data/YYYY-MM-DD_{テーマ名}/` — 収集した生データ・検証済みデータ
- `track-record/predictions.json` — 予測トラックレコード

---

## 2. ディレクトリ構成

```
D:\ビジネス\リサーチ\
│
├── SPEC.md                        ← 本仕様書
├── prompt.md                      ← 調査テーマ設定（実行時に差し替える）
│
├── _template/                     ← フェーズ定義ファイル群
│   ├── 00-orchestrator.md
│   ├── 00a-generate-prompt.md
│   ├── 01-clarify.md
│   ├── 02-persona.md
│   ├── 03-search-strategy.md
│   ├── 04-data-collection.md
│   ├── 05-verification.md
│   ├── 06-analysis.md
│   ├── 07-report.md
│   └── 08-track-record.md
│
├── .claude/
│   └── commands/                  ← Claude Code スキル
│       ├── deep-research.md
│       ├── market-research.md
│       ├── article-writing.md
│       ├── content-engine.md
│       ├── dispatching-parallel-agents.md
│       └── theme-factory.md
│
├── data/                          ← 調査データ（実行時に生成）
│   └── YYYY-MM-DD_{テーマ名}/
│       ├── raw/                   ← Phase 4 収集生データ
│       ├── verified/              ← Phase 5 検証済みデータ
│       └── analysis/              ← Phase 6 分析結果
│
├── report/                        ← 出力HTMLレポート（実行時に生成）
│   └── YYYY-MM-DD_{テーマ名}.html
│
├── track-record/                  ← Phase 8 予測記録（実行時に生成）
│   └── predictions.json
│
└── repos/                         ← 参照リポジトリ
    ├── storm/
    ├── gpt-researcher/
    ├── open_deep_research/
    └── everything-claude-code/
```

---

## 3. フェーズ仕様

### Phase 0 — オーケストレーター（`00-orchestrator.md`）

**役割:** 全フェーズの制御・並列実行管理・エラーハンドリング

**参照リソース:** STORM（パイプライン設計）、dispatching-parallel-agents

**処理:**
1. `prompt.md` を読み込む（存在しない場合は Phase 0.5 を先行実行）
2. 全フェーズのシーケンスを定義し、依存関係を確認する
3. 並列実行可能なフェーズを特定し、`/dispatching-parallel-agents` で並列投入する
4. 各フェーズの完了を確認し、次フェーズに入力を渡す
5. フェーズ失敗時のリトライ・スキップルールを適用する

**並列実行ルール:**
- Phase 3（検索）の各クエリは独立して並列実行する
- Phase 3 と Phase 4 は同一ソースに対してパイプライン並列化する
- Phase 5〜6 は Phase 4 完了後に順次実行する
- Phase 7 の各セクションは内容確定後に並列生成可能

**出力:** 実行ログ（各フェーズの開始・終了・ステータス）

---

### Phase 0.5 — prompt.md 自動生成（`00a-generate-prompt.md`）

**役割:** 新規テーマに対して `prompt.md` を対話的に生成する

**処理:**
1. ユーザーに調査テーマを質問する
2. 以下の項目を埋める:
   - 調査対象（企業名・製品名・市場）
   - 調査目的（何を知りたいか）
   - 地理的スコープ（国・地域）
   - 時間的スコープ（過去 N 年、現在）
   - 想定読者（経営層・事業開発・投資家）
   - 優先言語（日本語・英語・その他）
   - 除外条件（調査しない領域）
3. `prompt.md` として保存する

**出力:** `prompt.md`

---

### Phase 1 — 要件明確化（`01-clarify.md`）

**役割:** 調査目的の具体化とスコープ定義

**参照リソース:** gpt-researcher（サブ質問分解パターン）

**処理:**
1. `prompt.md` から調査目的を抽出する
2. 目的を **5〜10 個のサブ質問** に分解する
3. 各サブ質問に対して:
   - 重要度スコア（High / Medium / Low）
   - 推定調査時間
   - 必要なデータソース種別
   を付与する
4. スコープ境界を明示する（調査する／しない）
5. 成功基準を定義する（何がわかれば調査完了か）

**出力:**
```json
{
  "objective": "...",
  "sub_questions": [
    { "id": "Q1", "text": "...", "priority": "High", "sources": ["web", "news"] }
  ],
  "scope": { "in": [...], "out": [...] },
  "success_criteria": [...]
}
```

---

### Phase 2 — ペルソナ生成（`02-persona.md`）

**役割:** 多角的な視点から追加質問を生成し、調査の盲点を排除する

**参照リソース:** STORM（Perspective-Guided Question Asking）

**処理:**
1. `prompt.md` と Phase 1 のサブ質問を読み込む
2. 調査テーマに適した **仮想専門家 3〜5 人** を定義する
   - 例: アナリスト・エンジニア・マーケター・投資家・エンドユーザー
3. 各ペルソナが「自分の立場から知りたいこと」として追加質問を生成する（各 3〜5 問）
4. 重複・低価値の質問を除去する
5. Phase 1 のサブ質問リストに統合し、優先度を再評価する

**ペルソナ定義フォーマット:**
```
### {名前}（{役職}）
背景: ...
関心: ...
追加質問:
  - Q: ...（優先度: High）
  - Q: ...（優先度: Medium）
```

**出力:** 統合サブ質問リスト（Phase 1 拡張版）

---

### Phase 3 — 検索戦略（`03-search-strategy.md`）

**役割:** 効率的な情報収集のための検索クエリ設計と実行

**参照リソース:** gpt-researcher（Planner-Executor）、deep-research（WebSearch + WebFetch）

**検索ルール:**
- 各サブ質問に対して **最低 3 クエリ**（日本語・英語・業界用語バリエーション）を生成する
- 直近 1 年以内の情報を優先する（鮮度フィルタ）
- 公式サイト・ニュース・学術・SNS・決算資料を網羅する
- 1 クエリあたり上位 5〜10 件を取得する
- 独立したクエリは並列実行する（dispatching-parallel-agents）

**クエリ拡張ルール（Adaptive Query Expansion）:**
1. 初回クエリの結果から関連キーワードを抽出する
2. 不足情報を補うクエリを追加生成する
3. 最大 3 ラウンドまで繰り返す

**MCPなし構成:**
- `WebSearch` — 一般検索
- `WebFetch` — 特定 URL のコンテンツ取得

**出力:**
```
data/{テーマ}/raw/
  search_results_Q1.json
  search_results_Q2.json
  ...
```

---

### Phase 4 — データ収集（`04-data-collection.md`）

**役割:** 検索結果の構造化記録（省略・圧縮禁止）

**収集ルール:**
- 本文は**全文保存**する（要約・省略は後フェーズで行う）
- 各エントリに必ず以下を付与する:
  - `url` — 出典 URL
  - `title` — ページタイトル
  - `published_date` — 公開日（取得できない場合は `unknown`）
  - `retrieved_date` — 収集日時（ISO 8601）
  - `source_type` — `news` / `official` / `academic` / `sns` / `other`
  - `language` — `ja` / `en` / その他
  - `content` — 本文全文
- 重複 URL は記録しない（初出のみ保存）
- アクセス不能な URL はエラー記録する（スキップしない）

**出力フォーマット:**
```json
{
  "query_id": "Q1",
  "collected_at": "2026-03-25T10:00:00Z",
  "entries": [
    {
      "url": "https://...",
      "title": "...",
      "published_date": "2026-01-15",
      "retrieved_date": "2026-03-25T10:01:23Z",
      "source_type": "news",
      "language": "en",
      "content": "（全文）"
    }
  ]
}
```

---

### Phase 5 — 検証（`05-verification.md`）

**役割:** 収集データの信頼性評価と事実確認

**参照リソース:** gpt-researcher（Source Tracking & Filtering）

**検証プロセス:**

#### 5-1. 交差検証
- 同一の主張・数値が **2 ソース以上** で確認できる場合のみ `confirmed` とする
- 1 ソースのみの場合は `unverified` とマークする
- ソース間で矛盾する場合は `conflicted` として両論を記録する

#### 5-2. コンセンサス判定
- Confirmed 率が 70% 以上 → 高信頼
- Confirmed 率が 40〜70% → 中信頼（注記付き）
- Confirmed 率が 40% 未満 → 低信頼（仮説として扱う）

#### 5-3. 反証チェック
- 各主要主張に対し「反証・反論」を能動的に検索する
- 反証が存在する場合はレポートに必ず記載する

**出力:**
```
data/{テーマ}/verified/
  verified_facts.json   ← confirmed/unverified/conflicted 分類済み
  conflicts.json        ← 矛盾するデータの対照表
```

---

### Phase 6 — 分析（`06-analysis.md`）

**役割:** 検証済みデータの意味解釈・スコアリング・市場分析

**参照リソース:** open_deep_research（推論フェーズ分離）、market-research

**分析項目:**

#### 6-1. 情報圧縮・要約
- Verified Facts から重要度順に上位を抽出する
- 冗長・重複情報を統合する

#### 6-2. 競合スコアリング
各競合に対して以下の軸で 1〜5 点スコアリングする:

| 軸 | 定義 |
|---|---|
| 製品力 | 機能・品質・差別化 |
| 市場シェア | 推定占有率・成長率 |
| 資金力 | 調達額・黒字化状況 |
| ブランド | 認知度・NPS |
| 技術力 | 特許・エンジニア数・R&D |
| 脅威度 | 自社への直接的脅威レベル |

#### 6-3. 市場分析
- TAM / SAM / SOM 推定（出典付き）
- 市場成長率とトレンド
- 参入障壁・スイッチングコスト

#### 6-4. SWOT / ポジショニングマップ
- 対象テーマの SWOT を生成する
- 2 軸ポジショニングマップのデータを生成する（HTMLで可視化）

#### 6-5. 予測生成
- 6〜12 ヶ月後の予測を **3 つ** 生成する
- 各予測に: 根拠・信頼度（%）・検証期日 を付与する
- → Phase 8 トラックレコードに書き込む

**出力:**
```
data/{テーマ}/analysis/
  scores.json
  market.json
  swot.json
  predictions.json
```

---

### Phase 7 — レポート生成（`07-report.md`）

**役割:** 分析結果を単一 HTML ファイルとして出力する

**参照リソース:** ui-ux-pro-max、article-writing、theme-factory

**出力仕様:**

| 項目 | 仕様 |
|---|---|
| フォーマット | 単一 HTML ファイル（外部依存なし・CSS/JS inline） |
| ベースカラー | 白（#FFFFFF）ベース |
| ダークモード | `prefers-color-scheme: dark` で自動切替 |
| レスポンシブ | PC・タブレット・スマートフォン対応 |
| 印刷対応 | `@media print` で印刷最適化 |
| チャート | Chart.js inline で描画 |
| テーマ | theme-factory の 10 テーマから選択（デフォルト: Modern Minimalist） |

**レポート構成:**

```
1. エグゼクティブサマリー（1ページ相当）
   - 調査目的・期間・主要発見 3〜5 点

2. 市場概況
   - TAM/SAM/SOM（棒グラフ）
   - 市場成長率トレンド（折れ線グラフ）

3. 競合マップ
   - ポジショニングマップ（散布図）
   - 競合スコア比較（レーダーチャート）

4. 個社分析
   - 各競合の詳細（スコア・強み・弱み・最新動向）

5. SWOT 分析

6. 予測・インプリケーション
   - 6〜12 ヶ月予測 3 点（根拠付き）
   - 推奨アクション

7. 情報源一覧
   - 全引用ソース（URL・日付・信頼度）

8. 付録: 検証ステータス
   - Confirmed / Unverified / Conflicted の分類表
```

**品質ルール:**
- 数値は必ずソース付きで記載する
- `unverified` データには ⚠️ マークを付ける
- `conflicted` データは両論を併記する
- AI 生成であることを末尾に明示する

**出力:** `report/YYYY-MM-DD_{テーマ名}.html`

---

### Phase 8 — トラックレコード（`08-track-record.md`）

**役割:** 過去の予測の的中率を蓄積・検証する

**処理:**

#### 8-1. 予測の記録（Phase 6 から自動引き継ぎ）
```json
{
  "id": "pred_001",
  "theme": "...",
  "created_at": "2026-03-25",
  "prediction": "...",
  "confidence": 70,
  "verification_date": "2026-09-25",
  "status": "pending",  // pending / correct / incorrect / partial
  "actual_outcome": null
}
```

#### 8-2. 予測の検証（`verification_date` 到達時）
1. 対象予測を抽出する
2. 最新情報で結果を調査する
3. `status` を更新する（correct / incorrect / partial）
4. `actual_outcome` に実際の結果を記録する

#### 8-3. 的中率レポート
- テーマ別・信頼度別の的中率を集計する
- 系統的なバイアスを検出する（過信傾向・楽観バイアスなど）
- 次回調査の信頼度設定に反映する

**出力:** `track-record/predictions.json`（累積・追記）

---

## 4. スキル・ライブラリ対応表

### 実行方針（Option A）
> Pythonライブラリは**設計パターンの参照のみ**。直接呼び出しは行わない。
> 全検索・LLM処理は Claude Code の組み込みツール（WebSearch / WebFetch）で実行する。
> APIキー不要。Python 3.14 互換性問題の影響を受けない。

| フェーズ | 実行主体 | 参照パターン（ライブラリ） | 用途 |
|---|---|---|---|
| 0 | Claude Code | dispatching-parallel-agents スキル | 並列エージェント制御 |
| 0 | Claude Code | STORM（knowledge-storm）— **参照のみ** | 4段階パイプライン設計 |
| 1 | Claude Code | gpt-researcher — **参照のみ** | サブ質問分解パターン |
| 2 | Claude Code | STORM Perspective-Guided — **参照のみ** | ペルソナ駆動質問生成 |
| 3 | Claude Code（WebSearch/WebFetch） | deep-research スキル | 多言語検索・クエリ拡張 |
| 3 | Claude Code（WebSearch/WebFetch） | gpt-researcher — **参照のみ** | Planner-Executor 並列設計 |
| 4 | Claude Code | — | 独自収集ルール |
| 5 | Claude Code（WebSearch） | gpt-researcher — **参照のみ** | 交差検証・信頼度判定 |
| 6 | Claude Code | open_deep_research — **参照のみ** | 推論フェーズ分離・分析設計 |
| 6 | Claude Code | market-research スキル | 市場分析・スコアリング |
| 7 | Claude Code | ui-ux-pro-max スキル | HTML/CSS コンポーネント |
| 7 | Claude Code | article-writing スキル | 論拠付き文章生成 |
| 7 | Claude Code | theme-factory スキル | テーマ適用 |
| 8 | Claude Code | — | 独自トラックレコード |

---

## 5. 実行方法

### 新規テーマの調査開始
```
/00a-generate-prompt   ← prompt.md を自動生成
/00-orchestrator       ← 全フェーズを自動実行
```

### 既存 prompt.md がある場合
```
/00-orchestrator       ← 直接実行
```

### 特定フェーズのみ再実行
```
/06-analysis           ← Phase 6 のみ再実行
/07-report             ← レポートのみ再生成
```

### 予測の検証
```
/08-track-record       ← 期日到来の予測を検証
```

---

## 6. 制約・注意事項

- **APIキー不要:** WebSearch/WebFetch（Claude Code 組み込み）のみ使用。Exa・Firecrawl・外部LLM API 不使用。
- **Pythonライブラリは参照のみ:** knowledge-storm / gpt-researcher / open_deep_research はコードを直接実行しない。設計パターンの参照として使用する（Python 3.14 互換性問題・LLM APIキー依存を回避）。
- **言語:** 調査は日本語・英語を標準とする。`prompt.md` で追加言語を指定可能。
- **データ保持:** `data/` ディレクトリは上書きしない。日付付きで新規作成する。
- **レポート上書き禁止:** 同名ファイルが存在する場合は suffix を付ける（`_v2` など）。
- **AI 生成明示:** レポート末尾に生成日時・使用モデル・注意書きを必ず記載する。
