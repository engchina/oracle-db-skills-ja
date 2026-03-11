# ASH 分析 (Active Session History)

## 概要

**Active Session History (ASH)** は、Oracle のインメモリでサンプリングされたセッション・アクティビティ・リポジトリである。Oracle は 1 秒ごとに、アクティブな（アイドル状態ではない）すべてのセッションをサンプリングし、アクティブ・セッションごとに 1 行を循環インメモリ・バッファ (`V$ACTIVE_SESSION_HISTORY`) に記録する。ディスクにフラッシュされた ASH データは `DBA_HIST_ACTIVE_SESS_HISTORY` に保存され、AWR の一部として保持される。

ASH は、AWR（スナップショット・レベル、粗い粒度）とリアルタイムの V$ ビュー（現時点のみ）の間の重要なギャップを埋めるものである。これにより、継続的な監視を必要とせずに、1 秒ごとの遡及的な分析が可能になる。

**ライセンスに関する注意:** ASH は Oracle Diagnostics Pack の一部であり、ベースのデータベース・ライセンスとは別にライセンスが必要である。

---

## 主要な概念

### サンプリング・メカニズム

- Oracle は、アクティブな（アイドルでない）すべてのセッションを **1 秒に 1 回** サンプリングする。
- 各サンプル行には、セッション ID、SQL ID、待機イベント、アクセスされたオブジェクト、ユーザー、モジュール、アクション、プラン・ハッシュ値、ブロッキング・セッションなどが記録される。
- インメモリ・バッファには、約 **1 時間分** のデータが保持される（古いデータはディスクにフラッシュされる）。
- ディスク・ベースの `DBA_HIST_ACTIVE_SESS_HISTORY` は、長期保存のために ASH データの **10 分の 1 のサブサンプル**（10 秒ごと）を保持する。

### AAS (Average Active Sessions)

ASH から導出される主要なメトリック。特定の時間枠内の ASH 行数を数え、秒数で割ることで算出される：

```
AAS = COUNT(ash_rows) / 期間の秒数
```

AAS が物理 CPU 数を超えている場合、データベースが過飽和状態であることを意味する。AAS を待機クラスやイベントごとに分解することで、どこに時間が費やされているかが明らかになる。

### セッション状態 (Session States)

各 ASH 行には `SESSION_STATE` がある：
- `ON CPU` — サンプリング時にセッションが CPU を消費していた。
- `WAITING` — セッションが特定のイベントを待機していた。

`WAIT_CLASS` 列と `EVENT` 列によって、待機中のセッションがさらに分類される。

---

## 主要なビュー

### V$ACTIVE_SESSION_HISTORY (インメモリ、リアルタイム)

過去約 1 時間のサンプリング・データ。各行が 1 つのサンプルを表す。

```sql
-- スキーマの確認
DESC v$active_session_history
```

主要な列:

| 列名 | 説明 |
|---|---|
| `SAMPLE_TIME` | サンプルのタイムスタンプ（1 秒単位） |
| `SESSION_ID` | サンプリングされたセッションの SID |
| `SESSION_SERIAL#` | セッションを一意に識別するためのシリアル番号 |
| `USER_ID` | ユーザー ID |
| `SQL_ID` | サンプリング時に実行されていた SQL の ID |
| `SQL_PLAN_HASH_VALUE` | 使用されていた実行プランのハッシュ値 |
| `SESSION_STATE` | `ON CPU` または `WAITING` |
| `WAIT_CLASS` | 待機イベントのカテゴリ |
| `EVENT` | 具体的な待機イベント名 |
| `CURRENT_OBJ#` | アクセスされていたオブジェクト |
| `CURRENT_FILE#` | データファイル番号 |
| `CURRENT_BLOCK#` | ブロック番号（I/O 待機の場合） |
| `BLOCKING_SESSION` | ブロッカーの SID（ロック待機の場合） |
| `MODULE` | アプリケーションのモジュール名 |
| `ACTION` | アプリケーションのアクション名 |
| `PROGRAM` | クライアントのプログラム名 |
| `MACHINE` | クライアントのマシン名 |
| `PGA_ALLOCATED` | サンプリング時の PGA メモリ割り当て量 |
| `TEMP_SPACE_ALLOCATED` | サンプリング時のテンポラリ領域割り当て量 |

### DBA_HIST_ACTIVE_SESS_HISTORY (ディスク・ベース、履歴)

`V$ACTIVE_SESSION_HISTORY` と同じ構造だが、追加の列 (`SNAP_ID`, `DBID`, `INSTANCE_NUMBER`) があり、サンプリング頻度は 10 分の 1 に低下している。

---

## リアルタイム ASH 分析

### 現在のアクティビティ・スナップショット

```sql
-- 現在（過去 5 分間）何が起きているか
SELECT event,
       COUNT(*) AS samples,
       ROUND(COUNT(*) / (5*60), 2) AS avg_active_sessions,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 1) AS pct_total
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 5/1440
  AND  session_type = 'FOREGROUND'
GROUP  BY event
ORDER  BY samples DESC;
```

### ASH による Top SQL (過去 30 分間)

```sql
SELECT ash.sql_id,
       COUNT(*)                                    AS samples,
       ROUND(COUNT(*) / (30*60), 2)               AS aas,
       SUBSTR(st.sql_text, 1, 80)                 AS sql_text
FROM   v$active_session_history ash
LEFT   JOIN v$sql st ON ash.sql_id = st.sql_id
WHERE  ash.sample_time > SYSDATE - 30/1440
  AND  ash.session_type = 'FOREGROUND'
GROUP  BY ash.sql_id, SUBSTR(st.sql_text, 1, 80)
ORDER  BY samples DESC
FETCH  FIRST 15 ROWS ONLY;
```

### 待機クラス別 Top SQL (過去 1 時間)

```sql
SELECT sql_id,
       wait_class,
       COUNT(*)                              AS samples,
       ROUND(COUNT(*) / 3600, 2)            AS aas
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/24
  AND  session_type = 'FOREGROUND'
  AND  wait_class != 'Idle'
GROUP  BY sql_id, wait_class
ORDER  BY samples DESC
FETCH  FIRST 20 ROWS ONLY;
```

### アクティブ・セッションの推移 (分単位の分解)

```sql
-- 過去 1 時間の 1 分あたりの AAS — スパイクの特定
SELECT TRUNC(sample_time, 'MI')           AS sample_minute,
       COUNT(*)                           AS samples,
       ROUND(COUNT(*) / 60, 2)           AS aas,
       SUM(CASE WHEN session_state = 'ON CPU' THEN 1 ELSE 0 END) AS cpu_samples,
       SUM(CASE WHEN session_state = 'WAITING' THEN 1 ELSE 0 END) AS wait_samples
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/24
  AND  session_type = 'FOREGROUND'
GROUP  BY TRUNC(sample_time, 'MI')
ORDER  BY sample_minute;
```

### ブロッキング・セッション分析

```sql
-- ASH からブロッキング・チェーンを特定する
SELECT sample_time,
       session_id,
       blocking_session,
       event,
       sql_id,
       seconds_in_wait
FROM   v$active_session_history
WHERE  blocking_session IS NOT NULL
  AND  sample_time > SYSDATE - 30/1440
ORDER  BY sample_time DESC, seconds_in_wait DESC;
```

### セッションごとのアクティビティ

```sql
-- 特定のセッションが過去 1 時間に何をしていたか
SELECT sample_time,
       sql_id,
       session_state,
       event,
       wait_class,
       seconds_in_wait
FROM   v$active_session_history
WHERE  session_id     = :p_sid
  AND  session_serial# = :p_serial
  AND  sample_time > SYSDATE - 1/24
ORDER  BY sample_time;
```

---

## 履歴 ASH 分析

約 1 時間より古いイベントについては、`DBA_HIST_ACTIVE_SESS_HISTORY` をクエリする。このビューの解像度は 10 分の 1 であることに注意が必要である。

### 過去のインシデント時の Top 待機イベント

```sql
-- インシデントの時間枠を分析: 例: 昨日の午前 2:00 から 3:00
SELECT event,
       wait_class,
       COUNT(*)                              AS samples,
       ROUND(COUNT(*) * 10 / 3600, 2)       AS approx_aas  -- 1/10 サンプルのため 10 倍する
FROM   dba_hist_active_sess_history
WHERE  sample_time BETWEEN
         TO_TIMESTAMP('2026-03-05 02:00:00', 'YYYY-MM-DD HH24:MI:SS')
         AND
         TO_TIMESTAMP('2026-03-05 03:00:00', 'YYYY-MM-DD HH24:MI:SS')
  AND  session_type = 'FOREGROUND'
  AND  wait_class  != 'Idle'
GROUP  BY event, wait_class
ORDER  BY samples DESC;
```

### 履歴 Top SQL

```sql
SELECT ash.sql_id,
       COUNT(*) * 10                                   AS approx_seconds,  -- 1/10 サンプルの調整
       ROUND(COUNT(*) * 10 / 3600, 2)                 AS aas,
       SUBSTR(sql.sql_text, 1, 100)                   AS sql_text
FROM   dba_hist_active_sess_history ash
JOIN   dba_hist_sqltext sql USING (sql_id, dbid)
WHERE  ash.sample_time BETWEEN
         TO_TIMESTAMP('2026-03-05 02:00:00', 'YYYY-MM-DD HH24:MI:SS')
         AND
         TO_TIMESTAMP('2026-03-05 03:00:00', 'YYYY-MM-DD HH24:MI:SS')
  AND  ash.session_type = 'FOREGROUND'
GROUP  BY ash.sql_id, SUBSTR(sql.sql_text, 1, 100)
ORDER  BY approx_seconds DESC
FETCH  FIRST 20 ROWS ONLY;
```

### オブジェクトごとのホットスポット分析

```sql
-- どのオブジェクト（表/索引）が最も多くの I/O 待機を引き起こしていたか
SELECT o.owner,
       o.object_name,
       o.object_type,
       COUNT(*)           AS wait_samples
FROM   dba_hist_active_sess_history ash
JOIN   dba_objects o ON o.object_id = ash.current_obj#
WHERE  ash.sample_time > SYSDATE - 1
  AND  ash.wait_class = 'User I/O'
  AND  ash.current_obj# > 0
GROUP  BY o.owner, o.object_name, o.object_type
ORDER  BY wait_samples DESC
FETCH  FIRST 15 ROWS ONLY;
```

### 待機クラス別タイムシリーズ分解

```sql
-- 積み上げエリアチャート用のデータ: 5 分ごとのアクティビティ分解
SELECT TRUNC(sample_time, 'HH24') +
         FLOOR(TO_NUMBER(TO_CHAR(sample_time,'MI')) / 5) * 5 / 1440 AS bucket,
       wait_class,
       COUNT(*) * 10 AS approx_seconds
FROM   dba_hist_active_sess_history
WHERE  sample_time > SYSDATE - 7
  AND  session_type = 'FOREGROUND'
  AND  wait_class  != 'Idle'
GROUP  BY TRUNC(sample_time, 'HH24') +
             FLOOR(TO_NUMBER(TO_CHAR(sample_time,'MI')) / 5) * 5 / 1440,
           wait_class
ORDER  BY bucket, wait_class;
```

---

## ASH レポートの生成

Oracle には、AWR レポートに似た形式の ASH 分析レポートを生成する組み込みスクリプトが用意されている：

```sql
-- 対話型スクリプト (時間範囲やスナップ ID の入力を求められる)
@$ORACLE_HOME/rdbms/admin/ashrpt.sql

-- プログラムによる HTML レポート
SELECT output
FROM   TABLE(
         DBMS_WORKLOAD_REPOSITORY.ASH_REPORT_HTML(
           l_dbid       => (SELECT dbid FROM v$database),
           l_inst_num   => 1,
           l_btime      => TO_DATE('2026-03-05 02:00','YYYY-MM-DD HH24:MI'),
           l_etime      => TO_DATE('2026-03-05 03:00','YYYY-MM-DD HH24:MI')
         )
       );

-- プログラムによるテキスト・レポート
SELECT output
FROM   TABLE(
         DBMS_WORKLOAD_REPOSITORY.ASH_REPORT_TEXT(
           l_dbid       => (SELECT dbid FROM v$database),
           l_inst_num   => 1,
           l_btime      => TO_DATE('2026-03-05 02:00','YYYY-MM-DD HH24:MI'),
           l_etime      => TO_DATE('2026-03-05 03:00','YYYY-MM-DD HH24:MI')
         )
       );
```

### ASH レポートの主要セクション

1. **Top User Events** — 最も多くのサンプル時間を消費したイベント。
2. **Top Background Events** — LGWR, DBWR, CKPT などのアクティビティ。
3. **Top SQL with Top Events** — 消費時間順の SQL ID と関連する待機。
4. **Top SQL with Top Row Sources** — 実行プラン内のどこで時間が消費されたか。
5. **Top Sessions** — 最も多くの時間を消費したセッション。
6. **Top Objects/Files/Latches** — オブジェクト・レベルのホットスポット。
7. **Activity Over Time** — 問題の開始と終了を特定するためのタイムシリーズ・ビュー。

---

## セッション・レベルのボトルネックの特定

### シナリオ: 「午後 9:00 から 9:15 の間にクエリが遅かったという報告」

```sql
-- ステップ 1: アクティビティのスパイクを確認する
SELECT TRUNC(sample_time, 'MI') AS minute,
       COUNT(*)                 AS samples
FROM   dba_hist_active_sess_history
WHERE  sample_time BETWEEN
         TO_TIMESTAMP('2026-03-06 09:00:00','YYYY-MM-DD HH24:MI:SS')
         AND
         TO_TIMESTAMP('2026-03-06 09:15:00','YYYY-MM-DD HH24:MI:SS')
  AND  session_type = 'FOREGROUND'
GROUP  BY TRUNC(sample_time, 'MI')
ORDER  BY minute;

-- ステップ 2: インシデント時の Top SQL を特定する
SELECT sql_id, COUNT(*) AS samples
FROM   dba_hist_active_sess_history
WHERE  sample_time BETWEEN
         TO_TIMESTAMP('2026-03-06 09:00:00','YYYY-MM-DD HH24:MI:SS')
         AND
         TO_TIMESTAMP('2026-03-06 09:15:00','YYYY-MM-DD HH24:MI:SS')
  AND  session_type = 'FOREGROUND'
GROUP  BY sql_id
ORDER  BY samples DESC
FETCH  FIRST 10 ROWS ONLY;

-- ステップ 3: 該当するユーザー/セッションを特定する
SELECT session_id, user_id, module, action, program, machine,
       COUNT(*) AS samples
FROM   dba_hist_active_sess_history
WHERE  sql_id = :suspect_sql_id
  AND  sample_time BETWEEN
         TO_TIMESTAMP('2026-03-06 09:00:00','YYYY-MM-DD HH24:MI:SS')
         AND
         TO_TIMESTAMP('2026-03-06 09:15:00','YYYY-MM-DD HH24:MI:SS')
GROUP  BY session_id, user_id, module, action, program, machine
ORDER  BY samples DESC;

-- ステップ 4: セッションが何を待機していたかを確認する
SELECT event, COUNT(*) AS samples
FROM   dba_hist_active_sess_history
WHERE  sql_id = :suspect_sql_id
  AND  sample_time BETWEEN
         TO_TIMESTAMP('2026-03-06 09:00:00','YYYY-MM-DD HH24:MI:SS')
         AND
         TO_TIMESTAMP('2026-03-06 09:15:00','YYYY-MM-DD HH24:MI:SS')
GROUP  BY event
ORDER  BY samples DESC;
```

### シナリオ: 根本的なブロッカーの特定

```sql
-- ASH からブロッキング・チェーンを再構成する
SELECT LPAD(' ', 2*(LEVEL-1)) || session_id AS session_tree,
       blocking_session,
       event,
       sql_id,
       sample_time
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 10/1440
START  WITH blocking_session IS NULL
       AND  session_state = 'WAITING'
       AND  wait_class   != 'Idle'
CONNECT BY PRIOR session_id = blocking_session
       AND PRIOR sample_time = sample_time
ORDER  SIBLINGS BY session_id;
```

---

## ASH vs AWR: 使い分け

| シナリオ | 推奨ツール |
|---|---|
| 過去 60 分以内に発生したインシデント | `V$ACTIVE_SESSION_HISTORY` |
| 8 〜 30 日前までに発生したインシデント | `DBA_HIST_ACTIVE_SESS_HISTORY` |
| 秒単位の詳細な粒度が必要な場合 | `V$ACTIVE_SESSION_HISTORY` |
| ワークロード全体の傾向を把握したい場合 | AWR レポート |
| どの SQL が遅かったか具体的に特定したい場合 | ASH (サンプルごとの SQL_ID) |
| リリース間でのパフォーマンス低下を証明したい場合 | AWR 期間比較レポート |
| 古いデータで 10 秒未満の解像度が必要な場合 | 不可能（インメモリ ASH のみ 1 秒解像度） |

---

## ベスト・プラクティス

- **モジュール/アクションでインシデントを注釈する。** アプリケーション・コードで `DBMS_APPLICATION_INFO.SET_MODULE` と `SET_ACTION` を設定すると、インシデント後の ASH 分析が非常に容易になる。
- **必要以上に ASH データを削除しない。** ディスク保存時は 10 分の 1 にサンプリングされるため、履歴分析において各行は非常に貴重である。
- **常に `session_type = 'FOREGROUND'` でフィルタリングする。** 背景プロセスの分析が特に必要でない限りこれを行う。背景プロセスの待機は、ユーザーに直接見えるパフォーマンスではなく、システムの管理タスクを反映していることが多い。
- **インメモリとディスク・ベースの ASH を比較する際は 10 倍の乗数を考慮する。** `V$ACTIVE_SESSION_HISTORY` はすべてのサンプルを保持しているが、`DBA_HIST` は 1/10 である。
- **SQL 実行プランと組み合わせる。** ASH から Top SQL_ID を特定したら、`DBMS_XPLAN.DISPLAY_AWR` (23c 以降は `DISPLAY_WORKLOAD_REPOSITORY`) を使用して履歴プランを取得する。

```sql
-- ASH で見つかった SQL の履歴実行プランを取得する
-- 注: DISPLAY_AWR は Oracle 23c+ で非推奨。新しいコードでは DISPLAY_WORKLOAD_REPOSITORY を使用
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_AWR(
    sql_id        => 'abc123xyz',
    plan_hash_value => NULL,  -- NULL = すべてのプランを表示
    db_id         => NULL,
    format        => 'TYPICAL'
  )
);
```

---

## よくある間違い

| 間違い | 影響 | 対策 |
|---|---|---|
| `session_type = 'FOREGROUND'` を忘れる | 背景プロセスの待機が結果を汚染する | 常にフィルタを追加する |
| `DBA_HIST` の解析で 10 倍を忘れる | AAS が現実の 10 分の 1 に見える | ディスク・ベースのデータではカウントを 10 倍する |
| 過去 5 分間について `DBA_HIST` をクエリする | 行がまだフラッシュされていない可能性がある | 直近のデータには `V$ACTIVE_SESSION_HISTORY` を使用する |
| すべての ASH サンプルを正確に 1 秒として扱う | サンプリングの揺らぎ（GC 停止、高負荷）が存在する | 行ごとのタイミングではなく、時間範囲の集約を使用する |
| I/O 待機における `CURRENT_OBJ#` を無視する | 原因となっているオブジェクトを見逃す | `DBA_OBJECTS` と結合してホット・セグメントを特定する |
| `BLOCKING_SESSION` を根本原因と混同する | ブロッカー自身がブロックされている可能性がある | チェーン全体を追跡する。根本原因は他を待機していないセッションである |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Performance Tuning Guide (TGDBA)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)
- [V$ACTIVE_SESSION_HISTORY — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-ACTIVE_SESSION_HISTORY.html)
- [DBA_HIST_ACTIVE_SESS_HISTORY — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_ACTIVE_SESS_HISTORY.html)
- [DBMS_WORKLOAD_REPOSITORY — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_WORKLOAD_REPOSITORY.html)
- [DBMS_XPLAN — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_XPLAN.html)
ase/19/arpls/DBMS_XPLAN.html)
