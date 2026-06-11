# agent-memory 調査レポート

**作成日：** 2026年06月10日
**目的：** keiba-tansho-picker の agent-memory に「何を残し、何を削除するか」を判断するための材料整理

---

## 1. agent-memory はいつ発動するのか

Claude Code のサブエージェント（`.claude/agents/*.md`）は、frontmatter に `memory` フィールドを持つと永続メモリが有効になる [1]。動作タイミングは以下の3つ。

**起動時（読み込み）：** エージェント起動のたびに、メモリディレクトリの `MEMORY.md` の先頭200行（または25KB）がエージェントのシステムプロンプトに自動注入される [1]。個別のトピックファイル（本プロジェクトの `insight-*.md` 等）は起動時には読まれず、エージェントが必要と判断したときに Read ツールでオンデマンドに読む。

**作業中（書き込み）：** メモリ有効時は Read / Write / Edit が自動許可され、エージェントは「将来の会話で役立つ」と自分で判断した内容を任意のタイミングで保存する [1]。つまり**保存はルールベースではなくエージェントの裁量**であり、何を保存するかはエージェント定義本文の指示文に強く依存する。本プロジェクトの `keiba-tansho-picker.md` には「Update your agent memory as you discover patterns...」と記録例5項目が明記されているため、安田記念の収集時にメモリが書かれたのはこの指示どおりの動作。

**明示指示時：** ユーザーが「これを覚えて」と言えば保存、「忘れて」と言えば該当エントリを削除する [2]。

## 2. agent-memory はどこで制御するのか

結論：**エージェント定義ファイル（`.claude/agents/keiba-tansho-picker.md`）で制御する。CLAUDE.md ではない。**

| 制御したいこと | 場所・方法 |
|---|---|
| 有効/無効 | frontmatter の `memory` フィールド。削除すれば無効化 [1] |
| 保存場所・共有範囲 | `memory: user`（`~/.claude/agent-memory/`・全プロジェクト共通）/ `project`（`.claude/agent-memory/`・git共有可）/ `local`（`.claude/agent-memory-local/`・git管理外）[1] |
| **何を記録するか** | エージェント定義の**本文**に指示を書く。公式も「メモリ指示を直接 .md に含めよ」と推奨 [1]。記録基準を変えたいならここを編集するのが最も効く |
| 記録済み内容の修正・削除 | メモリファイルは平文Markdownなので直接編集・削除してよい [2] |

CLAUDE.md / auto memory との関係：CLAUDE.md は「人間が書く指示」、auto memory（`~/.claude/projects/<project>/memory/`）は**メインセッション用**の自動メモであり、サブエージェントの agent-memory とは別系統 [2]。サブエージェントは独立したコンテキストで動くため、agent-memory の記録方針を CLAUDE.md に書いても確実には届かない。エージェント自身の .md に書くこと。

**本プロジェクトの注意点：** `memory: project` は「git でチーム共有する」ことが本来の用途だが [1]、現状 `.gitignore` に `agent-memory/` が含まれており版管理されない。共有したいなら gitignore から除外、個人メモ扱いなら `memory: local` への変更が筋が通る（昨日のレビュー指摘6と同件）。

## 3. 「何を残し、何を消すか」の一般的な基準

Anthropic 公式（ドキュメント [1][2] および同社のメモリ整理スキル consolidate-memory [3]）と、エージェントメモリ研究・実務記事 [4][5] に共通する原則は以下に集約される。

**残すもの（durable：恒久的知識）**
- 何度も再利用するパターン・傾向（コードベースの規約に相当。本件ならコース特性）
- 再導出コストが高いノウハウ（試行錯誤の末に得た手順・回避策）
- ユーザーの好み・作業方針
- 判断の背景・理由（なぜそうしたか）

**消す・畳むもの（dated：日付つき情報）**
- 期日が過ぎたタスク・終わった案件の詳細。恒久的な学びだけを抽出して durable 側に畳み、元は削除する [3]
- 外部ソースから即座に再取得できる情報（キャッシュと重複するもの）[3]
- 重複・矛盾するエントリ（古い方を削除または統合）[4][5]
- 検証されていない予測・仮説（結果と照合してから「検証済みの教訓」として書き直す）

**運用ルール**
- 相対日付（「来週」「今年」）は絶対日付に直す [3]
- `MEMORY.md` は索引に徹する：1エントリ1行・約150字以内、全体200行以内 [3]
- 定期的に整理パスを回す（マージ・棚卸し・索引更新）[3][4]

## 4. 本プロジェクトへの推奨（私の結論）

判断基準を一言で言うと：**「メモリには判断の仕方を残し、判断の結果は predictions/feedback に残す」**。エージェントメモリは判断軸を磨くためのメタ知識置き場であり、個別レースの予想結果は SPEC の TODO-01/02 で計画している `predictions/`・`feedback/` フォルダの守備範囲。

現在の2ファイルへの具体的な適用：

**`feedback-netkeiba-collection-method.md` → そのまま残す。**
収集手法のノウハウ（resultページ+browser_batch、WebFetchの不可パターン）は典型的な durable 知識。再導出コストが高く、毎回の収集で再利用される。理想的なメモリの見本。

**`insight-yasuda-kinen-tokyo-mile.md` → 分割して整理。**
- 残す：東京芝1600mの構造的傾向（差し優勢・上がり勝負）、安田記念の恒久傾向（外枠好走・波乱含み）。`insight-tokyo-shiba-1600m.md` 等に改名して恒久知識として保持
- 削除（移管）：2026年出走馬の評価・「14ガイアフォースが軸最有力」等の単年予想。レース（6/7）は終了済みのため、実際の結果と照合した上で feedback 記録（TODO-02の第1号として手動作成を推奨）に移し、メモリからは削除する。**検証されていない予想をメモリに残すと、次回の予想がその先入観に引きずられる**（メモリは毎回システムプロンプトの索引経由で参照されるため）
- 照合の結果「外枠差し有利は今年も有効だった/無効だった」という検証済みの教訓が得られたら、それだけを恒久側に1行追記する

**今後のための仕組み化（推奨）：**
`keiba-tansho-picker.md` のメモリ指示部に以下の基準を追記する。

```
メモリに残すもの：競馬場・コース・距離の恒久的傾向 / 検証済みの判断教訓 / 収集手法のノウハウ
メモリに残さないもの：個別レースの予想・単年の出走馬評価（レース後にfeedbackへ移し削除）/ キャッシュから再取得できる情報
レース結果の判明後：関連する予想メモリを結果と照合し、教訓のみ残して予想本体は削除する
```

これにより「何を残すか」の判断がエージェント任せでなくなり、メモリの腐敗（stale化）を構造的に防げる。

---

## 参考文献

[1] Claude Code 公式ドキュメント「Create custom subagents — Enable persistent memory」 https://code.claude.com/docs/en/sub-agents
[2] Claude Code 公式ドキュメント「How Claude remembers your project」 https://code.claude.com/docs/en/memory
[3] Anthropic製 consolidate-memory スキル（Cowork同梱・メモリ整理の公式手順：durable/dated の分離・重複マージ・索引200行以内）
[4] A Practical Guide to Memory for Autonomous LLM Agents (Towards Data Science) https://towardsdatascience.com/a-practical-guide-to-memory-for-autonomous-llm-agents/
[5] Memory scaling for AI agents (Databricks Blog) https://www.databricks.com/blog/memory-scaling-ai-agents
