---
name: instinct-export
description: チームメイトや他のプロジェクトと共有するためにインスティンクトをエクスポート
command: /instinct-export
---

# インスティンクトエクスポートコマンド

共有可能な形式でインスティンクトをエクスポートします。以下の用途に最適：
- チームメイトとの共有
- 新しいマシンへの転送
- プロジェクト規約への貢献

## 使用方法

```
/instinct-export                           # すべての個人インスティンクトをエクスポート
/instinct-export --domain testing          # testingインスティンクトのみをエクスポート
/instinct-export --min-confidence 0.7      # 高信頼度のインスティンクトのみをエクスポート
/instinct-export --output team-instincts.yaml
```

## 実行内容

1. `~/.claude/homunculus/instincts/personal/` からインスティンクトを読み取り
2. フラグに基づいてフィルタリング
3. 機密情報を削除：
   - セッションIDを削除
   - ファイルパスを削除（パターンのみ保持）
   - 「先週」より古いタイムスタンプを削除
4. エクスポートファイルを生成

## 出力形式

YAMLファイルを作成：

```yaml
# インスティンクトエクスポート
# 生成日: 2025-01-22
# ソース: personal
# カウント: 12 インスティンクト

version: "2.0"
exported_by: "continuous-learning-v2"
export_date: "2025-01-22T10:30:00Z"

instincts:
  - id: prefer-functional-style
    trigger: "新しい関数を書くとき"
    action: "クラスより関数型パターンを使用"
    confidence: 0.8
    domain: code-style
    observations: 8

  - id: test-first-workflow
    trigger: "新機能を追加するとき"
    action: "まずテストを書いてから実装"
    confidence: 0.9
    domain: testing
    observations: 12

  - id: grep-before-edit
    trigger: "コードを変更するとき"
    action: "Grepで検索、Readで確認、その後Edit"
    confidence: 0.7
    domain: workflow
    observations: 6
```

## プライバシーに関する考慮事項

エクスポートに含まれるもの：
- ✅ トリガーパターン
- ✅ アクション
- ✅ 信頼度スコア
- ✅ ドメイン
- ✅ 観察カウント

エクスポートに含まれないもの：
- ❌ 実際のコードスニペット
- ❌ ファイルパス
- ❌ セッションのトランスクリプト
- ❌ 個人識別子

## フラグ

- `--domain <name>`: 指定されたドメインのみをエクスポート
- `--min-confidence <n>`: 最小信頼度閾値（デフォルト: 0.3）
- `--output <file>`: 出力ファイルパス（デフォルト: instincts-export-YYYYMMDD.yaml）
- `--format <yaml|json|md>`: 出力形式（デフォルト: yaml）
- `--include-evidence`: エビデンステキストを含める（デフォルト: 除外）
