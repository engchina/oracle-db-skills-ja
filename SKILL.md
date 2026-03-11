---
name: oracle-db-skills
description: SQL、PL/SQL、パフォーマンス・チューニング、セキュリティ、ORDS、SQLcl、移行など、102のOracle Databaseリファレンス・ガイドが含まれています。個々のスキル・ファイルをオンデマンドでロードし、Oracleの各トピックに関する専門的なガイダンスを得ることができます。
---

# Oracle DB Skills

Oracle Databaseに関する102の独立したリファレンス・ガイドのコレクションです。各ファイルは1つのトピックを扱い、解説、実用的な例、ベスト・プラクティス、およびよくある間違いを網羅しています。

## 使用方法

1. 以下のカテゴリ・ルーティング表を使用して、**適切なスキルを見つけます**。
2. ユーザーのタスクに関連する**ファイルのみを読み取ります**。すべてのファイルを一度にロードしないでください。
3. 質問への回答、コードの生成、または既存作業のレビューに**ガイダンスを適用します**。

## カテゴリ・ルーティング

| ユーザーの質問内容 | 読み取り先 |
|------------------|-----------|
| バックアップ、リカバリ、RMAN、Data Guard、REDO/UNDOログ、ユーザー | `skills/admin/` |
| JDBC、接続プール、JSON、XML、空間データ、全文検索、トランザクション | `skills/appdev/` |
| RAC、CDB/PDB、Exadata、インメモリー、OCI、ATP/ADW | `skills/architecture/` |
| ERD、データ・モデリング、パーティショニング、表領域 | `skills/design/` |
| Liquibase、Flyway、オンライン操作、EBR、utPLSQL、SQL用git | `skills/devops/` |
| アドバンスト・キューイング、DBMS_SCHEDULER、マテリアライズド・ビュー、DBリンク、APEX | `skills/features/` |
| PostgreSQL、MySQL、SQL Server、MongoDBなどからの移行 | `skills/migrations/` |
| アラートログ、ADR、adrci、領域管理、トップSQL、ヘルスチェック | `skills/monitoring/` |
| ORDS、REST API、OAuth2、AutoREST、PL/SQLゲートウェイ | `skills/ords/` |
| AWR、ASH、実行計画、索引、オプティマイザ統計、待機イベント、メモリー | `skills/performance/` |
| パッケージ、カーソル、コレクション、エラー処理、ユニット・テスト、デバッグ | `skills/plsql/` |
| 権限、VPD、TDE、暗号化、監査、ネットワーク・セキュリティ | `skills/security/` |
| SQLパターン、分析関数、CTE、動的SQL、インジェクション対策 | `skills/sql-dev/` |
| SQLclコマンド、スクリプト、Liquibase CLI、MCPサーバー、CI/CD | `skills/sqlcl/` |

## スキル・ディレクトリ

```
skills/
├── admin/          データベース管理 (バックアップ、リカバリ、ユーザー、REDO/UNDO)
├── appdev/         アプリケーション開発 (JSON、XML、空間データ、テキスト、プール)
├── architecture/   インフラストラクチャ (RAC、マルチテナント、Exadata、インメモリー、OCI)
├── design/         スキーマ設計 (ERD、モデリング、パーティショニング、表領域)
├── devops/         CI/CDとDevOps (移行、EBR、テスト、バージョン管理)
├── features/       Oracleの機能 (AQ、Scheduler、MV、DBリンク、APEX)
├── migrations/     他データベースからOracleへの移行
├── monitoring/     診断 (アラートログ、ADR、ヘルスチェック、領域、トップSQL)
├── ords/           Oracle REST Data Services
├── performance/    チューニング (AWR、ASH、索引、オプティマイザ、待機イベント、メモリー)
├── plsql/          PL/SQL開発 (パッケージ、カーソル、コレクション、テスト)
├── security/       セキュリティ (権限、VPD、TDE、監査、ネットワーク)
├── sql-dev/        SQL開発 (チューニング、パターン、動的SQL、インジェクション)
└── sqlcl/          SQLcl CLIツール (基本、スクリプト、Liquibase、MCPサーバー)
```

## 主要な開始ポイント

- **`skills/sqlcl/sqlcl-mcp-server.md`** — SQLcl MCPサーバー経由でのAIアシスタントのOracle接続
- **`skills/migrations/migration-assessment.md`** — データベース移行プロジェクトの開始点
- **`skills/performance/explain-plan.md`** — すべてのSQLパフォーマンス作業の基礎
- **`skills/plsql/plsql-package-design.md`** — PL/SQLアーキテクチャに関する質問の基礎
- **`skills/devops/schema-migrations.md`** — CI/CDパイプラインでのLiquibase/FlywayとOracleの活用
