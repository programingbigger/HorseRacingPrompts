# ステップ 1：対象日・対象レースの特定

## 手順

1. `https://race.netkeiba.com/top/` を Chrome in Claude（`mcp__claude-in-chrome__*`）で開く
2. **今日より後で最初の開催日**を特定する
   - 今日が開催日であっても当日は対象外。翌日以降を参照すること
3. その開催日の全競馬場・全レース（1R〜nR）の race_id リストを取得する

## 特定レースモード（追加手順）

レース名が指定された場合、追加で web 検索を使って race_id を自力特定する：

```
検索クエリ例：site:netkeiba.com 安田記念 2026 出走表
```

## 情報取得の方法

```
① Chrome in Claude（mcp__claude-in-chrome__navigate 等）でページを開く（推奨・優先）
      ↓ うまくいかない場合
② WebFetch で直接取得を試みる
      ↓ それでも取得できない場合
③ 「取得不可」と明記してスキップし、次の処理へ進む
```
