# チェックポイントコマンド

ワークフローでチェックポイントを作成または検証します。

## 使用方法

`/checkpoint [create|verify|list] [name]`

## チェックポイントの作成

チェックポイント作成時：

1. `/verify quick` を実行して現在の状態がクリーンであることを確認
2. チェックポイント名でgit stashまたはコミットを作成
3. チェックポイントを `.claude/checkpoints.log` に記録:

```bash
echo "$(date +%Y-%m-%d-%H:%M) | $CHECKPOINT_NAME | $(git rev-parse --short HEAD)" >> .claude/checkpoints.log
```

4. チェックポイント作成を報告

## チェックポイントの検証

チェックポイントとの比較検証時：

1. ログからチェックポイントを読み取り
2. 現在の状態とチェックポイントを比較:
   - チェックポイント以降に追加されたファイル
   - チェックポイント以降に変更されたファイル
   - 現在と当時のテスト合格率
   - 現在と当時のカバレッジ

3. 報告:
```
チェックポイント比較: $NAME
============================
変更されたファイル: X
テスト: +Y 合格 / -Z 失敗
カバレッジ: +X% / -Y%
ビルド: [成功/失敗]
```

## チェックポイントの一覧

以下の情報とともにすべてのチェックポイントを表示:
- 名前
- タイムスタンプ
- Git SHA
- 状態（現在、遅れ、先行）

## ワークフロー

一般的なチェックポイントの流れ：

```
[開始] --> /checkpoint create "feature-start"
   |
[実装] --> /checkpoint create "core-done"
   |
[テスト] --> /checkpoint verify "core-done"
   |
[リファクタリング] --> /checkpoint create "refactor-done"
   |
[PR] --> /checkpoint verify "feature-start"
```

## 引数

$ARGUMENTS:
- `create <name>` - 名前付きチェックポイントを作成
- `verify <name>` - 名前付きチェックポイントと検証
- `list` - すべてのチェックポイントを表示
- `clear` - 古いチェックポイントを削除（最新5件は保持）
