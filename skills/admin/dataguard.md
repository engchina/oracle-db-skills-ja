# Oracle Data Guard

## 概要

Oracle Data Guardは、Oracleの高可用性、災害復旧、およびデータ保護ソリューションである。本番データベース（**プライマリ**）の同期されたコピーを1つ以上、**スタンバイ・データベース**として維持する。プライマリ・データベースが使用不能になった場合、スタンバイをアクティブにして引き継がせることで、ダウンタイムとデータ損失を最小限に抑えることができる。

Data GuardはOracle Database Enterprise Editionでライセンス供与される。ミッション・クリティカルなデータベースのためのOracle推奨の災害復旧ソリューションであり、Maximum Availability Architecture (MAA)のコア・コンポーネントである。

---

## フィジカル・スタンバイ vs ロジカル・スタンバイ

### フィジカル・スタンバイ

フィジカル・スタンバイは、プライマリ・データベースのブロック単位で同一のコピーである。プライマリで生成されたredoデータはスタンバイに転送され、**メディア・リカバリ** (Redo Apply)を使用して適用される。スタンバイは常にリカバリ状態にある。

**特徴:**
- ブロック・レベルでプライマリとバイト単位で同一
- Redo Apply (MRP — Managed Recovery Process)を使用
- redoを適用しながら、読取り専用でオープンできる（Active Data Guard — 別途ライセンスが必要）
- すべてのデータ型およびオブジェクト型を制限なくサポート
- 構成が最も速く、保守が容易
- ほとんどの災害復旧（DR）および高可用性（HA）の展開に使用される

**使用場面:** あらゆるワークロードのDR/HA、Active Data Guardによる読取りオフロード、ローリング・アップグレード。

### ロジカル・スタンバイ

ロジカル・スタンバイは、プライマリからredoを受信し、それをSQL文に変換（解析）して、**SQL Apply** (LogMinerベース)を使用してそれらの文を適用する。スタンバイ・データベースは読取り/書込みの両方でオープンされ、プライマリには存在しない追加のオブジェクトを持つことができる。

**特徴:**
- 適用中に読取り/書込みの両方でオープン
- スタンバイ上にレポート用の追加テーブル、索引、またはスキーマを存在させることができる
- すべてのデータ型をサポートしているわけではない（例：一部のバージョンではBFILE、NCLOBに制限がある）
- 管理がより複雑である。高負荷下ではSQL ApplyがRedo Applyに遅れる可能性がある
- 適用中のデータの変換をサポート

**使用場面:** 読取り/書込みアクセスが必要なレポート用データベース、スタンバイ上のカスタム・スキーマ変更、選択的レプリケーション。

### スナップショット・スタンバイ

スナップショット・スタンバイは、テストのために一時的に読取り/書込み状態に変換されたフィジカル・スタンバイである。プライマリからのredoは引き続き受信されるが、適用はされない。フィジカル・スタンバイに戻すと、乖離した変更は破棄され、リカバリが再開される。

```sql
-- フィジカル・スタンバイをスナップショット・スタンバイに変換 (DGMGRL経由)
DGMGRL> CONVERT DATABASE standby_db TO SNAPSHOT STANDBY;

-- フィジカル・スタンバイに戻す
DGMGRL> CONVERT DATABASE standby_db TO PHYSICAL STANDBY;
```

---

## Redo転送と適用

### Redo転送

プライマリ・データベースは、redoログ・データをスタンバイ先に転送する。転送は同期または非同期で行うことができる。

**SYNC（同期）:** プライマリは、コミットする前にスタンバイからの確認を待つ。データ損失はゼロだが、コミットに遅延が発生する。

**ASYNC（非同期）:** プライマリは、スタンバイの確認を待たずにコミットする。パフォーマンスは向上するが、転送ラグに相当するデータ損失の可能性がある。

```sql
-- プライマリでのredo転送の構成 (例: LOG_ARCHIVE_DEST_2)
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2 =
  'SERVICE=standby_tns ASYNC NOAFFIRM
   DB_UNIQUE_NAME=standby_db
   VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)
   COMPRESSION=ENABLE'
  SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2 = ENABLE SCOPE=BOTH;
```

### Redo Apply (フィジカル・スタンバイ)

Managed Recovery Process (MRP)は、アーカイブされたredoログまたはスタンバイredoログからのredoを適用する（リアルタイム適用）。

```sql
-- 管理リカバリの開始 (フィジカル・スタンバイ側)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;

-- リアルタイム適用の開始 (アーカイブされる前に、到着したredoを適用)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;

-- 管理リカバリの停止
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

-- 適用ステータスの確認
SELECT process, status, sequence#, thread#
FROM v$managed_standby
ORDER BY process;
```

### SQL Apply (ロジカル・スタンバイ)

```sql
-- ロジカル・スタンバイでSQL Applyを開始
ALTER DATABASE START LOGICAL STANDBY APPLY IMMEDIATE;

-- SQL Applyを停止
ALTER DATABASE STOP LOGICAL STANDBY APPLY;

-- SQL Applyステータスの確認
SELECT status, applied_scn, latest_scn
FROM dba_logstdby_progress;
```

### スタンバイRedoログ

スタンバイRedoログ (SRL)は、リアルタイム適用および同期redo転送に必要である。プライマリの現在のオンラインredoログから（アーカイブ・ログに加えて）redoを受信する。

```sql
-- スタンバイRedoログ・グループの追加 (スタンバイ側。プライマリのグループ数 + 1を推奨)
-- 各グループはプライマリのオンラインredoログと同じサイズにする必要がある
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4
  ('/oradata/standby/stdby_redo04.log') SIZE 500M;

ALTER DATABASE ADD STANDBY LOGFILE GROUP 5
  ('/oradata/standby/stdby_redo05.log') SIZE 500M;

-- スタンバイRedoログの表示
SELECT group#, members, bytes/1048576 size_mb, status
FROM v$standby_log;
```

---

## Data Guard Broker (DGMGRL)

Data Guard Brokerは、Data Guard構成を管理するためのフレームワークである。構成、監視、およびロール遷移を自動化し、一元化する。手動でのData Guard管理よりも、Brokerの使用が強く推奨される。

### Brokerの有効化

```sql
-- プライマリとスタンバイの両方で有効化
ALTER SYSTEM SET dg_broker_start = TRUE SCOPE=BOTH;
```

### Broker構成の作成

```bash
# DGMGRLに接続 (プライマリまたはネットワーク・アクセスのあるホストから)
dgmgrl sys/<password>@primary_db

DGMGRL> CREATE CONFIGURATION 'my_dg_config'
          AS PRIMARY DATABASE IS primary_db
          CONNECT IDENTIFIER IS primary_tns;

DGMGRL> ADD DATABASE standby_db
          AS CONNECT IDENTIFIER IS standby_tns
          MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;
```

### 一般的なDGMGRLコマンド

```bash
# 構成と健全性の全体を表示
DGMGRL> SHOW CONFIGURATION;

# 特定のデータベースの詳細を表示
DGMGRL> SHOW DATABASE VERBOSE standby_db;

# 現在のラグを表示
DGMGRL> SHOW DATABASE standby_db 'ApplyLag';
DGMGRL> SHOW DATABASE standby_db 'TransportLag';

# プロパティの編集
DGMGRL> EDIT DATABASE standby_db SET PROPERTY LogXptMode='ASYNC';
DGMGRL> EDIT DATABASE primary_db SET PROPERTY RedoRoutes='(LOCAL : standby_db ASYNC)';

# 構成の検証
DGMGRL> VALIDATE DATABASE standby_db;
DGMGRL> VALIDATE DATABASE VERBOSE standby_db;
```

---

## スイッチオーバー vs フェイルオーバー

### スイッチオーバー

**スイッチオーバー**は、計画的な正常なロール交代である。両方のデータベースが維持され、データは失われない。計画的なメンテナンス、パッチ適用、またはテストに使用される。

**イベントの流れ:**
1. プライマリがスタンバイ・ロールに移行する（redoをフラッシュし、新しい接続を防止する）
2. スタンバイがプライマリ・ロールに移行する
3. 両方のデータベースが新しいロールで稼働する

```bash
# スイッチオーバー前に準備状況を検証
DGMGRL> VALIDATE DATABASE standby_db;

# スイッチオーバーの実行 (Brokerが両側の処理を自動的に行う)
DGMGRL> SWITCHOVER TO standby_db;

# 新しい構成を確認
DGMGRL> SHOW CONFIGURATION;
```

**手動スイッチオーバー (Brokerなし):**
```sql
-- プライマリ側: スイッチオーバーの開始
ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY WITH SESSION SHUTDOWN;

-- スタンバイ側: スイッチオーバーを完了してプライマリになる
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
ALTER DATABASE OPEN;
```

### フェイルオーバー

**フェイルオーバー**は、プライマリ・データベースが使用不能になった場合、または故障してすぐに復旧できない場合の緊急操作である。Maximum ProtectionまたはMaximum Availabilityモードで同期redo転送を使用し、すべてのredoを受信していない限り、データ損失の可能性がある。

**フェイルオーバーは、スタンバイを新しいプライマリとして永久にアクティブにする。** 旧プライマリは、スタンバイとして再有効化（リーンステート）しない限り使用できない。

```bash
# Brokerを使用したフェイルオーバー (推奨)
DGMGRL> FAILOVER TO standby_db;

# データ損失防止を試みる場合 (すべてのredoを待機)
DGMGRL> FAILOVER TO standby_db IMMEDIATE;
```

**手動フェイルオーバー (Brokerなし):**
```sql
-- スタンバイ側: 残りのアーカイブ・ログをリカバリしてアクティブ化
RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH;
ALTER DATABASE ACTIVATE PHYSICAL STANDBY DATABASE;
ALTER DATABASE OPEN;
```

### 旧プライマリの再有効化 (Reinstate)

フェイルオーバー後、旧プライマリをスタンバイとして再有効化できる：

```bash
DGMGRL> REINSTATE DATABASE old_primary_db;
```

これは、旧プライマリでFlashback Databaseを使用してフェイルオーバー・ポイントまでロールバックし、新しいプライマリと再同期することで行われる。

---

## ラグの監視

### 転送ラグと適用ラグ

- **転送ラグ (Transport Lag):** スタンバイがプライマリからredoを受信するのにどれだけ遅れているか
- **適用ラグ (Apply Lag):** スタンバイが受信したredoを適用するのにどれだけ遅れているか

```sql
-- スタンバイ・データベースからラグを表示
SELECT name, value, time_computed, datum_time
FROM v$dataguard_stats
WHERE name IN ('transport lag', 'apply lag', 'apply finish time');

-- プライマリから表示 (DBA_LOGSTDBY_LOG または V$ARCHIVE_DEST_STATUS が必要)
SELECT dest_id, dest_name, status, archived_seq#, applied_seq#,
       gap_status
FROM v$archive_dest_status
WHERE target = 'STANDBY';

-- アーカイブ・ギャップの確認
SELECT thread#, low_sequence#, high_sequence#
FROM v$archive_gap;
```

### DGMGRL経由の監視

```bash
DGMGRL> SHOW DATABASE standby_db 'ApplyLag';
DGMGRL> SHOW DATABASE standby_db 'TransportLag';
DGMGRL> SHOW DATABASE standby_db 'RecvQEntries';
DGMGRL> SHOW DATABASE standby_db 'SendQEntries';
```

### Enterprise Manager経由の監視

Enterprise ManagerのData Guard管理ページには、ラグのグラフ、構成トポロジー、およびアラートしきい値が表示される。自動通知の場合は、`ApplyLag`および`TransportLag`にEMメトリックのしきい値を構成する。

---

## Active Data Guard (読取りオフロード)

Active Data Guard (ADG)を使用すると、プライマリからのredoを適用しながら、フィジカル・スタンバイ・データベースを**読取り専用**でオープンでき、さらにActive Data Guardライセンスが必要である。

**ユースケース:**
- プライマリからレポート・クエリをオフロードする
- バックアップをオフロードする（本番ではなく、スタンバイからバックアップを取得する）
- Far Syncインスタンスを使用して、読取りワークロードをグローバルに分散する

### ADGを使用したフィジカル・スタンバイの読取り専用オープン

```sql
-- スタンバイ側: redoを適用し続けながら読取り専用でオープン
ALTER DATABASE OPEN READ ONLY;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
  USING CURRENT LOGFILE DISCONNECT FROM SESSION;

-- オープン中に適用されていることを確認
SELECT open_mode, db_unique_name FROM v$database;
-- open_mode が次であること: READ ONLY WITH APPLY
```

### Far Syncインスタンス

Far Syncインスタンスは、プライマリから同期的にredoを受信し、それをリモート制御スタンバイに非同期に転送するために、地理的にスタンバイの近くに配置される軽量なOracleインスタンス（データファイルなし）である。これにより、短距離（低遅延）での同期転送を実現しながらも、地理的に離れたスタンバイを保護することができる。

```bash
# Broker構成にfar syncインスタンスを追加
DGMGRL> ADD FAR_SYNC farsync_inst AS CONNECT IDENTIFIER IS farsync_tns;
DGMGRL> EDIT DATABASE primary_db SET PROPERTY RedoRoutes =
          '(LOCAL : farsync_inst SYNC)(farsync_inst : standby_db ASYNC)';
DGMGRL> ENABLE FAR_SYNC farsync_inst;
```

---

## 保護モード

Data Guardの保護モードは、データ保護（データ損失ゼロ）とプライマリ・データベースのパフォーマンス/可用性のトレードオフを定義する。

### 最高保護 (Maximum Protection)

- 少なくとも1つのスタンバイへの同期redo転送 (SYNC AFFIRM)が必要
- 同期されたスタンバイが使用できない場合、プライマリがシャットダウンする
- データ損失ゼロを保証
- スタンバイへの往復時間（RTT）に相当するコミット遅延が発生する

```sql
ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE PROTECTION;
```

### 最高可用性 (Maximum Availability)

- SYNC転送が必要。スタンバイが使用不可の場合、自動的に非同期に格下げされる（プライマリは停止しない）
- スタンバイに到達可能な場合はデータ損失ゼロ
- 保護と可用性の最良のバランスであり、ほとんどの本番環境で推奨される

```sql
ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE AVAILABILITY;
```

### 最高パフォーマンス (Maximum Performance — デフォルト)

- 非同期転送 (ASYNC)を使用
- プライマリはスタンバイの確認を待機しない
- 最高のパフォーマンス。転送ラグに相当するデータ損失の可能性がある
- デフォルト・モードであり、ある程度のデータ損失が許容される場合に適している

```sql
ALTER DATABASE SET STANDBY DATABASE TO MAXIMIZE PERFORMANCE;
```

---

## ベスト・プラクティス

- **Data Guard Broker (DGMGRL)を使用する。** 手動のlog_archive_destパラメータ管理は間違いやすく、保守も困難である。

- **プライマリとスタンバイの両方でスタンバイRedoログを構成する。** リアルタイム適用と同期転送には必須である。

- **最高可用性モードを使用する。** スタンバイへのネットワーク遅延が許容可能（通常はRTT 5ms未満）なクリティカルなOLTPワークロードにSYNC転送で適用する。

- **ラグをアクティブに監視する。** 4時間の適用ラグがあるスタンバイは、即時のフェイルオーバーではなく、4時間の復旧時間（RTO）を意味する。

- **スイッチオーバーを定期的にテストする**（最低でも四半期ごと）。一度も実行されたことのないフェイルオーバー・手順は、高リスクな仮定に過ぎない。

- **スタンバイからバックアップを取得する。** プライマリからのバックアップI/Oをオフロードできる。RMANはフィジカル・スタンバイからバックアップを実行可能である。

- **プライマリでFlashback Databaseを有効にする。** フェイルオーバー後の再有効化が可能になる。
  ```sql
  ALTER DATABASE FLASHBACK ON;
  ```

- **スタンバイRedoログを正しくサイズ設定する。** 各グループはプライマリのオンラインredoログと同じサイズにし、スレッドごとに（プライマリ・グループ数 + 1）個のスタンバイredoログ・グループを確保する。

---

## よくある間違いと回避策

**スタンバイRedoログの未構成**
SRLがないと、リアルタイム適用ができなくなる。redoはアーカイブ後にのみ適用されるため、適用ラグが不必要に増加する。

**LOG_ARCHIVE_DESTパラメータの構文の誤り**
`VALID_FOR`属性はデータベースの役割とログ・タイプに一致している必要がある。構成が間違っていると、redo転送が警告なしに停止する。

```sql
-- 正解: プライマリとして動作しているときにオンライン・ログを転送する
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2 =
  'SERVICE=standby ASYNC NOAFFIRM
   DB_UNIQUE_NAME=standby
   VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE)';
```

**DB_UNIQUE_NAMEの設定忘れ**
Data Guard構成内のすべてのデータベースは、一意の`DB_UNIQUE_NAME`を持つ必要がある。これらは同じ`DB_NAME`を共有できる。

```sql
-- 確認
SELECT db_unique_name, db_name FROM v$database;
```

**ギャップを確認せずにフェイルオーバーする**
手動でフェイルオーバーする前に、必ずアーカイブ・ギャップを確認すること：
```sql
SELECT * FROM v$archive_gap;
```

**プライマリでのFlashbackの未有効化**
Flashbackなしの場合、フェイルオーバー後の旧プライマリの再有効化にはバックアップからの完全な再構築が必要になる。フェイルオーバー前に必ずプライマリでFlashbackを有効にしておくこと。

**ADGライセンスなしでスタンバイを恒久的なレポート・データベースとして扱う**
スタンバイを読取り専用でオープンし、適用を一時停止することはActive Data Guardではない。データベースをオープンしたまま適用を継続するには、ADGライセンスが必要である。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Data Guard 概念および管理 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/sbydb/)
- [Oracle Database 19c SQL言語リファレンス — ALTER DATABASE (Data Guard句)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/ALTER-DATABASE.html)
- [Oracle Data Guard Broker 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/dgbkr/)
- [Oracle Database 19c リファレンス — V$DATAGUARD_STATS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DATAGUARD_STATS.html)
