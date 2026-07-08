---
name: keiba-dynamic-collector
description: >
  競馬の当日情報を収集するスキル。出走表URLを投げて「当日情報を収集して」と言われたら発動。
  当日馬場傾向（コーナー通過順位）と天気・馬場発表をInfo/dynamic/配下に保存する。
  当日収集、当日情報、馬場傾向、当日馬場、コーナー通過順位という言葉でも積極的にトリガーすること。
---

# keiba-dynamic-collector

## 目的

指定レース（出走表URL）の当日馬場傾向を収集し、`Info/dynamic/` 配下にキャッシュする。
単勝予想の参考として、脚質傾向（逃げ/差し有利）と直線バイアス（内/外有利）を判定する。

---

## 入力

| トリガー例 | 動作 |
|---|---|
| 「当日情報を収集して {URL}」 | URLからレース情報を特定し収集 |
| 「{競馬場名}{R番号}Rの当日傾向を調べて {URL}」 | 同上 |

**URL形式：** `https://race.netkeiba.com/race/shutuba.html?race_id=XXXXXXXXXXXX`

---

## 情報収集の基本方針（Chrome in Claude 優先）

**netkeiba のページは JavaScript 描画が多い。情報収集は必ず以下の優先順位で行うこと。**

```
① Chrome in Claude（mcp__claude-in-chrome__navigate 等）でブラウザ経由取得（推奨・優先）
      ↓ うまくいかない場合
② WebFetch または web 検索（過去1年フィルター）で補完を試みる
      ↓ それでも取得できない場合
③ 該当箇所に「取得不可」と明記してスキップし、次の処理へ進む（ループ禁止）
```

---

## ステップ 0：実行開始アナウンス（必須）

スキル開始時に必ず以下を出力してからデータ収集を開始する：

```
{競馬場名} {N}R 向け当日傾向を収集します。
参照レース：{A}R, {B}R, {C}R（{芝/ダート}のみ）
```

---

## 実行ステップ一覧

各ステップの詳細は対応する reference ファイルを参照すること。

| ステップ | 内容 | 参照ファイル |
|---|---|---|
| 1 | 対象R・参照Rの特定（race_id パース） | [reference/step1_target.md](reference/step1_target.md) |
| 2 | コーナー通過順位の収集（netkeiba） | [reference/step2_corner.md](reference/step2_corner.md) |
| 3 | 馬場傾向の分析・判断 | [reference/step3_analysis.md](reference/step3_analysis.md) |
| 4 | 天気・馬場発表の収集（tenki.jp） | [reference/step4_weather.md](reference/step4_weather.md) |

---

## MD ファイルの書式

保存する MD ファイルの書式は [assets/md_format.md](assets/md_format.md) を参照すること。

---

## 注意事項

- 同馬場（芝/ダート）のみを参照対象とする（異なる馬場は除外）
- 参照Rが同日に存在しない場合（1〜3R）の例外処理はステップ 1 参照
- `Info/dynamic/` 配下のフォルダが存在しない場合は適宜作成してよい
- 同一ファイルが既に存在する場合は上書きする
