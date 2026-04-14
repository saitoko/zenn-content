---
title: "Claude Code の /todo コマンドを英語対応した（LANG_ENV=en）"
emoji: "🌐"
type: "tech"
topics: ["claudecode", "i18n", "gtd", "github", "cli"]
published: false
---

## 前回のあらすじ

Claude Code 用の GTD タスク管理スラッシュコマンド `/todo` を作りました。

https://zenn.dev/tottoko_hamu/articles/claude-code-todo-gtd

今回は、このコマンドを英語でも使えるようにした話です。

## なぜ英語対応？

`/todo` を GitHub で公開した以上、英語圏の Claude Code ユーザーにも使ってほしい。実際、Claude Code 自体が英語ベースのツールなので、日本語オンリーだと「面白そうだけど読めない」で離脱されてしまいます。あと単純に、英語で `--due tomorrow` と打てたらかっこいいじゃないですか。

## やったこと

環境変数 `LANG_ENV=en` を設定するだけで、出力がすべて英語に切り替わるようにしました。

**日本語（デフォルト）:**
```
## ✅ Next Actions（次のアクション）
  🔴 #1  設計書を書く  [@PC]  📅 2026-04-10

---
📊 next: 1件  ⚠️ 期限超過: 0  📅 今週期限: 1
```

**英語（LANG_ENV=en）:**
```
## ✅ Next Actions
  🔴 #1  Write design doc  [@PC]  📅 2026-04-10

---
📊 next: 1  ⚠️ Overdue: 0  📅 Due this week: 1
```

## 設計: 単一ファイル完結の i18n

i18n ライブラリや外部 JSON ファイルは使っていません。i18next や gettext も検討しましたが、`/todo` はスラッシュコマンド1ファイルで完結するのが売り。外部依存を足した瞬間にその良さが消えるので、最初から「辞書をファイル内に持つ」一択でした。

`todo-engine.js` 内にメッセージ辞書を持ち、3つのヘルパー関数で切り替えます。

```javascript
const LANG = process.env.LANG_ENV || 'ja';

const MESSAGES = {
  ja: {
    'section.next': '## ✅ Next Actions（次のアクション）',
    'list.overdue': '期限超過',
    'template.saved': 'テンプレート「{name}」を保存しました。',
    // ... 約60キー
  },
  en: {
    'section.next': '## ✅ Next Actions',
    'list.overdue': 'Overdue',
    'template.saved': 'Template "{name}" saved.',
    // ...
  }
};

function t(key) {
  return (MESSAGES[LANG] || MESSAGES.ja)[key] || MESSAGES.ja[key] || key;
}

function tpl(key, vars) {
  let s = t(key);
  for (const [k, v] of Object.entries(vars))
    s = s.replace('{' + k + '}', v);
  return s;
}

function cnt(n) {
  return LANG === 'ja' ? n + '件' : String(n);
}
```

`t()` で単純メッセージ、`tpl()` でプレースホルダ付きメッセージ、`cnt()` で日本語の「件」サフィックスを処理。これだけで全コマンドの出力を切り替えられます。

### なぜこの設計にしたか

1. **依存ゼロ**: npm install 不要。`todo-engine.js` 1ファイルで完結
2. **フォールバック安全**: 未定義キーは日本語にフォールバック → 翻訳漏れがあっても壊れない
3. **追加が簡単**: 新しい言語を追加するなら `MESSAGES.fr = { ... }` を足すだけ

## 英語日付パターン

`--due` オプションで英語の相対表現が使えるようになりました。

```bash
/todo next Write report --due tomorrow
/todo next Monthly review --due "next friday"
/todo next Health check --due "in 3 days"
/todo next Plan vacation --due "end of next month"
```

対応パターン:

| 入力 | 結果 |
|------|------|
| `today` | 今日 |
| `tomorrow` | +1日 |
| `day after tomorrow` | +2日 |
| `next week` | +7日 |
| `next month` | +1ヶ月 |
| `this weekend` | 次の土曜 |
| `end of this month` | 今月末 |
| `end of next month` | 来月末 |
| `in N days/weeks/months` | +N日/週/月 |
| `next Monday` ~ `next Sunday` | 来週の曜日 |

### 言語非依存パース

面白いポイントとして、日本語パターンと英語パターンは**言語設定に関係なく両方とも常にチェック**します。

```bash
# 英語モードでも日本語日付が使える
LANG_ENV=en /todo next Task --due 明日

# 日本語モードでも英語日付が使える
/todo next タスク --due "next friday"
```

日本語と英語は文字列レベルで衝突しないので、分岐する必要がありません。

## 対応したコマンド

エンジン側で出力する全コマンドを英語対応しました:

- **list** — セクションヘッダー、サマリー、フィルタ結果メッセージ
- **dashboard** — 期限超過、今日、今週期限、完了数
- **stats** — カテゴリ別、優先度別、期限状況、完了実績
- **report** — 生産性レポート全体
- **weekly-summary** — 週次レビューサマリー
- **template/view** — テンプレート・ビューの操作メッセージ
- **エラーメッセージ** — バリデーションエラー全種

`todo.md`（スキル本体）側も変更し、Claude が英語で応答するよう条件分岐を追加しました。

たとえば `stats` コマンドの英語出力はこうなります:

```
📊 Task Statistics
──────────────────
By Priority:
  🔴 p1: 2
  🟡 p2: 3
  ⚪ p3: 1

Deadlines:
  ⚠️ Overdue: 1
  📅 Due today: 2
  📅 Due this week: 3
```

## テスト

英語出力テスト62件を追加し、合計 **363テスト** すべてパス（`grep -c '"en:' run-tests.sh` 実測）。

```bash
# 日本語テスト（デフォルト）
bash tests/run-tests.sh
# → 363 / 363 テスト通過

# 英語出力の検証テスト例
LIST_EN_OUT=$(LANG_ENV=en OPEN_ENV="$MOCK" node "$ENGINE" list-all)
assert_not_contains "en: no Japanese in header" "受信トレイ" "$LIST_EN_OUT"
assert_contains "en: English summary" "next: 2" "$LIST_EN_OUT"
```

## 使い方

```bash
# 1. 環境変数を設定
export LANG_ENV=en

# 2. または CLAUDE.md に記述
# 環境変数: LANG_ENV=en

# 3. あとは普通に /todo を使うだけ
```

英語ドキュメント `README_EN.md` も追加しました。

https://github.com/saitoko/claude-todo-gtd

## まとめ

- `LANG_ENV=en` で全出力が英語に切り替わる
- 英語日付パターン（tomorrow, next week, in 3 days 等）に対応
- 日本語/英語の日付入力は言語設定に関係なく両方使える
- 外部依存ゼロ、1ファイル完結の i18n 設計
