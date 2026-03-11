# OCI上のOracle Database: クラウド・サービス・リファレンス

## 概要

Oracle Cloud Infrastructure (OCI) は、完全に自己管理型の仮想マシン・データベースから、完全に自律した「自動運転」型のデータベースまで、幅広いデータベース・サービスを提供している。ワークロードにどのサービス・ティアが適しているかを判断するには、それぞれのオプションにおける制御、自動化、コスト、およびパフォーマンスのトレードオフを理解する必要がある。

OCI 上の主要な Oracle Database サービス・ファミリーは以下の3つである：

| サービス・ファミリー | 管理レベル | 最適な用途 |
|---|---|---|
| Autonomous Database (ATP/ADW) | 完全管理 + 自己チューニング | 新規アプリ、分析、開発/テスト、コスト重視 |
| Base Database Service (DBCS) | インフラを管理、DB は自社管理 | リフト＆シフト、既存アプリ、完全な DBA 権限 |
| Exadata Cloud Service (ExaCS) | インフラを管理、DB は自社管理 | ミッションクリティカルな OLTP、大規模な統合 |

---

## 1. Autonomous Transaction Processing (ATP)

ATP（Autonomous Transaction Processing）は、Oracle の完全管理型 OLTP データベース・サービスである。Oracle がインフラ、パッチ適用、バックアップ、チューニング、および可用性を管理する。顧客はスキーマ、SQL、およびアプリケーションの接続のみを管理する。

### 主な特徴

- **ワークロード・タイプ:** OLTP と運用レポートの混在
- **ストレージ形式:** デフォルトは行ストア。適格なオブジェクトに対してはインメモリー列ストアが自動的に有効化される
- **コンピュート・モデル:** OCPU ベース (1 OCPU = 2 vCPU)。OCPU 時間単位で課金
- **接続セキュリティ:** デフォルトで mTLS を強制。ウォレットベースの接続
- **Oracle バージョン:** 最新の Oracle 19c (以上)。四半期ごとに自動パッチ適用
- **並行性モデル:** 3 つの事前構成済みサービス：`_tp` (低レイテンシ)、`_tpurgent` (最高優先順位)、`_low` (バックグラウンド)

### ATP への接続

```sql
-- ATP は認証に Oracle ウォレット (mTLS) を必要とする
-- OCI コンソール、または OCI CLI を使用してウォレット (インスタンス・ウォレットまたはリージョン・ウォレット) をダウンロードする
-- oci db autonomous-database generate-wallet --autonomous-database-id <ocid> --file wallet.zip --password WalletPass#1

-- ウォレット・ディレクトリを指す TNS_ADMIN を使用した sqlplus 接続
-- export TNS_ADMIN=/home/oracle/wallet
-- sqlplus admin/YourPass#1@myatp_tp

-- アプリケーション用の JDBC 接続文字列
-- jdbc:oracle:thin:@myatp_tp?TNS_ADMIN=/wallet_dir

-- ATP で利用可能なサービス名の確認
SELECT name, network_name, goal
FROM   v$active_services
WHERE  name NOT IN ('SYS$BACKGROUND', 'SYS$USERS')
ORDER  BY name;
```

### ATP 固有の機能

```sql
-- Autonomous Database はパフォーマンス索引を自動的に作成する
-- 自動作成された索引の表示
SELECT index_name, table_name, visibility, status, auto
FROM   user_indexes
WHERE  auto = 'YES'
ORDER  BY table_name;

-- 自動パーティショニング: ATP はパフォーマンス向上のため、表を自動的にパーティション化できる
-- 自動パーティショニングの候補とステータスの確認
SELECT table_name, partition_count, strategy, status, last_analyzed
FROM   dba_auto_partition_config
ORDER  BY table_name;

-- 自動パーティショニング履歴の表示
SELECT report_date, table_owner, table_name, partition_count, status
FROM   dba_auto_partition_history
ORDER  BY report_date DESC;

-- ATP の機械学習 (ML) 機能
-- Oracle ML はインストール済み
SELECT *
FROM   user_mining_models;
```

---

## 2. Autonomous Data Warehouse (ADW)

ADW（Autonomous Data Warehouse）は、Oracle の完全管理型分析用データベース・サービスである。パラレル・クエリ実行、列形式ストレージ、および BI/レポート・ワークロード向けに構築されている。

### ATP との主な違い

| 機能 | ATP | ADW |
|---|---|---|
| 主なワークロード | OLTP、混在 | 分析、DW、レポート |
| デフォルトの並列度 | 低 (OLTP 向け) | 高 (自動パラレル・クエリ) |
| デフォルトのインメモリー | 選択的 | 積極的 (適格なオブジェクトに対して) |
| 自動索引作成 | 有効 | 無効 (分析では全表スキャンを優先) |
| デフォルトの圧縮 | アドバンスト行圧縮 | HCC Query High |
| デフォルトのサービス | `_tp` | `_high`, `_medium`, `_low` |

### ADW の接続サービス

ADW は、リソース・プロファイルの異なる 3 つの事前定義済みサービスを提供している：

| サービス | 並列度 | 優先順位 | ユースケース |
|---|---|---|---|
| `_high` | 最大並列度 (DOP) | 最高 | 単一のクリティカルなクエリ |
| `_medium` | 中程度の並列度 | 中 | 標準的な BI/レポート |
| `_low` | 最小限 | 最低 | ETL、データ・ロード、バックグラウンド |

```sql
-- ワークロード・タイプに適したサービスを使用して接続する
-- BI ツールの接続用: @myinstance_medium
-- ETL ロード用: @myinstance_low
-- アドホックなクリティカル・クエリ用: @myinstance_high

-- 現在のリソース・コンシューマ・グループの割り当てを確認
SELECT username, resource_consumer_group
FROM   v$session
WHERE  type = 'USER'
ORDER  BY username;

-- ADW は新しい表に HCC 圧縮を自動的に適用する
-- ADW 表の圧縮状態を確認
SELECT table_name, compression, compress_for
FROM   user_tables
ORDER  BY table_name;
```

---

## 3. 自動スケーリングと自動バックアップ

### 自動スケーリング (コンピュート)

Autonomous Database は、CPU に負荷がかかった場合にダウンタイムなしでコンピュート・リソースを自動的にスケーリング（増減）することをサポートしている。

```sql
-- スケーリングは OCI コンソール、CLI、または REST API 経由で管理される
-- サービスを有効化する OCI CLI 例:
-- oci db autonomous-database update \
--     --autonomous-database-id <ocid> \
--     --is-auto-scaling-enabled true

-- スケーリング動作を理解するために CPU 使用率を監視する
SELECT end_interval_time,
       ROUND(AVG(value), 2) AS avg_cpu_pct
FROM   dba_hist_sysmetric_summary
WHERE  metric_name = 'CPU Usage Per Sec'
  AND  end_interval_time >= SYSTIMESTAMP - INTERVAL '24' HOUR
GROUP  BY end_interval_time
ORDER  BY end_interval_time DESC
FETCH  FIRST 48 ROWS ONLY;

-- Autonomous Database は、ベース CPU 数からその 3 倍まで自動的にスケールする
-- 現在の OCPU 割り当ては以下で確認可能:
SELECT name, value
FROM   v$parameter
WHERE  name IN ('cpu_count', 'parallel_threads_per_cpu')
ORDER  BY name;
```

### 自動バックアップ

Autonomous Database は、デフォルトで 60 日間の保持期間を持ち、OCI オブジェクト・ストレージ（Object Storage）への自動日次バックアップを実行する。

```sql
-- OCI CLI: Autonomous Database で利用可能なバックアップのリストを表示
-- oci db autonomous-database-backup list \
--     --autonomous-database-id <ocid>

-- OCI CLI: 特定の時点にリストア
-- oci db autonomous-database restore \
--     --autonomous-database-id <ocid> \
--     --timestamp "2026-03-01T12:00:00.000Z"

-- データベース内からのバックアップ履歴の表示
SELECT input_type, status, start_time, end_time,
       input_bytes / 1024 / 1024 / 1024 AS input_gb,
       output_bytes / 1024 / 1024 / 1024 AS output_gb
FROM   v$rman_backup_job_details
ORDER  BY start_time DESC
FETCH  FIRST 10 ROWS ONLY;
```

---

## 4. Base Database Service (DBCS)

Base Database Service は、OCI コンピュート・シェイプ（仮想マシンまたはベア・メタル）上に Oracle Database をプロビジョニングする。Oracle が基盤となるインフラ（OS のパッチ適用、ストレージ・プロビジョニング）を管理し、DBA はデータベースに対して完全な制御権を持つ。

### 仮想マシン (VM) DB システム vs ベア・メタル (BM) DB システム

| 項目 | VM DB システム | BM DB システム |
|---|---|---|
| コンピュート | 共有または専用 VM | 専有ベア・メタル・ノード |
| ストレージ | iSCSI ブロック・ボリューム | NVMe ローカル SSD + ブロック |
| RAC サポート | 最大 2 ノード (RAC 2ノード) | 最大 2 ノード |
| 開始サイズ | 1 OCPU | 24 OCPU 以上 |
| ユースケース | 開発、テスト、小規模本番 | 大規模 OLTP、高 I/O |

### プロビジョニング時の考慮事項

```sql
-- OCI コンソールまたは CLI 経由でプロビジョニングした後、DB 構成を確認する
SELECT name, db_unique_name, log_mode, open_mode,
       flashback_on, force_logging, platform_name
FROM   v$database;

-- DBCS でのストレージ使用量の確認 (ブロック・ボリュームは ASM ディスク・グループとして見える)
SELECT group_number, name, type, state, total_mb, free_mb,
       ROUND((total_mb - free_mb) / total_mb * 100, 1) AS pct_used
FROM   v$asm_diskgroup
ORDER  BY name;

-- Enterprise Edition High Performance 以上の DBCS にはデフォルトで Data Guard が含まれる
-- Data Guard 構成の確認
SELECT db_unique_name, role, open_mode, protection_mode, protection_level
FROM   v$database;
```

### DBCS 固有の運用

```sql
-- アーカイブログ・モードの有効化または確認 (バックアップと Data Guard に必要)
ARCHIVE LOG LIST;

-- DBCS での RMAN バックアップ (Oracle が OCI オブジェクト・ストレージまたはローカルへのバックアップを管理)
-- OCI バックアップ・プラグイン (bkup_api) は DBCS にプリインストールされている
-- OCI オブジェクト・ストレージへの手動 RMAN バックアップ例:
-- RMAN> CONFIGURE CHANNEL DEVICE TYPE SBT
--   PARMS='SBT_LIBRARY=/opt/oracle/dcs/commonstore/pkgrepos/oss/odbcs/libopc.so
--          ENV=(OPC_PFILE=/opt/oracle/dcs/commonstore/objectstore/config/opctest.ora)';
-- RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- DBCS のパッチ適用には、DBA コンソール（OCI コンソール）または dbaascli ユーティリティを使用する
-- dbaascli dbpatch apply --db MYDB --patch_id <patch_id>
```

---

## 5. Exadata Cloud Service (ExaCS)

ExaCS は、スマート・スキャン、ストレージ索引、および HCC を含む Exadata ハードウェア・プラットフォームの全機能を OCI に提供する。Oracle が Exadata ハードウェアと Grid Infrastructure を管理し、顧客はデータベースを管理する。

### ExaCS インフラストラクチャ・オプション

| オプション | 説明 |
|---|---|
| Quarter Rack (1/4 ラック) | 2 つの DB サーバー, 3 つのストレージ・セル |
| Half Rack (1/2 ラック) | 4 つの DB サーバー, 6 つのストレージ・セル |
| Full Rack (フルラック) | 8 つの DB サーバー, 12 つのストレージ・セル |
| 弾力的な構成 (X9M 以降) | DB サーバー数とストレージ・セル数を個別に選択可能 |

### ExaCS vs ExaDB-C@C (Exadata Cloud at Customer)

- **ExaCS**: Exadata ハードウェアは OCI データセンターに配置される。顧客が DB を管理し、Oracle が Exadata インフラを管理する。
- **ExaDB-C@C**: Exadata ハードウェアは顧客のオンプレミス・データセンターに設置される。Oracle がインフラをリモート管理し、顧客が DB を管理する。

```sql
-- Exadata 機能がアクティブであることを確認
SELECT name, value
FROM   v$parameter
WHERE  name IN ('cell_offload_processing',
                'cell_offload_compaction',
                'cell_offload_plan_display',
                'enable_goldengate_replication')
ORDER  BY name;

-- スマート・スキャンが使用されていることの確認
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell physical IO interconnect bytes',
    'cell physical IO interconnect bytes returned by smart scan',
    'cell scans'
)
ORDER  BY name;
```

---

## 6. OCI 接続方法

### 標準接続 (Autonomous 以外)

```sql
-- DBCS への標準的な JDBC 接続
-- jdbc:oracle:thin:@//host:1521/service_name

-- TNS ベースの接続例
-- HOST_PORT_SN =
--   (DESCRIPTION =
--     (ADDRESS = (PROTOCOL = TCP)(HOST = mydbcs-host.subnet.vcn.oraclevcn.com)(PORT = 1521))
--     (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = mydb.subnet.vcn.oraclevcn.com))
--   )

-- DBCS でリスニングされているサービスの確認
SELECT name, network_name
FROM   v$active_services
WHERE  name NOT IN ('SYS$BACKGROUND', 'SYS$USERS')
ORDER  BY name;
```

### ウォレットベースの接続 (Autonomous Database)

```sql
-- Autonomous Database には 3 つのウォレット・タイプがある：
-- 1. インスタンス・ウォレット: 特定の DB インスタンスに固有
-- 2. リージョン・ウォレット: そのリージョン内の任意の Autonomous DB で動作
-- 3. mTLS (相互 TLS): 双方向の証明書認証

-- TLS のみの接続 (ウォレット不要、21c 以降の ADB)
-- TLS のみの接続を可能にするためのウォレット要件の無効化例:
-- oci db autonomous-database update \
--     --autonomous-database-id <ocid> \
--     --is-mtls-connection-required false

-- JDBC Easy Connect Plus 構文 (ウォレットなし、TLS のみ)
-- jdbc:oracle:thin:@myatp.adb.us-ashburn-1.oraclecloud.com:1522/dbname_tp.adb.oraclecloud.com

-- TLS 設定の確認
SELECT name, value
FROM   v$parameter
WHERE  name IN ('ssl_version', 'sqlnet.encryption_server', 'sqlnet.authentication_services');
```

### プライベート・エンドポイント接続

OCI は、インターネットに公開することなく、プライベート・エンドポイントを使用して VCN 内から Autonomous Database に接続することをサポートしている：

```sql
-- プライベート・エンドポイントは、すべての接続を VCN 経由に強制する
-- プロビジョニング時に構成するか、プロビジョニング後に OCI コンソール経由で追加する

-- プライベート・エンドポイントを有効にした後、エンドポイント詳細を確認する例:
-- oci db autonomous-database get --autonomous-database-id <ocid> \
--     --query 'data.{"private-endpoint": "private-endpoint", "private-ip": "private-ip"}'

-- プライベート・エンドポイントの接続文字列には、プライベート IP またはプライベート DNS 名を使用する
-- jdbc:oracle:thin:@10.0.1.25:1521/myatp_tp
```

---

## 7. Oracle Cloud Free Tier (常に無料)

Oracle Cloud Free Tier は、1 OCPU、20GB ストレージを搭載した 2 つの Autonomous Database を時間制限なしで永久無料で提供し、さらに 30 日間有効な他のサービス用クレジットを提供している。

### Free Tier の制限

| リソース | 常に無料 (Always Free) の制限 |
|---|---|
| Autonomous DB インスタンス | 2 (ATP 1 つ、ADW 1 つ) |
| インスタンスごとの OCPU | 1 (自動スケーリングなし) |
| インスタンスごとのストレージ | 20 GB |
| APEX ワークスペース | 含まれる |
| Oracle ML ノートブック | 含まれる |
| データ・ロード | REST, APEX, DB Actions |
| バックアップ | 60 日間含まれる |

```sql
-- Free Tier の Autonomous Database への接続
-- サービス名は有料 ADB と同じパターンに従う
-- 利用可能なサービス: _tp, _tpurgent, _low, _high, _medium

-- Free Tier のリソース制約の確認
SELECT name, value
FROM   v$parameter
WHERE  name IN ('cpu_count', 'sga_max_size', 'pga_aggregate_target')
ORDER  BY name;

-- Free Tier には Oracle APEX がプリインストールされている
SELECT version, status
FROM   apex_release;

-- ORDS (Oracle REST Data Services) は Free Tier で事前有効化されている
-- ORDS 構成の確認
SELECT *
FROM   user_ords_schemas;
```

---

## 8. クラウド固有機能のサマリー

### Autonomous Database 運用コマンド (OCI CLI)

```bash
# Autonomous Database の起動 / 停止
oci db autonomous-database start  --autonomous-database-id <ocid>
oci db autonomous-database stop   --autonomous-database-id <ocid>

# OCPU のスケーリング (ダウンタイムなし)
oci db autonomous-database update \
    --autonomous-database-id <ocid> \
    --cpu-core-count 8

# ストレージのスケーリング (ダウンタイムなし、増加のみ可能)
oci db autonomous-database update \
    --autonomous-database-id <ocid> \
    --data-storage-size-in-tbs 2

# Autonomous Database のクローン作成
oci db autonomous-database create-from-clone \
    --clone-type FULL \
    --source-id <source_ocid> \
    --display-name MyATP_Clone \
    --db-name MYATPCLONE \
    --admin-password "ClonePass#1" \
    --compartment-id <compartment_ocid>
```

### Database Actions (旧 SQL Developer Web)

すべての Autonomous Database インスタンスには、ブラウザベースの SQL および管理用 IDE である Database Actions（旧 SQL Developer Web）が含まれており、以下からアクセスできる：

```
https://<adb_host>/ords/sql-developer
```

主な Database Actions モジュール：
- SQL Worksheet — インタラクティブな SQL 実行
- データ・ロード — CSV/JSON/Parquet を表に直接アップロード
- データ・スタジオ — ビジネス・インテリジェンス、データ・インサイト
- Oracle ML — Jupyter 互換の ML ノートブック
- Oracle APEX — フル機能のアプリケーション開発プラットフォーム

---

## 9. ベスト・プラクティス

- **すべての本番用 Autonomous Database にはプライベート・エンドポイントを使用する。** パブリック・エンドポイントはデータベースをインターネットにさらすことになる。プライベート・エンドポイントはアクセスを VCN 内に制限し、IP 許可リスト（Allowlisting）の必要性をなくす。
- **ADW への BI ツール接続には `_medium` サービスを使用する。** `_high` サービスは最大並列度を使用するため、単一のクエリには適しているが、多数の BI ユーザーが同時にアクティブな場合には競合を引き起こす可能性がある。
- **初期デプロイ時に自動スケーリングを有効にし、負荷テスト後にのみ無効にする。** 本番環境での負荷急増による接続タイムアウトが発生した後に気づくよりも、テスト中に自動スケーリングが必要であることを発見する方がはるかに良い。
- **ウォレット・ファイルを安全に保管する。** Autonomous Database のウォレットには秘密鍵が含まれている。ウォレット・ファイルをソース管理にコミットしてはいけない。CI/CD パイプラインでのシークレット管理には OCI Vault を使用すること。
- **Autonomous への移行前に DBCS のサイズを適切に調整する。** 同じデータ量とワークロード・プロファイルを使用して。ATP 上でアプリケーションをテストすること。Autonomous の自動最適化によってクエリ計画が変わる可能性があるため、移行後に実行計画を検証すること。
- **Autonomous Database のセキュリティ・ポスチャ管理（セキュリティ態勢管理）には Data Safe を活用する。** Data Safe（ADB に含まれる）は、追加の構成なしで、セキュリティ・アセスメント、ユーザー・アセスメント、データ・マスキング、およびアクティビティ監査を提供する。

---

## 10. よくある間違いとその回避方法

### 間違い 1: OLTP アプリケーションから ATP の `_high` サービスに接続する

ATP の `_high` サービスは最大並列度を使用するため、負荷の軽い短い OLTP トランザクションに対してパラレル・クエリのオーバーヘッドとリソース競合を引き起こす。OLTP には `_tp`（低レイテンシ、並列処理なし）を、優先順位の高いトランザクションには `_tpurgent` を使用すること。

### 間違い 2: セキュリティ・イベント後にウォレットをローテーションしない

ウォレットは自動的には期限切れにならない。セキュリティ・インシデントや担当者の変更後は、新しいウォレットをダウンロードしてデプロイすること。古いウォレットは、データベースのパスワードを変更（ローテーション）することで明示的に無効化されるまで有効なままである。

```bash
# ADMIN パスワードを変更する (次回ダウンロード時に古いウォレットが無効になる)
oci db autonomous-database update \
    --autonomous-database-id <ocid> \
    --admin-password "NewSecurePass#2"
# その後、ウォレットを再ダウンロードする
oci db autonomous-database generate-wallet \
    --autonomous-database-id <ocid> \
    --file new_wallet.zip \
    --password "WalletPass#2"
```

### 間違い 3: DBCS のパッチが自動的に適用されると思い込む

Autonomous Database とは異なり、DBCS のパッチは**自動的には適用されない**。DBA は OCI コンソール（ワンクリックでのパッチ適用）または `dbaascli` を通じて、四半期ごとの DB パッチを手動で適用する必要がある。パッチが適用されていない DBCS インスタンスには、時間の経過とともに脆弱性（CVE）が蓄積される。

### 間違い 4: カスタム初期化パラメータが必要なワークロードに Autonomous Database を使用する

ATP および ADW では、ほとんどの `init.ora` パラメータを変更できない。特定の `optimizer_features_enable` やカスタム `event` 設定、あるいは標準外のメモリー・パラメータを必要とするワークロードは、Autonomous Database には適していない。そのようなワークロードには DBCS または ExaCS を使用すること。

### 間違い 5: データ・ロード時のエグレス・コストを無視する

Autonomous Database からオンプレミスに大規模なデータセットをダウンロードすると、OCI のエグレス（送信データ転送）料金が発生する。中継ステージとして OCI オブジェクト・ストレージを使用し（同じリージョン内の ADB と オブジェクト・ストレージ間のエグレスは無料）、その後オブジェクト・ストレージからオンプレミスに転送するように検討すること。

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Autonomous Database Documentation](https://docs.oracle.com/en/cloud/paas/autonomous-database/) — ATP, ADW, 自動スケーリング, 自動バックアップ, ウォレット接続
- [Oracle Base Database Service Documentation](https://docs.oracle.com/en/cloud/paas/base-database/) — DBCS プロビジョニング, VM vs BM, パッチ適用
- [Oracle Exadata Cloud Service Documentation](https://docs.oracle.com/en/engineered-systems/exadata-cloud-service/) — ExaCS インフラ・オプション
- [OCI CLI Reference](https://docs.oracle.com/en-us/iaas/tools/oci-cli/latest/) — autonomous-database コマンド
