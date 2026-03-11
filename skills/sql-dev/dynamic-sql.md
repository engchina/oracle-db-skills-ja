# Oracleにおける動的SQL

## 概要

動的SQLとは、解析時ではなく実行時に構築および実行されるSQL文を指す。SQLの構造自体（表名、列名、バインド変数の数、または句全体）が実行時まで確定できない場合に必要となる。Oracleは2つのメカニズムを提供している。ほとんどのユースケースに適した `EXECUTE IMMEDIATE` と、バインド変数の数が完全に動的である場合や、プロシージャの境界を越えてカーソルを共有する必要があるなどの高度なシナリオ向けの `DBMS_SQL` である。

動的SQLには、主にSQLインジェクションのリスクと、ハード解析によるパフォーマンス・オーバーヘッドというリスクが伴うため、これらを慎重に管理する必要がある。本ガイドでは、両方のメカニズムを安全かつ効率的に使用する方法について解説する。

---

## 動的SQLを使用すべきケース

静的SQLでは要件を満たせない場合に動的SQLを使用する。

- 表名または列名が実行時に決定される
- 実行ごとにバインド変数の数や名前が変わる
- PL/SQLからDDLを実行する必要がある（静的PL/SQLにDDLを含めることはできない）
- 提供されたフィルタ・パラメータに基づいて、WHERE句を条件に応じて構築する
- パラメータとして渡されたSQL文字列を実行する必要がある

次の場合には動的SQLを使用**しない**。
- 静的SQLで同じことが実現できる場合（静的SQLの方が常に高速で安全である）
- 別のスキーマでSQLを実行することで権限の問題を回避しようとする場合
- 「動的」な部分が、実際には既知の小さな値のセットである場合（代わりに CASE を使用する）

---

## EXECUTE IMMEDIATE

`EXECUTE IMMEDIATE` は、ほとんどのケースで推奨される動的SQLのメカニズムである。1ステップで文字列を解析し、実行する。

### 基本構文

```plsql
-- DMLまたはDDL（DDLにはバインド変数は不要）
EXECUTE IMMEDIATE 'CREATE TABLE temp_results (id NUMBER, result VARCHAR2(200))';

-- バインド変数を使用したDML（USING句）
EXECUTE IMMEDIATE 'UPDATE employees SET salary = :1 WHERE employee_id = :2'
  USING p_new_salary, p_employee_id;

-- 単一要素を返すクエリ（INTO句）
EXECUTE IMMEDIATE 'SELECT last_name FROM employees WHERE employee_id = :1'
  INTO  v_last_name
  USING p_employee_id;

-- 複数行を返すクエリ（OPEN ... FOR 構文）
OPEN v_ref_cursor FOR
  'SELECT employee_id, last_name FROM employees WHERE department_id = :1'
  USING p_dept_id;
```

### PL/SQLからのDDL実行

静的PL/SQLではDDLを実行できない。`EXECUTE IMMEDIATE` が標準的な回避策となる。

```plsql
CREATE OR REPLACE PROCEDURE create_staging_table (
  p_table_suffix IN VARCHAR2
) AS
  v_table_name VARCHAR2(128);
BEGIN
  -- サフィックスのバリデーション — バリデーションされていない入力をDDLに連結してはならない
  v_table_name := 'STAGING_' || DBMS_ASSERT.SIMPLE_SQL_NAME(p_table_suffix);

  EXECUTE IMMEDIATE
    'CREATE TABLE ' || v_table_name || ' ('
    || '  id          NUMBER GENERATED ALWAYS AS IDENTITY,'
    || '  load_date   DATE DEFAULT SYSDATE,'
    || '  payload     VARCHAR2(4000)'
    || ')';

  DBMS_OUTPUT.PUT_LINE('Created table: ' || v_table_name);
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE = -955 THEN  -- ORA-00955: 名前がすでに使用されている
      DBMS_OUTPUT.PUT_LINE('Table already exists: ' || v_table_name);
    ELSE
      RAISE;
    END IF;
END;
/
```

### EXECUTE IMMEDIATE での SELECT INTO

```plsql
CREATE OR REPLACE PROCEDURE get_column_value (
  p_table_name  IN  VARCHAR2,
  p_column_name IN  VARCHAR2,
  p_id          IN  NUMBER,
  p_value       OUT VARCHAR2
) AS
  v_sql         VARCHAR2(500);
  v_safe_table  VARCHAR2(128);
  v_safe_col    VARCHAR2(128);
BEGIN
  -- SQL構造を構築する前に識別子をバリデーションする
  v_safe_table := DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);
  v_safe_col   := DBMS_ASSERT.SIMPLE_SQL_NAME(p_column_name);

  v_sql := 'SELECT TO_CHAR(' || v_safe_col
        || ') FROM ' || v_safe_table
        || ' WHERE id = :id_val';

  EXECUTE IMMEDIATE v_sql INTO p_value USING p_id;

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_value := NULL;
END;
/
```

### EXECUTE IMMEDIATE での Ref Cursor

Ref Cursor (SYS_REFCURSOR) を使用すると、動的な結果セットを呼び出し元に返すことができる。これは、動的なクエリ結果を呼び出し元のアプリケーションや別のPL/SQLブロックに渡すための標準的なパターンである。

```plsql
CREATE OR REPLACE PROCEDURE search_employees (
  p_dept_id   IN  NUMBER   DEFAULT NULL,
  p_min_sal   IN  NUMBER   DEFAULT NULL,
  p_job_id    IN  VARCHAR2 DEFAULT NULL,
  p_cursor    OUT SYS_REFCURSOR
) AS
  v_sql       CLOB;
  v_where     VARCHAR2(1000) := ' WHERE 1=1';
BEGIN
  -- WHERE句を条件に応じて構築
  -- 各データ値は名前付きバインド変数として扱い、決して連結しない
  IF p_dept_id IS NOT NULL THEN
    v_where := v_where || ' AND department_id = :dept_id';
  END IF;
  IF p_min_sal IS NOT NULL THEN
    v_where := v_where || ' AND salary >= :min_sal';
  END IF;
  IF p_job_id IS NOT NULL THEN
    v_where := v_where || ' AND job_id = :job_id';
  END IF;

  v_sql := 'SELECT employee_id, last_name, salary, department_id, job_id'
        || ' FROM employees'
        || v_where
        || ' ORDER BY last_name';

  -- USING句は、WHERE句内のすべてのバインド変数と一致する必要がある
  -- これには追加された変数を把握しておく必要があるため、完全に動的なバインドにはヘルパーまたはDBMS_SQLを使用する
  IF p_dept_id IS NOT NULL AND p_min_sal IS NOT NULL AND p_job_id IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_dept_id, p_min_sal, p_job_id;
  ELSIF p_dept_id IS NOT NULL AND p_min_sal IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_dept_id, p_min_sal;
  ELSIF p_dept_id IS NOT NULL AND p_job_id IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_dept_id, p_job_id;
  ELSIF p_min_sal IS NOT NULL AND p_job_id IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_min_sal, p_job_id;
  ELSIF p_dept_id IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_dept_id;
  ELSIF p_min_sal IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_min_sal;
  ELSIF p_job_id IS NOT NULL THEN
    OPEN p_cursor FOR v_sql USING p_job_id;
  ELSE
    OPEN p_cursor FOR v_sql;
  END IF;

  -- 注: オプション引数が多い場合、上記の分岐は管理しきれなくなる。
  -- バインド変数の数が完全に動的な場合は、DBMS_SQL（後述）を使用すること。
END;
/
```

---

## DBMS_SQL

`DBMS_SQL` は動力SQLのための低レベルAPIである。コード量は多くなるが、以下の機能をサポートしている。

- 動的な数のバインド変数（条件分岐が不要）
- 動的な数の列へのフェッチ
- 解析済みカーソルの複数回実行における再利用（解析オーバーヘッドの削減）
- `DBMS_SQL` カーソルから `REF CURSOR` への変換、およびその逆

### DBMS_SQL の処理ステップ

1. `OPEN_CURSOR` — カーソル・ハンドルを取得
2. `PARSE` — SQL文字列を解析
3. `BIND_VARIABLE` / `BIND_ARRAY` — 値をバインド
4. `DEFINE_COLUMN` — 出力列を登録（SELECTの場合）
5. `EXECUTE` — 文を実行
6. `FETCH_ROWS` — 行をフェッチ（SELECTの場合）
7. `COLUMN_VALUE` — フェッチ後に列値を取得
8. `CLOSE_CURSOR` — リソースを解放

### 例: バインド数が可変の動的 INSERT

```plsql
CREATE OR REPLACE PROCEDURE dynamic_insert (
  p_table_name  IN VARCHAR2,
  p_col_names   IN SYS.ODCIVARCHAR2LIST,  -- 例: SYS.ODCIVARCHAR2LIST('name','salary')
  p_col_values  IN SYS.ODCIVARCHAR2LIST   -- すべての値を文字列として渡し、必要に応じてキャスト
) AS
  v_cursor   INTEGER;
  v_sql      CLOB;
  v_cols     VARCHAR2(4000) := '';
  v_binds    VARCHAR2(4000) := '';
  v_rows     INTEGER;
  v_table    VARCHAR2(128);
BEGIN
  -- 表名のバリデーション
  v_table := DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);

  -- 列リストとバインド・プレースホルダ・リストの構築
  FOR i IN 1..p_col_names.COUNT LOOP
    v_cols  := v_cols  || DBMS_ASSERT.SIMPLE_SQL_NAME(p_col_names(i));
    v_binds := v_binds || ':b' || i;
    IF i < p_col_names.COUNT THEN
      v_cols  := v_cols  || ', ';
      v_binds := v_binds || ', ';
    END IF;
  END LOOP;

  v_sql := 'INSERT INTO ' || v_table || ' (' || v_cols || ') VALUES (' || v_binds || ')';

  -- 解析とバインド
  v_cursor := DBMS_SQL.OPEN_CURSOR;
  BEGIN
    DBMS_SQL.PARSE(v_cursor, v_sql, DBMS_SQL.NATIVE);

    FOR i IN 1..p_col_values.COUNT LOOP
      DBMS_SQL.BIND_VARIABLE(v_cursor, ':b' || i, p_col_values(i));
    END LOOP;

    v_rows := DBMS_SQL.EXECUTE(v_cursor);
    DBMS_OUTPUT.PUT_LINE('Inserted ' || v_rows || ' row(s).');

    DBMS_SQL.CLOSE_CURSOR(v_cursor);
  EXCEPTION
    WHEN OTHERS THEN
      IF DBMS_SQL.IS_OPEN(v_cursor) THEN
        DBMS_SQL.CLOSE_CURSOR(v_cursor);
      END IF;
      RAISE;
  END;
END;
/
```

### 例: 列数が不明な動力 SELECT

```plsql
CREATE OR REPLACE PROCEDURE dump_query_results (
  p_sql IN VARCHAR2
) AS
  v_cursor    INTEGER;
  v_col_cnt   INTEGER;
  v_desc_tab  DBMS_SQL.DESC_TAB;
  v_val       VARCHAR2(4000);
  v_row_cnt   INTEGER;
  v_line      VARCHAR2(32767);
BEGIN
  v_cursor := DBMS_SQL.OPEN_CURSOR;
  BEGIN
    DBMS_SQL.PARSE(v_cursor, p_sql, DBMS_SQL.NATIVE);
    DBMS_SQL.DESCRIBE_COLUMNS(v_cursor, v_col_cnt, v_desc_tab);

    -- すべての列を登録
    FOR i IN 1..v_col_cnt LOOP
      DBMS_SQL.DEFINE_COLUMN(v_cursor, i, v_val, 4000);
    END LOOP;

    v_row_cnt := DBMS_SQL.EXECUTE(v_cursor);

    -- ヘッダーを印刷
    v_line := '';
    FOR i IN 1..v_col_cnt LOOP
      v_line := v_line || RPAD(v_desc_tab(i).col_name, 20);
    END LOOP;
    DBMS_OUTPUT.PUT_LINE(v_line);
    DBMS_OUTPUT.PUT_LINE(RPAD('-', 20 * v_col_cnt, '-'));

    -- 行をフェッチして印刷
    LOOP
      EXIT WHEN DBMS_SQL.FETCH_ROWS(v_cursor) = 0;
      v_line := '';
      FOR i IN 1..v_col_cnt LOOP
        DBMS_SQL.COLUMN_VALUE(v_cursor, i, v_val);
        v_line := v_line || RPAD(NVL(v_val, 'NULL'), 20);
      END LOOP;
      DBMS_OUTPUT.PUT_LINE(v_line);
    END LOOP;

    DBMS_SQL.CLOSE_CURSOR(v_cursor);
  EXCEPTION
    WHEN OTHERS THEN
      IF DBMS_SQL.IS_OPEN(v_cursor) THEN DBMS_SQL.CLOSE_CURSOR(v_cursor); END IF;
      RAISE;
  END;
END;
/
```

### DBMS_SQL と REF CURSOR 間の変換

バインドと実行の後、`DBMS_SQL` カーソルを `SYS_REFCURSOR` に変換できる。これは呼び出し元に結果を返す際に便利である。

```plsql
CREATE OR REPLACE FUNCTION flexible_query (
  p_table_name IN VARCHAR2,
  p_where_cols IN SYS.ODCIVARCHAR2LIST,  -- WHERE条件に使用する列名
  p_where_vals IN SYS.ODCIVARCHAR2LIST   -- 対応する値
) RETURN SYS_REFCURSOR AS
  v_cursor    INTEGER;
  v_ref_cur   SYS_REFCURSOR;
  v_sql       CLOB;
  v_where     VARCHAR2(4000) := '';
  v_dummy     INTEGER;
BEGIN
  -- バリデーション済みの列名とバインド・プレースホルダを使用してWHERE句を構築
  FOR i IN 1..p_where_cols.COUNT LOOP
    IF i > 1 THEN v_where := v_where || ' AND '; END IF;
    v_where := v_where || DBMS_ASSERT.SIMPLE_SQL_NAME(p_where_cols(i)) || ' = :b' || i;
  END LOOP;

  v_sql := 'SELECT * FROM ' || DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);
  IF v_where IS NOT NULL THEN
    v_sql := v_sql || ' WHERE ' || v_where;
  END IF;

  -- DBMS_SQL経由で解析・バインド
  v_cursor := DBMS_SQL.OPEN_CURSOR;
  DBMS_SQL.PARSE(v_cursor, v_sql, DBMS_SQL.NATIVE);

  FOR i IN 1..p_where_vals.COUNT LOOP
    DBMS_SQL.BIND_VARIABLE(v_cursor, ':b' || i, p_where_vals(i));
  END LOOP;

  v_dummy := DBMS_SQL.EXECUTE(v_cursor);

  -- 呼び出し元に返すために DBMS_SQL カーソルを REF CURSOR に変換
  v_ref_cur := DBMS_SQL.TO_REFCURSOR(v_cursor);
  -- 注意: TO_REFCURSOR の後には、v_cursor に対して CLOSE_CURSOR を呼び出さないこと。
  -- REF CURSOR がカーソル・リソースを所有するため。

  RETURN v_ref_cur;

EXCEPTION
  WHEN OTHERS THEN
    IF DBMS_SQL.IS_OPEN(v_cursor) THEN DBMS_SQL.CLOSE_CURSOR(v_cursor); END IF;
    RAISE;
END;
/
```

---

## EXECUTE IMMEDIATE と DBMS_SQL の選択

| 基準 | EXECUTE IMMEDIATE | DBMS_SQL |
|---|---|---|
| コードの簡潔さ | 簡潔 | 冗長 |
| 固定数のバインド変数 | 対応 | 対応 |
| 動的な数のバインド変数 | 非対応（分岐が必要） | 対応 |
| 動的な数の出力列 | 非対応 | 対応 |
| 解析済みカーソルの再利用 | 非対応（毎回再解析） | 対応 |
| REF CURSORとの変換 | 非対応 | 対応 (`TO_REFCURSOR`, `TO_CURSOR_NUMBER`) |
| DDLの実行 | 対応 | 対応 |
| パフォーマンス（単一実行） | 同等 | 同等 |
| パフォーマンス（繰り返し実行） | やや劣る（コードが少ない分有利） | 優れる（カーソルの再利用） |

**推奨事項:** デフォルトでは `EXECUTE IMMEDIATE` を使用する。以下の場合にのみ `DBMS_SQL` に切り替える。
- コンパイル時にバインド変数や列の数が不明で、条件分岐では管理しきれない場合
- 解析済みカーソルを複数の呼び出しにわたって再利用する必要がある場合（一回解析・多回実行パターン）
- `TO_REFCURSOR` 変換が必要な場合

---

## 動的SQLにおけるSQLインジェクションの回避

動的SQLは、PL/SQLにおけるSQLインジェクション攻撃の主要なターゲットとなる。以下のルールを厳守すること。

### ルール1: データ値には常にバインド変数を使用する

```plsql
-- 悪い例: データ値をSQL文字列に直接連結する（危険）
v_sql := 'SELECT * FROM t WHERE name = ''' || p_name || '''';

-- 良い例: データ値には常にバインド変数を使用する（安全）
v_sql := 'SELECT * FROM t WHERE name = :name';
EXECUTE IMMEDIATE v_sql INTO v_result USING p_name;
```

### ルール2: DBMS_ASSERT ですべての動的識別子をバリデーションする

```plsql
-- 動的な表名/列名/スキーマ名にはバインド変数を使用できない
-- 連結する前に必ずバリデーションを行う必要がある

v_safe_table  := DBMS_ASSERT.SQL_OBJECT_NAME(p_table_name);  -- DB内に存在する必要がある
v_safe_column := DBMS_ASSERT.SIMPLE_SQL_NAME(p_col_name);    -- 単純な識別子のみ
v_safe_schema := DBMS_ASSERT.SCHEMA_NAME(p_schema);          -- 有効なスキーマ名である必要がある

v_sql := 'SELECT ' || v_safe_column || ' FROM ' || v_safe_schema || '.' || v_safe_table;
```

### ルール3: 識別子以外の構造には明示的なホワイトリストを使用する

識別子以外（ソート方向、SQLキーワード、演算子記号）については、明示的にホワイトリストで制限する必要がある。

```plsql
-- ソート方向のホワイトリスト
v_direction := CASE UPPER(p_sort_dir)
                 WHEN 'ASC'  THEN 'ASC'
                 WHEN 'DESC' THEN 'DESC'
                 ELSE RAISE_APPLICATION_ERROR(-20001, 'Invalid sort direction.')
               END;

-- 演算子のホワイトリスト
v_operator := CASE p_operator
                WHEN '='  THEN '='
                WHEN '>'  THEN '>'
                WHEN '<'  THEN '<'
                WHEN '>=' THEN '>='
                WHEN '<=' THEN '<='
                ELSE NULL
              END;
IF v_operator IS NULL THEN
  RAISE_APPLICATION_ERROR(-20002, 'Invalid operator.');
END IF;
```

### ルール4: 入力ソースを決して信用しない

社内システム、API、ミドルウェアなどはすべてインジェクションのベクトルになり得る。入力がどこから来たかに関わらず、必ずバリデーションを行うこと。

---

## パフォーマンスに関する考慮事項

### ハード解析のコスト

新しいSQL文字列が作成されるたびに、ハード解析（解析、最適化、実行計画の生成）が発生する。ハード解析はコストが高く、共有プール・ラッチの競合を引き起こす。

```plsql
-- 悪い例: 実行ごとに一意のリテラルが使用される → ハード解析の嵐
FOR i IN 1..10000 LOOP
  EXECUTE IMMEDIATE 'SELECT count(*) FROM t WHERE id = ' || i INTO v_cnt;
END LOOP;

-- 良い例: バインド変数を使用して同じステートメント・テキストを再利用する
FOR i IN 1..10000 LOOP
  EXECUTE IMMEDIATE 'SELECT count(*) FROM t WHERE id = :1' INTO v_cnt USING i;
END LOOP;
```

### DBMS_SQL によるカーソルの再利用

Bulk Loadのループ内などで同じ動的な文を繰り返し実行する場合、`DBMS_SQL` を使用して一度だけ解析し、N回実行する。

```plsql
DECLARE
  v_cursor  INTEGER;
  v_sql     VARCHAR2(200) := 'INSERT INTO staging (id, val) VALUES (:1, :2)';
  v_rows    INTEGER;
BEGIN
  v_cursor := DBMS_SQL.OPEN_CURSOR;
  DBMS_SQL.PARSE(v_cursor, v_sql, DBMS_SQL.NATIVE);

  FOR i IN 1..100000 LOOP
    DBMS_SQL.BIND_VARIABLE(v_cursor, ':1', i);
    DBMS_SQL.BIND_VARIABLE(v_cursor, ':2', 'value_' || i);
    v_rows := DBMS_SQL.EXECUTE(v_cursor);  -- 再解析なし
  END LOOP;

  DBMS_SQL.CLOSE_CURSOR(v_cursor);
  COMMIT;
END;
/
```

---

## 動的SQLによるバルク操作

`EXECUTE IMMEDIATE` は、動的DMLに対して `BULK COLLECT` および `FORALL` をサポートしている。

```plsql
DECLARE
  TYPE id_list IS TABLE OF NUMBER;
  v_ids    id_list;
  v_sql    VARCHAR2(200);
BEGIN
  -- 動的なバルク・クエリ
  v_sql := 'SELECT employee_id FROM employees WHERE department_id = :1';
  EXECUTE IMMEDIATE v_sql BULK COLLECT INTO v_ids USING 50;

  -- 動的な FORALL
  v_sql := 'UPDATE employees SET salary = salary * 1.1 WHERE employee_id = :1';
  FORALL i IN 1..v_ids.COUNT
    EXECUTE IMMEDIATE v_sql USING v_ids(i);

  COMMIT;
END;
/
```

---

## ベスト・プラクティス

- **まずは静的SQLから検討する。** 静的SQLで要件をどうしても満たせない場合にのみ、動的SQLを導入する。
- **データ値はすべてバインドする。** ユーザー入力やデータ値をSQL文字列に直接連結してはならない。
- **構造上の要素（表名、列名など）はすべてバリデーションする。** 連結する前に、`DBMS_ASSERT` 関数または明示的なホワイトリストを使用する。
- **必ずカーソルを閉じる。** カーソル・リークを防ぐため、例外ハンドラ内で `DBMS_SQL.IS_OPEN` チェックを行う。
- **開発・テスト中に構築されたSQLをログ出力する。** 本番環境に移行する前に、最終的なSQLコードを検証するためにデバッグ・ログ呼び出しを追加する。
- **簡潔さのために `EXECUTE IMMEDIATE` を優先する。** 必要性が認められる場合にのみ `DBMS_SQL` を使用する。
- **長い動的SQLには `CLOB` を使用する。** PL/SQLの `VARCHAR2` は32,767バイトまでに制限されている。多くの列やサブクエリを含む長いクエリはこの制限を超える可能性がある。
- **トランザクション・コード内でのDDLは避ける。** DDLは暗黙的にコミットされるため、トランザクションの途中でDDLを実行すると、それまでのすべての処理がコミットされる。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| 文字列値の連結: `'...WHERE name = ''' \|\| p_name \|\| ''''` | SQLインジェクションのベクトル | バインド変数を使用する: `USING p_name` |
| `DBMS_ASSERT.NOOP` がバリデーションを行うと誤解する | `NOOP` は何もしない。文字列をそのまま返す | `SQL_OBJECT_NAME` または `SIMPLE_SQL_NAME` を使用する |
| 例外パスで `DBMS_SQL.CLOSE_CURSOR` を忘れる | カーソル・リークが発生し、最終的に `ORA-01000` に達する | すべての例外ハンドラで `IS_OPEN` をチェックして閉じる |
| `TO_REFCURSOR` の後に `CLOSE_CURSOR` を呼び出す | `TO_REFCURSOR` は所有権を移譲する。整数ハンドルのクローズはエラーとなる | `TO_REFCURSOR` 以後は `SYS_REFCURSOR` のみを閉じ、整数ハンドルは閉じない |
| 32,767バイトを超える VARCHAR2 で SQL 文字列を構築する | `PL/SQL: 数値または値のエラー` | SQL文字列変数に `CLOB` を使用する |
| ループ内でDDLを実行する | 各DDLが暗黙的にコミットされ、ハード解析を発生させる | ループ外でDDLを実行するか、スキーマセットアップ時に一括で実行する |
| DMLループ内で毎回 `EXECUTE IMMEDIATE` を呼び出す | 反復ごとにハード解析（文字列が変わる場合）または解析＋実行が発生する | `FORALL` と `EXECUTE IMMEDIATE` を使用するか、DBMS_SQLの一回解析・多回実行を使用する |
| USING引数の数とバインド変数の数が一致しない | `ORA-01008: バインドされていない変数があります` | SQL文字列内のプレースホルダをカウントし、USINGに正確にその数の引数を指定する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ガイドの基本的な指針は、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッド環境では、デフォルトや非推奨がリリース更新によって異なる場合があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c PL/SQL Language Reference (LNPLS)](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/)
- [DBMS_SQL — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQL.html)
- [DBMS_ASSERT — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_ASSERT.html)
