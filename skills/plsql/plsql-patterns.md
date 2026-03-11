# PL/SQL デザイン・パターン (Design Patterns)

## 概要

PL/SQL における実証済みのデザイン・パターンを適用することで、保守性が高く、パフォーマンスに優れ、テスト可能なコードを作成できます。このガイドでは、最も広く使用されているパターン（Table API、自律型トランザクション、パイプライン・ファンクション、レコード型、オブジェクト・リレーショナル型、複数結果セット・パターン、およびカーソルの再利用）について説明します。

---

## Table API (TAPI) パターン

Table API (TAPI) は、特定の表に対するすべての DML 操作をパッケージ内にカプセル化し、挿入、更新、削除、および問合せのための単一の制御ポイントを提供します。これにより、SQL をビジネス・ロジックから分離し、呼び出し側に影響を与えずにデータ・レイヤーのリファクタリングを可能にします。

```sql
-- EMPLOYEES 表用に生成された TAPI の例
CREATE OR REPLACE PACKAGE employees_tapi AS

  -- 表の行に対応するサブタイプ
  SUBTYPE t_row IS employees%ROWTYPE;

  -- 挿入: 新しい主キーを返す
  FUNCTION ins(
    p_first_name    IN employees.first_name%TYPE,
    p_last_name     IN employees.last_name%TYPE,
    p_email         IN employees.email%TYPE,
    p_hire_date     IN employees.hire_date%TYPE    DEFAULT SYSDATE,
    p_job_id        IN employees.job_id%TYPE,
    p_salary        IN employees.salary%TYPE       DEFAULT NULL,
    p_department_id IN employees.department_id%TYPE DEFAULT NULL
  ) RETURN employees.employee_id%TYPE;

  -- 主キーによる更新
  PROCEDURE upd(
    p_employee_id   IN employees.employee_id%TYPE,
    p_first_name    IN employees.first_name%TYPE    DEFAULT NULL,
    p_last_name     IN employees.last_name%TYPE     DEFAULT NULL,
    p_salary        IN employees.salary%TYPE        DEFAULT NULL,
    p_department_id IN employees.department_id%TYPE DEFAULT NULL
  );

  -- 主キーによる削除
  PROCEDURE del(
    p_employee_id IN employees.employee_id%TYPE
  );

  -- 主キーによる取得
  FUNCTION get(
    p_employee_id IN employees.employee_id%TYPE
  ) RETURN t_row;

  -- 存在チェック
  FUNCTION exists_by_id(
    p_employee_id IN employees.employee_id%TYPE
  ) RETURN BOOLEAN;

END employees_tapi;
/

CREATE OR REPLACE PACKAGE BODY employees_tapi AS

  FUNCTION ins(
    p_first_name    IN employees.first_name%TYPE,
    p_last_name     IN employees.last_name%TYPE,
    p_email         IN employees.email%TYPE,
    p_hire_date     IN employees.hire_date%TYPE    DEFAULT SYSDATE,
    p_job_id        IN employees.job_id%TYPE,
    p_salary        IN employees.salary%TYPE       DEFAULT NULL,
    p_department_id IN employees.department_id%TYPE DEFAULT NULL
  ) RETURN employees.employee_id%TYPE IS
    l_employee_id employees.employee_id%TYPE;
  BEGIN
    INSERT INTO employees (
      employee_id, first_name, last_name, email,
      hire_date, job_id, salary, department_id
    ) VALUES (
      employees_seq.NEXTVAL, p_first_name, p_last_name, p_email,
      p_hire_date, p_job_id, p_salary, p_department_id
    ) RETURNING employee_id INTO l_employee_id;

    RETURN l_employee_id;
  END ins;

  PROCEDURE upd(
    p_employee_id   IN employees.employee_id%TYPE,
    p_first_name    IN employees.first_name%TYPE    DEFAULT NULL,
    p_last_name     IN employees.last_name%TYPE     DEFAULT NULL,
    p_salary        IN employees.salary%TYPE        DEFAULT NULL,
    p_department_id IN employees.department_id%TYPE DEFAULT NULL
  ) IS
  BEGIN
    UPDATE employees
    SET    first_name    = NVL(p_first_name,    first_name),
           last_name     = NVL(p_last_name,     last_name),
           salary        = NVL(p_salary,        salary),
           department_id = NVL(p_department_id, department_id),
           updated_at    = SYSDATE
    WHERE  employee_id = p_employee_id;

    IF SQL%ROWCOUNT = 0 THEN
      RAISE_APPLICATION_ERROR(-20001, '従業員が見つかりません ID: ' || p_employee_id);
    END IF;
  END upd;
  
  -- 他のメソッドも同様に実装...

END employees_tapi;
/
```

**TAPI ツール**: Oracle には `tapi_gen` などのジェネレータがあり、SQL Developer のデータ・モデラーでも TAPI パッケージを自動生成できます。

---

## 監査とロギングのための自律型トランザクション・パターン

自律型トランザクション・パターンを使用すると、メインのトランザクションとは独立して監査やログの記録をコミットできます。これにより、メイン処理がロールバックされてもログを確実に残すことができます。

```sql
CREATE OR REPLACE PACKAGE BODY audit_pkg AS

  PROCEDURE log_change(
    p_table_name  IN VARCHAR2,
    p_operation   IN VARCHAR2,  -- INSERT / UPDATE / DELETE
    p_row_id      IN VARCHAR2,
    p_old_values  IN VARCHAR2 DEFAULT NULL,
    p_new_values  IN VARCHAR2 DEFAULT NULL
  ) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    INSERT INTO audit_log (
      audit_id, log_timestamp, db_user, table_name,
      operation, row_identifier, old_values, new_values
    ) VALUES (
      audit_seq.NEXTVAL, SYSTIMESTAMP, SYS_CONTEXT('USERENV', 'SESSION_USER'),
      p_table_name, p_operation, p_row_id, p_old_values, p_new_values
    );
    COMMIT;  -- 自律型コミット: 呼び出し側のトランザクションには影響しない
  EXCEPTION
    WHEN OTHERS THEN
      ROLLBACK;  -- 自律型トランザクションのみロールバック
      -- ビジネス・ロジックを中断させないため、ここでは例外を再発生させない
  END log_change;

END audit_pkg;
/

-- DML トリガーでの使用例
CREATE OR REPLACE TRIGGER employees_audit_trg
  AFTER INSERT OR UPDATE OR DELETE ON employees
  FOR EACH ROW
BEGIN
  IF INSERTING THEN
    audit_pkg.log_change('EMPLOYEES', 'INSERT', :NEW.employee_id, NULL, :NEW.salary);
  ELSIF UPDATING THEN
    audit_pkg.log_change('EMPLOYEES', 'UPDATE', :NEW.employee_id, :OLD.salary, :NEW.salary);
  END IF;
END;
/
```

---

## ストリーミング結果のためのパイプライン・ファンクション

パイプライン・ファンクションは、行が生成されるたびに呼び出し側に返します。結果セット全体をメモリに構築することなく処理できるため、大規模なデータ変換や ETL 処理に理想的です。

```sql
-- 戻り値の型を定義
CREATE OR REPLACE TYPE t_sales_summary_row AS OBJECT (
  region_name    VARCHAR2(50),
  units_sold     NUMBER,
  revenue        NUMBER(15,2)
);

CREATE OR REPLACE TYPE t_sales_summary_tab AS TABLE OF t_sales_summary_row;
/

-- パイプライン・ファンクション
CREATE OR REPLACE FUNCTION get_sales_summary(
  p_start_date IN DATE,
  p_end_date   IN DATE
) RETURN t_sales_summary_tab PIPELINED IS
  CURSOR c_sales IS
    SELECT region_name, SUM(quantity), SUM(quantity * price)
    FROM   sales
    WHERE  sale_date BETWEEN p_start_date AND p_end_date
    GROUP BY region_name;
BEGIN
  FOR rec IN c_sales LOOP
    PIPE ROW(t_sales_summary_row(rec.region_name, rec."SUM(QUANTITY)", rec."SUM(QUANTITY*PRICE)"));
  END LOOP;
  RETURN;
EXCEPTION
  WHEN NO_DATA_NEEDED THEN
    NULL;  -- 呼び出し側がデータの取得を途中で止めた場合（正常）
END;
/

-- SQL で表のように使用
SELECT * FROM TABLE(get_sales_summary(DATE '2024-01-01', DATE '2024-12-31'));
```

---

## PL/SQL レコード型 vs %ROWTYPE

### %ROWTYPE (表やカーソルに基づいた型)

```sql
DECLARE
  -- 表の構造に基づき、列が追加されれば自動的に適応する
  l_emp  employees%ROWTYPE;

  -- カーソルに基づき、選択された列のみを含む
  CURSOR c_report IS SELECT employee_id, last_name FROM employees;
  l_row  c_report%ROWTYPE;
BEGIN
  SELECT * INTO l_emp FROM employees WHERE employee_id = 100;
END;
```

### 明示的なレコード型 (カスタム構造)

```sql
DECLARE
  -- 集計や変換後のデータのためのカスタム型
  TYPE t_emp_summary IS RECORD (
    employee_id   NUMBER,
    full_name     VARCHAR2(100),
    salary_band   VARCHAR2(10)
  );
  l_summary t_emp_summary;
BEGIN
  l_summary.employee_id := 100;
  l_summary.full_name   := '田中 太郎';
END;
```

---

## PL/SQL におけるオブジェクト型

Oracle のオブジェクト型は、カプセル化、継承、ポリモーフィズムといったオブジェクト指向の概念をデータベースにもたらします。

```sql
-- メソッドを持つベースとなるオブジェクト型
CREATE OR REPLACE TYPE shape_t AS OBJECT (
  color  VARCHAR2(20),
  MEMBER FUNCTION area RETURN NUMBER
) NOT FINAL;  -- サブタイプの作成を許可
/

-- 矩形 (rectangle) サブタイプ
CREATE OR REPLACE TYPE rectangle_t UNDER shape_t (
  width  NUMBER,
  height NUMBER,
  OVERRIDING MEMBER FUNCTION area RETURN NUMBER
);
/
```

---

## REF CURSOR による複数結果セットの返却

複数の `OUT SYS_REFCURSOR` パラメータを持つプロシージャは、独立した複数の結果セットを呼び出し側に返すことができます。

```sql
CREATE OR REPLACE PROCEDURE get_order_dashboard(
  p_customer_id    IN  NUMBER,
  p_pending_orders OUT SYS_REFCURSOR,
  p_summary        OUT SYS_REFCURSOR
) AS
BEGIN
  -- 1つ目の結果セット: 保留中の注文
  OPEN p_pending_orders FOR
    SELECT order_id, order_date FROM orders
    WHERE customer_id = p_customer_id AND status = 'PENDING';

  -- 2つ目の結果セット: 集計統計
  OPEN p_summary FOR
    SELECT COUNT(*), SUM(total_amount) FROM orders
    WHERE customer_id = p_customer_id;
END;
/
```

---

## DBMS_SQL による「一度の解析、多数の実行」パターン

異なるバインド値で何度も実行される動的 SQL の場合、解析を一度だけ行い、再実行を繰り返すことでハード・パースを回避します。

```sql
DECLARE
  l_cursor INTEGER;
  l_rows   INTEGER;
BEGIN
  l_cursor := DBMS_SQL.OPEN_CURSOR;
  -- 解析 (一度だけ)
  DBMS_SQL.PARSE(l_cursor, 'INSERT INTO log_tab VALUES (:msg)', DBMS_SQL.NATIVE);

  -- 繰り返し実行
  FOR i IN 1..100 LOOP
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':msg', 'Log message ' || i);
    l_rows := DBMS_SQL.EXECUTE(l_cursor);
  END LOOP;

  DBMS_SQL.CLOSE_CURSOR(l_cursor);
END;
```

---

## ベスト・プラクティス

- DML を一元化するために TAPI を使用する。表の構造が変わっても TAPI だけを更新すればよくなり、呼び出し側への影響を最小限に抑えられます。
- 自律型トランザクションはロギングや監査目的のみに使用し、通常のビジネス・フローの回避策としては使用しない。
- パイプライン・ファンクションでは、呼び出し側が途中で取得を停止した場合に備え、必ず `NO_DATA_NEEDED` ハンドラを記述する。
- 表の構造と 1:1 で対応する場合はレコード型よりも `%ROWTYPE` を優先する。スキーマ変更への耐性が高まります。
- 複数の `REF CURSOR` を返す場合は、どの呼び出し側がカーソルを閉じる責務を持つかを明確にドキュメント化する。

---

## よくある間違い

| 間違い | 問題点 | 対策 |
|---|---|---|
| TAPI 内でコミットする | トランザクションの整合性を損なう | TAPI 内ではコミットせず、呼び出し側に委ねる |
| ビジネス DML に自律型トランザクションを使う | 整合性のない親子トランザクションが発生する | 監査/ログ目的のみに使用する |
| パイプライン・ファンクションで例外ハンドラ不足 | 呼び出し側の早期停止でエラーが発生する | `NO_DATA_NEEDED` をキャッチして正常終了させる |
| 構造が表と等しいのに明示的なレコード型を使う | 列の追加時にコードの修正が必要になる | `%ROWTYPE` を積極的に使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

---

## ソース

- [Oracle Database PL/SQL Language Reference 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/)
- [Oracle Database PL/SQL Language Reference 19c — PIPELINED Clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/CREATE-FUNCTION-statement.html)
- [DBMS_SQL (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQL.html)
