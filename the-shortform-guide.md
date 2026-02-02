# Everything Claude Code 簡易ガイド

![Header: Anthropic Hackathon Winner - Tips & Tricks for Claude Code](./assets/images/shortform/00-header.png)

---

**2月の実験的ロールアウト以来、熱心なClaude Codeユーザーであり、[@DRodriguezFX](https://x.com/DRodriguezFX)とともに[zenith.chat](https://zenith.chat)でAnthropic x Forum Venturesハッカソンで優勝しました - 完全にClaude Codeを使用して。**

10ヶ月以上の日常使用後の私の完全なセットアップ：スキル、フック、サブエージェント、MCP、プラグイン、そして実際に機能するもの。

---

## スキルとコマンド

スキルは特定のスコープとワークフローに制限されたルールのように動作します。特定のワークフローを実行する必要があるときのプロンプトの省略形です。

Opus 4.5での長いコーディングセッションの後、デッドコードと散らばった.mdファイルをクリーンアップしたいですか？`/refactor-clean`を実行します。テストが必要ですか？`/tdd`、`/e2e`、`/test-coverage`。スキルにはコードマップも含めることができます - Claudeがコンテキストを探索に消費することなく、コードベースをすばやくナビゲートする方法です。

![Terminal showing chained commands](./assets/images/shortform/02-chaining-commands.jpeg)
*コマンドを連鎖させる*

コマンドはスラッシュコマンドで実行されるスキルです。重複していますが、保存場所が異なります：

- **スキル**: `~/.claude/skills/` - より広いワークフロー定義
- **コマンド**: `~/.claude/commands/` - 素早く実行可能なプロンプト

```bash
# スキル構造の例
~/.claude/skills/
  pmx-guidelines.md      # プロジェクト固有のパターン
  coding-standards.md    # 言語のベストプラクティス
  tdd-workflow/          # README.mdを持つマルチファイルスキル
  security-review/       # チェックリストベースのスキル
```

---

## フック

フックは特定のイベントでトリガーされるベースの自動化です。スキルとは異なり、ツール呼び出しとライフサイクルイベントに制限されています。

**フックタイプ：**

1. **PreToolUse** - ツール実行前（検証、リマインダー）
2. **PostToolUse** - ツール終了後（フォーマット、フィードバックループ）
3. **UserPromptSubmit** - メッセージを送信するとき
4. **Stop** - Claudeが応答を終了するとき
5. **PreCompact** - コンテキスト圧縮前
6. **Notification** - パーミッションリクエスト

**例：長時間実行コマンド前のtmuxリマインダー**

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && tool_input.command matches \"(npm|pnpm|yarn|cargo|pytest)\"",
      "hooks": [
        {
          "type": "command",
          "command": "if [ -z \"$TMUX\" ]; then echo '[Hook] Consider tmux for session persistence' >&2; fi"
        }
      ]
    }
  ]
}
```

![PostToolUse hook feedback](./assets/images/shortform/03-posttooluse-hook.png)
*PostToolUseフック実行中にClaude Codeで得られるフィードバックの例*

**プロのヒント：** JSONを手動で書く代わりに、`hookify`プラグインを使用して会話形式でフックを作成します。`/hookify`を実行して、何が欲しいか説明します。

---

## サブエージェント

サブエージェントは、オーケストレーター（メインのClaude）が限られたスコープでタスクを委譲できるプロセスです。バックグラウンドまたはフォアグラウンドで実行でき、メインエージェントのコンテキストを解放します。

サブエージェントはスキルとうまく連携します - スキルのサブセットを実行できるサブエージェントには、タスクを委譲でき、それらのスキルを自律的に使用できます。また、特定のツールパーミッションでサンドボックス化することもできます。

```bash
# サブエージェント構造の例
~/.claude/agents/
  planner.md           # 機能実装の計画
  architect.md         # システム設計の決定
  tdd-guide.md         # テスト駆動開発
  code-reviewer.md     # 品質/セキュリティレビュー
  security-reviewer.md # 脆弱性分析
  build-error-resolver.md
  e2e-runner.md
  refactor-cleaner.md
```

適切なスコーピングのために、サブエージェントごとに許可されたツール、MCP、パーミッションを設定します。

---

## ルールとメモリ

`.rules`フォルダには、Claudeが常に従うべきベストプラクティスを含む`.md`ファイルが格納されています。2つのアプローチ：

1. **単一のCLAUDE.md** - すべてを1つのファイルに（ユーザーまたはプロジェクトレベル）
2. **ルールフォルダ** - 関心事ごとにグループ化されたモジュラーな`.md`ファイル

```bash
~/.claude/rules/
  security.md      # ハードコードされたシークレット禁止、入力検証
  coding-style.md  # 不変性、ファイル構成
  testing.md       # TDDワークフロー、80%カバレッジ
  git-workflow.md  # コミットフォーマット、PRプロセス
  agents.md        # いつサブエージェントに委譲するか
  performance.md   # モデル選択、コンテキスト管理
```

**ルールの例：**

- コードベースに絵文字を使用しない
- フロントエンドで紫色の色調を避ける
- デプロイ前に常にコードをテスト
- メガファイルよりモジュラーコードを優先
- console.logをコミットしない

---

## MCP（Model Context Protocol）

MCPはClaudeを外部サービスに直接接続します。APIの代替ではなく、より柔軟に情報をナビゲートできるプロンプト駆動のラッパーです。

**例：** Supabase MCPを使用すると、Claudeは特定のデータを取得し、コピペなしでアップストリームで直接SQLを実行できます。データベース、デプロイメントプラットフォームなども同様です。

![Supabase MCP listing tables](./assets/images/shortform/04-supabase-mcp.jpeg)
*パブリックスキーマ内のテーブルをリストするSupabase MCPの例*

**Chrome in Claude：** Claudeがブラウザを自律的に制御できる組み込みプラグインMCPです - クリックして動作を確認します。

**重要：コンテキストウィンドウ管理**

MCPは慎重に選択してください。すべてのMCPをユーザー設定に保持しますが、**未使用のものはすべて無効にします**。`/plugins`にナビゲートして下にスクロールするか、`/mcp`を実行します。

![/plugins interface](./assets/images/shortform/05-plugins-interface.jpeg)
*/pluginsを使用してMCPにナビゲートし、現在インストールされているものとそのステータスを確認*

圧縮前の200kコンテキストウィンドウは、有効になっているツールが多すぎると70kしかないかもしれません。パフォーマンスは大幅に低下します。

**経験則：** 設定に20-30のMCPを持ち、10未満を有効化/80未満のツールをアクティブに保つ。

```bash
# 有効なMCPを確認
/mcp

# ~/.claude.jsonのprojects.disabledMcpServersで未使用のものを無効化
```

---

## プラグイン

プラグインは、面倒な手動セットアップの代わりに簡単にインストールできるようにツールをパッケージ化します。プラグインはスキル + MCPの組み合わせ、またはフック/ツールのバンドルにすることができます。

**プラグインのインストール：**

```bash
# マーケットプレイスを追加
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Claudeを開き、/pluginsを実行し、新しいマーケットプレイスを見つけ、そこからインストール
```

![Marketplaces tab showing mgrep](./assets/images/shortform/06-marketplaces-mgrep.jpeg)
*新しくインストールされたMixedbread-Grepマーケットプレイスの表示*

**LSPプラグイン**は、エディタ外でClaude Codeを頻繁に実行する場合に特に便利です。Language Server Protocolにより、Claudeはリアルタイムの型チェック、定義への移動、IDEを開かなくてもインテリジェントな補完ができます。

```bash
# 有効なプラグインの例
typescript-lsp@claude-plugins-official  # TypeScriptインテリジェンス
pyright-lsp@claude-plugins-official     # Python型チェック
hookify@claude-plugins-official         # 会話形式でフック作成
mgrep@Mixedbread-Grep                   # ripgrepより優れた検索
```

MCPと同じ警告 - コンテキストウィンドウに注意。

---

## ヒントとコツ

### キーボードショートカット

- `Ctrl+U` - 行全体を削除（バックスペース連打より速い）
- `!` - クイックbashコマンドプレフィックス
- `@` - ファイルを検索
- `/` - スラッシュコマンドを開始
- `Shift+Enter` - 複数行入力
- `Tab` - 思考表示の切り替え
- `Esc Esc` - Claudeを中断/コードを復元

### 並列ワークフロー

- **フォーク** (`/fork`) - キューに入れられたメッセージをスパムする代わりに、会話をフォークして重複しないタスクを並列で実行
- **Gitワークツリー** - 競合なしで重複する並列Claudeのために。各ワークツリーは独立したチェックアウト

```bash
git worktree add ../feature-branch feature-branch
# 各ワークツリーで別々のClaudeインスタンスを実行
```

### 長時間実行コマンド用のtmux

Claudeが実行するログ/bashプロセスをストリームして監視：

https://github.com/user-attachments/assets/shortform/07-tmux-video.mp4

```bash
tmux new -s dev
# Claudeはここでコマンドを実行し、デタッチして再アタッチできます
tmux attach -t dev
```

### mgrep > grep

`mgrep`はripgrep/grepからの大幅な改善です。プラグインマーケットプレイス経由でインストールし、`/mgrep`スキルを使用します。ローカル検索とWeb検索の両方で動作します。

```bash
mgrep "function handleSubmit"  # ローカル検索
mgrep --web "Next.js 15 app router changes"  # Web検索
```

### その他の便利なコマンド

- `/rewind` - 以前の状態に戻る
- `/statusline` - ブランチ、コンテキスト%、todoでカスタマイズ
- `/checkpoints` - ファイルレベルのアンドゥポイント
- `/compact` - 手動でコンテキスト圧縮をトリガー

### GitHub Actions CI/CD

GitHub ActionsでPRのコードレビューを設定します。設定すると、Claudeは自動的にPRをレビューできます。

![Claude bot approving a PR](./assets/images/shortform/08-github-pr-review.jpeg)
*バグ修正PRを承認するClaude*

### サンドボックス

リスクのある操作にはサンドボックスモードを使用 - Claudeは実際のシステムに影響を与えない制限された環境で実行されます。

---

## エディタについて

エディタの選択はClaude Codeのワークフローに大きく影響します。Claude Codeはどのターミナルからでも動作しますが、優れたエディタと組み合わせることで、リアルタイムのファイル追跡、クイックナビゲーション、統合されたコマンド実行が可能になります。

### Zed（私の好み）

私は[Zed](https://zed.dev)を使用しています - Rustで書かれているので、本当に高速です。即座に開き、大規模なコードベースを難なく処理し、システムリソースをほとんど消費しません。

**Zed + Claude Codeが優れた組み合わせである理由：**

- **スピード** - Rustベースのパフォーマンスは、Claudeがファイルを高速に編集する際にラグがないことを意味します。エディタがついていきます
- **エージェントパネル統合** - ZedのClaude統合により、Claudeが編集するファイルの変更をリアルタイムで追跡できます。エディタを離れずにClaudeが参照するファイル間をジャンプ
- **CMD+Shift+Rコマンドパレット** - すべてのカスタムスラッシュコマンド、デバッガ、ビルドスクリプトに検索可能なUIでクイックアクセス
- **最小限のリソース使用** - 重い操作中にClaudeとRAM/CPUを競合しません。Opusを実行する際に重要
- **Vimモード** - お好みならフルvimキーバインディング

![Zed Editor with custom commands](./assets/images/shortform/09-zed-editor.jpeg)
*CMD+Shift+Rを使用したカスタムコマンドドロップダウン付きのZedエディタ。右下のブルズアイとしてフォローモードが表示されています。*

**エディタに依存しないヒント：**

1. **画面を分割** - 片側にClaude Codeのターミナル、もう片側にエディタ
2. **Ctrl + G** - Zedで現在Claudeが作業しているファイルをすばやく開く
3. **自動保存** - Claudeのファイル読み取りが常に最新になるように自動保存を有効化
4. **Git統合** - コミット前にエディタのgit機能を使用してClaudeの変更をレビュー
5. **ファイルウォッチャー** - ほとんどのエディタは変更されたファイルを自動リロードします。これが有効になっていることを確認

### VSCode / Cursor

これも実行可能な選択肢で、Claude Codeとうまく連携します。ターミナル形式で使用でき、`\ide`を使用してエディタと自動同期し、LSP機能を有効にします（プラグインで多少冗長になりました）。または、エディタとより統合され、マッチするUIを持つ拡張機能を選択できます。

![VS Code Claude Code Extension](./assets/images/shortform/10-vscode-extension.jpeg)
*VS Code拡張機能はClaude Codeのネイティブグラフィカルインターフェースを提供し、IDEに直接統合されます。*

---

## 私のセットアップ

### プラグイン

**インストール済み：**（通常、一度に4-5個だけ有効にしています）

```markdown
ralph-wiggum@claude-code-plugins       # ループ自動化
frontend-design@claude-code-plugins    # UI/UXパターン
commit-commands@claude-code-plugins    # Gitワークフロー
security-guidance@claude-code-plugins  # セキュリティチェック
pr-review-toolkit@claude-code-plugins  # PR自動化
typescript-lsp@claude-plugins-official # TSインテリジェンス
hookify@claude-plugins-official        # フック作成
code-simplifier@claude-plugins-official
feature-dev@claude-code-plugins
explanatory-output-style@claude-code-plugins
code-review@claude-code-plugins
context7@claude-plugins-official       # ライブドキュメント
pyright-lsp@claude-plugins-official    # Python型
mgrep@Mixedbread-Grep                  # より良い検索
```

### MCPサーバー

**設定済み（ユーザーレベル）：**

```json
{
  "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
  "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest", "--project-ref=YOUR_REF"]
  },
  "memory": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
  "sequential-thinking": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
  },
  "vercel": { "type": "http", "url": "https://mcp.vercel.com" },
  "railway": { "command": "npx", "args": ["-y", "@railway/mcp-server"] },
  "cloudflare-docs": { "type": "http", "url": "https://docs.mcp.cloudflare.com/mcp" },
  "cloudflare-workers-bindings": {
    "type": "http",
    "url": "https://bindings.mcp.cloudflare.com/mcp"
  },
  "clickhouse": { "type": "http", "url": "https://mcp.clickhouse.cloud/mcp" },
  "AbletonMCP": { "command": "uvx", "args": ["ableton-mcp"] },
  "magic": { "command": "npx", "args": ["-y", "@magicuidesign/mcp@latest"] }
}
```

これがキーです - 14のMCPを設定していますが、プロジェクトごとに約5-6個だけを有効にしています。コンテキストウィンドウを健全に保ちます。

### 主要なフック

```json
{
  "PreToolUse": [
    { "matcher": "npm|pnpm|yarn|cargo|pytest", "hooks": ["tmuxリマインダー"] },
    { "matcher": "Write && .mdファイル", "hooks": ["README/CLAUDE以外はブロック"] },
    { "matcher": "git push", "hooks": ["レビュー用にエディタを開く"] }
  ],
  "PostToolUse": [
    { "matcher": "Edit && .ts/.tsx/.js/.jsx", "hooks": ["prettier --write"] },
    { "matcher": "Edit && .ts/.tsx", "hooks": ["tsc --noEmit"] },
    { "matcher": "Edit", "hooks": ["console.log警告をgrep"] }
  ],
  "Stop": [
    { "matcher": "*", "hooks": ["変更されたファイルでconsole.logをチェック"] }
  ]
}
```

### カスタムステータスライン

ユーザー、ディレクトリ、ダーティインジケーター付きのgitブランチ、残りコンテキスト%、モデル、時間、todoカウントを表示：

![Custom status line](./assets/images/shortform/11-statusline.jpeg)
*Macルートディレクトリでのステータスラインの例*

```
affoon:~ ctx:65% Opus 4.5 19:52
▌▌ プランモードオン（shift+tabでサイクル）
```

### ルール構造

```
~/.claude/rules/
  security.md      # 必須のセキュリティチェック
  coding-style.md  # 不変性、ファイルサイズ制限
  testing.md       # TDD、80%カバレッジ
  git-workflow.md  # 従来型コミット
  agents.md        # サブエージェント委譲ルール
  patterns.md      # APIレスポンスフォーマット
  performance.md   # モデル選択（Haiku vs Sonnet vs Opus）
  hooks.md         # フックドキュメント
```

### サブエージェント

```
~/.claude/agents/
  planner.md           # 機能を分解
  architect.md         # システム設計
  tdd-guide.md         # テストを先に書く
  code-reviewer.md     # 品質レビュー
  security-reviewer.md # 脆弱性スキャン
  build-error-resolver.md
  e2e-runner.md        # Playwrightテスト
  refactor-cleaner.md  # デッドコード削除
  doc-updater.md       # ドキュメントを同期に保つ
```

---

## 重要なポイント

1. **複雑にしすぎない** - 設定はアーキテクチャではなく、ファインチューニングとして扱う
2. **コンテキストウィンドウは貴重** - 未使用のMCPとプラグインを無効化
3. **並列実行** - 会話をフォーク、gitワークツリーを使用
4. **繰り返しを自動化** - フォーマット、lint、リマインダー用のフック
5. **サブエージェントをスコープ化** - 限られたツール = 集中した実行

---

## 参考文献

- [プラグインリファレンス](https://code.claude.com/docs/en/plugins-reference)
- [フックドキュメント](https://code.claude.com/docs/en/hooks)
- [チェックポイント](https://code.claude.com/docs/en/checkpointing)
- [インタラクティブモード](https://code.claude.com/docs/en/interactive-mode)
- [メモリシステム](https://code.claude.com/docs/en/memory)
- [サブエージェント](https://code.claude.com/docs/en/sub-agents)
- [MCP概要](https://code.claude.com/docs/en/mcp-overview)

---

**注意：** これは詳細のサブセットです。高度なパターンについては[完全ガイド](./the-longform-guide.md)を参照してください。

---

*NYC で[@DRodriguezFX](https://x.com/DRodriguezFX)とともに[zenith.chat](https://zenith.chat)を構築してAnthropic x Forum Venturesハッカソンで優勝*
