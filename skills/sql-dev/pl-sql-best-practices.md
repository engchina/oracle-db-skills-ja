# PL/SQL ベストプラクティス

## 概要

PL/SQLは、OracleによるSQLの手続き型拡張である。データベース・サーバー内部で実行されるため、ネットワークのラウンドトリップを最小限に抑え、OracleのSQLエンジンとの密接な統合を可能にする。しかし、不適切に記述されたPL/SQLは、適切に記述されたものと比べて桁違いに遅くなる可能性がある。最も効果的な手法は、バルク処理（行ごとのコンテキスト・スイッチの排除）、適切な例外処理、規律あるカーソル管理、および構造化されたパッケージ設計である。

本ガイドでは、パフォーマンス、保守性、および信頼性に大きな実用的影響を与えるパターンについて解説する。

---

## コンテキスト・スイッチとその重要性

PL/SQLのパフォーマンスにおいて最も重要な概念は**コンテキスト・スイッチ**である。PL/SQLがSQL文を実行するためにSQLエンジンを呼び出すたびに、PL/SQL仮想マシン（PVM）とSQLエンジン間の切り替えによるオーバーヘッドが発生する。これがループ内で1行ごとに発生する場合（悪名高い「row-by-row」または「slow-by-slow」パターン）、オーバーヘッドは急速に蓄積される。

```plsql
-- 遅い例: 反復ごとにコンテキスト・スイッチが発生
FOR rec IN (SELECT employee_id, salary FROM employees WHERE department_id = 50) LOOP
  UPDATE employees
  SET    salary = rec.salary * 1.1
  WHERE  employee_id = rec.employee_id;   -- 1行ごとに1回のSQL呼び出し
END LOOP;
```

この解決策は、集合ベースのSQL文、またはバルク操作を使用して、処理をSQLエンジン側に押し込むことである。

---

## BULK COLLECT と FORALL

### BULK COLLECT

`BULK COLLECT` は、1回のSQL呼び出しで複数の行をコレクションにフェッチし、フェッチ側の行ごとのコンテキスト・スイッチを排除する。

```plsql
DECLARE
  TYPE emp_id_list   IS TABLE OF employees.employee_id%TYPE;
  TYPE salary_list   IS TABLE OF employees.salary%TYPE;

  v_emp_ids  emp_id_list;
  v_salaries salary_list;
BEGIN
  -- 1回のSQL呼び出しですべての一致する行をフェッチ
  SELECT employee_id, salary
  BULK COLLECT INTO v_emp_ids, v_salaries
  FROM   employees
  WHERE  department_id = 50;

  DBMS_OUTPUT.PUT_LINE('Fetched: ' || v_emp_ids.COUNT || ' employees');
END;
/
```

### LIMIT を使用した Bulk Collect の制限（メモリ管理）

非常に大きな結果セットの場合、一度にすべてをフェッチするとPGAメモリを使い果たす可能性がある。バッチで処理するには、カーソル・ループで `LIMIT` を使用する。

```plsql
DECLARE
  CURSOR emp_cur IS
    SELECT employee_id, salary
    FROM   employees
    WHERE  hire_date < DATE '2015-01-01';

  TYPE emp_rec_list IS TABLE OF emp_cur%ROWTYPE;
  v_batch   emp_rec_list;
  c_limit   CONSTANT PLS_INTEGER := 1000;
  v_total   PLS_INTEGER := 0;
BEGIN
  OPEN emp_cur;
  LOOP
    FETCH emp_cur BULK COLLECT INTO v_batch LIMIT c_limit;
    EXIT WHEN v_batch.COUNT = 0;

    -- 各バッチを処理
    FOR i IN 1..v_batch.COUNT LOOP
      -- ビジネスロジックをここに記述
      NULL;
    END LOOP;

    v_total := v_total + v_batch.COUNT;
    COMMIT;  -- 巨大なUNDOセグメントを避けるためにバッチごとにコミット
  END LOOP;
  CLOSE emp_cur;

  DBMS_OUTPUT.PUT_LINE('Processed: ' || v_total || ' rows');
END;
/
```

### FORALL

`FORALL` は、コレクション内の各要素に対してDML文を1回実行するが、すべてのDMLを一括してSQLエンジンに送信する。操作全体に対してコンテキスト・スイッチは1回のみとなる。

```plsql
DECLARE
  TYPE emp_id_list  IS TABLE OF employees.employee_id%TYPE;
  TYPE salary_list  IS TABLE OF employees.salary%TYPE;

  v_emp_ids   emp_id_list;
  v_new_sals  salary_list;
BEGIN
  -- データをフェッチ
  SELECT employee_id, salary * 1.1
  BULK COLLECT INTO v_emp_ids, v_new_sals
  FROM   employees
  WHERE  department_id = 50;

  -- バルクで更新を適用 — すべての行に対して1回のコンテキスト・スイッチ
  FORALL i IN 1..v_emp_ids.COUNT
    UPDATE employees
    SET    salary = v_new_sals(i)
    WHERE  employee_id = v_emp_ids(i);

  DBMS_OUTPUT.PUT_LINE('Updated: ' || SQL%ROWCOUNT || ' rows');
  COMMIT;
END;
/
```

### SAVE EXCEPTIONS を使用した FORALL

デフォルトでは、`FORALL` は最初のDMLエラーで停止する。`SAVE EXCEPTIONS` を使用すると処理を続行し、すべてのエラーをレビュー用に収集する。

```plsql
DECLARE
  TYPE id_list IS TABLE OF NUMBER;
  v_ids       id_list := id_list(101, 999999, 102, 103);  -- 999999 は存在しない
  e_dml_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_dml_errors, -24381);
BEGIN
  FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
    DELETE FROM employees WHERE employee_id = v_ids(i);

EXCEPTION
  WHEN e_dml_errors THEN
    FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        'Error at index ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX
        || ': ' || SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE)
      );
    END LOOP;
END;
/
```

---

## 例外処理パターン

### 例外には必ず名前を付ける

```plsql
DECLARE
  e_invalid_salary  EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_invalid_salary, -20001);  -- ORA-20001を名前にリンク

  e_emp_not_found   EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_emp_not_found, -20002);
BEGIN
  NULL;
END;
/
```

### 標準的な例外ブロック・パターン

```plsql
CREATE OR REPLACE PROCEDURE update_salary (
  p_employee_id IN  employees.employee_id%TYPE,
  p_new_salary  IN  employees.salary%TYPE,
  p_updated_by  IN  VARCHAR2 DEFAULT SYS_CONTEXT('USERENV','SESSION_USER')
) AS
  v_old_salary  employees.salary%TYPE;
BEGIN
  -- 入力バリデーション
  IF p_new_salary <= 0 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Salary must be positive.');
  END IF;

  -- 監査のために現在の値を取得
  SELECT salary INTO v_old_salary
  FROM   employees
  WHERE  employee_id = p_employee_id
  FOR UPDATE NOWAIT;  -- 即座にロック、不可ならエラー

  -- 変更を適用
  UPDATE employees
  SET    salary = p_new_salary,
         last_updated = SYSDATE
  WHERE  employee_id = p_employee_id;

  -- 監査ログ
  INSERT INTO salary_audit(employee_id, old_salary, new_salary, changed_by, changed_on)
  VALUES (p_employee_id, v_old_salary, p_new_salary, p_updated_by, SYSTIMESTAMP);

  COMMIT;

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RAISE_APPLICATION_ERROR(-20002,
      'Employee ' || p_employee_id || ' not found.');
  WHEN LOCK_TIMEOUT THEN
    RAISE_APPLICATION_ERROR(-20003,
      'Record is locked by another user. Try again.');
  WHEN OTHERS THEN
    ROLLBACK;
    -- 予期しないエラーを再発生させる前にログ出力
    INSERT INTO error_log(proc_name, error_code, error_msg, logged_on)
    VALUES ('update_salary', SQLCODE, SQLERRM, SYSTIMESTAMP);
    COMMIT;  -- ビジネス・トランザクションがロールバックされてもログ・エントリはコミット
    RAISE;   -- 元のエラーを呼び出し元に再発生させる
END;
/
```

### 例外を黙って無視しない

```plsql
-- 悪い例: すべてのエラーを隠蔽し、本番環境の問題診断を不可能にする
EXCEPTION
  WHEN OTHERS THEN
    NULL;

-- 悪い例: OTHERSをキャッチするが DBMS_OUTPUT にのみ出力（本番では失われる）
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);

-- 良い例: テーブルにログ出力した後、再発生させる、または raise_application_error
EXCEPTION
  WHEN OTHERS THEN
    log_error(p_context => 'update_salary', p_sqlcode => SQLCODE, p_sqlerrm => SQLERRM);
    RAISE;
```

### DBMS_UTILITY.FORMAT_ERROR_BACKTRACE の使用

`SQLERRM` はエラー・メッセージのみを報告する。`FORMAT_ERROR_BACKTRACE` は、エラーが発生した正確な行を示すフル・コール・スタックを返すため、ネストされたプロシージャ呼び出しのデバッグに非常に有用である。

```plsql
EXCEPTION
  WHEN OTHERS THEN
    INSERT INTO error_log(proc_name, error_code, error_msg, backtrace, logged_on)
    VALUES (
      'my_procedure',
      SQLCODE,
      SQLERRM,
      DBMS_UTILITY.FORMAT_ERROR_BACKTRACE,  -- 行番号を表示
      SYSTIMESTAMP
    );
    COMMIT;
    RAISE;
END;
/
```

---

## カーソル管理

### 暗黙的カーソル vs. 明示的カーソル

```plsql
-- 暗黙的カーソル（単一行SELECT INTO）: 最も単純な形式
-- Oracleが自動的にオープン、フェッチ、クローズを行う
DECLARE
  v_name VARCHAR2(100);
BEGIN
  SELECT last_name INTO v_name FROM employees WHERE employee_id = 100;
  DBMS_OUTPUT.PUT_LINE(v_name);
EXCEPTION
  WHEN NO_DATA_FOUND THEN DBMS_OUTPUT.PUT_LINE('Not found');
  WHEN TOO_MANY_ROWS THEN DBMS_OUTPUT.PUT_LINE('Multiple rows returned');
END;
/

-- 明示的カーソル（複数行）: オープン/フェッチ/クローズを完全に制御
DECLARE
  CURSOR dept_cur IS
    SELECT department_id, department_name FROM departments ORDER BY department_name;
  v_rec dept_cur%ROWTYPE;
BEGIN
  OPEN dept_cur;
  LOOP
    FETCH dept_cur INTO v_rec;
    EXIT WHEN dept_cur%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_rec.department_id || ': ' || v_rec.department_name);
  END LOOP;
  CLOSE dept_cur;  -- 明示的にオープンしたカーソルは常にクローズする
END;
/
```

### カーソル FOR ループ（簡潔さのため推奨）

カーソル FOR ループは、カーソルのオープン、フェッチ、およびクローズを自動的に行う。バルク操作を行わない場合に使用する。

```plsql
BEGIN
  FOR rec IN (SELECT department_id, department_name FROM departments ORDER BY department_name) LOOP
    DBMS_OUTPUT.PUT_LINE(rec.department_id || ': ' || rec.department_name);
  END LOOP;
  -- カーソルはここで自動的にクローズされる
END;
/
```

### 明示的にオープンしたカーソルは必ずクローズする

```plsql
-- パターン: カーソルの確実なクローズを保証するために例外処理を伴うネストされたブロックを使用
DECLARE
  v_cur SYS_REFCURSOR;
BEGIN
  BEGIN
    OPEN v_cur FOR SELECT * FROM employees WHERE department_id = 50;
    -- 処理...
    CLOSE v_cur;
  EXCEPTION
    WHEN OTHERS THEN
      IF v_cur%ISOPEN THEN
        CLOSE v_cur;
      END IF;
      RAISE;
  END;
END;
/
```

### パラメータ付カーソル

```plsql
DECLARE
  CURSOR emp_by_dept(p_dept_id NUMBER) IS
    SELECT employee_id, last_name, salary
    FROM   employees
    WHERE  department_id = p_dept_id
    ORDER BY salary DESC;
BEGIN
  FOR rec IN emp_by_dept(50) LOOP
    DBMS_OUTPUT.PUT_LINE(rec.last_name || ': ' || rec.salary);
  END LOOP;
END;
/
```

---

## パッケージ構造

パッケージはPL/SQLにおける構成の基本単位である。カプセル化、状態管理、および大幅なパフォーマンス上の利点（初回使用時にパッケージ全体が共有プールにロードされる）を提供する。

### 推奨されるパッケージ・レイアウト

```plsql
-- 仕様部（パブリック・インターフェース — 規約）
CREATE OR REPLACE PACKAGE emp_mgmt AS

  -- パブリック型
  TYPE emp_salary_rec IS RECORD (
    employee_id  employees.employee_id%TYPE,
    last_name    employees.last_name%TYPE,
    salary       employees.salary%TYPE
  );
  TYPE emp_salary_tab IS TABLE OF emp_salary_rec;

  -- パブリック定数
  c_max_salary_increase CONSTANT NUMBER := 0.30;  -- 30%

  -- パブリック・プロシージャ/ファンクションのシグネチャ
  PROCEDURE update_salary (
    p_employee_id IN  employees.employee_id%TYPE,
    p_new_salary  IN  employees.salary%TYPE
  );

  FUNCTION get_department_payroll (
    p_dept_id IN departments.department_id%TYPE
  ) RETURN NUMBER;

  FUNCTION get_high_earners (
    p_threshold IN NUMBER
  ) RETURN emp_salary_tab PIPELINED;

END emp_mgmt;
/

-- 本体部（実装 — 仕様部のシグネチャが変わらない限り、
--         依存オブジェクトを再コンパイルせずに変更可能）
CREATE OR REPLACE PACKAGE BODY emp_mgmt AS

  -- プライベート定数（パッケージ外からは見えない）
  c_min_salary CONSTANT NUMBER := 2000;

  -- プライベート・ヘルパー（仕様部にはない — 内部使用のみ）
  PROCEDURE validate_salary (p_salary IN NUMBER) AS
  BEGIN
    IF p_salary < c_min_salary THEN
      RAISE_APPLICATION_ERROR(-20010, 'Salary below minimum: ' || c_min_salary);
    END IF;
    IF p_salary > 999999 THEN
      RAISE_APPLICATION_ERROR(-20011, 'Salary exceeds maximum.');
    END IF;
  END validate_salary;

  PROCEDURE update_salary (
    p_employee_id IN employees.employee_id%TYPE,
    p_new_salary  IN employees.salary%TYPE
  ) AS
  BEGIN
    validate_salary(p_new_salary);
    UPDATE employees SET salary = p_new_salary WHERE employee_id = p_employee_id;
    IF SQL%ROWCOUNT = 0 THEN
      RAISE_APPLICATION_ERROR(-20002, 'Employee not found: ' || p_employee_id);
    END IF;
  END update_salary;

  FUNCTION get_department_payroll (
    p_dept_id IN departments.department_id%TYPE
  ) RETURN NUMBER AS
    v_total NUMBER;
  BEGIN
    SELECT NVL(SUM(salary), 0)
    INTO   v_total
    FROM   employees
    WHERE  department_id = p_dept_id;
    RETURN v_total;
  END get_department_payroll;

  -- パイプライン・ファンクション: 1行ずつ返し、ストリーミングを可能にする
  FUNCTION get_high_earners (
    p_threshold IN NUMBER
  ) RETURN emp_salary_tab PIPELINED AS
  BEGIN
    FOR rec IN (
      SELECT employee_id, last_name, salary
      FROM   employees
      WHERE  salary > p_threshold
      ORDER BY salary DESC
    ) LOOP
      PIPE ROW(emp_salary_rec(rec.employee_id, rec.last_name, rec.salary));
    END LOOP;
  END get_high_earners;

END emp_mgmt;
/
```

### セッション状態のためのパッケージ・レベル変数

```plsql
CREATE OR REPLACE PACKAGE session_context AS
  -- パッケージ変数はセッションの間保持される
  g_current_user    VARCHAR2(100);
  g_audit_enabled   BOOLEAN := TRUE;

  PROCEDURE initialize (p_user IN VARCHAR2);
  FUNCTION  is_audit_enabled RETURN BOOLEAN;
END session_context;
/

CREATE OR REPLACE PACKAGE BODY session_context AS
  PROCEDURE initialize (p_user IN VARCHAR2) AS
  BEGIN
    g_current_user  := p_user;
    g_audit_enabled := TRUE;
  END initialize;

  FUNCTION is_audit_enabled RETURN BOOLEAN AS
  BEGIN
    RETURN g_audit_enabled;
  END is_audit_enabled;
END session_context;
/
```

**注意:** パッケージ・レベル変数はセッション固有であり、セッション間で共有されない。接続プールでは、再利用された接続での新しい呼び出しにおいて、前のユーザーの古いパッケージ状態が見えてしまう可能性がある。論理的なトランザクションごとに、必ずパッケージ状態を初期化すること。

---

## NOCOPY ヒント

デフォルトでは、PL/SQLは `IN OUT` および `OUT` パラメータを値渡し（コピーが作成される）する。大きなコレクションやCLOBの場合、このコピーはコストが高くなる。`NOCOPY` は、代わりに参照渡しを行うようコンパイラに指示する。

```plsql
-- NOCOPYなし: 大きなコレクションが呼び出し時と復帰時にコピーされる
CREATE OR REPLACE PROCEDURE process_data_slow (
  p_data IN OUT large_collection_type
) AS
BEGIN
  NULL;
END;
/

-- NOCOPYあり: 参照渡し、コピーのオーバーヘッドなし
CREATE OR REPLACE PROCEDURE process_data_fast (
  p_data IN OUT NOCOPY large_collection_type
) AS
BEGIN
  NULL;
END;
/
```

**使い分け:** `NOCOPY` を使用すると、プロシージャ内で例外が発生した場合、コピーが破棄されないため、パラメータへの変更が呼び出し元に見えたままになる。この動作を受け入れてでもパフォーマンス向上の価値がある場合にのみ `NOCOPY` を使用すること — 通常、処理ルーチンに渡される読み取り専用の大きなコレクションに適している。

---

## コンテキスト・スイッチの回避: 集合ベース vs. 行ごと

ロジックをSQLで表現できる場合は、手続き型のループよりも単一のSQL文を常に優先すること。

```plsql
-- 遅い例: 行ごとの UPDATE を伴う PL/SQL ループ
BEGIN
  FOR rec IN (SELECT employee_id FROM employees WHERE department_id = 50) LOOP
    UPDATE employees
    SET    salary = salary * 1.1
    WHERE  employee_id = rec.employee_id;
  END LOOP;
  COMMIT;
END;
/

-- 速い例: 単一の SQL 文 — コンテキスト・スイッチがゼロ
BEGIN
  UPDATE employees
  SET    salary = salary * 1.1
  WHERE  department_id = 50;
  COMMIT;
END;
/

-- 速い例（行ごとのロジックが避けられない場合）: BULK COLLECT + FORALL
DECLARE
  TYPE id_list IS TABLE OF employees.employee_id%TYPE;
  v_ids id_list;
BEGIN
  SELECT employee_id BULK COLLECT INTO v_ids
  FROM   employees WHERE department_id = 50;

  FORALL i IN 1..v_ids.COUNT
    UPDATE employees SET salary = salary * 1.1 WHERE employee_id = v_ids(i);
  COMMIT;
END;
/
```

---

## ログ記録パターン

### エラー・ログ・テーブル

```sql
CREATE TABLE error_log (
  log_id       NUMBER GENERATED ALWAYS AS IDENTITY,
  log_time     TIMESTAMP DEFAULT SYSTIMESTAMP,
  username     VARCHAR2(128) DEFAULT SYS_CONTEXT('USERENV','SESSION_USER'),
  program      VARCHAR2(256),
  error_code   NUMBER,
  error_msg    VARCHAR2(4000),
  backtrace    CLOB,
  call_stack   CLOB,
  extra_info   VARCHAR2(4000)
);
```

### ログ記録パッケージ

```plsql
CREATE OR REPLACE PACKAGE app_logger AS
  PROCEDURE log_error (
    p_program    IN VARCHAR2,
    p_extra_info IN VARCHAR2 DEFAULT NULL
  );

  PROCEDURE log_info (
    p_program IN VARCHAR2,
    p_message IN VARCHAR2
  );
END app_logger;
/

CREATE OR REPLACE PACKAGE BODY app_logger AS

  -- 自律型トランザクションを使用。呼び出し元のトランザクションが
  -- ロールバックされてもログはコミットされる。
  PROCEDURE log_error (
    p_program    IN VARCHAR2,
    p_extra_info IN VARCHAR2 DEFAULT NULL
  ) AS
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    INSERT INTO error_log (program, error_code, error_msg, backtrace, call_stack, extra_info)
    VALUES (
      p_program,
      SQLCODE,
      SQLERRM,
      DBMS_UTILITY.FORMAT_ERROR_BACKTRACE,
      DBMS_UTILITY.FORMAT_CALL_STACK,
      p_extra_info
    );
    COMMIT;
  END log_error;

  PROCEDURE log_info (
    p_program IN VARCHAR2,
    p_message IN VARCHAR2
  ) AS
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    INSERT INTO error_log (program, error_code, error_msg)
    VALUES (p_program, 0, p_message);
    COMMIT;
  END log_info;

END app_logger;
/
```

**例外ハンドラでの使用例:**

```plsql
EXCEPTION
  WHEN OTHERS THEN
    app_logger.log_error(
      p_program    => 'emp_mgmt.update_salary',
      p_extra_info => 'employee_id=' || p_employee_id
    );
    RAISE;
END;
```

---

## ベストプラクティスのまとめ

- **BULK COLLECT + FORALL を使用する。** 数百行以上を処理するループに適用。
- **BULK COLLECT に制限を設ける。** PGAの使用量を制御するため、大きなテーブルにはバッチ・サイズ（1000〜10000）を使用。
- **例外は必ず再発生または変換する。** 決して黙って無視してはならない。
- **FORMAT_ERROR_BACKTRACE を含める。** エラー・ログには SQLERRM だけでなく、バックトレースも含める。
- **すべてのカーソルを閉じる。** 明示的にオープンしたカーソルは、例外パスであっても必ず閉じる。
- **コードをパッケージにまとめる。** 大規模システムでは、スタンドアロンのプロシージャやファンクションをデプロイしてはならない。
- **パッケージ仕様部は安定させる。** 仕様部の変更はすべての依存オブジェクトを無効化する。本体部は自由に変更可能。
- **NOCOPY を使用する。** 例外が問題にならない読み取り専用の大きなコレクション型パラメータに適用。
- **自律型トランザクションでログ出力する。** ログ・エントリがロールバックに耐えられるようにする。
- **パッケージ・レベルの状態を初期化する。** 接続プール環境では、各リクエストの開始時に初期化を行う。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| ループ内での行ごとの DML | 数千のコンテキスト・スイッチが発生し、非常に遅くなる | 集合ベースの UPDATE/DELETE、または BULK COLLECT + FORALL を使用する |
| `WHEN OTHERS THEN NULL` | すべてのエラーを黙って破棄する | 常にログ出力して再発生させる |
| 明示的カーソルのクローズ忘れ | オープン・カーソル・リークが発生し、最終的に `ORA-01000: 最大オープン・カーソル数を超えました` に達する | カーソル FOR ループを使用するか、例外ハンドラでクローズする |
| エラー・ログに `DBMS_OUTPUT.PUT_LINE` を使用 | 出力がバッファリングされ、本番環境では失われる。非同期ジョブでは表示されない | 自律型トランザクションを使用してエラー・テーブルにログを出力する |
| `LIMIT` なしの大きな `BULK COLLECT` | 数百万行のテーブルで PGA が枯渇する | フェッチ・ループ内で常に `LIMIT c_limit` を使用する |
| 接続プールでのパッケージ状態の放置 | セッション変数が前のユーザーのセッションの値を保持してしまう | リクエスト開始時に初期化呼び出しを通じてパッケージ状態を初期化する |
| ビジネス・ロジックに `PRAGMA AUTONOMOUS_TRANSACTION` を使用 | 呼び出し元のトランザクションから変更が見えなくなり、ロールバックが不可能になる | 自律型トランザクションはログ記録/監査のみに使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ガイドの基本的な指針は、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッド環境では、デフォルトや非推奨がリリース更新によって異なる場合があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c PL/SQL Language Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/)
- [Oracle Database 19c SQL Tuning Guide (TGSQL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [Oracle Database 19c PL/SQL Packages and Types Reference (ARPLS)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/)
