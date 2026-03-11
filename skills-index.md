# Oracle DB Skills インデックス

Oracle DBを扱うために作成されたskills.mdトピックの追跡ファイル。

## ステータス凡例
- [ ] 未着手
- [x] 完了

---

## データベース設計とモデリング
- [x] `erd-design.md` — エンティティ関係設計、正規化、命名規則
- [x] `data-modeling.md` — 論理モデリング vs 物理モデリング、スター/スノーフレーク・スキーマ
- [x] `partitioning-strategy.md` — レンジ、リスト、ハッシュ、コンポジット・パーティショニング
- [x] `tablespace-design.md` — サイジング、ストレージ・レイアウト、bigfile vs smallfile

## SQL開発
- [x] `sql-tuning.md` — 実行計画、ヒント、オプティマイザ統計
- [x] `sql-injection-avoidance.md` — バインド変数、動的SQLの安全性、DBMS_ASSERT
- [x] `pl-sql-best-practices.md` — 一括操作、例外処理、カーソル管理
- [x] `sql-patterns.md` — 分析関数、CTE、CONNECT BY、MODEL句
- [x] `dynamic-sql.md` — EXECUTE IMMEDIATE、DBMS_SQL、安全なパターン

## パフォーマンスとチューニング
- [x] `awr-reports.md` — AWRの読み方、主要メトリクス、Top SQL、待機イベント
- [x] `ash-analysis.md` — アクティブ・セッション履歴、リアルタイム・チューニング
- [x] `explain-plan.md` — 実行計画の読み方、DBMS_XPLAN、自動トレース
- [x] `index-strategy.md` — Bツリー、ビットマップ、関数ベース、不可視索引
- [x] `optimizer-stats.md` — DBMS_STATS、ヒストグラム、拡張統計
- [x] `wait-events.md` — 一般的な待機イベント、診断、リメディエーション
- [x] `memory-tuning.md` — SGA、PGA、バッファ・キャッシュ、共有プール・サイジング

## アプリケーション開発
- [x] `connection-pooling.md` — UCP、DRCP、接続に関するベスト・プラクティス
- [x] `transaction-management.md` — コミット頻度、セーブポイント、自律型トランザクション
- [x] `locking-concurrency.md` — 行ロック、デッドロック、NOWAIT/SKIP LOCKED
- [x] `sequences-identity.md` — シーケンス・キャッシュ、アイデンティティ列、UUIDの代替案
- [x] `json-in-oracle.md` — JSONデータ型、JSON_TABLE、ドット表記、索引
- [x] `xml-in-oracle.md` — XMLType、XQuery、XML索引
- [x] `spatial-data.md` — SDO_GEOMETRY、空間索引、クエリ・パターン
- [x] `oracle-text.md` — 全文検索、CONTEXT索引、CONTAINSクエリ

## セキュリティ
- [x] `privilege-management.md` — 最小権限、ロール、システム権限 vs オブジェクト権限
- [x] `row-level-security.md` — VPD/FGAC、RLSポリシー、アプリケーション・コンテキスト
- [x] `data-masking.md` — Oracle Data Masking、リダクション・ポリシー
- [x] `auditing.md` — 統合監査、ファイングレイン監査、監査ポリシー
- [x] `encryption.md` — TDE、列の暗号化、ウォレット管理
- [x] `network-security.md` — SSL/TLS、ネットワーク・アクセスのためのACL

## データベース管理
- [x] `backup-recovery.md` — RMAN戦略、バックアップ・セット、リカバリ・シナリオ
- [x] `dataguard.md` — スタンバイ設定、スイッチオーバー、フェイルオーバー、ラグ監視
- [x] `rman-basics.md` — 一般的なRMANコマンド、増分バックアップ、カタログ
- [x] `undo-management.md` — UNDOリテンション、ORA-01555防止、サイジング
- [x] `redo-log-management.md` — ログ・サイジング、アーカイブログ・モード、ログ・スイッチ
- [x] `user-management.md` — ユーザー作成、プロファイル、パスワード・ポリシー

## モニタリングと診断
- [x] `alert-log-analysis.md` — 一般的なエラー、パターン、ORA-エラー・リファレンス
- [x] `adrci-usage.md` — ADRリポジトリ、インシデント調査
- [x] `health-monitor.md` — データベース・ヘルスチェック、DBMS_HM、アドバイザ
- [x] `space-management.md` — セグメント・アドバイザ、領域の再利用、HWM
- [x] `top-sql-queries.md` — 高負荷SQLの特定、診断用V$ビュー

## アーキテクチャとインフラストラクチャ
- [x] `rac-concepts.md` — Real Application Clusters、インターコネクト、サービス
- [x] `multitenant.md` — CDB/PDBアーキテクチャ、プラガブル・データベース、クローニング
- [x] `oracle-cloud-oci.md` — ATP、ADW、Exadata Cloud、クラウド固有の機能
- [x] `exadata-features.md` — スマート・スキャン、ストレージ索引、HCC（ハイブリッド列圧縮）
- [x] `inmemory-column-store.md` — インメモリー・オプション、移入、分析クエリ

## DevOpsとCI/CD
- [x] `schema-migrations.md` — Liquibase、Flyway、オンライン再定義
- [x] `online-operations.md` — DBMS_REDEFINITION、オンライン索引再構築、エディション・ベースの再定義
- [x] `edition-based-redefinition.md` — ゼロダウンタイム・デプロイメントのためのEBR
- [x] `database-testing.md` — utPLSQL、テスト・データ管理、PL/SQL用TDD
- [x] `version-control-sql.md` — スキーマ・オブジェクトのソース管理、DDL抽出

## Oracleへの移行
- [x] `migrate-postgres-to-oracle.md` — データ型のマッピング、SQL方言の違い、シーケンス/シリアル変換、psql vs sqlplus
- [x] `migrate-mysql-to-oracle.md` — AUTO_INCREMENT、データ型、ストアド・プロシージャ構文、LIMIT/OFFSETからROWNUM/FETCHへの変換
- [x] `migrate-redshift-to-oracle.md` — 分散/ソート・キー、Redshift SQLの癖、COPYコマンドの代替、カラムナから行ベースへの考慮事項
- [x] `migrate-sqlserver-to-oracle.md` — T-SQLからPL/SQL、アイデンティティ列、TOPからFETCH、リンク・サーバーからDBリンク
- [x] `migrate-db2-to-oracle.md` — DB2 SQL方言、REORG代替、パッケージの概念
- [x] `migrate-sqlite-to-oracle.md` — 型アフィニティ・マッピング、軽量からエンタープライズ・パターンへの移行
- [x] `migrate-mongodb-to-oracle.md` — ドキュメント・モデルからリレーショナル/JSON Dualityへの変換、集計パイプラインからSQL
- [x] `migrate-snowflake-to-oracle.md` — ウェアハウス/スキーマのマッピング、Snowflake SQL方言、半構造化データ
- [x] `migrate-teradata-to-oracle.md` — Teradata SQL方言、BTEQスクリプト、マルチセット・テーブル、パーティショニングの違い
- [x] `migrate-sybase-to-oracle.md` — ASEからOracleへの型マッピング、Sybaseストアド・プロシージャの変換
- [x] `oracle-migration-tools.md` — SQL Developer移行ワークベンチ、AWS SCT、SSMA、ora2pgツール・ガイド
- [x] `migration-assessment.md` — 移行前アセスメント、複雑さのスコアリング、リスクの特定、作業量見積もり
- [x] `migration-data-validation.md` — 行数チェック、ハッシュ比較、データ・ドリフト検出、不整合レポート・スクリプト
- [x] `migration-cutover-strategy.md` — 切り替え計画、並行稼働、Go/No-Go基準、ロールバック計画

## PL/SQL開発
- [x] `plsql-package-design.md` — パッケージ・アーキテクチャ、仕様部 vs 本体、公開/非公開API、凝集度、初期化ブロック
- [x] `plsql-error-handling.md` — 例外階層、名前付き例外、PRAGMA EXCEPTION_INIT、SQLERRM/DBMS_UTILITY.FORMAT_ERROR_BACKTRACE、エラー・ログ記録パターン、再レイズ
- [x] `plsql-performance.md` — コンテキスト・スイッチの最小化、BULK COLLECT/FORALL、パイプライン関数、NOCOPY、結果キャッシュ (RESULT_CACHE)、確定（DETERMINISTIC）関数
- [x] `plsql-collections.md` — 結合配列、ネストした表、VARRAY — 宣言、メソッド (COUNT, FIRST, LAST, NEXT, DELETE)、一括操作、TABLE() 関数
- [x] `plsql-cursors.md` — 暗黙的 vs 明示的カーソル、カーソルFORループ、パラメータ化されたカーソル、REF CURSORs (強定型/弱定型)、SYS_REFCURSOR、呼び出し境界を越えるカーソル変数
- [x] `plsql-dynamic-sql.md` — EXECUTE IMMEDIATE、DBMS_SQL、動的SQLでのバインド変数、動的DDLパターン、インジェクション回避
- [x] `plsql-security.md` — AUTHID CURRENT_USER vs DEFINER権限、PL/SQLでのSQLインジェクション、DBMS_ASSERT、セキュア・コーディング・チェックリスト
- [x] `plsql-debugging.md` — DBMS_OUTPUT、DBMS_APPLICATION_INFO、SQL Developerデバッガ、コンパイル警告 (PLSQL_WARNINGS)、実行時エラーのトレース
- [x] `plsql-unit-testing.md` — utPLSQLフレームワーク、テスト・パッケージの執筆、アサーション、依存関係のモック、CI統合、コード・カバレッジ
- [x] `plsql-patterns.md` — 行パイプライン、ログ記録のための自律型トランザクション、表API (TAPI) パターン、PL/SQLでのオブジェクト型、PL/SQLレコード
- [x] `plsql-compiler-options.md` — PLSQL_OPTIMIZE_LEVEL、PLSQL_CODE_TYPE (ネイティブ vs 解釈実行)、条件付きコンパイル ($$PLSQL_LINE, $IF)、エディション・ベースのコンパイル
- [x] `plsql-code-quality.md` — 命名規則、コード・レビュー・チェックリスト、アンチパターンの回避 (WHEN OTHERS NULL, ハードコードされたリテラル, マジック・ナンバー)、PL/SQL Cop / Trivadisガイドラインによる静的解析

## SQLcl
- [x] `sqlcl-basics.md` — インストール、接続、基本コマンド、SQL*Plusとの違い
- [x] `sqlcl-scripting.md` — JavaScriptスクリプト・エンジン（Nashorn/GraalVM）、scriptコマンド、タスクの自動化
- [x] `sqlcl-liquibase.md` — 組み込みのLiquibase統合、lb generate-schema、lb update、SQLclからのチェンジログ
- [x] `sqlcl-formatting.md` — SETコマンド、列のフォーマッティング、出力形式（CSV, JSON, XML, INSERT, LOADER）、ANSICONSOLE
- [x] `sqlcl-ddl-generation.md` — DDLコマンド、スキーマ・オブジェクトのエクスポート、クリーンなDDL出力のためのオプション
- [x] `sqlcl-data-loading.md` — CSV取り込みのためのLOADコマンド、形式オプション、エラー処理
- [x] `sqlcl-cicd.md` — CI/CDパイプラインでのSQLclの使用、ヘッドレス/非対話モード、終了コード、ウォレット接続
- [x] `sqlcl-mcp-server.md` — SQLcl MCPサーバーのセットアップ、Oracle DBへのAIアシスタントの接続、使用可能なツール、セキュリティ上の考慮事項

## ORDS (Oracle REST Data Services)
- [x] `ords-architecture.md` — ORDSデプロイメント・モデル（スタンドアロン, Tomcat, WebLogic）、接続プール、ORDSスキーマ、リクエスト・ルーティング
- [x] `ords-installation.md` — ORDSのインストールと構成、プール構成、ATP/ADW用ウォレット設定、アップグレード
- [x] `ords-auto-rest.md` — 表/ビューのAutoREST有効化、生成されるエンドポイント・パターン、フィルタリング、ページネーション、順序付け
- [x] `ords-rest-api-design.md` — ORDS.DEFINE_MODULE/TEMPLATE/HANDLER、HTTPメソッド、バインド・パラメータ、暗黙的/明示的パラメータ
- [x] `ords-authentication.md` — OAuth2フロー（クライアント資格証明、認証コード）、権限定義、ロール・マッピング、JWT検証
- [x] `ords-pl-sql-gateway.md` — RESTからのPL/SQL呼び出し、OUTパラメータ、REF CURSORsによる結果セット、APEX_JSON、エラー処理
- [x] `ords-file-upload-download.md` — BLOBアップロード/ダウンロード・エンドポイント、マルチパート形式データ、コンテンツ・タイプ処理
- [x] `ords-metadata-catalog.md` — OpenAPI/Swaggerドキュメント生成、メタデータ・エンドポイント、APIのドキュメント化
- [x] `ords-security.md` — エンドポイントの保護、HTTPS、CORS構成、レート制限、許可されたオリジン、権限チェック
- [x] `ords-monitoring.md` — ORDSログ、リクエスト・ログ記録、パフォーマンス・チューニング、接続プール監視、エラー診断

## Oracle固有の機能
- [x] `advanced-queuing.md` — AQ/TQ メッセージング、プロデューサ/コンシューマ
- [x] `dbms-scheduler.md` — ジョブ・スケジューリング、チェーン、イベント・ベースのジョブ
- [x] `virtual-columns.md` — 仮想列、関数ベース・パターン
- [x] `materialized-views.md` — MVリフレッシュ戦略、クエリ・リライト
- [x] `database-links.md` — DBリンク、分散クエリ、リスク
- [x] `oracle-apex.md` — Oracle上でのローコード・アプリケーション開発
