# Snowflake から Oracle への移行

## 概要

Snowflake は、ストレージとコンピュートの分離、オンデマンド処理のための仮想ウェアハウス、PostgreSQL スタイルの構文に独自の拡張を加えた SQL 方言など、独自のアーキテクチャを持つクラウド・ネイティブなデータ・プラットフォームである。これに対し、Oracle は、オンプレミス、クラウド (Oracle Cloud Infrastructure)、または Oracle Autonomous Database としてデプロイ可能な、伝統的な共有ディスク型 RDBMS である。Snowflake から Oracle への移行では、通常、分析ワークロードを Oracle Autonomous Data Warehouse (ADW) やオンプレミスの Oracle データウェアハウス構成へ移行することになる。

主な違いには、Snowflake の半構造化データ型 (VARIANT, OBJECT, ARRAY)、Time Travel 機能（Oracle Flashback に相当）、仮想ウェアハウスのサイジング（Oracle のリソース管理に相当）、および SQL 方言の違いが挙げられる。

---

## Snowflake アーキテクチャ → Oracle アーキテクチャ

### 組織 / アカウント / データベース / スキーマ / テーブル

```
Snowflake の階層:              Oracle の階層:
Organization (組織)            (クラウド・アカウントまたはデータ・センター)
└── Account (アカウント)        └── Oracle インスタンス / CDB
    └── Database (データベース)    └── PDB (プラガブル・データベース)
        └── Schema (スキーマ)          └── スキーマ (= ユーザー)
            └── Table/View (表/ビュー)     └── 表/ビュー
```

**主なマッピング：**

| Snowflake オブジェクト | Oracle の同等機能 | 備考 |
|---|---|---|
| アカウント (Account) | Oracle インスタンスまたは CDB | 最上位レベルでの 1 対 1 のマッピング |
| データベース (Database) | Oracle PDB またはスキーマ | CDB を使用する場合は、論理データベースごとに PDB を使用 |
| スキーマ (Schema) | Oracle スキーマ (= ユーザー) | Snowflake のスキーマごとに 1 つの Oracle ユーザーを作成 |
| 仮想ウェアハウス (Virtual warehouse) | Oracle リソース・プラン / パラレル設定 | 直接的な同等機能はない — リソース・マネージャのセクションを参照 |
| ステージ (Stage) | Oracle ディレクトリ + 外部表 | 内部/外部ステージの代替 |
| Snowpipe | Oracle GoldenGate / ODI | 継続的なロード・パイプライン |
| タスク (Task) / ストリーム (Stream) | Oracle Scheduler + MV ログ | 増分処理用 |
| タイム・トラベル (Time Travel) | Oracle Flashback | Flashback のセクションを参照 |

### 仮想ウェアハウス → Oracle の並列処理

Snowflake の仮想ウェアハウスは、T シャツ・サイズ（X-Small から 6X-Large）で選ぶコンピュート・クラスターである。Oracle の並列処理は、以下のように構成する：

```sql
-- テーブルのパラレル度を設定 (大規模なウェアハウスに相当)
ALTER TABLE fact_sales PARALLEL (DEGREE 16);

-- セッションのパラレル DML を有効化 (一括操作用)
ALTER SESSION ENABLE PARALLEL DML;

-- マルチユーザー・ワークロードの分離のためのリソース・マネージャ・プラン
-- (Snowflake のマルチクラスター・ウェアハウスに相当)
-- 詳細は oracle-migration-tools.md の DBMS_RESOURCE_MANAGER の例を参照
```

---

## データ型マッピング

### 標準型

| Snowflake | Oracle | 備考 |
|---|---|---|
| `NUMBER(p,s)` / `DECIMAL(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `INT` / `INTEGER` | `NUMBER(38,0)` | Snowflake の INTEGER は NUMBER(38,0) |
| `BIGINT` | `NUMBER(19)` | |
| `SMALLINT` | `NUMBER(5)` | |
| `TINYINT` | `NUMBER(3)` | |
| `FLOAT` / `FLOAT4` / `FLOAT8` | `BINARY_DOUBLE` | |
| `DOUBLE` / `DOUBLE PRECISION` | `BINARY_DOUBLE` | |
| `REAL` | `BINARY_FLOAT` | |
| `VARCHAR(n)` | `VARCHAR2(n)` | Snowflake 最大 16,777,216、Oracle 最大 4,000/32,767 |
| `CHAR(n)` | `CHAR(n)` | |
| `STRING` | `VARCHAR2(4000)` または `CLOB` | Snowflake における VARCHAR のエイリアス |
| `TEXT` | `CLOB` | |
| `BOOLEAN` | CHECK 付き `NUMBER(1)` または Oracle 23c BOOLEAN | |
| `DATE` | `DATE` | Snowflake の DATE は日付のみ、Oracle は時刻を含む |
| `TIME(n)` | `VARCHAR2(15)` または秒数として保存 | Oracle には TIME 型がない |
| `TIMESTAMP_NTZ(n)` | `TIMESTAMP(n)` | タイムゾーンなし |
| `TIMESTAMP_LTZ(n)` | `TIMESTAMP(n) WITH LOCAL TIME ZONE` | ローカル・タイムゾーン |
| `TIMESTAMP_TZ(n)` | `TIMESTAMP(n) WITH TIME ZONE` | 明示的なタイムゾーン・オフセット付き |
| `BINARY` / `VARBINARY` | `RAW(n)` または `BLOB` | |

### 半構造化型: VARIANT, OBJECT, ARRAY → Oracle JSON

これは Snowflake から移行する際の最も重要な型変換の課題である。

| Snowflake | Oracle | 備考 |
|---|---|---|
| `VARIANT` | `JSON` (21c 以降) または `CLOB IS JSON` | 汎用的な半構造化型 |
| `OBJECT` | ルートがオブジェクトの `JSON` | JSON オブジェクト |
| `ARRAY` | ルートが配列の `JSON` | JSON 配列 |

```sql
-- Snowflake: 半構造化されたイベント・データの保存
CREATE TABLE events (
    event_id    NUMBER,
    event_ts    TIMESTAMP_TZ,
    payload     VARIANT    -- 任意の JSON 構造を保持可能
);

-- Oracle 21c 以降の同等実装
CREATE TABLE events (
    event_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_ts    TIMESTAMP WITH TIME ZONE,
    payload     JSON
);

-- Oracle 12c ～ 20c での同等実装
CREATE TABLE events (
    event_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_ts    TIMESTAMP WITH TIME ZONE,
    payload     CLOB,
    CONSTRAINT chk_events_json CHECK (payload IS JSON)
);
```

---

## Snowflake SQL 方言 → Oracle SQL

### Snowflake VARIANT アクセス → Oracle JSON 関数

Snowflake では、コロン `:` とドット `.` を使用して VARIANT データを操作できる。

```sql
-- Snowflake: VARIANT 列内のネストされた JSON フィールドへのアクセス
SELECT
    event_id,
    payload:user_id::NUMBER AS user_id,
    payload:action::VARCHAR AS action,
    payload:metadata:region::VARCHAR AS region
FROM events;

-- Oracle: JSON_VALUE を使用した同等実装
SELECT
    event_id,
    JSON_VALUE(payload, '$.user_id')  AS user_id,
    JSON_VALUE(payload, '$.action')   AS action,
    JSON_VALUE(payload, '$.metadata.region') AS region
FROM events;

-- Oracle: 配列要素へのアクセス
-- Snowflake: payload:items[0]:sku::VARCHAR
JSON_VALUE(payload, '$.items[0].sku')
```

### Snowflake FLATTEN → Oracle JSON_TABLE

Snowflake の `FLATTEN` 関数は、配列を行に展開する。

```sql
-- Snowflake: 配列の展開
SELECT
    e.event_id,
    f.value:sku::VARCHAR AS sku,
    f.value:qty::NUMBER  AS qty
FROM events e,
     LATERAL FLATTEN(input => e.payload:items) f;

-- Oracle: JSON_TABLE を使用した同等実装
SELECT
    e.event_id,
    jt.sku,
    jt.qty
FROM events e,
     JSON_TABLE(e.payload, '$.items[*]'
         COLUMNS (
             sku VARCHAR2(100) PATH '$.sku',
             qty NUMBER        PATH '$.qty'
         )
     ) jt;
```

### PARSE_JSON → JSON リテラル

```sql
-- Snowflake: JSON 文字列のパース
SELECT PARSE_JSON('{"key": "value"}') AS obj;
INSERT INTO t (data) VALUES (PARSE_JSON('{"a": 1, "b": 2}'));

-- Oracle: JSON 値は単なる文字列として扱い、IS JSON で検証する
INSERT INTO t (data) VALUES ('{"a": 1, "b": 2}');
-- Oracle 21c 以降の JSON 型では、列に直接 JSON 文字列を渡すことができる
```

### Snowflake OBJECT_CONSTRUCT → JSON_OBJECT

```sql
-- Snowflake: 列から JSON オブジェクトを構築
SELECT OBJECT_CONSTRUCT('name', first_name, 'email', email) AS contact
FROM customers;

-- Oracle 12c 以降
SELECT JSON_OBJECT('name' VALUE first_name, 'email' VALUE email) AS contact
FROM customers;
```

### Snowflake ARRAY_AGG → JSON_ARRAYAGG

```sql
-- Snowflake: 行を配列として集計
SELECT customer_id, ARRAY_AGG(product_id) AS products
FROM order_items
GROUP BY customer_id;

-- Oracle 12c 以降
SELECT customer_id, JSON_ARRAYAGG(product_id) AS products
FROM order_items
GROUP BY customer_id;

-- Oracle: 伝統的な文字列集計
SELECT customer_id, LISTAGG(TO_CHAR(product_id), ',') WITHIN GROUP (ORDER BY product_id)
FROM order_items
GROUP BY customer_id;
```

### QUALIFY → ウィンドウ関数のサブクエリ

Snowflake は、ウィンドウ関数の結果をインラインでフィルタリングする `QUALIFY` をサポートしている。

```sql
-- Snowflake: 各顧客の最新の注文のみを保持
SELECT customer_id, order_id, order_date
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;

-- Oracle: サブクエリまたは CTE が必要
SELECT customer_id, order_id, order_date
FROM (
    SELECT customer_id, order_id, order_date,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
WHERE rn = 1;
```

### Snowflake 固有の関数

| Snowflake 関数 | Oracle 同等物 |
|---|---|
| `ZEROIFNULL(n)` | `NVL(n, 0)` |
| `NULLIFZERO(n)` | `NULLIF(n, 0)` |
| `IFF(cond, t, f)` | `CASE WHEN cond THEN t ELSE f END` または `DECODE` |
| `BOOLAND_AGG(col)` | `MIN(CASE WHEN col = TRUE THEN 1 ELSE 0 END) = 1` |
| `BOOLOR_AGG(col)` | `MAX(CASE WHEN col = TRUE THEN 1 ELSE 0 END) = 1` |
| `DIV0(a, b)` | `CASE WHEN b = 0 THEN 0 ELSE a/b END` |
| `SQUARE(n)` | `POWER(n, 2)` |
| `CBRT(n)` | `POWER(n, 1/3)` |
| `HAVERSINE(lat1, lon1, lat2, lon2)` | 球面三角法を使用したカスタム PL/SQL 関数 |
| `STRTOK(s, delim, n)` | `REGEXP_SUBSTR(s, '[^' \|\| delim \|\| ']+', 1, n)` |
| `EDITDISTANCE(s1, s2)` | `UTL_MATCH.EDIT_DISTANCE(s1, s2)` |
| `SOUNDEX(s)` | `SOUNDEX(s)` — 同様 |
| `JAROWINKLER_SIMILARITY(s1, s2)` | `UTL_MATCH.JARO_WINKLER_SIMILARITY(s1, s2)` |
| `HASH(expr)` | `ORA_HASH(expr)` |
| `UUID_STRING()` | `LOWER(REGEXP_REPLACE(RAWTOHEX(SYS_GUID()), '(.{8})(.{4})(.{4})(.{4})(.{12})', '\1-\2-\3-\4-\5'))` |
| `GENERATOR(rowcount => n)` | `SELECT LEVEL FROM DUAL CONNECT BY LEVEL <= n` |
| `SEQ4()` / `SEQ8()` | `ROWNUM` または `ROW_NUMBER() OVER (ORDER BY 1)` |
| `UNIFORM(min, max, RANDOM())` | `DBMS_RANDOM.VALUE(min, max)` |
| `NORMAL(mean, std, RANDOM())` | Box-Muller 変換を使用したカスタム PL/SQL |

### 文字列関数

| Snowflake | Oracle 同等物 |
|---|---|
| `CONCAT_WS(sep, a, b, c)` | `a \|\| sep \|\| b \|\| sep \|\| c` (手動結合) |
| `SPLIT(s, delim)` | 直接的な同等物なし。ループ内で `REGEXP_SUBSTR` を使用 |
| `SPLIT_PART(s, delim, n)` | `REGEXP_SUBSTR(s, '[^' \|\| delim \|\| ']+', 1, n)` |
| `CHARINDEX(sub, s [, start])` | `INSTR(s, sub [, start])` |
| `CONTAINS(s, sub)` | `INSTR(s, sub) > 0` |
| `STARTSWITH(s, prefix)` | `s LIKE prefix \|\| '%'` |
| `ENDSWITH(s, suffix)` | `s LIKE '%' \|\| suffix` |
| `LTRIM(s, chars)` | `LTRIM(s, chars)` — 同様 |
| `RTRIM(s, chars)` | `RTRIM(s, chars)` — 同様 |
| `BASE64_ENCODE(s)` | `UTL_ENCODE.BASE64_ENCODE(UTL_I18N.STRING_TO_RAW(s))` |
| `BASE64_DECODE_STRING(s)` | `UTL_I18N.RAW_TO_CHAR(UTL_ENCODE.BASE64_DECODE(s))` |
| `HEX_ENCODE(s)` | `RAWTOHEX(UTL_I18N.STRING_TO_RAW(s, 'AL32UTF8'))` |
| `TRY_CAST(val AS type)` | `CAST` と PL/SQL での例外処理 |
| `TRY_TO_NUMBER(s)` | `VALIDATE_CONVERSION(s AS NUMBER)` (Oracle 12.2 以降) |

---

## Snowflake タイム・トラベル → Oracle Flashback

Snowflake のタイム・トラベル（Time Travel）は、`AT(TIMESTAMP => ...)` や `BEFORE(STATEMENT => ...)` 構文を使用して、最大 90 日前の過去データを照会できる。Oracle の **Flashback** 機能は、これと同等の機能を提供する。

### 特定の時点でのタイム・トラベル

```sql
-- Snowflake: 指定したタイムスタンプ時点のテーブルを照会
SELECT * FROM orders AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP_TZ);

-- Snowflake: 特定のトランザクション前の状態を照会
SELECT * FROM orders BEFORE(STATEMENT => '8e5d0ca9-005e-44e6-b858-a8f5b37c5726');

-- Oracle Flashback Query: 過去の時点のテーブルを照会
SELECT * FROM orders AS OF TIMESTAMP
    TO_TIMESTAMP('2024-01-15 10:00:00', 'YYYY-MM-DD HH24:MI:SS');

-- Oracle Flashback Query: n 分前の状態を照会
SELECT * FROM orders AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '30' MINUTE);

-- Oracle Flashback Query: 特定の SCN (System Change Number) 時点を照会
SELECT * FROM orders AS OF SCN 5000000;
```

### Undo データ vs タイム・トラベル

```
Snowflake タイム・トラベル:        Oracle Flashback:
- 最大 90 日 (Enterprise)        - UNDO_RETENTION パラメータで制御
- 1 日 (Standard)                - デフォルト 15 分、数日間/数週間に調整可能
- クラウド・ストレージに保存       - UNDO 表領域 (または Flashback ログ) に保存
- アカウントごとの保持期間         - データベース/テーブルごとの保持
- AT/BEFORE で照会可能            - AS OF TIMESTAMP/SCN で照会可能
```

### Oracle Flashback 保持期間の設定

```sql
-- 現在の UNDO 保持期間を確認
SHOW PARAMETER UNDO_RETENTION;

-- 長時間の Flashback を可能にするため保持期間を延長 (秒単位)
ALTER SYSTEM SET UNDO_RETENTION = 86400;  -- 1 日

-- Flashback Database の有効化 (アーカイブログモードが必要)
ALTER DATABASE FLASHBACK ON;
ALTER SYSTEM SET DB_FLASHBACK_RETENTION_TARGET = 4320;  -- 3 日間 (分単位)

-- 表領域での UNDO 保持保証の設定
ALTER TABLESPACE undotbs1 RETENTION GUARANTEE;
```

### Flashback Table (過去の状態へのリストア)

```sql
-- Snowflake: タイム・トラベルからテーブルをクローンしてリストアポイントを作成
CREATE TABLE orders_backup CLONE orders AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP_TZ);

-- Oracle: Flashback Table (過去の状態への復元)
-- 事前に行移動の有効化が必要
ALTER TABLE orders ENABLE ROW MOVEMENT;
FLASHBACK TABLE orders TO TIMESTAMP
    TO_TIMESTAMP('2024-01-15 10:00:00', 'YYYY-MM-DD HH24:MI:SS');
```

---

## Snowflake COPY INTO → Oracle 外部表 / Data Pump

### Snowflake COPY INTO (ステージから)

```sql
-- Snowflake: 内部/外部ステージからのロード
COPY INTO orders
FROM @my_s3_stage/data/orders/
FILE_FORMAT = (TYPE = 'PARQUET')
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

### Oracle 外部表による同等実装

```sql
-- Oracle ディレクトリ・オブジェクトの作成
CREATE OR REPLACE DIRECTORY stage_dir AS '/opt/oracle/staging';

-- CSV ファイル用の外部表作成
CREATE TABLE ext_orders (
    order_id    NUMBER,
    customer_id NUMBER,
    order_date  DATE,
    total       NUMBER(15,2)
)
ORGANIZATION EXTERNAL (
    TYPE ORACLE_LOADER
    DEFAULT DIRECTORY stage_dir
    ACCESS PARAMETERS (
        RECORDS DELIMITED BY NEWLINE
        SKIP 1
        FIELDS TERMINATED BY ','
        OPTIONALLY ENCLOSED BY '"'
        MISSING FIELD VALUES ARE NULL
        (
            order_id,
            customer_id,
            order_date DATE "YYYY-MM-DD",
            total
        )
    )
    LOCATION ('orders_*.csv')
)
REJECT LIMIT UNLIMITED;

-- 物理テーブルへのデータロード
INSERT /*+ APPEND PARALLEL(t, 8) */ INTO orders t
SELECT * FROM ext_orders;
COMMIT;
```

---

## Snowflake ストリームおよびタスク → Oracle の変更データ・キャプチャ

Snowflake ストリーム (Streams) はテーブルの DML 変更をキャプチャし、タスク (Tasks) はそれらを処理するストアド手続きである。

```
Snowflake ストリーム:              Oracle での同等機能:
- テーブルの変更追跡              - マテリアライズド・ビュー・ログ (MV ログ)
- I/U/D をキャプチャ              - LogMiner / GoldenGate
- タスクやクエリで消費            - Oracle Scheduler による処理
```

### 増分処理のためのマテリアライズド・ビュー・ログ

```sql
-- ソース表に変更を追跡するマテリアライズド・ビュー・ログを作成
CREATE MATERIALIZED VIEW LOG ON orders
WITH ROWID, SEQUENCE
(order_id, status, total, updated_at)
INCLUDING NEW VALUES;

-- ログを消費する高速リフレッシュ・マテリアライズド・ビューを作成
CREATE MATERIALIZED VIEW mv_order_summary
REFRESH FAST ON DEMAND
AS
SELECT customer_id, COUNT(*) AS order_count, SUM(total) AS total_spent
FROM orders
GROUP BY customer_id;

-- 定期実行の設定 (Snowflake のタスクに相当)
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'REFRESH_ORDER_SUMMARY',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN DBMS_MVIEW.REFRESH(''MV_ORDER_SUMMARY'', ''F''); END;',
        repeat_interval => 'FREQ=MINUTELY;INTERVAL=5',
        enabled         => TRUE
    );
END;
/
```

---

## ベスト・プラクティス

1. **Snowflake のデータベースを Oracle の PDB またはスキーマに体系的にマッピングする。** 移行前に階層構造を文書化し、オブジェクトの配置先が混乱しないようにすること。

2. **VARCHAR のサイズ制約を再評価する。** Snowflake の VARCHAR は最大 16 MB である。Oracle の VARCHAR2 は 4,000 バイト（拡張時は 32,767）までである。実際のデータ長を調査し、長いテキストには CLOB を使用すること。

3. **VARIANT 列を明示的に再設計する。** すべての VARIANT 列に対して、完全な正規化、JSON としての保存、または Duality View の活用のいずれを選択するかを決定する必要がある。クエリ・パターンに基づいて早期に決定すること。

4. **Snowflake の Schema-on-Read から Schema-on-Write へ転換する。** Snowflake の VARIANT は任意のデータを保存し、後からスキーマを決められる。Oracle は事前定義されたスキーマが必要となる。移行を機に適切な型を定義すること。

5. **本番切り替え前に Flashback 構成をテストする。** UNDO_RETENTION と UNDO 表領域のサイズが、Snowflake の Time Travel 保証に匹敵する Flashback ウィンドウをサポートしているか確認すること。

6. **パラレル・クエリ構成のベンチマークを行う。** Snowflake は大規模クエリに対して自動的に計算リソースを拡張する。Oracle では PARALLEL 句や PGA/SGA の調整が必要である。代表的なクエリを実行し、適切なパラレル度を調整すること。

---

## よくある移行の落とし穴

**落とし穴 1 — TIMESTAMP_NTZ と Oracle TIMESTAMP：**
Snowflake の TIMESTAMP_NTZ はタイムゾーンを持たない。Oracle の TIMESTAMP も同様である。マッピング自体は正しいが、アプリケーションが TIMESTAMP_NTZ を暗黙的に UTC とみなしている場合、移行時にその前提を明示的に保持する必要がある。

**落とし穴 2 — WHERE 句での BOOLEAN：**
```sql
-- Snowflake
WHERE is_active = TRUE
-- Oracle (23c BOOLEAN または NUMBER ベース)
WHERE is_active = 1   -- NUMBER(1) マッピングの場合
WHERE is_active       -- Oracle 23c ネイティブ BOOLEAN の場合
```

**落とし穴 3 — Snowflake の寛容な型変換：**
Snowflake は算術演算において文字列を暗黙的に数値に変換する。Oracle も一部のコンテキストではこれを行うが、エラーになる箇所もある。すべての演算式をテストすること。

**落とし穴 4 — 大文字小文字の区別：**
Snowflake は、引用符で囲まれていない識別子に対しては大文字小文字を区別しない（Oracle と同様にデフォルトは大文字になる）。これは互換性の高い領域だが、Snowflake コードで引用符付き識別子が使用されている場合は注意が必要である。

**落とし穴 5 — VARIANT NULL vs SQL NULL：**
Snowflake では、VARIANT 列に SQL NULL（値の欠落）と JSON null リテラルの両方が含まれる場合がある。Oracle の JSON NULL も同様に SQL NULL とは区別される。JSON 抽出クエリでの NULL 処理を確認すること。

**落とし穴 6 — Snowflake Iceberg テーブル：**
Snowflake アカウントが S3/Azure 上の Apache Iceberg 表を使用している場合、Oracle の外部表 (ORC/Parquet アクセス・ドライバ) を使用してデータファイル (Parquet) を直接読み取ることができ、抽出フェーズで Snowflake をバイパスできる。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — JSON_VALUE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/JSON_VALUE.html)
- [Oracle Database 19c SQL Language Reference — JSON_TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/JSON_TABLE.html)
- [Oracle Database 19c JSON Developer's Guide — Overview of Oracle Database Support for JSON](https://docs.oracle.com/en/database/oracle/oracle-database/19/adjsn/json-in-oracle-database.html)
- [Oracle Database 19c Administrator's Guide — Using Oracle Flashback Technology](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/using-oracle-flashback-technology.html)
- [Oracle Database 19c PL/SQL Packages Reference — DBMS_SCHEDULER](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SCHEDULER.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
