# フックシステム

## フックタイプ

- **PreToolUse**: ツール実行前（バリデーション、パラメータ修正）
- **PostToolUse**: ツール実行後（自動フォーマット、チェック）
- **Stop**: セッション終了時（最終検証）

## 現在のフック（~/.claude/settings.json内）

### PreToolUse（ツール使用前）
- **tmuxリマインダー**: 長時間実行コマンド（npm, pnpm, yarn, cargoなど）にtmuxを提案
- **git pushレビュー**: プッシュ前にZedでレビューを開く
- **docブロッカー**: 不要な.md/.txtファイルの作成をブロック

### PostToolUse（ツール使用後）
- **PR作成**: PR URLとGitHub Actionsステータスをログ出力
- **Prettier**: 編集後にJS/TSファイルを自動フォーマット
- **TypeScriptチェック**: .ts/.tsxファイル編集後にtscを実行
- **console.log警告**: 編集されたファイル内のconsole.logを警告

### Stop（セッション終了時）
- **console.log監査**: セッション終了前に変更されたすべてのファイルでconsole.logをチェック

## 自動承認パーミッション

注意して使用:
- 信頼できる、明確に定義された計画に対して有効化
- 探索的な作業では無効化
- dangerously-skip-permissionsフラグは決して使用しない
- 代わりに `~/.claude.json` で `allowedTools` を設定

## TodoWriteベストプラクティス

TodoWriteツールの用途:
- 複数ステップのタスクの進捗を追跡
- 指示の理解を検証
- リアルタイムのステアリングを可能にする
- 詳細な実装ステップを表示

Todoリストで明らかになること:
- 順序が間違っているステップ
- 欠落している項目
- 不要な追加項目
- 間違った粒度
- 誤解された要件
