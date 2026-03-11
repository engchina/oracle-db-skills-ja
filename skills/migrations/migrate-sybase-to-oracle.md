# Sybase ASE から Oracle への移行

## 概要

Sybase Adaptive Server Enterprise (ASE、現在は SAP ASE) は、Microsoft SQL Server と共通の系統を持っており、共に Sybase のオリジナルのデータベース・エンジンを祖先としている。その結果、Sybase ASE と SQL Server は、Transact-SQL (T-SQL) 方言、手続き型言語の構文、およびアーキテクチャが非常によく似ている。SQL Server 移行ガイド (`migrate-sqlserver-to-oracle.md`) に記載されている概念の多くは、Sybase ASE にも適用可能である。

本ガイドでは、Sybase 固有の挙動、および Sybase T-SQL と Oracle PL/SQL の違いに焦点を当てる。主なトピックには、データ型マッピング、ストアド・プロシージャの変換（Sybase T-SQL プロシージャ vs Oracle PL/SQL）、トランザクション処理の違い（Sybase の chained / unchained モード）、BCP (Bulk Copy Program) から SQL*Loader へのマッピング、および Sybase 固有の SQL 構文が含まれる。

---

## Sybase ASE から Oracle への型マッピング

### 数値型

| Sybase ASE | Oracle | 備考 |
|---|---|---|
| `TINYINT` | `NUMBER(3)` | 0 ～ 255 |
| `SMALLINT` | `NUMBER(5)` | −32,768 ～ 32,767 |
| `INT` / `INTEGER` | `NUMBER(10)` | |
| `BIGINT` | `NUMBER(19)` | ASE 15.0 以降 |
| `DECIMAL(p,s)` / `DEC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `NUMERIC(p,s)` / `NUM(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `FLOAT(n)` | `BINARY_FLOAT` または `BINARY_DOUBLE` | |
| `REAL` | `BINARY_FLOAT` | IEEE 754 32 ビット |
| `DOUBLE PRECISION` | `BINARY_DOUBLE` | IEEE 754 64 ビット |
| `MONEY` | `NUMBER(19,4)` | 8 バイトの Sybase 通貨型 |
| `SMALLMONEY` | `NUMBER(10,4)` | 4 バイト |
| `BIT` | CHECK (0,1) 付きの `NUMBER(1)` | 論理ビット型 |

### 文字列型

| Sybase ASE | Oracle | 備考 |
|---|---|---|
| `CHAR(n)` | `CHAR(n)` | Sybase 最大 255 バイト、Oracle 最大 2,000 |
| `VARCHAR(n)` | `VARCHAR2(n)` | Sybase 最大 16,383 バイト、Oracle 最大 4,000/32,767 |
| `NCHAR(n)` | `NCHAR(n)` | Unicode 固定長 |
| `NVARCHAR(n)` | `NVARCHAR2(n)` | Unicode 可変長 |
| `TEXT` | `CLOB` | Sybase 最大 2 GB |
| `UNITEXT` | `NCLOB` | Unicode TEXT (ASE 15.0 以降) |
| `IMAGE` | `BLOB` | バイナリ・ラージ・オブジェクト |

### 日付 / 時刻型

| Sybase ASE | Oracle | 備考 |
|---|---|---|
| `DATETIME` | `TIMESTAMP` | Sybase 精度：1/300 秒 |
| `SMALLDATETIME` | `DATE` | 1 分単位の精度 |
| `DATE` | `DATE` | ASE 12.5.1 以降。Sybase の DATE は日付のみ、Oracle は時刻も含む |
| `TIME` | 同等の型なし | `VARCHAR2(12)` または `NUMBER` を使用 |
| `BIGDATETIME` | `TIMESTAMP(6)` | ASE 15.7 以降。マイクロ秒の精度 |
| `BIGTIME` | 同等の型なし | ASE 15.7 以降。マイクロ秒単位の時刻 |

### バイナリおよびその他の型

| Sybase ASE | Oracle | 備考 |
|---|---|---|
| `BINARY(n)` | `RAW(n)` | 固定長バイナリ。Sybase 最大 255 バイト |
| `VARBINARY(n)` | `RAW(n)` | 可変長バイナリ |
| `TIMESTAMP` (行バージョン) | 同等の型なし | Sybase の TIMESTAMP は日時ではなく行バージョン・カウンタ。`ORA_ROWSCN` を使用 |
| `UNICHAR(n)` | `NCHAR(n)` | Sybase Unicode 固定長 |
| `UNIVARCHAR(n)` | `NVARCHAR2(n)` | Sybase Unicode 可変長 |

**重要な相違点：** Sybase の `TIMESTAMP` は 8 バイト・バイナリの行バージョン番号（SQL Server の ROWVERSION と同様）であり、日付/時刻型ではない。楽観的排他制御のために Sybase TIMESTAMP を使用しているアプリケーションは、Oracle では再設計が必要である。

```sql
-- Sybase: 楽観的ロック用の行バージョン
CREATE TABLE products (
    product_id   INT          IDENTITY NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    price        MONEY,
    row_version  TIMESTAMP    NOT NULL  -- 自動更新されるバイナリ・カウンタ
);

-- Oracle: ORA_ROWSCN を使用するか、明示的なバージョン列を追加する
CREATE TABLE products (
    product_id   NUMBER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_name VARCHAR2(200) NOT NULL,
    price        NUMBER(19,4),
    version_num  NUMBER(10)   DEFAULT 0 NOT NULL  -- アプリケーション管理のバージョン
);
-- または ORA_ROWSCN 擬似列（システム管理の SCN）を使用
SELECT ORA_ROWSCN, product_id FROM products WHERE product_id = 42;
```

---

## Sybase ストアド・プロシージャの PL/SQL への変換

### 基本的なプロシージャ構造

```sql
-- Sybase T-SQL ストアド・プロシージャ
CREATE PROCEDURE sp_update_salary
    @emp_id        INT,
    @new_salary    MONEY,
    @rows_affected INT OUTPUT
AS
BEGIN
    UPDATE employees
    SET salary = @new_salary,
        updated_date = GETDATE()
    WHERE emp_id = @emp_id;

    SELECT @rows_affected = @@ROWCOUNT;

    IF @rows_affected = 0
    BEGIN
        RAISERROR 20001 "Employee not found: %1!", @emp_id;
        RETURN -1;
    END;

    RETURN 0;
END;
GO

-- Oracle PL/SQL 同等実装
CREATE OR REPLACE PROCEDURE sp_update_salary(
    p_emp_id        IN  NUMBER,
    p_new_salary    IN  NUMBER,
    p_rows_affected OUT NUMBER
)
AS
BEGIN
    UPDATE employees
    SET salary       = p_new_salary,
        updated_date = SYSDATE
    WHERE emp_id = p_emp_id;

    p_rows_affected := SQL%ROWCOUNT;

    IF p_rows_affected = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Employee not found: ' || p_emp_id);
    END IF;
END sp_update_salary;
/
```

### 変数の宣言

```sql
-- Sybase
DECLARE
    @counter    INT,
    @total      MONEY,
    @name       VARCHAR(100);

SELECT @counter = 0, @total = 0.00, @name = '';

-- Oracle PL/SQL
DECLARE
    v_counter  NUMBER(10)    := 0;
    v_total    NUMBER(19,4)  := 0;
    v_name     VARCHAR2(100) := '';
BEGIN
    NULL;
END;
/
```

### 制御フロー

```sql
-- Sybase: IF / ELSE
IF @score >= 90
    SELECT 'A' AS grade
ELSE IF @score >= 80
    SELECT 'B' AS grade
ELSE
    SELECT 'C' AS grade;

-- Oracle PL/SQL
IF v_score >= 90 THEN
    v_grade := 'A';
ELSIF v_score >= 80 THEN
    v_grade := 'B';
ELSE
    v_grade := 'C';
END IF;
```

```sql
-- Sybase: BREAK/CONTINUE を使用した WHILE ループ
DECLARE @i INT = 1;
WHILE @i <= 100
BEGIN
    IF @i % 2 = 0
    BEGIN
        SET @i = @i + 1;
        CONTINUE;
    END;
    INSERT INTO odd_numbers (num) VALUES (@i);
    IF @i >= 99 BREAK;
    SET @i = @i + 1;
END;

-- Oracle PL/SQL
DECLARE
    v_i NUMBER := 1;
BEGIN
    WHILE v_i <= 100 LOOP
        IF MOD(v_i, 2) = 0 THEN
            v_i := v_i + 1;
            CONTINUE;
        END IF;
        INSERT INTO odd_numbers (num) VALUES (v_i);
        EXIT WHEN v_i >= 99;
        v_i := v_i + 1;
    END LOOP;
END;
/
```

### エラー処理

Sybase T-SQL は `@@ERROR`（各ステートメントの後にチェックする必要があるグローバル変数）と `RAISERROR` を使用する。Oracle は構造化された例外処理ブロックを使用する。

```sql
-- Sybase: 各ステートメントの後に @@ERROR をチェック
BEGIN TRANSACTION;

INSERT INTO orders (customer_id, total) VALUES (@cust_id, @total);
IF @@ERROR <> 0
BEGIN
    ROLLBACK TRANSACTION;
    RAISERROR 20100 "Failed to insert order";
    RETURN -1;
END;

UPDATE customer_stats SET order_count = order_count + 1
WHERE customer_id = @cust_id;
IF @@ERROR <> 0
BEGIN
    ROLLBACK TRANSACTION;
    RAISERROR 20101 "Failed to update stats";
    RETURN -1;
END;

COMMIT TRANSACTION;
RETURN 0;

-- Oracle PL/SQL: 構造化例外処理
BEGIN
    INSERT INTO orders (customer_id, total) VALUES (v_cust_id, v_total);
    UPDATE customer_stats SET order_count = order_count + 1
    WHERE customer_id = v_cust_id;
    COMMIT;
EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20100, 'Duplicate order detected');
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20101, 'Order processing failed: ' || SQLERRM);
END;
/
```

### 一時テーブル

```sql
-- Sybase: ローカル一時テーブル (セッション・スコープ)
CREATE TABLE #staging (
    id    INT IDENTITY,
    value VARCHAR(100)
);
INSERT INTO #staging (value) VALUES ('test');
SELECT * FROM #staging;
DROP TABLE #staging;

-- Sybase: グローバル一時テーブル (セッションを跨いでアクセス可能)
CREATE TABLE ##global_staging (id INT, value VARCHAR(100));

-- Oracle: グローバル一時表 (一度作成すれば、データはセッション・スコープ)
CREATE GLOBAL TEMPORARY TABLE staging (
    id    NUMBER,
    value VARCHAR2(100)
)
ON COMMIT DELETE ROWS;  -- または ON COMMIT PRESERVE ROWS

-- 使用方法 (通常のテーブルと同様。Oracle の GTT 自体はセッションを跨いで永続するがデータは永続しない)
INSERT INTO staging (id, value) VALUES (1, 'test');
SELECT * FROM staging;
-- データの削除タイミング：トランザクション終了時 (DELETE ROWS) またはセッション終了時 (PRESERVE ROWS)
```

### カーソル

```sql
-- Sybase カーソル
DECLARE order_cursor CURSOR FOR
    SELECT order_id, total FROM orders WHERE status = 'pending';

OPEN order_cursor;
FETCH order_cursor INTO @oid, @total;
WHILE @@SQLSTATUS = 0
BEGIN
    -- @oid, @total の処理...
    FETCH order_cursor INTO @oid, @total;
END;
CLOSE order_cursor;
DEALLOCATE CURSOR order_cursor;

-- Oracle PL/SQL カーソル (推奨される FOR ループ・アプローチ)
BEGIN
    FOR rec IN (SELECT order_id, total FROM orders WHERE status = 'pending') LOOP
        -- rec.order_id, rec.total を使用した処理...
        NULL;
    END LOOP;
END;
/

-- Oracle 明示的カーソル (より細かな制御が必要な場合)
DECLARE
    CURSOR order_cur IS
        SELECT order_id, total FROM orders WHERE status = 'pending';
    v_oid   orders.order_id%TYPE;
    v_total orders.total%TYPE;
BEGIN
    OPEN order_cur;
    LOOP
        FETCH order_cur INTO v_oid, v_total;
        EXIT WHEN order_cur%NOTFOUND;
        -- 処理...
    END LOOP;
    CLOSE order_cur;
END;
/
```

---

## トランザクション処理の違い

これは、Sybase から Oracle への移行における最も重要な領域の 1 つである。

### Sybase のトランザクション・モード

Sybase ASE は 2 つのモードで動作する。

**Unchained モード (デフォルト) :** 各 DML ステートメントがそれぞれ暗黙のトランザクションとなる。ステートメントをグループ化するには、明示的に `BEGIN TRANSACTION` を実行する必要がある。

**Chained モード (ANSI 準拠) :** 各 DML ステートメントが暗黙のトランザクションを開始し、明示的にコミットまたはロールバックする必要がある（Oracle の動作に近い）。

```sql
-- Sybase unchained モード (デフォルト) : 各ステートメントはオートコミットされる
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- ^ これは即座にコミットされ、明示的な COMMIT は不要

-- Sybase: 明示的なトランザクションのグループ化
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT TRANSACTION;
```

### Oracle のトランザクション・モデル

Oracle は常に chained (ANSI) モードで動作する。すべての DML ステートメントは、明示的にコミットまたはロールバックされるまで、開いているトランザクションの一部である。DDL はオートコミットされる。

```sql
-- Oracle: すべての DML は暗黙のトランザクション内にある
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- 未コミットのため、明示的に COMMIT する必要がある
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- エラー発生時のロールバック
BEGIN
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/
```

**移行への影響：** unchained モードで動作している Sybase アプリケーションは、個々のステートメントに対してオートコミット動作を行う。Oracle に移行すると、これらのアプリケーションでは、コミットされていない DML が同じセッション内では見えるが、他のセッションからは見えないことに注意が必要である。アプリケーションの接続管理とトランザクション・スコープを慎重に見直すこと。

### SAVE TRANSACTION → SAVEPOINT

```sql
-- Sybase
BEGIN TRANSACTION;
INSERT INTO t1 (col) VALUES ('a');
SAVE TRANSACTION sp1;
INSERT INTO t2 (col) VALUES ('b');
-- エラー発生時：
ROLLBACK TRANSACTION sp1;  -- トランザクション全体ではなくセーブポイントまで戻る
COMMIT TRANSACTION;

-- Oracle
INSERT INTO t1 (col) VALUES ('a');
SAVEPOINT sp1;
INSERT INTO t2 (col) VALUES ('b');
-- エラー発生時：
ROLLBACK TO SAVEPOINT sp1;
COMMIT;
```

---

## BCP から SQL*Loader へ

Sybase の Bulk Copy Program (BCP) は、一括データの抽出およびロードに使用される主要なツールである。

### Sybase からの BCP エクスポート

```bash
# BCP エクスポート — ネイティブ形式 (バイナリ、Sybase 固有)
bcp mydb..customers out /data/customers.dat -Smyhost -Umyuser -Pmypass -n

# BCP エクスポート — 文字形式 (移植性があり、Oracle への移行に推奨)
bcp mydb..customers out /data/customers.csv -Smyhost -Umyuser -Pmypass -c -t"," -r"\n"

# フォーマット・ファイルを使用した BCP エクスポート
bcp mydb..customers format nul -Smyhost -Umyuser -Pmypass -c -t"," -f customers.fmt
bcp mydb..customers out /data/customers.dat -Smyhost -Umyuser -Pmypass -f customers.fmt
```

### Oracle への SQL*Loader インポート

```
-- SQL*Loader コントロール・ファイル: customers.ctl
OPTIONS (DIRECT=TRUE, ROWS=5000, ERRORS=100)
LOAD DATA
INFILE '/data/customers.csv'
APPEND
INTO TABLE customers
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
TRAILING NULLCOLS
(
    customer_id,
    first_name,
    last_name,
    email,
    created_date   DATE "YYYY-MM-DD",
    last_login     TIMESTAMP "YYYY-MM-DD HH24:MI:SS.FF3",
    is_active      "DECODE(:is_active, '1', 1, '0', 0, NULL)"
)
```

```bash
sqlldr userid=myuser/mypass@mydb control=customers.ctl log=customers.log bad=customers.bad
```

---

## Sybase 固有の SQL → Oracle の同等物

### グローバル変数 → コンテキスト関数

| Sybase | Oracle 同等物 |
|---|---|
| `@@ROWCOUNT` | `SQL%ROWCOUNT` (PL/SQL 内) |
| `@@ERROR` | `SQLCODE` (PL/SQL 例外セクション内) |
| `@@TRANCOUNT` | 直接的な同等物なし |
| `@@IDENTITY` | `RETURNING INTO` 句を使用 |
| `@@SPID` | `SYS_CONTEXT('USERENV', 'SID')` |
| `@@VERSION` | `SELECT * FROM v$version` |
| `@@SERVERNAME` | `SYS_CONTEXT('USERENV', 'DB_NAME')` |
| `@@DBTS` | `SELECT SYSTIMESTAMP FROM DUAL` |
| `GETDATE()` | `SYSDATE` または `SYSTIMESTAMP` |
| `OBJECT_ID('tablename')` | `SELECT object_id FROM user_objects WHERE object_name = 'TABLENAME'` |

### 文字列関数

| Sybase 関数 | Oracle 同等物 |
|---|---|
| `CHARINDEX(sub, s [, start])` | `INSTR(s, sub [, start])` |
| `PATINDEX('%pat%', s)` | `REGEXP_INSTR(s, 'pat')` |
| `LEFT(s, n)` | `SUBSTR(s, 1, n)` |
| `RIGHT(s, n)` | `SUBSTR(s, -n)` |
| `STUFF(s, start, len, replacement)` | `SUBSTR(s,1,start-1) \|\| replacement \|\| SUBSTR(s,start+len)` |
| `SPACE(n)` | `RPAD(' ', n)` |
| `REPLICATE(s, n)` | カスタム PL/SQL 関数、または `RPAD(s, n*LENGTH(s), s)` |
| `REVERSE(s)` | カスタム PL/SQL 関数 |
| `STR(n, len, dec)` | `TO_CHAR(n, RPAD('9', len-dec-1, '9') \|\| '.' \|\| RPAD('0', dec, '0'))` |
| `ASCII(s)` | `ASCII(s)` — 同様 |
| `CHAR(n)` | `CHR(n)` |
| `SOUNDEX(s)` | `SOUNDEX(s)` — 同様 |
| `DIFFERENCE(s1, s2)` | 同等の関数なし。SOUNDEX 値を手動で比較 |

### 日付関数

| Sybase 関数 | Oracle 同等物 |
|---|---|
| `GETDATE()` | `SYSDATE` |
| `GETUTCDATE()` | `SYS_EXTRACT_UTC(SYSTIMESTAMP)` |
| `DATEADD(unit, n, d)` | `d + n` (日数), `ADD_MONTHS(d, n)` (月数) など |
| `DATEDIFF(unit, d1, d2)` | `d2 - d1` (日数), `MONTHS_BETWEEN(d2, d1)` (月数) |
| `DATEPART(unit, d)` | `EXTRACT(unit FROM d)` |
| `DATENAME(unit, d)` | `TO_CHAR(d, 'MONTH')` など |
| `DAY(d)` | `EXTRACT(DAY FROM d)` |
| `MONTH(d)` | `EXTRACT(MONTH FROM d)` |
| `YEAR(d)` | `EXTRACT(YEAR FROM d)` |
| `CONVERT(type, d, style)` | `TO_DATE`, `TO_CHAR`, `TO_NUMBER` |

---

## ベスト・プラクティス

1. **移行前に Sybase のトランザクション・モードを確認する。** Sybase で `SELECT @@TRANCHAINED` を実行し、アプリケーションが chained (1) または unchained (0) モードのどちらを使用しているかを確認すること。unchained の場合、Oracle では明示的なトランザクション管理を追加する必要がある。

2. **@@ERROR チェックをすべて監査する。** すべてのステートメントの後に `@@ERROR` をチェックする Sybase コードは冗長で壊れやすい。Oracle PL/SQL の例外処理ブロックの方が洗練されているため、移行時に @@ERROR パターンをそのまま再現するのではなく、エラー処理をリファクタリングすること。

3. **IDENTITY 列の移行を慎重にテストする。** Sybase の IDENTITY 列は Oracle のものと似ているが、開始値とステップ値のオプションが異なる。データ移行後、Oracle のアイデンティティ列の開始値が既存 ID の最大値より大きくなるように設定すること。

4. **TEXT および IMAGE 列を特別に処理する。** Sybase の TEXT/IMAGE LOB は、BCP での特別な処理が必要であり、通常の列とは格納方法が異なる。文字形式の BCP エクスポートを使用し、Oracle の CLOB/BLOB にインポートする前に LOB コンテンツのサイズを検証すること。

5. **Sybase のルール（Rules）とデフォルト（Defaults）をマッピングする。** Sybase には列やユーザー定義型にバインドされる `CREATE RULE` および `CREATE DEFAULT` オブジェクトがある。これらはそれぞれ Oracle の CHECK 制約および列の DEFAULT 値に変換する。

```sql
-- Sybase ルール (独立したオブジェクト)
CREATE RULE salary_rule AS @salary >= 0;
EXEC sp_bindrule 'salary_rule', 'employees.salary';

-- Oracle CHECK 制約 (インライン)
CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    salary NUMBER(10,2) CHECK (salary >= 0)
);
```

6. **Sybase のユーザー定義データ型 (UDT) を見直す。** Sybase は基本型にルールやデフォルトを組み合わせたユーザー定義型をサポートしている。Oracle ではドメイン (23c) を使用するか、単純に制約付きの基本型を使用する。

```sql
-- Sybase UDT
EXEC sp_addtype 'phone_num', 'varchar(15)', 'NOT NULL';
CREATE RULE phone_rule AS @p LIKE '[0-9][0-9][0-9]-[0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]';
EXEC sp_bindrule phone_rule, phone_num;

-- Oracle: インライン制約
CREATE TABLE contacts (
    phone VARCHAR2(15) NOT NULL
                       CHECK (REGEXP_LIKE(phone, '^\d{3}-\d{3}-\d{4}$'))
);
```

---

## よくある移行の落とし穴

**落とし穴 1 — Sybase unchained のオートコミット挙動：**
挙動の相違として最も一般的な原因である。Sybase unchained モードでは各 DML がオートコミットされるが、Oracle では一切オートコミットされない。UPDATE の直後に（他のセッションから見えるように）コミットされていることを前提としたアプリケーションは、動作が異なることになる。

**落とし穴 2 — RAISERROR 構文：**
Sybase の `RAISERROR` は、エラー番号が最初で、次にメッセージが来る。Oracle の `RAISE_APPLICATION_ERROR` は負の番号とメッセージ文字列を受け取る。また、Oracle で定義できるユーザー・エラー番号の範囲は -20000 から -20999 である。
```sql
-- Sybase
RAISERROR 20001 "Record not found";

-- Oracle
RAISE_APPLICATION_ERROR(-20001, 'Record not found');
```

**落とし穴 3 — 行バージョンとしての TIMESTAMP：**
型マッピングで述べたように、Sybase の `TIMESTAMP` は日時ではない。この列を日付として扱う Oracle コードは失敗するため、楽観的排他制御のパターンを再設計すること。

**落とし穴 4 — 副クエリ結果の比較：**
Sybase は一部の非標準的な副クエリ比較を許可するが、Oracle は厳格な副クエリ・セマンティクスを強制する。`= (副クエリ)` パターンが、常に正確に 1 行を返すことを確認するか、適切に修正すること。
```sql
-- Sybase ではリスクあり (複数行ある場合に最初の行を黙って返す)
WHERE emp_id = (SELECT emp_id FROM employees WHERE dept = 'IT');

-- Oracle: 副クエリが 2 行以上返すと ORA-01427 が発生
-- 修正：IN を使用する
WHERE emp_id IN (SELECT emp_id FROM employees WHERE dept = 'IT');
```

**落とし穴 5 — Sybase のロック・エスカレーション：**
Sybase にはページ・レベルおよび行レベルのロック・モードがある。Oracle は MVCC と行レベル・ロックを排他的に使用する。Sybase のロック挙動（特に未コミット・データの読み取り）を中心とした設計のアプリケーションは、MVCC への見直しが必要である。

**落とし穴 6 — 集計関数における NULL 処理：**
Sybase と Oracle は共に、ANSI SQL に従って集計関数 (COUNT, SUM 等) で NULL を無視する。これは互換性があるが、Sybase の `ISNULL()` 関数は Oracle の `NVL()` または `COALESCE()` に置き換える必要がある。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c PL/SQL Language Reference — PL/SQL Language Fundamentals](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-language-fundamentals.html)
- [Oracle Database 19c PL/SQL Language Reference — Error Handling (RAISE_APPLICATION_ERROR)](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-error-handling.html)
- [Oracle Database 19c SQL Language Reference — CREATE GLOBAL TEMPORARY TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [AWS Schema Conversion Tool User Guide — SAP ASE to Oracle](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)
