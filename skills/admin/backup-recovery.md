# Oracle RMANのバックアップとリカバリ

## 概要

Recovery Manager (RMAN)は、Oracleのバックアップ、リストア、およびリカバリ操作のための主要なツールである。ブロック・レベルのバックアップ整合性チェック、圧縮、暗号化、増分バックアップを提供し、Oracleデータベース・エンジンと密接に統合されている。RMANを使用することで、手動のバックアップ・スクリプト作成が不要になり、オープン状態のデータベースでも一貫性のあるバックアップを確保できる。

バックアップとリカバリの理解は、Oracle DBAにとって最も重要なスキルである。リカバリできないデータベースは、信頼に値しない。

---

## RMANのアーキテクチャ

### コア・コンポーネント

**RMAN実行ファイル**
ターゲット・データベース、およびオプションでリカバリ・カタログに接続する`rman`バイナリ。RMANコマンドを解釈し、ターゲット・データベースのサーバー・プロセスと通信して、バックアップのメタデータを管理する。

**ターゲット・データベース**
バックアップまたはリカバリの対象となるデータベース。RMANは専用のサーバー・プロセスを使用して接続し、制御ファイルまたはリカバリ・カタログからバックアップ・メタデータを読み取る。

**リカバリ・カタログ**
RMANのメタデータ（バックアップ履歴、データファイル情報、アーカイブ・ログ履歴）を格納する、オプションだが推奨される別のOracleスキーマ。カタログがない場合、メタデータはターゲットの制御ファイルにのみ格納される。カタログを使用すると、クロス・データベース・レポート、格納デポジトリ・スクリプト、および制御ファイル単体よりも長いバックアップ履歴の保持が可能になる。

**メディア管理レイヤー (MML)**
RMANがバックアップをテープ・ライブラリに直接書き込めるようにするための、オプションのサードパーティ・インターフェース（例：Oracle Secure Backup、Veritas NetBackup、Commvault）。RMANはSBT（System Backup to Tape）チャネル・タイプを介してMMLと通信する。

**チャネル**
実際にI/Oを実行するサーバー・プロセス。各チャネルは1つのバックアップ・ストリームに対応する。RMANは、自動チャネル（`CONFIGURE`で構成）または`RUN`ブロック内で手動で割り当てられたチャネルをサポートする。

```
RMANアーキテクチャ:
┌─────────────┐       ┌──────────────────────┐
│  RMANクライアント│──────▶│  ターゲット・データベース    │
│  (rman exe) │       │  (サーバー・プロセス)       │
│             │       │  読み取り: 制御ファイル     │
└─────────────┘       └──────────────────────┘
        │                        │
        │                        ▼
        ▼               ┌─────────────────────┐
┌──────────────────┐    │ バックアップ・ピース /    │
│ リカバリ・カタログ    │    │ ディスクまたはテープ上の  │
│ (別のDB)          │    │ イメージ・コピー         │
│ RMANスキーマ      │    └─────────────────────┘
└──────────────────┘
```

---

## バックアップ・セット vs イメージ・コピー

### バックアップ・セット

バックアップ・セットはRMAN独自のバックアップ形式である。1つ以上の**バックアップ・ピース**（物理ファイル）で構成される。RMANはデータファイルから使用済みブロックを読み取り、それらをバックアップ・ピースにまとめ、デフォルトで未使用ブロックをスキップする。これにより、バックアップ・セットはイメージ・コピーよりもサイズが小さくなる。

- 圧縮をサポート（`AS COMPRESSED BACKUPSET`によるBASIC, LOW, MEDIUM, HIGH）
- 暗号化をサポート
- 増分バックアップをネイティブにサポート
- テープ（SBT）バックアップに必須
- RMANのリストアなしではOracleで直接使用できない（使用前にリストアが必要）

```sql
-- データベースの完全バックアップ・セットを作成
BACKUP DATABASE;

-- 圧縮されたバックアップ・セットを作成
BACKUP AS COMPRESSED BACKUPSET DATABASE;

-- 特定の表領域をバックアップ・セットとしてバックアップ
BACKUP TABLESPACE users;
```

### イメージ・コピー

イメージ・コピーは、データファイル、アーカイブ・ログ、または制御ファイルのビット単位のコピーであり、元のファイルと形式が同一である。Oracleで直接使用できる（例：正しい場所に配置し、リストア・ステップなしでリカバリする）。

- 高速なリカバリ：リストア・ステップが不要で、スイッチしてリカバリするだけ
- ディスク上のサイズが大きい：未使用ブロックを含むすべてのブロックをコピーする
- `RECOVER COPY`（増分バックアップを使用してイメージ・コピーをロールフォワードする）により増分更新が可能
- 変換なしではSBT経由でテープに書き込めない

```sql
-- すべてのデータファイルのイメージ・コピーを作成
BACKUP AS COPY DATABASE;

-- 特定のデータファイルのイメージ・コピーを作成
BACKUP AS COPY DATAFILE '/oradata/users01.dbf';
```

### バックアップ・セット vs イメージ・コピー：使い分け

| 要因 | バックアップ・セット | イメージ・コピー |
|---|---|---|
| ディスク領域 | 小さい（空きブロックをスキップ） | 大きい（完全なコピー） |
| バックアップ時間 | 遅い（圧縮のオーバーヘッドの可能性あり） | 速い |
| リカバリ時間 | 遅い（リストア + リカバリ） | 速い（スイッチ + リカバリ） |
| テープ・サポート | あり | なし（直接は不可） |
| 増分更新の可否 | あり（増分バックアップ） | あり（RECOVER COPY） |
| RMANなしでの直接使用 | 不可 | 可 |

---

## 増分バックアップ (レベル0とレベル1)

増分バックアップは、前回のバックアップ以降に変更されたブロックのみをコピーする。RMANは、**ブロック変更トラッキング (BCT)**ファイルを使用して変更されたブロックを追跡する。これにより、データファイル全体のスキャンを回避し、増分バックアップを劇的に高速化できる。

### レベル0

レベル0増分バックアップはベースラインであり、フル・バックアップと同様に使用済みブロックをすべてコピーするが、増分ベースラインとしてタグ付けされる。後続のレベル1バックアップはこのレベル0に対して取得される。

```sql
-- フル増分ベースライン (レベル0)
BACKUP INCREMENTAL LEVEL 0 DATABASE;
```

### レベル1

レベル1増分バックアップは、直近のレベル0またはレベル1バックアップ以降に変更されたブロックのみをコピーする。以下の2つのタイプがある：

- **差分 (Differential)**（デフォルト）：前回のレベル0またはレベル1以降に変更されたブロックをコピーする
- **累積 (Cumulative)**：前回のレベル0以降に変更されたすべてのブロックをコピーする

```sql
-- 差分増分（デフォルト） — 前回のレベル0または1以降の変更をバックアップ
BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- 累積増分 — 前回のレベル0以降のすべての変更をバックアップ
BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;
```

### 一般的な週次増分戦略

```
日曜日: BACKUP INCREMENTAL LEVEL 0 DATABASE;   -- フル・ベースライン
月曜日: BACKUP INCREMENTAL LEVEL 1 DATABASE;   -- 月曜日の変更
火曜日: BACKUP INCREMENTAL LEVEL 1 DATABASE;   -- 火曜日の変更
水曜日: BACKUP INCREMENTAL LEVEL 1 DATABASE;   -- 水曜日の変更
...
土曜日: BACKUP INCREMENTAL LEVEL 1 DATABASE;   -- 土曜日の変更
```

### ブロック変更トラッキング

増分バックアップ中のフル・データファイル・スキャンを避けるためにBCTを有効にする：

```sql
-- ブロック変更トラッキングを有効化 (Enterprise Editionが必要)
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
  USING FILE '/oradata/bct/change_tracking.bct';

-- 確認
SELECT status, filename FROM v$block_change_tracking;
```

### 増分更新バックアップ (マージ戦略)

バックアップ・コピー（イメージ・コピー）と増分バックアップを組み合わせて、常に前日の状態に保たれたイメージ・コピーを維持する強力なテクニックであり、非常に高速なリカバリが可能になる。

```sql
-- 1日目: レベル0イメージ・コピー・ベースラインを作成
BACKUP INCREMENTAL LEVEL 0 AS COPY DATABASE;

-- 毎日: イメージ・コピーを昨日の変更でロールフォワードする
RECOVER COPY OF DATABASE WITH TAG 'daily_copy'
  UNTIL TIME 'SYSDATE - 1';
BACKUP INCREMENTAL LEVEL 1 FOR RECOVER OF COPY
  WITH TAG 'daily_copy' DATABASE;
```

---

## バックアップ保持ポリシー

RMAN保持ポリシーは、バックアップが「不要（obsolete）」と見なされるまでどのくらいの期間保持するかを定義する。

### リカバリ・ウィンドウによる保持

指定されたウィンドウ内の任意の時点へのリカバリを可能にするのに十分なバックアップを保持する：

```sql
-- 過去7日間の任意の時点にリカバリするために必要なバックアップを保持
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
```

### 冗長性による保持

特定の数のバックアップ・コピーを保持する：

```sql
-- 各データファイルのフル・コピーを2つ保持
CONFIGURE RETENTION POLICY TO REDUNDANCY 2;
```

### 保持ポリシーのクリア

```sql
-- 保持ポリシーなし（すべて保持 — 外部管理がない場合は非推奨）
CONFIGURE RETENTION POLICY TO NONE;
```

### 不要なバックアップの削除

保持ポリシーを設定した後、不要なバックアップ（obsolete）を削除できる：

```sql
-- 削除される対象をリスト
REPORT OBSOLETE;

-- 不要なバックアップ・ピースを削除
DELETE OBSOLETE;

-- すべての期限切れ（expired）バックアップ・レコードを削除（期待される場所にファイルがないもの）
CROSSCHECK BACKUP;
DELETE EXPIRED BACKUP;
```

---

## RMANカタログのセットアップ

### リカバリ・カタログを使用する理由

- 制御ファイルの`CONTROL_FILE_RECORD_KEEP_TIME`に収まる期間を超えてバックアップ履歴を保存できる
- 複数のデータベース間で共有される格納RMANスクリプトを使用できる
- 複数のターゲット・データベースにわたるバックアップのレポートをサポートする
- アクセス権限を委譲するためのRMAN仮想プライベート・カタログ(VPC)に必要

### カタログの作成

```sql
-- 1. カタログ・データベースに専用の表領域を作成
CREATE TABLESPACE rman_cat
  DATAFILE '/oradata/rmancat/rman_cat01.dbf' SIZE 500M AUTOEXTEND ON;

-- 2. カタログ所有者ユーザーを作成
CREATE USER rman_owner
  IDENTIFIED BY <password>
  DEFAULT TABLESPACE rman_cat
  QUOTA UNLIMITED ON rman_cat;

GRANT RECOVERY_CATALOG_OWNER TO rman_owner;
```

```bash
# 3. RMANに接続してカタログ・スキーマを作成
rman catalog rman_owner/<password>@catdb
RMAN> CREATE CATALOG;
```

### ターゲット・データベースの登録

```bash
rman target sys/<password>@proddb catalog rman_owner/<password>@catdb
RMAN> REGISTER DATABASE;
```

### カタログの再同期

```sql
-- カタログをターゲットの制御ファイルと同期する
RESYNC CATALOG;

-- フル再同期（より徹底的）
RESYNC CATALOG FROM CONTROLFILECOPY '/path/to/ctl_copy';
```

---

## リカバリ・シナリオ

### RMANへの接続

```bash
# ターゲットのみに接続（メタデータに制御ファイルを使用）
rman target sys/<password>@proddb

# リカバリ・カタログを使用して接続
rman target sys/<password>@proddb catalog rman_owner/<password>@catdb

# OS認証を使用してSYSDBAとして接続（ローカル）
rman target /
```

### 完全リカバリ

完全リカバリは、データベースを現在の時点までリカバリする（データ損失なし）。バックアップ以降から現在のSCNまでのすべてのアーカイブ・ログが必要。

```sql
-- データベースをマウント（オープンではない）し、リストアとリカバリを行う
STARTUP MOUNT;
RESTORE DATABASE;
RECOVER DATABASE;
ALTER DATABASE OPEN;
```

### 不完全リカバリ (Point-in-Time Recovery)

誤ったテーブル削除や破損が発生する前など、現在より前の時点までリカバリする必要がある場合に使用する。

**SCNによる指定:**
```sql
STARTUP MOUNT;
RESTORE DATABASE UNTIL SCN 5432100;
RECOVER DATABASE UNTIL SCN 5432100;
ALTER DATABASE OPEN RESETLOGS;
```

**時間による指定:**
```sql
STARTUP MOUNT;
RESTORE DATABASE UNTIL TIME "TO_DATE('2025-12-01 14:30:00','YYYY-MM-DD HH24:MI:SS')";
RECOVER DATABASE UNTIL TIME "TO_DATE('2025-12-01 14:30:00','YYYY-MM-DD HH24:MI:SS')";
ALTER DATABASE OPEN RESETLOGS;
```

**順序番号(Sequence)による指定:**
```sql
STARTUP MOUNT;
RESTORE DATABASE UNTIL SEQUENCE 1450 THREAD 1;
RECOVER DATABASE UNTIL SEQUENCE 1450 THREAD 1;
ALTER DATABASE OPEN RESETLOGS;
```

注：不完全リカバリの後には`RESETLOGS`が必要である。これによりログ・シーケンスがリセットされ、新しいインカーネーションが作成される。

### 表領域のポイント・イン・タイム・リカバリ (TSPITR)

データベースの他の部分は実行したまま、単一の表領域を過去の時点にリカバリする。自動的に補助インスタンスを使用する。

```sql
-- USERS表領域を2時間前の状態にリカバリ
RECOVER TABLESPACE users
  UNTIL TIME 'SYSDATE - 2/24'
  AUXILIARY DESTINATION '/tmp/tspitr_aux';
```

### データファイルのリカバリ

単一のデータファイルが紛失または破損した場合：

```sql
-- データファイルをオフラインにする
ALTER DATABASE DATAFILE '/oradata/users01.dbf' OFFLINE;

-- 切断されたデータファイルのみをリストア
RESTORE DATAFILE '/oradata/users01.dbf';

-- アーカイブ・ログを適用して最新の状態にする
RECOVER DATAFILE '/oradata/users01.dbf';

-- データファイルをオンラインに戻す
ALTER DATABASE DATAFILE '/oradata/users01.dbf' ONLINE;
```

### 制御ファイルのリカバリ

```sql
-- 自動バックアップから制御ファイルをリストア
STARTUP NOMOUNT;
RESTORE CONTROLFILE FROM AUTOBACKUP;
ALTER DATABASE MOUNT;
RECOVER DATABASE;
ALTER DATABASE OPEN RESETLOGS;
```

---

## ベスト・プラクティス

- **制御ファイルの自動バックアップを有効にする** — カタログがなくてもRMANが制御ファイルを確実にリカバリできるようにする。注：`COMPATIBLE`が12.2以上に設定されているデータベースでは、自動バックアップはデフォルトで「ON」になっているが、古い互換モードの場合は設定を確認すること。
  ```sql
  CONFIGURE CONTROLFILE AUTOBACKUP ON;
  CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/backup/ctl_%F';
  ```

- **常にアーカイブ・ログをバックアップする** — アーカイブ・ログのないデータベース・バックアップは、一貫性のある状態にリカバリできない。
  ```sql
  BACKUP DATABASE PLUS ARCHIVELOG DELETE INPUT;
  ```

- **定期的にバックアップをテストする** — バックアップ・ピースが完全であり、リストア可能であることを検証する。
  ```sql
  RESTORE DATABASE VALIDATE;
  RESTORE TABLESPACE users VALIDATE;
  ```

- **本番環境ではリカバリ・カタログを使用する** — 長期的な履歴を保持するには制御ファイルだけでは不十分である。

- **Enterprise Editionではブロック変更トラッキングを有効にする** — 増分バックアップを高速化するため。

- **バックアップをホスト外に保存する** — サーバー自体が紛失した場合、ホスト上のディスク・バックアップは役に立たない。NFS、ASM、オブジェクト・ストレージ、またはテープを使用すること。

- **リカバリ手順書を作成し、少なくとも年に1回はテストする** — テストされていないバックアップ戦略は、戦略とは言えない。

- **大きな変更（スキーマ変更、大規模パッチ、アップグレードなど）の前後でバックアップを取得する。**

---

## よくある間違いと回避策

**データベースと同じディスクへのバックアップ**
ディスクが故障すると、データベースとバックアップの両方が失われる。バックアップは必ず別のストレージ層に書き込むこと。

**リストアを一度もテストしない**
テストされていないバックアップは想定であり、保証ではない。四半期ごとに別のサーバーへのリストア・テストをスケジュールすること。

**アラートログ内のRMANアラートの無視**
失敗したバックアップ・ジョブはアラートログやRMANログにエラーを書き込むが、通知されないことが多い。`V$RMAN_STATUS`を使用してRMANジョブのステータス監視を設定すること。

```sql
-- 最近のRMANジョブのステータスを確認
SELECT start_time, end_time, status, input_bytes_display, output_bytes_display
FROM v$rman_backup_job_details
ORDER BY start_time DESC
FETCH FIRST 10 ROWS ONLY;
```

**制御ファイルとSPFILEを個別にバックアップしない**
```sql
-- 制御ファイルとSPFILEの明示的なバックアップ
BACKUP CURRENT CONTROLFILE;
BACKUP SPFILE;
```

**アーカイブ・ログによるFRAの圧迫**
```sql
-- FRAの使用状況を確認
SELECT name, space_limit/1048576 limit_mb,
       space_used/1048576 used_mb,
       space_reclaimable/1048576 reclaimable_mb
FROM v$recovery_file_dest;

-- 少なくとも1回はバックアップ済みのアーカイブ・ログを削除
DELETE ARCHIVELOG ALL BACKED UP 1 TIMES TO DEVICE TYPE DISK;
```

**開発/テスト以外でのNOARCHIVELOGモードの使用**
NOARCHIVELOGモードでは、最後のフル・バックアップまでしかリカバリできない。メディア障害が発生すると、それ以降のすべてのトランザクションが永久に失われる。本番環境では必ずARCHIVELOGモードを使用すること。

```sql
-- アーカイブログ・モードの確認
SELECT log_mode FROM v$database;

-- アーカイブログ・モードの有効化
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
```

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c バックアップおよびリカバリ・ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/)
- [Oracle Database 19c RMANリファレンス — BACKUPコマンド](https://docs.oracle.com/en/database/oracle/oracle-database/19/rcmrf/BACKUP.html)
- [Oracle Database 19c RMANリファレンス — CONFIGUREコマンド](https://docs.oracle.com/en/database/oracle/oracle-database/19/rcmrf/CONFIGURE.html)
- [Oracle Database 19c リファレンス — CONTROL_FILE_RECORD_KEEP_TIME](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/CONTROL_FILE_RECORD_KEEP_TIME.html)
