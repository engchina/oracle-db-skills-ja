# PL/SQL カーソル (Cursors)

## 概要

カーソルは、Oracle が SQL 文を処理するために作成するコンテキスト・エリアへのハンドルです。セッションで実行されるすべての SQL 文は、カーソルを使用します。PL/SQL には、暗黙的カーソル（自動管理）、明示的カーソル（開発者が制御）、およびカーソル変数（結果セットを渡すための REF CURSOR）があります。カーソルのライフサイクル、属性、および安全なパターンを理解することは、リソース・リークや不正確な結果処理を防ぐために不可欠です。

---

## 暗黙的カーソルの属性

Oracle は、明示的カーソルの一部ではないすべての SQL 文に対して、暗黙的カーソルを自動的に作成します。文の実行後、4 つの属性によって結果が記述されます。暗黙的カーソルには `SQL%` プレフィックスを使用してアクセスできます。

| 属性 | 型 | 説明 |
|---|---|---|
| `SQL%FOUND` | BOOLEAN | 直前の SQL が少なくとも 1 行に影響を与えた場合は TRUE |
| `SQL%NOTFOUND` | BOOLEAN | 直前の SQL が 1 行にも影響を与えなかった場合は TRUE |
| `SQL%ROWCOUNT` | INTEGER | 直前の SQL で処理された行数 |
| `SQL%ISOPEN` | BOOLEAN | 暗黙的カーソルの場合は常に FALSE (Oracle が即座に閉じるため) |

```sql
PROCEDURE update_employee_salary(
  p_employee_id IN employees.employee_id%TYPE,
  p_new_salary  IN employees.salary%TYPE
) IS
BEGIN
  UPDATE employees
  SET    salary = p_new_salary
  WHERE  employee_id = p_employee_id;

  IF SQL%NOTFOUND THEN
    RAISE_APPLICATION_ERROR(-20001, 'Employee not found: ' || p_employee_id);
  END IF;

  DBMS_OUTPUT.PUT_LINE('Updated ' || SQL%ROWCOUNT || ' row(s)');
  COMMIT;
END update_employee_salary;
/

PROCEDURE deactivate_old_sessions IS
BEGIN
  DELETE FROM user_sessions
  WHERE  last_activity < SYSDATE - 30;

  DBMS_OUTPUT.PUT_LINE('Deleted ' || SQL%ROWCOUNT || ' expired sessions');

  IF SQL%FOUND THEN
    COMMIT;
  END IF;
END deactivate_old_sessions;
/
```

**重要**: `SQL%` 属性は、最後に実行された SQL 文の結果のみを反映します。例外ハンドラ内など、他の SQL 文を実行すると、これらの属性は上書きされます。そのため、DML の直後に値をキャプチャする必要があります。

```sql
-- 誤り: SQL%ROWCOUNT は例外ハンドラ内の INSERT によって上書きされる
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 10;
DECLARE
  l_updated NUMBER := SQL%ROWCOUNT;  -- 正解: 直後にキャプチャする
BEGIN
  INSERT INTO salary_audit (changed_rows) VALUES (l_updated);
  -- ここでの SQL%ROWCOUNT は UPDATE ではなく INSERT の結果を反映する
END;
```

---

## 明示的カーソルのライフサイクル

明示的カーソルは、宣言、オープン、フェッチ、クローズのステップを踏みます。各ステップは独立しており、開発者が制御します。

```sql
DECLARE
  -- 1. 宣言 (DECLARE): クエリを定義 (まだ実行されない)
  CURSOR c_high_earners IS
    SELECT employee_id, last_name, salary
    FROM   employees
    WHERE  salary > 100000
    ORDER BY salary DESC;

  -- 行データを受け取るためのレコード型
  l_emp c_high_earners%ROWTYPE;
BEGIN
  -- 2. オープン (OPEN): クエリを実行し、カーソルを最初の行の前に配置
  OPEN c_high_earners;

  -- 3. フェッチ (FETCH): 次の行を変数に取得
  LOOP
    FETCH c_high_earners INTO l_emp;
    EXIT WHEN c_high_earners%NOTFOUND;  -- 行がなくなったらループを抜ける

    DBMS_OUTPUT.PUT_LINE(l_emp.last_name || ': $' || l_emp.salary);
  END LOOP;

  -- 4. クローズ (CLOSE): カーソルのリソースを解放
  CLOSE c_high_earners;
EXCEPTION
  WHEN OTHERS THEN
    -- リソース・リークを防ぐため、エラー時には必ずカーソルを閉じる
    IF c_high_earners%ISOPEN THEN
      CLOSE c_high_earners;
    END IF;
    RAISE;
END;
/
```

### 明示的カーソルの属性

| 属性 | 説明 |
|---|---|
| `cursor%ISOPEN` | カーソルがオープンされていれば TRUE |
| `cursor%FOUND` | 直前の FETCH で行が返されていれば TRUE |
| `cursor%NOTFOUND` | 直前の FETCH で行が返されなかった（結果の末尾）場合に TRUE |
| `cursor%ROWCOUNT` | そのカーソルからこれまでにフェッチされた累計行数 |

---

## カーソル FOR ループ (推奨パターン)

カーソル FOR ループは、すべての行を反復処理するための推奨される方法です。Oracle が暗黙的にカーソルのオープン、フェッチ、およびクローズを行うため、クローズし忘れるリスクがありません。

```sql
-- 推奨パターン: カーソル FOR ループ
-- Oracle が OPEN, FETCH, EXIT, CLOSE を自動的に処理
PROCEDURE print_department_employees(p_dept_id IN NUMBER) IS
BEGIN
  FOR emp IN (
    SELECT employee_id, last_name, salary
    FROM   employees
    WHERE  department_id = p_dept_id
    ORDER BY last_name
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(emp.last_name || ' - $' || emp.salary);
  END LOOP;
  -- 明示的なオープン/クローズは不要。ループが正常終了したとき、または例外発生時に自動で閉じる
END print_department_employees;
/

-- 名前付きカーソルを使用する場合 (カーソル属性が必要な場合に有効)
DECLARE
  CURSOR c_depts IS
    SELECT department_id, department_name FROM departments ORDER BY department_name;
BEGIN
  FOR dept IN c_depts LOOP
    DBMS_OUTPUT.PUT_LINE(
      dept.department_name ||
      ' (row ' || c_depts%ROWCOUNT || ')'
    );
  END LOOP;
END;
/
```

**カーソル FOR ループではなく明示的なサイクルを使用すべきケース**:
- `BULK COLLECT ... LIMIT` を使用してバッチでフェッチする場合
- `%ISOPEN` のチェックが必要な場合（他のプロシージャにカーソルを渡す場合など）
- 部分的な反復が必要な場合（すべての行を処理する前に終了する場合）

---

## パラメータ付きカーソル

カーソルはパラメータを受け取ることができるため、異なるフィルタ値で再利用できます。

```sql
DECLARE
  -- パラメータ付きカーソル: 呼び出しごとに異なる動作
  CURSOR c_employees(
    p_dept_id    IN employees.department_id%TYPE,
    p_min_salary IN employees.salary%TYPE DEFAULT 0
  ) IS
    SELECT employee_id, last_name, salary
    FROM   employees
    WHERE  department_id = p_dept_id
      AND  salary >= p_min_salary
    ORDER BY salary DESC;

  l_emp c_employees%ROWTYPE;
BEGIN
  -- 1回目の使用: 部門 10 の全員
  FOR emp IN c_employees(p_dept_id => 10) LOOP
    DBMS_OUTPUT.PUT_LINE('[Dept 10] ' || emp.last_name);
  END LOOP;

  -- 2回目の使用: 部門 20 の高額給与者のみ
  FOR emp IN c_employees(p_dept_id => 20, p_min_salary => 80000) LOOP
    DBMS_OUTPUT.PUT_LINE('[Dept 20 high earner] ' || emp.last_name);
  END LOOP;
END;
/
```

---

## 弱い REF CURSOR と 強い REF CURSOR

`REF CURSOR` はカーソル変数であり、プログラム間で渡すことができるカーソルへのポインタです。REF CURSOR を使用すると、PL/SQL からクライアント・アプリケーション、またはパッケージ間で結果セットを返すことができます。

### 弱い REF CURSOR (SYS_REFCURSOR)

戻り値の型の制約がありません。任意のクエリを指すことができます。`SYS_REFCURSOR` は組み込みの弱い型です。

```sql
-- 弱い REF CURSOR を返すファンクション
CREATE OR REPLACE FUNCTION get_employees_ref(
  p_dept_id IN NUMBER
) RETURN SYS_REFCURSOR IS
  l_cursor SYS_REFCURSOR;
BEGIN
  -- 任意のクエリでオープン可能
  OPEN l_cursor FOR
    SELECT employee_id, last_name, salary
    FROM   employees
    WHERE  department_id = p_dept_id
    ORDER BY last_name;

  RETURN l_cursor;  -- クローズは呼び出し側の責任
END get_employees_ref;
/

-- 呼び出し側の使用例
DECLARE
  l_cursor SYS_REFCURSOR;
  l_id     NUMBER;
  l_name   VARCHAR2(50);
  l_sal    NUMBER;
BEGIN
  l_cursor := get_employees_ref(10);

  LOOP
    FETCH l_cursor INTO l_id, l_name, l_sal;
    EXIT WHEN l_cursor%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(l_name || ': ' || l_sal);
  END LOOP;

  CLOSE l_cursor;  -- 呼び出し側が閉じる必要がある
END;
/
```

### 強い REF CURSOR

宣言された戻り値の型を持ちます。Oracle はコンパイル時に、カーソルが互換性のあるクエリでオープンされているかどうかを検証します。

```sql
-- パッケージ仕様部で強い REF CURSOR 型を定義
CREATE OR REPLACE PACKAGE employee_pkg AS
  -- 強い型: 正確にこの構造を返す必要がある
  TYPE t_employee_cursor IS REF CURSOR RETURN employees%ROWTYPE;

  FUNCTION get_department_employees(
    p_dept_id IN NUMBER
  ) RETURN t_employee_cursor;

END employee_pkg;
/

CREATE OR REPLACE PACKAGE BODY employee_pkg AS

  FUNCTION get_department_employees(
    p_dept_id IN NUMBER
  ) RETURN t_employee_cursor IS
    l_cursor t_employee_cursor;
  BEGIN
    OPEN l_cursor FOR
      SELECT * FROM employees WHERE department_id = p_dept_id;
    -- Oracle はコンパイル時に SELECT * FROM employees が
    -- t_employee_cursor の戻り値型 (employees%ROWTYPE) と一致するか検証する
    RETURN l_cursor;
  END get_department_employees;

END employee_pkg;
/
```

| | 弱い REF CURSOR | 強い REF CURSOR |
|---|---|---|
| 戻り値の型 | 任意 | 型定義時に宣言 |
| コンパイル時チェック | なし | あり |
| 柔軟性 | 高い | 低い |
| エラー検出 | 実行時 | コンパイル時 |
| `SYS_REFCURSOR` の利用 | 可 (組み込み済み) | 不可 (型の宣言が必要) |

---

## プロシージャ間でのカーソル変数の受け渡し

REF CURSOR はパラメータとして渡すことができるため、あるプロシージャでカーソルをオープンし、別のプロシージャでそれを処理することが可能です。

```sql
-- 生産者: カーソルをオープンし OUT パラメータで返す
PROCEDURE open_report_cursor(
  p_start_date IN  DATE,
  p_end_date   IN  DATE,
  p_cursor     OUT SYS_REFCURSOR
) IS
BEGIN
  OPEN p_cursor FOR
    SELECT o.order_id, o.order_date, c.customer_name, o.total_amount
    FROM   orders o
    JOIN   customers c ON c.customer_id = o.customer_id
    WHERE  o.order_date BETWEEN p_start_date AND p_end_date
    ORDER  BY o.order_date;
END open_report_cursor;
/

-- 消費者: カーソルを処理する
PROCEDURE process_report_cursor(
  p_cursor IN OUT SYS_REFCURSOR
) IS
  l_order_id     NUMBER;
  l_order_date   DATE;
  l_customer     VARCHAR2(100);
  l_total        NUMBER;
BEGIN
  LOOP
    FETCH p_cursor INTO l_order_id, l_order_date, l_customer, l_total;
    EXIT WHEN p_cursor%NOTFOUND;
    -- 各行を処理...
    DBMS_OUTPUT.PUT_LINE(l_customer || ': $' || l_total);
  END LOOP;
  -- 消費者がカーソルを閉じる
  CLOSE p_cursor;
END process_report_cursor;
/

-- オーケストレータ
DECLARE
  l_cur SYS_REFCURSOR;
BEGIN
  open_report_cursor(SYSDATE - 30, SYSDATE, l_cur);
  process_report_cursor(l_cur);
END;
/
```

---

## カーソル式 (SELECT 内の CURSOR)

カーソル式は SELECT 文の中にサブカーソルを埋め込み、階層的な結果セットを可能にします。

```sql
-- 各部門と、その部門に属する従業員のネストしたカーソルを返す
SELECT
  d.department_name,
  CURSOR(
    SELECT e.last_name, e.salary
    FROM   employees e
    WHERE  e.department_id = d.department_id
    ORDER BY e.salary DESC
  ) AS employee_cursor
FROM departments d;
```

カーソル式は主に OCI (C 言語) アプリケーションや、JDBC に階層データを渡す場合に使用されます。PL/SQL では、ネストした REF CURSOR 変数を使用して処理できます。

---

## カーソル・リークの回避

オープンされたまま閉じられないカーソルは、**カーソル・リーク**と呼ばれます。リークがたまると、セッションは `OPEN_CURSORS` の制限に達します (`ORA-01000: maximum open cursors exceeded`)。

### 一般的なリーク・パターンと修正方法

```sql
-- リーク: CLOSE の前に例外が発生
DECLARE
  CURSOR c IS SELECT * FROM large_table;
  l_row large_table%ROWTYPE;
BEGIN
  OPEN c;
  FETCH c INTO l_row;
  risky_procedure;  -- ここで例外が発生！
  CLOSE c;          -- ここには到達しない
EXCEPTION
  WHEN OTHERS THEN
    -- c は開いたまま — リーク発生！
    RAISE;
END;

-- 修正方法: 例外ハンドラで %ISOPEN をチェック
EXCEPTION
  WHEN OTHERS THEN
    IF c%ISOPEN THEN CLOSE c; END IF;
    RAISE;
END;

-- 最善の修正方法: カーソル FOR ループを使用 (正常終了時も例外時も自動クローズ)
BEGIN
  FOR row IN (SELECT * FROM large_table) LOOP
    risky_procedure;  -- ここで例外が起きても、カーソルは閉じられる
  END LOOP;
END;
```

### REF CURSOR リーク・パターン

```sql
-- リーク: ファンクションがカーソルを返すが、呼び出し側がクローズを忘れる
DECLARE
  l_cur SYS_REFCURSOR;
BEGIN
  l_cur := get_employees_ref(10);
  -- ... 行を処理 ...
  -- CLOSE l_cur; を忘れている
END;  -- セッションが続く限りカーソルがリーク

-- 修正方法: 例外時を含め、常に REF CURSOR をクローズする
DECLARE
  l_cur SYS_REFCURSOR;
BEGIN
  l_cur := get_employees_ref(10);
  -- 処理...
  CLOSE l_cur;
EXCEPTION
  WHEN OTHERS THEN
    IF l_cur%ISOPEN THEN CLOSE l_cur; END IF;
    RAISE;
END;
```

### オープン・カーソルの監視

```sql
-- オープン・カーソルが多いセッションを特定
SELECT s.sid, s.username, s.program, COUNT(*) AS open_cursor_count
FROM   v$open_cursor oc
JOIN   v$session     s  ON s.sid = oc.sid
WHERE  oc.cursor_type = 'OPEN'
GROUP BY s.sid, s.username, s.program
ORDER BY open_cursor_count DESC;

-- セッションの OPEN_CURSORS 制限値
SHOW PARAMETER open_cursors;
```

---

## ベスト・プラクティス

- 単純な反復処理には、**カーソル FOR ループを優先**してください。リークが発生せず、構文も簡潔です。
- バルク処理が必要な場合は、**`BULK COLLECT ... LIMIT`** を使用してください。バルク処理の効率が必要な場合は、カーソル FOR ループを使用しないでください。
- パッケージ間での結果セットの受け渡しや、アプリケーションへのカーソル返却には、**`SYS_REFCURSOR`** (弱い型) を使用してください。
- 戻り値の構造が固定されており、コンパイル時に検証を行いたい場合は、**強い REF CURSOR** を使用してください。
- 明示的カーソルおよび REF CURSOR は**常に閉じてください**。例外ハンドラで `%ISOPEN` チェックを使用するのが確実です。
- DML 文の直後、他の SQL が実行される前に、即座に **`SQL%ROWCOUNT` を取得**してください。
- 類似した複数のカーソルを作成するのではなく、**カーソルをパラメータ化**してください。
- **`SQL%ISOPEN` は絶対にチェックしないでください**。暗黙的カーソルの場合は常に FALSE です。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| エラー発生時に CLOSE し忘れる | カーソル・リーク → ORA-01000 | カーソル FOR ループを使用するか、ハンドラで `%ISOPEN` チェックを行う |
| 別の SQL を実行した後に `SQL%ROWCOUNT` をチェック | 誤った行数が返される | DML の直後に変数にキャプチャする |
| `SQL%ISOPEN` の使用 | 常に FALSE であり、無意味 | チェックしない。明示的カーソルには `cursor_name%ISOPEN` を使用する |
| すでにオープンされているカーソルを再度更新 | ORA-06511 | OPEN 前に `%ISOPEN` をチェックする |
| クローズされたカーソルを操作 | ORA-01001 | FETCH 前に `%ISOPEN` をチェックする |
| 呼び出し側がクローズしない REF CURSOR を返す | カーソル・リーク | 誰がクローズを担当するか明文化し、一貫したルールに従う |
| EOF 後に EXIT なしで FETCH | 無限ループ | 常に `EXIT WHEN cursor%NOTFOUND` を記述する |
| ループ内で DML を行うカーソル FOR ループ | `SQL%ROWCOUNT` はフェッチ件数ではなく最後の DML 結果を反映 | フェッチ件数には `cursor%ROWCOUNT` を使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **全バージョン**: `SYS_REFCURSOR` は Oracle 9i 以降で使用可能です。
- **Oracle 12c以降**: `DBMS_SQL.RETURN_RESULT` による暗黙的な結果セット。明示的な OUT パラメータなしでストアド・プロシージャから JDBC/OCI 呼び出し側に結果セットを返せます。SQL Server からの移行互換性に便利です。

```sql
-- Oracle 12c以降: 暗黙的な結果セット (DBMS_SQL.RETURN_RESULT)
CREATE OR REPLACE PROCEDURE get_dept_report(p_dept_id IN NUMBER) AS
  l_cursor SYS_REFCURSOR;
BEGIN
  OPEN l_cursor FOR
    SELECT * FROM employees WHERE department_id = p_dept_id;

  -- OUT パラメータなしでクライアントに結果セットを送信
  DBMS_SQL.RETURN_RESULT(l_cursor);
  -- 送信後、Oracle が自動的にカーソルを閉じる
END get_dept_report;
/
```

---

## ソース

- [Oracle Database PL/SQL Language Reference 19c — Cursors](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/static-sql.html) — 暗黙的カーソル、明示的カーソル、カーソル FOR ループ、REF CURSOR
- [DBMS_SQL (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQL.html) — RETURN_RESULT (12c+), TO_REFCURSOR (11gR2+)
- [Oracle Database Reference 19c — V$OPEN_CURSOR](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-OPEN_CURSOR.html) — オープン・カーソルの監視
