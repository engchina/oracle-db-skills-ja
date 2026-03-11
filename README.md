# Oracle DB Skills

Oracle DB Skillsは、Oracle Databaseを扱うための、ドキュメントに基づいた100以上の実用的なガイドを厳選したライブラリです。SQLおよびPL/SQL開発、パフォーマンス・チューニング、セキュリティ、管理、モニタリング、アーキテクチャ、DevOps、移行、SQLcl、ORDS、およびOracle固有の機能といったドメインごとに整理されています。各ガイドには、実行可能な例、ベスト・プラクティス、よくある落とし穴、ソース、および19cと26aiに関する明示的なOracleバージョンの互換性ノートが含まれています。

## バージョン対応標準

- バージョン固有の動作を含むスキルには、`## Oracle Version Notes (19c vs 26ai)`というセクションを含める必要があります。
- 特に断りのない限り、Oracle Database 19cをベースラインの互換性ターゲットとして使用します。
- 新しいリリースを必要とする機能は明示的に指摘し、可能な場合は19c互換の代替案を提供します。

## GitHub ルールセット

- デフォルトブランチのルールセット定義は`.github/rulesets/main.json`に保存されています。
- 以下のコマンドで適用します：
  - `export GITHUB_TOKEN=<token-with-repo-admin-permission>`
  - `./scripts/apply-github-ruleset.sh krisrice oracle-db-skills`

---

## カテゴリ

| カテゴリ | ファイル数 | パス |
|----------|-------|------|
| [データベース設計とモデリング](#データベース設計とモデリング) | 4 | `skills/design/` |
| [SQL開発](#sql開発) | 5 | `skills/sql-dev/` |
| [パフォーマンスとチューニング](#パフォーマンスとチューニング) | 7 | `skills/performance/` |
| [アプリケーション開発](#アプリケーション開発) | 8 | `skills/appdev/` |
| [セキュリティ](#セキュリティ) | 6 | `skills/security/` |
| [データベース管理](#データベース管理) | 6 | `skills/admin/` |
| [モニタリングと診断](#モニタリングと診断) | 5 | `skills/monitoring/` |
| [アーキテクチャとインフラストラクチャ](#アーキテクチャとインフラストラクチャ) | 5 | `skills/architecture/` |
| [DevOpsとCI/CD](#devopsとcicd) | 5 | `skills/devops/` |
| [Oracleへの移行](#oracleへの移行) | 14 | `skills/migrations/` |
| [PL/SQL開発](#plsql開発) | 12 | `skills/plsql/` |
| [Oracle固有の機能](#oracle固有の機能) | 6 | `skills/features/` |
| [SQLcl](#sqlcl) | 8 | `skills/sqlcl/` |
| [ORDS (Oracle REST Data Services)](#ords-oracle-rest-data-services) | 10 | `skills/ords/` |

---

## データベース設計とモデリング

`skills/design/`

| ファイル | 説明 |
|------|-------------|
| `erd-design.md` | エンティティ関係設計、正規化（第1正規形〜第5正規形）、Oracleの命名規則、予約語 |
| `data-modeling.md` | 論理モデリング vs 物理モデリング、スター/スノーフレーク・スキーマ、ODS、SCDタイプ |
| `partitioning-strategy.md` | レンジ、リスト、ハッシュ、コンポジット・パーティショニング、パーティション・プルーニング、ローカル vs グローバル索引 |
| `tablespace-design.md` | サイジング、bigfile vs smallfile、ASSM vs MSSM、本番環境のレイアウト・パターン |

---

## SQL開発

`skills/sql-dev/`

| ファイル | 説明 |
|------|-------------|
| `sql-tuning.md` | 実行計画、オプティマイザ・ヒント、SQLプロファイル、計画ベースライン |
| `sql-injection-avoidance.md` | バインド変数、DBMS_ASSERT、安全な動的SQLパターン |
| `pl-sql-best-practices.md` | BULK COLLECT/FORALL、例外処理、カーソル管理、パッケージ構造 |
| `sql-patterns.md` | 分析関数（ウィンドウ関数）、CTE（共通表式）、CONNECT BY、PIVOT/UNPIVOT、MERGE、MODEL句 |
| `dynamic-sql.md` | EXECUTE IMMEDIATE、DBMS_SQL、1回の解析で複数回実行、インジェクション防止 |

---

## パフォーマンスとチューニング

`skills/performance/`

| ファイル | 説明 |
|------|-------------|
| `awr-reports.md` | AWRレポートの生成と読み方、主要セクション、ベースライン、ボトルネックの特定 |
| `ash-analysis.md` | アクティブ・セッション履歴（ASH）、リアルタイム分析 vs 履歴分析、ASHレポートの生成 |
| `explain-plan.md` | DBMS_XPLAN、実行計画の読み方、自動トレース、問題のある計画の特定 |
| `index-strategy.md` | Bツリー、ビットマップ、関数ベース、コンポジット、不可視索引。再構築 vs 結合 |
| `optimizer-stats.md` | DBMS_STATS、ヒストグラム、拡張統計、保留中の統計、増分統計 |
| `wait-events.md` | 一般的な待機イベント、診断クエリ、各イベント・タイプのリメディエーション（修復策） |
| `memory-tuning.md` | SGAコンポーネント、PGA管理、AMM vs ASMM、アドバイザ・ビュー |

---

## アプリケーション開発

`skills/appdev/`

| ファイル | 説明 |
|------|-------------|
| `connection-pooling.md` | UCP、DRCP、プール・サイジング、接続検証、JDBC/Python/Node.jsの例 |
| `transaction-management.md` | ACID特性、セーブポイント、自律型トランザクション、分散トランザクション |
| `locking-concurrency.md` | MVCC、SELECT FOR UPDATE、NOWAIT/SKIP LOCKED、デッドロックの回避 |
| `sequences-identity.md` | シーケンス・キャッシュ、アイデンティティ列、UUIDの代替案、ギャップ動作 |
| `json-in-oracle.md` | ネイティブJSON型、JSON_VALUE/QUERY/TABLE、JSON Duality Views (23c) |
| `xml-in-oracle.md` | XMLTypeストレージ、XQuery、XMLTable、XML索引、XMLDBリポジトリ |
| `spatial-data.md` | SDO_GEOMETRY、空間索引、SDO_RELATE、座標系 |
| `oracle-text.md` | CONTEXT/CTXCAT索引、CONTAINS、あいまい/ステミング検索、HIGHLIGHT/SNIPPET |

---

## セキュリティ

`skills/security/`

| ファイル | 説明 |
|------|-------------|
| `privilege-management.md` | 最小権限、ロール、DBMS_PRIVILEGE_CAPTURE、PUBLICへの権限付与の回避 |
| `row-level-security.md` | VPD/FGAC、DBMS_RLS、アプリケーション・コンテキスト、すべてのポリシー・タイプ |
| `data-masking.md` | Oracle Data Redaction (DBMS_REDACT)、完全/部分/正規表現/ランダム・リダクション |
| `auditing.md` | 統合監査、CREATE AUDIT POLICY、ファイングレイン監査 (DBMS_FGA) |
| `encryption.md` | TDE, Oracleウォレットのセットアップ、表領域/列の暗号化、キーのローテーション |
| `network-security.md` | SSL/TLS、sqlnet.oraの暗号化、ネットワーク・パッケージのACL、リスナーの要塞化 |

---

## データベース管理

`skills/admin/`

| ファイル | 説明 |
|------|-------------|
| `backup-recovery.md` | RMANアーキテクチャ、バックアップ・セット vs イメージ・コピー、増分バックアップ、リカバリ・シナリオ |
| `dataguard.md` | フィジカル/ロジカル・スタンバイ、Data Guard Broker、スイッチオーバー vs フェイルオーバー、Active Data Guard |
| `rman-basics.md` | 一般的なRMANコマンド、チャネル構成、圧縮、暗号化、レポート |
| `undo-management.md` | UNDOサイズ設定、UNDO_RETENTION、ORA-01555の原因と防止、Undoアドバイザ |
| `redo-log-management.md` | ログ・サイズ設定、アーカイブログ・モード、多重化、スイッチ頻度の監視 |
| `user-management.md` | CREATE USER、プロファイル、パスワード・ポリシー、プロキシ認証、CDB/PDBユーザー |

---

## モニタリングと診断

`skills/monitoring/`

| ファイル | 説明 |
|------|-------------|
| `alert-log-analysis.md` | アラートログの場所、重要なORA-エラー、自動監視パターン |
| `adrci-usage.md` | ADRリポジトリ、adrciコマンド、IPSパッケージング、インシデントの相関 |
| `health-monitor.md` | DBMS_HMヘルスチェック、SQLチューニング・アドバイザ、セグメント・アドバイザ、メモリ・アドバイザ |
| `space-management.md` | 表領域の監視、HWM（高水位標）、SHRINK SPACE vs MOVE、LOB領域、一時領域 |
| `top-sql-queries.md` | V$SQL/V$SQLAREA、リソース別トップSQL、AWRトップSQL、V$SQL_MONITOR |

---

## アーキテクチャとインフラストラクチャ

`skills/architecture/`

| ファイル | 説明 |
|------|-------------|
| `rac-concepts.md` | キャッシュ・フュージョン、GCS/GES、サービス、ノード・アフィニティ、RAC待機イベント、TAF/FCF |
| `multitenant.md` | CDB/PDBアーキテクチャ、クローニング、プラグ/アンプラグ、リソース管理、アプリケーション・コンテナ |
| `oracle-cloud-oci.md` | ATP、ADW、Base Database Service、ExaCS、接続方法、Free Tier |
| `exadata-features.md` | スマート・スキャン、ストレージ索引、HCC圧縮、IORM、オフロード監視 |
| `inmemory-column-store.md` | IMCSアーキテクチャ、オブジェクトのロード、結合グループ、インメモリー集計、AIM |

---

## DevOpsとCI/CD

`skills/devops/`

| ファイル | 説明 |
|------|-------------|
| `schema-migrations.md` | OracleでのLiquibaseとFlywayの使用、バージョン管理 vs 繰り返し可能なマイグレーション、CI/CDパイプライン |
| `online-operations.md` | DBMS_REDEFINITION、オンラインでの索引再構築/作成、ALTER TABLE ONLINE |
| `edition-based-redefinition.md` | ゼロダウンタイム・デプロイメントのためのEBR、エディション・ビュー、クロスエディション・トリガー |
| `database-testing.md` | utPLSQLフレームワーク、アサーション、モック、コード・カバレッジ、GitHub Actions統合 |
| `version-control-sql.md` | DBMS_METADATA DDL抽出、git構造、ドリフト検出、べき等な権限付与 |

---

## Oracleへの移行

`skills/migrations/`

| ファイル | 説明 |
|------|-------------|
| `migrate-postgres-to-oracle.md` | データ型のマッピング、SQL方言の違い、SERIAL→アイデンティティ列、psql vs sqlplus |
| `migrate-mysql-to-oracle.md` | AUTO_INCREMENT、LIMIT→FETCH、ストアド・プロシージャの変換、mysqldumpからOracleへの移行 |
| `migrate-redshift-to-oracle.md` | MPP vs Oracle、分散/ソート・キー、COPYコマンド、WLM→リソース・マネージャ |
| `migrate-sqlserver-to-oracle.md` | T-SQL→PL/SQL、TRY/CATCH→EXCEPTION処理、リンク・サーバー→DBリンク、SSMAガイド |
| `migrate-db2-to-oracle.md` | DB2 SQL方言、REORG→MOVE、RUNSTATS→DBMS_STATS、LOCATE vs INSTR |
| `migrate-sqlite-to-oracle.md` | 型アフィニティ、AUTOINCREMENT, プラグマ、組み込みからエンタープライズへのスケーリング |
| `migrate-mongodb-to-oracle.md` | ドキュメント型からリレーショナル表への変換、JSON Duality Views, 集計パイプライン→SQL |
| `migrate-snowflake-to-oracle.md` | VARIANT/OBJECT→JSON、QUALIFY→ウィンドウ関数、タイム・トラベル→フラッシュバック |
| `migrate-teradata-to-oracle.md` | BTEQ→SQL*Plus、マルチセット・テーブル、QUALIFY、TPT→SQL*Loader |
| `migrate-sybase-to-oracle.md` | 連鎖/非連鎖トランザクション、RAISERROR→RAISE_APPLICATION_ERROR, BCP→SQL*Loader |
| `oracle-migration-tools.md` | SQL Developer移行ワークベンチ、AWS SCT, ora2pg, Oracle ZDM, GoldenGate |
| `migration-assessment.md` | 移行前チェックリスト、複雑さのスコアリング、リスク・マトリックス、作業量見積もり |
| `migration-data-validation.md` | 行数カウント、ORA_HASHフィンガープリント、不整合レポート、ドリフト検出 |
| `migration-cutover-strategy.md` | 切り替えフェーズ、並行稼働、Go/No-Go基準、ロールバック計画、関係者コミュニケーション |

---

## PL/SQL開発

`skills/plsql/`

| ファイル | 説明 |
|------|-------------|
| `plsql-package-design.md` | 仕様部 vs 本体、公開/非公開API, 初期化ブロック、ACCESSIBLE BY, オーバーロード |
| `plsql-error-handling.md` | 例外階層、PRAGMA EXCEPTION_INIT, FORMAT_ERROR_BACKTRACE, 自律型ログ記録 |
| `plsql-performance.md` | コンキテクスト・スイッチ、BULK COLLECT/FORALL, パイプライン関数、RESULT_CACHE, PRAGMA UDF |
| `plsql-collections.md` | 結合配列、ネストした表、VARRAY, コレクション・メソッド、SQLでのTABLE()使用 |
| `plsql-cursors.md` | 暗黙的/明示的カーソル、カーソルFORループ、REF CURSORs, SYS_REFCURSOR, リーク防止 |
| `plsql-dynamic-sql.md` | EXECUTE IMMEDIATE, DBMS_SQL, 1回の解析で複数回実行、インジェクション防止 |
| `plsql-security.md` | AUTHID DEFINER vs CURRENT_USER, インジェクション・ベクター、DBMS_ASSERT, セキュア・コーディング・チェックリスト |
| `plsql-debugging.md` | DBMS_OUTPUT, DBMS_APPLICATION_INFO, SQL Developerデバッガ、PLSQL_WARNINGS, DBMS_TRACE |
| `plsql-unit-testing.md` | utPLSQL, テスト・パッケージ、アサーション、モック、CI統合、コード・カバレッジ |
| `plsql-patterns.md` | TAPIパターン、自律型トランザクション・ログ記録、パイプライン関数、オブジェクト型 |
| `plsql-compiler-options.md` | PLSQL_OPTIMIZE_LEVEL, ネイティブ vs 解釈実行、条件付きコンパイル、PLSQL_CCFLAGS |
| `plsql-code-quality.md` | 命名規則、Trivadisガイドライン、アンチパターン、レビュー・チェックリスト、PL/SQL Cop |

---

## Oracle固有の機能

`skills/features/`

| ファイル | 説明 |
|------|-------------|
| `advanced-queuing.md` | AQ/トランザクション・イベント・キュー、DBMS_AQ/DBMS_AQADM, 伝播、JMS, TEQ (21c) |
| `dbms-scheduler.md` | ジョブ、スケジュール、チェーン、イベント・ベースのスケジューリング、ウィンドウ、監視 |
| `virtual-columns.md` | GENERATED ALWAYS AS, 仮想列の索引付け、パーティション・キー、制限事項 |
| `materialized-views.md` | COMPLETE/FAST/FORCEリフレッシュ、ON COMMIT, MVログ、クエリ・リライト |
| `database-links.md` | 固定/接続/共有リンク、分散DML, 2フェーズ・コミット、セキュリティ・リスク |
| `oracle-apex.md` | APEXアーキテクチャ、認証、ORDS統合、REST API, CI/CDデプロイメント |

---

## SQLcl

`skills/sqlcl/`

| ファイル | 説明 |
|------|-------------|
| `sqlcl-basics.md` | インストール、接続（TNS/簡易接続/ウォレット）、SQL*Plusとの主な違い |
| `sqlcl-scripting.md` | JavaScriptエンジン（Nashorn/GraalVM）、scriptコマンド、Java相互運用性、自動化の例 |
| `sqlcl-liquibase.md` | 組み込みのLiquibase, lb generate-schema, lb update/rollback, CI/CD統合 |
| `sqlcl-formatting.md` | SET SQLFORMATモード（CSV, JSON, XML, INSERT, LOADER）、COLUMN, SPOOL |
| `sqlcl-ddl-generation.md` | DDLコマンド、ストレージ句の抑制、スキーマ全体の抽出、バージョン管理 |
| `sqlcl-data-loading.md` | CSV/JSON用LOADコマンド、列マッピング、日付形式、エラー処理 |
| `sqlcl-cicd.md` | ヘッドレス/非対話モード、終了コード、ウォレット接続、GitHub Actions/GitLab CI |
| `sqlcl-mcp-server.md` | MCPサーバーのセットアップ、Claude/AIアシスタントのOracleへの接続、使用可能なツール、セキュリティ |

---

## ORDS (Oracle REST Data Services)

`skills/ords/`

| ファイル | 説明 |
|------|-------------|
| `ords-architecture.md` | デプロイメント・モデル（Jetty/Tomcat/WebLogic/OCI）、リクエスト処理ルーティング、モジュール階層 |
| `ords-installation.md` | ORDSのインストール、`ords config set`、ウォレットベースの資格証明保存、ATP/ADW用mTLS |
| `ords-auto-rest.md` | ORDS.ENABLE_SCHEMA/OBJECT, エンドポイント・パターン、JSONフィルタ構文、ページネーション |
| `ords-rest-api-design.md` | DEFINE_MODULE/TEMPLATE/HANDLER, ソース・タイプ、暗黙的なバインド・パラメータ、CRUDの例 |
| `ords-authentication.md` | OAuth2 クライアント資格証明および認証コード・フロー、JWT検証、ロール・マッピング |
| `ords-pl-sql-gateway.md` | RESTからのPL/SQL呼び出し、REF CURSORs, APEX_JSON, エラー処理、CLOB/BLOB |
| `ords-file-upload-download.md` | BLOBのアップロード/ダウンロード、マルチパート形式データ、Content-Type/Content-Disposition |
| `ords-metadata-catalog.md` | OpenAPI 3.0生成、Swagger UI/Postman統合、メタデータ・ビュー |
| `ords-security.md` | HTTPSの強制、`ords config set`経由のCORS, ウォレットベースのシークレット、リクエスト検証 |
| `ords-monitoring.md` | ログ構成、リクエスト・ログ記録、接続プール監視、エラー診断 |

---

## 構造

```
oracle-db-skills/
├── README.md
├── skills-index.md          # すべてのファイルの完了ステータスを含むフル・チェックリスト
└── skills/
    ├── admin/               # データベース管理
    ├── appdev/              # アプリケーション開発
    ├── architecture/        # アーキテクチャとインフラストラクチャ
    ├── design/              # データベース設計とモデリング
    ├── devops/              # DevOpsとCI/CD
    ├── features/            # Oracle固有の機能
    ├── migrations/          # Oracleへの移行
    ├── monitoring/          # モニタリングと診断
    ├── ords/                # Oracle REST Data Services
    ├── performance/         # パフォーマンスとチューニング
    ├── plsql/               # PL/SQL開発
    ├── security/            # セキュリティ
    ├── sql-dev/             # SQL開発
    └── sqlcl/               # SQLcl
```
