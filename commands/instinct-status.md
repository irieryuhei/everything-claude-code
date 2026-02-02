---
name: instinct-status
description: 学習したすべてのインスティンクトと信頼度レベルを表示
command: true
---

# インスティンクトステータスコマンド

学習したすべてのインスティンクトを信頼度スコアとともにドメイン別にグループ化して表示します。

## 実装

プラグインルートパスを使用してインスティンクトCLIを実行：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" status
```

または `CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
```

## 使用方法

```
/instinct-status
/instinct-status --domain code-style
/instinct-status --low-confidence
```

## 実行内容

1. `~/.claude/homunculus/instincts/personal/` からすべてのインスティンクトファイルを読み取り
2. `~/.claude/homunculus/instincts/inherited/` から継承されたインスティンクトを読み取り
3. 信頼度バーとともにドメイン別にグループ化して表示

## 出力形式

```
📊 インスティンクトステータス
==================

## コードスタイル（4インスティンクト）

### prefer-functional-style
トリガー: 新しい関数を書くとき
アクション: クラスより関数型パターンを使用
信頼度: ████████░░ 80%
ソース: session-observation | 最終更新: 2025-01-22

### use-path-aliases
トリガー: モジュールをインポートするとき
アクション: 相対インポートの代わりに@/パスエイリアスを使用
信頼度: ██████░░░░ 60%
ソース: repo-analysis (github.com/acme/webapp)

## テスト（2インスティンクト）

### test-first-workflow
トリガー: 新機能を追加するとき
アクション: まずテストを書いてから実装
信頼度: █████████░ 90%
ソース: session-observation

## ワークフロー（3インスティンクト）

### grep-before-edit
トリガー: コードを変更するとき
アクション: Grepで検索、Readで確認、その後Edit
信頼度: ███████░░░ 70%
ソース: session-observation

---
合計: 9 インスティンクト（4 個人、5 継承）
オブザーバー: 実行中（最終分析: 5分前）
```

## フラグ

- `--domain <name>`: ドメインでフィルタリング（code-style、testing、gitなど）
- `--low-confidence`: 信頼度 < 0.5 のインスティンクトのみ表示
- `--high-confidence`: 信頼度 >= 0.7 のインスティンクトのみ表示
- `--source <type>`: ソースでフィルタリング（session-observation、repo-analysis、inherited）
- `--json`: プログラム使用のためにJSONで出力
