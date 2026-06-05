---
name: keiba-static-collector
description: >
  競馬の静的情報を事前収集・キャッシュするスキル。スケジューラによる自動実行を想定。
  「前日情報を収集して」「直近の情報を収集」「直近の〇〇記念に向けて収集」と言われたら発動。
  翌日以降で最初に開催されるレース全頭の馬・騎手・調教師・競馬場情報をInfo/static/配下に保存する。
  競馬、前日情報、静的情報収集、馬情報、騎手情報、調教師情報、競馬場情報という言葉でも積極的にトリガーすること。
---

# keiba-static-collector

## 目的

翌日以降で最初の開催日のレース情報を事前収集し、`Info/static/` 配下にキャッシュする。
スケジューラによる自動実行を想定しているため、**当日の開催情報は対象外**。

---

## 入力

| トリガー例 | 動作モード |
|---|---|
| 「前日情報を収集して」「直近の情報を収集」など | 直近モード |
| 「直近の安田記念に向けて収集」など特定レース名あり | 特定レースモード |

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
次回開催の {MM}月{DD}日（{曜日}）の情報を取得します。
対象競馬場：{競馬場A}・{競馬場B}・...
```

---

## 実行ステップ一覧

各ステップの詳細は対応する reference ファイルを参照すること。

| ステップ | 内容 | 参照ファイル |
|---|---|---|
| 1 | 対象日・対象レースの特定 | [reference/step1_target.md](reference/step1_target.md) |
| 2 | 競馬場情報の収集（コース状態） | [reference/step2_course.md](reference/step2_course.md) |
| 3 | 馬情報の収集（⑧⑪⑫） | [reference/step3_horse.md](reference/step3_horse.md) |
| 4 | 騎手情報の収集（⑥） | [reference/step4_jockey.md](reference/step4_jockey.md) |
| 5 | 調教師情報の収集 | [reference/step5_trainer.md](reference/step5_trainer.md) |
| 7 | その他の情報収集 | [reference/step7_other.md](reference/step7_other.md) |
| 8 | 特定レース専用フォルダ作成（特定レースモードのみ） | [reference/step8_special_race.md](reference/step8_special_race.md) |

---

## MD ファイルの書式

保存する MD ファイルの書式は [assets/md_format.md](assets/md_format.md) を参照すること。

---

## 注意事項

- **当日の開催情報は収集対象外**（スケジューラ運用のため。当日に実行した場合は翌開催日を対象とすること）
- 馬場の「継続使用」には必ず注意すること（ステップ 2 参照）
- `Info/static/` 配下のフォルダが存在しない場合は適宜作成してよい
