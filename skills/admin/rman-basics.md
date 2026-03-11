# RMANの基本操作ガイド

## 概要

Recovery Manager (RMAN)は、Oracleデータベースのバックアップ、復元(リストア)、および回復(リカバリ)を管理するための推奨ツールである。本稿では、日常的な管理業務で使用する最も一般的なRMANコマンドと構成例を紹介する。

RMANは、SQL*Plusと同様のコマンドラインから操作するか、シェル・スクリプトを介して自動化して使用する。

---

## 接続

### ターゲット・データベースへの接続

```bash
# OS認証を使用してローカルに接続
rman target /

# TNSサービス名を使用してリモートに接続
rman target sys/<password>@pdb_name

# バックアップ・メタデータを保持するリカバリ・カタログを使用して接続
rman target / catalog rman_owner/password@catalog_db
```

---

## 構成設定 (Persistent Configuration)

RMANの設定は制御ファイル内に永続的に保存されるため、毎回指定する必要はない。

### 一般的な構成の表示と変更

```sql
-- 現在のすべての構成設定を表示
SHOW ALL;

-- 制御ファイルの自動バックアップを有効化
CONFIGURE CONTROLFILE AUTOBACKUP ON;

-- バックアップの保持ポリシーを「7日間のリカバリ・ウィンドウ」に設定
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;

-- デフォルトのデバイス・タイプをディスクに設定
CONFIGURE DEFAULT DEVICE TYPE TO DISK;

-- デフォルトのバックアップ・タイプを「圧縮バックアップセット」に設定
CONFIGURE DEVICE TYPE DISK BACKUP TYPE TO COMPRESSED BACKUPSET;

-- デフォルトの並列度(Parallelism)を2に設定
CONFIGURE DEVICE TYPE DISK PARALLELISM 2;

-- 設定をデフォルトに戻す
CONFIGURE RETENTION POLICY CLEAR;
```

---

## バックアップ操作

### データベース全体のバックアップ

```sql
-- データベース全体、アーカイブ・ログ、および制御ファイルをバックアップ
BACKUP DATABASE PLUS ARCHIVELOG;

-- 使用済みブロックのみをバックアップ（圧縮）
BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;

-- すべてのアーカイブ・ログのみをバックアップし、成功後にログを削除
BACKUP ARCHIVELOG ALL DELETE INPUT;
```

### 特定のコンポーネントのバックアップ

```sql
-- 特定の表領域をバックアップ
BACKUP TABLESPACE users, exports;

-- 特定のデータファイルをバックアップ
BACKUP DATAFILE 4, '/u01/app/oracle/oradata/PROD/system01.dbf';

-- 現在の制御ファイルのみをバックアップ
BACKUP CURRENT CONTROLFILE;

-- サーバー・パラメータ・ファイル(SPFILE)をバックアップ
BACKUP SPFILE;
```

### 増分バックアップ (Incremental Backups)

```sql
-- 増分ベースライン (レベル0)
BACKUP INCREMENTAL LEVEL 0 DATABASE;

-- 差分増分 (レベル1: 前回のレベル0または1以降の変更)
BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- 累積増分 (レベル1: 前回のレベル0以降のすべての変更)
BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;
```

---

## リストアとリカバリ

### リスト・コマンドによる確認

アクションを実行する前に、RMANが認識しているバックアップを確認する。

```sql
-- すべてのバックアップ・セットのサマリーを表示
LIST BACKUP SUMMARY;

-- 特定の表領域のバックアップを表示
LIST BACKUP OF TABLESPACE system;

-- すべてのアーカイブ・ログを表示
LIST ARCHIVELOG ALL;

-- 期限切れ（パスにファイルが存在しない）バックアップを表示
LIST EXPIRED BACKUP;
```

### データベースのリカバリ (完全リカバリ)

```sql
-- データベースを停止してマウント状態で起動
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;

-- バックアップからファイルを復元
RESTORE DATABASE;

-- アーカイブ・ログを適用して最新状態にする
RECOVER DATABASE;

-- データベースをオープン
ALTER DATABASE OPEN;
```

### ポイント・イン・タイム・リカバリ (DBPITR)

```sql
-- 特定の時刻まで戻す
RUN {
  SET UNTIL TIME "TO_DATE('2025-10-21 14:00:00','YYYY-MM-DD HH24:MI:SS')";
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}

-- 特定のSCN(System Change Number)まで戻す
RUN {
  SET UNTIL SCN 1234567;
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}
```

---

## メンテナンスとクリーンアップ

### 整合性チェック

```sql
-- バックアップ・ファイルがディスク/テープ上に物理的に存在するか確認
CROSSCHECK BACKUP;

-- アーカイブ・ログが存在するか確認
CROSSCHECK ARCHIVELOG ALL;

-- データベースの破損をチェック（実際にリストアせずに読み取りテストを行う）
VALIDATE DATABASE;

-- 特定のバックアップ・セットの妥当性を確認
VALIDATE BACKUPSET 120;
```

### 不要なバックアップの削除

```sql
-- 保持ポリシーに基づいて「不要」と判断されたバックアップを削除
DELETE OBSOLETE;

-- CROSSCHECKで見つからなかった「期限切れ」レコードをカタログから削除
DELETE EXPIRED BACKUP;

-- 2回以上バックアップ済みの古いアーカイブ・ログを削除
DELETE ARCHIVELOG ALL BACKED UP 2 TIMES TO DEVICE TYPE DISK;
```

---

## 役立つレポート

```sql
-- 保持ポリシーに従って削除可能なバックアップを一覧表示
REPORT OBSOLETE;

-- 現在の保持ポリシーに基づいてリカバリできない（バックアップが不足している）ファイルを一覧表示
REPORT NEED BACKUP;

-- データベースのスキーマ構成を表示
REPORT SCHEMA;
```

---

## ベスト・プラクティス

1. **制御ファイルの自動バックアップを有効にする:** `CONFIGURE CONTROLFILE AUTOBACKUP ON;` を設定しておくことで、制御ファイルが失われてもリカバリが可能になる。
2. **定期的なCROSSCHECKとDELETE OBSOLETE:** ストレージ容量を管理するために、週に一度は実行するスクリプトを作成する。
3. **アーカイブ・ログの頻繁なバックアップ:** アーカイブ・ログはリカバリの鍵である。少なくとも1時間に一度、または生成量に応じてバックアップする。
4. **プレビュー機能の使用:** 実際に実行する前に、どのバックアップが使用されるかを確認する。
   ```sql
   RESTORE DATABASE PREVIEW;
   ```
5. **タグ(TAG)の活用:** 重要なバックアップに名前を付けて管理しやすくする。
   ```sql
   BACKUP DATABASE TAG 'BEFORE_UPGRADE_2025';
   ```

---

## よくある間違い

- **RMANの外でバックアップ・ファイルを削除する:** 
  OSコマンドでファイルを削除すると、RMANのカタログと実際のファイルの状態が乖離する。必ずRMANの `DELETE` コマンドを使用するか、OSで削除した後は `CROSSCHECK` を実行すること。
- **ログ・モードの確認ミス:** 
  NOARCHIVELOGモードでは、オンライン状態でのバックアップやポイント・イン・タイム・リカバリはできない。
- **バックアップ先(FRA: Fast Recovery Area)の容量不足:** 
  容量がいっぱいになるとアーカイブができなくなり、データベースがハングする。定期的なクリーンアップを自動化すること。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c RMANリファレンス](https://docs.oracle.com/en/database/oracle/oracle-database/19/rcmrf/index.html)
- [Oracle Database 19c バックアップおよびリカバリ・ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/index.html)
