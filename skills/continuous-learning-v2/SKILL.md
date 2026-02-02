---
name: continuous-learning-v2
description: フックを通じてセッションを監視し、信頼度スコアリングを持つアトミックなインスティンクトを作成し、それらをスキル/コマンド/エージェントに進化させるインスティンクトベースの学習システム。
version: 2.0.0
---

# 継続的学習 v2 - インスティンクトベースアーキテクチャ

Claude Codeセッションをアトミックな「インスティンクト」- 信頼度スコアリングを持つ小さな学習済み行動 - を通じて再利用可能な知識に変換する高度な学習システム。

## v2の新機能

| 機能 | v1 | v2 |
|------|----|----|
| 監視 | Stopフック（セッション終了時） | PreToolUse/PostToolUse（100%信頼性） |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） |
| 粒度 | 完全なスキル | アトミックな「インスティンクト」 |
| 信頼度 | なし | 0.3-0.9の重み付け |
| 進化 | スキルに直接 | インスティンクト → クラスタ → スキル/コマンド/エージェント |
| 共有 | なし | インスティンクトのエクスポート/インポート |

## インスティンクトモデル

インスティンクトは小さな学習済み行動です：

```yaml
---
id: prefer-functional-style
trigger: "新しい関数を書くとき"
confidence: 0.7
domain: "code-style"
source: "session-observation"
---

# 関数型スタイルを優先

## アクション
適切な場合、クラスよりも関数型パターンを使用する。

## 証拠
- 関数型パターン優先の5つのインスタンスを観察
- 2025-01-15にユーザーがクラスベースのアプローチを関数型に修正
```

**プロパティ:**
- **アトミック** — 1つのトリガー、1つのアクション
- **信頼度加重** — 0.3 = 仮、0.9 = ほぼ確実
- **ドメインタグ付き** — code-style、testing、git、debugging、workflow など
- **証拠裏付け** — 何が作成したかを追跡

## 仕組み

```
セッションアクティビティ
      │
      │ フックがプロンプト + ツール使用をキャプチャ（100%信頼性）
      ▼
┌─────────────────────────────────────────┐
│         observations.jsonl              │
│   （プロンプト、ツール呼び出し、結果）    │
└─────────────────────────────────────────┘
      │
      │ オブザーバーエージェントが読み取り（バックグラウンド、Haiku）
      ▼
┌─────────────────────────────────────────┐
│          パターン検出                    │
│   • ユーザー修正 → インスティンクト       │
│   • エラー解決 → インスティンクト         │
│   • 繰り返しワークフロー → インスティンクト│
└─────────────────────────────────────────┘
      │
      │ 作成/更新
      ▼
┌─────────────────────────────────────────┐
│         instincts/personal/             │
│   • prefer-functional.md (0.7)          │
│   • always-test-first.md (0.9)          │
│   • use-zod-validation.md (0.6)         │
└─────────────────────────────────────────┘
      │
      │ /evolve がクラスタ化
      ▼
┌─────────────────────────────────────────┐
│              evolved/                   │
│   • commands/new-feature.md             │
│   • skills/testing-workflow.md          │
│   • agents/refactor-specialist.md       │
└─────────────────────────────────────────┘
```

## クイックスタート

### 1. 監視フックの有効化

`~/.claude/settings.json`に追加してください。

**プラグインとしてインストールした場合**（推奨）：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/hooks/observe.sh pre"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/hooks/observe.sh post"
      }]
    }]
  }
}
```

**手動で`~/.claude/skills`にインストールした場合**：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh pre"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh post"
      }]
    }]
  }
}
```

### 2. ディレクトリ構造の初期化

Python CLIが自動的に作成しますが、手動でも作成できます：

```bash
mkdir -p ~/.claude/homunculus/{instincts/{personal,inherited},evolved/{agents,skills,commands}}
touch ~/.claude/homunculus/observations.jsonl
```

### 3. インスティンクトコマンドの使用

```bash
/instinct-status     # 学習したインスティンクトと信頼度スコアを表示
/evolve              # 関連するインスティンクトをスキル/コマンドにクラスタ化
/instinct-export     # 共有用にインスティンクトをエクスポート
/instinct-import     # 他者からインスティンクトをインポート
```

## コマンド

| コマンド | 説明 |
|---------|------|
| `/instinct-status` | 学習したすべてのインスティンクトと信頼度を表示 |
| `/evolve` | 関連するインスティンクトをスキル/コマンドにクラスタ化 |
| `/instinct-export` | 共有用にインスティンクトをエクスポート |
| `/instinct-import <file>` | 他者からインスティンクトをインポート |

## 設定

`config.json`を編集：

```json
{
  "version": "2.0",
  "observation": {
    "enabled": true,
    "store_path": "~/.claude/homunculus/observations.jsonl",
    "max_file_size_mb": 10,
    "archive_after_days": 7
  },
  "instincts": {
    "personal_path": "~/.claude/homunculus/instincts/personal/",
    "inherited_path": "~/.claude/homunculus/instincts/inherited/",
    "min_confidence": 0.3,
    "auto_approve_threshold": 0.7,
    "confidence_decay_rate": 0.05
  },
  "observer": {
    "enabled": true,
    "model": "haiku",
    "run_interval_minutes": 5,
    "patterns_to_detect": [
      "user_corrections",
      "error_resolutions",
      "repeated_workflows",
      "tool_preferences"
    ]
  },
  "evolution": {
    "cluster_threshold": 3,
    "evolved_path": "~/.claude/homunculus/evolved/"
  }
}
```

## ファイル構造

```
~/.claude/homunculus/
├── identity.json           # プロフィール、技術レベル
├── observations.jsonl      # 現在のセッション観察
├── observations.archive/   # 処理済み観察
├── instincts/
│   ├── personal/           # 自動学習インスティンクト
│   └── inherited/          # 他者からインポート
└── evolved/
    ├── agents/             # 生成された専門エージェント
    ├── skills/             # 生成されたスキル
    └── commands/           # 生成されたコマンド
```

## Skill Creatorとの統合

[Skill Creator GitHub App](https://skill-creator.app)を使用すると、**両方**が生成されます：
- 従来のSKILL.mdファイル（後方互換性のため）
- インスティンクトコレクション（v2学習システム用）

リポジトリ分析からのインスティンクトは`source: "repo-analysis"`を持ち、ソースリポジトリURLを含みます。

## 信頼度スコアリング

信頼度は時間とともに進化します：

| スコア | 意味 | 動作 |
|--------|------|------|
| 0.3 | 仮 | 提案されるが強制されない |
| 0.5 | 中程度 | 関連する場合に適用 |
| 0.7 | 強い | 適用の自動承認 |
| 0.9 | ほぼ確実 | コア動作 |

**信頼度が上昇する場合：**
- パターンが繰り返し観察される
- ユーザーが提案された動作を修正しない
- 他のソースからの類似インスティンクトが一致

**信頼度が低下する場合：**
- ユーザーが明示的に動作を修正
- パターンが長期間観察されない
- 矛盾する証拠が出現

## なぜ監視にスキルではなくフックを使うのか？

> "v1は監視にスキルを使用していました。スキルは確率的です - Claudeの判断に基づいて50-80%の確率で発火します。"

フックは**100%の確率で**、決定論的に発火します。これは以下を意味します：
- すべてのツール呼び出しが観察される
- パターンが見逃されない
- 学習が包括的

## 後方互換性

v2はv1と完全に互換性があります：
- 既存の`~/.claude/skills/learned/`スキルは引き続き動作
- Stopフックも実行される（ただしv2にもフィード）
- 段階的な移行パス：両方を並行して実行可能

## プライバシー

- 観察はあなたのマシンに**ローカル**で保存
- **インスティンクト**（パターン）のみエクスポート可能
- 実際のコードや会話内容は共有されない
- エクスポートするものを自分で制御

## 関連

- [Skill Creator](https://skill-creator.app) - リポジトリ履歴からインスティンクトを生成
- [Homunculus](https://github.com/humanplane/homunculus) - v2アーキテクチャのインスピレーション
- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 継続的学習セクション

---

*インスティンクトベース学習：一度の観察で、Claudeにあなたのパターンを教える。*
