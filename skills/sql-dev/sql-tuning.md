# OracleにおけるSQLチューニング

## 概要

SQLチューニングとは、Oracleのコストベース・オプティマイザ（CBO）がどのように実行計画を生成・実行するかを把握し、クエリのパフォーマンスを向上させるプロセスである。効果的なチューニングには、実行計画の読み取り、結合手法やアクセス・パスの理解、オプティマイザ統計の管理、そしてヒント、プロファイル、またはベースラインを使用してオプティマイザに影響を与えるタイミングと方法を知ることが不可欠である。

OracleのCBOは、候補となる実行計画を評価し、CPU、I/O、およびメモリから算出される「推定コスト」が最も低いものを選択する。CBOが不適切な選択をする主な原因は、統計情報が古かったり、欠落していたり、誤解を招く内容であったりすること、またはクエリの構造が効率的なアクセス・パスを妨げていることである。

---

## 実行計画

### 実行計画の生成

主な方法は2つある。`EXPLAIN PLAN`（推定のみ）と `DBMS_XPLAN`（実測または推定）である。

```sql
-- 推定のみの実行計画（実行は行わない）
EXPLAIN PLAN FOR
SELECT e.last_name, d.department_name
FROM   employees e
JOIN   departments d ON e.department_id = d.department_id
WHERE  e.salary > 10000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

```sql
-- SQL*Plus の autotrace を使用した実際の実行統計
SET AUTOTRACE ON
SELECT e.last_name, d.department_name
FROM   employees e
JOIN   departments d ON e.department_id = d.department_id
WHERE  e.salary > 10000;
```

```sql
-- 推奨：実行後のカーソル・キャッシュから実際の計画を取得する
-- まずクエリを実行し、その SQL_ID を特定
SELECT sql_id, sql_text
FROM   v$sql
WHERE  sql_text LIKE '%last_name%employees%'
AND    sql_text NOT LIKE '%v$sql%';

-- 特定した SQL_ID を使用して、実測行数を含む計画を取得
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR('abc12345xyz', 0, 'ALLSTATS LAST')
);
```

### 実行計画の読み方

`DBMS_XPLAN` 出力の主要な列は以下の通り：

| 列名 | 意味 |
|---|---|
| `Id` | 操作のステップ番号。子ステップはインデントされる |
| `Operation` | Oracleが何を行っているか（TABLE ACCESS, INDEX RANGE SCAN など） |
| `Name` | アクセスされているオブジェクト名 |
| `Rows` (E-Rows) | 推定行数 — オプティマイザによる予測 |
| `A-Rows` | 実際の返却行数（`ALLSTATS` が必要） |
| `Bytes` | 推定データ量 |
| `Cost (%CPU)` | オプティマイザの推定コストとCPUの割合 |
| `Time` | 推定所要時間 |
| `Starts` | そのステップが実行された回数 |

E-Rows と A-Rows の間に大きな乖離がある場合、カーディナリティ推定の問題を示唆しており、これが不適切な実行計画の根本原因であることが多い。

### 一般的なアクセス・パス

- **TABLE ACCESS FULL** — セグメント内のすべてのブロックを読み取る。大量のデータを取得する場合は効率的だが、選択性の高いクエリには不向き。
- **INDEX RANGE SCAN** — 値の範囲に対してBツリー索引を検索する。選択性の高い述語に適している。
- **INDEX UNIQUE SCAN** — 一意索引に対する単一のプローブ。最も効率的な単一行アクセス。
- **INDEX FAST FULL SCAN** — マルチブロックI/Oを使用してすべての索引ブロックを読み取る。索引列のみを使用するクエリに有効。
- **INDEX SKIP SCAN** — 複合索引において、先頭列が述語に含まれていない場合に使用される。通常は、より適切な索引が必要であることを示す兆候である。

---

## 結合手法

Oracleは主に3つの結合アルゴリズムをサポートしている。オプティマイザは、カーディナリティ、利用可能な索引、および利用可能なメモリに基づいて選択を行う。

### ネステッド・ループ (NL)

駆動行セットが小さく、内部表に適切な索引がある場合に最適。

```
NESTED LOOPS
  TABLE ACCESS FULL   EMPLOYEES      (駆動表、結果が小さい)
  INDEX RANGE SCAN    DEPT_ID_IDX    (駆動行ごとに内部表を検索)
```

- メモリ消費が少ない
- 駆動セットが大きいとスケーラビリティが低下する
- 駆動行数に比例してパフォーマンスが低下する

### ハッシュ結合

結合列に利用可能な索引がなく、2つの大きなセットを結合する場合に最適。小さい方の表をメモリ上にハッシュ展開し、大きい方の表をプローブ（照合）する。

```
HASH JOIN
  TABLE ACCESS FULL   DEPARTMENTS    (ビルド側 — 小さい方)
  TABLE ACCESS FULL   EMPLOYEES      (プローブ側 — 大きい方)
```

- ハッシュテーブル用に `PGA` メモリが必要
- ビルド側が PGA に収まらない場合、TEMP へ退避（スローダウン）する
- 等価結合にのみ機能する

### ソート・マージ結合

両方の入力を結合キーでソートし、マージする。入力がすでにソートされている場合や、ハッシュ結合が利用できない場合に使用される。

- ハッシュ結合よりもメモリ要件が高い
- データが索引によって事前ソートされている場合、ハッシュ結合より速くなることがある

---

## オプティマイザ・ヒント

ヒントは、SQLコメント内に埋め込まれた指示で、オプティマイザに特定の実行計画要素を使用するよう指示する。ヒントは控えめに使用し、まずは統計情報や索引の修正を優先すべきである。

### 構文

```sql
SELECT /*+ HINT_NAME(parameter) */ col1 FROM table1;
```

### よく使われるヒント

```sql
-- 索引の使用を強制
SELECT /*+ INDEX(e EMP_DEPT_IDX) */ *
FROM   employees e
WHERE  department_id = 30;

-- 全表スキャンを強制（索引をバイパス）
SELECT /*+ FULL(e) */ *
FROM   employees e
WHERE  status = 'A';

-- 結合手法の制御
SELECT /*+ USE_NL(e d) */ e.last_name, d.department_name
FROM   employees e
JOIN   departments d ON e.department_id = d.department_id;

SELECT /*+ USE_HASH(e d) */ e.last_name, d.department_name
FROM   employees e
JOIN   departments d ON e.department_id = d.department_id;

-- 結合順序の制御 (e が最初/駆動表になる)
SELECT /*+ LEADING(e d) USE_NL(d) */ e.last_name, d.department_name
FROM   employees e
JOIN   departments d ON e.department_id = d.department_id;

-- パラレル実行
SELECT /*+ PARALLEL(e, 4) */ COUNT(*) FROM employees e;

-- パラレル無効化 (OLTPで有効)
SELECT /*+ NO_PARALLEL(e) */ * FROM employees e;

-- カーディナリティ・ヒント (統計が正しくない場合)
SELECT /*+ CARDINALITY(e 100) */ *
FROM   employees e
WHERE  complex_function(salary) > 5000;
```

### ヒントを使用すべきでないケース

- ヒントは壊れやすい — 表名や索引名が変わると機能しなくなる
- データの分布の変化にオプティマイザが適応する能力を妨げる
- 根本原因（より良い統計情報の収集、索引の追加、またはクエリの書き換え）の修正を優先すべき
- 本番環境の安定化には、代わりに SQL プロファイルやベースラインを使用すること

---

## 統計情報の収集と解釈

OracleのCBOはオブジェクト統計に依存している。古かったり欠落していたりする統計情報は、不適切な実行計画の最大の原因である。

### 表および索引の統計情報の収集

```sql
-- 単一の表に対して統計情報を収集（すべての列にヒストグラムを作成）
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  ownname          => 'HR',
  tabname          => 'EMPLOYEES',
  estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
  method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
  cascade          => TRUE   -- 索引の統計情報も収集
);

-- スキーマ全体の統計情報を収集
EXEC DBMS_STATS.GATHER_SCHEMA_STATS(
  ownname          => 'HR',
  estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
  method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
  cascade          => TRUE
);
```

### 統計情報の確認

```sql
-- 表レベルの統計
SELECT table_name, num_rows, blocks, avg_row_len, last_analyzed
FROM   dba_tables
WHERE  owner = 'HR';

-- 列レベルの統計とヒストグラム
SELECT column_name, num_distinct, num_nulls, histogram, num_buckets,
       low_value, high_value, last_analyzed
FROM   dba_tab_col_statistics
WHERE  owner = 'HR'
AND    table_name = 'EMPLOYEES';

-- 失効した統計情報のチェック (前回収集から10%以上の変更があったもの)
SELECT owner, table_name, stale_stats
FROM   dba_tab_statistics
WHERE  owner = 'HR'
AND    stale_stats = 'YES';
```

### ヒストグラム

ヒストグラムは列のデータの偏りを記述するもので、データが不均一な列にとって重要である。

```sql
-- 頻度ヒストグラム (カーディナリティが低い列用)
-- 一意の値が 254 以下の列に対し、SIZE AUTO を指定すると自動的に作成される

-- ヒストグラムのバケットを確認
SELECT endpoint_number, endpoint_value
FROM   dba_histograms
WHERE  owner = 'HR'
AND    table_name = 'EMPLOYEES'
AND    column_name = 'DEPARTMENT_ID'
ORDER BY endpoint_number;
```

### 拡張統計 (複数列の相関)

列間に相関がある場合（例：`city` と `state`）、オプティマイザがカーディナリティを過小評価することがある。拡張統計はこの問題を修正する。

```sql
-- 列グループに対して拡張統計を作成
SELECT DBMS_STATS.CREATE_EXTENDED_STATS(
  ownname => 'HR',
  tabname => 'EMPLOYEES',
  extension => '(DEPARTMENT_ID, JOB_ID)'
) FROM dual;

-- その後、統計を再収集して値を反映
EXEC DBMS_STATS.GATHER_TABLE_STATS('HR', 'EMPLOYEES',
  method_opt => 'FOR ALL COLUMNS SIZE AUTO');
```

---

## 索引の使用と戦略

### 不足している索引の特定

```sql
-- カーソル・キャッシュから、大きな表に対する全表スキャンを検索
SELECT s.sql_id, s.executions, p.object_name, p.operation, p.options,
       s.disk_reads / NULLIF(s.executions, 0) AS reads_per_exec
FROM   v$sql s
JOIN   v$sql_plan p ON s.sql_id = p.sql_id AND s.child_number = p.child_number
WHERE  p.operation = 'TABLE ACCESS'
AND    p.options   = 'FULL'
AND    p.object_name NOT IN ('DUAL')
ORDER BY reads_per_exec DESC NULLS LAST
FETCH FIRST 20 ROWS ONLY;
```

### 索引の監視

```sql
-- Oracle 12.2以降 — 索引の使用状況統計を確認
SELECT index_name, table_name, monitoring, used, start_monitoring, end_monitoring
FROM   v$object_usage
WHERE  table_name = 'EMPLOYEES';

-- 索引の監視を開始
ALTER INDEX hr.emp_name_idx MONITORING USAGE;
```

### 不可視索引

検証が終わるまで本番の計画に影響を与えずに、新しい索引をテストできる。

```sql
-- 不可視索引として作成（プランにはまだ影響しない）
CREATE INDEX emp_salary_idx ON employees(salary) INVISIBLE;

-- 自分のセッションのみでテスト
ALTER SESSION SET optimizer_use_invisible_indexes = TRUE;
EXPLAIN PLAN FOR SELECT * FROM employees WHERE salary BETWEEN 5000 AND 8000;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 問題なければ可視化
ALTER INDEX emp_salary_idx VISIBLE;
```

---

## SQL プロファイルおよび SQL 計画ベースライン

### SQL プロファイル

SQL プロファイルは、特定のクエリに対して追加の統計情報や補正係数を保持し、計画を固定することなくオプティマイザの推定精度を向上させる。

```sql
-- SQL チューニング・アドバイザを使用してプロファイルを生成
DECLARE
  l_task_name VARCHAR2(30);
  l_sql_id    VARCHAR2(13) := 'abc12345xyz78';
BEGIN
  l_task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(
    sql_id      => l_sql_id,
    scope       => DBMS_SQLTUNE.SCOPE_COMPREHENSIVE,
    time_limit  => 60,
    task_name   => 'tune_task_1'
  );
  DBMS_SQLTUNE.EXECUTE_TUNING_TASK(task_name => l_task_name);
END;
/

-- 推奨事項を確認
SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('tune_task_1') FROM dual;

-- 推奨された場合、SQL プロファイルを適用
EXEC DBMS_SQLTUNE.ACCEPT_SQL_PROFILE(
  task_name    => 'tune_task_1',
  replace      => TRUE
);
```

### SQL 計画ベースライン (SPM)

SQL 計画管理 (SPM) は、既知の良好な計画を把握し、実行計画の「退行（劣化）」を防ぐ。

```sql
-- 既存の SQL_ID に基づき、カーソル・キャッシュからベースラインを取得
DECLARE
  l_count PLS_INTEGER;
BEGIN
  l_count := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
    sql_id => 'abc12345xyz78'
  );
  DBMS_OUTPUT.PUT_LINE('Baselines loaded: ' || l_count);
END;
/

-- 既存のベースラインを確認
SELECT sql_handle, plan_name, enabled, accepted, fixed, origin
FROM   dba_sql_plan_baselines
ORDER BY created DESC;

-- 未承認の計画の進化（テストを行い、より良い計画を昇格させる）
DECLARE
  l_report CLOB;
BEGIN
  l_report := DBMS_SPM.EVOLVE_SQL_PLAN_BASELINE(
    sql_handle => 'SQL_abc123',
    time_limit => 30,
    verify     => 'YES',
    commit     => 'YES'
  );
  DBMS_OUTPUT.PUT_LINE(l_report);
END;
/
```

---

## ベストプラクティス

- **常に実測行数 (Actual) と推定行数 (Estimated) を比較する。** 10倍以上の乖離がある場合、ほぼ確実に不適切な計画が生成される。
- **データ変化の激しい表では定期的に統計情報を収集する。** 大量にDMLが発生する表については、夜間ジョブ等で収集を行う。
- **`AUTO_SAMPLE_SIZE` を使用する。** Oracleのインクリメンタル統計アルゴリズムにより、低コストで100%に近い精度を得られる。
- **WHERE 句内の索引列に対して関数呼び出しを行わない。** 必要に応じて関数ベース索引を使用する。
- **クエリをパラメータ化する。** リテラル値を使用すると、実行計画の共有が妨げられ、共有プールがハード解析によって圧迫される。
- **本番環境に適用する前に、計画変更を検証する。** 不可視索引や SPM ベースラインを使用して、リスクなしでテストを行う。
- **`SELECT *` を避ける。** 必要な列のみを取得するようにする。不要な列を取得すると、索引のみを使用したアクセス・パス（カバリング・インデックス）が妨げられる。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| 索引列に関数を使用: `WHERE UPPER(name) = 'SMITH'` | 索引が無効化される | 関数ベース索引を作成する: `CREATE INDEX ... ON t(UPPER(name))` |
| 暗黙の型変換: `WHERE emp_id = '100'` (emp_id が NUMBER の場合) | 索引の使用が妨げられる | データ型を明示的に一致させる |
| 大量ロード後の失効した統計情報 | CBO が誤ったカーディナリティを使用してしまう | 大量DMLの後は `GATHER_TABLE_STATS` を実行する |
| アプリケーション・コード内でヒントを多用する | 壊れやすく、オブジェクト名の変更に対応できない | 代わりに SQL プロファイル/ベースラインを使用する |
| `ORDER BY` の前に `ROWNUM` を使用 | ソートされていない行を返してからフィルタリングしてしまう | `FETCH FIRST N ROWS ONLY` またはサブクエリを使用する |
| 過剰なインデックス作成 | DMLが遅くなり、領域を浪費する | 索引の使用状況を監視し、未使用の索引を削除する |
| `TEMP` 表領域への退避を無視する | ハッシュ/ソートの退避はパフォーマンスを著しく低下させる | PGAを増やすか、中間セットを減らすようにクエリを書き換える |

---

## Oracle 固有の考慮事項

- `OPTIMIZER_ADAPTIVE_PLANS` パラメータ (12c以降、デフォルト ON) により、実測行数に基づいて実行中に結合手法を切り替えることができる。これは有効だが、予期せぬ計画変更の原因にもなる。
- `OPTIMIZER_STATISTICS_ADVISOR` (12.2以降) がメンテナンス中に自動実行され、統計情報収集に関する改善を推奨する。
- Exadata では、実行計画内の `CELL_OFFLOAD_PROCESSING` は Smart Scan が有効であることを示す。Exadata ではストレージ・セル側で処理が行われるため、全表スキャンが非常に高速になる場合がある。
- `/*+ GATHER_PLAN_STATISTICS */` ヒントを使用すると、autotrace を使用せずにクエリに `A-Rows` と `A-Time` 情報を追加でき、アプリケーション・コードのテストに非常に有用である。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ガイドの基本的な指針は、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッド環境では、デフォルトや非推奨がリリース更新によって異なる場合があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Tuning Guide (TGSQL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [Oracle Database 19c SQL Language Reference (SQLRF)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [DBMS_XPLAN — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_XPLAN.html)
- [DBMS_SQLTUNE — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQLTUNE.html)
- [DBMS_SPM — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SPM.html)
