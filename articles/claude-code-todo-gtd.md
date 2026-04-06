---
title: "Claude Code で GTD を回す /todo スラッシュコマンドを作った"
emoji: "✅"
type: "tech"
topics: ["claudecode", "gtd", "github", "cli", "productivity"]
published: true
---

## 作ったもの

Claude Code 用の GTD タスク管理スラッシュコマンド `/todo` を作りました。

https://github.com/saitoko/claude-todo-gtd

GitHub Issues をバックエンドに使い、ターミナルから `/todo` と打つだけでタスクの追加・管理・レビューが完結します。

## なぜ作ったか

Claude Code をオーケストレーションハブとして使い始めたところ、タスク管理が自然と必要になりました。

既存ツール（Todoist、Notion等）を使う手もありましたが、

- Claude Code の中で完結したい（コンテキストスイッチを減らしたい）
- `/todo next 設計書を書く @PC --due 明日` のように自然言語に近い形で入力したい
- GitHub Issues なら Web UI でも確認できる

という理由から自作することにしました。

## 主な機能

### GTD 6カテゴリで仕分け

inbox / next / waiting / someday / project / reference の6カテゴリをラベルで管理。

```bash
/todo next 上司に報告する @会社
/todo waiting 見積もり回答待ち @上司
/todo someday Rust を勉強する
```

### 日本語で日付を指定できる

`--due` オプションは日本語の相対表現に対応しています。

```bash
/todo next レポートを書く --due 明日
/todo next 月次報告 --due 来週金曜
/todo next 健康診断 --due 3日後
```

「今日 / 明日 / 明後日 / 来週 / 来月 / 今週末 / 今月末 / 来月末 / N日後 / N週間後 / Nヶ月後 / 来週月曜〜日曜」の14パターンに対応。Node.js で変換しています。

### 繰り返しタスク

完了時に次回分を自動作成します。

```bash
/todo next 週次レポート --due 来週月曜 --recur weekly
/todo done 5
# → ✅ #5 を完了しました。繰り返しタスク #12 を 2026-04-14 で作成しました。
```

daily / weekly / monthly / weekdays の4パターン対応。

### コンテキストでフィルタ

GTD の「コンテキスト」をラベルで表現。場所や状況に応じてタスクを絞り込めます。

```bash
/todo list @PC        # PCがあるときにやれるタスク
/todo list @会社      # 会社にいるときにやるタスク
/todo list next @PC   # next × @PC のANDフィルタ
```

### 週次レビュー

GTD の肝である週次レビューを対話形式で実施。

```bash
/todo weekly-review
```

1. Inbox 仕分け → 2. Next 確認 → 3. Waiting 確認 → 4. Projects 確認 → 5. Someday 見直し → 6. サマリー表示

### その他

- **優先度**: p1(🔴) / p2(🟡) / p3 の3段階
- **一括操作**: `bulk done`, `bulk move`, `bulk tag` 等
- **テンプレート**: 繰り返し作成するタスクをテンプレート化
- **統計**: `/todo stats` でカテゴリ別・優先度別の状況を表示
- **検索/アーカイブ**: 完了タスクの検索・再オープン

## セキュリティ

GitHub Issues の body にはユーザーが何でも書けるため、セキュリティは最初から意識しました。

- シェルインジェクション対策（ユーザー入力は変数経由で渡す）
- プロンプトインジェクション対策（Issue データは命令として実行しない）
- 入力値バリデーション（日付・数値・ラベル名を POSIX case 文で検証）

計8ルールを todo.md の冒頭に配置しています。

## 技術的なこと

- **実装形式**: Claude Code カスタムスラッシュコマンド（Markdown + Bash + Node.js）
- **バックエンド**: GitHub Issues API（`gh` CLI 経由）
- **テスト**: ローカルユニットテスト 145+ 件、GitHub 統合テスト 41+ 件
- **日本語日付変換**: Node.js で14パターンの相対表現を `YYYY-MM-DD` に変換

サーバーは不要で、`todo.md` を `~/.claude/commands/` にコピーするだけで動きます。

## 2日間の開発記録

Day 0（4/4）に構想・初期実装、Day 1（4/5）にテスト運用・バグ修正・機能追加、Day 2（4/6）にリファクタリング・一括操作・統計機能を追加しました。

開発日記は CHANGELOG-DIARY.md としてリポジトリに公開しています。設計判断の背景や、ハマったポイント（Bash のサブシェル変数スコープ問題など）も記録しています。

## 使い方

```bash
# 1. ファイルをコピー
cp todo.md ~/.claude/commands/todo.md
echo '{}' > ~/.claude/todo-templates.json

# 2. todo.md 冒頭のリポジトリ名を自分のものに変更

# 3. Claude Code で /todo と入力
```

詳しくは README をご覧ください。

https://github.com/saitoko/claude-todo-gtd

フィードバックや改善提案は Issue でお待ちしています。
