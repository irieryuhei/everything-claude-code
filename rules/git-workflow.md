# Gitワークフロー

## コミットメッセージ形式

```
<type>: <description>

<optional body>
```

タイプ: feat, fix, refactor, docs, test, chore, perf, ci

注意: 帰属表示は ~/.claude/settings.json でグローバルに無効化されています。

## プルリクエストワークフロー

PR作成時:
1. 完全なコミット履歴を分析（最新コミットだけでなく）
2. `git diff [base-branch]...HEAD` で全変更を確認
3. 包括的なPR要約を作成
4. TODOを含むテストプランを記載
5. 新しいブランチの場合は `-u` フラグでプッシュ

## 機能実装ワークフロー

1. **まず計画**
   - **planner** エージェントで実装計画を作成
   - 依存関係とリスクを特定
   - フェーズに分割

2. **TDDアプローチ**
   - **tdd-guide** エージェントを使用
   - まずテストを書く（RED）
   - テストを通すように実装（GREEN）
   - リファクタリング（IMPROVE）
   - 80%以上のカバレッジを検証

3. **コードレビュー**
   - コード作成直後に **code-reviewer** エージェントを使用
   - CRITICALとHIGHの問題に対処
   - 可能であればMEDIUMの問題も修正

4. **コミット & プッシュ**
   - 詳細なコミットメッセージ
   - Conventional Commits形式に従う
