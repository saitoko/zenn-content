---
title: "Claude Code でAIチームを組織した — CEO・DevOps・Writer・Researcher の役割分担と運用設計"
emoji: "🏢"
type: "tech"
topics: ["claudecode", "ai", "multiagent", "productivity", "automation"]
published: false
---

「1つの Claude Code で何でもやる」に限界を感じたのは、CLAUDE.md が50行を超えたあたりでした。

コーディングルール、タスク管理のフロー、記事の書き方、パスの方針――全部1つのファイルに書いていたら、Claude が指示を守ったり守らなかったりする。200行を超えると遵守率が下がるという話は本当でした。

そこで、Claude Code を「1人の万能アシスタント」から「役割を持ったチーム」に再編成することにしました。この記事では、7体のエージェントからなるマルチエージェント組織を設計・構築した過程を、実際の設定ファイルとともに紹介します。

## できあがった組織図

最終的にこういう構成になりました。

```
CEO ─── オーケストレーション（人間が直接対話）
├── devops ──── 開発基盤・インフラメンテナンス
├── resarcher ── 調査・分析・レビュー（品質管理）
├── secretary ── 情報整理・GitHub Issue管理
├── writer ───── 記事執筆・Zenn/Qiita/はてな投稿
├── todo-dev ─── /todo スキル開発
└── health-dev ─ /health スキル開発
```
> ※ この記事は執筆時点（2026年4月初旬）の構成です。その後 reviewer を独立エージェントとして分離しました。

人間が対話するのは CEO だけ。CEO が状況を判断して、各エージェントに仕事を振ります。開発はしない。コードも書かない。**指示と判断だけ**が CEO の仕事です。

## なぜ「組織」にしたのか

きっかけは3つありました。

**1. CLAUDE.md の肥大化**

最初は1つの CLAUDE.md に全ルールを書いていました。「日本語で応答して」「ログは logs/ に保存して」「パスは $HOME ベースで」「記事はこういうトーンで」……。便利に使うほど CLAUDE.md が太っていく。50行を超えたあたりで、後半に書いたルールが無視されることが増えました。

**2. コンテキストの混線**

記事を書いている最中に「テストも直して」と頼むと、ライターモードと開発モードが混ざって中途半端な結果が返ってくる。1つのインスタンスに複数の人格を持たせるのは無理がありました。

**3. 「1エージェント＝1責務」の原則**

ソフトウェア設計の単一責任原則と同じです。1つのエージェントに1つの役割。「何をするエージェントか」が明確なほど、Claude のパフォーマンスは上がります。

## インフラ3層の設計

組織を支えるインフラは3つの層で構成されています。

### 第1層: CLAUDE.md 階層 — 「性格付け」

ルートの CLAUDE.md はコアルールだけ。約20行に圧縮しました。

```markdown
# CLAUDE.md — パートナープロジェクト

このプロジェクトはClaude Codeの中心的なオーケストレーションハブです。

## 基本ルール

- 常に日本語で応答する
- 作業ログは `logs/` に記録する（命名規則・引き継ぎヘッダーは `logs/README.md` 参照）
- エージェントへの指示書は `agents/` に保存する
- 重要な決定事項はメモリに保存する

## エージェント組織

詳細は `.claude/rules/agent-roles.md` を参照。サブエージェント定義は `.claude/agents/` に配置。

CEO ─ devops / resarcher / secretary / writer / todo-dev / health-dev

## パス記述方針

詳細は `.claude/rules/path-policy.md` を参照。

- 絶対パスを避け、相対パスまたは `$HOME` ベースを使う
- `.exe` 参照は `$OSTYPE` 判定で OS 分岐する
```

**ポイントは「詳細は別ファイルを参照」という書き方**。エージェント組織の詳細は `.claude/rules/agent-roles.md` に、パス方針は `.claude/rules/path-policy.md` に分離しました。ルート CLAUDE.md は「全エージェントが毎回読むコア」なので、短ければ短いほどいい。

各エージェントのワーキングディレクトリにも個別の CLAUDE.md を置いて「性格付け」しています。たとえば resarcher の CLAUDE.md はこうです（抜粋）。

```markdown
# CLAUDE.md — Resarcher

## 役割
調査・分析を行うリサーチエージェント。加えて、他エージェントの成果物に対する
レビューも担当する。

## 担当業務

### 調査・分析
- デイリーリサーチ（`agents/daily-research.md` の指示に従う）
- テーマ別の深掘り調査
- 競合分析・市場調査

### レビュー（品質管理）
- 指示書レビュー: `review: required` が付いた指示書を実行前にレビューする
- 記事レビュー: Writer が `logs/` に置いたドラフトのファクトチェック・読みやすさ確認
- スキルレビュー: Todo-Dev / Health-Dev の成果物に対する整合性チェック
- レビュー結果は `logs/YYYY-MM-DD_review_{トピック}.md` として記録する
```

「あなたは調査担当です」と書くだけで、Claude は調査に特化した動きをしてくれます。余計なことをしない。開発もしない。記事も書かない。**役割を絞るだけで精度が上がる**のは、人間のチームと同じです。

### 第2層: `.claude/agents/` — サブエージェント定義

Claude Code の公式機能で、エージェントごとにツール制限とモデルを指定できます。実際のファイルはこうなっています。

```markdown
# .claude/agents/resarcher.md
---
name: resarcher
description: 調査・分析・レビューを行うリサーチエージェント
tools: Read, Grep, Glob, WebSearch, WebFetch, Write, Bash
model: sonnet
---

調査・分析を行うリサーチエージェント。加えて、他エージェントの成果物に対する
レビューも担当する。
```

```markdown
# .claude/agents/devops.md
---
name: devops
description: 開発基盤・インフラのメンテナンスを行うDevOpsエージェント
tools: Read, Grep, Glob, Write, Edit, Bash
model: sonnet
---

開発基盤・インフラのメンテナンス（スクリプト・設定・テスト・OS対応）を担当。
```

ここでの設計判断をいくつか。

**全エージェントを sonnet にした**。CEO（人間が直接対話）だけ opus を使い、サブエージェントは全員 sonnet。コスト効率を優先しました。sonnet でも十分な品質で、ほとんどの作業は問題なくこなせます。

**ツールは必要最小限**。たとえば writer には `Edit` を与えていません。記事は `Write` で新規作成するのがメインで、既存コードを直接編集する必要はないからです。逆に devops は `Edit` が必須ですが、`WebSearch` は不要。**使えるツールを絞ることで、エージェントが余計なことをするリスクが減ります**。

### 第3層: `.claude/rules/` — ルールのモジュール化

ルート CLAUDE.md から分離した詳細ルールを、トピック別のファイルに配置しています。

```markdown
# .claude/rules/path-policy.md
---
description: スクリプト・指示書でのパス記述方針
---

- スクリプト・指示書では絶対パスを避け、相対パスまたは `$HOME` ベースを使う
- Node.js では `os.homedir()` の代わりに `process.env.HOME || os.homedir()` を使う
- `.exe` 参照は `$OSTYPE` 判定で OS 分岐する
- `.mcp.json` や `settings.local.json` は環境変数展開不可のため、OS移行時に手動で変更する
```

`paths` フィールドを使えば、特定のディレクトリでのみ自動ロードされるルールも作れます。たとえば `paths: ["logs/**"]` と書けば、`logs/` 配下のファイルを扱うときだけそのルールが読み込まれる。コンテキストの節約になります（自プロジェクトでは現時点では未使用ですが、ルールが増えてきたら活用予定です）。

整備後のディレクトリ構成はこうなりました。

```
.claude/
├── agents/
│   ├── resarcher.md    # 調査・レビュー
│   ├── writer.md       # 記事執筆
│   ├── secretary.md    # Issue管理
│   └── devops.md       # インフラ
├── commands/
│   ├── daily-research.md  # デイリーリサーチ
│   └── gtd-collect.md     # GTD収集
└── rules/
    ├── agent-roles.md  # 組織図・責務
    └── path-policy.md  # パス方針
```

## エージェント間通信: logs/ プロトコル

エージェント同士の情報受け渡しには、MCP も API も使っていません。**ただの Markdown ファイル**です。

### ファイル命名規則

```
logs/YYYY-MM-DD_{種別}_{トピック}.md
```

種別は5つ。

| 種別 | 用途 | 例 |
|------|------|-----|
| `report` | 調査レポート | `2026-04-07_report_multi-agent-instruction-management.md` |
| `session` | 作業記録 | `2026-04-07_session_multi-agent-infra.md` |
| `ideas` | ネタ帳 | `2026-04-07_ideas_multi-agent-org-article.md` |
| `handoff` | 引き継ぎ | `2026-04-07_handoff_multi-agent-articles.md` |
| `draft` | 記事ドラフト | `2026-04-07_draft_zenn-article.md` |

### YAML frontmatter で宛先と状態を管理

各ログファイルの冒頭にこういうヘッダーをつけます。

```yaml
---
from: resarcher
to: CEO
status: ready
review: skip
---
```

`from` は作成者、`to` は宛先、`status` は状態（`draft` / `ready` / `in_progress` / `done`）。`review` は指示書の事前レビューを制御するフィールドで、`required`（レビュー必須）か `skip`（直接実行可）を指定します。これだけで「誰から誰へ」「どの段階か」「レビューが必要か」がわかります。

未処理のタスクを検出するのも簡単です。

```bash
grep -rl "status: ready" logs/
```

素朴ですが、素朴だからこそ壊れにくい。MCP サーバーが落ちる心配もない。ファイルシステムが動いていれば通信できます。

## レビュー差し戻しの実例

「組織が回っている」ことを示す一番わかりやすい証拠は、**品質ゲートが機能しているか**だと思います。

実際に、DevOps エージェントが作成したCLIツールのインストール指示書を CEO がレビューして差し戻しました。配置先パスが既存スクリプトと不一致、アセット名が不正確、エージェント用途に不要なツールの混入――「動くけど正しくない」を事前に潰すのが品質ゲートの役割です。

この差し戻しでは `review: required` フィールドが活きました。インフラ系の指示書にはレビュー必須を設定しておくことで、resarcher が実行前にファクトチェックを行います。記事執筆のような定型業務は `review: skip` で直接実行。**全件レビューではなくリスクベースで品質ゲートを制御する**仕組みです。

差し戻しの具体的なやりとり（指摘6点の詳細）は、[続編の体験記事](https://zenn.dev/tottoko_hamu/articles/claude-code-multi-agent-day)で紹介しています。

## Before / After

この組織化で何が変わったか、数字で見てみます。

| 指標 | Before | After |
|------|--------|-------|
| ルート CLAUDE.md | 50行 | 約20行 |
| エージェント定義 | なし | `.claude/agents/` に4ファイル |
| ルール管理 | CLAUDE.md に全部入り | `.claude/rules/` に分離 |
| エージェント間通信 | 口頭（チャットで伝達） | logs/ プロトコル |
| 品質ゲート | なし | レビュー→差し戻し→修正サイクル |

一番変わったのは、**CLAUDE.md の遵守率**です。20行まで削ったルート CLAUDE.md は、ほぼ100%守られるようになりました。以前は後半のルールが無視されることがあったのが解消されています。

## はじめ方: 最小構成から

いきなり7体のエージェントを作る必要はありません。最小構成は「CEO + 1エージェント」です。

### ステップ1: ルート CLAUDE.md を書く

```markdown
# CLAUDE.md
- 常に日本語で応答する
- 作業ログは logs/ に記録する
```

### ステップ2: `.claude/agents/` にエージェントを1つ作る

```markdown
# .claude/agents/researcher.md
---
name: researcher
description: 調査・分析を行うリサーチエージェント
tools: Read, Grep, Glob, WebSearch, WebFetch, Write, Bash
model: sonnet
---

調査・分析を担当。成果物は logs/ に保存する。
```

### ステップ3: logs/ でやり取りする

エージェントの成果物を `logs/` に保存するルールを決めるだけで、非同期のコミュニケーションが成立します。

2体で回してみて、「もっと分割したい」と感じたら増やす。**組織は必要に応じて育てるもの**で、最初から完成形を目指す必要はありません。

## まとめ

Claude Code のマルチエージェント組織は、特別な技術なしで作れます。使うのは Markdown ファイルだけ。

- **CLAUDE.md 階層**で各エージェントの性格を定義する
- **`.claude/agents/`** でツール制限とモデルを設定する
- **`.claude/rules/`** でルールをモジュール化する
- **logs/ プロトコル**でエージェント間の通信をファイルベースで行う
- **レビュー→差し戻し→修正**のサイクルで品質を担保する

個人開発者でも「組織」は作れます。1人の万能アシスタントより、役割を持ったチームのほうが強い。それは AI でも人間でも同じでした。
