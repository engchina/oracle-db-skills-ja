# OracleにおけるSQLパターン

## 概要

Oracle SQLには、基本的な SELECT/INSERT/UPDATE/DELETE を超えた、高度な構文が豊富に用意されている。これらのパターンを習得することで、複雑な分析、階層、および変換ロジックをすべてSQL内で表現できるようになり、同等のPL/SQL手続き型コードと比較して、パフォーマンスと可読性の両面で劇的なメリットが得られることが多い。

本ガイドでは、実務上最も影響力の大きい高度なSQLパターン（分析（ウィンドウ）関数、共通テーブル式（CTE）、階層クエリ、PIVOT/UNPIVOT、MERGE文、およびMODEL句）について解説する。

---

## 分析（ウィンドウ）関数

分析関数は、GROUP BY のように結果セットを1行にまとめることなく、現在の行に関連する行の「ウィンドウ」全体で値を計算する。分析関数は WHERE、GROUP BY、HAVING 句の後に評価されるが、最終的な ORDER BY の前に評価される。

**構文テンプレート:**

```sql
関数名([引数])
  OVER (
    [PARTITION BY 分割列]
    [ORDER BY ソート列]
    [ROWS | RANGE BETWEEN フレーム開始 AND フレーム終了]
  )
```

### ROW_NUMBER, RANK, および DENSE_RANK

```sql
-- ROW_NUMBER: 重複に関係なく、一意で連続した番号を付与
-- RANK: 同じ値の行には同じ番号を付与。次の番号はスキップされる (1, 2, 2, 4)
-- DENSE_RANK: 同じ値の行には同じ番号を付与。番号に欠番は生じない (1, 2, 2, 3)
SELECT
  employee_id,
  last_name,
  department_id,
  salary,
  ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS row_num,
  RANK()       OVER (PARTITION BY department_id ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dense_rnk
FROM employees
ORDER BY department_id, salary DESC;
```

**よくあるパターン: グループごとの上位N件**

```sql
-- 各部門の給与上位3名を取得
SELECT *
FROM (
  SELECT
    employee_id,
    last_name,
    department_id,
    salary,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rnk
  FROM employees
)
WHERE rnk <= 3
ORDER BY department_id, rnk;
```

### LAG と LEAD

`LAG` は前の行の値にアクセスし、`LEAD` は後の行の値にアクセスする。どちらも自己結合を回避できる。

```sql
-- 各従業員の給与を、同じ部門の直近の採用者と比較
SELECT
  employee_id,
  last_name,
  hire_date,
  salary,
  LAG(salary, 1, 0) OVER (PARTITION BY department_id ORDER BY hire_date) AS prev_hire_salary,
  salary - LAG(salary, 1, 0) OVER (PARTITION BY department_id ORDER BY hire_date) AS salary_diff,
  LEAD(hire_date, 1) OVER (PARTITION BY department_id ORDER BY hire_date) AS next_hire_date
FROM employees
ORDER BY department_id, hire_date;
```

### 分析関数としての SUM, AVG, COUNT (累計)

```sql
-- 採用日順の、部門ごとの給与累計
SELECT
  employee_id,
  last_name,
  hire_date,
  salary,
  SUM(salary) OVER (
    PARTITION BY department_id
    ORDER BY hire_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total,
  AVG(salary) OVER (
    PARTITION BY department_id
    ORDER BY hire_date
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW  -- 3行移動平均
  ) AS moving_avg_3
FROM employees
ORDER BY department_id, hire_date;
```

### NTILE, PERCENT_RANK, CUME_DIST

```sql
-- 四分位数によるグルーピングとパーセンタイル分布
SELECT
  employee_id,
  last_name,
  salary,
  NTILE(4)       OVER (ORDER BY salary) AS salary_quartile,
  PERCENT_RANK() OVER (ORDER BY salary) AS pct_rank,       -- 0 から 1 まで
  CUME_DIST()    OVER (ORDER BY salary) AS cumulative_dist  -- 0 から 1 まで
FROM employees
ORDER BY salary;
```

### FIRST_VALUE と LAST_VALUE

```sql
-- 各従業員の給与とともに、部門内の最高給与額を表示
SELECT
  employee_id,
  last_name,
  department_id,
  salary,
  FIRST_VALUE(salary) OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_max_salary,
  LAST_VALUE(salary)  OVER (
    PARTITION BY department_id ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- LAST_VALUE を正しく機能させるために必要
  ) AS dept_min_salary
FROM employees
ORDER BY department_id, salary DESC;
```

注意: `LAST_VALUE` を使用する場合、デフォルトのウィンドウ・フレーム（先頭から現在行まで）を上書きするために、`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` を追加する必要がある。そうしないと、現在行までの範囲内でのみ評価されてしまう。

---

## 共通テーブル式 (WITH 句)

CTE (Common Table Expressions) は、メインクエリ内で複数回参照できる名前付きのサブクエリを定義する。可読性が向上し、`MATERIALIZED` ヒントを併用することで、サブクエリを一度だけ計算するように制御してパフォーマンスを向上させることも可能。

### 基本的な CTE

```sql
-- 一度計算して二度参照される名前付きサブクエリ
WITH dept_payroll AS (
  SELECT
    department_id,
    SUM(salary)  AS total_salary,
    COUNT(*)     AS headcount,
    AVG(salary)  AS avg_salary
  FROM   employees
  GROUP BY department_id
)
SELECT
  d.department_name,
  dp.total_salary,
  dp.headcount,
  dp.avg_salary,
  dp.total_salary / SUM(dp.total_salary) OVER () AS pct_of_company_payroll
FROM   dept_payroll  dp
JOIN   departments   d ON dp.department_id = d.department_id
ORDER BY dp.total_salary DESC;
```

### 重ね書き CTE (複数の WITH 句)

```sql
WITH
-- ステップ 1: 部門統計を計算
dept_stats AS (
  SELECT department_id, AVG(salary) AS avg_sal, MAX(salary) AS max_sal
  FROM   employees
  GROUP BY department_id
),
-- ステップ 2: 部門平均を上回る給与の従業員を特定
above_avg AS (
  SELECT e.employee_id, e.last_name, e.department_id, e.salary,
         ds.avg_sal AS dept_avg
  FROM   employees e
  JOIN   dept_stats ds ON e.department_id = ds.department_id
  WHERE  e.salary > ds.avg_sal
),
-- ステップ 3: 部門テーブルと結合して最終出力
final_result AS (
  SELECT aa.last_name, d.department_name, aa.salary, aa.dept_avg,
         ROUND((aa.salary - aa.dept_avg) / aa.dept_avg * 100, 1) AS pct_above_avg
  FROM   above_avg aa
  JOIN   departments d ON aa.department_id = d.department_id
)
SELECT * FROM final_result ORDER BY pct_above_avg DESC;
```

### 再帰 CTE

Oracleは、`CONNECT BY` に加えて 11gR2 から再帰 CTE をサポートしており、`SEARCH` および `CYCLE` 句も使用可能。

```sql
-- 再帰 CTE: 従業員の組織図
WITH org_chart (employee_id, manager_id, last_name, lvl, path) AS (
  -- アンカー: 最上位の従業員 (マネージャーがいない)
  SELECT employee_id, manager_id, last_name, 1,
         CAST(last_name AS VARCHAR2(4000)) AS path
  FROM   employees
  WHERE  manager_id IS NULL

  UNION ALL

  -- 再帰ステップ: すでに結果セットに含まれている者に報告する従業員
  SELECT e.employee_id, e.manager_id, e.last_name, oc.lvl + 1,
         oc.path || ' > ' || e.last_name
  FROM   employees e
  JOIN   org_chart  oc ON e.manager_id = oc.employee_id
)
SEARCH DEPTH FIRST BY last_name SET order_seq
CYCLE employee_id SET is_cycle TO '1' DEFAULT '0'
SELECT lvl, LPAD(' ', (lvl-1)*4) || last_name AS org_chart, path
FROM   org_chart
WHERE  is_cycle = '0'
ORDER BY order_seq;
```

### マテリアライズ・ヒント

```sql
-- CTE を一度だけ計算し、結果を一時表として保持するよう Oracle に強制する
-- オプティマイザがメインクエリ内に複数回インライン展開するのを防ぐ
WITH expensive_subquery AS (
  SELECT /*+ MATERIALIZE */ department_id, complex_calculation(salary) AS result
  FROM   employees
)
SELECT * FROM expensive_subquery e1
JOIN   expensive_subquery e2 ON e1.department_id != e2.department_id;
```

---

## 階層クエリ (CONNECT BY)

`CONNECT BY` は、標準 SQL の再帰 CTE よりも前から存在する Oracle 固有の階層クエリ構文である。簡潔であり、Oracle 固有の擬似列や関数による豊富なサポートが特徴。

### 基本的な階層走査

```sql
-- CONNECT BY を使用した従業員の組織図
SELECT
  LEVEL,
  LPAD(' ', (LEVEL-1)*4) || last_name AS org_chart,
  employee_id,
  manager_id,
  SYS_CONNECT_BY_PATH(last_name, ' / ') AS full_path
FROM   employees
START WITH manager_id IS NULL        -- ルートノードの指定
CONNECT BY PRIOR employee_id = manager_id  -- 親子の関連付け
ORDER SIBLINGS BY last_name;         -- 同じ階層内でのソート
```

### 主要な CONNECT BY 擬似列と関数

| 機能 | 説明 |
|---|---|
| `LEVEL` | 階層の深さ (1 = ルート) |
| `CONNECT_BY_ROOT expr` | 現在のブランチのルートにおける `expr` の値 |
| `CONNECT_BY_ISLEAF` | 現在の行に子がいなければ 1、いれば 0 |
| `CONNECT_BY_ISCYCLE` | 循環が検出された場合は 1 (`NOCYCLE` キーワードが必要) |
| `SYS_CONNECT_BY_PATH(col, delim)` | ルートから現在行までのパス |

```sql
-- マネージャー 101 の部下（直接および間接）をすべて検索
-- すべての行にトップ・マネージャーの名前を表示する
SELECT
  LEVEL,
  employee_id,
  last_name,
  CONNECT_BY_ROOT last_name  AS root_manager,
  CONNECT_BY_ISLEAF          AS is_leaf,
  SYS_CONNECT_BY_PATH(last_name, ' > ') AS path
FROM   employees
START WITH employee_id = 101
CONNECT BY PRIOR employee_id = manager_id;
```

### 循環の検出と処理

```sql
-- データ品質チェック: 階層内の循環参照を特定する
SELECT employee_id, last_name, manager_id
FROM   employees
WHERE  CONNECT_BY_ISCYCLE = 1
START WITH manager_id IS NULL
CONNECT BY NOCYCLE PRIOR employee_id = manager_id;
```

### CONNECT BY LEVEL による行生成

```sql
-- カレンダー表用の日付データを生成 (ソーステーブル不要)
SELECT TRUNC(SYSDATE, 'YEAR') + LEVEL - 1 AS calendar_date
FROM   dual
CONNECT BY LEVEL <= 365;

-- 数値シーケンスの生成
SELECT LEVEL AS n FROM dual CONNECT BY LEVEL <= 10;
```

---

## PIVOT と UNPIVOT

### PIVOT: 行から列へ

```sql
-- 部門ごと、職種ごとの従業員数を集計
-- PIVOT なしの場合: 列ごとに CASE 式が必要
-- PIVOT ありの場合: 宣言的で簡潔

SELECT *
FROM (
  SELECT department_id, job_id, salary
  FROM   employees
)
PIVOT (
  SUM(salary)   AS total_sal,
  COUNT(*)      AS headcount
  FOR job_id IN (
    'IT_PROG'  AS it_prog,
    'SA_REP'   AS sales_rep,
    'ST_CLERK' AS stock_clerk,
    'MK_MAN'   AS mkt_mgr
  )
)
ORDER BY department_id;

-- 結果の列: DEPARTMENT_ID, IT_PROG_TOTAL_SAL, IT_PROG_HEADCOUNT, 
--           SALES_REP_TOTAL_SAL, SALES_REP_HEADCOUNT など
```

### 動的 PIVOT (コンパイル時に列リストが不明な場合)

列リストは解析時に確定している必要があるため、動的な列リストには動的SQLが必要。

```plsql
DECLARE
  v_cols  CLOB;
  v_sql   CLOB;
BEGIN
  -- 一意の値から IN リストを構築
  SELECT LISTAGG('''' || job_id || ''' AS ' || LOWER(REPLACE(job_id,'-','_')), ', ')
         WITHIN GROUP (ORDER BY job_id)
  INTO   v_cols
  FROM  (SELECT DISTINCT job_id FROM employees WHERE department_id IN (50,60,80));

  v_sql := 'SELECT * FROM ('
        || '  SELECT department_id, job_id, salary FROM employees'
        || ') PIVOT (SUM(salary) FOR job_id IN (' || v_cols || '))'
        || ' ORDER BY department_id';

  EXECUTE IMMEDIATE v_sql;  -- 実際には Ref Cursor を開いて返すことが多い
END;
/
```

### UNPIVOT: 列から行へ

```sql
-- 幅の広いテーブルをキー・値の構造に正規化する
-- ソース: quarterly_sales(product_id, q1_sales, q2_sales, q3_sales, q4_sales)
SELECT product_id, quarter, sales_amount
FROM   quarterly_sales
UNPIVOT (
  sales_amount          -- 値を格納する列の名前
  FOR quarter           -- キーを格納する列の名前
  IN (
    q1_sales AS 'Q1',
    q2_sales AS 'Q2',
    q3_sales AS 'Q3',
    q4_sales AS 'Q4'
  )
)
ORDER BY product_id, quarter;
```

`INCLUDE NULLS` / `EXCLUDE NULLS` (デフォルト): 値が NULL の行を含めるかどうかを制御。

---

## MERGE 文 (Upsert)

`MERGE` は、INSERT と UPDATE（およびオプションで DELETE）を単一のアトミックなステートメントに統合する。ターゲット・テーブルを一度しか読み取らないため、増分ロードや同期処理において効率的。

### 基本的な MERGE (Upsert)

```sql
-- ステージング・テーブルから従業員テーブルへ同期
MERGE INTO employees tgt
USING (
  SELECT employee_id, first_name, last_name, salary, department_id, hire_date
  FROM   employees_staging
) src
ON (tgt.employee_id = src.employee_id)
WHEN MATCHED THEN
  UPDATE SET
    tgt.first_name     = src.first_name,
    tgt.last_name      = src.last_name,
    tgt.salary         = src.salary,
    tgt.department_id  = src.department_id
  WHERE tgt.salary != src.salary   -- 変更があった場合のみ更新するオプションフィルタ
WHEN NOT MATCHED THEN
  INSERT (employee_id, first_name, last_name, salary, department_id, hire_date)
  VALUES (src.employee_id, src.first_name, src.last_name, 
          src.salary, src.department_id, src.hire_date);
```

### DELETE を伴う MERGE

```sql
-- ソース側で非アクティブとマークされている一致行を削除
MERGE INTO employees tgt
USING employees_staging src
ON (tgt.employee_id = src.employee_id)
WHEN MATCHED THEN
  UPDATE SET tgt.salary = src.salary
  DELETE WHERE src.status = 'TERMINATED'  -- DELETE は更新されたばかりの行に適用される
WHEN NOT MATCHED THEN
  INSERT (employee_id, last_name, salary, hire_date)
  VALUES (src.employee_id, src.last_name, src.salary, src.hire_date);
```

### 条件付き MERGE (存在しない場合のみ INSERT)

```sql
-- 行が存在しない場合のみ挿入し、更新は一切行わない
MERGE INTO order_lookup tgt
USING (SELECT 101 AS order_id, 'PENDING' AS status FROM dual) src
ON (tgt.order_id = src.order_id)
WHEN NOT MATCHED THEN
  INSERT (order_id, status, created_at)
  VALUES (src.order_id, src.status, SYSDATE);
```

### MERGE ベストプラクティス

- 結果が非決定論的になるのを避けるため、`ON` 句には常に一意キーまたは主キーを含める。
- `WHEN MATCHED THEN UPDATE` に `WHERE` 句を追加して、変更のない行の更新をスキップする。これにより無駄な Redo 生成を回避できる。
- `MERGE` は、ターゲット・テーブルに対して `INSERT` と `UPDATE` の両方のトリガーを起動する可能性があることに注意する。
- ソース側に重複した `ON` 条件の一致がある場合、Oracle は `ORA-30926: ソース表の安定した行セットを取得できません` を発生させる。MERGE 前にソースの重複を除去しておくこと。

---

## MODEL 句

`MODEL` 句は、リレーショナルな結果セットに対して表計算ソフトのような計算を可能にする。配列形式の記法を使用して、他のセルの値に基づいてセルを参照したり計算したりできる。財務モデリング、予算配分、および標準的な SQL では困難な逐次計算に強力な威力を発揮する。

### 基本的な MODEL 構文

```sql
-- 過去の成長率に基づいて将来の売上を予測
SELECT year, region, sales_amount
FROM (
  SELECT 2021 AS year, 'EAST' AS region, 150000 AS sales_amount FROM dual UNION ALL
  SELECT 2022, 'EAST', 165000 FROM dual UNION ALL
  SELECT 2023, 'EAST', 180000 FROM dual UNION ALL
  SELECT 2021, 'WEST', 200000 FROM dual UNION ALL
  SELECT 2022, 'WEST', 220000 FROM dual UNION ALL
  SELECT 2023, 'WEST', 242000 FROM dual
)
MODEL
  PARTITION BY (region)               -- 地域ごとに計算グリッドを分ける
  DIMENSION BY (year)                 -- グリッド内の行キー
  MEASURES     (sales_amount)         -- 計算または参照する値
  RULES (
    -- 10% の成長率に基づき、2024年と2025年を予測
    sales_amount[2024] = sales_amount[2023] * 1.10,
    sales_amount[2025] = sales_amount[2024] * 1.10
    -- 標準の MODEL では PARTITION BY を越えた参照はできない
    -- その場合は REFERENCE MODEL を使用する
  )
ORDER BY region, year;
```

### 反復を伴う MODEL (ITERATE)

```sql
-- 複利計算: 収束するまで、または N 回反復
SELECT period, balance
FROM (SELECT 0 AS period, 10000 AS balance FROM dual)
MODEL
  DIMENSION BY (period)
  MEASURES (balance)
  RULES ITERATE (10) (   -- 10 回反復実行
    balance[ITERATION_NUMBER + 1] = ROUND(balance[ITERATION_NUMBER] * 1.05, 2)
  )
ORDER BY period;
```

### MODEL 参照ルール

| 記法 | 意味 |
|---|---|
| `col[2024]` | 次元 = 2024 の特定のセル |
| `col[CV()]` | 現在行のセル (CV = Current Value) |
| `col[CV()-1]` | 前の行のセル |
| `col[ANY]` | すべてのセルにマッチするワイルドカード |
| `col[year BETWEEN 2020 AND 2025]` | セルの範囲指定 |

```sql
-- MODEL を使用した累計計算 (年ごとの累計売上)
SELECT year, sales, cumulative_sales
FROM   annual_sales
MODEL
  DIMENSION BY (year)
  MEASURES (sales, 0 AS cumulative_sales)
  RULES (
    cumulative_sales[year > 2019] ORDER BY year = 
      cumulative_sales[CV()-1] + sales[CV()]
  )
ORDER BY year;
```

**MODEL と分析関数の使い分け:** 標準的な累計、ランキング、移動平均には、分析関数の方がほぼ常に高速で明快である。真のセル参照セマンティクス、行をまたぐ代入、またはウィンドウ関数では表現できない反復計算が必要な場合にのみ、`MODEL` を使用する。

---

## ベストプラクティス

- **自己結合の代わりに分析関数を使用する。** 行比較が必要なほとんどのケースで、`LAG`/`LEAD` や `FIRST_VALUE`/`LAST_VALUE` を使うことで、コストの高い自己結合を回避できる。
- **CTE は可読性のために使用するが、必ずしもパフォーマンスが向上するとは限らない。** オプティマイザは CTE をインライン展開することができる。挙動を明示的に制御するには `/*+ MATERIALIZE */` または `/*+ INLINE */` を検討する。
- **シンプルな階層には CONNECT BY、ポータビリティを重視するなら再帰 CTE。** Oracle のみの環境では `CONNECT BY` の方が簡潔。再帰 CTE は SQL 標準準拠である。
- **階層データでは必ず循環を処理する。** `CONNECT BY NOCYCLE` または再帰 CTE の `CYCLE` 句を必ず使用する。
- **MERGE が常に個別の INSERT/UPDATE よりも速いわけではない。** 超大量のデータロードでは、ダイレクト・パス INSERT の後に特定の行のみ UPDATE する方が速い場合がある。両方を測定すること。
- **PIVOT の列リストは解析時に既知である必要がある。** 動的な列リストが必要な場合は、動的SQLを使用する。
- **UNION ALL の連鎖の代わりに UNPIVOT。** 正規化において、複数の `UNION ALL` ブランチを並べるよりも `UNPIVOT` の方が可読性が高く、多くの場合高速である。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| フレーム句を指定せずに `LAST_VALUE` を使用 | デフォルトのフレームが現在行で終わるため、誤った値が返る | 必ず `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` を追加する |
| MERGE ソースに重複した `ON` キーの行が存在する | `ORA-30926: ソース表の安定した行セットを取得できません` | `SELECT DISTINCT` やパーティション化された行フィルタリング等でソースの重複を排除する |
| `CONNECT BY ORDER BY` で `SIBLINGS` を忘れる | `SIBLINGS` なしの `ORDER BY` は階層構造を壊してしまう | 各階層内でソートするには `ORDER SIBLINGS BY` を使用する |
| `START WITH` ではなく `WHERE` 句で `LEVEL` を絞り込む | `WHERE` 句で絞り込んでも、全ツリーを走査してしまう | ルートの条件は `START WITH` に記述する |
| MODEL 句の過剰な使用 | 複雑になり、遅く、保守が困難になる | ほとんどの問題には分析関数や PL/SQL を使用し、真にセル参照が必要な場合にのみ MODEL を使用する |
| 大規模な中間セットに対する分析関数を伴う CTE | 大規模な CTE のマテリアライズはコストが高くなる可能性がある | 実行計画を確認し、条件のプッシュダウンや `INLINE` ヒントの使用を検討する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ガイドの基本的な指針は、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッド環境では、デフォルトや非推奨がリリース更新によって異なる場合があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference (SQLRF)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [Oracle Database 19c SQL Tuning Guide (TGSQL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [Oracle Database 19c Data Warehousing Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/dwhsg/)
