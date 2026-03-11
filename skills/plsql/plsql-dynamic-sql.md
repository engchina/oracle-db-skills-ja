# PL/SQL 動的 SQL (Dynamic SQL)

## 概要

動的 SQL は、コンパイル時ではなく実行時に SQL 文を構築して実行します。非常に強力ですが、SQL インジェクションのリスク、ハード・パースのオーバーヘッド、コンパイラによるエラー検出の減少といったリスクも伴います。このガイドでは、動的 SQL が正当化されるケース、2 つの実行アプローチ（`EXECUTE IMMEDIATE` と `DBMS_SQL`）、インジェクションの防止、およびパフォーマンスに関する考慮事項について説明します。

---

## 動的 SQL が正当化されるケースと回避すべきケース

### 回避すべきケース — 代わりに静的 SQL を使用する

```sql
-- 回避すべき例: 条件が常に分かっている場合に動的な WHERE 句を使用する
-- (「貧乏人の動的 SQL」)
PROCEDURE get_employees(p_dept_id IN NUMBER) IS
  l_sql VARCHAR2(500);
  l_cur SYS_REFCURSOR;
BEGIN
  l_sql := 'SELECT * FROM employees WHERE department_id = ' || p_dept_id;
  OPEN l_cur FOR l_sql;  -- バインド変数を使用していない！
  -- ...
END;

-- より良い例: バインド変数を使用した静的 SQL
PROCEDURE get_employees(p_dept_id IN NUMBER) IS
BEGIN
  FOR emp IN (SELECT * FROM employees WHERE department_id = p_dept_id) LOOP
    -- ...
  END LOOP;
END;
```

### 正当なユースケース

| シナリオ | 動的 SQL が必要な理由 |
|---|---|
| DDL の実行 (CREATE, ALTER, DROP) | DDL は静的 PL/SQL として記述できない |
| 表名や列名をパラメータにする | 構造的な SQL 要素はバインド変数にできない |
| 実行時にスキーマ名を決定する | オブジェクトの解決に実行時のスキーマ名が必要 |
| SELECT で条件によって列を含める | 列リストを入力に基づいて変更する |
| メタデータから SQL を構築する | コンパイル時にクエリ構造が不明 |
| TRUNCATE TABLE | DDL であり、静的には記述できない |
| 任意の SQL を実行する必要がある | 汎用的なレポート作成ツールやクエリ・ツール |

---

## EXECUTE IMMEDIATE

`EXECUTE IMMEDIATE` は、動的 SQL の主要な機能です。DDL、DML、および単一行のクエリを処理します。

### DDL の実行

```sql
-- DDL は静的 PL/SQL に埋め込むことができないため、
-- EXECUTE IMMEDIATE を使用する必要がある
PROCEDURE create_archive_table(p_year IN NUMBER) IS
  l_table_name VARCHAR2(50);
BEGIN
  -- 使用前に検証 (インジェクション防止)
  IF p_year NOT BETWEEN 2000 AND 2099 THEN
    RAISE_APPLICATION_ERROR(-20001, '無効な年度です: ' || p_year);
  END IF;

  l_table_name := 'ORDERS_ARCHIVE_' || p_year;  -- 年度（数値）なので安全

  EXECUTE IMMEDIATE
    'CREATE TABLE ' || l_table_name || ' AS SELECT * FROM orders WHERE 1=0';

  DBMS_OUTPUT.PUT_LINE('作成されました: ' || l_table_name);
END create_archive_table;
/

-- TRUNCATE: DDL のみ。バインド変数は使用不可 (TRUNCATE には不要)
PROCEDURE truncate_staging(p_table_name IN VARCHAR2) IS
  l_safe_name VARCHAR2(128);
BEGIN
  l_safe_name := DBMS_ASSERT.SIMPLE_SQL_NAME(p_table_name);  -- 検証！
  EXECUTE IMMEDIATE 'TRUNCATE TABLE ' || l_safe_name;
END truncate_staging;
/
```

### バインド変数を使用した DML

```sql
-- 単一のバインド変数
PROCEDURE deactivate_old_sessions(p_days IN NUMBER) IS
BEGIN
  EXECUTE IMMEDIATE
    'DELETE FROM user_sessions WHERE last_activity < SYSDATE - :1'
    USING p_days;  -- バインド変数。インジェクションのリスクなし

  DBMS_OUTPUT.PUT_LINE('削除数: ' || SQL%ROWCOUNT);
END deactivate_old_sessions;
/

-- 名前付きバインド変数 (複数のバインドがある場合に読みやすい)
PROCEDURE update_employee_salary(
  p_employee_id IN NUMBER,
  p_new_salary  IN NUMBER,
  p_reason      IN VARCHAR2
) IS
BEGIN
  EXECUTE IMMEDIATE
    'UPDATE employees
     SET    salary = :new_sal,
            updated_at = SYSDATE,
            update_reason = :reason
     WHERE  employee_id = :emp_id'
    USING p_new_salary, p_reason, p_employee_id;
    -- USING 内の名前付きバインドは、最初に出現する順序と一致させる必要がある

  IF SQL%ROWCOUNT = 0 THEN
    RAISE_APPLICATION_ERROR(-20001, '従業員が見つかりません: ' || p_employee_id);
  END IF;
END update_employee_salary;
/
```

### EXECUTE IMMEDIATE による単一行クエリ

```sql
-- EXECUTE IMMEDIATE による SELECT INTO
DECLARE
  l_salary employees.salary%TYPE;
  l_name   employees.last_name%TYPE;
BEGIN
  EXECUTE IMMEDIATE
    'SELECT salary, last_name FROM employees WHERE employee_id = :1'
    INTO l_salary, l_name
    USING 100;

  DBMS_OUTPUT.PUT_LINE(l_name || ': ' || l_salary);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('見つかりません');
  WHEN TOO_MANY_ROWS THEN
    DBMS_OUTPUT.PUT_LINE('複数の行が返されました');
END;
/
```

### OUT バインド変数 (DML RETURNING 用)

```sql
DECLARE
  l_new_id    NUMBER;
  l_created   DATE;
BEGIN
  EXECUTE IMMEDIATE
    'INSERT INTO orders (customer_id, status) VALUES (:1, :2)
     RETURNING order_id, created_at INTO :3, :4'
    USING 12345, 'PENDING'
    RETURNING INTO l_new_id, l_created;

  DBMS_OUTPUT.PUT_LINE('注文作成: ' || l_new_id || ' (' || l_created || ')');
END;
/
```

---

## SYS_REFCURSOR による複数行クエリ

複数の行を返す動的クエリの場合は、`OPEN ... FOR` 構文の `EXECUTE IMMEDIATE` で `SYS_REFCURSOR` を開きます。

```sql
-- 動的な複数行クエリ
PROCEDURE report_by_status(
  p_status IN VARCHAR2,
  p_cursor OUT SYS_REFCURSOR
) IS
BEGIN
  -- p_status はバインド変数として安全に処理される
  OPEN p_cursor FOR
    'SELECT order_id, customer_id, total_amount, created_at
     FROM   orders
     WHERE  status = :1
     ORDER BY created_at DESC'
    USING p_status;
  -- 呼び出し側が p_cursor からフェッチし、クローズする必要がある
END report_by_status;
/

-- オプションのフィルタがある動的クエリ (SQL 構築を慎重に行う)
PROCEDURE get_orders_filtered(
  p_status    IN VARCHAR2  DEFAULT NULL,
  p_from_date IN DATE      DEFAULT NULL,
  p_to_date   IN DATE      DEFAULT NULL,
  p_cursor    OUT SYS_REFCURSOR
) IS
  l_sql    VARCHAR2(2000) := 'SELECT * FROM orders WHERE 1=1';
  l_status VARCHAR2(20);
  l_from   DATE;
  l_to     DATE;
BEGIN
  -- 条件に応じて SQL を構築し、どのバインドがアクティブかを追跡する
  IF p_status IS NOT NULL THEN
    l_sql    := l_sql || ' AND status = :status';
    l_status := p_status;  -- 下でバインドされる
  END IF;

  IF p_from_date IS NOT NULL THEN
    l_sql   := l_sql || ' AND created_at >= :from_date';
    l_from  := p_from_date;
  END IF;

  IF p_to_date IS NOT NULL THEN
    l_sql  := l_sql || ' AND created_at <= :to_date';
    l_to   := p_to_date;
  END IF;

  l_sql := l_sql || ' ORDER BY created_at DESC';

  -- カーソルのオープン: USING 句は上記で追加されたバインド変数と一致させる必要がある
  -- これは壊れやすいため、バインド数が変わる場合は DBMS_SQL の検討を推奨
  IF p_status IS NOT NULL AND p_from_date IS NOT NULL AND p_to_date IS NOT NULL THEN
    OPEN p_cursor FOR l_sql USING l_status, l_from, l_to;
  ELSIF p_status IS NOT NULL AND p_from_date IS NOT NULL THEN
    OPEN p_cursor FOR l_sql USING l_status, l_from;
  ELSIF p_status IS NOT NULL THEN
    OPEN p_cursor FOR l_sql USING l_status;
  ELSE
    OPEN p_cursor FOR l_sql;
  END IF;
END get_orders_filtered;
/
```

上記のような `USING` 句が複雑になる組み合わせの問題を解決するには、`DBMS_SQL` が適しています。

---

## 可変バインド数と不明な列構造のための DBMS_SQL

`DBMS_SQL` は、より低レベルな動的 SQL API です。記述は冗長になりますが、`EXECUTE IMMEDIATE` では処理できないシナリオに対応できます。
1. コンパイル時にバインド変数の数が不明な場合
2. コンパイル時に SELECT 列の数や型が不明な場合
3. パフォーマンス向上のための「一度パースして何度も実行」パターン

### 可変バインド数 (組み合わせ問題の解決)

```sql
CREATE OR REPLACE PROCEDURE get_orders_flexible(
  p_filters IN SYS.ODCIVARCHAR2LIST,  -- 'column=value' 形式の文字列リスト
  p_cursor  OUT SYS_REFCURSOR
) IS
  l_sql      VARCHAR2(4000) := 'SELECT * FROM orders WHERE 1=1';
  l_cur      INTEGER;
  l_rc       INTEGER;
BEGIN
  -- 名前付きバインドを使用して動的に SQL を構築
  FOR i IN 1..p_filters.COUNT LOOP
    l_sql := l_sql || ' AND ' || p_filters(i);
    -- 重要: p_filters の内容は直接ユーザー入力ではなくホワイトリスト形式にすること
  END LOOP;

  l_cur := DBMS_SQL.OPEN_CURSOR;

  DBMS_SQL.PARSE(l_cur, l_sql, DBMS_SQL.NATIVE);

  -- 各変数をバインド
  FOR i IN 1..p_filters.COUNT LOOP
    DBMS_SQL.BIND_VARIABLE(l_cur, ':val' || i, 'some_value_' || i);
  END LOOP;

  -- REF CURSOR に変換
  l_rc      := DBMS_SQL.EXECUTE(l_cur);
  p_cursor  := DBMS_SQL.TO_REFCURSOR(l_cur);
  -- 注意: TO_REFCURSOR の後は DBMS_SQL はカーソルを管理しなくなる
  -- REF CURSOR の呼び出し側が CLOSE する責任を持つ

EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.IS_OPEN(l_cur) THEN
      DBMS_SQL.CLOSE_CURSOR(l_cur);
    END IF;
    RAISE;
END get_orders_flexible;
/
```

---

## DBMS_SQL.DESCRIBE_COLUMNS

クエリ結果の列構造がコンパイル時に不明な場合（汎用レポート作成、メタデータ駆動ツールなど）に使用します。

```sql
CREATE OR REPLACE PROCEDURE describe_query_result(p_sql IN VARCHAR2) IS
  l_cursor      INTEGER;
  l_col_count   INTEGER;
  l_col_descs   DBMS_SQL.DESC_TAB;
  l_value       VARCHAR2(4000);
BEGIN
  l_cursor := DBMS_SQL.OPEN_CURSOR;

  DBMS_SQL.PARSE(l_cursor, p_sql, DBMS_SQL.NATIVE);

  -- 列の記述: カウントとメタデータを取得
  DBMS_SQL.DESCRIBE_COLUMNS(l_cursor, l_col_count, l_col_descs);

  -- フェッチのための各列の定義 (EXECUTE 前に行う必要がある)
  FOR i IN 1..l_col_count LOOP
    -- 汎用的な表示のため、すべての列を VARCHAR2 として定義
    DBMS_SQL.DEFINE_COLUMN(l_cursor, i, l_value, 4000);
    DBMS_OUTPUT.PUT_LINE(
      '列 ' || i || ': ' ||
      l_col_descs(i).col_name || ' (' ||
      CASE l_col_descs(i).col_type
        WHEN 1   THEN 'VARCHAR2'
        WHEN 2   THEN 'NUMBER'
        WHEN 12  THEN 'DATE'
        WHEN 112 THEN 'CLOB'
        ELSE 'OTHER(' || l_col_descs(i).col_type || ')'
      END || ')'
    );
  END LOOP;

  -- 実行とフェッチ
  DBMS_SQL.EXECUTE(l_cursor);

  WHILE DBMS_SQL.FETCH_ROWS(l_cursor) > 0 LOOP
    FOR i IN 1..l_col_count LOOP
      DBMS_SQL.COLUMN_VALUE(l_cursor, i, l_value);
      DBMS_OUTPUT.PUT(l_col_descs(i).col_name || '=' || l_value || '  ');
    END LOOP;
    DBMS_OUTPUT.NEW_LINE;
  END LOOP;

  DBMS_SQL.CLOSE_CURSOR(l_cursor);
EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.IS_OPEN(l_cursor) THEN DBMS_SQL.CLOSE_CURSOR(l_cursor); END IF;
    RAISE;
END describe_query_result;
/
```

---

## DBMS_SQL による一度のパースと複数回の実行

同じパラメータ化された SQL をバッチ内で何度も実行する場合、一度パースしてから、各行に対して再バインドと実行を行います。

```sql
CREATE OR REPLACE PROCEDURE load_employees_batch(
  p_employees IN emp_staging_collection_t
) IS
  l_cursor   INTEGER;
  l_rows     INTEGER;

  c_sql CONSTANT VARCHAR2(500) :=
    'INSERT INTO employees (first_name, last_name, email, hire_date, job_id, salary)
     VALUES (:fname, :lname, :email, :hdate, :jobid, :sal)';
BEGIN
  l_cursor := DBMS_SQL.OPEN_CURSOR;

  -- 一度のパース: 構文チェックとコンパイルはここでのみ行われる
  DBMS_SQL.PARSE(l_cursor, c_sql, DBMS_SQL.NATIVE);

  -- 複数回の実行: 再パースなしで再バインドと実行を繰り返す
  FOR i IN 1..p_employees.COUNT LOOP
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':fname', p_employees(i).first_name);
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':lname', p_employees(i).last_name);
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':email', p_employees(i).email);
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':hdate', p_employees(i).hire_date);
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':jobid', p_employees(i).job_id);
    DBMS_SQL.BIND_VARIABLE(l_cursor, ':sal',   p_employees(i).salary);

    l_rows := DBMS_SQL.EXECUTE(l_cursor);
  END LOOP;

  DBMS_SQL.CLOSE_CURSOR(l_cursor);
  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.IS_OPEN(l_cursor) THEN DBMS_SQL.CLOSE_CURSOR(l_cursor); END IF;
    RAISE;
END load_employees_batch;
/
```

**注意**: ほとんどの本番バッチ・シナリオでは、コレクションを使用した静的 SQL の `FORALL` の方が、DBMS_SQL の一度パースするパターンよりも高速でシンプルです。SQL 構造自体が動的である必要がある場合にのみ、DBMS_SQL の一度パースするパターンを使用してください。

---

## 動的 SQL におけるインジェクション対策

### 3 つのルール

1. **データ値**: 常にバインド変数 (`:1`, `:name`) を使用し、直接連結しないでください。
2. **オブジェクト名** (表、列、スキーマ): 連結する前に `DBMS_ASSERT` で検証してください。
3. **SQL キーワード**: ホワイトリストに対して検証してください。ユーザー入力の生の値を SQL キーワードとして渡さないでください。

```sql
CREATE OR REPLACE PROCEDURE safe_dynamic_query(
  p_table_name  IN VARCHAR2,  -- 構造要素: 検証が必要
  p_sort_column IN VARCHAR2,  -- 構造要素: 検証が必要
  p_sort_dir    IN VARCHAR2,  -- キーワード: ホワイトリスト化が必要
  p_filter_val  IN VARCHAR2,  -- データ: バインド変数を使用
  p_cursor      OUT SYS_REFCURSOR
) IS
  l_table  VARCHAR2(128);
  l_col    VARCHAR2(128);
  l_dir    VARCHAR2(4);
  l_sql    VARCHAR2(1000);
BEGIN
  -- ルール 2: DBMS_ASSERT で構造要素を検証
  l_table := DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);  -- 存在確認も含む
  l_col   := DBMS_ASSERT.SIMPLE_SQL_NAME(p_sort_column);

  -- ルール 3: SQL キーワードをホワイトリストでチェック
  IF UPPER(p_sort_dir) NOT IN ('ASC', 'DESC') THEN
    RAISE_APPLICATION_ERROR(-20001, '無効なソート方向です: ' || p_sort_dir);
  END IF;
  l_dir := UPPER(p_sort_dir);

  -- ルール 1: データ値にはバインド変数を使用
  l_sql :=
    'SELECT * FROM ' || l_table ||
    ' WHERE last_name LIKE :filter_val' ||  -- データ値用にバインド変数
    ' ORDER BY ' || l_col || ' ' || l_dir;  -- 安全: 上記で検証済み

  OPEN p_cursor FOR l_sql USING p_filter_val || '%';
END safe_dynamic_query;
/
```

### 列のホワイトリスト・パターン

セキュリティを最大化するには、既知の安全な値のホワイトリストと照らし合わせて検証します。

```sql
FUNCTION is_valid_sort_column(
  p_table  IN VARCHAR2,
  p_column IN VARCHAR2
) RETURN BOOLEAN IS
  l_count PLS_INTEGER;
BEGIN
  -- 列が実際に表内に存在するか確認
  SELECT COUNT(*) INTO l_count
  FROM   all_tab_columns
  WHERE  owner      = SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA')
    AND  table_name = UPPER(p_table)
    AND  column_name = UPPER(p_column);
  RETURN l_count > 0;
END is_valid_sort_column;
/
```

---

## パフォーマンスに関する考慮事項

### ハード・パースのコスト

一意の SQL 文字列ごとに、構文チェック、セキュリティ・チェック、実行計画の作成というハード・パースが必要です。ハード・パースは CPU 負荷が高く、実行をシリアル化するラッチを必要とします。共有プール・キャッシュにパース済みカーソルを保存し、再利用することがスケーラビリティにとって重要です。

```sql
-- 回避すべき例: 呼び出しごとに一意の SQL = 毎回ハード・パース
EXECUTE IMMEDIATE 'SELECT * FROM orders WHERE order_id = ' || p_id;
-- 各 p_id の値ごとに異なる SQL 文字列 = 異なるカーソル = ハード・パースが発生

-- 推奨例: バインド変数 = 1 つのカーソルを毎回再利用
EXECUTE IMMEDIATE 'SELECT * FROM orders WHERE order_id = :1' USING p_id;
-- 毎回同じ SQL 文字列 = ソフト・パース (カーソル再利用)

-- ハード・パース vs ソフト・パースの割合を監視
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('parse count (hard)', 'parse count (total)');
-- OLTP システムでは Hard / Total が 1% 未満であることが望ましい
```

### 共有プールの断片化

一回限りの SQL 文字列を大量に生成する動的 SQL は、一度しか使われないカーソルで共有プールを断片化させます。これにより他のカーソルが押し出され、システム全体のハード・パースが増加します。

```sql
-- カーソル再利用の効率を確認
SELECT sql_text, executions, parse_calls,
       ROUND(parse_calls/NULLIF(executions,0) * 100, 1) AS parse_pct
FROM   v$sql
WHERE  parsing_schema_name = SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA')
  AND  executions > 0
  AND  ROUND(parse_calls/NULLIF(executions,0) * 100, 1) > 50
ORDER BY executions DESC;
-- parse_pct が高く実行回数が多い行は、インジェクションの懸念があるかバインド変数が不足している
```

---

## EXECUTE IMMEDIATE vs DBMS_SQL 判断ガイド

| 要件 | EXECUTE IMMEDIATE | DBMS_SQL |
|---|---|---|
| 単純な DDL 実行 | 可 | 可 |
| 固定されたバインド数での DML | 可 (推奨) | 可 |
| 単一行の SELECT | 可 | 可 |
| 複数行の SELECT | OPEN ... FOR | FETCH_ROWS ループ |
| 可変のバインド数 | 困難 (USING 句の組み合わせ) | 可 (ループ内 BIND_VARIABLE) |
| コンパイル時に不明な列構造 | 不可 | 可 (DESCRIBE_COLUMNS) |
| 一度パースし、何度も実行するパターン | 不可 | 可 |
| DBMS_SQL カーソルを REF CURSOR に変換 | 不可 | 可 (TO_REFCURSOR) |
| 読みやすさ | 高い | 低い |

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| SQL にデータ値を直接連結する | SQL インジェクションのリスク | バインド変数を使用する |
| 表名や列名に DBMS_ASSERT を使用しない | 構造要素を通じたインジェクション | すべての識別子を検証する |
| SQL キーワードをホワイトリスト化しない | `ORDER BY CASE WHEN ... DROP TABLE` 等の攻撃 | ホワイトリスト形式にする(`IN ('ASC', 'DESC')`) |
| 例外時に DBMS_SQL カーソルを閉じない | リソース・リーク、ORA-01000 | 例外ハンドラで `IF IS_OPEN THEN CLOSE` を行う |
| 頻繁なループ内で EXECUTE IMMEDIATE を使用 | 繰り返しハード・パースが発生 | バインド変数を持つ静的 SQL か FORALL を使用する |
| OPEN ... FOR の USING 句の数不一致 | ORA-01006: バインド変数が見つからない | バインド数を一致させる。可変の場合は DBMS_SQL を検討 |
| エラー・メッセージに SQL 文をそのまま出す | 攻撃者にスキーマ情報を与えてしまう | 内部でログに記録し、クライアントには汎用メッセージを返す |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **Oracle 8i以降**: `EXECUTE IMMEDIATE` および `OPEN cursor FOR dynamic_sql USING` が導入されました。ほとんどのケースで `DBMS_SQL` を置き換えました。
- **Oracle 11gR2以降**: `DBMS_SQL.TO_REFCURSOR` と `DBMS_SQL.TO_CURSOR_NUMBER` により、DBMS_SQL と REF CURSOR 型の相互変換が可能になりました。
- **Oracle 12cR1以降**: プロシージャからの暗黙的な結果セット用の `DBMS_SQL.RETURN_RESULT` が追加されました。
- **Oracle 21c以降**: 動的 SQL 構築パターンでの JSON サポートが向上しました。
- **全バージョン**: `DBMS_ASSERT` は 10.2 以降で利用可能です。動的識別子の検証には一貫してこれを使用してください。

---

## ソース

- [Oracle Database PL/SQL Language Reference 19c — Dynamic SQL](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/dynamic-sql.html) — EXECUTE IMMEDIATE, OPEN...FOR, USING clause
- [DBMS_SQL (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQL.html) — DBMS_SQL パッケージ, TO_REFCURSOR, RETURN_RESULT
- [DBMS_ASSERT (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_ASSERT.html) — インジェクション防止のための SQL_OBJECT_NAME, SIMPLE_SQL_NAME
