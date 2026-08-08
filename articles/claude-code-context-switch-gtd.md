---
title: "割り込みで頭がリセットされる問題を、Claude Codeで解消した話"
emoji: "🔄"
type: "idea"
topics: ["claudecode", "gtd", "productivity", "github", "ai"]
published: true
canonical_url: "https://blog.saitoko.net/entry/2026/04/18/200000"
---

# 割り込みで頭がリセットされる問題を、Claude Codeで解消した話

## 「あれ、さっき何やってたっけ」は何度でも来る

Claude Codeで作業しているとき、Slackの通知。

> 「お客さんが至急XXの件について回答してほしいと言っています」

割り込みを処理して戻ってくると、画面にはコードが開いたまま。さっきどこまで考えていたか、完全に頭から消えている。「ああそうか、この関数を……いや待って、なんでこのファイルを開いてたんだっけ」。

これを繰り返してきた。

## CPUのコンテキストスイッチより、人間の頭のほうがずっとコストが高い

コンピュータ用語の「コンテキストスイッチ」は、CPUが複数のプロセスを切り替えるときに状態を保存・復元するコストのことを指す。ただでさえ負荷がかかる処理で、設計上の悩みどころでもある。

人間の頭はそれよりはるかに遅い。

グロリア・マーク（Gloria Mark, UC Irvine）は著書 *Attention Span*（2023）のなかで、「割り込まれたプロジェクトに戻るまでに平均25分以上かかる」と述べている。彼女自身のフィールド研究（Mark, González & Harris, CHI 2005）では、情報労働者は平均3〜12分ごとに作業を切り替えており、割り込まれた作業に戻るまでに2つ以上の別作業を経由することが観察されている。

「集中している状態に戻る」のコストがこれだけ大きいのに、割り込みは選べない。Slackの通知は来るし、急ぎの依頼は断れない。

では、どうするか。「割り込みを減らす」ではなく、「戻るコストを下げる」方向で考えてみた。

結論を先に言うと、Claude Code単体でも今日から使える方法がある。さらに一歩進めたい場合にGitHub Issueとの連携が選択肢に入る、という段階構成だ。

## 解決策①: Claude Code自体が記憶を持っている

同じセッション内であれば、Claude Codeは会話の文脈をそのまま持っている。割り込みから戻ってきたとき、こう打つだけでいい。

```
残作業なんだっけ？
```

返答の例（イメージ）:

> 現在の状況をまとめます。
> - `src/api/user.ts` の `updateProfile` 関数のエラーハンドリングを修正中でした
> - try-catchの追加は完了、次は `validateInput` のnullチェックを実装するステップです
> - テストは未実行です

さっきまでの作業文脈をそのまま返してくれる。自分で手帳に書かなくてもいい。

さらに便利なのが `/recap` 機能だ。一定時間操作しないと、セッションに戻るタイミングで「今やっていたこと」のサマリーが自動表示される。こちらは特に何もしなくていい。戻ってきたときにすでに要約が出ている。

なお、Claude Codeにはセッションを再開する機能（`--resume`）もある。セッションが途切れること自体は問題ではない。

## 解決策②: Claude Codeの外も一箇所で管理する `/todo`（読み飛ばしOK）

本当の課題は別のところにある——Claude Codeの中の文脈と、外にあるタスク（Slack・依頼メール）が別々の場所に散らばっていることだ。「Claude Codeでは覚えているが、Issue側に反映されていない」「Issueに次のアクションはあるが、Claude Codeのセッションにはその背景がない」という状態が起きる。

ここで使えるのが `/todo` スキルだ。自分でGTD運用のために作ったカスタムスキルなので、そのまま使うには別途セットアップが必要になる（作り方は[こちらの記事](https://zenn.dev/tottoko_hamu/articles/claude-code-todo-gtd)）。ただし、「GitHub Issueをハブにする」という考え方自体はスキルなしでも応用できる。`/todo` を打つと現在のNEXTアクションをIssueから引き出して表示してくれる。

```
/todo
```

返答の例（イメージ）:

> ## 現在のNEXTアクション
> - #42 `updateProfile` のバリデーション修正（🔴 Today）
>   - next: validateInputのnullチェック実装 → テスト実行
> - #38 APIドキュメント更新（🟡 今週中）
> - #51 Aさんからの依頼：見積もり回答（🔴 Today）
>   - next: 工数をまとめてSlackで返信

「どのIssueの、どのステップが残っているか」が即座にわかる。セッションをまたいでも、プロジェクトをまたいでも、ここを見れば戻れる。

### なぜGitHub Issueか

GTDでいう「頭の外に出す」先として、GitHub Issueが使いやすい理由がある。

- ラベル・優先度・依存関係（#番号でのリンク）で構造化できる
- Claude Codeから直接参照・更新できる
- エンジニアが普段使っているツールなので、別途覚えることがない

Issueに「次はこれをやる」と書いておくこと自体が、割り込みコストを下げる。

`/todo` はこの記事のために作った架空の例ではなく、公開している OSS の Claude Code スキル `claude-todo-gtd` だ。Issue の一覧表示・優先度変更・週次レビューまで、ここで見せたコマンドがそのまま動く。気になる方はリポジトリを覗いてみてほしい。

公開リポジトリ: https://github.com/saitoko/claude-todo-gtd

## うまくいかないこと

正直に書く。

**割り込みが来る前にIssueへ記録できないことが多い**。理想は「割り込まれる前にNextアクションを書いておく」だが、Slackの通知は脈絡なく来る。「今日こそ記録してから切り替えよう」と思っても、急ぎの依頼は記録を待ってくれない。

結果として、「戻ってきてからClaude Codeに聞く→その内容をIssueに書く」という後追い記録になることが多い。理想の順序とは逆だが、ゼロよりはずっといい。

**セッションを長く保つほど有効**という制約もある。「残作業は？」の精度は、セッション内で何をやっていたかに依存する。1時間以上経つと、セッション自体が長くなりすぎてコンテキストが薄まることもある。そのときは `/todo` のIssue側に頼ることになる。

## まとめ: Claude Codeの中と外を一箇所でつなぐ

| 状況 | 使う手段 |
|------|---------|
| Claude Code内の作業に戻るとき | 「残作業は？」と聞く、または `/recap` を見る |
| Claude Code外のタスクも含めて把握したいとき | `/todo`（筆者自作スキル）でIssueのNEXTアクションを確認 |

Claude Code単体でも、今日から「残作業は？」と聞くだけで使える。`/todo` はGitHub Issueとの連携が必要だが、一度仕組みを作るとClaude Codeの中と外の両方を一つの場所で管理できる状態になる。

割り込みはなくせない。でも、戻るコストは設計でかなり下げられる。

---

## 参考

- Gloria Mark, *Attention Span: A Groundbreaking Way to Restore Balance, Happiness and Productivity* (2023)
- Mark, González & Harris, "No task left behind? Examining the nature of fragmented work", CHI 2005
- Masicampo & Baumeister, "Consider it done! Plan making can eliminate the cognitive effects of unfulfilled goals", *Journal of Personality and Social Psychology*, 2011 (PubMed: 21688924)

---

## 関連記事

- [Claude Code で GTD を回す /todo スラッシュコマンドを作った](https://zenn.dev/tottoko_hamu/articles/claude-code-todo-gtd)（`/todo` スキルの詳細はこちら）
- [今度こそ！ Claude CodeでGTDを回す——/todo 完全ガイド](https://zenn.dev/tottoko_hamu/articles/claude-todo-gtd-guide)（インストール手順から日常運用まで1本でまとめたガイド）

この記事の実践をまとめた一冊があります。

[コードを書けない私がClaude Codeで「AIチーム」を作るまで（Zenn Books）](https://zenn.dev/tottoko_hamu/books/leading-ai-agents-without-code)

## この記事で使ったツールに関連する本

- [実践Claude Code入門（技術評論社）](https://www.amazon.co.jp/dp/4297153548/?tag=saitokohamu1-22)
- [Claude CodeによるAI駆動開発入門（技術評論社）](https://www.amazon.co.jp/dp/4297152754/?tag=saitokohamu1-22)
- [新装版 はじめてのGTD ストレスフリーの整理術（デビッド・アレン）](https://www.amazon.co.jp/dp/B0FQB574SS/?tag=saitokohamu1-22)

## この記事のテーマを深掘りした本

**[コードを書けない私がClaude Codeに「仕事」を任せるまで](https://zenn.dev/tottoko_hamu/books/delegating-work-to-claude-code)**
GTDタスク管理をまるごとClaude Codeに任せるまでの実録（序章無料）

シリーズ全6冊: [Vol.1 作るまで](https://zenn.dev/tottoko_hamu/books/leading-ai-agents-without-code) ／ [Vol.2 回すまで](https://zenn.dev/tottoko_hamu/books/operating-ai-team-without-code) ／ [Vol.3 書き続けるまで](https://zenn.dev/tottoko_hamu/books/writing-with-ai-team-without-code) ／ [Vol.4 仕組みを渡すまで](https://zenn.dev/tottoko_hamu/books/configuring-claude-code-without-code) ／ [Vol.5 仕事を任せるまで](https://zenn.dev/tottoko_hamu/books/delegating-work-to-claude-code) ／ [Vol.6 1人エージェントチーム](https://zenn.dev/tottoko_hamu/books/building-agent-team-with-claude-code)
