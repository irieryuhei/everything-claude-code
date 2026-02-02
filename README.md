**言語:** [English](docs/en/README.md) | [繁體中文](docs/zh-TW/README.md) | 日本語

# Everything Claude Code

[![Stars](https://img.shields.io/github/stars/affaan-m/everything-claude-code?style=flat)](https://github.com/affaan-m/everything-claude-code/stargazers)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Shell](https://img.shields.io/badge/-Shell-4EAA25?logo=gnu-bash&logoColor=white)
![TypeScript](https://img.shields.io/badge/-TypeScript-3178C6?logo=typescript&logoColor=white)
![Go](https://img.shields.io/badge/-Go-00ADD8?logo=go&logoColor=white)
![Markdown](https://img.shields.io/badge/-Markdown-000000?logo=markdown&logoColor=white)

---

<div align="center">

**🌐 Language / 语言 / 語言**

[**English**](docs/en/README.md) | [简体中文](README.zh-CN.md) | [繁體中文](docs/zh-TW/README.md) | [日本語](README.md)

</div>

---

**Anthropicハッカソン優勝者によるClaude Code設定の完全コレクション。**

10ヶ月以上の集中的な日常使用で進化した、本番環境対応のエージェント、スキル、フック、コマンド、ルール、MCP設定集です。

---

## ガイド

このリポジトリは生のコードのみです。ガイドですべてを説明しています。

<table>
<tr>
<td width="50%">
<a href="https://x.com/affaanmustafa/status/2012378465664745795">
<img src="https://github.com/user-attachments/assets/1a471488-59cc-425b-8345-5245c7efbcef" alt="Everything Claude Code 簡略ガイド" />
</a>
</td>
<td width="50%">
<a href="https://x.com/affaanmustafa/status/2014040193557471352">
<img src="https://github.com/user-attachments/assets/c9ca43bc-b149-427f-b551-af6840c368f0" alt="Everything Claude Code 詳細ガイド" />
</a>
</td>
</tr>
<tr>
<td align="center"><b>簡略ガイド</b><br/>セットアップ、基礎、哲学。<b>まずこちらをお読みください。</b></td>
<td align="center"><b>詳細ガイド</b><br/>トークン最適化、メモリ永続化、評価、並列化。</td>
</tr>
</table>

| トピック | 学べること |
|-------|-------------------|
| トークン最適化 | モデル選択、システムプロンプトの軽量化、バックグラウンドプロセス |
| メモリ永続化 | セッション間でコンテキストを自動的に保存/読み込みするフック |
| 継続学習 | セッションからパターンを自動抽出して再利用可能なスキルに変換 |
| 検証ループ | チェックポイントvs継続評価、評価者タイプ、pass@kメトリクス |
| 並列化 | Gitワークツリー、カスケード方式、インスタンスをスケールするタイミング |
| サブエージェント オーケストレーション | コンテキスト問題、反復的取得パターン |

---

## 🚀 クイックスタート

2分以内でセットアップ完了：

### ステップ1: プラグインのインストール

```bash
# マーケットプレイスを追加
/plugin marketplace add affaan-m/everything-claude-code

# プラグインをインストール
/plugin install everything-claude-code@everything-claude-code
```

### ステップ2: ルールのインストール（必須）

> ⚠️ **重要:** Claude Codeプラグインは`rules`を自動配布できません。手動でインストールしてください：

```bash
# まずリポジトリをクローン
git clone https://github.com/affaan-m/everything-claude-code.git

# ルールをコピー（全プロジェクトに適用）
cp -r everything-claude-code/rules/* ~/.claude/rules/
```

### ステップ3: 使用開始

```bash
# コマンドを試す
/plan "ユーザー認証を追加"

# 利用可能なコマンドを確認
/plugin list everything-claude-code@everything-claude-code
```

✨ **完了です！** 15以上のエージェント、30以上のスキル、20以上のコマンドが利用可能になりました。

---

## 🌐 クロスプラットフォーム対応

このプラグインは**Windows、macOS、Linux**に完全対応しています。すべてのフックとスクリプトは最大限の互換性のためにNode.jsで書き直されています。

### パッケージマネージャーの検出

プラグインは以下の優先順位で推奨パッケージマネージャー（npm、pnpm、yarn、bun）を自動検出します：

1. **環境変数**: `CLAUDE_PACKAGE_MANAGER`
2. **プロジェクト設定**: `.claude/package-manager.json`
3. **package.json**: `packageManager`フィールド
4. **ロックファイル**: package-lock.json、yarn.lock、pnpm-lock.yaml、bun.lockbから検出
5. **グローバル設定**: `~/.claude/package-manager.json`
6. **フォールバック**: 最初に利用可能なパッケージマネージャー

推奨パッケージマネージャーを設定するには：

```bash
# 環境変数で設定
export CLAUDE_PACKAGE_MANAGER=pnpm

# グローバル設定で設定
node scripts/setup-package-manager.js --global pnpm

# プロジェクト設定で設定
node scripts/setup-package-manager.js --project bun

# 現在の設定を検出
node scripts/setup-package-manager.js --detect
```

または、Claude Codeで`/setup-pm`コマンドを使用してください。

---

## 📦 内容物

このリポジトリは**Claude Codeプラグイン**です。直接インストールするか、コンポーネントを手動でコピーできます。

```
everything-claude-code/
|-- .claude-plugin/   # プラグインとマーケットプレイスのマニフェスト
|   |-- plugin.json         # プラグインのメタデータとコンポーネントパス
|   |-- marketplace.json    # /plugin marketplace add用のマーケットプレイスカタログ
|
|-- agents/           # 委譲用の専門サブエージェント
|   |-- planner.md           # 機能実装計画
|   |-- architect.md         # システム設計の決定
|   |-- tdd-guide.md         # テスト駆動開発
|   |-- code-reviewer.md     # 品質とセキュリティレビュー
|   |-- security-reviewer.md # 脆弱性分析
|   |-- build-error-resolver.md
|   |-- e2e-runner.md        # Playwright E2Eテスト
|   |-- refactor-cleaner.md  # デッドコードのクリーンアップ
|   |-- doc-updater.md       # ドキュメント同期
|   |-- go-reviewer.md       # Goコードレビュー（新規）
|   |-- go-build-resolver.md # Goビルドエラー解決（新規）
|
|-- skills/           # ワークフロー定義とドメイン知識
|   |-- coding-standards/           # 言語のベストプラクティス
|   |-- backend-patterns/           # API、データベース、キャッシングパターン
|   |-- frontend-patterns/          # React、Next.jsパターン
|   |-- continuous-learning/        # セッションからパターンを自動抽出（詳細ガイド）
|   |-- continuous-learning-v2/     # 信頼度スコアリング付きインスティンクトベース学習
|   |-- iterative-retrieval/        # サブエージェント用のプログレッシブコンテキスト精製
|   |-- strategic-compact/          # 手動コンパクション提案（詳細ガイド）
|   |-- tdd-workflow/               # TDD方法論
|   |-- security-review/            # セキュリティチェックリスト
|   |-- eval-harness/               # 検証ループ評価（詳細ガイド）
|   |-- verification-loop/          # 継続的検証（詳細ガイド）
|   |-- golang-patterns/            # Goのイディオムとベストプラクティス（新規）
|   |-- golang-testing/             # Goテストパターン、TDD、ベンチマーク（新規）
|
|-- commands/         # クイック実行用のスラッシュコマンド
|   |-- tdd.md              # /tdd - テスト駆動開発
|   |-- plan.md             # /plan - 実装計画
|   |-- e2e.md              # /e2e - E2Eテスト生成
|   |-- code-review.md      # /code-review - 品質レビュー
|   |-- build-fix.md        # /build-fix - ビルドエラー修正
|   |-- refactor-clean.md   # /refactor-clean - デッドコード削除
|   |-- learn.md            # /learn - セッション中にパターンを抽出（詳細ガイド）
|   |-- checkpoint.md       # /checkpoint - 検証状態を保存（詳細ガイド）
|   |-- verify.md           # /verify - 検証ループを実行（詳細ガイド）
|   |-- setup-pm.md         # /setup-pm - パッケージマネージャー設定
|   |-- go-review.md        # /go-review - Goコードレビュー（新規）
|   |-- go-test.md          # /go-test - Go TDDワークフロー（新規）
|   |-- go-build.md         # /go-build - Goビルドエラー修正（新規）
|   |-- skill-create.md     # /skill-create - git履歴からスキル生成（新規）
|   |-- instinct-status.md  # /instinct-status - 学習したインスティンクトを表示（新規）
|   |-- instinct-import.md  # /instinct-import - インスティンクトをインポート（新規）
|   |-- instinct-export.md  # /instinct-export - インスティンクトをエクスポート（新規）
|   |-- evolve.md           # /evolve - インスティンクトをスキルにクラスタリング（新規）
|
|-- rules/            # 常に従うガイドライン（~/.claude/rules/にコピー）
|   |-- security.md         # 必須セキュリティチェック
|   |-- coding-style.md     # イミュータビリティ、ファイル構成
|   |-- testing.md          # TDD、80%カバレッジ要件
|   |-- git-workflow.md     # コミット形式、PRプロセス
|   |-- agents.md           # サブエージェントに委譲するタイミング
|   |-- performance.md      # モデル選択、コンテキスト管理
|
|-- hooks/            # トリガーベースの自動化
|   |-- hooks.json                # 全フック設定（PreToolUse、PostToolUse、Stopなど）
|   |-- memory-persistence/       # セッションライフサイクルフック（詳細ガイド）
|   |-- strategic-compact/        # コンパクション提案（詳細ガイド）
|
|-- scripts/          # クロスプラットフォームNode.jsスクリプト（新規）
|   |-- lib/                     # 共有ユーティリティ
|   |   |-- utils.js             # クロスプラットフォームファイル/パス/システムユーティリティ
|   |   |-- package-manager.js   # パッケージマネージャー検出と選択
|   |-- hooks/                   # フック実装
|   |   |-- session-start.js     # セッション開始時にコンテキストを読み込み
|   |   |-- session-end.js       # セッション終了時に状態を保存
|   |   |-- pre-compact.js       # コンパクション前の状態保存
|   |   |-- suggest-compact.js   # 戦略的コンパクション提案
|   |   |-- evaluate-session.js  # セッションからパターンを抽出
|   |-- setup-package-manager.js # 対話式PM設定
|
|-- tests/            # テストスイート（新規）
|   |-- lib/                     # ライブラリテスト
|   |-- hooks/                   # フックテスト
|   |-- run-all.js               # 全テスト実行
|
|-- contexts/         # 動的システムプロンプト注入コンテキスト（詳細ガイド）
|   |-- dev.md              # 開発モードコンテキスト
|   |-- review.md           # コードレビューモードコンテキスト
|   |-- research.md         # リサーチ/探索モードコンテキスト
|
|-- examples/         # 設定例とセッション
|   |-- CLAUDE.md           # プロジェクトレベル設定例
|   |-- user-CLAUDE.md      # ユーザーレベル設定例
|
|-- mcp-configs/      # MCPサーバー設定
|   |-- mcp-servers.json    # GitHub、Supabase、Vercel、Railwayなど
|
|-- marketplace.json  # セルフホスト型マーケットプレイス設定（/plugin marketplace add用）
```

---

## 🛠️ エコシステムツール

### スキルクリエイター

リポジトリからClaude Codeスキルを生成する2つの方法：

#### オプションA: ローカル分析（組み込み）

外部サービスなしでローカル分析を行う`/skill-create`コマンドを使用：

```bash
/skill-create                    # 現在のリポジトリを分析
/skill-create --instincts        # 継続学習用のインスティンクトも生成
```

これはgit履歴をローカルで分析し、SKILL.mdファイルを生成します。

#### オプションB: GitHub App（高度）

高度な機能（10k以上のコミット、自動PR、チーム共有）用：

[GitHub Appをインストール](https://github.com/apps/skill-creator) | [ecc.tools](https://ecc.tools)

```bash
# 任意のissueでコメント：
/skill-creator analyze

# またはデフォルトブランチへのpushで自動トリガー
```

どちらのオプションでも作成されるもの：
- **SKILL.mdファイル** - Claude Code用のすぐに使えるスキル
- **インスティンクトコレクション** - continuous-learning-v2用
- **パターン抽出** - コミット履歴から学習

### 🧠 継続学習 v2

インスティンクトベースの学習システムが自動的にパターンを学習：

```bash
/instinct-status        # 信頼度付きで学習したインスティンクトを表示
/instinct-import <file> # 他者からインスティンクトをインポート
/instinct-export        # 共有用にインスティンクトをエクスポート
/evolve                 # 関連するインスティンクトをスキルにクラスタリング
```

完全なドキュメントは`skills/continuous-learning-v2/`を参照してください。

---

## 📋 要件

### Claude Code CLIバージョン

**最小バージョン: v2.1.0以降**

このプラグインは、プラグインシステムがフックを処理する方法の変更により、Claude Code CLI v2.1.0以降が必要です。

バージョンを確認：
```bash
claude --version
```

### 重要: フックの自動読み込み動作

> ⚠️ **コントリビューター向け:** `.claude-plugin/plugin.json`に`"hooks"`フィールドを追加しないでください。これは回帰テストで強制されています。

Claude Code v2.1以降は、慣例により、インストールされたプラグインから`hooks/hooks.json`を**自動的に読み込みます**。これを明示的に`plugin.json`で宣言すると、重複検出エラーが発生します：

```
Duplicate hooks file detected: ./hooks/hooks.json resolves to already-loaded file
```

**履歴:** これにより、このリポジトリで修正/リバートサイクルが繰り返されました（[#29](https://github.com/affaan-m/everything-claude-code/issues/29)、[#52](https://github.com/affaan-m/everything-claude-code/issues/52)、[#103](https://github.com/affaan-m/everything-claude-code/issues/103)）。動作がClaude Codeのバージョン間で変更され、混乱を招きました。現在は、これが再導入されるのを防ぐための回帰テストがあります。

---

## 📥 インストール

### オプション1: プラグインとしてインストール（推奨）

このリポジトリを使用する最も簡単な方法 - Claude Codeプラグインとしてインストール：

```bash
# このリポジトリをマーケットプレイスとして追加
/plugin marketplace add affaan-m/everything-claude-code

# プラグインをインストール
/plugin install everything-claude-code@everything-claude-code
```

または、`~/.claude/settings.json`に直接追加：

```json
{
  "extraKnownMarketplaces": {
    "everything-claude-code": {
      "source": {
        "source": "github",
        "repo": "affaan-m/everything-claude-code"
      }
    }
  },
  "enabledPlugins": {
    "everything-claude-code@everything-claude-code": true
  }
}
```

これにより、すべてのコマンド、エージェント、スキル、フックに即座にアクセスできます。

> **注意:** Claude Codeプラグインシステムは、プラグイン経由での`rules`の配布をサポートしていません（[上流の制限](https://code.claude.com/docs/en/plugins-reference)）。ルールを手動でインストールする必要があります：
>
> ```bash
> # まずリポジトリをクローン
> git clone https://github.com/affaan-m/everything-claude-code.git
>
> # オプションA: ユーザーレベルのルール（全プロジェクトに適用）
> cp -r everything-claude-code/rules/* ~/.claude/rules/
>
> # オプションB: プロジェクトレベルのルール（現在のプロジェクトのみに適用）
> mkdir -p .claude/rules
> cp -r everything-claude-code/rules/* .claude/rules/
> ```

---

### 🔧 オプション2: 手動インストール

インストールするものを手動で制御したい場合：

```bash
# リポジトリをクローン
git clone https://github.com/affaan-m/everything-claude-code.git

# エージェントをClaude設定にコピー
cp everything-claude-code/agents/*.md ~/.claude/agents/

# ルールをコピー
cp everything-claude-code/rules/*.md ~/.claude/rules/

# コマンドをコピー
cp everything-claude-code/commands/*.md ~/.claude/commands/

# スキルをコピー
cp -r everything-claude-code/skills/* ~/.claude/skills/
```

#### settings.jsonにフックを追加

`hooks/hooks.json`のフックを`~/.claude/settings.json`にコピーしてください。

#### MCPを設定

`mcp-configs/mcp-servers.json`から必要なMCPサーバーを`~/.claude.json`にコピーしてください。

**重要:** `YOUR_*_HERE`プレースホルダーを実際のAPIキーに置き換えてください。

---

## 🎯 主要コンセプト

### エージェント

サブエージェントは限定されたスコープで委譲されたタスクを処理します。例：

```markdown
---
name: code-reviewer
description: コードの品質、セキュリティ、保守性をレビュー
tools: ["Read", "Grep", "Glob", "Bash"]
model: opus
---

あなたはシニアコードレビュアーです...
```

### スキル

スキルはコマンドやエージェントによって呼び出されるワークフロー定義です：

```markdown
# TDDワークフロー

1. まずインターフェースを定義
2. 失敗するテストを書く（RED）
3. 最小限のコードを実装（GREEN）
4. リファクタリング（IMPROVE）
5. 80%以上のカバレッジを検証
```

### フック

フックはツールイベントで発火します。例 - console.logについて警告：

```json
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.(ts|tsx|js|jsx)$\"",
  "hooks": [{
    "type": "command",
    "command": "#!/bin/bash\ngrep -n 'console\\.log' \"$file_path\" && echo '[Hook] console.logを削除してください' >&2"
  }]
}
```

### ルール

ルールは常に従うガイドラインです。モジュール化して保持：

```
~/.claude/rules/
  security.md      # ハードコードされた秘密情報禁止
  coding-style.md  # イミュータビリティ、ファイル制限
  testing.md       # TDD、カバレッジ要件
```

---

## 🧪 テストの実行

プラグインには包括的なテストスイートが含まれています：

```bash
# 全テストを実行
node tests/run-all.js

# 個別のテストファイルを実行
node tests/lib/utils.test.js
node tests/lib/package-manager.test.js
node tests/hooks/hooks.test.js
```

---

## 🤝 コントリビューション

**コントリビューションを歓迎し、奨励します。**

このリポジトリはコミュニティリソースとして意図されています。以下をお持ちの場合：
- 便利なエージェントやスキル
- 巧妙なフック
- より良いMCP設定
- 改善されたルール

ぜひコントリビューションしてください！ガイドラインは[CONTRIBUTING.md](CONTRIBUTING.md)を参照してください。

### コントリビューションのアイデア

- 言語固有のスキル（Python、Rustパターン）- Goは既に含まれています！
- フレームワーク固有の設定（Django、Rails、Laravel）
- DevOpsエージェント（Kubernetes、Terraform、AWS）
- テスト戦略（様々なフレームワーク）
- ドメイン固有の知識（ML、データエンジニアリング、モバイル）

---

## 📖 背景

私は実験的ロールアウトからClaude Codeを使用しています。2025年9月のAnthropic x Forum Venturesハッカソンで[@DRodriguezFX](https://x.com/DRodriguezFX)と共に[zenith.chat](https://zenith.chat)を構築して優勝しました - すべてClaude Codeを使用して。

これらの設定は複数の本番アプリケーションで実戦テスト済みです。

---

## ⚠️ 重要な注意事項

### コンテキストウィンドウ管理

**重要:** すべてのMCPを一度に有効にしないでください。200kのコンテキストウィンドウが、有効化されたツールが多すぎると70kに縮小する可能性があります。

目安：
- 20-30のMCPを設定
- プロジェクトごとに10未満を有効化
- 80未満のツールをアクティブに

プロジェクト設定で`disabledMcpServers`を使用して未使用のものを無効化してください。

### カスタマイズ

これらの設定は私のワークフロー用です。あなたは：
1. 共感するものから始める
2. あなたのスタックに合わせて修正
3. 使わないものを削除
4. 独自のパターンを追加

---

## 🌟 スター履歴

[![Star History Chart](https://api.star-history.com/svg?repos=affaan-m/everything-claude-code&type=Date)](https://star-history.com/#affaan-m/everything-claude-code&Date)

---

## 🔗 リンク

- **簡略ガイド（まずこちら）:** [Everything Claude Code 簡略ガイド](https://x.com/affaanmustafa/status/2012378465664745795)
- **詳細ガイド（上級）:** [Everything Claude Code 詳細ガイド](https://x.com/affaanmustafa/status/2014040193557471352)
- **フォロー:** [@affaanmustafa](https://x.com/affaanmustafa)
- **zenith.chat:** [zenith.chat](https://zenith.chat)

---

## 📄 ライセンス

MIT - 自由に使用、必要に応じて修正、可能であれば貢献を返してください。

---

**このリポジトリが役立ったらスターを。両方のガイドを読んでください。素晴らしいものを作りましょう。**
