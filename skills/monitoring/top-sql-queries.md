# Top SQL クエリと SQL モニタリング

## 概要

最もリソースを消費する SQL ステートメントを特定し、チューニングすることは、DBA が行うことができる最も効果の高い活動の 1 つである。たった 1 つの不適切に書かれたクエリが、高負荷なシステムの CPU の 90% を消費することもあり、それに対処することで、すべてのユーザーに影響を与えるパフォーマンス向上を実現できる。Oracle は、これらのクエリを見つけるための複数のビューとツールを提供している。リアルタイムの `V$SQL` や `V$SQLAREA` ビューから、`DBA_HIST_SQLSTAT` のような過去の自動ワークロード・リポジトリ (AWR) の表まで多岐にわたる。

このガイドでは、さまざまなリポジトリ次元で Top SQL を見つけるための主要なビュー、AWR に基づく履歴分析、および実行中に長時間実行クエリの可視性を提供するリアルタイム SQL モニタリング機能 (`V$SQL_MONITOR`) について説明する。

---

## 主要なビュー: V$SQL と V$SQLAREA

### V$SQL vs V$SQLAREA

| ビュー | 粒度 | ユースケース |
|------|-------------|----------|
| `V$SQL` | 子カーソルごと (実行コンテキストごと) に 1 行 | バインド変数の覗き見、セッション固有の統計を含む詳細な分析 |
| `V$SQLAREA` | 親カーソルごとに 1 行 (子カーソルを集約) | 同じ SQL テキストのすべての実行にわたるサマリー統計 |

実際には、まず `V$SQLAREA` で総リソース消費量が多い Top-N クエリを特定し、次に `V$SQL` で特定の実行計画やバインド変数の詳細を調査する。

### 重要な列

```sql
-- 利用可能な列（最も重要なサブセット）を探索する
SELECT column_name, comments
FROM   dict_columns
WHERE  table_name = 'V$SQLAREA'
  AND  column_name IN (
       'SQL_ID','SQL_TEXT','PARSING_SCHEMA_NAME',
       'EXECUTIONS','ELAPSED_TIME','CPU_TIME',
       'BUFFER_GETS','DISK_READS','ROWS_PROCESSED',
       'SORTS','FETCHES','LOADS','INVALIDATIONS',
       'SHARABLE_MEM','PERSISTENT_MEM','RUNTIME_MEM',
       'FIRST_LOAD_TIME','LAST_ACTIVE_TIME',
       'PARSING_USER_ID','CHILD_NUMBER','PLAN_HASH_VALUE'
       );
```

主要な列の説明:

| 列名 | 単位 | 説明 |
|--------|------|-------------|
| `SQL_ID` | — | 一意の SQL 識別子 (14 文字のハッシュ) |
| `EXECUTIONS` | 回数 | このカーソルが実行された回数 |
| `ELAPSED_TIME` | マイクロ秒 | すべての実行にわたる実時間の合計 |
| `CPU_TIME` | マイクロ秒 | すべての実行にわたる CPU 時間の合計 |
| `BUFFER_GETS` | ブロック数 | 論理読み取り (メモリー読み取り) の合計 |
| `DISK_READS` | ブロック数 | ディスクからの物理読み取りの合計 |
| `ROWS_PROCESSED` | 行数 | 返された/処理された行の合計 |
| `SORTS` | 回数 | 実行されたソートの合計 |
| `SHARABLE_MEM` | バイト | 共有プール内のこのカーソルで使用されるメモリー |
| `LAST_ACTIVE_TIME` | 日付 | このカーソルが最後にアクティブになった時刻 |
| `PLAN_HASH_VALUE` | 数値 | 現在の実行計画のハッシュ値 |

---

## リソース次元による Top SQL の特定

### CPU 時間による Top SQL

CPU 負荷の高いクエリは、全表スキャン、過度なソート、または適切なフィルタリングなしでの大規模な結果セットの処理を行っていることが多い。

```sql
-- 総 CPU 時間（すべての実行の合計）による Top 20 SQL
SELECT sql_id,
       ROUND(cpu_time / 1e6, 1)              AS total_cpu_sec,
       ROUND(cpu_time / NULLIF(executions, 0) / 1e6, 3) AS avg_cpu_sec,
       executions,
       ROUND(elapsed_time / 1e6, 1)          AS total_elapsed_sec,
       buffer_gets,
       disk_reads,
       parsing_schema_name,
       SUBSTR(sql_text, 1, 80)               AS sql_preview
FROM   v$sqlarea
WHERE  executions > 0
ORDER BY cpu_time DESC
FETCH FIRST 20 ROWS ONLY;
```

### 経過時間による Top SQL

経過時間 (Elapsed Time) は、待機時間 (I/O、ロック、ネットワーク) を含む実時間を捉える。CPU 時間は短いが経過時間が長い場合は、待機にボトルネックがある SQL を示している。

```sql
-- 総経過時間による Top 20 SQL
SELECT sql_id,
       ROUND(elapsed_time / 1e6, 1)               AS total_elapsed_sec,
       ROUND(elapsed_time / NULLIF(executions, 0) / 1e6, 3) AS avg_elapsed_sec,
       executions,
       ROUND(cpu_time    / 1e6, 1)                AS total_cpu_sec,
       ROUND((elapsed_time - cpu_time) / 1e6, 1)  AS total_wait_sec,
       ROUND((elapsed_time - cpu_time) * 100.0
             / NULLIF(elapsed_time, 0), 1)         AS wait_pct,
       parsing_schema_name,
       SUBSTR(sql_text, 1, 80)                     AS sql_preview
FROM   v$sqlarea
WHERE  executions > 0
ORDER BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;
```

### 物理 I/O (ディスク読み取り) による Top SQL

ディスク読み取りが多い場合は、インデックスの欠落や未使用、大規模な表の全表スキャン、またはバッファ・キャッシュ・サイズの不足を示していることが多い。

```sql
-- 総物理読み取りによる Top 20 SQL
SELECT sql_id,
       disk_reads,
       ROUND(disk_reads / NULLIF(executions, 0), 0) AS avg_disk_reads,
       buffer_gets,
       ROUND(buffer_gets / NULLIF(executions, 0), 0) AS avg_buffer_gets,
       executions,
       ROUND(elapsed_time / 1e6, 1)                  AS total_elapsed_sec,
       parsing_schema_name,
       SUBSTR(sql_text, 1, 80)                        AS sql_preview
FROM   v$sqlarea
WHERE  executions > 0
  AND  disk_reads > 0
ORDER BY disk_reads DESC
FETCH FIRST 20 ROWS ONLY;
```

### 論理読み取り (バッファ・ゲット) による Top SQL

バッファ・ゲットが多い場合は、全表スキャン、非効率なインデックス使用、または必要以上のデータをスキャンするクエリを示している。これはメモリー圧迫の目安となる。

```sql
-- 総論理読み取り (論理 I/O) による Top 20 SQL
SELECT sql_id,
       buffer_gets,
       ROUND(buffer_gets / NULLIF(executions, 0), 0) AS avg_buffer_gets,
       ROUND(disk_reads  / NULLIF(buffer_gets,   0) * 100, 1) AS cache_miss_pct,
       executions,
       ROUND(cpu_time / 1e6, 1) AS total_cpu_sec,
       parsing_schema_name,
       SUBSTR(sql_text, 1, 80)  AS sql_preview
FROM   v$sqlarea
WHERE  executions  > 0
  AND  buffer_gets > 0
ORDER BY buffer_gets DESC
FETCH FIRST 20 ROWS ONLY;
```

### 高頻度クエリ vs 高負荷クエリ

実行あたりのコストは高いクエリもあれば、実行あたりのコストは低いが数千回実行されるクエリもある。これらを区別する：

```sql
-- 実行頻度が高く、実行あたりのコストが低いクエリ (結果キャッシュやバッチ化による最適化の可能性)
SELECT sql_id,
       executions,
       ROUND(elapsed_time / NULLIF(executions, 0) / 1000, 2) AS avg_ms,
       ROUND(elapsed_time / 1e6, 1)   AS total_elapsed_sec,
       ROUND(cpu_time     / 1e6, 1)   AS total_cpu_sec,
       SUBSTR(sql_text, 1, 80)        AS sql_preview
FROM   v$sqlarea
WHERE  executions > 1000
ORDER BY executions DESC
FETCH FIRST 20 ROWS ONLY;
```

```sql
-- 多次元ランキング：加重インパクト・スコアによる Top SQL
SELECT sql_id,
       executions,
       ROUND(elapsed_time / 1e6, 1)               AS elapsed_sec,
       ROUND(cpu_time     / 1e6, 1)               AS cpu_sec,
       buffer_gets,
       disk_reads,
       -- 加重複合スコア
       ROUND(
           (elapsed_time / NULLIF(MAX(elapsed_time) OVER (), 0) * 40) +
           (cpu_time     / NULLIF(MAX(cpu_time)     OVER (), 0) * 30) +
           (buffer_gets  / NULLIF(MAX(buffer_gets)  OVER (), 0) * 20) +
           (disk_reads   / NULLIF(MAX(disk_reads)   OVER (), 0) * 10)
       , 2) AS impact_score,
       SUBSTR(sql_text, 1, 80) AS sql_preview
FROM   v$sqlarea
WHERE  executions > 0
ORDER BY impact_score DESC
FETCH FIRST 30 ROWS ONLY;
```

### 完全な SQL テキストの取得

`V$SQLAREA.SQL_TEXT` は 1000 文字に切り捨てられる。完全なステートメントを取得するには `V$SQLTEXT` を使用する：

```sql
-- 指定した SQL_ID の完全な SQL テキストを取得する
SELECT piece,
       sql_text
FROM   v$sqltext
WHERE  sql_id    = 'abc123def456g'
ORDER BY piece;
```

```sql
-- または V$SQL.SQL_FULLTEXT を使用する (CLOB, 11g 以降で利用可能)
SELECT sql_fulltext
FROM   v$sql
WHERE  sql_id = 'abc123def456g'
  AND  child_number = 0;
```

---

## AWR Top SQL クエリ

自動ワークロード・リポジトリ (AWR) は、主要な統計情報のスナップショットを 1 時間おきに (デフォルト) キャプチャする。この履歴データは `DBA_HIST_*` 表に保存され、時間の経過に伴う傾向分析や期間の比較を可能にする。

**必須:** 本番環境で AWR データを使用するには、Oracle Diagnostics Pack のライセンスが必要である。

### スナップショット範囲の特定

```sql
-- 利用可能な AWR スナップショット
SELECT snap_id,
       begin_interval_time,
       end_interval_time
FROM   dba_hist_snapshot
ORDER BY snap_id DESC
FETCH FIRST 24 ROWS ONLY;  -- 直近 24 スナップショット (通常 24 時間分)
```

### DBA_HIST_SQLSTAT: AWR SQL 統計の核心

`DBA_HIST_SQLSTAT` は、スナップショットごとの SQL 統計を保存し、SQL テキストについては `DBA_HIST_SQLTEXT` とリンクされている。

```sql
-- 指定した 2 つのスナップショット間での経過時間による Top SQL
SELECT st.sql_id,
       ROUND(SUM(st.elapsed_time_delta) / 1e6, 1)        AS elapsed_sec,
       ROUND(SUM(st.cpu_time_delta)     / 1e6, 1)        AS cpu_sec,
       SUM(st.executions_delta)                           AS executions,
       ROUND(SUM(st.elapsed_time_delta)
             / NULLIF(SUM(st.executions_delta), 0) / 1e6, 3) AS avg_elapsed_sec,
       SUM(st.buffer_gets_delta)                          AS buffer_gets,
       SUM(st.disk_reads_delta)                           AS disk_reads,
       SUBSTR(t.sql_text, 1, 80)                          AS sql_preview
FROM   dba_hist_sqlstat  st
JOIN   dba_hist_sqltext  t  ON st.sql_id = t.sql_id AND st.dbid = t.dbid
WHERE  st.snap_id BETWEEN 100 AND 120   -- 自身のスナップショット ID に置き換える
  AND  st.dbid = (SELECT dbid FROM v$database)
  AND  st.executions_delta > 0
GROUP BY st.sql_id, t.sql_text
ORDER BY elapsed_sec DESC
FETCH FIRST 20 ROWS ONLY;
```

### 日付範囲による AWR CPU 回数 Top SQL

```sql
-- AWR を使用した、過去 7 日間の CPU 負荷の高い SQL
SELECT st.sql_id,
       ROUND(SUM(st.cpu_time_delta) / 1e6, 1)           AS total_cpu_sec,
       SUM(st.executions_delta)                           AS executions,
       ROUND(SUM(st.cpu_time_delta)
             / NULLIF(SUM(st.executions_delta), 0) / 1e6, 3) AS avg_cpu_sec,
       ROUND(SUM(st.elapsed_time_delta) / 1e6, 1)        AS total_elapsed_sec,
       SUBSTR(t.sql_text, 1, 80)                          AS sql_preview
FROM   dba_hist_sqlstat  st
JOIN   dba_hist_sqltext  t  ON st.sql_id = t.sql_id AND st.dbid = t.dbid
JOIN   dba_hist_snapshot sn ON st.snap_id = sn.snap_id AND st.dbid = sn.dbid
WHERE  sn.begin_interval_time > SYSDATE - 7
  AND  st.executions_delta > 0
GROUP BY st.sql_id, t.sql_text
ORDER BY total_cpu_sec DESC
FETCH FIRST 20 ROWS ONLY;
```

### 時間経過に伴う SQL パフォーマンスの追跡 (退行の検出)

```sql
-- 特定の SQL の平均経過時間を AWR スナップショット間で比較する
-- 実行計画の変更やパフォーマンスの退行を検出するのに有用
SELECT sn.snap_id,
       sn.begin_interval_time,
       st.plan_hash_value,
       st.executions_delta,
       ROUND(st.elapsed_time_delta / NULLIF(st.executions_delta, 0) / 1e6, 4) AS avg_elapsed_sec,
       ROUND(st.cpu_time_delta     / NULLIF(st.executions_delta, 0) / 1e6, 4) AS avg_cpu_sec,
       ROUND(st.buffer_gets_delta  / NULLIF(st.executions_delta, 0), 0)       AS avg_buffer_gets
FROM   dba_hist_sqlstat  st
JOIN   dba_hist_snapshot sn ON st.snap_id = sn.snap_id AND st.dbid = sn.dbid
WHERE  st.sql_id = 'abc123def456g'
  AND  sn.begin_interval_time > SYSDATE - 30
  AND  st.executions_delta > 0
ORDER BY sn.snap_id;
```

`avg_elapsed_sec` の急激な増加と `plan_hash_value` の変更が組み合わさっている場合は、実行計画の変更が退行の原因であることを示している。

### AWR SQL 実行計画

```sql
-- 特定の SQL_ID に対して AWR に保存されている実行計画
SELECT plan_hash_value,
       timestamp,
       operation,
       options,
       object_owner,
       object_name,
       cost,
       cardinality,
       bytes
FROM   dba_hist_sql_plan
WHERE  sql_id = 'abc123def456g'
  AND  dbid   = (SELECT dbid FROM v$database)
ORDER BY plan_hash_value, id;
```

```sql
-- DBMS_XPLAN を使用して AWR から書式設定された実行計画を生成する
SELECT *
FROM   TABLE(DBMS_XPLAN.display_awr(
               sql_id         => 'abc123def456g',
               plan_hash_value => NULL,   -- NULL の場合はすべての計画を表示
               db_id          => NULL,    -- NULL の場合は現在の DB を使用
               format         => 'ALL'
           ));
```

---

## V$SQL_MONITOR: リアルタイム SQL モニタリング

SQL モニタリングは Oracle 11g で導入され、長時間実行される SQL ステートメントに対してリアルタイムで実行ごとの可視性を提供する。以下のいずれかの条件を満たすステートメントに対して、自動的に有効になる：
- 5 秒以上の CPU 時間または I/O 時間を消費する
- パラレル実行を使用する
- `MONITOR` ヒントが適用されている

**必須:** Oracle Diagnostics Pack および Tuning Pack のライセンスが必要である。

### 監視対象のアクティブな SQL の表示

```sql
-- モニタリングが有効で、現在実行中の SQL
SELECT sql_id,
       sql_exec_id,
       status,
       username,
       ROUND(elapsed_time / 1e6, 1)    AS elapsed_sec,
       ROUND(cpu_time     / 1e6, 1)    AS cpu_sec,
       ROUND(queuing_time / 1e6, 1)    AS queue_sec,
       buffer_gets,
       disk_reads,
       ROUND(physical_read_bytes  / 1048576, 1) AS phys_read_mb,
       ROUND(physical_write_bytes / 1048576, 1) AS phys_write_mb,
       ROUND(io_interconnect_bytes / 1048576, 1) AS io_ic_mb,
       px_servers_requested,
       px_servers_allocated,
       SUBSTR(sql_text, 1, 80)         AS sql_preview
FROM   v$sql_monitor
WHERE  status = 'EXECUTING'
ORDER BY elapsed_time DESC;
```

```sql
-- 最近完了した監視対象の SQL (過去 1 時間)
SELECT sql_id,
       sql_exec_start,
       status,
       username,
       ROUND(elapsed_time / 1e6, 1)    AS elapsed_sec,
       ROUND(cpu_time     / 1e6, 1)    AS cpu_sec,
       buffer_gets,
       disk_reads,
       ROUND(physical_read_bytes  / 1048576, 1) AS phys_read_mb,
       SUBSTR(sql_text, 1, 80)         AS sql_preview
FROM   v$sql_monitor
WHERE  sql_exec_start > SYSDATE - 1/24
  AND  status != 'EXECUTING'
ORDER BY elapsed_time DESC
FETCH FIRST 30 ROWS ONLY;
```

### V$SQL_PLAN_MONITOR による実行計画レベルの監視

`V$SQL_PLAN_MONITOR` は、**実行計画の操作レベル**での統計情報を提供する。計画内のどのステップが最もリソースを消費しているかを正確に示す：

```sql
-- 実行中または最近完了したクエリの操作ごとの統計
SELECT pm.plan_line_id       AS line,
       pm.plan_operation     AS operation,
       pm.plan_options,
       pm.plan_object_name   AS object_name,
       pm.output_rows,
       pm.starts,
       pm.workarea_mem       AS mem_bytes,
       pm.workarea_tempseg   AS temp_bytes,
       ROUND(pm.elapsed_time / 1e6, 3) AS elapsed_sec,
       ROUND(pm.cpu_time     / 1e6, 3) AS cpu_sec,
       pm.physical_read_requests       AS phys_reads,
       pm.physical_write_requests      AS phys_writes
FROM   v$sql_plan_monitor pm
WHERE  pm.sql_id      = 'abc123def456g'
  AND  pm.sql_exec_id = 16777216     -- V$SQL_MONITOR から取得する
ORDER BY pm.plan_line_id;
```

### SQL モニタ報告書の生成

SQL モニタリング・データを表示する最も強力な方法は、書式設定された HTML またはテキスト形式の報告書である。これらは実行計画、統計、およびタイミングの可視化を提供する：

```sql
-- HTML 形式の SQL モニタ報告書を生成する (ブラウザ表示に最適)
SELECT DBMS_SQLTUNE.report_sql_monitor(
           sql_id     => 'abc123def456g',
           type       => 'HTML',
           report_level => 'ALL'
       ) AS report
FROM dual;
```

```sql
-- テキスト形式の報告書を生成する (メールやターミナルに適している)
SELECT DBMS_SQLTUNE.report_sql_monitor(
           sql_id     => 'abc123def456g',
           type       => 'TEXT',
           report_level => 'ALL'
       ) AS report
FROM dual;
```

```sql
-- 特定の実行に関する報告書を生成する (V$SQL_MONITOR の sql_exec_id を使用する)
SELECT DBMS_SQLTUNE.report_sql_monitor(
           sql_id     => 'abc123def456g',
           sql_exec_id => 16777216,
           type        => 'HTML',
           report_level => 'ALL'
       ) AS report
FROM dual;
```

```sql
-- アクティブなモニタリング報告書を生成する (ブラウザでリアルタイムに自動更新される)
SELECT DBMS_SQLTUNE.report_sql_monitor(
           sql_id       => 'abc123def456g',
           type         => 'ACTIVE',
           report_level => 'ALL'
       ) AS report
FROM dual;
```

HTML 出力を `.html` ファイルに保存し、ブラウザで開くと、以下の内容を含むインタラクティブな報告書を確認できる：
- リアルタイムの実行進捗
- 操作ごとの統計とタイミング・バー
- 操作ごとのメモリーおよび一時領域の使用量
- パラレル実行サーバーの分散状況
- 操作ごとの I/O 統計情報

---

## V$ ビューと AWR データの組み合わせ：実践的なワークフロー

### ステップ 1: 現在最も負荷の高い SQL を特定する

```sql
-- 現在の負荷が高い上位 5 件を迅速に特定
SELECT sql_id,
       ROUND(elapsed_time / 1e6, 1) AS elapsed_sec,
       ROUND(cpu_time     / 1e6, 1) AS cpu_sec,
       buffer_gets,
       disk_reads,
       executions,
       SUBSTR(sql_text, 1, 60)      AS preview
FROM   v$sqlarea
ORDER BY elapsed_time DESC
FETCH FIRST 5 ROWS ONLY;
```

### ステップ 2: これが新しい問題か再発している問題かを確認する

```sql
-- この SQL の過去のパフォーマンスはどうだったか？
SELECT sn.begin_interval_time,
       st.executions_delta,
       ROUND(st.elapsed_time_delta / NULLIF(st.executions_delta, 0) / 1e6, 3) AS avg_elapsed_sec,
       st.plan_hash_value
FROM   dba_hist_sqlstat  st
JOIN   dba_hist_snapshot sn ON st.snap_id = sn.snap_id AND st.dbid = sn.dbid
WHERE  st.sql_id = 'abc123def456g'
  AND  sn.begin_interval_time > SYSDATE - 14
  AND  st.executions_delta > 0
ORDER BY sn.snap_id DESC;
```

### ステップ 3: 現在の実行計画を表示する

```sql
-- ライブラリ・キャッシュから現在の計画を取得
SELECT *
FROM   TABLE(DBMS_XPLAN.display_cursor(
               sql_id       => 'abc123def456g',
               cursor_child_no => 0,
               format       => 'ALLSTATS LAST'
           ));
```

### ステップ 4: SQL モニタ報告書を生成する

```sql
-- この SQL の最新の sql_exec_id を取得
SELECT sql_exec_id, sql_exec_start, elapsed_time/1e6 AS elapsed_sec, status
FROM   v$sql_monitor
WHERE  sql_id = 'abc123def456g'
ORDER BY sql_exec_start DESC
FETCH FIRST 1 ROW ONLY;
```

その後、上記で示したように HTML 報告書を生成する。

---

## ベスト・プラクティス

1. **CPU 時間ではなく、経過時間でソートされた `V$SQLAREA` から始めること。** 経過時間には、待機時間を含むユーザー・エクスペリエンスのすべてが含まれる。実時間の 10 分のうち 100 秒の CPU を使用するクエリよりも、10 秒の CPU を使用しながら他のプロセスを 30 分間ブロックしているクエリの方が緊急性が高い。

2. **実行あたりのコストを確認するために、常に実行回数で割ること。** クエリが 100 万回実行される場合、総経過時間は誤解を招く可能性がある。実行あたりの平均を比較して、最適化の焦点を実行回数の削減（結果キャッシュ、アプリケーション・ロジックの改善）に置くべきか、それとも実行あたりのコストの削減（インデックス作成、実行計画の変更）に置くべきかを判断する。

3. **実行計画の変更を検出するには `plan_hash_value` を使用すること。** AWR 履歴における `plan_hash_value` の急激な変化は、ほぼ間違いなくパフォーマンスの変化に対応している。退行を調査する際は、これを念頭に置くこと。

4. **重大なインシデントについては SQL モニタ報告書を保存しておくこと。** HTML 形式の SQL モニタ報告書は、事後分析において非常に貴重な資料となる。モニタリング・データが期限切れになる前に（完了したステートメントのデフォルト保持期間は 30 日）、共有の場所に保存しておく。

5. **重要な短時間クエリに対してモニタリングを強制するには `MONITOR` ヒントを使用すること。** デフォルトでは、SQL モニタリングは長時間実行されるクエリに対してのみ有効になる。重要だが短時間のクエリ（5 秒未満だが頻度が高い）については、調査中に `/*+ MONITOR */` を追加して操作ごとの統計をキャプチャする。

6. **バッファ・ゲットの前に `disk_reads/executions` の比率を調査すること。** 物理 I/O は論理 I/O よりも桁違いに遅い。1 実行あたりの物理ブロック読み取りが 10,000 件あるクエリは、キャッシュから 1,000,000 ブロックを読み取るクエリよりも優先順位を高くすべきである。

7. **月次のキャパシティ・プランニングには `DBA_HIST_SQLSTAT` を使用すること。** SQL リソース消費の数か月にわたる傾向を把握することで、現在のハードウェアがいつ飽和状態に達するかを予測し、インフラ変更のためのビジネス・ケースを作成するのに役立つ。

---

## よくある間違いと回避策

**間違い: 実行中かどうかを確認せずに V$SQLAREA に基づいてチューニングを行う。**
`V$SQLAREA` には、最後のカーソルのフラッシュまたはインスタンスの再起動以降のカーソル・データが保持されている。経過時間が最も長いクエリでも、数日間実行されていない可能性がある。チューニングを行う前に、必ず `LAST_ACTIVE_TIME` を確認すること。

**間違い: 完全な SQL テキストを取得せずに V$SQL または V$SQLAREA を使用する。**
`V$SQLAREA` の `SQL_TEXT` は 1000 文字に切り捨てられている。完全なテキストがなければ、クエリを正しく特定したり分析したりすることはできない。チューニングのためにクエリを送信する際は、必ず `V$SQLTEXT` または `V$SQL.SQL_FULLTEXT` を使用すること。

**間違い: 解析 (Parse) アクティビティを無視する。**
`V$SQLAREA` の `LOADS` や `INVALIDATIONS` カウントが高い場合は、カーソルが繰り返しハード・パースまたは無効化されていることを示している。これは経過時間のランキングには目立って現れないが、深刻なスケーラビリティの問題を引き起こす。`LOADS/EXECUTIONS` 比率を監視すること。

**間違い: スナップショット期間の違いを考慮せずに AWR 統計を比較する。**
AWR の `_delta` 値は、スナップショットの間隔中に蓄積されたものである。2 時間のスナップショットには、1 時間のスナップショットの 2 倍のデルタ値が含まれる。長さの異なるスナップショット間を比較する場合は、間隔の時間で割って正規化すること。

**間違い: SQL モニタリングにライセンスが必要であることを忘れる。**
`V$SQL_MONITOR` および `DBMS_SQLTUNE.report_sql_monitor` の使用には、Oracle Diagnostics Pack および Tuning Pack が必要である。ライセンスのない環境では、代わりに `V$SQL` および `DBMS_XPLAN.display_cursor` を使用すること。これらは追加ライセンスを必要としない。

**間違い: リアルタイムの診断に AWR だけを頼る。**
AWR スナップショットは通常 1 時間おきに取得される。リアルタイムのパフォーマンス危機発生時は、`V$SQL`、`V$SESSION`、`V$SESSION_WAIT`、および `V$SQL_MONITOR` を使用してリアルタイムの情報を取得すること。AWR は傾向分析や事後分析のためのものである。

---


## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 19c と 26ai では、リリース更新によってデフォルト設定や非推奨機能が異なる場合があるため、両方の環境をサポートする場合は構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Tuning Guide — Identifying High-Load SQL](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/identifying-high-load-sql.html)
- [Oracle Database 19c Reference — V$SQLAREA](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQLAREA.html)
- [Oracle Database 19c Reference — V$SQL_MONITOR](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQL_MONITOR.html)
- [Oracle Database 19c Reference — DBA_HIST_SQLSTAT](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_SQLSTAT.html)
- [Oracle Database 19c PL/SQL Packages Reference — DBMS_SQLTUNE (report_sql_monitor)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQLTUNE.html)
