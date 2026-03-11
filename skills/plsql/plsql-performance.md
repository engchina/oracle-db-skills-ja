# PL/SQL パフォーマンス (Performance)

## 概要

PL/SQL のパフォーマンス最適化の核となるのは、PL/SQL エンジンと SQL エンジンの間のコンテキスト・スイッチを最小限に抑えること、行単位ではなく集合単位でデータを処理すること、および適切な場所で結果をキャッシュすることです。このガイドでは、本番システムで最大のパフォーマンス向上をもたらす主要な手法について説明します。

---

## コンテキスト・スイッチのコスト

PL/SQL が SQL 文を SQL エンジンに送信するたびに（またはその逆）、**コンテキスト・スイッチ**が発生します。これらのスイッチにはオーバーヘッドが伴います。エンジン間でのハンドシェイク、データの転送、処理の再開が必要です。100,000 行を 1 行ずつループで処理する場合、100,000 回のコンテキスト・スイッチが発生し、これが PL/SQL コードが遅くなる主な原因となります。

```
PL/SQL エンジン                 SQL エンジン
      |                               |
      |  -- SELECT (1行) -->          |
      |  <-- 結果 -------------       |
      |  -- SELECT (1行) -->          |  x N行 = N回のコンテキスト・スイッチ
      |  <-- 結果 -------------       |
      ...
```

目標は、処理をまとめて実行することです。各行に対して 1 つずつ SQL 文を送信するのではなく、1 つの SQL 文で多くの行を処理するようにします。

---

## 行単位処理 vs 集合ベース処理

### 行単位処理 (Slow by Slow)

```sql
-- アンチパターン: カーソル・ループ内で個別に DML を実行する
PROCEDURE apply_10pct_raise IS
  CURSOR c_employees IS
    SELECT employee_id FROM employees WHERE department_id = 10;
BEGIN
  FOR rec IN c_employees LOOP
    -- 各反復ごとに SQL エンジンへのコンテキスト・スイッチが 1 回発生
    UPDATE employees
    SET    salary = salary * 1.1
    WHERE  employee_id = rec.employee_id;
  END LOOP;
  COMMIT;
END apply_10pct_raise;
-- 10,000 人の従業員がいる場合: 10,000 回の UPDATE コンテキスト・スイッチ
```

### 集合ベース処理 (高速)

```sql
-- 推奨: 1 つの SQL 文ですべての処理を行う
PROCEDURE apply_10pct_raise IS
BEGIN
  UPDATE employees
  SET    salary = salary * 1.1
  WHERE  department_id = 10;
  COMMIT;
END apply_10pct_raise;
-- 行数に関わらずコンテキスト・スイッチは 1 回
```

### PL/SQL ロジックが必要な場合

純粋な SQL では変換を表現できない場合（複雑な計算、行ごとの PL/SQL ファンクションの呼び出しなど）は、バルク操作を使用します。

---

## LIMIT 句を使用した BULK COLLECT

`BULK COLLECT` は、1 回のコンテキスト・スイッチで複数の行をコレクションにフェッチします。`LIMIT` 句を使用することで、メモリ使用量を制限できます。

### LIMIT なし (大量データセットでは危険)

```sql
-- リスクあり: すべての行を一度にメモリにフェッチする
DECLARE
  TYPE t_emp_tab IS TABLE OF employees%ROWTYPE;
  l_employees t_emp_tab;
BEGIN
  SELECT * BULK COLLECT INTO l_employees FROM employees;
  -- 従業員が 500 万人いる場合、数ギガバイトの PGA を消費する可能性がある
END;
```

### LIMIT あり (推奨されるパターン)

```sql
PROCEDURE process_all_employees IS
  CURSOR c_emp IS
    SELECT employee_id, salary, department_id
    FROM   employees
    WHERE  status = 'ACTIVE';

  TYPE t_emp_tab IS TABLE OF c_emp%ROWTYPE;
  l_employees t_emp_tab;

  c_batch_size CONSTANT PLS_INTEGER := 1000;
BEGIN
  OPEN c_emp;
  LOOP
    -- 1 回の反復で最大 1000 行をフェッチ = 1 バッチにつき 1 回のコンテキスト・スイッチ
    FETCH c_emp BULK COLLECT INTO l_employees LIMIT c_batch_size;
    EXIT WHEN l_employees.COUNT = 0;

    -- バッチ処理
    FOR i IN 1..l_employees.COUNT LOOP
      -- PL/SQL のみの処理 (ここでは SQL エンジンへのスイッチは発生しない)
      IF l_employees(i).salary < 30000 THEN
        l_employees(i).salary := 30000;  -- 最低賃金の調整
      END IF;
    END LOOP;

    -- 処理済みバッチに対してバルク DML を実行
    FORALL i IN 1..l_employees.COUNT
      UPDATE employees
      SET    salary = l_employees(i).salary
      WHERE  employee_id = l_employees(i).employee_id;

    COMMIT;  -- 各バッチごとにコミットし、UNDO/REDO を管理
  END LOOP;
  CLOSE c_emp;
END process_all_employees;
```

**LIMIT サイズの選択**: 通常は 100 〜 1000 が適当です。小さすぎるとコンテキスト・スイッチが多くなり、大きすぎると PGA メモリを過剰に消費します。データ量と利用可能な PGA に基づいて調整してください。

---

## SAVE EXCEPTIONS を使用した FORALL

`FORALL` はコレクション全体を 1 回の呼び出しで SQL エンジンに送信し、DML の行ごとのコンテキスト・スイッチを排除します。

### 基本的な FORALL

```sql
DECLARE
  TYPE t_id_tab  IS TABLE OF employees.employee_id%TYPE;
  TYPE t_sal_tab IS TABLE OF employees.salary%TYPE;

  l_ids     t_id_tab  := t_id_tab(101, 102, 103, 104, 105);
  l_salaries t_sal_tab := t_sal_tab(50000, 55000, 60000, 65000, 70000);
BEGIN
  FORALL i IN 1..l_ids.COUNT
    UPDATE employees
    SET    salary = l_salaries(i)
    WHERE  employee_id = l_ids(i);

  COMMIT;
END;
```

### SAVE EXCEPTIONS を使用した FORALL

`SAVE EXCEPTIONS` がない場合、1 行でも失敗すると `FORALL` 全体が停止しロールバックされます。`SAVE EXCEPTIONS` を使用すると、エラーが収集され、処理が続行されます。

```sql
DECLARE
  TYPE t_order_tab IS TABLE OF orders%ROWTYPE;
  l_orders t_order_tab;

  -- バルク・エラー用例外
  e_bulk_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_bulk_errors, -24381);

  l_error_count NUMBER;
BEGIN
  -- ステージング表からコレクションにデータを投入
  SELECT * BULK COLLECT INTO l_orders FROM orders_staging;

  BEGIN
    FORALL i IN 1..l_orders.COUNT SAVE EXCEPTIONS
      INSERT INTO orders VALUES l_orders(i);

  EXCEPTION
    WHEN e_bulk_errors THEN
      l_error_count := SQL%BULK_EXCEPTIONS.COUNT;
      DBMS_OUTPUT.PUT_LINE(l_error_count || ' 行が失敗しました:');

      FOR i IN 1..l_error_count LOOP
        DBMS_OUTPUT.PUT_LINE(
          '  コレクション索引: ' || SQL%BULK_EXCEPTIONS(i).ERROR_INDEX ||
          '  エラー: '     || SQLERRM(-SQL%BULK_EXCEPTIONS(i).ERROR_CODE)
        );

        -- エラー・ログの記録またはエラー用テーブルへの移動
        INSERT INTO orders_load_errors (
          order_id, error_code, error_message
        ) VALUES (
          l_orders(SQL%BULK_EXCEPTIONS(i).ERROR_INDEX).order_id,
          SQL%BULK_EXCEPTIONS(i).ERROR_CODE,
          SQLERRM(-SQL%BULK_EXCEPTIONS(i).ERROR_CODE)
        );
      END LOOP;
  END;

  COMMIT;
END;
/
```

`SQL%BULK_EXCEPTIONS` は 2 つのフィールドを持つ疑似コレクションです：
- `ERROR_INDEX`: 失敗したコレクションの要素番号
- `ERROR_CODE`: Oracle エラー番号 (正の値。`SQLERRM` で使用するには負の値にする必要があります)

`SQL%BULK_ROWCOUNT(i)` は、`FORALL` 内の i 番目の文によって影響を受けた行数を報告します。

---

## パイプライン・テーブル・ファンクション (Pipelined Table Functions)

パイプライン・ファンクションは、`PIPE ROW` を使用して行を 1 つずつ返します。これにより、呼び出し側は行が生成されるたびに処理を開始でき（Unix のパイプのような動作）、結果セット全体をメモリに構築するのを避けることができます。

```sql
-- 戻り値の型を定義
CREATE TYPE t_report_row AS OBJECT (
  department_name VARCHAR2(50),
  employee_count  NUMBER,
  avg_salary      NUMBER
);

CREATE TYPE t_report_tab IS TABLE OF t_report_row;
/

-- パイプライン・ファンクション
CREATE OR REPLACE FUNCTION get_dept_report
  RETURN t_report_tab PIPELINED
IS
  CURSOR c_depts IS
    SELECT d.department_name,
           COUNT(e.employee_id) AS emp_count,
           AVG(e.salary)        AS avg_sal
    FROM   departments d
    LEFT JOIN employees e ON e.department_id = d.department_id
    GROUP BY d.department_name;
BEGIN
  FOR rec IN c_depts LOOP
    -- PIPE ROW は呼び出し側に行を即座に送る
    -- ファンクション全体が終了する前に、呼び出し側は処理を開始できる
    PIPE ROW(t_report_row(rec.department_name, rec.emp_count, rec.avg_sal));
  END LOOP;
  RETURN;  -- パイプライン・ファンクションでは値なしの RETURN が必要
END get_dept_report;
/

-- SQL でテーブル・ソースとして使用
SELECT * FROM TABLE(get_dept_report()) ORDER BY avg_salary DESC;

-- JOIN での使用
SELECT r.department_name, r.avg_salary, b.budget
FROM   TABLE(get_dept_report()) r
JOIN   dept_budgets b ON b.department_name = r.department_name;
```

**利点**: 大規模な結果セットに対してメモリ消費量が少ないこと、呼び出し側が行を受け取り次第処理を開始できること、結果に対してパラレル・クエリが可能になることが挙げられます。

---

## NOCOPY ヒント

デフォルトでは、`IN OUT` パラメータは「値渡し」されます。つまり、プロシージャ開始時と終了時にデータのコピーが作成されます。大規模なコレクションの場合、このコピー処理は高コストになります。`NOCOPY` を指定すると、「参照渡し」になり、コピーが発生しません。

```sql
-- NOCOPY なし: コレクションが 2 回コピーされる (入力時と出力時)
PROCEDURE sort_employees(p_employees IN OUT emp_collection_t) IS
BEGIN
  -- ... ソート・ロジック ...
END sort_employees;

-- NOCOPY あり: 参照が渡されるため、コピーのオーバーヘッドがない
PROCEDURE sort_employees(p_employees IN OUT NOCOPY emp_collection_t) IS
BEGIN
  -- ... ソート・ロジック ...
END sort_employees;
```

### NOCOPY の比較

| 特性 | 値渡し (デフォルト) | NOCOPY (参照渡し) |
|---|---|---|
| パフォーマンス | 大規模コレクションでは遅い | 高速 — コピーなし |
| 分離性 | 例外発生時、OUT パラメータは元の状態に戻る | 例外発生時、一部変更されたデータが残る可能性がある |
| 安全性 | ロールバックが必要な場合に安全 | エラー時に呼び出し側が一部変更されたデータを見てしまう |
| 大規模コレクション (>100 要素) | オーバーヘッドが顕著 | 大幅に改善 |

**いつ使用すべきか**: パフォーマンスのボトルネックであることが測定された、数百から数千の要素を持つコレクション。エラー時に一部変更されたデータが残ることが許容されないケースには適していません。

---

## ファンクションの RESULT_CACHE

`RESULT_CACHE` は、入力パラメータ値に基づいてファンクションの結果を格納します。同じ引数で再度呼び出されると、ファンクションの本体を実行せずにキャッシュされた結果を返します。

```sql
-- ファンクションの結果はセッションをまたいで共有プールにキャッシュされる
CREATE OR REPLACE FUNCTION get_tax_rate(
  p_country_code IN VARCHAR2,
  p_tax_category IN VARCHAR2
) RETURN NUMBER
  RESULT_CACHE RELIES_ON (tax_rates)
IS
  l_rate NUMBER;
BEGIN
  SELECT rate INTO l_rate
  FROM   tax_rates
  WHERE  country_code = p_country_code
    AND  category     = p_tax_category;
  RETURN l_rate;
EXCEPTION
  WHEN NO_DATA_FOUND THEN RETURN 0;
END get_tax_rate;
/
```

`RELIES_ON (tax_rates)` は、`tax_rates` テーブルが変更されたときにキャッシュを無効化するように Oracle に指示します。Oracle 11gR2 以降では `RELIES_ON` はオプションであり、Oracle が自動的に依存関係を検出します。

### キャッシュの無効化

結果キャッシュは、以下の場合に自動的に無効化されます：
- `RELIES_ON` にリストされたテーブルが更新された（DML + コミット）
- ファンクションが再コンパイルされた
- `DBMS_RESULT_CACHE.FLUSH` が手動で実行された
- 共有プールがフラッシュされた

```sql
-- 手動のキャッシュ管理
EXEC DBMS_RESULT_CACHE.FLUSH;          -- すべてフラッシュ
EXEC DBMS_RESULT_CACHE.BYPASS(TRUE);   -- キャッシュを一時停止
EXEC DBMS_RESULT_CACHE.BYPASS(FALSE);  -- 再開

-- キャッシュの監視
SELECT name, status, scan_count, invalidations
FROM   v$result_cache_objects
WHERE  type = 'Result'
ORDER BY scan_count DESC;
```

---

## DETERMINISTIC ファンクション

`DETERMINISTIC` は、同じ入力値に対して常に同じ結果を返すことを Oracle に伝えます。これにより Oracle は：
1. 単一の SQL 文の中で結果をキャッシュできる（同じ入力で何度も呼び出さない）
2. ファンクション索引で使用できる

```sql
CREATE OR REPLACE FUNCTION full_name(
  p_first IN VARCHAR2,
  p_last  IN VARCHAR2
) RETURN VARCHAR2 DETERMINISTIC IS
BEGIN
  RETURN p_last || ', ' || p_first;
END full_name;
/

-- クエリ実行中、同じ (first_name, last_name) の組み合わせには 1 回だけ呼び出される
SELECT full_name(first_name, last_name) FROM employees;

-- ファンクション索引での使用が可能
CREATE INDEX idx_emp_fullname ON employees(full_name(first_name, last_name));
```

**RESULT_CACHE との主な違い**: `DETERMINISTIC` は単一の SQL 実行内でのヒントであり、`RESULT_CACHE` は呼び出しやセッションをまたいで結果を保持します。

**警告**: 外部の状態（順序、`SYSDATE`、セッション変数など）に依存しているファンクションを `DETERMINISTIC` と宣言すると、誤った結果が返されます。正しい宣言かどうかを Oracle は検証しないため、開発者の責任となります。

---

## 不要なパースの回避

ハード・パースは高コストです。SQL エンジンは構文チェック、オブジェクトの解決、実行計画の作成、実行可能コードの生成を行う必要があります。ソフト・パース（キャッシュされたカーソルの再利用）ははるかに低コストです。PL/SQL では、コンパイル時にバインドを行う「静的 SQL」を使用することでこれを支援します。

```sql
-- 推奨例: PL/SQL 静的 SQL — 1 回のハード・パースで済み、毎回再利用される
PROCEDURE get_employee(p_id IN NUMBER) IS
  l_emp employees%ROWTYPE;
BEGIN
  SELECT * INTO l_emp FROM employees WHERE employee_id = p_id;
END;

-- 回避すべき例: リテラルを連結した動的 SQL — 呼び出しごとにハード・パース
PROCEDURE get_employee_bad(p_id IN NUMBER) IS
  l_emp employees%ROWTYPE;
  l_sql VARCHAR2(200);
BEGIN
  l_sql := 'SELECT * FROM employees WHERE employee_id = ' || p_id;
  -- ^ p_id ごとに異なる SQL テキスト = 異なるカーソル = 毎回ハード・パース
  EXECUTE IMMEDIATE l_sql INTO l_emp;
END;

-- 許容例: バインド変数を使用した動的 SQL — 1 回のパースでカーソルを再利用
PROCEDURE get_employee_dynamic(p_id IN NUMBER) IS
  l_emp employees%ROWTYPE;
BEGIN
  EXECUTE IMMEDIATE
    'SELECT * FROM employees WHERE employee_id = :1'
    INTO l_emp USING p_id;
  -- 毎回同じ SQL テキスト = 1 回のハード・パース後、カーソルを再利用
END;
```

---

## パフォーマンス・チェックリスト

| 項目 | 問題点 | 解決策 |
|---|---|---|
| ループ内での DML | 行単位処理 = 1行ごとのコンテキスト・スイッチ | コレクションを使用した `FORALL` を使用 |
| ループ内での `SELECT` | N+1 クエリ・パターン | `BULK COLLECT` で一括取得してから処理 |
| `LIMIT` なしの `BULK COLLECT` | メモリ使用量が無制限 | 常に `LIMIT 500` などで制限をかける |
| 大規模コレクションの `IN OUT` | 開始・終了時のコピーによるオーバーヘッド | `NOCOPY` ヒントを追加 |
| 同じパラメータでの繰り返し呼び出し | 同一 SQL の重複実行 | `RESULT_CACHE` を使用 |
| 値を連結した動的 SQL | 値ごとにハード・パースが発生 | `:n` バインド変数を使用 |
| 戻り値全体を構築してから返すパイプライン | 高メモリ、最初の行の遅延 | `PIPELINED` と `PIPE ROW` を使用 |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **Oracle 11g以降**: ファンクションの `RESULT_CACHE` が導入されました。11gR1 では `RELIES_ON` が必須でしたが、11gR2 以降は自動検出されます。
- **Oracle 12c以降**: `PRAGMA UDF` (User Defined Function) ヒントは、SQL から PL/SQL ファンクションを呼び出す際のコンテキスト・スイッチを削減します。SQL 式で代替できない場合に使用します。
- **全バージョン**: `BULK COLLECT` と `FORALL` は Oracle 9i から利用可能で、現在も主要なバルク操作ツールです。

```sql
-- Oracle 12c+: PRAGMA UDF により、SQL から呼ばれるファンクションのスイッチを削減
CREATE OR REPLACE FUNCTION calculate_bonus(
  p_salary     IN NUMBER,
  p_percentage IN NUMBER
) RETURN NUMBER IS
  PRAGMA UDF;  -- ヒント: この関数は SQL から呼ばれるため、最適化を指示
BEGIN
  RETURN ROUND(p_salary * p_percentage / 100, 2);
END calculate_bonus;
/
```

---

## ソース

- Oracle Database 19c PL/SQL Language Reference — Optimization and Tuning: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-optimization-and-tuning.html
- Oracle Database 19c PL/SQL Packages Reference — DBMS_RESULT_CACHE: https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESULT_CACHE.html
- Oracle Database 19c Reference — PLSQL_OPTIMIZE_LEVEL: https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/PLSQL_OPTIMIZE_LEVEL.html
- Oracle Database 19c PL/SQL Language Reference — INLINE Pragma: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/INLINE-pragma.html
