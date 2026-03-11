# IBM DB2 から Oracle への移行

## 概要

IBM DB2 (現在は IBM Db2) と Oracle は、共にエンタープライズ RDBMS プラットフォームとして長い歴史を共有している。この 2 つは他のデータベースの組み合わせよりも共通点が多く、どちらも ANSI 標準 SQL を使用し、PL/SQL スタイルの手続き型拡張（DB2 では SQL PL）をサポートし、成熟したオプティマイザ技術を持ち、XML、パーティショニング、マテリアライズド・ビューなどの高度な機能をサポートしている。この共通の基盤により、DB2 から Oracle への移行は、MySQL や SQL Server からの移行よりも予測しやすいものとなるが、それでも SQL 方言、データ型、管理概念、および手続き型言語には意味のある違いが存在する。

---

## DB2 SQL 方言の違い

### FROM なしの SELECT

DB2 は (PostgreSQL と同様に) `FROM` なしの `SELECT` を許可する。

```sql
-- DB2
SELECT CURRENT DATE;
SELECT CURRENT TIMESTAMP;
SELECT 1 + 1;

-- Oracle は FROM DUAL が必須
SELECT CURRENT_DATE FROM DUAL;
SELECT CURRENT_TIMESTAMP FROM DUAL;
SELECT 1 + 1 FROM DUAL;
```

### CURRENT DATE / CURRENT TIMESTAMP (互換性)

DB2 と Oracle は共に ANSI 標準の `CURRENT_DATE` および `CURRENT_TIMESTAMP` をサポートしているが、DB2 は Oracle が受け付けないスペース入りの形式 (`CURRENT DATE`) を使用する。

```sql
-- DB2 (CURRENT と修飾子の間にスペースが可能)
SELECT CURRENT DATE FROM SYSIBM.SYSDUMMY1;
SELECT CURRENT TIMESTAMP FROM SYSIBM.SYSDUMMY1;
SELECT CURRENT TIME FROM SYSIBM.SYSDUMMY1;

-- Oracle
SELECT CURRENT_DATE FROM DUAL;          -- スペースではなくアンダースコア
SELECT CURRENT_TIMESTAMP FROM DUAL;
SELECT TO_CHAR(SYSDATE, 'HH24:MI:SS') FROM DUAL;  -- TIME に相当する型はない
```

### SYSIBM.SYSDUMMY1 vs DUAL

DB2 は単一の値を返すクエリに `SYSIBM.SYSDUMMY1` (または `SYSIBM.DUAL`) を使用する。Oracle は `DUAL` を使用する。

```sql
-- DB2
SELECT 'hello' FROM SYSIBM.SYSDUMMY1;

-- Oracle
SELECT 'hello' FROM DUAL;
```

### FETCH FIRST n ROWS ONLY (互換性)

この領域は互換性がある。DB2 が Oracle よりも先に `FETCH FIRST n ROWS ONLY` を導入し、Oracle は 12c で同じ ANSI 標準構文を採用した。

```sql
-- DB2 (元の構文)
SELECT * FROM employees ORDER BY salary DESC FETCH FIRST 10 ROWS ONLY;

-- Oracle 12c 以降 (同じ構文)
SELECT * FROM employees ORDER BY salary DESC FETCH FIRST 10 ROWS ONLY;
-- Oracle では以下も有効：
SELECT * FROM employees ORDER BY salary DESC FETCH NEXT 10 ROWS ONLY;
```

### OFFSET 付きの FETCH FIRST

```sql
-- DB2
SELECT * FROM employees ORDER BY emp_id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle 12c 以降 (同一)
SELECT * FROM employees ORDER BY emp_id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### WITH UR / CS / RR 分離レベルのヒント

DB2 はインラインで分離レベルのヒントをサポートしている。Oracle は MVCC を使用しており、同等のヒントはない。

```sql
-- DB2 分離レベル・ヒント
SELECT * FROM orders WITH UR;   -- 非コミット読取り (ダーティ・リード)
SELECT * FROM orders WITH CS;   -- カーソル固定 (デフォルト)
SELECT * FROM orders WITH RS;   -- 読取り固定
SELECT * FROM orders WITH RR;   -- 反復読取り

-- Oracle: すべての WITH 分離ヒントを削除する
-- Oracle の MVCC により、読取りがブロックされることはなく、常に一貫性が保たれる
SELECT * FROM orders;
```

### 特殊レジスタ (SPECIAL REGISTERS)

DB2 の特殊レジスタは Oracle の同等物にマップされる。

| DB2 特殊レジスタ | Oracle 同等物 |
|---|---|
| `CURRENT DATE` | `CURRENT_DATE` または `TRUNC(SYSDATE)` |
| `CURRENT TIME` | `TO_CHAR(SYSDATE, 'HH24:MI:SS')` |
| `CURRENT TIMESTAMP` | `CURRENT_TIMESTAMP` または `SYSTIMESTAMP` |
| `CURRENT USER` | `USER` または `SYS_CONTEXT('USERENV', 'SESSION_USER')` |
| `CURRENT SCHEMA` | `SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA')` |
| `CURRENT SERVER` | `SYS_CONTEXT('USERENV', 'DB_NAME')` |
| `SESSION_USER` | `USER` |
| `CURRENT TIMEZONE` | `SESSIONTIMEZONE` |

---

## データ型マッピング

### 数値型

| DB2 | Oracle | 備考 |
|---|---|---|
| `SMALLINT` | `NUMBER(5)` または `SMALLINT` | |
| `INTEGER` / `INT` | `NUMBER(10)` または `INTEGER` | Oracle の INTEGER は NUMBER(38) に相当 |
| `BIGINT` | `NUMBER(19)` | |
| `DECIMAL(p,s)` / `DEC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `NUMERIC(p,s)` / `NUM(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `REAL` | `BINARY_FLOAT` | IEEE 754 32 ビット |
| `DOUBLE` / `FLOAT(n)` | `BINARY_DOUBLE` | IEEE 754 64 ビット |
| `DECFLOAT(16)` | `BINARY_FLOAT` (近似) | DB2 の 10 進浮動小数点型 |
| `DECFLOAT(34)` | `BINARY_DOUBLE` (近似) | |

### 文字列型

| DB2 | Oracle | 備考 |
|---|---|---|
| `CHAR(n)` | `CHAR(n)` | DB2 最大 254 バイト、Oracle 最大 2,000 バイト |
| `VARCHAR(n)` | `VARCHAR2(n)` | DB2 最大 32,672 バイト、Oracle 最大 4,000/32,767 |
| `LONG VARCHAR` | `CLOB` | DB2 で非推奨。CLOB を使用 |
| `CLOB(n)` | `CLOB` | DB2 は最大サイズを指定、Oracle の CLOB は最大 128 TB |
| `DBCLOB(n)` | `NCLOB` | 2 バイト文字用 CLOB |
| `GRAPHIC(n)` | `NCHAR(n)` | 固定長 2 バイト文字 |
| `VARGRAPHIC(n)` | `NVARCHAR2(n)` | 可変長 2 バイト文字 |

### バイナリ型

| DB2 | Oracle | 備考 |
|---|---|---|
| `BINARY(n)` | `RAW(n)` | 固定長バイナリ (DB2 10.5 以降) |
| `VARBINARY(n)` | `RAW(n)` | 可変長バイナリ |
| `BLOB(n)` | `BLOB` | バイナリ・ラージ・オブジェクト |
| `FOR BIT DATA` (サフィックス) | `RAW(n)` または `BLOB` | バイナリ文字データ用の DB2 修飾子 |

### 日付 / 時刻型

| DB2 | Oracle | 備考 |
|---|---|---|
| `DATE` | `DATE` | DB2 の DATE は日付のみ、Oracle の DATE は時刻も含む |
| `TIME` | 同等の型なし | `VARCHAR2(8)` または `NUMBER` を使用 |
| `TIMESTAMP(n)` | `TIMESTAMP(n)` | 共に秒の小数部をサポート |
| `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP WITH TIME ZONE` | 互換性あり |
| `TIMESTAMP WITH LOCAL TIME ZONE` | `TIMESTAMP WITH LOCAL TIME ZONE` | 互換性あり |

**重要：** DB2 の `DATE` は日付のみ（時刻なし）を保持する。Oracle の `DATE` は時刻が含まれる。DB2 の DATE 列をマップする場合：

```sql
-- DB2: この比較は安全
WHERE order_date = DATE '2024-01-15'

-- Oracle: 真夜中以外の時刻が含まれる行を見逃す可能性がある
WHERE order_date = DATE '2024-01-15'

-- Oracle: 安全な同等実装
WHERE TRUNC(order_date) = DATE '2024-01-15'
-- あるいは時刻が含まれていないことが確実な場合：
WHERE order_date >= DATE '2024-01-15' AND order_date < DATE '2024-01-16'
```

### XML とその他の型

| DB2 | Oracle | 備考 |
|---|---|---|
| `XML` | `XMLTYPE` | Oracle は XMLTYPE を使用。同等の XML/SQL 機能が利用可能 |
| `ROWID` | `ROWID` / `UROWID` | 内部フォーマットは異なる |

---

## DB2 パッケージ vs Oracle パッケージ

これは、この 2 つのデータベース間で最も互換性の高い領域の 1 つである。どちらも、仕様部（Specification）と本体部（Body）を持つ PL/SQL スタイルのパッケージをサポートしている。

### DB2 パッケージ構造

```sql
-- DB2 ストアド・プロシージャ (SQL PL)
CREATE OR REPLACE PROCEDURE calculate_order_total(
    IN  p_order_id   INTEGER,
    OUT p_total      DECIMAL(10,2),
    OUT p_item_count INTEGER
)
LANGUAGE SQL
BEGIN
    SELECT SUM(line_amount), COUNT(*)
    INTO   p_total, p_item_count
    FROM   order_lines
    WHERE  order_id = p_order_id;

    IF p_total IS NULL THEN
        SET p_total = 0;
        SET p_item_count = 0;
    END IF;
END@
```

### Oracle パッケージの同等実装

```sql
-- Oracle: 関連するプロシージャをパッケージにグループ化
CREATE OR REPLACE PACKAGE order_pkg AS
    PROCEDURE calculate_order_total(
        p_order_id   IN  NUMBER,
        p_total      OUT NUMBER,
        p_item_count OUT NUMBER
    );
    FUNCTION get_order_status(p_order_id IN NUMBER) RETURN VARCHAR2;
END order_pkg;
/

CREATE OR REPLACE PACKAGE BODY order_pkg AS
    PROCEDURE calculate_order_total(
        p_order_id   IN  NUMBER,
        p_total      OUT NUMBER,
        p_item_count OUT NUMBER
    ) AS
    BEGIN
        SELECT SUM(line_amount), COUNT(*)
        INTO   p_total, p_item_count
        FROM   order_lines
        WHERE  order_id = p_order_id;

        IF p_total IS NULL THEN
            p_total      := 0;
            p_item_count := 0;
        END IF;
    END calculate_order_total;

    FUNCTION get_order_status(p_order_id IN NUMBER) RETURN VARCHAR2 AS
        v_status VARCHAR2(20);
    BEGIN
        SELECT status INTO v_status FROM orders WHERE order_id = p_order_id;
        RETURN v_status;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN RETURN 'NOT FOUND';
    END get_order_status;
END order_pkg;
/
```

### DB2 SQL PL と Oracle PL/SQL の違い

| 概念 | DB2 SQL PL | Oracle PL/SQL |
|---|---|---|
| 代入 | `SET v_count = 10` | `v_count := 10` |
| 条件分岐 | `IF ... THEN ... ELSEIF ... ELSE ... END IF` | `IF ... THEN ... ELSIF ... ELSE ... END IF` |
| ループ | `WHILE ... DO ... END WHILE` | `WHILE ... LOOP ... END LOOP` |
| For ループ | `FOR row AS cursor_name CURSOR FOR ... DO ... END FOR` | `FOR rec IN (SELECT ...) LOOP ... END LOOP` |
| ループ中断 | `LEAVE label` | `EXIT` |
| エラー発生 | `SIGNAL SQLSTATE '70001' SET MESSAGE_TEXT = 'error'` | `RAISE_APPLICATION_ERROR(-20001, 'error')` |
| 例外処理 | `DECLARE CONTINUE HANDLER FOR NOT FOUND ...` | `EXCEPTION WHEN NO_DATA_FOUND THEN ...` |

```sql
-- DB2: SIGNAL
IF v_balance < 0 THEN
    SIGNAL SQLSTATE '70001' SET MESSAGE_TEXT = 'Balance cannot be negative';
END IF;

-- Oracle での同等実装
IF v_balance < 0 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Balance cannot be negative');
END IF;
```

---

## REORG TABLE → ALTER TABLE MOVE

DB2 の `REORG TABLE` は、削除された行から領域を回収し、テーブル・データを再編成する。Oracle での同等操作は `ALTER TABLE ... MOVE` (セグメントの再構築) であり、その後に索引の再構築が必要になる。

```sql
-- DB2: 領域の回収とクラスタリングの再編
REORG TABLE employees;
REORG TABLE employees INDEX employee_idx;  -- 特定の索引を使用して再編

-- Oracle での同等操作
ALTER TABLE employees MOVE;
-- MOVE の後、すべての索引が UNUSABLE 状態になるため再構築が必要：
ALTER INDEX employees_pk REBUILD;
ALTER INDEX employees_dept_idx REBUILD;

-- オンラインでの再編成 (ロックなし) の場合：
ALTER TABLE employees MOVE ONLINE;

-- 再構築が必要な索引を確認する：
SELECT index_name, status FROM user_indexes WHERE table_name = 'EMPLOYEES';

-- 指定したテーブルのすべての使用不可な索引を再構築する PL/SQL：
BEGIN
    FOR idx IN (SELECT index_name FROM user_indexes
                WHERE table_name = 'EMPLOYEES' AND status = 'UNUSABLE') LOOP
        EXECUTE IMMEDIATE 'ALTER INDEX ' || idx.index_name || ' REBUILD';
    END LOOP;
END;
/
```

### RUNSTATS → DBMS_STATS

```sql
-- DB2: 統計情報の収集
RUNSTATS ON TABLE myschema.employees WITH DISTRIBUTION AND DETAILED INDEXES ALL;

-- Oracle での同等操作
EXEC DBMS_STATS.GATHER_TABLE_STATS('MYSCHEMA', 'EMPLOYEES',
    CASCADE        => TRUE,
    METHOD_OPT     => 'FOR ALL COLUMNS SIZE AUTO',
    ESTIMATE_PERCENT => DBMS_STATS.AUTO_SAMPLE_SIZE
);
```

---

## 主要な DB2 関数と Oracle の同等物

### 文字列関数

| DB2 関数 | Oracle 同等物 |
|---|---|
| `SUBSTR(s, pos, len)` | `SUBSTR(s, pos, len)` — 同様 |
| `LENGTH(s)` | `LENGTH(s)` — 同様 |
| `UPPER(s)` | `UPPER(s)` — 同様 |
| `LOWER(s)` | `LOWER(s)` — 同様 |
| `LTRIM(s)` | `LTRIM(s)` — 同様 |
| `RTRIM(s)` | `RTRIM(s)` — 同様 |
| `TRIM(s)` | `TRIM(s)` — 同様 |
| `LPAD(s, n, pad)` | `LPAD(s, n, pad)` — 同様 |
| `RPAD(s, n, pad)` | `RPAD(s, n, pad)` — 同様 |
| `LOCATE(sub, s [, start])` | `INSTR(s, sub [, start])` — 引数の順序が異なる |
| `POSSTR(s, sub)` | `INSTR(s, sub)` |
| `LEFT(s, n)` | `SUBSTR(s, 1, n)` |
| `RIGHT(s, n)` | `SUBSTR(s, -n)` |
| `REPEAT(s, n)` | カスタム PL/SQL 関数 |
| `REPLACE(s, from, to)` | `REPLACE(s, from, to)` — 同様 |
| `TRANSLATE(s, from, to)` | `TRANSLATE(s, from, to)` — 同様 |
| `HEX(n)` | `TO_CHAR(n, 'XXXXXXXX')` |
| `DIGITS(n)` | `LPAD(TO_CHAR(n), precision, '0')` |
| `CHAR(n)` | コンテキストに応じて `TO_CHAR(n)` または `CHR(n)` |
| `VARCHAR(expr)` | `TO_CHAR(expr)` |
| `STRIP(s)` | `TRIM(s)` |
| `SPACE(n)` | `RPAD(' ', n)` |
| `VALUE(a, b)` | `NVL(a, b)` または `COALESCE(a, b)` |

**LOCATE と INSTR の引数順序の違い：**
```sql
-- DB2: LOCATE(検索文字列, 対象文字列)
SELECT LOCATE('foo', col) FROM t;

-- Oracle: INSTR(対象文字列, 検索文字列) — 順序が逆
SELECT INSTR(col, 'foo') FROM t;
```

### 日付関数

| DB2 関数 | Oracle 同等物 |
|---|---|
| `YEAR(d)` | `EXTRACT(YEAR FROM d)` |
| `MONTH(d)` | `EXTRACT(MONTH FROM d)` |
| `DAY(d)` | `EXTRACT(DAY FROM d)` |
| `DAYOFWEEK(d)` | `TO_NUMBER(TO_CHAR(d, 'D'))` |
| `DAYOFYEAR(d)` | `d - TRUNC(d, 'YEAR') + 1` |
| `WEEK(d)` | `TO_NUMBER(TO_CHAR(d, 'WW'))` |
| `HOUR(d)` | `EXTRACT(HOUR FROM CAST(d AS TIMESTAMP))` |
| `MINUTE(d)` | `EXTRACT(MINUTE FROM CAST(d AS TIMESTAMP))` |
| `SECOND(d)` | `EXTRACT(SECOND FROM CAST(d AS TIMESTAMP))` |
| `DAYS(d)` | `d - DATE '0001-01-01' + 1` (西暦 1 年からの日数) |
| `JULIAN_DAY(d)` | `TO_NUMBER(TO_CHAR(d, 'J'))` |
| `MIDNIGHT_SECONDS(d)` | `(d - TRUNC(d)) * 86400` |
| `DATE(ts)` | `TRUNC(ts)` |
| `TIME(ts)` | `TO_CHAR(ts, 'HH24:MI:SS')` |
| `TIMESTAMP(d, t)` | `d + TO_DSINTERVAL('0 ' \|\| t)` |
| `MONTHS_BETWEEN(d1, d2)` | `MONTHS_BETWEEN(d1, d2)` — 同様 |
| `ADD_MONTHS(d, n)` | `ADD_MONTHS(d, n)` — 同様 |
| `LAST_DAY(d)` | `LAST_DAY(d)` — 同様 |
| `NEXT_DAY(d, day)` | `NEXT_DAY(d, day)` — 同様 |
| `TIMESTAMPDIFF(unit, ts1, ts2)` | `ts2 - ts1` (日数) または 月数の差には MONTHS_BETWEEN |

### 数学関数

| DB2 関数 | Oracle 同等物 |
|---|---|
| `MOD(a, b)` | `MOD(a, b)` — 同様 |
| `ABS(n)` | `ABS(n)` — 同様 |
| `CEILING(n)` / `CEIL(n)` | `CEIL(n)` |
| `FLOOR(n)` | `FLOOR(n)` — 同様 |
| `ROUND(n, d)` | `ROUND(n, d)` — 同様 |
| `TRUNC(n, d)` | `TRUNC(n, d)` — 同様 |
| `SQRT(n)` | `SQRT(n)` — 同様 |
| `POWER(b, e)` | `POWER(b, e)` — 同様 |
| `LN(n)` | `LN(n)` — 同様 |
| `LOG(n)` | `LOG(10, n)` (DB2 は常用対数。Oracle は引数で底を指定) |
| `EXP(n)` | `EXP(n)` — 同様 |
| `RAND()` | `DBMS_RANDOM.VALUE` |
| `SIGN(n)` | `SIGN(n)` — 同様 |

### 集計関数

| DB2 | Oracle | 備考 |
|---|---|---|
| `COUNT(*)` | `COUNT(*)` — 同様 | |
| `SUM`, `AVG`, `MIN`, `MAX` | 同様 | |
| `LISTAGG(col, sep)` | `LISTAGG(col, sep) WITHIN GROUP (ORDER BY col)` | DB2 11.1 以降対応。Oracle は WITHIN GROUP 必須 |
| `XMLAGG(XMLELEMENT(NAME v, col))` | `XMLAGG(XMLELEMENT("v", col))` | XML ベースの文字列集計 |
| `GROUPING SETS` | `GROUPING SETS` — 同様 | |
| `ROLLUP` | `ROLLUP` — 同様 | |
| `CUBE` | `CUBE` — 同様 | |

---

## DB2 一括エクスポートから Oracle SQL*Loader へのワークフロー

DB2 における主要な一括エクスポート・ツールは以下の通りである。

- `EXPORT TO` コマンド — クエリ結果を CSV/DEL 形式でエクスポート
- `db2move` — スキーマとデータを DB2 内部形式で一括エクスポート
- `db2look` — DDL の生成

### DB2 エクスポート → Oracle インポートのワークフロー

**ステップ 1 : DB2 からのエクスポート**

```bash
# テーブルをカンマ区切りファイル (DEL 形式) にエクスポート
db2 "EXPORT TO /tmp/customers.del OF DEL MODIFIED BY COLDEL, CHARDEL\" SELECT * FROM customers"

# タイムスタンプ形式を指定してエクスポート
db2 "EXPORT TO /tmp/customers.csv OF DEL MODIFIED BY COLDEL, TIMESTAMPFORMAT='YYYY-MM-DD HH:MM:SS' SELECT customer_id, first_name, last_name, email, created_date FROM customers"

# db2look を使用して DDL をエクスポート
db2look -d mydb -e -o schema.ddl -a
```

> ⚠️ 注意：DB2 の EXPORT コマンドのオプション構文（COLDEL, CHARDEL など）は、DB2 のバージョンによって異なる場合がある。使用前に使用しているバージョンの IBM Db2 ドキュメントを確認すること。

**ステップ 2 : DDL の変換**

db2look の出力を確認し、前述の型マッピングを適用する。主な変更点：
- `SMALLINT` → `NUMBER(5)`
- `INTEGER` → `NUMBER(10)`
- `BIGINT` → `NUMBER(19)`
- `VARCHAR(n)` → `VARCHAR2(n)`
- `CLOB(n)` → `CLOB`
- `TIMESTAMP` → `TIMESTAMP`
- `COMPRESS YES/NO` の削除
- `PCTFREE` 以外の DB2 固有のストレージ・オプションを削除し Oracle の同等物に置き換える
- `GENERATED ALWAYS AS IDENTITY` はそのままで移行可能 (Oracle 12c 以降)

**ステップ 3 : Oracle へのロード**

```sql
-- DB2 DEL エクスポート用の SQL*Loader コントロール・ファイル
OPTIONS (DIRECT=TRUE, ROWS=5000)
LOAD DATA
INFILE '/tmp/customers.csv'
APPEND
INTO TABLE customers
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
TRAILING NULLCOLS
(
    customer_id,
    first_name,
    last_name,
    email,
    created_date TIMESTAMP "YYYY-MM-DD HH24:MI:SS"
)
```

---

## ベスト・プラクティス

1. **DB2 と Oracle の構文の類似性を活用する。** DB2 SQL PL と Oracle PL/SQL は多くの構造を共有している。移行の取り組みを、ゼロからの書き換えではなく、相違点（代入演算子、ループ構文、条件ハンドラ）に集中させること。

2. **可能であれば GENERATED AS IDENTITY を使用する。** DB2 11.1 以降と Oracle 12c 以降は、同じ `GENERATED ALWAYS AS IDENTITY` 構文をサポートしている。

3. **DB2 の圧縮設定を確認する。** DB2 の表圧縮（ROW COMPRESSION, VALUE COMPRESSION）は直接マップできない。Oracle 独自の圧縮オプション（COMPRESS FOR OLTP, COMPRESS BASIC 等）から、最適なものを再評価すること。

4. **DB2 表領域の違いを考慮する。** DB2 の表領域（DMS/SMS/AUTOMATIC）は、Oracle の AUTOEXTEND されるローカル管理表領域にマップされる。DB2 のバッファ・プールは Oracle のバッファ・キャッシュ構成（SGA）にマップされる。

5. **引数順序の相違を徹底的にテストする。** DB2 の `LOCATE` と Oracle の `INSTR` の引数順序は逆である。これはアプリケーション・コードやプロシージャにおけるよくあるバグの温床となる。

6. **DB2 TIMESTAMP の精度を確認する。** DB2 のタイムスタンプはマイクロ秒（小数点以下 6 桁）までサポートしている。Oracle TIMESTAMP は最大 9 桁までサポートしており、移行時に精度が失われることはない。

---

## よくある移行の落とし穴

**落とし穴 1 — DB2 DATE vs Oracle DATE：**
前述の通り、DB2 の DATE には時刻が含まれず、Oracle の DATE には含まれる。挿入・比較ロジックにおいてこの違いを考慮する必要がある。

**落とし穴 2 — DB2 スキーマ vs Oracle スキーマ：**
DB2 のスキーマはデータベース内の名前空間であるが、Oracle のスキーマは「ユーザー」と同義である。複数のスキーマを持つ DB2 データベースは、複数の Oracle ユーザー/スキーマにマップされる。

**落とし穴 3 — CYCLE 付きの IDENTITY 列：**
Oracle 12c 以降のアイデンティティ列は、DB2 と同様に `CYCLE` オプションをサポートしている。

**落とし穴 4 — DB2 の CONCAT 関数：**
DB2 の `CONCAT` は 2 つの引数しか受け取れない。Oracle の `CONCAT` も同様である。複数部分の結合には、両方のデータベースで `||` 演算子を使用すること。

**落とし穴 5 — EXCEPTION テーブル名の衝突：**
`EXCEPTION` は Oracle PL/SQL の予約語である。DB2 アプリケーションに `EXCEPTION` という名前のテーブルがある場合は、名前を変更する必要がある。

**落とし穴 6 — 分離レベルの互換性：**
DB2 の `WITH UR` (ダーティ・リード) は、Oracle の MVCC アーキテクチャ上不要である。すべての分離ヒントを削除しても、デフォルトで非ブロッキングな読取りが行われる。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c PL/SQL Language Reference — PL/SQL Language Fundamentals](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-language-fundamentals.html)
- [Oracle Database 19c Administrator's Guide — Managing Schema Objects (ALTER TABLE MOVE)](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-schema-objects.html)
- [Oracle Database 19c PL/SQL Packages Reference — DBMS_STATS](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_STATS.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [Oracle SQL Developer Migration Workbench](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
