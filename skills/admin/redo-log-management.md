# Oracle redoログ管理

## 概要

redoログはOracleの先行書き込みログ（write-ahead log）である。データベースに対して行われるすべての変更（INSERT、UPDATE、DELETE、DDL、さらには内部操作まで）は、まずオンラインredoログ・バッファにredoエントリとして記録され、その後Log Writer (LGWR)プロセスによってディスク上の**オンラインredoログ・ファイル**にフラッシュされる。これにより耐久性が確保される。データベースがクラッシュした場合、redoログを使用して最後のチェックポイント以降のすべてのコミット済み変更を再生できる。

redoログ・インフラストラクチャを正しく理解し、適切なサイズに設定することは、データベースのパフォーマンスと可用性にとって極めて重要である。サイズ設定が不適切なredoログは、過度なログ・スイッチやチェックポイント・プレッシャーを引き起こし、スループットを著しく低下させる可能性がある。

---

## オンラインredoログ・グループとメンバー

### グループ

Oracleのオンラインredoログは**グループ**で構成され、各グループには1つ以上の**メンバー**（物理ファイル）が含まれる。常にLGWRは正確に1つのグループ、つまり**カレント（current）**・グループに書き込みを行っている。ログ・スイッチが発生すると、LGWRは次のグループに移動する。

- データベースが動作するには最低2つのグループが必要
- 本番データベースでは、アーカイブ処理が追いつくまでの余裕を持たせるため、少なくとも3つのグループが必要
- グループはグループ番号（1、2、3...）で識別される

### メンバー (多重化)

各グループには、複数のメンバー・ファイル（異なるディスク上の同じredoデータの物理コピー）を含めることができる。グループ内のすべてのメンバーには同一のredoデータが含まれる。1つのメンバーが失われても（ディスク故障など）、残りのメンバーを使用してグループは機能し続ける。

- 1グループあたり最低1メンバー。本番環境では2〜3メンバーを推奨。
- 冗長性を確保するため、メンバーは異なる物理ディスクやストレージ・コントローラに配置すべきである
- グループ内のすべてのメンバーは同じサイズでなければならない

```
オンラインredoログの構造:
┌─────────────────────────────────────────┐
│ グループ 1: /disk1/redo01a.log            │  ← CURRENT (LGWRが書き込み中)
│           /disk2/redo01b.log  (ミラー)   │
├─────────────────────────────────────────┤
│ グループ 2: /disk1/redo02a.log            │  ← ACTIVE (インスタンス・リカバリに必要)
│           /disk2/redo02b.log  (ミラー)   │
├─────────────────────────────────────────┤
│ グループ 3: /disk1/redo03a.log            │  ← INACTIVE (LGWRが再利用可能)
│           /disk2/redo03b.log  (ミラー)   │
└─────────────────────────────────────────┘
```

### 現在のredoログ構成の表示

```sql
-- すべてのグループとそのステータスを表示
SELECT group#, members, bytes/1048576 size_mb, status, archived
FROM v$log
ORDER BY group#;

-- すべてのメンバーとそのパスを表示
SELECT l.group#, l.sequence#, l.status,
       lf.member, lf.status member_status
FROM v$log l
JOIN v$logfile lf ON l.group# = lf.group#
ORDER BY l.group#, lf.member;

-- グループ・ステータスの値:
-- CURRENT   = LGWRが現在このグループに書き込み中
-- ACTIVE    = インスタンス・リカバリに必要（まだチェックポイントが完了していない）
-- INACTIVE  = リカバリに不要。再利用可能
-- UNUSED    = 一度も書き込まれていない
-- CLEARING  = 再作成中（ALTER DATABASE CLEAR LOGFILEが進行中）
```

---

## redoログの追加、削除、およびサイズ変更

### 新しいグループとメンバーの追加

```sql
-- 2つのメンバーを持つ新しいグループを追加 (多重化)
ALTER DATABASE ADD LOGFILE GROUP 4
  ('/disk1/redo04a.log', '/disk2/redo04b.log') SIZE 500M;

-- 既存のグループにメンバーを追加 (既存グループの多重化)
ALTER DATABASE ADD LOGFILE MEMBER
  '/disk2/redo01b.log' TO GROUP 1;

-- すべてのグループに一度にメンバーを追加 (スクリプト内でのループ)
ALTER DATABASE ADD LOGFILE MEMBER '/disk2/redo01b.log' TO GROUP 1;
ALTER DATABASE ADD LOGFILE MEMBER '/disk2/redo02b.log' TO GROUP 2;
ALTER DATABASE ADD LOGFILE MEMBER '/disk2/redo03b.log' TO GROUP 3;
```

### グループとメンバーの削除

INACTIVEな（CURRENTでもACTIVEでもない）グループのみを削除できる。削除後にグループ数が2未満になる場合は削除できない。

```sql
-- redoログ・グループを削除
ALTER DATABASE DROP LOGFILE GROUP 4;

-- 特定のメンバーを削除 (古いバージョンではOSファイルは自動的に削除されない)
ALTER DATABASE DROP LOGFILE MEMBER '/disk2/redo01b.log';

-- 削除後、必要に応じて手動でOSファイルを削除する
-- (SQLではなくホストOSのコマンド)
```

### redoログのサイズ変更

Oracleは`ALTER DATABASE RESIZE LOGFILE`をサポートしていない。サイズを変更するには、正しいサイズの新しいグループを追加し、古いグループをINACTIVEにしてから削除する。

```sql
-- ステップ 1: 希望のサイズで新しいグループを追加
ALTER DATABASE ADD LOGFILE GROUP 4 ('/oradata/redo04a.log') SIZE 1G;
ALTER DATABASE ADD LOGFILE GROUP 5 ('/oradata/redo05a.log') SIZE 1G;
ALTER DATABASE ADD LOGFILE GROUP 6 ('/oradata/redo06a.log') SIZE 1G;

-- ステップ 2: ログ・スイッチを強制して古いグループを循環させる
ALTER SYSTEM SWITCH LOGFILE;   -- 古いグループがINACTIVEになるまで繰り返す
ALTER SYSTEM CHECKPOINT;       -- チェックポイントを進めてACTIVEをINACTIVEにする

-- ステップ 3: サイズが不適切な古いグループを削除
ALTER DATABASE DROP LOGFILE GROUP 1;
ALTER DATABASE DROP LOGFILE GROUP 2;
ALTER DATABASE DROP LOGFILE GROUP 3;

-- ステップ 4: 新しい構成を確認
SELECT group#, bytes/1048576 size_mb, status FROM v$log ORDER BY group#;
```

---

## redoログのサイズ設定 (頻繁なログ・スイッチの回避)

### ログ・スイッチの頻度が重要な理由

すべてのログ・スイッチは**チェックポイント**（DBWRがすべてのダーティ・バッファをディスクに書き込むよう要求すること）をトリガーし、ARCHIVELOGモードでは、再利用前にARCnプロセスがログをアーカイブする必要がある。頻繁なログ・スイッチは以下の影響を及ぼす：

- チェックポイント活動によるI/Oプレッシャーの増大
- 未アーカイブのグループに戻った場合にLGWRが停止する可能性がある（ログ・スイッチ待機）
- 過剰なアーカイブ・ログの生成
- アラートログやV$SESSION_WAITに`log file switch`待機イベントとして表示される

Oracleは、通常の負荷下でログ・スイッチが15〜30分に1回程度になるようにサイズ設定することを推奨している。

### 現在のログ・スイッチ頻度の測定

```sql
-- 1時間あたりのログ・スイッチ頻度 (V$LOG_HISTORY経由)
SELECT TO_CHAR(first_time, 'YYYY-MM-DD HH24') hour_bucket,
       COUNT(*) switches
FROM v$log_history
WHERE first_time > SYSDATE - 7
GROUP BY TO_CHAR(first_time, 'YYYY-MM-DD HH24')
ORDER BY 1 DESC
FETCH FIRST 48 ROWS ONLY;

-- 1スイッチあたりの平均生成redo量 (新しいログのサイズ設定に役立つ)
SELECT ROUND(AVG(blocks * block_size) / 1048576, 1) avg_mb_per_log
FROM v$archived_log
WHERE first_time > SYSDATE - 7
  AND standby_dest = 'NO';

-- 現在のログ・グループ・サイズ vs 実際の使用量
SELECT l.group#, l.bytes/1048576 size_mb,
       l.status,
       lh.blocks * lh.block_size / 1048576 last_used_mb
FROM v$log l
LEFT JOIN v$archived_log lh ON l.sequence# = lh.sequence#
ORDER BY l.group#;
```

### サイズ設定の推奨事項

一般的な目安：ピーク時の負荷下で15〜30分に1回のログ・スイッチが発生するようにredoログのサイズを設定する。

現在のログが200MBで、ピーク時に3分ごとにスイッチが発生している場合、ピーク時のredo生成率は約200MB / 3分 = 66 MB/分である。20分ごとのスイッチを目指す場合のターゲット・ログ・サイズは、66 MB/分 × 20分 = 1.3 GBとなる。

```sql
-- より正確なサイズ設定: UNDOSTATから10分間あたりの最大undoブロック数を確認する
-- (UNDOSTATはundoブロックを追跡する。redoにはV$SYSSTATを使用)
SELECT statistic#, name, value
FROM v$sysstat
WHERE name IN ('redo size', 'redo entries', 'redo log space requests',
               'redo log space wait time');
```

---

## ARCHIVELOGモード

### ARCHIVELOGモードの役割

**ARCHIVELOGモード**では、Oracleがオンラインredoログ・グループを再利用する前に、ARCn（アーカイバ）プロセスがそれを**アーカイブ・ログ出力先**にコピーする。このアーカイブされたコピーにより、以下のことが可能になる：

- データベースを任意の時点にリカバリする（直近のフル・バックアップだけでなく）
- RMANを使用してオンライン（ホット）バックアップを実行する
- Data Guard（スタンバイ・データベース）を使用する
- Oracle Streams、LogMiner、またはGoldenGateを使用する

**NOARCHIVELOGモード**では、オンラインredoログは単に上書きされる。リカバリは直近のフル・バックアップまでしかできず、それ以降のコミット済み変更はメディア障害発生時に復旧できない。NOARCHIVELOGは開発またはテスト用のデータベースにのみ許容される。

### ARCHIVELOGモードの確認と有効化

```sql
-- 現在のモードを確認
SELECT log_mode, name FROM v$database;

-- アーカイバのステータスを表示
SELECT archiver FROM v$instance;

-- ARCHIVELOGモードの有効化
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- 確認
SELECT log_mode FROM v$database;
-- 結果が ARCHIVELOG であること
```

### アーカイブ・ログ出力先の構成

```sql
-- 主要なアーカイブ・ログ出力先を設定
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1 =
  'LOCATION=/oradata/archive VALID_FOR=(ALL_LOGFILES,ALL_ROLES)'
  SCOPE=BOTH;

-- 出力先を有効化
ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_1 = ENABLE SCOPE=BOTH;

-- 高速リカバリ領域 (FRA) をアーカイブ出力先に設定
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '/oradata/fra' SCOPE=BOTH;
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 100G SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1 =
  'LOCATION=USE_DB_RECOVERY_FILE_DEST' SCOPE=BOTH;

-- 現在のアーカイブ出力先を表示
SELECT dest_id, dest_name, status, target, archiver, schedule,
       destination, transmit_mode
FROM v$archive_dest
WHERE status != 'INACTIVE'
ORDER BY dest_id;
```

### 手動ログ・スイッチとアーカイブ

```sql
-- ログ・スイッチを強制
ALTER SYSTEM SWITCH LOGFILE;

-- 未アーカイブのすべてのログをアーカイブ (手動アーカイブまたはテスト用)
ALTER SYSTEM ARCHIVE LOG ALL;

-- カレント・ログをアーカイブ
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

---

## アーカイブ・ログの管理

アーカイブ・ログは継続的に蓄積される。アクティブに管理しないと、最終的にアーカイブ出力先または高速リカバリ領域がいっぱいになり、データベースがハングする原因となる。

### アーカイブ・ログ領域の監視

```sql
-- FRAの使用状況を確認
SELECT name, space_limit/1073741824 limit_gb,
       space_used/1073741824 used_gb,
       space_reclaimable/1073741824 reclaimable_gb,
       ROUND(space_used / space_limit * 100, 1) pct_used
FROM v$recovery_file_dest;

-- ディスク上のアーカイブ・ログの数とサイズ
SELECT dest_id, COUNT(*) log_count,
       SUM(blocks * block_size) / 1073741824 total_gb
FROM v$archived_log
WHERE standby_dest = 'NO'
  AND deleted = 'NO'
GROUP BY dest_id;

-- ディスク上の最も古いアーカイブ・ログと最新のアーカイブ・ログ
SELECT MIN(first_time) oldest_log, MAX(first_time) newest_log
FROM v$archived_log
WHERE standby_dest = 'NO'
  AND deleted = 'NO';
```

### アーカイブ・ログの安全な削除

アーカイブ・ログはOSコマンドではなく、必ずRMAN経由で削除すること：

```sql
-- RMAN経由: 少なくとも1回はバックアップ済みのアーカイブ・ログを削除
DELETE ARCHIVELOG ALL BACKED UP 1 TIMES TO DEVICE TYPE DISK;

-- 7日より前のアーカイブ・ログを削除 (バックアップの有無に関わらず。注意して使用すること)
DELETE ARCHIVELOG UNTIL TIME 'SYSDATE-7';

-- 少なくとも2回バックアップ済みのすべてのアーカイブ・ログを削除 (非常に安全)
DELETE ARCHIVELOG ALL BACKED UP 2 TIMES TO DEVICE TYPE DISK;
```

誤ってOSコマンドでアーカイブ・ログを削除してしまった場合（RMAN外）：
```sql
-- 存在しないファイルを見つけてEXPIREDとしてマークする
CROSSCHECK ARCHIVELOG ALL;

-- リポジトリから期限切れのレコードを削除する
DELETE EXPIRED ARCHIVELOG ALL;
```

---

## redoログの多重化

多重化とは、各redoログ・グループの複数のコピー（メンバー）を異なるディスク上に維持することである。1つのメンバーが失われても、LGWRは残りのメンバーへの書き込みを継続し、データベースの停止を回避できる。

```sql
-- 既存の各グループに2番目のメンバーを追加 (多重化)
-- グループ 1-3 が存在すると想定
ALTER DATABASE ADD LOGFILE MEMBER '/disk2/redo01b.log' TO GROUP 1;
ALTER DATABASE ADD LOGFILE MEMBER '/disk2/redo02b.log' TO GROUP 2;
ALTER DATABASE ADD LOGFILE MEMBER '/disk2/redo03b.log' TO GROUP 3;

-- 多重化の確認
SELECT l.group#, lf.member, lf.type, lf.status
FROM v$log l JOIN v$logfile lf ON l.group# = lf.group#
ORDER BY l.group#, lf.member;
```

### 紛失したredoログ・メンバーの復旧

ディスク故障などでメンバー・ファイルが失われたが、グループの他のメンバーが損なわれていない場合：

```sql
-- グループのステータスは CURRENT または ACTIVE だが、1つのメンバーが INVALID または STALE になる
-- 紛失したメンバーを削除
ALTER DATABASE DROP LOGFILE MEMBER '/disk2/redo01b.log';

-- 新しいメンバーを再度追加 (Oracleがファイルを作成する)
ALTER DATABASE ADD LOGFILE MEMBER '/disk3/redo01b.log' TO GROUP 1;
-- 注: 新しいメンバーは最初 INVALID で、次のログ・スイッチでカレントになる
```

### 紛失したカレント・ログ・グループからの復旧 (メディア障害)

非常に深刻なシナリオである。カレント・グループが消失した場合：

```sql
-- まず、ログのクリアを試みる (新しい空のログを作成し、未アーカイブのredoは失われる)
-- 警告: データ損失が発生します。真に復旧不可能な場合にのみ使用してください。
ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 1;

-- クリア後、バックアップからのリカバリが必要になる場合がある。
-- すべてのデータファイルに整合性があるか確認
SELECT file#, name, status FROM v$datafile WHERE status != 'ONLINE';
```

---

## ログ・スイッチ頻度の監視

### アラートログの監視

ログ・スイッチはアラートログにタイムスタンプと共に記録される。数分ごとの頻繁なスイッチは、redoログのサイズ不足を示している。

```sql
-- 日ごとのログ・スイッチ履歴 (キャパシティ・プランニングに有用)
SELECT TRUNC(first_time, 'DD') log_date,
       COUNT(*) daily_switches,
       ROUND(COUNT(*) / 24, 1) switches_per_hour_avg
FROM v$log_history
WHERE first_time > SYSDATE - 30
GROUP BY TRUNC(first_time, 'DD')
ORDER BY 1 DESC;

-- 特定の時間帯におけるピーク時のスイッチ
SELECT TO_CHAR(first_time, 'YYYY-MM-DD HH24') hour,
       COUNT(*) switches
FROM v$log_history
WHERE first_time > SYSDATE - 7
GROUP BY TO_CHAR(first_time, 'YYYY-MM-DD HH24')
ORDER BY 2 DESC
FETCH FIRST 10 ROWS ONLY;
```

### ログ・スイッチ待機イベント

LGWRが（チェックポイントが完了していない、またはアーカイブ中であるために）ACTIVEな次のログ・グループに切り替えられない場合、セッションは待機状態になる：

```sql
-- ログ・スイッチ待機イベントの確認
SELECT event, total_waits, time_waited, average_wait
FROM v$system_event
WHERE event LIKE 'log file switch%'
ORDER BY time_waited DESC;

-- イベント: "log file switch (checkpoint incomplete)" → グループ数を増やすかI/Oを高速化する必要がある
-- イベント: "log file switch (archiving needed)"    → ARCnが追いついていない。アーカイバのラグを確認
-- イベント: "log file switch completion"            → 時折発生するスイッチのオーバーヘッド (少量なら正常)

-- 現在のセッション待機状況
SELECT s.sid, s.serial#, s.username, s.event, s.state, s.seconds_in_wait
FROM v$session s
WHERE s.event LIKE 'log file switch%'
   OR s.event = 'log file sync';
```

---

## ベスト・プラクティス

- **本番環境では常にARCHIVELOGモードで実行する。** NOARCHIVELOGモードではポイント・イン・タイム・リカバリができない。

- **別の物理ディスクにredoログを多重化する。** カレントredoログを失うリスクに比べれば、追加コピーのコストは微々たるものである。

- **ピーク時のログ・スイッチを15〜30分に設定する。** 小さすぎるとスイッチが頻発しパフォーマンスが低下し、大きすぎるとメディア・リカバリ時に適用するredo量が増えて時間がかかる。

- **高速リカバリ領域 (FRA) を使用する。** アーカイブ・ログ管理が簡素化され、RMANによる自動クリーンアップが可能になる。

- **FRA領域を毎日監視する。** FRAがいっぱいになると、空きができるまでデータベースがハングする。FRAの使用率にOEMアラートを設定すること。

- **OSコマンドでアーカイブ・ログを削除しない。** リポジトリが正確に保たれるよう、必ずRMANを使用して削除する。

- **必要になる前にグループを追加しておく。** 少なくとも3〜5グループあれば、アーカイバが追いつく余裕ができる。大量のデータ・ロードやバルクDMLが発生する場合は、6グループ以上が必要になることもある。

- **LGWRのI/Oを高速に保つ。** redoログは低遅延のストレージ（SSD、専用ドライブ、または高冗長性のASM）に配置すべきである。LGWRの遅延はコミットの応答時間に直結する。

---

## よくある間違いと回避策

**redoログ・グループが2つしかない**
2つのグループしかないと、LGWRがグループ1を使い切り、ARCnがまだアーカイブを終えていない場合、LGWRはグループ1が解放されるまでハングする。必ず3つ以上のグループを使用すること。

**redoログをミラー化していない**
カレントredoログ・グループの唯一のメンバーが失われる（ディスク故障など）と、データ損失を受け入れて`CLEAR UNARCHIVED LOGFILE`を使用しない限り、復旧不可能なデータベースになる。必ず多重化すること。

**redoログをデータファイルと同じ物理ディスクに配置する**
書き込み負荷が高い場合、redoログのI/OがデータファイルのI/Oと競合する。redoログはデータファイルやアーカイブ・ログとは別の専用ディスクに配置すること。

**FRA領域の監視を怠る**
```sql
-- 日次の監視スクリプトに追加
SELECT name, space_limit/1073741824 limit_gb,
       space_used/1073741824 used_gb,
       ROUND(space_used/space_limit*100,1) pct_full
FROM v$recovery_file_dest;
```
FRAが100%になると、次のログ・スイッチ時にデータベースがハングする。

**アーカイブ出力先を構成せずにARCHIVELOGモードに切り替える**
`LOG_ARCHIVE_DEST_1`が設定されておらず、FRAもない場合、OracleはデフォルトのOSの場所にアーカイブするが、そこには適切な領域がなかったり、バックアップ対象外だったりすることがある。

**ACTIVE状態のログ・グループを削除しようとする**
これは ORA-00350 で失敗する。まず `ALTER SYSTEM CHECKPOINT` を発行してチェックポイントを進め、グループを INACTIVE に切り替えてから削除すること。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 管理者ガイド 19c — redoログの管理](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-the-redo-log.html)
- [Oracle Database 19c リファレンス — V$LOG](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOG.html)
- [Oracle Database 19c リファレンス — V$LOGFILE](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOGFILE.html)
- [Oracle Database 19c リファレンス — V$LOG_HISTORY](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOG_HISTORY.html)
