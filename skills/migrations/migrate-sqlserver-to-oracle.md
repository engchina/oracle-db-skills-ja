# SQL Server から Oracle への移行

## 概要

Microsoft SQL Server と Oracle は、エンタープライズ環境で最も一般的に導入されているリレーショナル・データベースであり、それらの間の移行は業界で最も頻繁に行われるケースの 1 つである。どちらも標準 SQL を幅広くサポートする成熟した ACID 準拠の RDBMS エンジンであるが、手続き型言語 (T-SQL vs PL/SQL)、トランザクション管理、DDL の動作、データ型の命名、および管理概念において劇的に異なっている。本ガイドでは、SQL Server から Oracle への移行に関する主要な変換要件をすべて網羅する。

Microsoft は **AWS Schema Conversion Tool (SCT)** を提供しており、Oracle は **SQL Developer Migration Workbench** を提供することで、これらの作業の多くを自動化している。注：「SSMA for Oracle」(Microsoft の製品) は Oracle から SQL Server へ（逆方向）移行するためのツールである。SQL Server から Oracle への移行には、AWS SCT または SQL Developer Migration Workbench を使用すること。本ガイドでは、これらのツールが自動的に処理する内容と、手動での介入が必要な内容について説明する。

---

## T-SQL から PL/SQL への変換

### バッチ構造

```sql
-- T-SQL: GO がバッチを区切る
CREATE TABLE departments (dept_id INT, dept_name VARCHAR(100));
GO
INSERT INTO departments VALUES (1, 'Engineering');
GO

-- Oracle: スラッシュ (/) が PL/SQL ブロックの終了を示す。DDL はスタンドアロンで動作
CREATE TABLE departments (dept_id NUMBER(10), dept_name VARCHAR2(100));
/
INSERT INTO departments VALUES (1, 'Engineering');
COMMIT;
/
```

### 変数の宣言と代入

```sql
-- T-SQL
DECLARE @employee_count INT = 0;
DECLARE @dept_name NVARCHAR(100);
SET @employee_count = 10;
SELECT @dept_name = dept_name FROM departments WHERE dept_id = 1;

-- Oracle PL/SQL
DECLARE
    v_employee_count NUMBER(10) := 0;
    v_dept_name VARCHAR2(100);
BEGIN
    v_employee_count := 10;
    SELECT dept_name INTO v_dept_name FROM departments WHERE dept_id = 1;
END;
/
```

### IF / BEGIN-END → IF / THEN / END IF

```sql
-- T-SQL
IF @salary > 100000
BEGIN
    UPDATE employees SET bonus = salary * 0.15 WHERE emp_id = @emp_id;
    PRINT 'High-earner bonus applied';
END
ELSE IF @salary > 50000
BEGIN
    UPDATE employees SET bonus = salary * 0.10 WHERE emp_id = @emp_id;
END
ELSE
BEGIN
    UPDATE employees SET bonus = salary * 0.05 WHERE emp_id = @emp_id;
END;

-- Oracle PL/SQL
IF v_salary > 100000 THEN
    UPDATE employees SET bonus = salary * 0.15 WHERE emp_id = v_emp_id;
    DBMS_OUTPUT.PUT_LINE('High-earner bonus applied');
ELSIF v_salary > 50000 THEN
    UPDATE employees SET bonus = salary * 0.10 WHERE emp_id = v_emp_id;
ELSE
    UPDATE employees SET bonus = salary * 0.05 WHERE emp_id = v_emp_id;
END IF;
```

### TRY-CATCH → EXCEPTION

```sql
-- T-SQL
BEGIN TRY
    INSERT INTO accounts (account_id, balance) VALUES (1001, 500.00);
    UPDATE accounts SET balance = balance - 200 WHERE account_id = 1001;
    COMMIT;
END TRY
BEGIN CATCH
    ROLLBACK;
    PRINT 'Error: ' + ERROR_MESSAGE();
    RAISERROR(ERROR_MESSAGE(), 16, 1);
END CATCH;

-- Oracle PL/SQL
BEGIN
    INSERT INTO accounts (account_id, balance) VALUES (1001, 500.00);
    UPDATE accounts SET balance = balance - 200 WHERE account_id = 1001;
    COMMIT;
EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20001, 'Duplicate account ID');
    WHEN OTHERS THEN
        ROLLBACK;
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        RAISE;
END;
/
```

### PRINT → DBMS_OUTPUT.PUT_LINE

```sql
-- T-SQL
PRINT 'Processing record: ' + CAST(@id AS VARCHAR(10));

-- Oracle PL/SQL
DBMS_OUTPUT.PUT_LINE('Processing record: ' || TO_CHAR(v_id));

-- SQL*Plus または SQL Developer で出力を有効にする：
SET SERVEROUTPUT ON SIZE UNLIMITED;
```

### WHILE ループ

```sql
-- T-SQL
DECLARE @i INT = 1;
WHILE @i <= 10
BEGIN
    INSERT INTO audit_log (seq_num) VALUES (@i);
    SET @i = @i + 1;
END;

-- Oracle PL/SQL
DECLARE
    v_i NUMBER := 1;
BEGIN
    WHILE v_i <= 10 LOOP
        INSERT INTO audit_log (seq_num) VALUES (v_i);
        v_i := v_i + 1;
    END LOOP;
END;
/
```

### ストアド・プロシージャの構造

```sql
-- T-SQL ストアド・プロシージャ
CREATE PROCEDURE usp_get_customer_orders
    @customer_id   INT,
    @start_date    DATE,
    @order_count   INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT order_id, order_date, total_amount
    FROM orders
    WHERE customer_id = @customer_id
      AND order_date >= @start_date;

    SELECT @order_count = COUNT(*)
    FROM orders
    WHERE customer_id = @customer_id
      AND order_date >= @start_date;
END;
GO

-- Oracle PL/SQL プロシージャ
-- 注：Oracle のプロシージャは SYS_REFCURSOR を介して結果セットを返す
CREATE OR REPLACE PROCEDURE usp_get_customer_orders(
    p_customer_id IN  NUMBER,
    p_start_date  IN  DATE,
    p_order_count OUT NUMBER,
    p_result      OUT SYS_REFCURSOR
)
AS
BEGIN
    OPEN p_result FOR
        SELECT order_id, order_date, total_amount
        FROM orders
        WHERE customer_id = p_customer_id
          AND order_date >= p_start_date;

    SELECT COUNT(*) INTO p_order_count
    FROM orders
    WHERE customer_id = p_customer_id
      AND order_date >= p_start_date;
END usp_get_customer_orders;
/
```

---

## アイデンティティ列

### SQL Server IDENTITY → Oracle

```sql
-- T-SQL
CREATE TABLE products (
    product_id   INT          IDENTITY(1,1) PRIMARY KEY,
    product_name NVARCHAR(200) NOT NULL,
    price        DECIMAL(10,2)
);

-- Oracle 12c 以降 (アイデンティティ列 — 推奨)
CREATE TABLE products (
    product_id   NUMBER       GENERATED ALWAYS AS IDENTITY (START WITH 1 INCREMENT BY 1) PRIMARY KEY,
    product_name VARCHAR2(200 CHAR) NOT NULL,
    price        NUMBER(10,2)
);

-- Oracle — シーケンス・ベース (12c 未満または明示的な制御が必要な場合)
CREATE SEQUENCE products_seq START WITH 1 INCREMENT BY 1 CACHE 20;
CREATE TABLE products (
    product_id   NUMBER       DEFAULT products_seq.NEXTVAL PRIMARY KEY,
    product_name VARCHAR2(200 CHAR) NOT NULL,
    price        NUMBER(10,2)
);
```

### 生成された最新 ID の取得

```sql
-- T-SQL
INSERT INTO products (product_name, price) VALUES ('Widget A', 9.99);
SELECT SCOPE_IDENTITY() AS new_id;
-- または：
SELECT @@IDENTITY;

-- Oracle (PL/SQL における RETURNING INTO)
DECLARE
    v_new_id products.product_id%TYPE;
BEGIN
    INSERT INTO products (product_name, price) VALUES ('Widget A', 9.99)
    RETURNING product_id INTO v_new_id;
    DBMS_OUTPUT.PUT_LINE('New product ID: ' || v_new_id);
END;
/
```

---

## TOP N → FETCH FIRST

```sql
-- T-SQL
SELECT TOP 10 * FROM products ORDER BY price DESC;
SELECT TOP 10 PERCENT * FROM products ORDER BY price DESC;

-- Oracle 12c 以降
SELECT * FROM products ORDER BY price DESC FETCH FIRST 10 ROWS ONLY;

-- TOP PERCENT の直接的な同等物はないため、件数を計算するか以下を使用する：
SELECT * FROM (
    SELECT p.*, NTILE(10) OVER (ORDER BY price DESC) AS pct_group
    FROM products p
)
WHERE pct_group = 1;  -- およそ上位 10%
```

---

## データ型マッピング

### 文字列型

| SQL Server | Oracle | 備考 |
|---|---|---|
| `CHAR(n)` | `CHAR(n)` | SS は最大 8,000 バイト、Oracle は 2,000 バイト |
| `VARCHAR(n)` | `VARCHAR2(n)` | SS は最大 8,000 バイト、Oracle は 4,000 (拡張した場合 32,767) バイト |
| `VARCHAR(MAX)` | `CLOB` | SS は最大 2 GB、Oracle の CLOB は最大 128 TB |
| `NCHAR(n)` | `NCHAR(n)` | Unicode 固定長 |
| `NVARCHAR(n)` | `NVARCHAR2(n)` | Unicode 可変長 |
| `NVARCHAR(MAX)` | `NCLOB` | Unicode LOB |
| `TEXT` (非推奨) | `CLOB` | |
| `NTEXT` (非推奨) | `NCLOB` | |
| `XML` | `XMLTYPE` | Oracle のネイティブ XML 型 |

**NVARCHAR と文字セット：**
Oracle のデータベース文字セットが AL32UTF8 である場合、`VARCHAR2` はすでにすべての Unicode 文字を処理できるため、`NVARCHAR2` は冗長である。AL32UTF8 データベースでは、すべての文字データに `VARCHAR2` を使用することを推奨する。

```sql
-- SQL Server (NVARCHAR によるマルチバイト Unicode)
CREATE TABLE messages (
    message_id   INT          IDENTITY PRIMARY KEY,
    content      NVARCHAR(MAX)
);

-- Oracle (AL32UTF8 データベース — VARCHAR2 がすべての Unicode を処理)
CREATE TABLE messages (
    message_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    content    CLOB
);
```

### 数値型

| SQL Server | Oracle | 備考 |
|---|---|---|
| `TINYINT` | `NUMBER(3)` | 0 ～ 255 |
| `SMALLINT` | `NUMBER(5)` | −32,768 ～ 32,767 |
| `INT` | `NUMBER(10)` | |
| `BIGINT` | `NUMBER(19)` | |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `MONEY` | `NUMBER(19,4)` | SS の 8 バイト通貨型 |
| `SMALLMONEY` | `NUMBER(10,4)` | |
| `FLOAT(n)` | `BINARY_FLOAT` または `BINARY_DOUBLE` | |
| `REAL` | `BINARY_FLOAT` | IEEE 754 32 ビット |
| `BIT` | CHECK (0,1) 付きの `NUMBER(1)` | 論理ビット型 |

### 日付と時間型

SQL Server の日付/時刻処理は Oracle よりも粒度が細かく、豊富である。これは翻訳において最も複雑な領域の 1 つである。

| SQL Server | Oracle | 備考 |
|---|---|---|
| `DATE` | `DATE` | Oracle の DATE は時刻を含む。SS の DATE は日付のみ |
| `TIME(n)` | 同等の型なし | `VARCHAR2(12)` または `NUMBER` (真夜中からの秒の小数部) として格納 |
| `DATETIME` | `DATE` または `TIMESTAMP` | SS の精度：1/300 秒 |
| `DATETIME2(n)` | `TIMESTAMP(n)` | SS は最大 小数点以下 7 桁。Oracle は 9 桁 |
| `SMALLDATETIME` | `DATE` | 1 分単位の精度 |
| `DATETIMEOFFSET(n)` | `TIMESTAMP(n) WITH TIME ZONE` | タイムゾーン・オフセットを保持 |

**GETDATE() および関連関数：**

```sql
-- SQL Server
SELECT GETDATE();            -- 現在の日時
SELECT GETUTCDATE();         -- UTC の日時
SELECT SYSDATETIMEOFFSET(); -- TZ オフセット付きの日時
SELECT YEAR(GETDATE());
SELECT MONTH(GETDATE());
SELECT DAY(GETDATE());
SELECT DATEPART(weekday, GETDATE());
SELECT DATEADD(month, 3, GETDATE());
SELECT DATEDIFF(day, '2024-01-01', GETDATE());
SELECT FORMAT(GETDATE(), 'yyyy-MM-dd');

-- Oracle
SELECT SYSDATE FROM DUAL;
SELECT SYS_EXTRACT_UTC(SYSTIMESTAMP) FROM DUAL;
SELECT SYSTIMESTAMP FROM DUAL;
SELECT EXTRACT(YEAR FROM SYSDATE) FROM DUAL;
SELECT EXTRACT(MONTH FROM SYSDATE) FROM DUAL;
SELECT EXTRACT(DAY FROM SYSDATE) FROM DUAL;
SELECT TO_NUMBER(TO_CHAR(SYSDATE, 'D')) FROM DUAL;
SELECT ADD_MONTHS(SYSDATE, 3) FROM DUAL;
SELECT TRUNC(SYSDATE) - DATE '2024-01-01' FROM DUAL;
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD') FROM DUAL;
```

### その他の型

| SQL Server | Oracle | 備考 |
|---|---|---|
| `UNIQUEIDENTIFIER` | `RAW(16)` または `VARCHAR2(36)` | Oracle では `SYS_GUID()` で生成 |
| `BINARY(n)` | `RAW(n)` | 固定長バイナリ |
| `VARBINARY(n)` | `RAW(n)` | 可変長。Oracle では最大 2,000 バイト |
| `VARBINARY(MAX)` | `BLOB` | |
| `IMAGE` (非推奨) | `BLOB` | |
| `ROWVERSION` / `TIMESTAMP` | 同等の型なし | 楽観的排他制御用。Oracle の `ORA_ROWSCN` を使用するか `VERSION_NUM` 列を追加 |
| `SQL_VARIANT` | 同等の型なし | 型指定された列を使用するように再設計が必要 |
| `HIERARCHYID` | 同等の型なし | Oracle の入れ子集合（nested set）や隣接リスト・パターンを使用 |
| `GEOGRAPHY` / `GEOMETRY` | `SDO_GEOMETRY` | Oracle Spatial |
| `JSON` (SS 2016 以降は NVARCHAR として格納) | `JSON` (21c 以降) または `CLOB IS JSON` | |

---

## リンク サーバー → データベース リンク

SQL Server はリモート・データベースへのクエリにリンク サーバーを使用する。Oracle はデータベース・リンクを使用する。

```sql
-- SQL Server: リンク サーバー経由のクエリ
SELECT * FROM [LinkedServer].[RemoteDB].[dbo].[orders];
INSERT INTO [LinkedServer].[RemoteDB].[dbo].[archive_orders]
SELECT * FROM orders WHERE order_date < '2020-01-01';

-- Oracle: データベース・リンクの作成と使用
CREATE DATABASE LINK remote_db_link
    CONNECT TO remote_user IDENTIFIED BY password
    USING 'remote_tns_alias';

-- データベース・リンク経由のクエリ
SELECT * FROM orders@remote_db_link;
INSERT INTO archive_orders@remote_db_link
SELECT * FROM orders WHERE order_date < DATE '2020-01-01';
```

---

## AWS SCT による SQL Server から Oracle への移行手順

AWS Schema Conversion Tool (SCT) は、SQL Server から Oracle へのスキーマ変換を自動化し、移行の複雑さを評価できる。

> ⚠️ 注：「SSMA for Oracle」(Microsoft の製品) は Oracle から SQL Server へ（逆方向）移行するためのツールである。SQL Server から Oracle への移行には、**AWS SCT** または **SQL Developer Migration Workbench** を使用すること。

### AWS SCT ワークフロー (SQL Server → Oracle)

1. **AWS SCT のインストール**：AWS のダウンロード・ページから入手し、デスクトップ・アプリケーションとして実行する。

2. **移行プロジェクトの作成**：
   - File → New Project を選択
   - ソース：SQL Server、ターゲット：Oracle を選択
   - 接続情報を入力

3. **評価レポートの実行**：
   - View → Assessment Report を選択
   - 変換カテゴリ（自動変換可能、注意が必要、変換不可）を確認する。

4. **スキーマの変換**：
   - 左ペインでオブジェクトを選択
   - 右クリック → Convert Schema を選択
   - Output および Error List ペインで警告を確認し、解決する。

5. **同期前の手動修正**：
   - SCT は自動変換できないオブジェクトに対して変換ノートを生成する。
   - 一般的な手動項目：動的 SQL、データベースをまたがる参照、フルテキスト検索、CLR オブジェクト。

6. **Oracle ターゲットへの適用**：
   - 変換されたスキーマを右クリック → Apply to Database を選択。

7. **データの移行**：
   - SCT はデータの移行は行わないため、一括データ転送には SQL*Loader または Oracle Data Pump を使用する。

8. **テストと検証**：
   - 行数の一致確認（リコンシリエーション・クエリ）を実行。
   - アプリケーションのリグレッション・テストを実施。

### AWS SCT が自動変換するもの

- 表定義（ほとんどのデータ型）
- 索引と制約
- ビュー
- 単純なストアド・プロシージャと関数
- 基本的な T-SQL 制御フロー

### AWS SCT が変換できないもの (手動作業が必要)

- 複雑な文字列操作を含む動的 SQL
- CLR (共通言語ランタイム) オブジェクト
- フルテキスト検索索引 → Oracle Text への変換が必要
- SQL Server Agent ジョブ → Oracle Scheduler への変換が必要
- リンク サーバー クエリ → データベース リンクへの変換が必要
- `OPENROWSET` / `OPENQUERY`
- `FOR XML` 句 → Oracle の XMLGEN 同等物への変換が必要
- `PIVOT` / `UNPIVOT` — 条件付き集計への変換を検討
- `MERGE` ステートメントの相違（共にサポートするが構文が異なる）

---

## 日時処理の相違

### スタイル・コード付きの CONVERT

```sql
-- SQL Server CONVERT スタイル・コード
SELECT CONVERT(VARCHAR(10), GETDATE(), 101);  -- MM/DD/YYYY
SELECT CONVERT(VARCHAR(10), GETDATE(), 103);  -- DD/MM/YYYY
SELECT CONVERT(VARCHAR(10), GETDATE(), 112);  -- YYYYMMDD
SELECT CONVERT(DATETIME, '2024-01-15', 120);

-- Oracle TO_CHAR / TO_DATE の同等物
SELECT TO_CHAR(SYSDATE, 'MM/DD/YYYY') FROM DUAL;
SELECT TO_CHAR(SYSDATE, 'DD/MM/YYYY') FROM DUAL;
SELECT TO_CHAR(SYSDATE, 'YYYYMMDD') FROM DUAL;
SELECT TO_DATE('2024-01-15', 'YYYY-MM-DD') FROM DUAL;
```

### 日付に対する文字列関数

```sql
-- SQL Server
SELECT CAST('2024-01-15' AS DATE);
SELECT CAST(GETDATE() AS VARCHAR(20));

-- Oracle
SELECT TO_DATE('2024-01-15', 'YYYY-MM-DD') FROM DUAL;
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS') FROM DUAL;
```

---

## ベスト・プラクティス

1. **AWS SCT 評価レポートから開始する。** 移行スクリプトを書き始める前に、AWS SCT の評価を実行して、オブジェクト・カテゴリごとの変換の複雑さスコアを取得し、現実的な計画を立てること。

2. **AL32UTF8 データベースでは NVARCHAR を VARCHAR2 に正規化する。** AL32UTF8 では、各国語文字型を使用せずに Unicode サポートが組み込まれている。

3. **T-SQL コード内のすべての SET オプションを確認する。** SQL Server のプロシージャでは `SET NOCOUNT ON`, `SET XACT_ABORT ON` などが頻繁に使用されるが、Oracle PL/SQL にはこれらはないため、個別に同等の処理を検討するか削除する。

4. **tempdb の使用を処理する。** SQL Server は一時テーブル (`#temp`, `##global_temp`) やテーブル変数 (`@TableVar`) を多用する。Oracle での同等物はグローバル一時表 (GTT) または PL/SQL コレクションである。GTT は事前定義が必要である。

```sql
-- SQL Server 一時テーブル
CREATE TABLE #staging (id INT, value VARCHAR(100));

-- Oracle グローバル一時表 (GTT)
CREATE TABLE staging (
    id    NUMBER,
    value VARCHAR2(100)
)
GLOBAL TEMPORARY
ON COMMIT DELETE ROWS;  -- または ON COMMIT PRESERVE ROWS
```

5. **照合順序（Collation）に依存する比較をテストする。** SQL Server はデータベースや列ごとに大文字小文字/アクセントの区別を設定できる。Oracle の比較はデフォルトで常に区別される。ロケールを考慮したソートには `NLS_SORT` および `NLS_COMP` パラメータを使用すること。

6. **NOLOCK ヒントを置き換える。** `WITH (NOLOCK)` は不正確な読み取り（ダーティ・リード）を許容する一般的なヒントである。Oracle はマルチバージョン一貫性制御 (MVCC) を使用しており、ダーティ・リードを必要としないため、すべての NOLOCK ヒントを削除してもクエリはブロックされない。

---

## よくある移行の落とし穴

**落とし穴 1 — NULL + 空文字の挙動：**
```sql
-- SQL Server: NULL + '' = NULL だが、'' は NULL とは異なる
-- Oracle 21c 以前: '' IS NULL (空文字は NULL として格納される)
-- 空文字が意味のある値として使用されている列を確認すること
```

**落とし穴 2 — トランザクション・アイソレーション：**
SQL Server はロックを伴う READ COMMITTED がデフォルトである。Oracle は MVCC を伴う READ COMMITTED がデフォルトである。NOLOCK ヒントでブロッキングを回避していたアプリケーションは、Oracle の MVCC 下では挙動が異なる場合があるため、十分にテストすること。

**落とし穴 3 — 暗黙的なトランザクション：**
SQL Server はデフォルトでオートコミットされるが、Oracle では常に明示的な `COMMIT` または `ROLLBACK` が必要である。SQL Server のオートコミット動作に依存しているアプリケーション・コードの確認が必要である。

**落とし穴 4 — MERGE ステートメントの違い：**
両者とも `MERGE` をサポートしているが、構文が異なる。SQL Server の MERGE はセミコロンによる終了が必須であり、`MATCHED`/`NOT MATCHED` 句の構造も若干異なる。すべての MERGE ステートメントを個別に確認すること。

**落とし穴 5 — スキーマ vs データベース：**
SQL Server では 1 つの「データベース」内に複数の「スキーマ」が含まれ、`MyDB.dbo.MyTable` のようにデータベースをまたいでの JOIN が可能である。Oracle では「スキーマ」は「ユーザー」そのものであり、データベースをまたがるアクセスにはデータベース・リンクが必要である。

**落とし穴 6 — 文字列比較における大文字小文字の区別：**
デフォルト設定の SQL Server は大文字小文字を区別しないことが多いが、Oracle は常に区別する。大文字小文字を区別しない一致を前提としたクエリには、`UPPER(col)` によるラップとそれに対応する関数ベース索引が必要になる。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c PL/SQL Language Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/)
- [Oracle Database 19c SQL Language Reference — PL/SQL vs T-SQL reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [Oracle SQL Developer Migration documentation](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
- [AWS Schema Conversion Tool — SQL Server to Oracle](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Source.SQLServer.html)
