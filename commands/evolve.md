---
name: evolve
description: 関連するインスティンクトをスキル、コマンド、またはエージェントにクラスタリング
command: true
---

# Evolveコマンド

## 実装

プラグインルートパスを使用してインスティンクトCLIを実行：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" evolve [--generate]
```

または `CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve [--generate]
```

インスティンクトを分析し、関連するものをより高レベルの構造にクラスタリングします：
- **コマンド**: インスティンクトがユーザーが呼び出すアクションを記述している場合
- **スキル**: インスティンクトが自動トリガーされる動作を記述している場合
- **エージェント**: インスティンクトが複雑なマルチステッププロセスを記述している場合

## 使用方法

```
/evolve                    # すべてのインスティンクトを分析し進化を提案
/evolve --domain testing   # testingドメインのインスティンクトのみを進化
/evolve --dry-run          # 作成せずに何が作成されるかを表示
/evolve --threshold 5      # クラスタ化に5つ以上の関連インスティンクトを要求
```

## 進化ルール

### → コマンド（ユーザー呼び出し）
ユーザーが明示的に要求するアクションをインスティンクトが記述している場合：
- 「ユーザーが...を要求したとき」に関する複数のインスティンクト
- 「新しいXを作成するとき」のようなトリガーを持つインスティンクト
- 繰り返し可能なシーケンスに従うインスティンクト

例：
- `new-table-step1`: 「データベーステーブルを追加するとき、マイグレーションを作成」
- `new-table-step2`: 「データベーステーブルを追加するとき、スキーマを更新」
- `new-table-step3`: 「データベーステーブルを追加するとき、型を再生成」

→ 作成: `/new-table` コマンド

### → スキル（自動トリガー）
自動的に発生すべき動作をインスティンクトが記述している場合：
- パターンマッチングトリガー
- エラーハンドリングのレスポンス
- コードスタイルの強制

例：
- `prefer-functional`: 「関数を書くとき、関数型スタイルを優先」
- `use-immutable`: 「状態を変更するとき、イミュータブルパターンを使用」
- `avoid-classes`: 「モジュールを設計するとき、クラスベースの設計を避ける」

→ 作成: `functional-patterns` スキル

### → エージェント（深さ/分離が必要）
分離が有効な複雑なマルチステッププロセスをインスティンクトが記述している場合：
- デバッグワークフロー
- リファクタリングシーケンス
- 調査タスク

例：
- `debug-step1`: 「デバッグするとき、まずログをチェック」
- `debug-step2`: 「デバッグするとき、失敗するコンポーネントを分離」
- `debug-step3`: 「デバッグするとき、最小限の再現を作成」
- `debug-step4`: 「デバッグするとき、テストで修正を検証」

→ 作成: `debugger` エージェント

## 実行内容

1. `~/.claude/homunculus/instincts/` からすべてのインスティンクトを読み取り
2. インスティンクトを以下でグループ化：
   - ドメインの類似性
   - トリガーパターンの重複
   - アクションシーケンスの関係
3. 3つ以上の関連インスティンクトのクラスタごとに：
   - 進化タイプを決定（コマンド/スキル/エージェント）
   - 適切なファイルを生成
   - `~/.claude/homunculus/evolved/{commands,skills,agents}/` に保存
4. 進化した構造をソースインスティンクトにリンク

## 出力形式

```
🧬 Evolve分析
==================

進化の準備ができた3つのクラスタを発見：

## クラスタ1: データベースマイグレーションワークフロー
インスティンクト: new-table-migration, update-schema, regenerate-types
タイプ: コマンド
信頼度: 85%（12の観察に基づく）

作成予定: /new-table コマンド
ファイル:
  - ~/.claude/homunculus/evolved/commands/new-table.md

## クラスタ2: 関数型コードスタイル
インスティンクト: prefer-functional, use-immutable, avoid-classes, pure-functions
タイプ: スキル
信頼度: 78%（8の観察に基づく）

作成予定: functional-patterns スキル
ファイル:
  - ~/.claude/homunculus/evolved/skills/functional-patterns.md

## クラスタ3: デバッグプロセス
インスティンクト: debug-check-logs, debug-isolate, debug-reproduce, debug-verify
タイプ: エージェント
信頼度: 72%（6の観察に基づく）

作成予定: debugger エージェント
ファイル:
  - ~/.claude/homunculus/evolved/agents/debugger.md

---
これらのファイルを作成するには `/evolve --execute` を実行してください。
```

## フラグ

- `--execute`: 進化した構造を実際に作成（デフォルトはプレビュー）
- `--dry-run`: 作成せずにプレビュー
- `--domain <name>`: 指定されたドメインのインスティンクトのみを進化
- `--threshold <n>`: クラスタを形成するために必要な最小インスティンクト数（デフォルト: 3）
- `--type <command|skill|agent>`: 指定されたタイプのみを作成

## 生成されるファイル形式

### コマンド
```markdown
---
name: new-table
description: マイグレーション、スキーマ更新、型生成を伴う新しいデータベーステーブルの作成
command: /new-table
evolved_from:
  - new-table-migration
  - update-schema
  - regenerate-types
---

# New Tableコマンド

[クラスタ化されたインスティンクトに基づいて生成されたコンテンツ]

## ステップ
1. ...
2. ...
```

### スキル
```markdown
---
name: functional-patterns
description: 関数型プログラミングパターンの強制
evolved_from:
  - prefer-functional
  - use-immutable
  - avoid-classes
---

# Functional Patternsスキル

[クラスタ化されたインスティンクトに基づいて生成されたコンテンツ]
```

### エージェント
```markdown
---
name: debugger
description: 体系的なデバッグエージェント
model: sonnet
evolved_from:
  - debug-check-logs
  - debug-isolate
  - debug-reproduce
---

# Debuggerエージェント

[クラスタ化されたインスティンクトに基づいて生成されたコンテンツ]
```
