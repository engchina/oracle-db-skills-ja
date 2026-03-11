# PostgreSQL から Oracle への移行

## 概要

PostgreSQL と Oracle は、どちらも豊富な SQL サポートを備えた成熟した ACID 準拠のリレーショナル・データベースであるが、構文、データ型、手続き言語、およびエコシステムのツールにおいて大きく異なる点がある。本ガイドでは、型マッピング、SQL 方言の違い、関数の同等物、および PostgreSQL からデータを抽出して Oracle にロードするメカニズムなど、PostgreSQL のワークロードを Oracle Database に移行する際に遭遇する主要な変換課題をすべて網羅する。

---

## データ型マッピング

### 数値型

| PostgreSQL | Oracle | 備考 |
|---|---|---|
| `SMALLINT` | `NUMBER(5)` または `SMALLINT` | Oracle は ANSI SMALLINT をサポート |
| `INTEGER` / `INT` | `NUMBER(10)` または `INTEGER` | Oracle の INTEGER は NUMBER(38) に相当 |
| `BIGINT` | `NUMBER(19)` | 正確に適合 |
| `SERIAL` | `NUMBER` + `SEQUENCE` または `GENERATED AS IDENTITY` | 詳細は後述 |
| `BIGSERIAL` | `NUMBER(19)` + `SEQUENCE` | 同様のパターン |
| `NUMERIC(p,s)` / `DECIMAL(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `REAL` | `BINARY_FLOAT` | IEEE 754 単精度 |
| `DOUBLE PRECISION` | `BINARY_DOUBLE` | IEEE 754 倍精度 |
| `MONEY` | `NUMBER(19,4)` | 直接の Oracle 型はないため NUMBER で格納 |

### SERIAL → シーケンス / アイデンティティ列

PostgreSQL の `SERIAL` は、シーケンスに裏打ちされた整数型の短縮形である。Oracle では 2 つのアプローチがある。

**移行元 PostgreSQL：**
```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Oracle — シーケンスの使用 (12c 未満互換)：**
```sql
CREATE SEQUENCE orders_seq
    START WITH 1
    INCREMENT BY 1
    NOCACHE
    NOCYCLE;

CREATE TABLE orders (
    order_id    NUMBER(10)   DEFAULT orders_seq.NEXTVAL PRIMARY KEY,
    customer_id NUMBER(10)   NOT NULL,
    created_at  TIMESTAMP    DEFAULT SYSTIMESTAMP
);
```

**Oracle — GENERATED AS IDENTITY の使用 (12c 以降、推奨)：**
```sql
CREATE TABLE orders (
    order_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id NUMBER(10)  NOT NULL,
    created_at  TIMESTAMP   DEFAULT SYSTIMESTAMP
);
```

アイデンティティ（IDENTITY）列のアプローチの方が簡潔で、個別のシーケンス・オブジェクトを管理する必要がないが、シーケンス・アプローチの方がキャッシュやサイクリングの動作をより細かく制御できる。

### 文字 / テキスト型

| PostgreSQL | Oracle | 備考 |
|---|---|---|
| `CHAR(n)` | `CHAR(n)` | 直接的な同等物。Oracle は空白でパディングする |
| `VARCHAR(n)` | `VARCHAR2(n)` | VARCHAR ではなく VARCHAR2 を使用（Oracle は将来の変更のために VARCHAR を予約している） |
| `TEXT` | `VARCHAR2(4000)` または `CLOB` | 詳細は後述 |
| `NAME` (システム型) | `VARCHAR2(128)` | 内部的な PG 型。通常は識別子の長さにマップされる |

**TEXT → VARCHAR2 または CLOB の判断：**
- 実際に格納される文字数が 4000 文字未満である場合は、`VARCHAR2(4000)` を使用する。
- 4000 文字を超える可能性がある場合（説明文、メモ、全文ドキュメントなど）は、`CLOB` を使用する。
- Oracle 12c 以降では、`MAX_STRING_SIZE=EXTENDED` に設定することで `VARCHAR2(32767)` まで拡張可能であり、CLOB の複雑さを回避しつつ、ほとんどの TEXT 用途をカバーできる。

```sql
-- PostgreSQL
CREATE TABLE articles (
    id      SERIAL PRIMARY KEY,
    title   VARCHAR(255),
    body    TEXT,
    summary TEXT
);

-- Oracle (12c 以降、MAX_STRING_SIZE=EXTENDED の場合)
CREATE TABLE articles (
    id      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title   VARCHAR2(255),
    body    CLOB,
    summary VARCHAR2(4000)
);
```

### Boolean → NUMBER(1)

PostgreSQL にはネイティブの BOOLEAN 型があるが、Oracle にはない（23c 未満）。標準的な Oracle の慣習は以下の通りである。

```sql
-- PostgreSQL
CREATE TABLE feature_flags (
    flag_name VARCHAR(100) PRIMARY KEY,
    is_active BOOLEAN NOT NULL DEFAULT FALSE
);

-- Oracle
CREATE TABLE feature_flags (
    flag_name VARCHAR2(100) PRIMARY KEY,
    is_active NUMBER(1,0) DEFAULT 0 NOT NULL,
    CONSTRAINT chk_is_active CHECK (is_active IN (0, 1))
);
```

0/1 の意味を強制するために CHECK 制約を追加する。アプリケーション・コード側で `true`/`false` を `1`/`0` に変換する必要がある。Oracle 23c を使用している場合は、ネイティブの BOOLEAN 型が利用可能であり、直接使用できる。

### バイナリ型

| PostgreSQL | Oracle | 備考 |
|---|---|---|
| `BYTEA` | `BLOB` | 任意のバイナリ・データを格納 |
| `BIT(n)` | `RAW(n)` | 固定長のビット文字列用 |
| `BIT VARYING(n)` | `RAW(n)` | 可変長の生のバイナリ用 |

```sql
-- PostgreSQL
CREATE TABLE file_attachments (
    id        SERIAL PRIMARY KEY,
    filename  VARCHAR(255),
    content   BYTEA
);

-- Oracle
CREATE TABLE file_attachments (
    id        NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    filename  VARCHAR2(255),
    content   BLOB
);
```

### 日付と時間型

| PostgreSQL | Oracle | 備考 |
|---|---|---|
| `DATE` | `DATE` | Oracle の DATE は時刻を含む。PG の DATE は日付のみ。 |
| `TIME` | 直接の同等物なし | `VARCHAR2(8)` を使用するか、午前 0 時からの秒数を `NUMBER` で格納 |
| `TIMESTAMP` | `TIMESTAMP` | 共に秒の小数部を含む |
| `TIMESTAMPTZ` | `TIMESTAMP WITH TIME ZONE` | 直接的な同等物 |
| `INTERVAL` | `INTERVAL DAY TO SECOND` / `INTERVAL YEAR TO MONTH` | Oracle はインターバルを 2 つの型に分けている |

**重要な違い：** Oracle の `DATE` 型は日付と時刻（秒まで）の両方を格納する。PostgreSQL の `DATE` は日付のみである。日付のみを格納する PostgreSQL の `DATE` 列をマップする場合、Oracle の `DATE` を使用するが、Oracle は時刻コンポーネントとして真夜中 (00:00:00) を格納することに注意すること。

### その他の型

| PostgreSQL | Oracle | 備考 |
|---|---|---|
| `UUID` | `RAW(16)` または `VARCHAR2(36)` | 効率重視なら RAW(16)、可読性重視なら VARCHAR2(36) |
| `JSON` / `JSONB` | `JSON` (21c 以降) または IS JSON 制約付きの `CLOB` | 詳細は JSON の章を参照 |
| `ARRAY` | 直接の同等物なし | 子テーブルに正規化するか、ネストした表コレクションを使用 |
| `HSTORE` | `JSON` またはキー・値形式の子テーブル | |
| `INET` / `CIDR` | `VARCHAR2(45)` | 文字列として格納し、IP 操作のためにアプリケーション・ロジックを追加 |
| `ENUM` | `VARCHAR2(n)` + CHECK 制約 | またはルックアップ・テーブルを使用 |

---

## SQL 方言の違い

### ILIKE → UPPER + LIKE

PostgreSQL の大文字小文字を区別しない LIKE：

```sql
-- PostgreSQL
SELECT * FROM customers WHERE last_name ILIKE '%smith%';

-- Oracle
SELECT * FROM customers WHERE UPPER(last_name) LIKE UPPER('%smith%');
-- 一般的には：
SELECT * FROM customers WHERE UPPER(last_name) LIKE '%SMITH%';
```

頻繁に使用する場合は、パターンを効率的にサポートするために関数ベースの索引を作成する。

```sql
CREATE INDEX idx_customers_upper_last_name
    ON customers (UPPER(last_name));
```

### LIMIT / OFFSET → FETCH FIRST / OFFSET

```sql
-- PostgreSQL
SELECT * FROM products ORDER BY price DESC LIMIT 10 OFFSET 20;

-- Oracle 12c 以降 (標準 SQL 構文)
SELECT * FROM products
ORDER BY price DESC
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle 11g 以前 (ROWNUM の使用 — サブクエリが必要)
SELECT * FROM (
    SELECT p.*, ROWNUM rn
    FROM (
        SELECT * FROM products ORDER BY price DESC
    ) p
    WHERE ROWNUM <= 30
)
WHERE rn > 20;
```

新規の Oracle コードでは常に 12c 以降の `FETCH FIRST` 構文を使用すること。`ROWNUM` アプローチは、ORDER BY が適用される前に ROWNUM が割り当てられるため、エラーが発生しやすい。

### 型キャスト： :: → CAST()

```sql
-- PostgreSQL
SELECT '2024-01-15'::DATE;
SELECT 3.14::VARCHAR;
SELECT id::TEXT FROM orders;

-- Oracle
SELECT CAST('2024-01-15' AS DATE) FROM DUAL;
SELECT CAST(3.14 AS VARCHAR2(20)) FROM DUAL;
SELECT CAST(id AS VARCHAR2(20)) FROM orders;

-- Oracle では明示的な変換に TO_DATE / TO_CHAR / TO_NUMBER も利用可能：
SELECT TO_DATE('2024-01-15', 'YYYY-MM-DD') FROM DUAL;
SELECT TO_CHAR(3.14) FROM DUAL;
```

### 文字列結合

```sql
-- PostgreSQL (両方可能)
SELECT first_name || ' ' || last_name FROM employees;
SELECT CONCAT(first_name, ' ', last_name) FROM employees;

-- Oracle
SELECT first_name || ' ' || last_name FROM employees;
-- Oracle の CONCAT は引数を 2 つしか受け取れない：
SELECT CONCAT(CONCAT(first_name, ' '), last_name) FROM employees;
```

### NULL 処理

```sql
-- PostgreSQL: NULLIF, COALESCE, IS DISTINCT FROM
SELECT NULLIF(qty, 0) FROM order_lines;
SELECT COALESCE(phone, 'N/A') FROM contacts;
SELECT * FROM t WHERE a IS DISTINCT FROM b;

-- Oracle
SELECT NULLIF(qty, 0) FROM order_lines;       -- 同様
SELECT COALESCE(phone, 'N/A') FROM contacts;  -- 同様
SELECT * FROM t WHERE DECODE(a, b, 0, 1) = 1; -- IS DISTINCT FROM の同等物
-- 或者使用 NVL2 / CASE:
SELECT * FROM t WHERE (a <> b OR (a IS NULL AND b IS NOT NULL) OR (a IS NOT NULL AND b IS NULL));
```

### RETURNING 句

```sql
-- PostgreSQL
INSERT INTO orders (customer_id, total)
VALUES (42, 199.99)
RETURNING order_id, created_at;

-- Oracle (RETURNING INTO を使用 — PL/SQL 内でのみ利用可能)
DECLARE
    v_order_id orders.order_id%TYPE;
    v_created  orders.created_at%TYPE;
BEGIN
    INSERT INTO orders (customer_id, total)
    VALUES (42, 199.99)
    RETURNING order_id, created_at INTO v_order_id, v_created;
    DBMS_OUTPUT.PUT_LINE('New order ID: ' || v_order_id);
END;
/
```

### シーケンスとシーケンス関数

```sql
-- PostgreSQL
SELECT nextval('orders_seq');
SELECT currval('orders_seq');
SELECT setval('orders_seq', 1000);

-- Oracle
SELECT orders_seq.NEXTVAL FROM DUAL;
SELECT orders_seq.CURRVAL FROM DUAL;
-- RESTART を使用したシーケンスのリセット (Oracle 18c 以降で利用可能)：
ALTER SEQUENCE orders_seq RESTART START WITH 1000;
-- 18c 未満のワークアラウンド：
ALTER SEQUENCE orders_seq INCREMENT BY (1000 - orders_seq.CURRVAL);
SELECT orders_seq.NEXTVAL FROM DUAL;
ALTER SEQUENCE orders_seq INCREMENT BY 1;
```

---

## Oracle に同等物のない PostgreSQL 関数とワークアラウンド

### 文字列関数

| PostgreSQL | Oracle 同等物 |
|---|---|
| `STRING_AGG(col, ',')` | `LISTAGG(col, ',') WITHIN GROUP (ORDER BY col)` |
| `ARRAY_AGG(col)` | 直接の同等物なし。`LISTAGG` を使用するかネストした表に格納 |
| `REGEXP_REPLACE(s, pat, repl)` | `REGEXP_REPLACE(s, pat, repl)` — 同様 |
| `REGEXP_MATCHES(s, pat)` | `REGEXP_SUBSTR(s, pat)` — 最初のマッチを返す |
| `SPLIT_PART(s, delim, n)` | `REGEXP_SUBSTR(s, '[^' || delim || ']+', 1, n)` |
| `INITCAP(s)` | `INITCAP(s)` — 同様 |
| `MD5(s)` | `RAWTOHEX(DBMS_CRYPTO.HASH(UTL_I18N.STRING_TO_RAW(s,'AL32UTF8'), DBMS_CRYPTO.HASH_MD5))` |
| `LEFT(s, n)` | `SUBSTR(s, 1, n)` |
| `RIGHT(s, n)` | `SUBSTR(s, -n)` |
| `REPEAT(s, n)` | `RPAD('', n * LENGTH(s), s)` または `LPAD(s, n*LENGTH(s), s)` — またはカスタム関数を作成 |
| `LPAD`, `RPAD` | `LPAD`, `RPAD` — 同様 |
| `POSITION(sub IN s)` | `INSTR(s, sub)` |
| `OVERLAY(s PLACING r FROM n FOR m)` | `SUBSTR(s,1,n-1) || r || SUBSTR(s, n+m)` |

**STRING_AGG の例：**
```sql
-- PostgreSQL
SELECT dept_id, STRING_AGG(last_name, ', ' ORDER BY last_name) AS employees
FROM emp
GROUP BY dept_id;

-- Oracle
SELECT dept_id, LISTAGG(last_name, ', ') WITHIN GROUP (ORDER BY last_name) AS employees
FROM emp
GROUP BY dept_id;
```

### 日付 / 時刻関数

| PostgreSQL | Oracle 同等物 |
|---|---|
| `NOW()` | `SYSTIMESTAMP` (TZ 付) または `SYSDATE` |
| `CURRENT_TIMESTAMP` | `CURRENT_TIMESTAMP` — 同様 |
| `DATE_TRUNC('month', d)` | `TRUNC(d, 'MM')` |
| `DATE_PART('year', d)` | `EXTRACT(YEAR FROM d)` |
| `AGE(d1, d2)` | `d1 - d2` は日数を返す。月数の差には MONTHS_BETWEEN を使用 |
| `EXTRACT(epoch FROM d)` | `(d - DATE '1970-01-01') * 86400` |
| `TO_TIMESTAMP(s, fmt)` | `TO_TIMESTAMP(s, fmt)` — 同様だがフォーマット・マスクが若干異なる |
| `MAKE_DATE(y, m, d)` | `TO_DATE(y || '-' || m || '-' || d, 'YYYY-MM-DD')` |
| `GENERATE_SERIES(start, end, step)` | `CONNECT BY LEVEL` または再帰的 CTE を使用 |

**GENERATE_SERIES の同等物：**
```sql
-- PostgreSQL: 日付シリーズの生成
SELECT generate_series('2024-01-01'::DATE, '2024-01-31'::DATE, '1 day'::INTERVAL)::DATE AS dt;

-- Oracle
SELECT DATE '2024-01-01' + LEVEL - 1 AS dt
FROM DUAL
CONNECT BY LEVEL <= (DATE '2024-01-31' - DATE '2024-01-01' + 1);
```

### 数学 / 分析関数

| PostgreSQL | Oracle 同等物 |
|---|---|
| `RANDOM()` | `DBMS_RANDOM.VALUE` |
| `TRUNC(n)` | `TRUNC(n)` — 同様 |
| `DIV(a, b)` | `TRUNC(a/b)` |
| `MOD(a, b)` | `MOD(a, b)` — 同様 |
| `CBRT(n)` | `POWER(n, 1/3)` |
| `FACTORIAL(n)` | 組み込み関数なし。PL/SQL 関数を記述 |
| `WIDTH_BUCKET` | `WIDTH_BUCKET` — 同様 (標準 SQL) |

---

## psql vs SQL*Plus

| 機能 | psql | SQL*Plus |
|---|---|---|
| 接続 | `psql -h host -U user -d db` | `sqlplus user/pass@host:1521/service` |
| ファイル実行 | `\i file.sql` | `@file.sql` |
| 表一覧を表示 | `\dt` | `SELECT table_name FROM user_tables;` |
| 表定義を表示 | `\d tablename` | `DESC tablename` |
| データベース表示 | `\l` | `SELECT name FROM v$database;` |
| 実行時間表示 | `\timing` | `SET TIMING ON` |
| 出力をファイルへ | `\o output.txt` | `SPOOL output.txt` |
| 終了 | `\q` | `EXIT` または `QUIT` |
| 変数置換 | `:varname` | `&varname` または `&&varname` |
| Null 表記 | `\pset null 'NULL'` | `SET NULL 'NULL'` |
| 行セパレータ | 自動 | `SET PAGESIZE 50` |

### SQL*Plus スクリプトの例：

```sql
-- psql スクリプト
\timing
\o /tmp/report.txt
SELECT dept, COUNT(*) FROM emp GROUP BY dept ORDER BY dept;
\o
\q

-- SQL*Plus 同等物
SET TIMING ON
SPOOL /tmp/report.txt
SELECT dept, COUNT(*) FROM emp GROUP BY dept ORDER BY dept;
SPOOL OFF
EXIT
```

---

## pg_dump から Oracle へのインポート・ワークフロー

PostgreSQL には、Oracle が直接読み取れるネイティブのエクスポート形式はない。一般的なパイプラインは以下の通りである。

### ステップ 1 — PostgreSQL からの抽出

```bash
# CSV 形式でのエクスポート (最もポータブル)
psql -U myuser -d mydb -c "\COPY mytable TO '/tmp/mytable.csv' WITH CSV HEADER"

# スキーマ参照用にすべての表を plain SQL 形式で pg_dump
pg_dump -U myuser -d mydb --schema-only -f schema.sql

# スクリプトを使用して特定の表を CSV にエクスポート
for tbl in customers orders products; do
  psql -U myuser -d mydb -c "\COPY $tbl TO '/tmp/${tbl}.csv' WITH CSV HEADER NULL ''"
done
```

### ステップ 2 — Oracle スキーマの作成

前述の型マッピングを使用して、pg_dump の出力から `CREATE TABLE` ステートメントを手動で変換する。ora2pg のようなツールでこのステップを自動化することも可能である（`oracle-migration-tools.md` を参照）。

### ステップ 3 — SQL*Loader を使用した Oracle へのロード

```sql
-- SQL*Loader 控制ファイル: customers.ctl
OPTIONS (SKIP=1, ROWS=1000, DIRECT=TRUE)
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
    created_at DATE "YYYY-MM-DD HH24:MI:SS"
)
```

```bash
sqlldr userid=myuser/mypass@mydb control=customers.ctl log=customers.log
```

### ステップ 4 — 行数の検証

```sql
-- PostgreSQL で実行
SELECT 'customers' AS tbl, COUNT(*) AS cnt FROM customers
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'products', COUNT(*) FROM products;

-- Oracle で実行 (結果を比較)
SELECT 'customers' AS tbl, COUNT(*) AS cnt FROM customers
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'products', COUNT(*) FROM products;
```

### SQL Developer Migration Workbench によるスキーマ変換の自動化

Oracle SQL Developer Migration Workbench は PostgreSQL を移行元としてサポートしており、Oracle へのスキーマ変換の多くを自動化できる。Oracle が移行先のワークロードでは、ora2pg よりもこちらを使用することが推奨される。

> ⚠️ **ora2pg の方向に関する警告：** ora2pg は Oracle データベースを **PostgreSQL へ** 移行するためのツールであり、PostgreSQL から Oracle へ移行するためのツールではない。Oracle が移行先となる場合には ora2pg を使用しないこと。一部のガイドでこの方向での使用が言及されていることがあるが、不適切である。PostgreSQL → Oracle のスキーマ変換には SQL Developer Migration Workbench または AWS SCT を使用すること。

---

## ベスト・プラクティス

1. **NULL の動作を早期に監査する。** PostgreSQL は空文字 `''` を非 NULL として扱うが、Oracle は伝統的に NULL として扱う（ただし、標準文字列モードを備えた 23c では変更された）。空文字がセンチネル値として使用されている列を確認すること。

2. **データ・ロード後にシーケンスを移行する。** シーケンスを使用する場合、主キーの競合を避けるために、その START WITH 値を既存の最大 ID よりも大きな値に設定すること。

3. **BOOLEAN 列を徹底的にテストする。** Python の `True`/`False` や Java の `boolean` 値を渡すアプリケーション・レイヤーでは、Oracle 用に `1`/`0` への明示的な変換が必要になる場合がある。

4. **FROM なしの SELECT を避ける。** PostgreSQL では `SELECT 1 + 1` が許可されるが、Oracle では `FROM DUAL` が必要である： `SELECT 1 + 1 FROM DUAL`。

5. **RETURNING 句の使用に注意する。** `INSERT ... RETURNING` に依存している PostgreSQL アプリケーションは、PL/SQL ブロックまたは Oracle JDBC の `RETURN_GENERATED_KEYS` メカニズムを使用するようにリファクタリングする必要がある。

6. **識別子の大文字小文字の区別。** PostgreSQL は引用符で囲まれていない識別子を小文字に畳み込む。Oracle は大文字に畳み込む。引用符で囲まれた識別子は両者ともケースを保持するが、慣習が混在するとデバッグが困難な問題を引き起こす。Oracle の大文字畳み込みが均一に適用されるよう、引用符なしの識別子を推奨する。

7. **VARCHAR ではなく VARCHAR2 を使用する。** Oracle において `VARCHAR` は、将来 SQL 標準の文字型として再定義されるために予約されている。不測の事態を避けるため、常に `VARCHAR2` を使用すること。

8. **スキーマ特権モデルが異なる。** PostgreSQL はカタログ、スキーマ、ロールを Oracle とは異なる方法で分離している。Oracle では、スキーマはユーザーそのものである。移行前にユーザー/スキーマ・アーキテクチャを計画すること。

---

## よくある移行の落とし穴

**落とし穴 1 — 時刻コンポーネントを持つ DATE 型：**
Oracle の `DATE` は時刻を保持するが、PostgreSQL の `DATE` は保持しない。`WHERE created_date = DATE '2024-01-15'` のような比較は、`created_date` に真夜中以外の時刻コンポーネントがある場合、Oracle では暗黙的に行を見逃してしまう。
```sql
-- 安全な Oracle の日付比較
WHERE created_date >= DATE '2024-01-15'
  AND created_date < DATE '2024-01-16'
-- または：
WHERE TRUNC(created_date) = DATE '2024-01-15'
```

**落とし穴 2 — 空文字 vs NULL：**
```sql
-- PostgreSQL: これらは別物
WHERE email = ''     -- 空文字の行を検索
WHERE email IS NULL  -- NULL の行を検索

-- Oracle 21c 以前: '' IS NULL は TRUE と評価される
-- '' を INSERT するとすべて NULL になる
INSERT INTO t (col) VALUES ('');  -- Oracle は '' ではなく NULL を格納する
```

**落とし穴 3 — OUTER JOIN 構文：**
PostgreSQL は古い Oracle の `(+)` 結合構文をサポートしていることがあるが、現代的な ANSI 結合もサポートしている。Oracle における `(+)` には PostgreSQL の挙動と異なる特殊なケースがあるため、すべての結合が ANSI 構文を使用していることを確認すること。

**落とし穴 4 — トリガー構文：**
PostgreSQL のトリガーはトリガー関数を呼び出すが、Oracle はトリガー本体にロジックを埋め込む。完全な書き換えが必要。

**落とし穴 5 — スキーマと search_path：**
PostgreSQL は修飾されていないオブジェクト名を解決するために `search_path` を使用する。Oracle はカレント・スキーマに対して解決する。スキーマをまたがる参照には `schema.object` 表記を使用する必要がある。

**落とし穴 6 — ウィンドウ関数の FILTER 句：**
```sql
-- PostgreSQL はウィンドウ関数で FILTER をサポート
SELECT SUM(amount) FILTER (WHERE status = 'paid') OVER (PARTITION BY customer_id)
FROM orders;

-- Oracle: 集計関数内で CASE を使用する
SELECT SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END)
       OVER (PARTITION BY customer_id)
FROM orders;
```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE (GENERATED AS IDENTITY)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c SQL Language Reference — Analytic Functions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Analytic-Functions.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [Oracle SQL Developer Migration Workbench](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
- [AWS Schema Conversion Tool User Guide](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)
