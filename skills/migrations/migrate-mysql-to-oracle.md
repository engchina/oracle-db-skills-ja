# MySQL から Oracle への移行

## 概要

MySQL と Oracle は、Oracle Corporation による所有という共通の系統を持っているが、その 2 つのエンジンはアーキテクチャおよび構文の両面で大きく異なる。MySQL の寛容な型強制、バッククォートによる識別子、`AUTO_INCREMENT` 列、および簡素化されたストアド・プロシージャ構文はすべて、慎重な変換を必要とする。本ガイドでは、データ型マッピング、SQL 方言の違い、ストアド・プロシージャの変換、およびデータ抽出戦略など、MySQL から Oracle への移行中に遭遇する主要な相違点をすべて網羅する。

---

## データ型マッピング

### 整数とオートインクリメント型

MySQL の `AUTO_INCREMENT` は、インクリメントする整数を自動的に割り当てる列ごとのプロパティである。Oracle では、アイデンティティ（IDENTITY）列 (12c 以降) とシーケンスという 2 つのメカニズムが用意されている。

| MySQL | Oracle | 備考 |
|---|---|---|
| `TINYINT` | `NUMBER(3)` | 範囲：−128 ～ 127 |
| `SMALLINT` | `NUMBER(5)` | 範囲：−32,768 ～ 32,767 |
| `MEDIUMINT` | `NUMBER(7)` | MySQL 固有。Oracle に同等の型はない |
| `INT` / `INTEGER` | `NUMBER(10)` | |
| `BIGINT` | `NUMBER(19)` | |
| `TINYINT(1)` | `NUMBER(1)` | BOOLEAN に対する MySQL の慣習的な型 |
| `INT AUTO_INCREMENT` | `NUMBER GENERATED ALWAYS AS IDENTITY` | 詳細は後述の例を参照 |
| `BIGINT AUTO_INCREMENT` | `NUMBER(19) GENERATED ALWAYS AS IDENTITY` | |

**AUTO_INCREMENT → アイデンティティ列：**

```sql
-- MySQL
CREATE TABLE customers (
    customer_id INT          AUTO_INCREMENT PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    created_at  DATETIME     DEFAULT CURRENT_TIMESTAMP
);

-- Oracle (12c 以降、アイデンティティ列)
CREATE TABLE customers (
    customer_id NUMBER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       VARCHAR2(255) NOT NULL,
    created_at  TIMESTAMP    DEFAULT SYSTIMESTAMP,
    CONSTRAINT uq_customers_email UNIQUE (email)
);
```

**AUTO_INCREMENT → シーケンス + トリガー (12c 未満または明示的な制御が必要な場合)：**

```sql
-- Oracle シーケンス
CREATE SEQUENCE customers_seq
    START WITH 1
    INCREMENT BY 1
    CACHE 20
    NOCYCLE;

CREATE TABLE customers (
    customer_id NUMBER        DEFAULT customers_seq.NEXTVAL PRIMARY KEY,
    email       VARCHAR2(255) NOT NULL,
    created_at  TIMESTAMP     DEFAULT SYSTIMESTAMP
);
```

### 文字列型

| MySQL | Oracle | 備考 |
|---|---|---|
| `CHAR(n)` | `CHAR(n)` | 直接的な同等物 |
| `VARCHAR(n)` | `VARCHAR2(n)` | MySQL では最大 65,535 バイト。Oracle では 4000 / 32767 バイト |
| `TINYTEXT` | `VARCHAR2(255)` | 最大 255 バイト |
| `TEXT` | `VARCHAR2(4000)` または `CLOB` | MySQL では最大 65,535 バイト |
| `MEDIUMTEXT` | `CLOB` | 最大 16 MB |
| `LONGTEXT` | `CLOB` | 最大 4 GB |
| `TINYBLOB` | `RAW(255)` | 最大 255 バイト |
| `BLOB` | `BLOB` | バイナリ・ラージ・オブジェクト |
| `MEDIUMBLOB` | `BLOB` | 最大 16 MB |
| `LONGBLOB` | `BLOB` | 最大 4 GB |
| `ENUM('a','b','c')` | `VARCHAR2(n)` + CHECK 制約 | 詳細は後述の例を参照 |
| `SET('a','b','c')` | `VARCHAR2(n)` または交差テーブル | 複数選択可能な MySQL 型。非正規化するか正規化する |

**ENUM → CHECK 制約：**

```sql
-- MySQL
CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    status   ENUM('pending','processing','shipped','delivered','cancelled') NOT NULL DEFAULT 'pending'
);

-- Oracle
CREATE TABLE orders (
    order_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status   VARCHAR2(20) DEFAULT 'pending' NOT NULL,
    CONSTRAINT chk_orders_status
        CHECK (status IN ('pending','processing','shipped','delivered','cancelled'))
);
```

### 日付と時間型

| MySQL | Oracle | 備考 |
|---|---|---|
| `DATE` | `DATE` | Oracle の DATE は時刻も保持する（未設定時は午前 0 時） |
| `TIME` | 同等の型なし | `VARCHAR2(8)` または `NUMBER` (秒数) として格納 |
| `DATETIME` | `DATE` または `TIMESTAMP` | 秒の小数部を保持するには TIMESTAMP を使用 |
| `TIMESTAMP` | `TIMESTAMP WITH TIME ZONE` | MySQL の TIMESTAMP は UTC に変換される。Oracle はタイムゾーンを明示的に格納 |
| `YEAR` | `NUMBER(4)` | 1 バイトの年を表す MySQL 固有の型 |

**重要な違い：** MySQL の `TIMESTAMP` は、挿入時に UTC に自動変換され、セッションのタイムゾーンに従って選択時に再度変換される。Oracle の `TIMESTAMP WITH TIME ZONE` は、オフセットをそのまま格納する。MySQL の暗黙的な UTC 変換に依存しているアプリケーションについては再検討すること。

```sql
-- 自動更新付きの MySQL TIMESTAMP
CREATE TABLE records (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- トリガーを使用した Oracle での同等実装
CREATE TABLE records (
    id         NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP
);

CREATE OR REPLACE TRIGGER trg_records_updated
BEFORE UPDATE ON records
FOR EACH ROW
BEGIN
    :NEW.updated_at := SYSTIMESTAMP;
END;
/
```

### 数値型

| MySQL | Oracle | 備考 |
|---|---|---|
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `FLOAT` | `BINARY_FLOAT` | IEEE 754 32 ビット |
| `DOUBLE` / `DOUBLE PRECISION` | `BINARY_DOUBLE` | IEEE 754 64 ビット |
| `BIT(n)` | `RAW(CEIL(n/8))` | ビット・フィールド型 |

### JSON と空間型

| MySQL | Oracle | 備考 |
|---|---|---|
| `JSON` | `JSON` (21c 以降) または `CLOB IS JSON` | Oracle 12c：IS JSON 制約付きの CLOB を使用 |
| `GEOMETRY` | `SDO_GEOMETRY` | Oracle Spatial。エディションによっては個別のライセンスが必要 |
| `POINT`, `LINESTRING` など | `SDO_GEOMETRY` subtypes | |

---

## SQL 方言の違い

### バッククォートによる識別子 → ダブルクォーテーションによる識別子

MySQL は識別子を囲むためにバッククォートを使用する（特に予約語と衝突する場合）。Oracle はダブルクォーテーションを使用する。最も安全なアプローチは、引用符を全く必要としないようにオブジェクト名を変更することである。

```sql
-- MySQL (バッククォートによる引用)
SELECT `order`, `desc`, `key` FROM `order_details`;
CREATE TABLE `user` (`group` INT, `select` VARCHAR(100));

-- Oracle (ダブルクォーテーションによる引用 — 可能な限り避けること)
SELECT "order", "desc", "key" FROM "order_details";

-- ベスト・プラクティス：予約語を避けるために名前を変更する
CREATE TABLE user_accounts (user_group NUMBER, selection VARCHAR2(100));
SELECT order_num, description, access_key FROM order_details;
```

**重要：** Oracle でダブルクォーテーションで囲まれた識別子は、大文字小文字が区別されるようになる。Oracle において `"MyTable"` と `mytable` は異なるオブジェクトである。Oracle では引用符なしの識別子（大文字に畳み込まれる）を推奨する。

### LIMIT → FETCH FIRST

```sql
-- MySQL
SELECT * FROM products ORDER BY price ASC LIMIT 10;
SELECT * FROM products ORDER BY price ASC LIMIT 10 OFFSET 30;

-- Oracle 12c 以降
SELECT * FROM products ORDER BY price ASC FETCH FIRST 10 ROWS ONLY;
SELECT * FROM products ORDER BY price ASC OFFSET 30 ROWS FETCH NEXT 10 ROWS ONLY;

-- Oracle 11g 以前
SELECT * FROM (SELECT * FROM products ORDER BY price ASC) WHERE ROWNUM <= 10;
```

### GROUP BY の動作

MySQL は (`ONLY_FULL_GROUP_BY` を含まない `sql_mode` の場合) 集計されていない、かつグループ化されていない列を選択することを許可する。Oracle は厳格な GROUP BY 準拠を強制する。

```sql
-- MySQL (寛容なモード — 動作はするが未定義の挙動)
SELECT dept_id, last_name, COUNT(*) FROM employees GROUP BY dept_id;

-- Oracle — エラー: ORA-00979: not a GROUP BY expression
-- last_name を GROUP BY に含めるか、集計関数を使用する必要がある：
SELECT dept_id, MAX(last_name) AS sample_name, COUNT(*) FROM employees GROUP BY dept_id;
```

### IF and IF NOT EXISTS

```sql
-- MySQL DDL (IF EXISTS / IF NOT EXISTS 付き)
CREATE TABLE IF NOT EXISTS audit_log (id INT PRIMARY KEY, action VARCHAR(100));
DROP TABLE IF EXISTS temp_staging;
ALTER TABLE orders ADD COLUMN IF NOT EXISTS notes TEXT;

-- Oracle 23c 以降 (ネイティブ・サポート)
CREATE TABLE IF NOT EXISTS audit_log (id NUMBER PRIMARY KEY, action VARCHAR2(100));
DROP TABLE IF EXISTS temp_staging;

-- Oracle 23c 未満：PL/SQL の例外処理を使用
BEGIN
    EXECUTE IMMEDIATE 'DROP TABLE temp_staging';
EXCEPTION
    WHEN OTHERS THEN
        IF SQLCODE != -942 THEN RAISE; END IF;
END;
/
```

### 文字列関数

| MySQL 関数 | Oracle 同等物 |
|---|---|
| `CONCAT(a, b, c)` | `a \|\| b \|\| c` または `CONCAT(CONCAT(a,b),c)` |
| `CONCAT_WS(',', a, b, c)` | `a \|\| ',' \|\| b \|\| ',' \|\| c` (手動結合) |
| `GROUP_CONCAT(col SEPARATOR ',')` | `LISTAGG(col, ',') WITHIN GROUP (ORDER BY col)` |
| `SUBSTRING(s, pos, len)` | `SUBSTR(s, pos, len)` |
| `LOCATE(sub, s)` | `INSTR(s, sub)` |
| `INSTR(s, sub)` | `INSTR(s, sub)` — 同様 |
| `LCASE(s)` / `LOWER(s)` | `LOWER(s)` |
| `UCASE(s)` / `UPPER(s)` | `UPPER(s)` |
| `TRIM(s)` | `TRIM(s)` — 同様 |
| `LTRIM(s)` | `LTRIM(s)` — 同様 |
| `RTRIM(s)` | `RTRIM(s)` — 同様 |
| `SPACE(n)` | `RPAD(' ', n)` または `LPAD(' ', n)` |
| `REPEAT(s, n)` | カスタム PL/SQL 関数を記述 |
| `REVERSE(s)` | 組み込み関数なし。PL/SQL 関数を使用 |
| `FORMAT(n, d)` | `TO_CHAR(n, 'FM999,999,999.00')` |
| `LEFT(s, n)` | `SUBSTR(s, 1, n)` |
| `RIGHT(s, n)` | `SUBSTR(s, -n)` |
| `LENGTH(s)` | `LENGTH(s)` — 同様 |
| `CHAR_LENGTH(s)` | `LENGTH(s)` |
| `ASCII(s)` | `ASCII(s)` — 同様 |
| `CHAR(n)` | `CHR(n)` |

**GROUP_CONCAT の例：**

```sql
-- MySQL
SELECT dept_id,
       GROUP_CONCAT(last_name ORDER BY last_name SEPARATOR ', ') AS employees
FROM emp
GROUP BY dept_id;

-- Oracle
SELECT dept_id,
       LISTAGG(last_name, ', ') WITHIN GROUP (ORDER BY last_name) AS employees
FROM emp
GROUP BY dept_id;
```

### 日付関数

| MySQL 関数 | Oracle 同等物 |
|---|---|
| `NOW()` | `SYSDATE` (小数秒なし) または `SYSTIMESTAMP` |
| `CURDATE()` | `TRUNC(SYSDATE)` |
| `CURTIME()` | `TO_CHAR(SYSDATE, 'HH24:MI:SS')` |
| `DATE(dt)` | `TRUNC(dt)` |
| `DATE_FORMAT(d, fmt)` | `TO_CHAR(d, fmt)` — フォーマット・マスクは異なる |
| `DATE_ADD(d, INTERVAL n DAY)` | `d + n` |
| `DATE_SUB(d, INTERVAL n DAY)` | `d - n` |
| `DATE_ADD(d, INTERVAL n MONTH)` | `ADD_MONTHS(d, n)` |
| `DATEDIFF(d1, d2)` | `TRUNC(d1) - TRUNC(d2)` |
| `TIMESTAMPDIFF(MONTH, d1, d2)` | `MONTHS_BETWEEN(d2, d1)` |
| `YEAR(d)` | `EXTRACT(YEAR FROM d)` |
| `MONTH(d)` | `EXTRACT(MONTH FROM d)` |
| `DAY(d)` | `EXTRACT(DAY FROM d)` |
| `HOUR(d)` | `EXTRACT(HOUR FROM d)` |
| `DAYOFWEEK(d)` | `TO_NUMBER(TO_CHAR(d, 'D'))` |
| `LAST_DAY(d)` | `LAST_DAY(d)` — 同様 |
| `STR_TO_DATE(s, fmt)` | `TO_DATE(s, fmt)` — フォーマット・マスクは異なる |
| `UNIX_TIMESTAMP(d)` | `(d - DATE '1970-01-01') * 86400` |
| `FROM_UNIXTIME(n)` | `DATE '1970-01-01' + n/86400` |

**日付フォーマット・マスクの違い：**

| MySQL | Oracle | 意味 |
|---|---|---|
| `%Y` | `YYYY` | 4 桁の年 |
| `%m` | `MM` | 月番号 |
| `%d` | `DD` | 日 |
| `%H` | `HH24` | 時 (0-23) |
| `%i` | `MI` | 分 |
| `%s` | `SS` | 秒 |

---

## ストアド・プロシージャの構文の違い

MySQL と Oracle のストアド・プロシージャ言語は根本的に異なる。MySQL の手続き型 SQL は比較的単純であるが、Oracle の PL/SQL は例外処理、レコード型、コレクション、およびパッケージを備えたより高度なものである。

### 基本的なプロシージャ構造

```sql
-- MySQL
DELIMITER $$
CREATE PROCEDURE update_customer_status(
    IN  p_customer_id INT,
    IN  p_status      VARCHAR(20),
    OUT p_rows_updated INT
)
BEGIN
    UPDATE customers
    SET status = p_status
    WHERE customer_id = p_customer_id;

    SET p_rows_updated = ROW_COUNT();
END$$
DELIMITER ;

-- Oracle PL/SQL
CREATE OR REPLACE PROCEDURE update_customer_status(
    p_customer_id IN  NUMBER,
    p_status      IN  VARCHAR2,
    p_rows_updated OUT NUMBER
)
AS
BEGIN
    UPDATE customers
    SET status = p_status
    WHERE customer_id = p_customer_id;

    p_rows_updated := SQL%ROWCOUNT;
END update_customer_status;
/
```

### 変数と宣言

```sql
-- MySQL
BEGIN
    DECLARE v_total DECIMAL(10,2) DEFAULT 0;
    DECLARE v_name VARCHAR(100);
    SET v_total = 100.00;
    SET v_name = 'Alice';
END

-- Oracle PL/SQL
DECLARE
    v_total NUMBER(10,2) := 0;
    v_name  VARCHAR2(100);
BEGIN
    v_total := 100.00;
    v_name  := 'Alice';
END;
/
```

### 制御フロー

```sql
-- MySQL IF-ELSEIF-ELSE
IF v_score >= 90 THEN
    SET v_grade = 'A';
ELSEIF v_score >= 80 THEN
    SET v_grade = 'B';
ELSE
    SET v_grade = 'C';
END IF;

-- Oracle PL/SQL
IF v_score >= 90 THEN
    v_grade := 'A';
ELSIF v_score >= 80 THEN        -- 注意：ELSEIF ではなく ELSIF
    v_grade := 'B';
ELSE
    v_grade := 'C';
END IF;
```

```sql
-- MySQL WHILE ループ
WHILE v_counter <= 10 DO
    SET v_sum = v_sum + v_counter;
    SET v_counter = v_counter + 1;
END WHILE;

-- Oracle PL/SQL
WHILE v_counter <= 10 LOOP
    v_sum := v_sum + v_counter;
    v_counter := v_counter + 1;
END LOOP;
```

### カーソル処理

```sql
-- MySQL カーソル
DECLARE cur CURSOR FOR SELECT id, name FROM products WHERE active = 1;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
OPEN cur;
read_loop: LOOP
    FETCH cur INTO v_id, v_name;
    IF done THEN LEAVE read_loop; END IF;
    -- process...
END LOOP;
CLOSE cur;

-- Oracle PL/SQL カーソル
DECLARE
    CURSOR cur IS SELECT id, name FROM products WHERE active = 1;
    v_id   products.id%TYPE;
    v_name products.name%TYPE;
BEGIN
    OPEN cur;
    LOOP
        FETCH cur INTO v_id, v_name;
        EXIT WHEN cur%NOTFOUND;
        -- process...
    END LOOP;
    CLOSE cur;
END;
/

-- Oracle: よりシンプルな FOR カーソル・ループ (推奨)
BEGIN
    FOR rec IN (SELECT id, name FROM products WHERE active = 1) LOOP
        -- rec.id, rec.name を使用して処理
        NULL;
    END LOOP;
END;
/
```

### 例外処理

```sql
-- MySQL の例外処理
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    RESIGNAL;
END;

-- Oracle PL/SQL
BEGIN
    -- ... DML ...
EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20001, 'Duplicate key violation');
    WHEN NO_DATA_FOUND THEN
        -- 行が見つからない場合の処理
        NULL;
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/
```

---

## mysqldump から Oracle へのワークフロー

### ステップ 1 — MySQL からデータをエクスポート

```bash
# スキーマのみをエクスポート
mysqldump -u root -p --no-data mydb > schema.sql

# テーブルごとに CSV 形式でデータをエクスポート (Oracle へのロードに最適)
mysql -u root -p mydb -e "SELECT * FROM customers INTO OUTFILE '/tmp/customers.csv'
    FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"'
    LINES TERMINATED BY '\n';"

# INSERT ステートメントを含む mysqldump (参考資料としては有用だが大幅な変換が必要)
mysqldump -u root -p --skip-extended-insert mydb customers > customers_inserts.sql
```

### ステップ 2 — スキーマの変換

自動化または実施すべき主な変換：
- `\`` (バッククォート) を削除 (または引用が必要な場合は `"` に置換)
- `AUTO_INCREMENT` を `GENERATED ALWAYS AS IDENTITY` に置換
- `INT` を `NUMBER(10)` などに置換
- `VARCHAR(n)` を `VARCHAR2(n)` に置換
- `DATETIME` を `TIMESTAMP` に置換
- `ENGINE=InnoDB`, `CHARSET=utf8mb4` などの MySQL 固有オプションを削除
- `TINYINT(1)` を `NUMBER(1)` に置換

**SQL Developer Migration Workbench** を使用すると、これらのスキーマ変換の多くを自動化できる（`oracle-migration-tools.md` を参照）。注：ora2pg は Oracle から PostgreSQL への移行用であり、MySQL から Oracle への移行には適用できない。

### ステップ 3 — SQL*Loader を使用してデータをロード

```
-- SQL*Loader コントロール・ファイル: customers.ctl
OPTIONS (SKIP=0, ROWS=5000, DIRECT=TRUE, ERRORS=100)
LOAD DATA
CHARACTERSET UTF8
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
    status,
    created_at  TIMESTAMP "YYYY-MM-DD HH24:MI:SS"
)
```

```bash
sqlldr userid=myuser/mypass@mydb control=customers.ctl log=customers.log bad=customers.bad
```

### ステップ 4 — ロード後のシーケンス

データをロードした後、アイデンティティ列のシーケンスをロードされた最大値から開始するように更新する。

```sql
-- シーケンスを使用している場合 (12c 未満)：
DECLARE
    v_max NUMBER;
BEGIN
    SELECT MAX(customer_id) INTO v_max FROM customers;
    EXECUTE IMMEDIATE 'ALTER SEQUENCE customers_seq RESTART START WITH ' || (v_max + 1);
END;
/

-- アイデンティティ列を使用している場合 (12c 以降、再作成または ALTER TABLE が必要)：
ALTER TABLE customers MODIFY customer_id GENERATED ALWAYS AS IDENTITY (START WITH LIMIT VALUE);
```

---

## ベスト・プラクティス

1. **移行前に MySQL を厳格な（strict）SQL モードで実行する。** `sql_mode = 'ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION'` に設定し、Oracle 移行を開始する前にすべてのエラーを修正すること。これにより、GROUP BY や型強制の問題が表面化する。

2. **ENUM および SET 列を正規化する。** Oracle では ENUM に相当するものとして CHECK 制約が機能するが、SET（複数値）列には交差テーブルまたは JSON 列が必要である。スキーマ変更を早期に計画すること。

3. **MySQL の「零日付（zero-dates）」をチェックする。** MySQL は `0000-00-00` を日付値として許可するが、Oracle の DATE 型にはそのような表現はない。NULL またはセンチネル日付値に置き換えること。

4. **ON UPDATE CASCADE を確認する。** Oracle はカスケード削除をネイティブにサポートしているが、主キーに対するカスケード更新はサポートしていない。`ON UPDATE CASCADE` に依存しているアプリケーションは、アプリケーション・レベルでの変更が必要になる。

5. **文字セットに注意する。** MySQL の `utf8` は実際には 3 バイトの UTF-8（絵文字不可）であり、`utf8mb4` が真の 4 バイト UTF-8 である。Oracle の `AL32UTF8` は完全な UTF-8 である。すべての Unicode 文字を処理できるように、Oracle 側は `AL32UTF8` に設定すること。

6. **GROUP_CONCAT / LISTAGG の長さ制限をテストする。** Oracle 12c の `LISTAGG` は、連結された出力が 4000 文字を超えるとエラーを発生させる。`LISTAGG(...) ON OVERFLOW TRUNCATE` (12.2 以降) を使用するか、CLOB を使用したワークアラウンドを検討すること。

---

## よくある移行の落とし穴

**落とし穴 1 — MySQL の大文字小文字を区別しない文字列比較：**
MySQL の比較は、`utf8_general_ci` 照合順序の場合、デフォルトで大文字小文字が区別されない。Oracle の比較は常に大文字小文字が区別される。
```sql
-- MySQL: 大文字小文字が混在していても結果を返す
SELECT * FROM users WHERE username = 'ALICE';  -- 'alice' を見つける

-- Oracle: 何も返さない
SELECT * FROM users WHERE username = 'ALICE';  -- 'ALICE' のみを見つける

-- Oracle での修正：挿入時にケースを正規化するか、UPPER() 関数を使用した比較を行う
SELECT * FROM users WHERE UPPER(username) = 'ALICE';
```

**落とし穴 2 — MySQL での整数除算が実数を返す場合：**
```sql
-- MySQL: 整数除算
SELECT 5 / 2;  -- 2.5000 を返す (MySQL は実際には小数を返す)
SELECT 5 DIV 2; -- 2 を返す (整数除算)

-- Oracle: 常に正確な結果を返す
SELECT 5 / 2 FROM DUAL;    -- 2.5 を返す
SELECT TRUNC(5/2) FROM DUAL; -- 2 を返す
```

**落とし穴 3 — NULL 安全な等価比較：**
```sql
-- MySQL: NULL 安全な等価演算子
SELECT * FROM t WHERE a <=> b;  -- 両方が NULL の場合に true

-- Oracle での同等実装
SELECT * FROM t WHERE (a = b OR (a IS NULL AND b IS NULL));
-- または： DECODE(a, b, 1, 0) = 1 (DECODE は NULL 同士を等しいとみなす)
```

**落とし穴 4 — DDL による暗黙的なコミット：**
MySQL (デフォルト設定) と Oracle は共に DDL 実行時にコミットするが、MySQL は DDL ステートメントの実行前に開いているトランザクションもコミットする。アプリケーションのトランザクション管理においてこれを考慮すること。

**落とし穴 5 — MySQL の IF() 関数：**
```sql
-- MySQL
SELECT IF(status = 'active', 'Yes', 'No') FROM customers;

-- Oracle
SELECT CASE WHEN status = 'active' THEN 'Yes' ELSE 'No' END FROM customers;
-- または DECODE を使用：
SELECT DECODE(status, 'active', 'Yes', 'No') FROM customers;
```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE (GENERATED AS IDENTITY)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c SQL Language Reference — String Functions (LISTAGG)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/LISTAGG.html)
- [Oracle Database 19c PL/SQL Language Reference — Control Structures](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-control-statements.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [Oracle SQL Developer Migration Workbench](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
