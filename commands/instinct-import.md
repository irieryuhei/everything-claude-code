---
name: instinct-import
description: チームメイト、Skill Creator、または他のソースからインスティンクトをインポート
command: true
---

# インスティンクトインポートコマンド

## 実装

プラグインルートパスを使用してインスティンクトCLIを実行：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" import <file-or-url> [--dry-run] [--force] [--min-confidence 0.7]
```

または `CLAUDE_PLUGIN_ROOT` が設定されていない場合（手動インストール）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file-or-url>
```

以下からインスティンクトをインポート：
- チームメイトのエクスポート
- Skill Creator（リポジトリ分析）
- コミュニティコレクション
- 以前のマシンのバックアップ

## 使用方法

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import --from-skill-creator acme/webapp
```

## 実行内容

1. インスティンクトファイルを取得（ローカルパスまたはURL）
2. フォーマットを解析して検証
3. 既存のインスティンクトとの重複をチェック
4. 新しいインスティンクトをマージまたは追加
5. `~/.claude/homunculus/instincts/inherited/` に保存

## インポートプロセス

```
📥 インスティンクトをインポート中: team-instincts.yaml
================================================

インポートする12件のインスティンクトを発見。

競合を分析中...

## 新規インスティンクト (8)
以下が追加されます：
  ✓ use-zod-validation (信頼度: 0.7)
  ✓ prefer-named-exports (信頼度: 0.65)
  ✓ test-async-functions (信頼度: 0.8)
  ...

## 重複インスティンクト (3)
類似のインスティンクトが既にあります：
  ⚠️ prefer-functional-style
     ローカル: 信頼度0.8、12回の観察
     インポート: 信頼度0.7
     → ローカルを保持（より高い信頼度）

  ⚠️ test-first-workflow
     ローカル: 信頼度0.75
     インポート: 信頼度0.9
     → インポートに更新（より高い信頼度）

## 競合インスティンクト (1)
これらはローカルインスティンクトと矛盾します：
  ❌ use-classes-for-services
     競合先: avoid-classes
     → スキップ（手動解決が必要）

---
8件を新規追加、1件を更新、3件をスキップしますか？
```

## マージ戦略

### 重複の場合
既存のものと一致するインスティンクトをインポートする場合：
- **より高い信頼度が優先**: 信頼度の高い方を保持
- **エビデンスをマージ**: 観察カウントを結合
- **タイムスタンプを更新**: 最近検証されたとしてマーク

### 競合の場合
既存のものと矛盾するインスティンクトをインポートする場合：
- **デフォルトでスキップ**: 競合するインスティンクトをインポートしない
- **レビュー用にフラグ**: 両方に注意が必要とマーク
- **手動解決**: ユーザーがどちらを保持するか決定

## ソースの追跡

インポートされたインスティンクトには以下がマークされます：
```yaml
source: "inherited"
imported_from: "team-instincts.yaml"
imported_at: "2025-01-22T10:30:00Z"
original_source: "session-observation"  # または "repo-analysis"
```

## スキルクリエイター統合

Skill Creatorからインポートする場合：

```
/instinct-import --from-skill-creator acme/webapp
```

これはリポジトリ分析から生成されたインスティンクトを取得します：
- ソース: `repo-analysis`
- より高い初期信頼度（0.7以上）
- ソースリポジトリにリンク

## フラグ

- `--dry-run`: インポートせずにプレビュー
- `--force`: 競合があってもインポート
- `--merge-strategy <higher|local|import>`: 重複の処理方法
- `--from-skill-creator <owner/repo>`: Skill Creator分析からインポート
- `--min-confidence <n>`: 閾値以上のインスティンクトのみをインポート

## 出力

インポート後：
```
✅ インポート完了！

追加: 8 インスティンクト
更新: 1 インスティンクト
スキップ: 3 インスティンクト（2 重複、1 競合）

新しいインスティンクトの保存先: ~/.claude/homunculus/instincts/inherited/

/instinct-status を実行してすべてのインスティンクトを確認してください。
```
