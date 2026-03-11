# OracleにおけるSQLインジェクション対策

## 概要

SQLインジェクションとは、攻撃者がユーザー提供の入力を操作することによって、クエリに悪意のあるSQL断片を挿入するコード・インジェクション手法である。Oracle環境では、アプリケーション・コードと動的PL/SQLの両方でインジェクションが発生する可能性がある。その影響は、不正なデータ・アクセスやデータ操作から、権限昇格、さらにはデータベース全体の完全な侵害にまで及ぶ。

Oracleは、多層的な防御策を提供している。バインド変数（最も重要な保護策）、ホワイトリスト・バリデーションのための `DBMS_ASSERT` パッケージ、慎重な動的SQLの構築、およびアプリケーション・レベルでの入力バリデーションである。本ガイドでは、脆弱なパターンと安全なパターンの具体例を交えながら、各層について解説する。

---

## 根本原因: 文字列の連結

SQLインジェクションは、ユーザー提供の値をバインド変数として渡すのではなく、SQL文字列に直接連結したときに発生する。Oracleはその結果として生成された文字列をSQLとして解析するため、意図した構造と挿入された入力を区別することが不可能になる。

### 脆弱なパターン

```plsql
-- 危険: ユーザー入力がクエリ文字列に直接連結されている
CREATE OR REPLACE PROCEDURE get_employee_unsafe (
  p_name IN VARCHAR2
) AS
  v_sql VARCHAR2(500);
  v_result employees%ROWTYPE;
BEGIN
  -- もし p_name = ' OR 1=1 --' だった場合、すべての行が返される
  -- もし p_name = 'Smith'' UNION SELECT username,password,null,null FROM dba_users--' だった場合、
  -- DBAのユーザー・テーブルが漏洩する
  v_sql := 'SELECT * FROM employees WHERE last_name = ''' || p_name || '''';
  EXECUTE IMMEDIATE v_sql INTO v_result;
END;
/
```

攻撃者が `' OR '1'='1` を渡すと、生成されるSQLは以下のようになる。
```sql
SELECT * FROM employees WHERE last_name = '' OR '1'='1'
```
これにより、テーブル内のすべての行が返されてしまう。

攻撃者が `' UNION SELECT username, password, NULL, NULL FROM dba_users--` を渡すと、生成されるSQLによってDBAの認証情報が漏洩する。

---

## バインド変数: 主要な防御策

バインド変数（バインド・パラメータまたはパラメータ化クエリとも呼ばれる）は、SQLの構造とデータ値を分離する。クエリはプレースホルダを使用して一度だけ解析され、実際の値は実行時に代入される。データベース・エンジンは、挿入されたコンテンツを含めてステートメントを再解析することはない。挿入された文字列はSQL構文としてではなく、リテラル・データ値として扱われる。

### 静的SQLにおけるバインド変数の使用（安全）

```plsql
-- 安全: バインド変数によってインジェクションを防止。実行計画の再利用によりパフォーマンスも向上。
CREATE OR REPLACE PROCEDURE get_employee_safe (
  p_name IN VARCHAR2
) AS
  v_result employees%ROWTYPE;
BEGIN
  SELECT *
  INTO   v_result
  FROM   employees
  WHERE  last_name = p_name;   -- p_name はここでバインド変数として扱われる
  
  DBMS_OUTPUT.PUT_LINE(v_result.first_name || ' ' || v_result.last_name);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('No employee found.');
  WHEN TOO_MANY_ROWS THEN
    DBMS_OUTPUT.PUT_LINE('Multiple employees found.');
END;
/
```

### EXECUTE IMMEDIATE を使用した動的SQLにおけるバインド変数の使用（安全）

```plsql
-- 安全: USING句を伴う EXECUTE IMMEDIATE は、SQL構造からデータを分離してバインドする
CREATE OR REPLACE PROCEDURE get_employee_dynamic_safe (
  p_name IN VARCHAR2
) AS
  v_sql       VARCHAR2(500);
  v_first     employees.first_name%TYPE;
  v_last      employees.last_name%TYPE;
BEGIN
  v_sql := 'SELECT first_name, last_name FROM employees WHERE last_name = :1';
  EXECUTE IMMEDIATE v_sql INTO v_first, v_last USING p_name;

  DBMS_OUTPUT.PUT_LINE(v_first || ' ' || v_last);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('No employee found.');
END;
/
```

### 複数のバインド変数の使用

```plsql
CREATE OR REPLACE PROCEDURE search_employees (
  p_dept_id  IN employees.department_id%TYPE,
  p_min_sal  IN employees.salary%TYPE,
  p_max_sal  IN employees.salary%TYPE
) AS
  TYPE emp_cursor IS REF CURSOR;
  v_cur    emp_cursor;
  v_sql    VARCHAR2(500);
  v_emp    employees%ROWTYPE;
BEGIN
  v_sql := 'SELECT * FROM employees '
        || 'WHERE department_id = :dept '
        || 'AND   salary BETWEEN :min_sal AND :max_sal '
        || 'ORDER BY last_name';

  OPEN v_cur FOR v_sql USING p_dept_id, p_min_sal, p_max_sal;
  LOOP
    FETCH v_cur INTO v_emp;
    EXIT WHEN v_cur%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_emp.last_name || ': ' || v_emp.salary);
  END LOOP;
  CLOSE v_cur;
END;
/
```

---

## DBMS_ASSERT: 動的構造のためのホワイトリスト・バリデーション

バインド変数はデータ値を保護するが、動的SQLの構造要素（表名、列名、スキーマ名、SQLキーワードなど）を保護することはできない。これらはホワイトリストに対してバリデーションを行う必要がある。Oracleの `DBMS_ASSERT` パッケージは、組み込みのバリデータを提供している。

### DBMS_ASSERT の関数

| 関数 | 役割 |
|---|---|
| `DBMS_ASSERT.SQL_OBJECT_NAME(str)` | 文字列が有効で、かつ存在するスキーマ・オブジェクト名であることを確認 |
| `DBMS_ASSERT.SIMPLE_SQL_NAME(str)` | 文字列が単純なSQL識別子のパターン（引用符、スペース、特殊文字なし）に一致することを確認 |
| `DBMS_ASSERT.QUALIFIED_SQL_NAME(str)` | 文字列が有効な修飾名（例: `schema.table`）であることを確認 |
| `DBMS_ASSERT.SCHEMA_NAME(str)` | 文字列が存在するスキーマ名であることを確認 |
| `DBMS_ASSERT.ENQUOTE_NAME(str)` | 文字列を適切に引用符で囲まれた識別子として返す |
| `DBMS_ASSERT.ENQUOTE_LITERAL(str)` | 文字列を適切に引用符で囲まれたSQL文字列リテラルとして返す（単一引用符のエスケープを含む） |
| `DBMS_ASSERT.NOOP(str)` | 文字列をそのまま返す — ドキュメンテーション・マーカーとしてのみ使用され、バリデーションは行わない |

バリデーション関数は失敗時に特定のエラーを発生させる。`SCHEMA_NAME` は `ORA-44001`、`SQL_OBJECT_NAME` は `ORA-44002`、`SIMPLE_SQL_NAME` は `ORA-44003`、`QUALIFIED_SQL_NAME` は `ORA-44004` を発生させる。

### 動的な表名 — 脆弱vs安全

```plsql
-- 危険: 攻撃者は 'employees WHERE 1=1 UNION SELECT ...' のような値を渡すことができる
CREATE OR REPLACE PROCEDURE count_rows_unsafe (
  p_table_name IN VARCHAR2
) AS
  v_count NUMBER;
BEGIN
  EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM ' || p_table_name
    INTO v_count;
  DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
END;
/

-- 安全: DBMS_ASSERT.SQL_OBJECT_NAME は、表が存在しない場合にエラーを発生させ、
-- SQL構文文字を含む文字列を拒否する
CREATE OR REPLACE PROCEDURE count_rows_safe (
  p_table_name IN VARCHAR2
) AS
  v_count      NUMBER;
  v_safe_name  VARCHAR2(128);
BEGIN
  v_safe_name := DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);
  EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM ' || v_safe_name
    INTO v_count;
  DBMS_OUTPUT.PUT_LINE('Count: ' || v_count);
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE IN (-44001, -44002, -44003, -44004) THEN
      RAISE_APPLICATION_ERROR(-20001, 'Invalid table name provided.');
    ELSE
      RAISE;
    END IF;
END;
/
```

### SIMPLE_SQL_NAME による動的な列名の保護

```plsql
-- 安全: 列名が単純な識別子であることをバリデーションする
CREATE OR REPLACE PROCEDURE get_column_value (
  p_table_name  IN VARCHAR2,
  p_column_name IN VARCHAR2,
  p_row_id      IN NUMBER
) AS
  v_sql         VARCHAR2(500);
  v_value       VARCHAR2(4000);
  v_safe_table  VARCHAR2(128);
  v_safe_col    VARCHAR2(128);
BEGIN
  v_safe_table := DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);
  v_safe_col   := DBMS_ASSERT.SIMPLE_SQL_NAME(p_column_name);

  v_sql := 'SELECT TO_CHAR(' || v_safe_col || ') FROM ' || v_safe_table
        || ' WHERE id = :1';

  EXECUTE IMMEDIATE v_sql INTO v_value USING p_row_id;
  DBMS_OUTPUT.PUT_LINE(v_value);
END;
/
```

### 動的SQLでの文字列値に対する ENQUOTE_LITERAL

DDL文内などでバインド変数をどうしても使用できない場合は、`DBMS_ASSERT.ENQUOTE_LITERAL` を使用して文字列値を安全に引用符で囲む。

```plsql
-- 埋め込まれた単一引用符を適切にエスケープし、引用符で囲む
DECLARE
  v_input    VARCHAR2(100) := 'O''Brien';   -- 単一引用符を含む
  v_safe_lit VARCHAR2(200);
  v_sql      VARCHAR2(500);
BEGIN
  v_safe_lit := DBMS_ASSERT.ENQUOTE_LITERAL(v_input);
  -- v_safe_lit は 'O''Brien' となる（適切にエスケープ済み）

  v_sql := 'INSERT INTO audit_log(name) VALUES(' || v_safe_lit || ')';
  EXECUTE IMMEDIATE v_sql;
END;
/
```

---

## 危険なパターンと安全な代替策

### パターン 1: リストからの動的な WHERE 句の構築

```plsql
-- 危険: 連結された入力から IN リストを構築する
-- 入力: '1,2,3' -> WHERE id IN (1,2,3) -- 安全そうに見えるが脆弱
-- 入力: '1,2) OR (1=1' -> WHERE id IN (1,2) OR (1=1) -- インジェクション！
v_sql := 'SELECT * FROM orders WHERE customer_id IN (' || p_id_list || ')';

-- 安全: コレクションと TABLE() 演算子を使用する
CREATE OR REPLACE TYPE number_list AS TABLE OF NUMBER;
/

CREATE OR REPLACE PROCEDURE get_orders_by_ids (
  p_ids IN number_list
) AS
  TYPE order_cur IS REF CURSOR;
  v_cur order_cur;
BEGIN
  OPEN v_cur FOR
    SELECT * FROM orders
    WHERE  customer_id IN (SELECT COLUMN_VALUE FROM TABLE(p_ids));
  -- カーソルを処理...
  CLOSE v_cur;
END;
/
```

### パターン 2: 動的な ORDER BY 列

```plsql
-- 危険: ユーザー入力から ORDER BY を構築
v_sql := 'SELECT * FROM employees ORDER BY ' || p_sort_col;

-- 安全: 既知の列名のホワイトリストと照合する
CREATE OR REPLACE PROCEDURE get_employees_sorted (
  p_sort_col IN VARCHAR2
) AS
  v_safe_col VARCHAR2(30);
  v_sql      VARCHAR2(500);
BEGIN
  -- 明示的なホワイトリスト。リストにないものはすべて拒否する。
  v_safe_col := CASE p_sort_col
                  WHEN 'last_name'   THEN 'last_name'
                  WHEN 'salary'      THEN 'salary'
                  WHEN 'hire_date'   THEN 'hire_date'
                  WHEN 'employee_id' THEN 'employee_id'
                  ELSE NULL
                END;

  IF v_safe_col IS NULL THEN
    RAISE_APPLICATION_ERROR(-20002, 'Invalid sort column: ' || p_sort_col);
  END IF;

  v_sql := 'SELECT * FROM employees ORDER BY ' || v_safe_col;
  EXECUTE IMMEDIATE v_sql;
END;
/
```

### パターン 3: ログイン / 認証クエリ

```plsql
-- 危険: クラシックなログイン回避
-- ユーザー名入力: admin' --
-- 結果: WHERE username = 'admin' --' AND password = '...'
-- パスワード・チェックがコメントアウトされてしまう！
v_sql := 'SELECT COUNT(*) FROM users WHERE username = ''' || p_user
      || ''' AND password = ''' || p_pass || '''';

-- 安全: バインド変数を使用する。挿入された引用符はデータとして扱われる。
SELECT COUNT(*)
INTO   v_count
FROM   users
WHERE  username = p_user
AND    password = STANDARD_HASH(p_pass, 'SHA256');  -- 注意: パスワードを平文で保存しない
```

### パターン 4: DBMS_SQL による動的バインド変数（列数が完全に動的な場合）

```plsql
-- 安全: DBMS_SQL は実行時のバインド変数の動的な数を許容する
-- バインド変数の数が実行時まで確定しない場合に使用する
CREATE OR REPLACE PROCEDURE flexible_search (
  p_conditions IN SYS.ODCIVARCHAR2LIST,  -- '列名=値' のペアの配列
  p_values     IN SYS.ODCIVARCHAR2LIST
) AS
  v_cursor  INTEGER;
  v_sql     VARCHAR2(4000) := 'SELECT employee_id, last_name FROM employees WHERE 1=1';
  v_rows    INTEGER;
BEGIN
  -- バインド・プレースホルダを使用して WHERE 句を構築（決して値を連結しない）
  FOR i IN 1..p_conditions.COUNT LOOP
    -- 追加する前に、列名をホワイトリストでバリデーションする
    v_sql := v_sql || ' AND '
          || DBMS_ASSERT.SIMPLE_SQL_NAME(p_conditions(i))
          || ' = :b' || i;
  END LOOP;

  v_cursor := DBMS_SQL.OPEN_CURSOR;
  DBMS_SQL.PARSE(v_cursor, v_sql, DBMS_SQL.NATIVE);

  FOR i IN 1..p_values.COUNT LOOP
    DBMS_SQL.BIND_VARIABLE(v_cursor, ':b' || i, p_values(i));
  END LOOP;

  v_rows := DBMS_SQL.EXECUTE(v_cursor);
  DBMS_SQL.CLOSE_CURSOR(v_cursor);
END;
/
```

---

## 入力バリデーション層

防御の「深層防護（Defense in Depth）」とは、複数の層でバリデーションを適用することを意味する。バインド変数はデータ値を担当するが、これ以外の要素については以下の手法で対応する。

### アプリケーション層でのバリデーション

- データベースに送信する前にデータ型を検証する（数値入力が実際に数値であることを確認する）。
- 長さ制限を課す。例えば「名字」フィールドに 4,000 文字も受け入れるべきではない。
- ブラックリスト（禁止文字）ではなく、ホワイトリスト（許可文字）を使用する。ブラックリストでは未知の回避策を見落とすリスクがある。
- ヌル・バイト (`chr(0)`)、エスケープ・シーケンス、および Unicode 正規化のトリックを拒否する。

### PL/SQL 層でのバリデーション

```plsql
-- ユーティリティ: 入力が正の整数であることを確認する
CREATE OR REPLACE FUNCTION is_positive_integer (p_input IN VARCHAR2)
RETURN BOOLEAN AS
  v_num NUMBER;
BEGIN
  v_num := TO_NUMBER(p_input);
  RETURN v_num = TRUNC(v_num) AND v_num > 0;
EXCEPTION
  WHEN VALUE_ERROR THEN
    RETURN FALSE;
END;
/

-- ユーティリティ: 英数字以外の文字を削除する（注意して使用 — なるべくホワイトリストを優先）
CREATE OR REPLACE FUNCTION sanitize_alphanumeric (p_input IN VARCHAR2)
RETURN VARCHAR2 AS
BEGIN
  RETURN REGEXP_REPLACE(p_input, '[^A-Za-z0-9 _-]', '');
END;
/
```

### データベース・レベルでの最小限の権限（最小権限の原則）

たとえインジェクションが発生しても、権限が制限されたスキーマであれば被害を最小限に抑えられる。

```sql
-- アプリケーション・スキーマには必要最小限の権限のみを付与する
GRANT SELECT, INSERT, UPDATE ON hr.employees TO app_user;
-- アプリケーション用アカウントに DBA 権限や ANY 特権を付与してはならない

-- アプリケーション接続用には、専用の低権限スキーマを使用する
-- アプリケーション・コードから SYS や SYSTEM として接続してはならない
```

---

## ベストプラクティスのまとめ

- **データ値には常にバインド変数を使用する**。静的SQLと動的SQLの両方で適用すること。これが最も効果的な防御策である。
- **動的識別子（表名/列名）を検証する**。SQL文字列に埋め込む前に、`DBMS_ASSERT` 関数または明示的なホワイトリストを使用する。
- **バリデーションされていない入力からSQLを構築しない**。社内ソース、権限の低いミドルウェアなど、あらゆる入力元を疑うこと。
- **最小権限の原則を適用する**。アプリケーションが使用するアカウントには最低限の権限のみを付与する。特定のテーブルへの `SELECT` しか持たないアカウントであれば、SQLを注入されてもテーブル削除はできない。
- **DBMS_ASSERT の例外をログに記録し、アラートを発する**。アサーション失敗が繰り返される場合、アクティブな調査が行われている強い兆候である。
- **機密性の高いクエリを監査する**。ファイングレイン監査 (FGA) を使用することで、インジェクションの試行を追跡可能にする。
- **アプリケーション・コンテキストに応じた接続プールを使用する**。DBAレベルの認証情報は絶対に使用しない。

---

## よくある間違い

| 間違い | なぜ危険か | 解決策 |
|---|---|---|
| 数値入力を直接連結する | 数値であってもインジェクションが可能: `1 UNION SELECT...` | `TO_NUMBER()` で型を検証し、バインド変数を使用する |
| エスケープ/エンコーディングを信頼する | エンコーディングのトリック（URLエンコード、マルチバイト文字）によって単純なエスケープはバイパスされ得る | バインド変数を使用する。エスケープは代替手段にはならない |
| `DBMS_ASSERT.NOOP` がバリデーションを行うと誤解する | `NOOP` は何もしない。単なるドキュメンテーション・マーカーである | `SIMPLE_SQL_NAME` や `SQL_OBJECT_NAME` を使用する |
| UI 層でのみホワイトリストをチェックする | UI はバイパスできる。API や DB への直接アクセスは UI を介さない | 常に PL/SQL / データベース層でバリデーションを行う |
| ユーザー入力による動的 DDL の実行 | 動的SQLでの `CREATE TABLE`、`DROP TABLE`、`GRANT` は極めて危険 | DDL の構造にユーザー入力を反映させてはならない |
| ソース・コード内への認証情報の保存 | コードが公開された場合、認証情報そのものがインジェクションの足がかりになる | Oracle Wallet や秘匿情報管理ツールを使用する |

---

## Oracle 固有の考慮事項

- Oracle の **ファイングレイン監査 (FGA)** (`DBMS_FGA.ADD_POLICY`) を使用すると、機密性の高い列にアクセスがあったときにアラートを発することができ、予防だけでなく検知も可能になる。
- **仮想プライベート・データベース (VPD)** は、アプリケーション・コンテキストに基づいてクエリに自動的に述語（条件）を追加し、クエリが注入された場合でもデータの露出を制限する。
- **Oracle Label Security** および **Database Vault** は、厳密なデータ分離が必要な環境に追加の制御を提供する。
- `CURSOR_SHARING = FORCE` パラメータは、システム全体でバインド変数の使用を強制できるが、オプティマイザの精度を犠牲にする可能性がある。これは最終手段であり、アプリケーション・コードでの適切なバインド変数使用の代わりにはならない。
- **Oracle SQL Firewall** (21c+) は、データベース・アカウントごとに許可されるSQL文を正確にホワイトリスト化し、学習されたベースラインに一致しないSQLをブロックできる。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ガイドの基本的な指針は、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッド環境では、デフォルトや非推奨がリリース更新によって異なる場合があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference (SQLRF)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [DBMS_ASSERT — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_ASSERT.html)
- [DBMS_SQL — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQL.html)
- [Oracle Database Security Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/)
