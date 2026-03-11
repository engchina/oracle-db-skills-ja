# Oracle DB Skills

AIエージェント向けの102のOracle Databaseリファレンス・ガイドです。各ファイルは独立したスキルとして、例、ベスト・プラクティス、およびよくある間違いを1つのトピックごとに網羅しています。

**インストール:** `npx skills add krisrice/oracle-db-skills`

---

| パス | カテゴリ | 説明 |
|------|----------|-------------|
| `skills/design/erd-design.md` | design | エンティティ関係設計、正規化（1NF–5NF）、Oracleの命名規則、予約語 |
| `skills/design/data-modeling.md` | design | 論理モデリング vs 物理モデリング、スター/スノーフレーク・スキーマ、ODS、SCDタイプ |
| `skills/design/partitioning-strategy.md` | design | レンジ、リスト、ハッシュ、コンポジット・パーティショニング、パーティション・プルーニング、ローカル vs グローバル索引 |
| `skills/design/tablespace-design.md` | design | サイジング、bigfile vs smallfile、ASSM vs MSSM、本番環境のレイアウト・パターン |
| `skills/sql-dev/sql-tuning.md` | sql-dev | 実行計画、オプティマイザ・ヒント、SQLプロファイル、計画ベースライン |
| `skills/sql-dev/sql-injection-avoidance.md` | sql-dev | バインド変数、DBMS_ASSERT、安全な動的SQLパターン |
| `skills/sql-dev/pl-sql-best-practices.md` | sql-dev | BULK COLLECT/FORALL、例外処理、カーソル管理、パッケージ構造 |
| `skills/sql-dev/sql-patterns.md` | sql-dev | 分析関数、CTE、CONNECT BY、PIVOT/UNPIVOT、MERGE、MODEL句 |
| `skills/sql-dev/dynamic-sql.md` | sql-dev | EXECUTE IMMEDIATE、DBMS_SQL、1回の解析で複数回実行、インジェクション防止 |
| `skills/performance/awr-reports.md` | performance | AWRレポートの生成と読み方、主要セクション、ベースライン、ボトルネックの特定 |
| `skills/performance/ash-analysis.md` | performance | アクティブ・セッション履歴、リアルタイム vs 履歴分析、ASHレポートの生成 |
| `skills/performance/explain-plan.md` | performance | DBMS_XPLAN、実行計画の読み方、自動トレース、問題のある計画の特定 |
| `skills/performance/index-strategy.md` | performance | Bツリー、ビットマップ、関数ベース、コンポジット、不可視索引。再構築 vs 結合 |
| `skills/performance/optimizer-stats.md` | performance | DBMS_STATS、ヒストグラム、拡張統計、保留中の統計、増分統計 |
| `skills/performance/wait-events.md` | performance | 一般的な待機イベント、診断クエリ、各イベント・タイプのリメディエーション |
| `skills/performance/memory-tuning.md` | performance | SGAコンポーネント、PGA管理、AMM vs ASMM、アドバイザ・ビュー |
| `skills/appdev/connection-pooling.md` | appdev | UCP、DRCP、プール・サイジング、接続検証、JDBC/Python/Node.jsの例 |
| `skills/appdev/transaction-management.md` | appdev | ACID特性、セーブポイント、自律型トランザクション、分散トランザクション |
| `skills/appdev/locking-concurrency.md` | appdev | MVCC、SELECT FOR UPDATE、NOWAIT/SKIP LOCKED、デッドロックの回避 |
| `skills/appdev/sequences-identity.md` | appdev | シーケンス・キャッシュ、アイデンティティ列、UUIDの代替案、ギャップ動作 |
| `skills/appdev/json-in-oracle.md` | appdev | ネイティブJSON型、JSON_VALUE/QUERY/TABLE、JSON Duality Views (23c) |
| `skills/appdev/xml-in-oracle.md` | appdev | XMLTypeストレージ、XQuery、XMLTable、XML索引、XMLDBリポジトリ |
| `skills/appdev/spatial-data.md` | appdev | SDO_GEOMETRY、空間索引、SDO_RELATE、座標系 |
| `skills/appdev/oracle-text.md` | appdev | CONTEXT/CTXCAT索引、CONTAINS、あいまい/ステミング検索、HIGHLIGHT/SNIPPET |
| `skills/security/privilege-management.md` | security | 最小権限、ロール、DBMS_PRIVILEGE_CAPTURE、PUBLICへの権限付与の回避 |
| `skills/security/row-level-security.md` | security | VPD/FGAC、DBMS_RLS、アプリケーション・コンテキスト、すべてのポリシー・タイプ |
| `skills/security/data-masking.md` | security | Oracle Data Redaction (DBMS_REDACT)、完全/部分/正規表現/ランダム・リダクション |
| `skills/security/auditing.md` | security | 統合監査、CREATE AUDIT POLICY、ファイングレイン監査 (DBMS_FGA) |
| `skills/security/encryption.md` | security | TDE、Oracleウォレットのセットアップ、表領域/列の暗号化、キーのローテーション |
| `skills/security/network-security.md` | security | SSL/TLS、sqlnet.oraの暗号化、ネットワーク・パッケージのACL、リスナーの要塞化 |
| `skills/admin/backup-recovery.md` | admin | RMANアーキテクチャ、バックアップ・セット vs イメージ・コピー、増分バックアップ、リカバリ・シナリオ |
| `skills/admin/dataguard.md` | admin | フィジカル/ロジカル・スタンバイ、Data Guard Broker、スイッチオーバー vs フェイルオーバー、Active Data Guard |
| `skills/admin/rman-basics.md` | admin | 一般的なRMANコマンド、チャネル構成、圧縮、暗号化、レポート |
| `skills/admin/undo-management.md` | admin | UNDOサイズ設定、UNDO_RETENTION、ORA-01555の原因と防止、Undoアドバイザ |
| `skills/admin/redo-log-management.md` | admin | ログ・サイズ設定、アーカイブログ・モード、多重化、スイッチ頻度の監視 |
| `skills/admin/user-management.md` | admin | CREATE USER、プロファイル、パスワード・ポリシー、プロキシ認証、CDB/PDBユーザー |
| `skills/monitoring/alert-log-analysis.md` | monitoring | アラートログの場所、重要なORA-エラー、自動監視パターン |
| `skills/monitoring/adrci-usage.md` | monitoring | ADRリポジトリ、adrciコマンド、IPSパッケージング、インシデントの相関 |
| `skills/monitoring/health-monitor.md` | monitoring | DBMS_HMヘルスチェック、SQLチューニング・アドバイザ、セグメント・アドバイザ、メモリ・アドバイザ |
| `skills/monitoring/space-management.md` | monitoring | 表領域の監視、HWM、SHRINK SPACE vs MOVE、LOB領域、一時領域 |
| `skills/monitoring/top-sql-queries.md` | monitoring | V$SQL/V$SQLAREA、リソース別トップSQL、AWRトップSQL、V$SQL_MONITOR |
| `skills/architecture/rac-concepts.md` | architecture | キャッシュ・フュージョン、GCS/GES、サービス、ノード・アフィニティ、RAC待機イベント、TAF/FCF |
| `skills/architecture/multitenant.md` | architecture | CDB/PDBアーキテクチャ、クローニング、プラグ/アンプラグ、リソース管理、アプリケーション・コンテナ |
| `skills/architecture/oracle-cloud-oci.md` | architecture | ATP、ADW、Base Database Service、ExaCS、接続方法、Free Tier |
| `skills/architecture/exadata-features.md` | architecture | スマート・スキャン、ストレージ索引、HCC圧縮、IORM、オフロード監視 |
| `skills/architecture/inmemory-column-store.md` | architecture | IMCSアーキテクチャ、オブジェクトのロード、結合グループ、インメモリー集計、AIM |
| `skills/devops/schema-migrations.md` | devops | OracleでのLiquibaseとFlywayの使用、バージョン管理 vs 繰り返し可能なマイグレーション、CI/CDパイプライン |
| `skills/devops/online-operations.md` | devops | DBMS_REDEFINITION、オンラインでの索引再構築/作成、ALTER TABLE ONLINE |
| `skills/devops/edition-based-redefinition.md` | devops | ゼロダウンタイム・デプロイメントのためのEBR、エディション・ビュー、クロスエディション・トリガー |
| `skills/devops/database-testing.md` | devops | utPLSQLフレームワーク、アサーション、モック、コード・カバレッジ、GitHub Actions統合 |
| `skills/devops/version-control-sql.md` | devops | DBMS_METADATA DDL抽出、git構造、ドリフト検出、べき等な権限付与 |
| `skills/migrations/migrate-postgres-to-oracle.md` | migrations | データ型のマッピング、SQL方言の違い、SERIAL→アイデンティティ列、psql vs sqlplus |
| `skills/migrations/migrate-mysql-to-oracle.md` | migrations | AUTO_INCREMENT、LIMIT→FETCH、ストアド・プロシージャの変換、mysqldumpからOracleへの移行 |
| `skills/migrations/migrate-redshift-to-oracle.md` | migrations | MPP vs Oracle、分散/ソート・キー、COPYコマンド、WLM→リソース・マネージャ |
| `skills/migrations/migrate-sqlserver-to-oracle.md` | migrations | T-SQL→PL/SQL、TRY/CATCH→EXCEPTION処理、リンク・サーバー→DBリンク、SSMAガイド |
| `skills/migrations/migrate-db2-to-oracle.md` | migrations | DB2 SQL方言、REORG→MOVE、RUNSTATS→DBMS_STATS、LOCATE vs INSTR |
| `skills/migrations/migrate-sqlite-to-oracle.md` | migrations | 型アフィニティ、AUTOINCREMENT、プラグマ、組み込みからエンタープライズへのスケーリング |
| `skills/migrations/migrate-mongodb-to-oracle.md` | migrations | ドキュメント型からリレーショナル表への変換、JSON Duality Views、集計パイプライン→SQL |
| `skills/migrations/migrate-snowflake-to-oracle.md` | migrations | VARIANT/OBJECT→JSON、QUALIFY→ウィンドウ関数、タイム・トラベル→フラッシュバック |
| `skills/migrations/migrate-teradata-to-oracle.md` | migrations | BTEQ→SQL*Plus、マルチセット・テーブル、QUALIFY、TPT→SQL*Loader |
| `skills/migrations/migrate-sybase-to-oracle.md` | migrations | 連鎖/非連鎖トランザクション、RAISERROR→RAISE_APPLICATION_ERROR、BCP→SQL*Loader |
| `skills/migrations/oracle-migration-tools.md` | migrations | SQL Developer移行ワークベンチ、AWS SCT、ora2pg、Oracle ZDM、GoldenGate |
| `skills/migrations/migration-assessment.md` | migrations | 移行前チェックリスト、複雑さのスコアリング、リスク・マトリックス、作業量見積もり |
| `skills/migrations/migration-data-validation.md` | migrations | 行数カウント、ORA_HASHフィンガープリント、不整合レポート、ドリフト検出 |
| `skills/migrations/migration-cutover-strategy.md` | migrations | 切り替えフェーズ、並行稼働、Go/No-Go基準、ロールバック計画、関係者コミュニケーション |
| `skills/plsql/plsql-package-design.md` | plsql | 仕様部 vs 本体、公開/非公開API、初期化ブロック、ACCESSIBLE BY、オーバーロード |
| `skills/plsql/plsql-error-handling.md` | plsql | 例外階層、PRAGMA EXCEPTION_INIT、FORMAT_ERROR_BACKTRACE、自律型ログ記録 |
| `skills/plsql/plsql-performance.md` | plsql | コンキテクスト・スイッチ、BULK COLLECT/FORALL、パイプライン関数、RESULT_CACHE、PRAGMA UDF |
| `skills/plsql/plsql-collections.md` | plsql | 結合配列、ネストした表、VARRAY、コレクション・メソッド、SQLでのTABLE()使用 |
| `skills/plsql/plsql-cursors.md` | plsql | 暗黙的/明示的カーソル、カーソルFORループ、REF CURSORs、SYS_REFCURSOR、リーク防止 |
| `skills/plsql/plsql-dynamic-sql.md` | plsql | EXECUTE IMMEDIATE、DBMS_SQL、1回の解析で複数回実行、インジェクション防止 |
| `skills/plsql/plsql-security.md` | plsql | AUTHID DEFINER vs CURRENT_USER、インジェクション・ベクター、DBMS_ASSERT、セキュア・コーディング・チェックリスト |
| `skills/plsql/plsql-debugging.md` | plsql | DBMS_OUTPUT、DBMS_APPLICATION_INFO、SQL Developerデバッガ、PLSQL_WARNINGS、DBMS_TRACE |
| `skills/plsql/plsql-unit-testing.md` | plsql | utPLSQL、テスト・パッケージ、アサーション、モック、CI統合、コード・カバレッジ |
| `skills/plsql/plsql-patterns.md` | plsql | TAPIパターン、自律型トランザクション・ログ記録、パイプライン関数、オブジェクト型 |
| `skills/plsql/plsql-compiler-options.md` | plsql | PLSQL_OPTIMIZE_LEVEL、ネイティブ vs 解釈実行、条件付きコンパイル、PLSQL_CCFLAGS |
| `skills/plsql/plsql-code-quality.md` | plsql | Naming規則、Trivadisガイドライン、アンチパターン、レビュー・チェックリスト、PL/SQL Cop |
| `skills/features/advanced-queuing.md` | features | AQ/トランザクション・イベント・キュー、DBMS_AQ/DBMS_AQADM、伝播、JMS、TEQ (21c) |
| `skills/features/dbms-scheduler.md` | features | ジョブ、スケジュール、チェーン、イベント・ベースのスケジューリング、ウィンドウ、監視 |
| `skills/features/virtual-columns.md` | features | GENERATED ALWAYS AS、仮想列の索引付け、パーティション・キー、制限事項 |
| `skills/features/materialized-views.md` | features | COMPLETE/FAST/FORCEリフレッシュ、ON COMMIT、MVログ、クエリ・リライト |
| `skills/features/database-links.md` | features | 固定/接続/共有リンク、分散DML、2フェーズ・コミット、セキュリティ・リスク |
| `skills/features/oracle-apex.md` | features | APEXアーキテクチャ、認証、ORDS統合、REST API、CI/CDデプロイメント |
| `skills/sqlcl/sqlcl-basics.md` | sqlcl | インストール、接続（TNS/簡易接続/ウォレット）、SQL*Plusとの主な違い |
| `skills/sqlcl/sqlcl-scripting.md` | sqlcl | JavaScriptエンジン（Nashorn/GraalVM）、scriptコマンド、Java相互運用性、自動化の例 |
| `skills/sqlcl/sqlcl-liquibase.md` | sqlcl | 組み込みのLiquibase、lb generate-schema、lb update/rollback、CI/CD統合 |
| `skills/sqlcl/sqlcl-formatting.md` | sqlcl | SET SQLFORMATモード（CSV, JSON, XML, INSERT, LOADER）、COLUMN、SPOOL |
| `skills/sqlcl/sqlcl-ddl-generation.md` | sqlcl | DDLコマンド、ストレージ句の抑制、スキーマ全体の抽出、バージョン管理 |
| `skills/sqlcl/sqlcl-data-loading.md` | sqlcl | CSV/JSON用LOADコマンド、列マッピング、日付形式、エラー処理 |
| `skills/sqlcl/sqlcl-cicd.md` | sqlcl | ヘッドレス/非対話モード、終了コード、ウォレット接続、GitHub Actions/GitLab CI |
| `skills/sqlcl/sqlcl-mcp-server.md` | sqlcl | MCPサーバーのセットアップ、Claude/AIアシスタントのOracleへの接続、使用可能なツール、セキュリティ |
| `skills/ords/ords-architecture.md` | ords | デプロイメント・モデル（Jetty/Tomcat/WebLogic/OCI）、リクエスト処理ルーティング、モジュール階層 |
| `skills/ords/ords-installation.md` | ords | ORDSのインストール、`ords config set`、ウォレットベースの資格証明保存、ATP/ADW用mTLS |
| `skills/ords/ords-auto-rest.md` | ords | ORDS.ENABLE_SCHEMA/OBJECT、エンドポイント・パターン、JSONフィルタ構文、ページネーション |
| `skills/ords/ords-rest-api-design.md` | ords | DEFINE_MODULE/TEMPLATE/HANDLER、ソース・タイプ、暗黙的なバインド・パラメータ、CRUDの例 |
| `skills/ords/ords-authentication.md` | ords | OAuth2 クライアント資格証明および認証コード・フロー、JWT検証、ロール・マッピング |
| `skills/ords/ords-pl-sql-gateway.md` | ords | RESTからのPL/SQL呼び出し、REF CURSORs、APEX_JSON、エラー処理、CLOB/BLOB |
| `skills/ords/ords-file-upload-download.md` | ords | BLOBのアップロード/ダウンロード、マルチパート形式データ、Content-Type/Content-Disposition |
| `skills/ords/ords-metadata-catalog.md` | ords | OpenAPI 3.0生成、Swagger UI/Postman統合、メタデータ・ビュー |
| `skills/ords/ords-security.md` | ords | HTTPSの強制、`ords config set`経由のCORS、ウォレットベースのシークレット、リクエスト検証 |
| `skills/ords/ords-monitoring.md` | ords | ログ構成、リクエスト・ログ記録、接続プール監視、エラー診断 |
