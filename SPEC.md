# HorseRacingPrompts — プロジェクト仕様書

**最終更新：** 2026年07月07日  
**目的：** 単勝馬券の回収率をプラスにする自己改善型AIシステムの構築

---

## 1. プロジェクト概要

生成AI（Claude）を使って競馬の単勝馬券予想を行い、**予想 → 結果検証 → 自己フィードバック → 判断軸の改善** というサイクルを回すことで、長期的に回収率をプラスにすることを目指すリポジトリ。

### 基本方針

```
回収率プラス = 的中率 × オッズ > 1.0
```

単純に当てにいくのではなく、**「過小評価されている馬（オッズ妙味がある馬）」を見つける** ことを最優先とする。

---

## 2. エージェントの起動方法

**重要：** `keiba-tansho-picker` エージェントは、必ず以下の形式で明示的に指定して起動する。  
指定しないとエージェントが起動しない。

```
keiba-tansho-pickerエージェントを使って、{指示}
```

### 具体例

| やりたいこと | 入力例 |
|------------|--------|
| 前日情報を収集する | `keiba-tansho-pickerエージェントを使って、前日情報を収集して` |
| 特定レース向けに収集 | `keiba-tansho-pickerエージェントを使って、直近の安田記念に向けて収集して` |
| 予想する | `keiba-tansho-pickerエージェントを使って、予想して https://race.netkeiba.com/race/shutuba.html?race_id=XXXXXX` |
| 当日情報を収集 | `keiba-tansho-pickerエージェントを使って、当日収集して` |
| 結果を記録・振り返る | `keiba-tansho-pickerエージェントを使って、結果を記録して` |
| サマリーを生成する | `keiba-tansho-pickerエージェントを使って、サマリーを作って` |

---

## 3. ファイル構成（現状）

```
HorseRacingPrompts/
│
├── .claude/
│   ├── agents/
│   │   └── keiba-tansho-picker.md          # Claude Agentの定義
│   ├── skills/
│   │   ├── keiba-static-collector/
│   │   │   ├── SKILL.md                    # 前日情報収集スキルの定義
│   │   │   ├── assets/                     # MDフォーマット等
│   │   │   └── reference/                  # 各ステップの詳細仕様
│   │   ├── keiba-dynamic-collector/
│   │   │   ├── SKILL.md                    # 当日馬場傾向収集スキルの定義
│   │   │   ├── assets/                     # MDフォーマット等
│   │   │   └── reference/                  # 各ステップの詳細仕様
│   │   ├── keiba-result-recorder/
│   │   │   ├── SKILL.md                    # 結果記録・振り返りスキルの定義
│   │   │   ├── assets/csv_format.md        # 予想レコードDBの列定義（真実源）
│   │   │   └── reference/analysis_guide.md # 現状分析・ネクストアクションの観点
│   │   └── keiba-summary-reporter/
│   │       ├── SKILL.md                    # 構築者向けサマリー生成スキルの定義
│   │       └── assets/summary_format.md    # summary.md のフォーマット
│   └── agent-memory/
│       └── keiba-tansho-picker/            # エージェントの記憶（蓄積型）
│
├── records/                                # ← 結果を貯めるDB（git管理対象）
│   ├── predictions.csv                     # 1予想=1レコード（予想/結果/参照情報/判断理由/分析/ネクストアクション）
│   └── summary.md                          # 構築者レビュー用サマリー（依頼時に生成）
│
├── Info/                                   # ← gitignore 対象（ローカルのみ）
│   ├── static/                             # 前日収集キャッシュ
│   │   ├── 馬情報/{馬ID}.md
│   │   ├── 騎手/{騎手ID}.md
│   │   ├── 調教師/{調教師ID}.md
│   │   ├── 競馬場/{中央or地方}/{競馬場名}.md
│   │   └── その他/
│   │       ├── コース別脚質傾向/{競馬場名}{芝orダート}{距離}m.md
│   │       ├── 開催カレンダー.md
│   │       └── {レース名}/
│   └── dynamic/                            # 当日収集キャッシュ
│       └── {YYYY-MM-DD}/{競馬場名}/馬場傾向_{N}R向け.md
│
├── README.md
├── SPEC.md                                 # 本ファイル
└── LICENSE
```

---

## 4. 各コンポーネントの役割

### 4-1. `keiba-tansho-picker` エージェント（`.claude/agents/`）

| 項目 | 内容 |
|------|------|
| **役割** | ユーザーの指示を受け取り、5つのモードで動作するエージェント |
| **モデル** | Claude Opus（高精度分析のため） |
| **起動方法** | `keiba-tansho-pickerエージェントを使って、{指示}` |
| **記憶** | `.claude/agent-memory/keiba-tansho-picker/` に経験を蓄積 |

#### 動作モード

| モード | トリガー例 | 動作 |
|--------|-----------|------|
| **情報収集モード** | 「前日情報を収集して」「直近の〇〇記念に向けて収集」 | `keiba-static-collector` スキルを起動し `Info/static/` にキャッシュ保存 |
| **予想モード** | 出走表URL + 「予想して」 | `Info/static/`・`Info/dynamic/` のキャッシュを参照し単勝3頭を推奨。予想レコードを `records/predictions.csv` に追記（status=結果待ち） |
| **当日収集モード** | 「当日収集して」「当日情報を収集して {URL}」 | `keiba-dynamic-collector` スキルを起動し `Info/dynamic/` に保存 |
| **結果記録モード** | 「結果を記録して」「答え合わせして」 | `keiba-result-recorder` スキルを起動し、結果待ちレコードに結果・現状分析・ネクストアクションを記入 |
| **サマリーモード** | 「サマリーを作って」「成績をまとめて」 | `keiba-summary-reporter` スキルを起動し `records/summary.md` を生成（依頼時のみ） |

#### 情報参照先（予想モード）

| 情報 | 参照先 |
|------|--------|
| 馬の距離適性・前走展開・体重ベスト値・脚質 | `Info/static/馬情報/{馬ID}.md` |
| 騎手の当コース成績 | `Info/static/騎手/{騎手ID}.md` |
| 調教師の当該条件成績・仕上げ傾向 | `Info/static/調教師/{調教師ID}.md` |
| 競馬場のコース状態・内外有利 | `Info/static/競馬場/{中央or地方}/{競馬場名}.md` |
| コース別脚質傾向 | `Info/static/その他/コース別脚質傾向/{競馬場名}{芝orダート}{距離}m.md` |
| 特定重賞の過去傾向 | `Info/static/その他/{レース名}/` |
| 当日の脚質傾向・バイアス・馬場発表 | `Info/dynamic/{YYYY-MM-DD}/{競馬場名}/馬場傾向_{N}R向け.md`（ある場合のみ） |

### 4-2. `keiba-static-collector` スキル（`.claude/skills/`）

| 項目 | 内容 |
|------|------|
| **役割** | 前日情報を事前収集・キャッシュするスキル（スケジューラ自動実行想定） |
| **収集データ** | 馬情報（距離適性・前走展開・体重ベスト値）、騎手情報、調教師情報、競馬場情報 |
| **保存先** | `Info/static/` 配下（gitignore 対象のローカルキャッシュ） |
| **使い方** | 情報収集モードで `keiba-tansho-picker` から自動起動 |
| **取得方法** | Chrome in Claude 優先 → WebFetch → 取得不可時はスキップ |

### 4-3. `keiba-dynamic-collector` スキル（`.claude/skills/`）

| 項目 | 内容 |
|------|------|
| **役割** | 当日の馬場傾向をリアルタイム収集するスキル |
| **収集データ** | 直近3R分のコーナー通過順位（脚質・バイアス判定）、天気・馬場状態 |
| **保存先** | `Info/dynamic/{日付}/{競馬場}/馬場傾向_{N}R向け.md` |
| **使い方** | 当日収集モードで `keiba-tansho-picker` から自動起動 |
| **取得方法** | Chrome in Claude 優先 → WebFetch → 取得不可時はスキップ |

### 4-4. `Info/static/` キャッシュ

| 項目 | 内容 |
|------|------|
| **役割** | 前日収集した情報を保存するローカルキャッシュ |
| **注意** | `.gitignore` 対象。リポジトリには含まれない |
| **運用** | 予想前に情報収集モードを実行してキャッシュを更新すること |

### 4-5. `Info/dynamic/` キャッシュ

| 項目 | 内容 |
|------|------|
| **役割** | 当日収集した馬場傾向を保存するローカルキャッシュ |
| **注意** | `.gitignore` 対象。リポジトリには含まれない |
| **運用** | 当日収集モードを実行することで自動作成される |

### 4-6. `keiba-result-recorder` スキル（`.claude/skills/`）

| 項目 | 内容 |
|------|------|
| **役割** | レース結果を取得し、予想レコードに結果・現状分析・ネクストアクションを記入するスキル |
| **記入内容** | 着順・的中区分・回収率／現状分析（的中精度・見逃し分析・判断根拠検証・オッズ妙味・見逃しパターン）／ネクストアクション（追加観点の提案・取得方法・構成修正箇所） |
| **保存先** | `records/predictions.csv`（既存行を更新し `status=記録済み` へ） |
| **使い方** | 結果記録モードで `keiba-tansho-picker` から自動起動 |
| **取得方法** | Chrome in Claude 優先 → WebFetch → 取得不可時はユーザーに手入力を依頼 |

### 4-7. `keiba-summary-reporter` スキル（`.claude/skills/`）

| 項目 | 内容 |
|------|------|
| **役割** | 蓄積レコードを横断集計し、構築者レビュー用サマリーを生成するスキル |
| **集計内容** | 成績ダッシュボード（的中率・回収率）／予想・結果一覧／参照情報の傾向／判断基準の有効性／現状分析まとめ／ネクストアクション優先度リスト |
| **保存先** | `records/summary.md`（生成のたびに上書き） |
| **使い方** | サマリーモードで `keiba-tansho-picker` から自動起動。**依頼時のみ生成**（自動更新なし） |

### 4-8. `records/` 予想レコードDB

| 項目 | 内容 |
|------|------|
| **役割** | 予想と結果を貯めるDB。**1予想=1レコード（CSVの1行）** |
| **レコード内容** | ①予想（◎○▲）②結果（着順・的中・回収率）③参照情報（リンク付き）④判断理由（判断基準）＋現状分析＋ネクストアクション＋status |
| **列定義** | `.claude/skills/keiba-result-recorder/assets/csv_format.md` が真実源 |
| **注意** | git管理対象（`Info/` と異なりリポジトリに含める）。学習資産として蓄積する |

---

## 5. 現状のデータフロー

```
【前日】
keiba-tansho-pickerエージェントを使って、前日情報を収集して
  ↓
keiba-static-collector スキルが起動
  ↓ Chrome in Claude でnetkeiba を巡回
出走表 / 馬情報 / 騎手成績 / 競馬場状態 を収集
  ↓
Info/static/ 配下にキャッシュ保存

【当日・馬場傾向収集】
keiba-tansho-pickerエージェントを使って、当日情報を収集して {URL}
  ↓
keiba-dynamic-collector スキルが起動
  ↓
直近3R分のコーナー通過順位を収集（netkeiba）
脚質・バイアス判定 + tenki.jp から天気取得
  ↓
Info/dynamic/{日付}/{競馬場}/馬場傾向_{N}R向け.md に保存

【当日・予想時】
keiba-tansho-pickerエージェントを使って、予想して {URL}
  ↓
URL から race_id を抽出し出走馬を特定
  ↓
Info/static/ と Info/dynamic/（当日傾向・ある場合）のキャッシュを参照して評価軸でスコアリング
  - 距離・間隔適性（高）
  - 前走ペース・展開の恩恵（高）
  - コース状態との適合（高）
  - 当日馬場傾向との適合：脚質・バイアス（高・Info/dynamic/ がある場合のみ）
  - 騎手の当コース成績（中）
  - 調教師の当該条件成績（中）
  - 馬体重のベスト値との比較（中）
  - コース別脚質傾向との一致（中）
  ↓
単勝推奨3頭（◎本命・○対抗・▲単穴）と全頭評価サマリーをチャット出力
  ↓
records/predictions.csv に予想レコードを追記（予想・参照情報・判断理由、status=結果待ち）
※当日傾向キャッシュがない場合は「当日収集モードの実行を推奨」と明記される

【レース後・結果記録】
keiba-tansho-pickerエージェントを使って、結果を記録して
  ↓
keiba-result-recorder スキルが起動
  ↓
netkeiba 結果ページから着順・オッズを取得
  ↓
結果待ちレコードに 結果 / 現状分析（5観点） / ネクストアクション（観点提案・取得方法・構成修正案）を記入
  → status=記録済み

【任意・構築者レビュー】
keiba-tansho-pickerエージェントを使って、サマリーを作って
  ↓
keiba-summary-reporter スキルが起動
  ↓
records/predictions.csv の記録済みレコードを横断集計
  ↓
records/summary.md を生成（的中率・回収率・判断基準の有効性・ネクストアクション優先度リスト）
```

**現状の課題：** オッズのリアルタイム取得が未実装。当日のオッズは手動で確認が必要。

---

## 6. 次回改修内容（TODO）

### 🔵 優先度 HIGH

---

#### ~~TODO-01：フォルダ構造の分離（情報収集 vs AI予想）~~ ✅ 完了（設計変更）

予想・結果は `records/predictions.csv`（1予想=1レコードのCSV DB）に一元化する設計に変更して実装済み。
情報キャッシュ（`Info/`・gitignore対象）とAIの予想・フィードバック（`records/`・git管理対象）が分離された。

---

#### ~~TODO-02：自己反省ループの実装（predictions → feedback）~~ ✅ 完了

`keiba-result-recorder` スキルとして実装済み。
フィードバックの5観点（的中精度・見逃し分析・判断根拠の検証・オッズ妙味の検証・見逃しパターン）は
`.claude/skills/keiba-result-recorder/reference/analysis_guide.md` に定義され、`analysis` 列に記録される。
加えてネクストアクション（追加観点の提案・取得方法・構成修正箇所）を `next_action` 列に記録する。

---

#### TODO-03：判断軸ドキュメントの自動生成・更新

**背景：** フィードバックを積み重ねることで「これなら当たる」という判断軸を自動でアップデートしたい。

**現状：** `keiba-summary-reporter` が「有効だった判断軸」「無効だった判断軸」の集計まで実装済み（`records/summary.md` セクション4）。

**残作業：**

1. 集計結果から `judgment_criteria.md`（判断軸マスタードキュメント）を自動更新する
2. 予想モードの評価軸・重みを `judgment_criteria.md` から読み込むように変更する

---

### 🟡 優先度 MEDIUM

---

#### ~~TODO-04：当日収集モードの実装（skill②）~~ ✅ 完了

`keiba-dynamic-collector` スキルとして実装済み。
- 直近3R分のコーナー通過順位による脚質・バイアス判定
- tenki.jp / JRA公式からの天気・馬場発表取得
- `Info/dynamic/{日付}/{競馬場}/馬場傾向_{N}R向け.md` に保存

---

#### ~~TODO-05：予想結果の永続化~~ ✅ 完了（設計変更）

予想モードの最終ステップとして `records/predictions.csv` への1行追記を実装済み。
レース後に結果を記入できる空フィールド（result_*・analysis・next_action、`status=結果待ち`）を含む。

---

## 7. 将来のシステム全体像（v2 ターゲット）

```
【前日】
keiba-tansho-pickerエージェントを使って、前日情報を収集して
  ↓
keiba-static-collector スキル
  └─ Info/static/{馬・騎手・競馬場}.md に情報保存

【当日・予想】
keiba-tansho-pickerエージェントを使って、予想して {URL}
  ↓
① Info/static/・Info/dynamic/ キャッシュ参照
② judgment_criteria.md（判断軸マスタ）で重み付け（⭐️ 未実装・TODO-03）
③ 期待値計算・推奨選定
④ records/predictions.csv に予想レコードを追記（✅ 実装済み）
⑤ ユーザーへ推奨をチャット出力

【レース後】
keiba-tansho-pickerエージェントを使って、結果を記録して
  ↓
① 予想レコードと結果を照合（✅ keiba-result-recorder 実装済み）
② 現状分析・ネクストアクションをレコードに記入（✅ 実装済み）
③ judgment_criteria.md を自動更新（⭐️ 未実装・TODO-03。現状は keiba-summary-reporter による集計まで）
```

---

## 8. 免責事項

- 本リポジトリで提供するプロンプト・スキル・エージェントは、回収率向上を目的としていますが、的中を保証するものではありません。
- 競馬はギャンブルです。余裕資金の範囲内でお楽しみください。
