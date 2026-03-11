# Oracle DB Skills — エージェント向け指示事項

このリポジトリは、Oracle Databaseに関する102の独立したリファレンス・ガイドのコレクションです。各ファイルは1つのトピックを扱い、解説、実用的な例、ベスト・プラクティス、およびよくある間違いを網羅しています。

## このコレクションの使用方法

1. **適切なスキルを見つける** — リポジトリのルートにある`SKILLS.md`をスキャンして、説明付きの全スキルのフラット・インデックスを確認するか、以下のカテゴリ・ルーティングを使用してください。
2. **オンデマンドでロードする** — ユーザーのタスクに関連する特定のスキル・ファイルのみを読み取ってください。一度にすべてのファイルをロードしようとしないでください。
3. **ガイダンスを適用する** — コンテンツを使用して、質問への回答、コードの生成、または既存作業のレビューを行ってください。

## ディレクトリ構造

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

## 知っておくべき主要なスキル

- **`skills/sqlcl/sqlcl-mcp-server.md`** — SQLcl MCPサーバー経由でAIアシスタント（Claudeを含む）をOracleに接続する方法
- **`skills/migrations/migration-assessment.md`** — すべてのデータベース移行プロジェクトの開始点
- **`skills/performance/explain-plan.md`** — すべてのSQLパフォーマンス作業の基礎
- **`skills/plsql/plsql-package-design.md`** — PL/SQLアーキテクチャに関する質問の基礎
- **`skills/devops/schema-migrations.md`** — CI/CDパイプラインでのLiquibase/FlywayとOracleの活用
