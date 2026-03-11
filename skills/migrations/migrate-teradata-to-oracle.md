# Teradata から Oracle への移行

## 概要

Teradata は、大規模な分析ワークロードの超並列処理 (MPP) に最適化された、専用設計の分析データウェアハウスである。移行先としては、Oracle Autonomous Data Warehouse (ADW) や Oracle Exadata が主なターゲットとなる。両者ともエンタープライズ・グレードのプラットフォームであるが、SQL 方言、アーキテクチャ・モデル、データ・ロード・ツール、および分析機能において実質的な違いがある。

本ガイドでは、Teradata 固有の SQL 構文 (BTEQ や SEL 短縮形を含む)、MULTISET 表と SET 表のセマンティクス、データ型マッピング、QUALIFY 句、Teradata 固有の集計、および Teradata TPT (Teradata Parallel Transporter) から Oracle SQL*Loader や Data Pump への移行について解説する。

---

## Teradata SQL 方言

### SEL 短縮形と BTEQ コマンド

Teradata では `SELECT` の短縮形として `SEL` を使用できる。また、BTEQ (Basic Teradata Query) では `.` プレフィックスを付けるスクリプト・コマンドが使用される。

```sql
-- Teradata BTEQ セッション
.LOGON myhost/myuser,mypassword;
.SET WIDTH 200;
.SET TITLEDASHES OFF;

SEL * FROM customers WHERE cust_id = 12345;
SEL COUNT(*) FROM orders;

-- SEL を使用したインサート
INSERT INTO archive_orders SEL * FROM orders WHERE order_date < '2020-01-01' (DATE);

.LOGOFF;
.QUIT;
```

```sql
-- Oracle SQL*Plus での同等実装
CONNECT myuser/mypassword@myhost:1521/MYDB
SET LINESIZE 200
SET UNDERLINE OFF

SELECT * FROM customers WHERE cust_id = 12345;
SELECT COUNT(*) FROM orders;

-- サブクエリを使用したインサート
INSERT INTO archive_orders SELECT * FROM orders WHERE order_date < DATE '2020-01-01';

EXIT;
```

### BTEQ → SQL*Plus/SQLcl コマンド・マッピング

| BTEQ コマンド | SQL*Plus/SQLcl 同等機能 |
|---|---|
| `.LOGON host/user,pass` | `CONNECT user/pass@host` |
| `.LOGOFF` | `DISCONNECT` |
| `.QUIT` | `EXIT` |
| `.SET WIDTH n` | `SET LINESIZE n` |
| `.SET MAXERROR n` | `WHENEVER SQLERROR EXIT` |
| `.EXPORT FILE=out.txt` | `SPOOL out.txt` |
| `.EXPORT RESET` | `SPOOL OFF` |
| `.IF ERRORCODE <> 0 THEN .QUIT 12` | `WHENEVER SQLERROR EXIT 12` |
| `.SYSTEM cmd` | `HOST cmd` または `!cmd` |
| `.REMARK text` | `-- text` または `REM text` |
| `SHOW TABLE tbl` | `DESC tbl` |

### Teradata SELECT 構文のバリエーション

```sql
-- Teradata: AS なしの列エイリアス (AS は任意)
SEL
    cust_id               (INTEGER),     -- 出力時の明示的な型キャスト
    first_name (CHAR(50)) customer,      -- フォーマットとエイリアス
    last_name             ln,            -- 位置によるエイリアス
    order_date (FORMAT 'YYYY-MM-DD')     -- 出力フォーマット
FROM customers;

-- Oracle での同等実装
SELECT
    CAST(cust_id AS NUMBER(10))  AS cust_id,
    RPAD(first_name, 50)         AS customer,
    last_name                    AS ln,
    TO_CHAR(order_date, 'YYYY-MM-DD') AS order_date
FROM customers;
```

---

## MULTISET 表 vs SET 表

これは移行前に理解しておくべき最重要の Teradata 概念の 1 つである。

### Teradata の表セマンティクス

Teradata における表の種類：
- **SET 表** (古い TD のデフォルト) : インサート時に、プライマリ・インデックス (PI) や一意制約に基づいて重複行が暗黙的に拒否される。エラーは発生せず、インサートも行われない。
- **MULTISET 表** : 重複行を許可する (標準的な SQL の挙動)。

```sql
-- Teradata SET 表 (デフォルト) : 重複は暗黙的に破棄される
CREATE SET TABLE customers, NO FALLBACK (
    cust_id     INTEGER     NOT NULL,
    cust_name   VARCHAR(100)
)
PRIMARY INDEX (cust_id);

-- Teradata MULTISET 表 : 重複を許可する
CREATE MULTISET TABLE fact_events, NO FALLBACK (
    event_id    BIGINT,
    event_type  VARCHAR(50),
    event_date  DATE
)
PRIMARY INDEX (event_id);
```

### Oracle での同等実装

Oracle のテーブルはデフォルトで重複を許可する (MULTISET に相当)。一意性を強制するには、PRIMARY KEY または UNIQUE 制約を追加する必要がある。

```sql
-- SET 表に相当する Oracle での実装 (制約による一意性強制)
CREATE TABLE customers (
    cust_id   NUMBER(10)   NOT NULL,
    cust_name VARCHAR2(100),
    CONSTRAINT pk_customers PRIMARY KEY (cust_id)
);

-- MULTISET 表に相当する Oracle での実装 (一意性強制なし)
CREATE TABLE fact_events (
    event_id   NUMBER(19),
    event_type VARCHAR2(50),
    event_date DATE
);
-- 注意：プライマリ・キーがないため重複が許可される
```

**重要な移行課題：** Teradata の SET 表で、暗黙的な重複拒否をデータ・クレンジング・メカニズムとして利用していた場合、その挙動は Oracle には存在しない。Teradata で「正常に動作（重複が暗黙的に削除）」していたデータは、Oracle でプライマリ・キー制約を追加すると ORA-00001 (一意制約違反) エラーを引き起こす。制約を適用する前に、データの重複状況をプロファイリングすること。

---

## データ型マッピング

### 数値型

| Teradata | Oracle | 備考 |
|---|---|---|
| `BYTEINT` | `NUMBER(3)` | −128 ～ 127 |
| `SMALLINT` | `NUMBER(5)` | |
| `INTEGER` / `INT` | `NUMBER(10)` | |
| `BIGINT` | `NUMBER(19)` | |
| `DECIMAL(p,s)` / `DEC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `NUMERIC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `FLOAT` / `REAL` | `BINARY_FLOAT` | IEEE 754 |
| `DOUBLE PRECISION` | `BINARY_DOUBLE` | |
| `NUMBER(n)` | `NUMBER(n)` | |

### 文字列型

| Teradata | Oracle | 備考 |
|---|---|---|
| `CHAR(n)` | `CHAR(n)` | Teradata 最大 64,000 バイト、Oracle 最大 2,000 |
| `VARCHAR(n)` / `CHAR VARYING(n)` | `VARCHAR2(n)` | Teradata 最大 64,000、Oracle 最大 4,000/32,767 |
| `LONG VARCHAR` | `CLOB` | Teradata では非推奨。最大 約 64K |
| `CLOB(n)` | `CLOB` | キャラクター・ラージ・オブジェクト |
| `BYTE(n)` | `RAW(n)` | 固定長バイナリ |
| `VARBYTE(n)` | `RAW(n)` | 可変長バイナリ |
| `BLOB(n)` | `BLOB` | バイナリ・ラージ・オブジェクト |

### 日付 / 時刻型

| Teradata | Oracle | 備考 |
|---|---|---|
| `DATE` | `DATE` | Teradata は日付のみ。Oracle は時刻も含む |
| `TIME(n)` | 同等の型なし | `VARCHAR2(15)` または秒数として保存 |
| `TIMESTAMP(n)` | `TIMESTAMP(n)` | |
| `TIME WITH TIME ZONE` | `TIMESTAMP WITH TIME ZONE` | |
| `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP WITH TIME ZONE` | |

**Teradata の DATE 格納：** Teradata は内部的に日付を整数 (year-1900)*10000 + month*100 + day の形式で保持する。Oracle は 7 バイトの内部フォーマットで保持する。SQL を使用する分には透明だが、バイナリ・レベルの一括データ転送ではこの違いが重要になる。

```sql
-- Teradata: DATE 演算
SEL CURRENT_DATE + 30;             -- 30 日追加
SEL CURRENT_DATE - CAST(7 AS INTEGER);

-- Oracle
SELECT SYSDATE + 30 FROM DUAL;
SELECT SYSDATE - 7 FROM DUAL;
```

### 特殊な Teradata 型

| Teradata | Oracle | 備考 |
|---|---|---|
| `INTERVAL YEAR` | `INTERVAL YEAR TO MONTH` | |
| `INTERVAL YEAR TO MONTH` | `INTERVAL YEAR TO MONTH` | |
| `INTERVAL DAY` | `INTERVAL DAY TO SECOND` | |
| `INTERVAL DAY TO SECOND` | `INTERVAL DAY TO SECOND` | |
| `PERIOD(DATE)` | 2 つの DATE 列（開始/終了） | Teradata の時間的期間型。Oracle に同等の単一型はない |
| `PERIOD(TIMESTAMP)` | 2 つの TIMESTAMP 列 | |
| `JSON` | `JSON` (21c 以降) または `CLOB IS JSON` | |
| `ST_GEOMETRY` | `SDO_GEOMETRY` | Oracle Spatial が必要 |
| `XML` | `XMLTYPE` | |

---

## QUALIFY 句 → ウィンドウ関数を伴うサブクエリ

Teradata の `QUALIFY` は、集計に対する `HAVING` のように、ウィンドウ関数の結果に適用されるフィルタリング句である。Oracle を含む他の主要な RDBMS には、これをネイティブにサポートしているものはない。

```sql
-- Teradata: 顧客ごとに最新の注文 1 件のみを保持
SEL
    cust_id,
    order_id,
    order_date,
    total_amount
FROM orders
QUALIFY RANK() OVER (PARTITION BY cust_id ORDER BY order_date DESC) = 1;

-- Teradata: ページネーションのための行番号フィルタ
SEL * FROM products
QUALIFY ROW_NUMBER() OVER (ORDER BY price DESC) BETWEEN 11 AND 20;

-- Oracle での同等実装: サブクエリまたは CTE
SELECT cust_id, order_id, order_date, total_amount
FROM (
    SELECT cust_id, order_id, order_date, total_amount,
           RANK() OVER (PARTITION BY cust_id ORDER BY order_date DESC) AS rnk
    FROM orders
)
WHERE rnk = 1;

-- CTE による Oracle ページネーション
WITH ranked AS (
    SELECT product_id, product_name, price,
           ROW_NUMBER() OVER (ORDER BY price DESC) AS rn
    FROM products
)
SELECT product_id, product_name, price
FROM ranked
WHERE rn BETWEEN 11 AND 20;
```

---

## Teradata 固有の集計と関数

### 文字列の集計

```sql
-- Teradata: XML ベースの文字列集計 (一般的な回避策)
SELECT cust_id,
       TRIM(TRAILING ',' FROM
            XMLAGG(XMLELEMENT(NAME x, TRIM(product_name) || ',')
                   ORDER BY product_name).EXTRACT('//text()').GETCLOBVAL()
       ) AS products
FROM order_items
GROUP BY cust_id;

-- Oracle: LISTAGG (より簡潔)
SELECT cust_id,
       LISTAGG(product_name, ',') WITHIN GROUP (ORDER BY product_name) AS products
FROM order_items
GROUP BY cust_id;
```

### 統計関数

| Teradata 関数 | Oracle 同等物 |
|---|---|
| `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY col)` | `PERCENTILE_CONT(p) WITHIN GROUP (ORDER BY col)` — 同様 |
| `PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY col)` | `PERCENTILE_DISC(p) WITHIN GROUP (ORDER BY col)` — 同様 |
| `REGR_SLOPE(y, x)` | `REGR_SLOPE(y, x)` — 同様 |
| `REGR_INTERCEPT(y, x)` | `REGR_INTERCEPT(y, x)` — 同様 |
| `CORR(y, x)` | `CORR(y, x)` — 同様 |
| `STDDEV_SAMP(n)` | `STDDEV(n)` |
| `STDDEV_POP(n)` | `STDDEV_POP(n)` — 同様 |
| `VAR_SAMP(n)` | `VARIANCE(n)` |
| `VAR_POP(n)` | `VAR_POP(n)` — 同様 |
| `KURTOSIS(n)` | 組み込み関数なし。統計パッケージを使用 |
| `SKEWNESS(n)` | 組み込み関数なし |

### 一般的な Teradata 関数

| Teradata 関数 | Oracle 同等物 |
|---|---|
| `OREPLACE(s, old, new)` | `REPLACE(s, old, new)` |
| `OTRANSLATE(s, from, to)` | `TRANSLATE(s, from, to)` |
| `INDEX(s, sub)` | `INSTR(s, sub)` |
| `SUBSTRING(s FROM pos FOR len)` | `SUBSTR(s, pos, len)` |
| `CHARACTERS(s)` / `CHAR_LENGTH(s)` | `LENGTH(s)` |
| `BYTES(expr)` | `LENGTHB(expr)` |
| `TRIM(BOTH 'x' FROM s)` | `TRIM('x' FROM s)` |
| `ZEROIFNULL(n)` | `NVL(n, 0)` |
| `NULLIFZERO(n)` | `NULLIF(n, 0)` |
| `HASHROW(col)` | `ORA_HASH(col)` (近似値) |
| `DAY_OF_WEEK(d)` | `TO_NUMBER(TO_CHAR(d, 'D'))` |
| `MONTH_OF_YEAR(d)` | `EXTRACT(MONTH FROM d)` |

---

## Teradata Temporal 表 → Oracle での時間的クエリ

Teradata は、`PERIOD` 型の列と時間的 DML を使用した時間的 (Temporal) 表 (バイテンポラル表) を組み込みでサポートしている。Oracle にはネイティブの拡張構文はないが、手動で同様のパターンを実装できる。

```sql
-- Teradata 時間的表
CREATE TABLE employee_salary (
    emp_id     INTEGER,
    salary     DECIMAL(10,2),
    valid_time PERIOD(DATE) NOT NULL AS VALIDTIME,
    trans_time PERIOD(TIMESTAMP(6) WITH TIME ZONE) NOT NULL AS TRANSACTIONTIME
)
PRIMARY INDEX (emp_id);

-- Oracle での明示的な期間列を使用した同等実装
CREATE TABLE employee_salary (
    emp_id          NUMBER(10)    NOT NULL,
    salary          NUMBER(10,2)  NOT NULL,
    valid_from      DATE          NOT NULL,
    valid_to        DATE          DEFAULT DATE '9999-12-31' NOT NULL,
    transaction_from TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP NOT NULL,
    transaction_to   TIMESTAMP WITH TIME ZONE DEFAULT TIMESTAMP '9999-12-31 00:00:00 UTC' NOT NULL,
    CONSTRAINT pk_emp_salary PRIMARY KEY (emp_id, valid_from, transaction_from)
);

-- 現在有効な状態を照会
SELECT emp_id, salary
FROM employee_salary
WHERE valid_from <= TRUNC(SYSDATE) AND valid_to > TRUNC(SYSDATE)
  AND transaction_from <= SYSTIMESTAMP AND transaction_to > SYSTIMESTAMP;
```

---

## TPT (Teradata Parallel Transporter) → Oracle SQL*Loader / Data Pump

### Teradata TPT によるエクスポート

```bash
# TPT エクスポート・スクリプト (tbuild)
# ファイル: export_orders.tpt
DEFINE JOB export_orders_job
(
    DEFINE OPERATOR export_oper
    TYPE EXPORT
    SCHEMA *
    ATTRIBUTES
    (
        VARCHAR DirectoryPath    = '/data/export',
        VARCHAR FileWritingRule  = 'Truncate',
        VARCHAR Format           = 'DELIMITED',
        VARCHAR TextDelimiter    = ','
    );
    -- ... スキーマ定義とステップ定義 ...
);
```

### Oracle SQL*Loader によるインポート

Teradata から CSV にエクスポートした後：

```
-- SQL*Loader コントロール・ファイル: orders.ctl
OPTIONS (DIRECT=TRUE, PARALLEL=TRUE, ROWS=10000)
LOAD DATA
INFILE '/data/export/orders*.dat'
APPEND
INTO TABLE orders
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
TRAILING NULLCOLS
(
    order_id,
    customer_id,
    order_date    DATE "YYYY-MM-DD",
    total_amount
)
```

```bash
sqlldr userid=myuser/mypass@mydb control=orders.ctl log=orders.log
```

---

## Teradata FastLoad / MultiLoad → Oracle の対応ツール

| Teradata ユーティリティ | Oracle 同等ツール | ユースケース |
|---|---|---|
| FastLoad | SQL*Loader DIRECT=TRUE | 空テーブルへの大量ロード |
| MultiLoad | SQL*Loader (APPEND 指定) | 既存テーブルへの大量 DML |
| FastExport | 外部表 + SQL | 高速なデータ・エクスポート |
| BTEQ | SQL*Plus / SQLcl | 対話型クエリとスクリプト実行 |
| TPT | SQL*Loader + シェル・スクリプト | 並列 ETL 処理 |

---

## ベスト・プラクティス

1. **PI (Primary Index) の使用状況をプロファイリングする。** Teradata の PI は、AMP 間でのデータ分散を決定する。Oracle 移行時、PI 列はパーティション・キーや主キーの有力な候補となる。PI でのフィルタリングや結合がどのように行われているかを理解し、Oracle の索引戦略に役立てること。

2. **移行前に MULTISET 表の重複を処理する。** MULTISET 表の重複行を確認すること：
   ```sql
   -- Teradata: MULTISET 表内の重複行を特定
   SEL order_id, COUNT(*) cnt FROM fact_orders GROUP BY order_id HAVING cnt > 1;
   ```
   重複を除去するか、保持するか、または代替キーを追加するかを検討すること。

3. **QUALIFY を体系的にサブクエリに書き換える。** すべての `QUALIFY` 使用箇所を特定し、同等のサブクエリまたは CTE パターンに変換すること。これは機械的な作業であるが、すべての箇所で行う必要がある。

4. **Teradata のマクロ・オブジェクトを見直す。** Teradata マクロは、データベース内に保存されたパラメータ化された SQL テンプレートである。これらは、Oracle のストアド・プロシージャに移行するのが最も自然である。

5. **Teradata UDF を Oracle にマッピングする。** C 言語で記述された Teradata UDF は、PL/SQL または Java (Oracle Java ストアド・プロシージャ) での書き直しが必要になる場合がある。移行前にすべての UDF 依存関係を特定すること。

---

## よくある移行の落とし穴

**落とし穴 1 — SET 表の暗黙的な重複拒否：**
前述の通り、これは最も一般的な「静かな」データ移行の問題である。SET 表のセマンティクスは Oracle にはないため、明示的な制約に置き換える必要がある。まずデータに重複がないか監査すること。

**落とし穴 2 — BYTEINT 型：**
Oracle には 1 バイトの整数型がない。`NUMBER(3)` または `NUMBER(5)` を使用する。BYTEINT 値に対してビット演算を行っているアプリケーションは注意が必要である。

**落とし穴 3 — ANSI モード vs Teradata モード：**
Teradata セッションは ANSI モードまたは Teradata モードで動作する。Teradata モードでは、表の種類、暗黙的な型変換、トランザクション処理が異なる。Teradata モードを使用している場合は、Oracle の ANSI 準拠の挙動と一致しない可能性があることに注意。

**落とし穴 4 — 列レベルの COMPRESS キーワード：**
Teradata の列レベル圧縮 (COMPRESS) は、NULL や頻出値を効率的に格納する内部最適化である。Oracle では、代わりにテーブル・レベルの圧縮を使用すること。
```sql
-- Teradata: 列レベルの圧縮
salary DECIMAL(10,2) COMPRESS (0.00, NULL)

-- Oracle: テーブル・レベルの圧縮
CREATE TABLE employees (...) COMPRESS FOR OLTP;
```

**落とし穴 5 — BTEQ エラー処理：**
BTEQ の `.IF ERRORCODE <> 0 THEN .QUIT` によるエラー処理は、SQL*Plus や SQLcl に直接変換できない。SQL*Plus では `WHENEVER SQLERROR EXIT n ROLLBACK` などを使用して同等の挙動を実現すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c SQL Language Reference — Analytic Functions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Analytic-Functions.html)
- [Oracle Database 19c SQL Language Reference — LISTAGG](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/LISTAGG.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [Oracle Database 19c Utilities — Data Pump Export (expdp)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-data-pump-export-utility.html)
- [Oracle SQL Developer Migration Workbench](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
