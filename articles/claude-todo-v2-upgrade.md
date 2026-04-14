---
title: "Claude Code の /todo スキルをOSS化するまで——V1モノリスからV2三層設計へ"
emoji: "🗂️"
type: "tech"
topics: ["claudecode", "gtd", "nodejs", "octokit", "typescript"]
published: false
---

## V1を振り返る

Claude Code の `/todo` スキルを最初に作ったとき、すべてを1つのMarkdownファイルに書いた。

Claudeへの指示、バリデーション用のbash case文、日付変換ロジック、エラーハンドリング——これらが混在した巨大な `todo.md` だ。開発日記によると、V2の開発途中でこのファイルが2,011行に達した記録が残っている（V1当初の正確な行数はgit履歴で確認できなかった）。

何が問題だったか。Claudeへの「指示」と、毎回同じ結果を返す「定型コード」が区別されていなかったことだ。

Claudeはファイル全体を読んでから実行する。バリデーションのbash case文も、API呼び出しのコードブロックも、毎回コンテキストを消費する。修正したいときはMarkdownの中からロジックを探す必要があり、セキュリティバリデーションのコードレビューすらMarkdownを読まないとできない状態だった。

## V2: 三層への分離

V2で採った設計はシンプルだ。

```
V2の構成:
  todo.md          172行  ← Claudeへの指示のみ
  todo-engine.js   ????行  ← Node.jsエンジン（コードのみ）
  todo.sh            33行  ← シェルラッパー（接続のみ）
```

`todo.md` の行数は172行（実測）。`todo.sh` は33行（実測）。`todo-engine.js` のサイズは110,422バイトだが、正確な行数は現時点で確認できていない（[要確認]）。

分離の設計思想はこうだ。**Claudeへの指示書には、Claudeが判断に使う情報だけを書く。定型コードは書かない。**

`todo.md` にはコマンドの仕様と振る舞いのルールだけを記述する。`todo-engine.js` は普通のNode.jsスクリプトで、Claude非依存だ。Claudeはこのエンジンを呼び出す指示を出すだけでよい。

「コードをJSに移しただけで、総量は変わっていないじゃないか」という指摘はそのとおりだ。コードの量は減っていない。何が変わったかというと、**テストが書けるようになった**ことと、**コードレビューの対象が明確になった**ことだ。Markdownに埋め込まれたbash case文はユニットテストを書けない。分離されたJS関数なら書ける。

## Octokitマイグレーションと、Windowsで踏んだ2つの罠

V1では GitHub 操作をすべて `gh` CLI で実行していた。各コマンドがプロセスを生成するため、複数のAPI操作が必要な場合は逐次プロセス起動が走る。

V2では `@octokit/rest` を使ってNode.jsから直接HTTP通信する構成に変えた。これで `apiMain` という1つの非同期関数にすべてのGitHub API操作を集約できた（list-issues, view-issue, create-issue, edit-issue, close-issue など）。

Windowsで実装したため、2つの罠を踏んだ。

**罠1: `@octokit/rest` はESM only**

`require()` が使えない。`import()` を使う必要があるが、Windowsのパスをそのまま渡すと動かない。`pathToFileURL()` で変換してから動的インポートする形で解決した。

```js
const { Octokit } = await import(pathToFileURL(octokitPath).href);
```

**罠2: Node.js 24 + Octokit で exit code 3221226505**

`process.exit(1)` を呼ぶとHTTP接続プールが開いたまま libuv の assertion error が発生し、exit code 3221226505（16進で 0xC0000409、Windows のスタック破壊エラー）でクラッシュする。

対処は `process.exit` を `throw` に変えること。例外を上位に伝播させてNode.jsが自然終了するようにした。現状のコードを確認したところ、`process.exit` は63箇所残っている（デプロイ済みファイルで実測）。入力バリデーション（コマンド引数チェック）など、Octokit起動前のパスに集中している。Octokit使用後のパスでは `throw` を使っている。完全な置き換えではなく、クラッシュが発生する経路を限定的に対処した形だ。

この exit code 3221226505 の件が公式 Issue に報告されているかは未確認。

## 新機能: GTDの「回し方」が変わった

アーキテクチャ刷新に伴って、いくつかの機能を追加した。

### `routine` カテゴリ（GTD 6→7カテゴリ）

V1のGTDカテゴリは6つだった。

```
V1: inbox / next / waiting / someday / project / reference
V2: inbox / next / routine / waiting / someday / project / reference
```

`routine` は繰り返しタスク専用のカテゴリだ。GTDの原典にはないが、習慣タスクと緊急タスクが `next` に混在する問題を解決するために追加した。

### `today` コマンドと `dashboard`

`/todo today` で今日のスコープだけを表示する。4セクション構成だ。

- 期限超過タスク
- 今日が期限のタスク
- 今日のルーティン
- 未実施のルーティン

`dashboard` は1週間スコープ。朝は `today`、週次レビュー前は `dashboard` と使い分けられる。

### `help` コマンド

コマンド数が増えてコマンド名を覚えられなくなったため追加した。カテゴリ別（タスク管理 / コンテキスト / 一括操作 / レビュー・分析）で一覧表示する。

### i18n（英語対応）

`LANG_ENV=en` を設定すると全出力メッセージが英語になる。翻訳辞書（`MESSAGES` オブジェクト）のキー数は100以上とネタ帳には記載があるが、正確なキー数は未確認（[要確認]）。日付入力は言語設定に関係なく日英両対応しており、「明日」も「tomorrow」も常に受け付ける。

### ラベルの絵文字化

GitHub Mobileでの視認性向上のため、GTDラベルに絵文字プレフィックスを追加した。

```
📥 inbox  🎯 next  🔁 routine  ⏳ waiting  🌈 someday  📁 project  📎 reference
```

## OSS化のためにやったこと

V1は自分のリポジトリ名をコードに直書きしていた。

```
# V1（例）
REPO="saitoko/000-partner"
```

これを環境変数に変えた。

```
# V2
export TODO_REPO_OWNER="your-org"
export TODO_REPO_NAME="your-repo"
```

`.env.example` を用意してあるので、クローンして変数を設定するだけで動く。`GH_TOKEN` は `.env` に書くか、`gh auth token` で自動取得する。

セットアップ手順は README に書いてある。ただし前提は「Node.js が入っていること」と「GitHub CLI (`gh`) がインストールされていること」だ。CoWork（Claude の Web環境）でも動くよう、`todo.sh` が `.env` ファイルを複数階層で探索する仕組みも入れている。

## テストについて正直に書く

「テスト600件超」とネタ帳に書いてあったが、実際に `run-tests.sh` のアサーション数を数えたところ378件だった（実測）。

開発日記に記録されているテスト数の変遷はこうだ。

| 時点 | 件数 |
|------|------|
| Day 1 終了 | 174件 |
| Day 2 夜 | 300件 |
| Day 2 深夜 | 330件 |
| Day 3（i18n対応後） | 437件 |

現在の378件がこの系列のどの時点に対応するかは、スクリプトの内容から判断できない。機能追加・削除・統合でアサーションが増減した可能性がある。

テストの構成はローカルテスト（§1〜§26）とGitHub統合テスト（§A〜§AC）の2層だ。ローカルテストは `todo-engine.js` に依存しない純粋な関数テスト（日付正規化、バリデーション、文字列組み立て）で、GitHub APIを使わずに実行できる。

## 現在地と残課題

公開リポジトリは https://github.com/saitoko/claude-todo-gtd だ。

まだできていないこと、うまくいっていないことも書いておく。

- `process.exit` が63箇所残っており、Windows環境での挙動は「問題が出るパスを避けた」状態。完全な解決ではない
- テスト件数の「600件超」という数字は実態と合っていなかった。正確な計測ができていなかった
- i18n の翻訳カバレッジは未測定。「全出力メッセージが英語化」と書いているが、漏れがないかの確認は取れていない
- CoWork以外の環境での動作確認は限定的。Windows + Node.js 24 の組み合わせでしか実測していない

Pro版機能（ダッシュボード、デイリーレビュー、カスタムビュー、レポート）の実装詳細は別記事に書く予定だ。

V1ユーザーは `todo-engine.js` と `todo.sh` を追加し、`.env` に環境変数を設定すれば移行できる。`todo.md` の内容も更新が必要なため、リポジトリの最新版を参照してほしい。
