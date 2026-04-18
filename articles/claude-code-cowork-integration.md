---
title: "公式ドキュメントが教えてくれないCowork×Claude Code連携の正解"
emoji: "🔗"
type: "tech"
topics: ["claudecode", "cowork", "claude", "ai", "gtd"]
published: false
---

:::message alert
CoworkおよびClaude Codeは機能追加・UI変更が頻繁に行われています。この記事の情報は**2026年4月18日時点**のものです。特にCoworkのUI手順は変更されやすいため、最新の公式情報と併せて確認してください。
:::

## この記事で伝えること

CoworkはデスクトップアプリからClaudeが直接ファイル操作・ブラウザ自動化を行える環境で、Claude Codeとは別の仕組みで動く。

Claude Coworkは2026年4月9日にGA（一般提供）へ移行したAnthropicのデスクトップAIエージェント環境だ（[Anthropic launches Claude Cowork in General Availability](https://www.testingcatalog.com/anthropic-launches-claude-cowork-in-general-availability/)）。重要な制約として、CoworkはLinuxサンドボックス上で動いており、Windows側の環境変数やファイルシステムと完全に分離されている。これを最初に知らないと、連携の試みが全て外れる。

問題は「同じプロジェクトでClaude CodeとCoworkを同時に使う」方法が公式ドキュメントの目立つ場所に書かれていないことだ。さらに、Coworkは2026年3月にUIが大きく変わっており、古い情報（セッション中にフォルダ追加ボタンでマウントできる）はすでに使えない。

この記事では、筆者が2026-04-10にCoworkを導入して実際に踏んだ3つの失敗と、それを経てたどり着いた解決策、そして現在運用中の3つの連携パターンを書く。失敗の再現手順から設計の背景まで、試した順番に記録する。

---

## 1. なぜ連携が必要か — 同じプロジェクトを2つのAIが操作する問題

マルチエージェント構成でなくても、Claude Codeで作業中のプロジェクトにCoworkを組み合わせると同じ問題が発生する。

Claude Codeで構築したマルチエージェント構成（researcher・writer・secretary など）を持つプロジェクトに、CoworkのAIエージェントを追加で参加させようとすると、すぐに問題が出る。

- **どのルールに従うか**が環境ごとにバラバラになる（CLAUDE.mdを参照できなければ別々のAIとして動く）
- **タスクの状態**がCode側とCowork側で分裂する（Code側でIssueを更新してもCowork側が知らない）
- **ファイルの変更**が片方にしか届かない（GitHubを介さないとCoworkが書いたファイルをCodeが読めない）

要するに、連携なしで両環境を動かすと「同じプロジェクトを全く別のAIが担当している」状態になる。ルール・タスク・ファイルの3つを共有する仕組みが必要だ。

---

## 2. 正解にたどり着くまでの3つの失敗

### 試行1: Chrome経由でGitHubのWebUIを開く

CoworkのBash環境からプライベートリポジトリにアクセスする方法がわからず、まずChrome経由でGitHubのWebUIを開くことにした。読み取りはできた。しかしファイルの書き込みやgit pushは当然できない。コピペで仲介するしかなく、ファイル1件ごとに手作業が発生する。

さらに追い打ちをかけたのが、デフォルトブランチの名前問題だった。`main` だと思ってURLを叩いたら404が返ってきた。実際は `master` だった。些細なことだが、手探り状態のときは小さなエラーが思いのほか消耗する。

**この方法の限界**: 読み取り専用、コピペ手作業、ブランチ名の確認が必要。1ファイルを反映するのにCoworkとブラウザを行き来する工程が毎回発生する。

### 試行2: GITHUB_TOKENをWindows側のBash環境に設定

「BashからAPIで直接アクセスすればいい」と考え、Windows側のシェルで `GITHUB_TOKEN` を設定した。しかし何度試してもCoworkのBash環境では認識されない。

冒頭に書いた通り、CoworkはLinuxサンドボックス上で動いており、Windows側の環境変数とは完全に分離されている。Windows側でどれだけ変数を設定しても、Coworkには届かない。

この挙動を知らないと「なぜ反映されないのか」でかなりの時間を溶かす。実際、設定方法を変えながら同じことを3回試した。

**この方法の限界**: CoworkのサンドボックスはWindows環境変数を継承しない。`GITHUB_TOKEN` をCoworkに渡す方法は環境変数経由では機能しない。

### 解決: プロジェクト作成時にフォルダを登録する（現在の正しい手順）

試行錯誤の末にたどり着いた正解は、**プロジェクト作成フロー**でローカルフォルダを事前に登録することだ。

ただし、ここには重要な変遷がある。

**過去の手順（現在は使えない）**: 以前はセッション中にチャット入力欄のフォルダ追加ボタン（`request_cowork_directory`）からマウントできた。しかしこのツールは2026年2月ごろ削除されており、セッション途中でのフォルダ追加は現在できない。復活予定もなく、[GitHub Issue #25797](https://github.com/anthropics/claude-code/issues/25797) が "not planned" でクローズされており、現時点で復活の予定は示されていない。

**現在の正しい手順**: 2026年3月下旬にProjects機能が導入され、UIがセッションベースから「プロジェクト（永続ワークスペース）」ベースへ転換した。ローカルフォルダへのアクセスは、プロジェクト作成時に設定する。

**フォルダマウントの具体的な手順（2026年4月時点）**:

以下は新規作成（ゼロから）の場合の手順:

1. Coworkの左サイドバーの「New project」をクリック
2. プロジェクト名を入力して作成
3. 「Context」セクションの「Add context」→「Local folder」を選択
4. ファイルダイアログでリポジトリフォルダを選択（例: `C:\Users\yourname\Documents\my-project`）
5. 「Add to project」で確定

プロジェクト作成の起点は3つある:
1. 新規作成（ゼロから）
2. 既存クラウドプロジェクトからインポート
3. 既存ローカルフォルダを接続

各プロジェクトには以下を設定できる:
- **指示**（トーン・ルール）
- **コンテキスト**（ローカルフォルダ、チャットプロジェクト、URL）
- **スケジュール済みタスク**
- **メモリ**（セッション間での記憶）

コンテキストの「ローカルフォルダ」でリポジトリフォルダ（例: `C:\Users\yourname\Documents\my-project`）を指定すると、そのプロジェクト内でCoworkのファイルツールがフォルダ以下を直接読み書きできるようになる。

**Windowsでの注意点**: ホームディレクトリ（`C:\Users\<username>`）配下のフォルダのみマウント可能だ。Dドライブなど別ドライブのフォルダは拒否される（既知バグ、[GitHub Issue #29583](https://github.com/anthropics/claude-code/issues/29583)）。シンボリックリンクによる回避も機能しない。ホームディレクトリ外にプロジェクトを置いている場合は移動が必要になる。

マウント後は `content/drafts/`、`logs/cowork/`、CLAUDE.md、`.mcp.json` など、リポジトリ内の全ファイルに直接アクセスできる。ファイルの作成・更新・削除がCoworkから直接行える。

---

## 3. 3つの連携パターン

ローカルマウントを前提に、現在実際に運用している連携パターンが3つある。

### パターン1: CLAUDE.mdのGit共有でエージェント挙動を統一

リポジトリのルートに置いた `CLAUDE.md` は、Claude Codeが自動で読み込む。CoworkのAIも、マウントしたディレクトリ内の `CLAUDE.md` を参照できる。

ただし、Claude CodeはCLAUDE.mdを起動時に自動で読むのに対し、**CoworkはセッションごとにAIへ手動で「CLAUDE.mdを読んで」と指示する必要がある**。この差は現時点では解消できていない。「Code側は自動、Cowork側は手動」と割り切って運用するか、毎回の指示をプロジェクトの「指示」欄に組み込む方法を検討中だ。

CLAUDE.mdをGit管理することで、Code側で更新した内容を次のCoworkセッションで参照できる。両環境のAIがエージェントの役割定義・ルール・ディレクトリ構成を共有できる。

**CLAUDE.mdの構造例**（一部省略）:

```
# CLAUDE.md — プロジェクト名

## エージェント組織
CEO ─ devops / researcher / reviewer / secretary / writer

## ディレクトリ構成
content/      — コンテンツ制作成果物
logs/         — セッションログ
ops/          — 運用ドキュメント
```

### パターン2: MCPを両環境に同一設定してツールを共有

Claude Codeは `.mcp.json` でMCPサーバーを設定する。Cowork側はプロジェクトの設定画面のMCPタブから同じサーバーを追加する（2026年4月時点のUI）。同じPAT（Personal Access Token）を使い、同じMCPサーバー（GitHub MCP、Slack MCP など）を両環境に登録することで、ツールセットを揃えられる。

```json
// .mcp.json（Code側、ファイルパス: プロジェクトルート/.mcp.json）
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "..."
      }
    }
  }
}
```

Cowork側はプロジェクトの設定画面（右上の歯車アイコン）→「MCP」タブを開き、「Add MCP server」から追加する。入力項目はサーバー名（例: `github`）・コマンド（例: `npx`）・引数（例: `-y @modelcontextprotocol/server-github`）・環境変数（`GITHUB_PERSONAL_ACCESS_TOKEN`）で、`.mcp.json` の内容と対応している。これにより、Code側でGitHub Issueを操作するスキルと同等の操作がCowork側でも可能になる。

**注意点**: 前述の通り、CoworkのサンドボックスはWindowsバイナリを実行できない。`npx` 経由のNode.jsベースなど、クロスプラットフォーム対応のサーバーを選ぶこと。

### パターン3: GitHub Issueをハブにしたタスク透過運用

なお、`gh` CLI（GitHub CLI）はCoworkのサンドボックスに標準インストールされていないため、Cowork側はMCPサーバー経由でGitHub APIを操作する。

Code側とCowork側でタスク管理を分けると、どちらが何をやっているかが見えなくなる。これを避けるため、GitHub Issueを「Single Source of Truth」にした。

- **Code側**: `~/.claude/` 配下にデプロイされた `/todo` カスタムスキルでIssueを操作する
- **Cowork側**: GitHub MCPサーバー経由でIssueを直接操作する（AIに「GitHub Issueを作成して」と指示する）
- **共通**: どちらもGitHub Issueを作成・更新・クローズする

この設計により、Code側でIssueを作ったものをCowork側で完了させる、あるいはその逆も、リポジトリを介して双方が把握できる状態になる。

現在の運用では、Code側は専用の `/todo` スキル（Bash + Node.jsで構築）を使っており、Cowork側はGitHub MCPサーバーのAPI経由で同じIssueを操作している。Cowork側にも `.claude/commands/` にカスタムコマンドを置けば `/todo` と入力するだけで呼び出せるようになるが、現時点では未作成のため、AIへの自然言語指示で代用している。実際の指示例: 「ラベルinboxでGitHub Issueを作成して。タイトル: 〇〇、本文: 〇〇」「Issue #12をクローズして」のように伝えると、GitHub MCPサーバー経由で操作される。

将来的に作成する場合のカスタムコマンドの構成例:

```markdown
# /todo コマンド（Cowork用の構成例）

GitHub Issueをタスクとして管理する。

## 使い方
- 「/todo add タスク名」でGitHub Issueを作成する
- 「/todo done #番号」でIssueをクローズする

## 実行方法
GitHub MCPサーバーの `create_issue` / `update_issue` を使う。
ラベルは `inbox` で作成し、仕分け後に `next` / `waiting` に変更する。
```

このファイルを `.claude/commands/todo.md` に置けば、Coworkのチャットで `/todo` と入力するだけでコマンドが呼び出される。ただし現時点では自然言語指示で十分回っているため、優先度は低い。

**「GTD透過運用」とは**: Code環境とCowork環境のどちらから操作しても、タスクの状態（Inbox/Next/Waiting/Done）がGitHub Issue上で一元管理される運用のこと。どちらの環境を使っているかを意識せずにタスクを進められる。

---

## 4. 現時点での限界と未解決課題

正直に書く。

**未解決: CoworkからのGit操作とファイル同期バグ**

プロジェクトにフォルダをマウントするとファイルの読み書きはできる。しかしgit操作（`git add`、`git commit`、`git push`）はCoworkのBash環境から安定して動かない場面がある。

また、コミュニティで複数報告されている既知バグ（[GitHub Issue #30364](https://github.com/anthropics/claude-code/issues/30364)、重複クローズ済み）として、マウント後にホストへ同期されるのが最初の1ファイルのみで、以降の書き込みがホスト側に反映されないことがあるとされている。筆者は現時点でこの挙動を確認していないが、Coworkが書いたファイルをCode側でコミット・プッシュするという分業は、Git操作の安定性という別の理由もあわせて合理的な設計として採用している。

**未解決: MCP設定の同期**

`.mcp.json` を更新してもCowork側のMCP設定は自動で更新されない。Code側で設定変更したら手動でCowork側の設定UIも更新する必要がある。小さな手間だが、設定ズレによるトラブルの原因になりうる。自動同期の仕組みは検討中。

**限界: CoworkのCLIツール**

`gh` コマンドや `npx zenn` などのCLIツールはCoworkのサンドボックスに標準インストールされておらず、そのままでは使えない。Zennの記事プレビューやGitHub CLIを使うタスクはCode側に限定されている。CoworkはMCP経由のAPI操作とファイル読み書きに絞る、という役割分担を最初から意図するほうが設計がシンプルになる。

**コスト面の注意**

CoworkはClaude Proプランに含まれているが、使用量には上限がある（筆者環境では現時点で上限に達していない。正確な上限値はAnthropicのusageページで確認できる）。特にComputer Use（スクリーンショットベースの操作）は消費が大きい。今回紹介したファイル操作・MCP操作中心の使い方は比較的軽量だが、長時間のブラウザ自動化タスクを連続実行する場合は使用量に注意が必要だ。

---

## 5. まとめ

3つのポイントに絞る。

1. **ローカルフォルダのマウントはプロジェクト作成時に設定する**: セッション中のフォルダ追加（`request_cowork_directory`）は2026年2月ごろ削除済みで復活予定なし。2026年3月下旬のProjects機能導入以降、マウントはプロジェクト作成フローで行う。Windowsではホームディレクトリ外のパスは拒否される（[GitHub Issue #29583](https://github.com/anthropics/claude-code/issues/29583)）点に注意。

2. **ルールとタスクの共有はGit経由でできる**: CLAUDE.mdをGit管理することで両環境のAIが同じ文脈で動く。GitHub Issueをタスクのハブにすることで、どちらの環境からでも作業状態が見える。

3. **CoworkとCodeは競合でなく役割分担**: ファイル同期バグ（[Issue #30364](https://github.com/anthropics/claude-code/issues/30364)）の存在も踏まえ、Coworkが書いてCode側がGit管理するという分業が現実的だ。重複させず棲み分けることで、両方の強みが生きる。

2026-04-18時点で8日間この構成で運用しており（2026-04-10導入）、ファイル読み書き・CLAUDE.md参照・MCP経由のGitHub操作は問題なく動いている。ただし前述のGit操作の安定性・ファイル同期バグ・MCP設定同期の3点は未解決のため、長期安定性については引き続き確認中だ。未解決課題が解消したら追記する予定。
