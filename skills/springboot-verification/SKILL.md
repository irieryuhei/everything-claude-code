---
name: springboot-verification
description: Spring Bootプロジェクトの検証ループ：ビルド、静的解析、カバレッジ付きテスト、セキュリティスキャン、リリースまたはPR前のdiffレビュー。
---

# Spring Boot検証ループ

PR前、大きな変更後、デプロイ前に実行。

## フェーズ1: ビルド

```bash
mvn -T 4 clean verify -DskipTests
# または
./gradlew clean assemble -x test
```

ビルドが失敗した場合は停止して修正。

## フェーズ2: 静的解析

Maven（一般的なプラグイン）：
```bash
mvn -T 4 spotbugs:check pmd:check checkstyle:check
```

Gradle（設定済みの場合）：
```bash
./gradlew checkstyleMain pmdMain spotbugsMain
```

## フェーズ3: テスト + カバレッジ

```bash
mvn -T 4 test
mvn jacoco:report   # 80%以上のカバレッジを確認
# または
./gradlew test jacocoTestReport
```

レポート：
- 総テスト数、パス/失敗
- カバレッジ%（行/ブランチ）

## フェーズ4: セキュリティスキャン

```bash
# 依存関係CVE
mvn org.owasp:dependency-check-maven:check
# または
./gradlew dependencyCheckAnalyze

# シークレット（git）
git secrets --scan  # 設定済みの場合
```

## フェーズ5: リント/フォーマット（オプションゲート）

```bash
mvn spotless:apply   # Spotlessプラグインを使用している場合
./gradlew spotlessApply
```

## フェーズ6: Diffレビュー

```bash
git diff --stat
git diff
```

チェックリスト：
- デバッグログが残っていない（`System.out`、ガードなしの`log.debug`）
- 意味のあるエラーとHTTPステータス
- 必要な場所にトランザクションとバリデーション
- 設定変更がドキュメント化されている

## 出力テンプレート

```
検証レポート
===================
ビルド:     [PASS/FAIL]
静的解析:   [PASS/FAIL] (spotbugs/pmd/checkstyle)
テスト:     [PASS/FAIL] (X/Yパス、Z%カバレッジ)
セキュリティ: [PASS/FAIL] (CVE検出数: N)
Diff:       [X個のファイルが変更]

総合:       [READY / NOT READY]

修正すべき問題：
1. ...
2. ...
```

## 継続モード

- 大きな変更後または長いセッションで30〜60分ごとにフェーズを再実行
- 短いループを維持: `mvn -T 4 test` + spotbugsで素早いフィードバック

**覚えておくこと**: 早いフィードバックは遅い驚きに勝る。ゲートを厳格に保つ - 本番システムでは警告を欠陥として扱う。
