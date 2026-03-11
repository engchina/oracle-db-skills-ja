# SQLite から Oracle への移行

## 概要

SQLite は、データベース全体を単一のファイルに保存する、サーバーレスの組み込み型データベース・エンジンである。シンプルさ、移植性、および低リソース環境向けに設計されており、モバイル・アプリ、IoT デバイス、デスクトップ・ソフトウェア、およびテスト環境に組み込まれている。一方、Oracle Database は、クライアント/サーバー・アーキテクチャ、複雑なメモリー管理、および大規模な同時マルチユーザー・ワークロードに適した豊富な機能を備えた、フル機能のエンタープライズ RDBMS である。

SQLite から Oracle への移行は、通常、規模の拡大、同時実行要件の増加、または運用上の成熟度の向上を意味する。課題としては、SQLite 独自の型アフィニティ（Type Affinity）システム（厳格な型ではない）、Oracle に同等のものがない SQLite 固有の PRAGMA ステートメント、および単一ファイルの組み込みデータベースからマルチプロセス・エンタープライズ・サーバーへの概念的な転換が挙げられる。

---

## SQLite の型アフィニティ vs Oracle の厳格な型

SQLite は、厳格なデータ型ではなく、**型アフィニティ（Type Affinity）** という概念を使用する。列の宣言された型は「ヒント」であり、強制ではない。どの列にも任意の型の任意の値を格納できる。これに対し、Oracle は厳格な型指定を強制する。

### SQLite の型アフィニティ・ルール

SQLite は、宣言された型名に基づいてアフィニティを割り当てる：
- `INT` という単語を含む型 → INTEGER アフィニティ
- `CHAR`, `CLOB`, または `TEXT` を含む型 → TEXT アフィニティ
- `BLOB` 型または型指定なし → BLOB アフィニティ (任意の型を格納)
- `REAL`, `FLOA`, または `DOUB` を含む型 → REAL アフィニティ
- その他すべて → NUMERIC アフィニティ

これは、SQLite ではエラーなしに以下が可能であることを意味する：
```sql
-- SQLite: これは実際に動作する (型アフィニティは推奨事項に過ぎない)
CREATE TABLE demo (
    id   INTEGER PRIMARY KEY,
    val  INTEGER
);
INSERT INTO demo (id, val) VALUES (1, 'not an integer');  -- TEXT として格納される
INSERT INTO demo (id, val) VALUES (2, 3.14);              -- REAL として格納される
```

Oracle では、これらのインサートは型不一致エラーで拒否される。

### 型マッピング表

| SQLite 宣言型 | Oracle 型 | 備考 |
|---|---|---|
| `INTEGER` | `NUMBER(10)` | SQLite の INTEGER は最大 8 バイト。BIGINT 範囲には NUMBER(19) を使用 |
| `INTEGER PRIMARY KEY` | `NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY` | SQLite の ROWID エイリアス |
| `REAL` | `BINARY_DOUBLE` | 64 ビット IEEE 754 浮動小数点 |
| `TEXT` | `VARCHAR2(4000)` または `CLOB` | 期待される長さに依存 |
| `BLOB` | `BLOB` | バイナリ・データ |
| `NUMERIC` | `NUMBER` | その他すべてをキャッチ |
| `BOOLEAN` | CHECK (0,1) 付きの `NUMBER(1)` | SQLite は TRUE/FALSE を 1/0 として格納 |
| `DATE` | `DATE` | SQLite は日付を TEXT または INTEGER として格納 |
| `DATETIME` | `TIMESTAMP` | SQLite は TEXT `YYYY-MM-DD HH:MM:SS` として格納 |
| `FLOAT` | `BINARY_FLOAT` または `BINARY_DOUBLE` | |
| `DOUBLE` | `BINARY_DOUBLE` | |
| `VARCHAR(n)` | `VARCHAR2(n)` | SQLite は n を無視。Oracle は強制 |
| `CHAR(n)` | `CHAR(n)` | SQLite は n を無視。Oracle は強制 |
| `NCHAR(n)` | `NCHAR(n)` | |
| `DECIMAL(p,s)` | `NUMBER(p,s)` | SQLite は浮動小数点または整数として格納 |

### スキーマ変換の例

```sql
-- SQLite スキーマ (.schema コマンドによる出力)
CREATE TABLE products (
    product_id   INTEGER  PRIMARY KEY AUTOINCREMENT,
    product_name TEXT     NOT NULL,
    price        REAL     NOT NULL DEFAULT 0.0,
    quantity     INTEGER  DEFAULT 0,
    description  TEXT,
    image_data   BLOB,
    is_active    INTEGER  NOT NULL DEFAULT 1,
    created_at   TEXT     DEFAULT (datetime('now')),
    category_id  INTEGER  REFERENCES categories(category_id)
);

-- Oracle での同等実装
CREATE TABLE products (
    product_id   NUMBER       GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    product_name VARCHAR2(500)  NOT NULL,
    price        NUMBER(15,4)   NOT NULL DEFAULT 0,
    quantity     NUMBER(10)     DEFAULT 0,
    description  CLOB,
    image_data   BLOB,
    is_active    NUMBER(1)      NOT NULL DEFAULT 1,
    created_at   TIMESTAMP      DEFAULT SYSTIMESTAMP,
    category_id  NUMBER(10),
    CONSTRAINT chk_products_is_active CHECK (is_active IN (0,1)),
    CONSTRAINT fk_products_category
        FOREIGN KEY (category_id) REFERENCES categories(category_id)
);
```

---

## SQLite の日付格納パターン

SQLite にはネイティブの DATE 型がない。アプリケーションは通常、以下の 3 つの方法で日付を格納する。
1. **TEXT** (ISO 8601 形式) : `'2024-01-15 14:30:00'`
2. **INTEGER** (Unix エポック秒) : `1705330200`
3. **REAL** (ユリウス日番号) : `2460325.1041667`

それぞれ、Oracle へのロード戦略が異なる。

```sql
-- TEXT として格納された SQLite の日付: '2024-01-15 14:30:00'
-- Oracle SQL*Loader マッピング:
created_at TIMESTAMP "YYYY-MM-DD HH24:MI:SS"

-- INTEGER (Unix タイムスタンプ) として格納された SQLite の日付
-- Oracle での変換：
SELECT DATE '1970-01-01' + created_at_unix / 86400 AS created_at FROM staging;
-- または TIMESTAMP 精度の場合：
SELECT TIMESTAMP '1970-01-01 00:00:00' +
       NUMTODSINTERVAL(created_at_unix, 'SECOND') AS created_at
FROM staging;

-- REAL (ユリウス日) として格納された SQLite の日付
-- Oracle での変換 (ユリウス日から日付へ) :
SELECT TO_DATE('4713-01-01', 'YYYY-MM-DD', 'NLS_CALENDAR=JULIAN') +
       (julian_day - 0.5) AS created_at  -- 近似値
FROM staging;
-- より簡単な方法：Oracle の TO_DATE とユリウス形式を使用
SELECT TO_DATE(ROUND(julian_day), 'J') AS created_at FROM staging;
```

---

## SQLite PRAGMA — Oracle に同等機能なし

SQLite は、データベース構成やメタデータ・クエリに PRAGMA ステートメントを使用する。Oracle にはこれらは存在しない。以下に、それぞれの目的と Oracle でのアプローチを示す。

| SQLite PRAGMA | 目的 | Oracle でのアプローチ |
|---|---|---|
| `PRAGMA journal_mode = WAL` | Write-Ahead Logging | Oracle は REDO ログを使用。ユーザー設定は不要 |
| `PRAGMA foreign_keys = ON` | FK 強制の有効化 | Oracle は常に FK を強制（無効化しない限り） |
| `PRAGMA synchronous = NORMAL` | ディスク同期制御 | `FAST_START_MTTR_TARGET` や `LOG_BUFFER` を使用 |
| `PRAGMA cache_size = 10000` | ページ・キャッシュ・サイズ | `DB_CACHE_SIZE` (SGA パラメータ) を使用 |
| `PRAGMA page_size = 4096` | DB ページ・サイズ | `DB_BLOCK_SIZE` (作成時に設定) を使用 |
| `PRAGMA auto_vacuum = FULL` | 自動領域回収 | `ALTER TABLE ... SHRINK SPACE` を使用 |
| `PRAGMA integrity_check` | データ整合性チェック | `DBMS_REPAIR`, `ANALYZE TABLE ... VALIDATE STRUCTURE` |
| `PRAGMA table_info(tbl)` | 列メタデータ | `SELECT * FROM user_tab_columns WHERE table_name = 'TBL'` |
| `PRAGMA index_list(tbl)` | 索引一覧 | `SELECT * FROM user_indexes WHERE table_name = 'TBL'` |
| `PRAGMA foreign_key_list(tbl)` | FK 制約一覧 | `SELECT * FROM user_constraints WHERE table_name = 'TBL'` |
| `PRAGMA compile_options` | ビルド時オプション | 該当なし |
| `PRAGMA database_list` | アタッチされた DB 一覧 | `SELECT * FROM v$database` |
| `PRAGMA user_version` | アプリ・スキーマ・バージョン | 独自のスキーマ・バージョン表を作成する |

### アプリケーション・スキーマのバージョン管理

SQLite アプリケーションは、スキーマ移行の追跡に `PRAGMA user_version` をよく使用する。Oracle では、専用のバージョン表を実装すること。

```sql
-- PRAGMA user_version に相当する Oracle での実装例
CREATE TABLE schema_version (
    version_number NUMBER(10)  NOT NULL,
    applied_at     TIMESTAMP   DEFAULT SYSTIMESTAMP,
    description    VARCHAR2(500)
);

INSERT INTO schema_version (version_number, description) VALUES (1, 'Initial schema');
COMMIT;

-- 現在のバージョンを照会
SELECT MAX(version_number) FROM schema_version;
```

---

## SQLite SQL 方言 → Oracle SQL

### AUTOINCREMENT vs ROWID

SQLite の `INTEGER PRIMARY KEY` は内部 ROWID のエイリアスである。`AUTOINCREMENT` を付けると、ID が単調増加することが保証される。

```sql
-- SQLite
CREATE TABLE t (id INTEGER PRIMARY KEY AUTOINCREMENT);
-- または
CREATE TABLE t (id INTEGER PRIMARY KEY);  -- 削除された行の ID が再利用される可能性がある

-- Oracle: 厳密なインクリメントには常に GENERATED ALWAYS AS IDENTITY を使用する
CREATE TABLE t (id NUMBER GENERATED ALWAYS AS IDENTITY (ORDER) PRIMARY KEY);
```

### 文字列関数

| SQLite 関数 | Oracle 同等物 |
|---|---|
| `LENGTH(s)` | `LENGTH(s)` — 同様 |
| `UPPER(s)` | `UPPER(s)` — 同様 |
| `LOWER(s)` | `LOWER(s)` — 同様 |
| `TRIM(s)` | `TRIM(s)` — 同様 |
| `LTRIM(s)` | `LTRIM(s)` — 同様 |
| `RTRIM(s)` | `RTRIM(s)` — 同様 |
| `SUBSTR(s, pos, len)` | `SUBSTR(s, pos, len)` — 同様 |
| `INSTR(s, sub)` | `INSTR(s, sub)` — 同様 |
| `REPLACE(s, old, new)` | `REPLACE(s, old, new)` — 同様 |
| `PRINTF('%05d', n)` | `TO_CHAR(n, '00000')` |
| `FORMAT(fmt, args)` (SQLite 3.38 以降) | `TO_CHAR` / `LPAD` 等 |
| `HEX(blob)` | `RAWTOHEX(blob)` |
| `QUOTE(s)` | 同等の関数なし。パラメータ化クエリを使用 |
| `SOUNDEX(s)` | `SOUNDEX(s)` — Oracle でも利用可能 |
| `LIKE(pattern, s)` | `s LIKE pattern` (引数が逆) |
| `GLOB(pattern, s)` | 同等の関数なし。パターンを変換して `REGEXP_LIKE` を使用 |

### 日付と時刻関数

```sql
-- SQLite 日付関数
SELECT DATE('now');
SELECT DATE('now', '+30 days');
SELECT DATETIME('now', 'localtime');
SELECT STRFTIME('%Y-%m', date_col);
SELECT JULIANDAY('now');
SELECT UNIXEPOCH('now');
SELECT UNIXEPOCH(datetime_col);

-- Oracle での同等操作
SELECT TRUNC(SYSDATE) FROM DUAL;
SELECT TRUNC(SYSDATE) + 30 FROM DUAL;
SELECT SYSDATE FROM DUAL;  -- すでにローカル・タイム
SELECT TO_CHAR(date_col, 'YYYY-MM') FROM dual;
SELECT TO_NUMBER(TO_CHAR(SYSDATE, 'J')) FROM DUAL;
SELECT (SYSDATE - DATE '1970-01-01') * 86400 FROM DUAL;
SELECT (date_col - DATE '1970-01-01') * 86400 FROM dual;
```

### LIMIT / OFFSET

```sql
-- SQLite
SELECT * FROM products ORDER BY name LIMIT 10;
SELECT * FROM products ORDER BY name LIMIT 10 OFFSET 20;
SELECT * FROM products ORDER BY name LIMIT -1 OFFSET 5;  -- SQLite: -1 は制限なしを意味する

-- Oracle 12c 以降
SELECT * FROM products ORDER BY name FETCH FIRST 10 ROWS ONLY;
SELECT * FROM products ORDER BY name OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
SELECT * FROM products ORDER BY name OFFSET 5 ROWS;  -- 上限なし
```

### UPSERT (INSERT OR REPLACE / ON CONFLICT)

```sql
-- SQLite
INSERT OR REPLACE INTO settings (key, value) VALUES ('theme', 'dark');
INSERT OR IGNORE INTO settings (key, value) VALUES ('theme', 'dark');
INSERT INTO settings (key, value) VALUES ('theme', 'dark')
    ON CONFLICT(key) DO UPDATE SET value = excluded.value;

-- Oracle (MERGE ステートメント)
MERGE INTO settings tgt
USING (SELECT 'theme' AS key, 'dark' AS value FROM DUAL) src
ON (tgt.key = src.key)
WHEN MATCHED THEN
    UPDATE SET tgt.value = src.value
WHEN NOT MATCHED THEN
    INSERT (key, value) VALUES (src.key, src.value);
```

### ウィンドウ関数

SQLite はバージョン 3.25 (2018 年) でウィンドウ関数をサポートした。Oracle はより以前からサポートしており、構文はほぼ互換性がある。

```sql
-- SQLite (3.25 以降) および Oracle (同一構文)
SELECT name, salary,
       RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS dept_rank,
       SUM(salary) OVER (ORDER BY hire_date ROWS UNBOUNDED PRECEDING) AS running_salary
FROM employees;
```

### 共通表式 (CTE)

SQLite (3.8.3 以降) と Oracle は共に、互換性のある構文で再帰的および非再帰的 CTE をサポートしている。

```sql
-- SQLite 再帰的 CTE (組織階層) および Oracle 同等実装
WITH RECURSIVE org_tree (emp_id, name, manager_id, level) AS (
    SELECT emp_id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, t.level + 1
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.emp_id
)
SELECT * FROM org_tree ORDER BY level, name;
```

---

## SQLite からのデータ抽出

### 方法 1 — SQLite .dump (SQL INSERT 形式)

```bash
# すべてのデータを INSERT ステートメントとしてエクスポート
sqlite3 myapp.db .dump > dump.sql

# 特定のテーブルをエクスポート
sqlite3 myapp.db ".dump products" > products_dump.sql

# CSV としてエクスポート
sqlite3 -header -csv myapp.db "SELECT * FROM products;" > products.csv
```

`.dump` の出力には SQLite 固有の構文が含まれるため、Oracle にロードする前に手動での変換（CREATE TABLE ステートメント、AUTOINCREMENT 等）が必要である。

### 方法 2 — CSV エクスポートと SQL*Loader

```bash
# ヘッダー付き CSV としてエクスポート
sqlite3 -header -csv myapp.db \
  "SELECT product_id, product_name, price, quantity, is_active,
          strftime('%Y-%m-%d %H:%M:%S', created_at) AS created_at
   FROM products;" > products.csv
```

```
-- SQL*Loader コントロール・ファイル
OPTIONS (DIRECT=TRUE, SKIP=1)
LOAD DATA
INFILE 'products.csv'
APPEND
INTO TABLE products
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
TRAILING NULLCOLS
(
    product_id,
    product_name,
    price,
    quantity,
    is_active,
    created_at TIMESTAMP "YYYY-MM-DD HH24:MI:SS"
)
```

### 方法 3 — Python ETL スクリプト

複雑なデータや型変換が必要な場合は、Python スクリプトを使用するのが最も制御しやすい。

```python
import sqlite3
import oracledb  # python-oracledb を使用
from datetime import datetime

# ソースに接続
src_conn = sqlite3.connect('myapp.db')
src_conn.row_factory = sqlite3.Row
src_cur = src_conn.cursor()

# ターゲットに接続 (Thin モード)
tgt_conn = oracledb.connect(user='user', password='pass', dsn='localhost:1521/ORCL')
tgt_cur = tgt_conn.cursor()

# データの移行
src_cur.execute("SELECT * FROM products")
rows = src_cur.fetchall()

insert_sql = """
INSERT INTO products (product_id, product_name, price, quantity,
                      is_active, created_at)
VALUES (:1, :2, :3, :4, :5, TO_TIMESTAMP(:6, 'YYYY-MM-DD HH24:MI:SS'))
"""

batch = []
for row in rows:
    batch.append((
        row['product_id'],
        row['product_name'],
        float(row['price']) if row['price'] else 0.0,
        int(row['quantity']) if row['quantity'] else 0,
        1 if row['is_active'] else 0,
        row['created_at']
    ))

tgt_cur.executemany(insert_sql, batch)
tgt_conn.commit()

src_conn.close()
tgt_conn.close()
```

---

## 拡張に関する考慮事項：組み込みからエンタープライズへ

SQLite はシングル・ライター、組み込み用途に最適化されている。Oracle への移行では、いくつかの設計前提を再考する必要がある。

### 接続管理

```
SQLite:                          Oracle:
- 単一プロセス、構成不要       - クライアント/サーバー、リスナー必須
- ファイル・ベースのロック       - マルチバージョン一貫性制御 (MVCC)
- 接続プール不要              - 接続プール (DRCP, UCP 等) を使用
- 一度に 1 つのライター         - 数千の同時ライター
```

### トランザクション・バウンダリ

SQLite の WAL モードでは 1 つのライターと複数のリーダーが可能。Oracle の MVCC では、無制限の同時リーダーとライターが可能。SQLite のシリアル化された書き込み向けに設計されたアプリケーションは、Oracle の同時書き込みシナリオを正しく処理できない可能性があるため、トランザクションの分離に関する前提条件を確認すること。

### ファイル参照と LOB 処理

SQLite BLOB のパフォーマンスは限定的であるため、ファイル・コンテンツ自体ではなくファイル・パスを保存することが多い。Oracle の BLOB および SecureFiles は非常に高いパフォーマンスを提供するため、移行時にコンテンツをデータベース内に取り込むことを検討する良い機会となる。

```sql
-- SQLite パターン (ファイル・パスを格納)
CREATE TABLE documents (
    id       INTEGER PRIMARY KEY,
    filepath TEXT   -- '/var/app/uploads/doc123.pdf'
);

-- Oracle 案 2: コンテンツをインライン化 (エンタープライズ向け)
CREATE TABLE documents (
    id       NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    content  BLOB
) LOB (content) STORE AS SECUREFILE (
    DEDUPLICATE
    COMPRESS HIGH
);
```

---

## ベスト・プラクティス

1. **ダック・タイピング（Duck-typed）された列を監査する。** 移行前に、SQLite の `typeof()` 関数を使用して各列に実際に保存されている型を調査すること。ある列に INTEGER と TEXT が混在している可能性がある。

2. **初日から Oracle で制約を適用する。** SQLite の制約適用はオプションである。この移行を機に、NOT NULL, CHECK, FK 制約を適切に追加すること。

3. **VARCHAR2 列のサイズを適切に設定する。** SQLite の TEXT は事実上無制限である。Oracle の VARCHAR2 のサイズを宣言する前に、実際の最大長を調査すること。

4. **INSERT OR REPLACE から MERGE への変換をテストする。** SQLite の `INSERT OR REPLACE` は、衝突時に行を一度削除してから挿入する（他の列のデータが消える可能性がある）。対して Oracle の `MERGE` はその場で更新する。

5. **SQLite の集計関数を置き換える。** SQLite の `group_concat()` は、Oracle の `LISTAGG` に置き換える。

---

## よくある移行の落とし穴

**落とし穴 1 — SQLite の BOOLEAN は実際には INTEGER である：**
`WHERE is_active = TRUE` (1 として格納) は SQLite では動作するが、Oracle では `TRUE` に PL/SQL 以外で意味がないため失敗する。`WHERE is_active = 1` を使用すること。

**落とし穴 2 — NULL 算術：**
SQLite と Oracle は共に NULL を算術演算で伝播させる（NULL + 1 = NULL）が、型強制における `COALESCE` の挙動が端的なケースで異なる場合がある。

**落とし穴 3 — LIKE パターンの大文字小文字の区別：**
SQLite の `LIKE` は ASCII 文字に対して大文字小文字を区別しない。Oracle の `LIKE` は区別する。

**落とし穴 4 — AUTOINCREMENT なしの PRIMARY KEY による ID 再利用：**
SQLite で `AUTOINCREMENT` なしの `INTEGER PRIMARY KEY` を使用している場合、削除された最大 ID が再利用されることがある。Oracle のアイデンティティ列は通常再利用されない。

**落とし穴 5 — GLOB パターンの構文：**
SQLite の `GLOB` 演算子は Unix スタイルのワイルドカードを使用する。Oracle には `GLOB` はないため、`REGEXP_LIKE` や `LIKE` に変換する必要がある。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c SQL Language Reference — Row Limiting (FETCH FIRST)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/SELECT.html)
- [Oracle Database 19c SQL Language Reference — MERGE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/MERGE.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [python-oracledb ドキュメント](https://python-oracledb.readthedocs.io/en/latest/)
