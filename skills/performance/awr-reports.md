# AWR レポート — Automatic Workload Repository

## 概要

**Automatic Workload Repository (AWR)** は、Oracle に組み込まれたパフォーマンス・データの収集および分析フレームワークである。デフォルトでは 60 分ごとにパフォーマンス統計のスナップショットを自動的に取得し、SYSAUX 表領域に保存する。AWR データは、パフォーマンス問題の診断、ワークロード傾向の把握、および調整による変更の影響を検証するための基礎となる。

AWR レポートは 2 つのスナップショットを比較し、その間のアクティビティを要約する。これは、ほとんどの DBA がパフォーマンス・インシデントを調査する際に最初に使用するツールである。

**ライセンスに関する注意:** AWR は Oracle Diagnostics Pack の一部である。ベースのデータベース・ライセンスに加えてライセンスが必要である。本番環境で AWR を使用する前に、ライセンスを確認すること。

---

## 主要な概念

### スナップショット (Snapshots)

スナップショットは、V$ ビュー（DB ブロック取得数、解析回数、待機イベント合計など）からのすべての累積統計を、ある一時点で取得したものである。AWR レポートは 2 つのスナップショット間の増分（デルタ）を計算し、その期間中のアクティビティを表示する。

- デフォルトの保持期間: 8 日間
- デフォルトの取得間隔: 60 分
- 保存場所: `SYS` スキーマ下の `SYSAUX` 表領域（WRM$ および WRH$ 表）

### DB Time

**DB Time** は、AWR レポートにおいて最も重要な単一のメトリックである。これは、データベース・コールを実行しているすべてのフォアグラウンド・セッションが費やした合計経過時間（待機時間を含む）を表す。アイドル待機時間は含まれない。

```
DB Time = CPU Time + 非アイドル待機時間
```

1 秒あたりの DB Time が物理 CPU 数を超えている場合、キャパシティや効率性の問題が発生している。

### 経過時間 (Elapsed Time)

スナップショット間隔の実時間。DB Time を経過時間で割ることで、平均アクティブ・セッション数 (AAS) が得られる：

```
AAS = DB Time (秒) / 経過時間 (秒)
```

AAS が物理 CPU 数に近いかそれを超えている場合、多くの場合サチュレーション（飽和状態）を示している。

---

## DBMS_WORKLOAD_REPOSITORY によるスナップショット管理

### 手動スナップショットの作成

```sql
-- すぐにスナップショットを作成（変更の前後などで有用）
EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();

-- 作成されたか確認
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
ORDER  BY snap_id DESC
FETCH  FIRST 5 ROWS ONLY;
```

### AWR 設定の変更

```sql
-- 間隔を 30 分に変更し、保持期間を 14 日間に設定
BEGIN
  DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(
    retention => 14 * 24 * 60,  -- 分単位
    interval  => 30             -- 分単位
  );
END;
/

-- 現在の設定を確認
SELECT snap_interval, retention
FROM   dba_hist_wr_control;
```

### スナップショットの削除

```sql
-- SYSAUX 領域を解放するために、範囲を指定してスナップショットを削除
BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE(
    low_snap_id  => 100,
    high_snap_id => 150
  );
END;
/
```

### 特定の時間枠のスナップショット ID を探す

```sql
SELECT snap_id,
       TO_CHAR(begin_interval_time, 'YYYY-MM-DD HH24:MI') AS begin_time,
       TO_CHAR(end_interval_time,   'YYYY-MM-DD HH24:MI') AS end_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time >= SYSDATE - 1
ORDER  BY snap_id;
```

---

## AWR レポートの生成

### テキスト形式レポート (SQL*Plus)

```sql
-- 対話型: スナップショット ID とインスタンスの入力を求められる
@$ORACLE_HOME/rdbms/admin/awrrpt.sql

-- PL/SQL 関数を直接使用した非対話型
SELECT output
FROM   TABLE(
         DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_TEXT(
           l_dbid      => (SELECT dbid FROM v$database),
           l_inst_num  => 1,
           l_bid       => 200,   -- 開始スナップ ID
           l_eid       => 201    -- 終了スナップ ID
         )
       );
```

### HTML 形式レポート (可読性が高いため推奨)

```sql
SELECT output
FROM   TABLE(
         DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
           l_dbid      => (SELECT dbid FROM v$database),
           l_inst_num  => 1,
           l_bid       => 200,
           l_eid       => 201
         )
       );
```

### RAC グローバル AWR レポート

```sql
@$ORACLE_HOME/rdbms/admin/awrgrpt.sql
```

### 期間比較レポート (ベースライン vs 現在)

```sql
@$ORACLE_HOME/rdbms/admin/awrddrpt.sql
```

---

## 主要セクションの読み方

### 1. レポート・ヘッダー

データベースのバージョン、インスタンス名、ホスト、CPU、およびスナップショットの時間枠を確認できる。結論を出す前に、これらが対象の環境と一致していることを必ず確認すること。

### 2. Load Profile

主要なメトリックの 1 秒あたりおよび 1 トランザクションあたりのレートが表示される：

| メトリック | 意味 |
|---|---|
| DB Time(s) | 平均アクティブ・セッション（経過秒数で割る） |
| DB CPU(s) | 1 秒あたりに実際に消費された CPU |
| Redo size | 書き込み負荷の指標 |
| Logical reads | バッファ・キャッシュ I/O |
| Block changes | DML アクティビティ |
| Physical reads | 実際のディスク I/O |
| Hard parses | カーソル再利用の問題 |
| Parses | 合計解析コール（ハード + ソフト） |
| Logons | 接続の変動（チャーン） |

**ハード解析率が 100/秒を超える場合**、ほとんどの場合、バインド変数の欠如や接続プールの問題を意味する。

```sql
-- AWR 履歴からハード解析率を検証する
SELECT snap_id,
       hard_parses,
       hard_parses / elapsed_time_delta * 1e6 AS hard_parses_per_sec
FROM (
  SELECT snap_id,
         value                                                   AS hard_parses,
         LAG(value) OVER (ORDER BY snap_id)                     AS prev_val,
         (end_interval_time - begin_interval_time) * 86400      AS elapsed_time_delta
  FROM   dba_hist_sysstat s
  JOIN   dba_hist_snapshot sn USING (snap_id, dbid, instance_number)
  WHERE  stat_name = 'hard parses'
    AND  snap_id BETWEEN 200 AND 220
)
WHERE prev_val IS NOT NULL;
```

### 3. Instance Efficiency Percentages

| メトリック | 目標 | 懸念される値 |
|---|---|---|
| Buffer Cache Hit % | > 95% | < 90% |
| Library Cache Hit % | > 99% | < 95% |
| In-memory Sort % | > 95% | < 90% |
| Soft Parse % | > 95% | < 90% |
| Execute to Parse % | > 50% | 極端に低い値 |

**Buffer Nowait %** と **Redo Nowait %** は、どちらも 100% に近い必要がある。

### 4. Top 10 Foreground Wait Events

このセクションには、DB Time を最も多く消費したイベントがリストされる。非アイドル・イベントに注目すること：

```
Event                           Waits    Time(s)  Avg wait  % DB time
------------------------------- -------- -------- --------- ---------
db file sequential read         450,321  1,823.4     4.05ms    18.2%
log file sync                    89,234    412.1     4.62ms     4.1%
buffer busy waits                12,456    234.5    18.83ms     2.3%
```

注意すべきイベント:

| イベント名 | 一般的な原因 |
|---|---|
| `db file sequential read` | シングル・ブロック I/O、索引スキャン、行の取得 |
| `db file scattered read` | 全表/索引スキャン（マルチブロック読み込み） |
| `log file sync` | COMMIT の頻度、Redo ログ I/O |
| `buffer busy waits` | ホット・ブロック、セグメント・ヘッダーの競合 |
| `enq: TX - row lock contention` | 行レベルのロック、アプリケーションの設計問題 |
| `library cache: mutex X` | ハード解析、カーソル共有の問題 |
| `latch: cache buffers chains` | バッファ・キャッシュ内のホット・ブロック |

### 5. SQL Statistics

AWR は Top SQL をいくつかのサブセクションに分解して表示する：

- **SQL ordered by Elapsed Time** — 調査の開始点として最適。
- **SQL ordered by CPU Time** — CPU 負荷の高いクエリ。
- **SQL ordered by Gets** — 論理 I/O 負荷が高いクエリ。
- **SQL ordered by Reads** — 物理 I/O 負荷が高いクエリ。
- **SQL ordered by Executions** — 実行頻度。コストの低い SQL でも大規模環境では重要。
- **SQL ordered by Parse Calls** — カーソル再利用の問題。

各 SQL 項目について、SQL ID、実行回数、実行あたりの経過時間、および SQL テキストの冒頭を確認すること。

```sql
-- AWR 履歴からプログラムで Top SQL を取得する
SELECT sql_id,
       ROUND(elapsed_time_total / 1e6, 2)          AS total_elapsed_sec,
       executions_total,
       ROUND(elapsed_time_total / NULLIF(executions_total,0) / 1e6, 4) AS avg_elapsed_sec,
       SUBSTR(sql_text, 1, 80)                      AS sql_text
FROM   dba_hist_sqlstat
JOIN   dba_hist_sqltext USING (sql_id, dbid)
WHERE  snap_id BETWEEN 200 AND 201
ORDER  BY elapsed_time_total DESC
FETCH  FIRST 20 ROWS ONLY;
```

### 6. Segments Statistics

どのセグメントが最も多くの I/O やバッファ取得を消費しているかを示す。ホットな表や索引を特定するのに役立つ。

### 7. Dictionary Cache と Library Cache

ここでのミス率が高い場合は、共有プール（Shared Pool）の圧迫を示している。

```sql
-- 現在のライブラリ・キャッシュのパフォーマンス
SELECT namespace,
       gets,
       gethits,
       ROUND(gethitratio * 100, 2) AS hit_pct,
       pins,
       pinhits,
       ROUND(pinhitratio * 100, 2) AS pin_hit_pct,
       reloads,
       invalidations
FROM   v$librarycache
ORDER  BY gets DESC; intermission

-- ディクショナリ・キャッシュのミス (2% 未満であるべき)
SELECT parameter,
       gets,
       getmisses,
       ROUND(getmisses / NULLIF(gets,0) * 100, 2) AS miss_pct
FROM   v$rowcache
WHERE  gets > 0
ORDER  BY getmisses DESC
FETCH  FIRST 15 ROWS ONLY;
```

---

## AWR によるボトルネックの特定

### CPU バウンド (CPU Bound)

- DB CPU が DB Time に近いか、それを超えている。
- 解析 CPU (Parse CPU) や実行 CPU (Execute CPU) が高い。
- 全表スキャン、索引の不足、非効率な SQL を確認すること。

### I/O バウンド (I/O Bound)

- 待機イベントの上位に `db file sequential read` や `db file scattered read` がある。
- Load Profile の物理読み込み回数が高い。
- Reads 順の Top SQL を調査する。索引の追加やストレージの最適化を検討すること。

### 競合バウンド (Contention Bound)

- `buffer busy waits`、`enq:` 待機、ラッチ待機が支配的である。
- 多くの場合、アプリケーションの設計上の問題（ホットなシーケンス、反転キー索引の必要性など）である。

### メモリ圧迫 (Memory Pressure)

- ソフト解析ミス率が高い、またはライブラリ・キャッシュのミス率が 1% を超えている。
- 待機イベントに `free buffer waits` が現れる（バッファ・キャッシュが不足している）。
- OS 統計セクションで高い `paged-in` 値が見られる。

### Redo / コミットのオーバーヘッド

- 待機イベントの上位に `log file sync` がある。
- Load Profile で 1 秒あたりの Redo サイズが高い。
- 非同期コミット、コミットのバッチ化、またはより高速なストレージを検討すること。

---

## AWR ベースライン

ベースラインはスナップショットの範囲を保存し、通常の保持期間による削除の対象外とする。これにより、期間比較レポートが可能になる。

```sql
-- 固定ベースラインの作成
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_BASELINE(
    start_snap_id => 200,
    end_snap_id   => 210,
    baseline_name => 'PRE_PATCH_BASELINE',
    expiration    => 30  -- 日数、NULL は期限なし
  );
END;
/

-- 既存のベースラインを確認
SELECT baseline_name, start_snap_id, end_snap_id, expiration
FROM   dba_hist_baseline;

-- ベースラインの削除
BEGIN
  DBMS_WORKLOAD_REPOSITORY.DROP_BASELINE(
    baseline_name => 'PRE_PATCH_BASELINE',
    cascade       => FALSE  -- TRUE の場合、スナップショットも削除される
  );
END;
/
```

---

## ベスト・プラクティス

- **変更（パッチ、スキーマ変更、パラメータ変更）の前後で必ずスナップショットを取得する。** これにより、正確な前後比較レポートを生成できる。
- **対話型分析には HTML レポートを使用し**、テキスト版は自動解析に使用するのが適している。
- **DB Time への寄与度に注目する。** 待機回数は数百万回でも、合計経過時間が非常に短いイベントは問題ではない。
- **AWR を OS 統計と関連付ける。** OS セクションの CPU 使用率、メモリ・ページング、およびディスク I/O は、データベース内の指標を裏付ける、あるいは矛盾を指摘する材料となる。
- **重要なシステムでは保持期間を少なくとも 30 日間に増やす。** 主要なマイルストーン（アップグレードの前後など）ではベースラインを保持すること。
- **忙しいシステムで AWR スナップショットを頻繁に（15 分未満）実行しない。** SYSAUX への書き込みオーバーヘッドが増大するためである。
- **SYSAUX 領域を監視する。** AWR データは保持期間とスナップショット頻度に応じて増大する。`v$sysaux_occupants` をクエリして AWR の使用状況を確認すること。

```sql
-- SYSAUX 内の AWR の使用量を確認する
SELECT occupant_name,
       schema_name,
       ROUND(space_usage_kbytes / 1024, 2) AS space_mb
FROM   v$sysaux_occupants
WHERE  occupant_name LIKE 'SM/%'
ORDER  BY space_usage_kbytes DESC;
```

---

## よくある間違い

| 間違い | 問題点 | 対策 |
|---|---|---|
| 夏時間 (DST) の変更をまたいでスナップショットを比較する | 経過時間の計算が正しくなくなる | タイムゾーンの遷移に注意する。可能であれば UTC ベースのタイムスタンプを使用する |
| 5 分間のスパイクを調査するために 1 時間のスナップショットを分析する | スパイクが希釈されてしまう | ターゲットを絞った手動スナップショットを取得するか、秒単位の分析には ASH を使用する |
| 「1 トランザクションあたり」の列を無視する | ワークロードの特性の変化を見逃す | 1 秒あたりと 1 トランザクションあたりの両方のレートを比較する |
| 待機時間ではなく、待機回数に注目する | 誤った結論を導く | 常に Time(s) 列を最優先のソート基準とする |
| SQL の実行回数を確認しない | 低コストでも 100 万回実行される SQL は「遅い」可能性がある | 「平均経過時間 × 実行回数」で全体への影響を算出する |
| バッファ・キャッシュ・ヒット率を絶対視する | ヒット率が 99% でも、ワークロードが膨大であれば I/O 問題は発生し得る | 物理読み込み回数を絶対的な数値で確認する |
| 1 分未満のインシデントに AWR を使用する | 解像度が不足している | リアルタイムのドリルダウンには ASH (V$ACTIVE_SESSION_HISTORY) を使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Performance Tuning Guide (TGDBA)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)
- [DBMS_WORKLOAD_REPOSITORY — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_WORKLOAD_REPOSITORY.html)
- [DBA_HIST_SNAPSHOT — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_SNAPSHOT.html)
- [DBA_HIST_WR_CONTROL — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_WR_CONTROL.html)
- [DBA_HIST_SQLSTAT — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_SQLSTAT.html)
- [V$LIBRARYCACHE — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LIBRARYCACHE.html)
- [V$ROWCACHE — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-ROWCACHE.html)
