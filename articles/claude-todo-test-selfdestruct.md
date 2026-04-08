---
title: "テストが ~/.claude/ を消し飛ばした話 — HOME 差し替えの罠"
emoji: "💥"
type: "tech"
topics: ["テスト", "bash", "nodejs", "windows", "claudecode"]
published: true
---

## テストは通る。だが代償があった。

```
==========================================
結果: 363 / 363 テスト通過
==========================================
```

最高の気分でした。英語対応のテストを53件追加して、全部グリーン。完璧。

「よし、次の作業に...」と Claude Code に話しかけたら、こう返されました。

**「ログインしてください」**

は？ さっきまで普通に動いてたよね？

再ログインして作業を続け、しばらくしてまたテストを回す。全パス。Claude Code に戻る。

**「ログインしてください」**

ここで気づきました。テストを回すたびにログイン情報が消えている。363個のテストは全部通っているのに、テスト自体が Claude Code を殺していたのです。

**テストが自分の開発環境を自己破壊していました。**

## 何が起きていたか

Claude Code 用の `/todo` コマンドを開発中、テストを実行するたびに **`~/.claude/` ディレクトリごと消滅**していました。ログイン情報（`.credentials.json`）、セッション、設定ファイル、全部。

## 罠その1: HOME 復元後の rm -rf

テンプレート機能の英語出力テストで、本物の `~/.claude/todo-templates.json` を汚染しないよう `HOME` を一時ディレクトリに差し替えていました。

### 犯人のコード

```bash
# HOME を一時ディレクトリに差し替え
REAL_HOME="$HOME"
export HOME=$(mktemp -d /tmp/todo-test-home-XXXXXX)
mkdir -p "$HOME/.claude"
printf '{}' > "$HOME/.claude/todo-templates.json"

# ... テスト実行 ...

# HOME を戻す
export HOME="$REAL_HOME"

# 一時ディレクトリをクリーンアップ
rm -rf "$HOME/.claude"   # ← ここ
```

最後の行、お気づきでしょうか。

`HOME` を本物に戻した**後**に `$HOME/.claude` を削除しています。つまり **本物の `~/.claude/` が消えます。**

ログイン情報（`.credentials.json`）、セッション情報、設定ファイル、すべて消滅。

### 修正

一時ディレクトリのパスを別変数で保持するだけです。

```bash
REAL_HOME="$HOME"
FAKE_HOME=$(mktemp -d /tmp/todo-test-home-XXXXXX)  # パスを保持
export HOME="$FAKE_HOME"
mkdir -p "$HOME/.claude"
printf '{}' > "$HOME/.claude/todo-templates.json"

# ... テスト実行 ...

export HOME="$REAL_HOME"
rm -rf "$FAKE_HOME"   # 一時ディレクトリだけ消す
```

直して安心...と思ったら、テストを回すとまだテンプレートファイルが汚染されていました。

## 罠その2: Windows の `os.homedir()` は `HOME` を無視する

テスト対象のエンジン（Node.js）はテンプレートのパスを `os.homedir()` で計算しています。

```javascript
function getTemplatePath() {
  return path.join(os.homedir(), '.claude', 'todo-templates.json');
}
```

Linux/macOS なら `os.homedir()` は `HOME` 環境変数を参照します。しかし **Windows では `USERPROFILE` を参照**します。

```bash
# Windows の bash (Git Bash) で検証
HOME=/tmp/fake node -e "console.log(require('os').homedir())"
# → C:\Users\saito    ← HOME を無視！
```

テストで `HOME` を差し替えても、Node.js は本物のホームディレクトリを見続けていたのです。

| OS | `os.homedir()` が参照する値 |
|----|----|
| Linux / macOS | `HOME` 環境変数 |
| Windows | `USERPROFILE` 環境変数 |

### 修正

`HOME` と `USERPROFILE` の両方を差し替えます。

```bash
REAL_HOME="$HOME"
REAL_USERPROFILE="${USERPROFILE:-}"
FAKE_HOME=$(mktemp -d /tmp/todo-test-home-XXXXXX)
export HOME="$FAKE_HOME"
export USERPROFILE="$FAKE_HOME"

# ... テスト実行 ...

export HOME="$REAL_HOME"
if [ -n "$REAL_USERPROFILE" ]; then
  export USERPROFILE="$REAL_USERPROFILE"
else
  unset USERPROFILE
fi
rm -rf "$FAKE_HOME"
```

## おまけ: grep が絵文字を食えない

テスト修正中にもう1件バグを見つけました。

```bash
echo '## 📥 Inbox' | grep -aq '## 📥 Inbox' && echo MATCH || echo NO_MATCH
# → NO_MATCH
```

Windows の bash 環境で `grep` が4バイト UTF-8 絵文字（`📥` = U+1F4E5）をマッチできません。3バイト絵文字の `✅` (U+2705) は動くのに、4バイトの `📥` はダメ。`grep -P`（Perl正規表現）なら動きます。

テストの意図は「英語モードで日本語ヘッダーが出ない」ことの確認だったので、アプローチを変更しました。

```bash
# Before: 絵文字を含むパターンでマッチ（失敗する）
assert_contains "en: Inbox header" "## 📥 Inbox" "$OUTPUT"

# After: 日本語が含まれないことを確認
assert_not_contains "en: no JP in header" "受信トレイ" "$OUTPUT"
```

## 教訓

### 1. テストの後片付けは「差し替え前のパス」で行う

環境変数を差し替えたら、クリーンアップは差し替え先の変数で。復元後の変数を使うと本番環境を破壊します。

```bash
# NG
export HOME="$REAL_HOME"
rm -rf "$HOME/.claude"        # ← 本物を消す

# OK
export HOME="$REAL_HOME"
rm -rf "$FAKE_HOME"           # ← 偽物を消す
```

### 2. Windows + Node.js では `USERPROFILE` も差し替える

`os.homedir()` の挙動は OS で異なります。クロスプラットフォームのテストでは `HOME` と `USERPROFILE` の両方を差し替えましょう。

### 3. テストが何を壊すかは、テスト自身は教えてくれない

テストは全部グリーンでした。「363テスト通過」と表示された直後に、Claude Code が再ログインを求めてきます。テストの成功は「テスト対象が正しい」ことしか保証しません。テスト自体の副作用は別の話です。

## まとめ

テストを安全に書くのは、テスト対象のコードを書くのと同じくらい大事です。特にファイルシステムや環境変数を操作するテストでは、「何を触って、何を戻すか」を慎重に設計する必要があります。

幸い、`~/.claude/` は再ログインで復元できるものだったので実害は限定的でしたが、これが `~/.ssh/` や `~/.gnupg/` だったらと考えると背筋が冷えます。
