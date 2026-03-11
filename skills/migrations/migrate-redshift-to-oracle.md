# Amazon Redshift から Oracle への移行

## 概要

Amazon Redshift は、PostgreSQL 8.0 をベースにした、列指向（カラムナ）の超並列処理（MPP）データウェアハウスである。これに対し、Oracle Database は伝統的な行指向の RDBMS であり、オプションのインメモリ・カラム・ストア (IMCS) を介してインメモリの列指向処理も可能である。Redshift から Oracle への移行は、単なる構文の変換にとどまらず、根本的なアーキテクチャの転換を伴う。分散キーに基づく並列処理から共有キャッシュ（または RAC）ベースのスケールアウトへ、そして列指向ストレージ・モデルから、オプションで列ストア・オーバーレイを備えた行ベースのヒープ構造への移行である。

本ガイドでは、アーキテクチャの違い、SQL 方言の変換、データ型マッピング、COPY コマンドに代わる手法、およびワークロード管理の同等機能について解説する。

---

## アーキテクチャの違い

### Redshift MPP vs Oracle アーキテクチャ

Redshift はシェアード・ナッシング MPP クラスターである。すべてのテーブルは、分散キー（Distribution Key）またはラウンドロビン/ALL 戦略に基づいて、各コンピュート・ノードに物理的に分散配置される。クエリはコンパイルされ、すべてのノードで並列実行され、各ノードがローカルのデータ・スライスを処理する。

Oracle には、Redshift の分散モデルに直接相当するものはない。Oracle の並列処理は以下の通りである：
- **パラレル・クエリ (PQ) :** 単一インスタンス内で、複数のパラレル実行サーバーが同一テーブルの異なる ROWID 範囲をスキャン・処理する。
- **Oracle RAC :** 複数のデータベース・インスタンスが同一のストレージを共有し、グローバル・キャッシュ・コーディネーションを行う。
- **Oracle インメモリ・カラム・ストア :** 分析クエリを高速化する、オプションの列形式メモリー領域。Redshift の列形式ストレージに最も近い機能。

| Redshift の概念 | Oracle の同等機能 | 備考 |
|---|---|---|
| 分散キー (Distribution key) | パーティション・キー（概ね） | 高カーディナリティの列によるパーティショニングで同様のローカル・アクセスを実現 |
| 分散スタイル ALL | 直接的な同等機能なし | Redshift の ALL は小表を全ノードに複製する。Oracle はバッファ・キャッシュでこれを処理 |
| 分散スタイル EVEN | 直接的な同等機能なし | Redshift のラウンドロビン。Oracle は全表スキャンを伴うパラレル・クエリに依存 |
| ソート・キー (Sort key) / 複合 | 索引またはパーティション | B-tree 索引、パーティショニング、IMCS ソート済みセグメント |
| ソート・キー / インターリーブ | 直接的な同等機能なし | インターリーブ・ソート・キーは Redshift 固有。Oracle 独自のビットマップ索引や IMCS を使用 |
| 列指向ストレージ | Oracle インメモリ・カラム・ストア | オプションの RAM ベース機能。ディスク上のフォーマットは変更しない |
| MPP ノード・レベル・データ | Oracle パラレル実行サーバー | 実行モデルは異なるが、クエリのファンアウト効果は同様 |

### スキーマ設計への影響

Redshift の列ストアから移行する場合：

1. **Oracle のデフォルトは行ベース・ストレージである。** 数百万行をスキャンして集計するクエリは、Oracle オプティマイザによる索引、パーティショニング、パラレル・クエリの活用に依存する。大規模なファクト表にはパーティショニングを追加すること。

2. **分析ワークロードには Oracle IMCS を有効にする。** 頻繁にスキャンされる分析用テーブルをインメモリ・カラム・ストアに指定する：

```sql
-- インメモリ・カラム・ストアの有効化 (SGA 構成が必要)
ALTER SYSTEM SET INMEMORY_SIZE = 10G SCOPE=SPFILE;
-- 再起動が必要

-- テーブルをインメモリ列指向ストレージに設定
ALTER TABLE fact_sales INMEMORY
    MEMCOMPRESS FOR QUERY HIGH
    PRIORITY HIGH;

-- 特定の列のみを指定
ALTER TABLE fact_sales INMEMORY
    MEMCOMPRESS FOR QUERY HIGH (sale_amount, sale_date, product_id);
```

3. **大規模なファクト表の年度・日付列にはレンジまたはレンジ・インターバル・パーティショニングを使用する。** これは Redshift の日付に対する複合ソート・キー（範囲スキャンの効率化目的）に相当する。

```sql
CREATE TABLE fact_sales (
    sale_id      NUMBER,
    sale_date    DATE NOT NULL,
    product_id   NUMBER,
    customer_id  NUMBER,
    sale_amount  NUMBER(15,2),
    region_code  VARCHAR2(10)
)
PARTITION BY RANGE (sale_date)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_initial VALUES LESS THAN (DATE '2020-01-01')
);
```

---

## データ型マッピング

| Redshift | Oracle | 備考 |
|---|---|---|
| `SMALLINT` | `NUMBER(5)` | |
| `INTEGER` / `INT` | `NUMBER(10)` | |
| `BIGINT` | `NUMBER(19)` | |
| `DECIMAL(p,s)` / `NUMERIC(p,s)` | `NUMBER(p,s)` | 直接的な同等物 |
| `REAL` | `BINARY_FLOAT` | 32 ビット IEEE 754 |
| `DOUBLE PRECISION` | `BINARY_DOUBLE` | 64 ビット IEEE 754 |
| `BOOLEAN` | CHECK (0,1) 付きの `NUMBER(1)` | Oracle 23c 以降はネイティブの BOOLEAN あり |
| `CHAR(n)` | `CHAR(n)` | |
| `VARCHAR(n)` / `CHARACTER VARYING(n)` | `VARCHAR2(n)` | Redshift 最大 65,535、Oracle 最大 4000/32767 |
| `TEXT` (エイリアス) | `CLOB` | |
| `DATE` | `DATE` | Oracle の DATE は時刻を含むが Redshift は日付のみ |
| `TIMESTAMP` | `TIMESTAMP` | |
| `TIMESTAMPTZ` / `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP WITH TIME ZONE` | |
| `TIMEZONEOID` | 該当なし | 内部的な型 |
| `GEOMETRY` | `SDO_GEOMETRY` | Oracle Spatial が必要 |
| `HLLSKETCH` | 同等の型なし | HyperLogLog スケッチ。`APPROX_COUNT_DISTINCT` を使用して再計算 |
| `SUPER` | `JSON` (21c 以降) または `CLOB IS JSON` | Redshift の半構造化型 |
| `VARBYTE` | `RAW(n)` または `BLOB` | 可変長バイナリ |

---

## Redshift SQL 独自の挙動 → Oracle の同等物

### NVL2 と ISNULL

```sql
-- Redshift (NVL2 は Oracle でも同様の構文で利用可能)
SELECT NVL2(phone, 'Has phone', 'No phone') FROM customers;

-- ISNULL は Redshift/SQL Server の構文であり、Oracle では使用不可
SELECT ISNULL(phone, 'N/A') FROM customers;      -- Redshift

-- Oracle 同等実装
SELECT NVL(phone, 'N/A') FROM customers;
SELECT COALESCE(phone, 'N/A') FROM customers;
```

### ILIKE (大文字小文字を区別しない LIKE)

```sql
-- Redshift
SELECT * FROM products WHERE name ILIKE '%widget%';

-- Oracle
SELECT * FROM products WHERE UPPER(name) LIKE UPPER('%widget%');
SELECT * FROM products WHERE UPPER(name) LIKE '%WIDGET%';
```

### LIMIT / OFFSET

```sql
-- Redshift
SELECT * FROM fact_sales ORDER BY sale_date DESC LIMIT 100 OFFSET 200;

-- Oracle 12c 以降
SELECT * FROM fact_sales ORDER BY sale_date DESC
OFFSET 200 ROWS FETCH NEXT 100 ROWS ONLY;
```

### DATEADD および DATEDIFF

```sql
-- Redshift
SELECT DATEADD(day, 30, order_date) AS due_date FROM orders;
SELECT DATEDIFF(day, start_date, end_date) AS days_elapsed FROM projects;

-- Oracle
SELECT order_date + 30 AS due_date FROM orders;
SELECT end_date - start_date AS days_elapsed FROM projects;
-- 月や年を指定した DATEADD の場合：
SELECT ADD_MONTHS(order_date, 1) FROM orders;
```

### GETDATE() およびその他の日時関数

```sql
-- Redshift
SELECT GETDATE();
SELECT SYSDATE;           -- Redshift でも利用可能
SELECT CURRENT_TIMESTAMP;

-- Oracle
SELECT SYSDATE FROM DUAL;          -- 小数秒なし
SELECT SYSTIMESTAMP FROM DUAL;     -- 小数秒とタイムゾーンオフセットあり
SELECT CURRENT_TIMESTAMP FROM DUAL; -- セッション・タイムゾーン
```

### CONVERT および CAST

```sql
-- Redshift
SELECT CONVERT(VARCHAR, sale_date, 112);   -- SQL Server スタイルの CONVERT
SELECT CAST(sale_amount AS VARCHAR(20));

-- Oracle
SELECT TO_CHAR(sale_date, 'YYYYMMDD') FROM DUAL;
SELECT CAST(sale_amount AS VARCHAR2(20)) FROM DUAL;
SELECT TO_CHAR(sale_amount) FROM DUAL;
```

### 文字列関数

| Redshift | Oracle 同等物 |
|---|---|
| `LISTAGG(col, ',')` | `LISTAGG(col, ',') WITHIN GROUP (ORDER BY col)` — 同様 |
| `NVL(a, b)` | `NVL(a, b)` — 同様 |
| `DECODE(expr, ...)` | `DECODE(expr, ...)` — 同様 |
| `SPLIT_PART(s, delim, n)` | `REGEXP_SUBSTR(s, '[^' \|\| delim \|\| ']+', 1, n)` |
| `STRTOL(s, base)` | 同等の関数なし。PL/SQL 関数を作成する必要あり |
| `TRANSLATE(s, from, to)` | `TRANSLATE(s, from, to)` — 同様 |
| `CHARINDEX(sub, s)` | `INSTR(s, sub)` |
| `LEN(s)` | `LENGTH(s)` |
| `BTRIM(s)` | `TRIM(s)` |

### ウィンドウ関数

Redshift は標準 SQL のウィンドウ関数の多くをサポートしている。Oracle もそれらすべてを、同一またはほぼ同一の構文でサポートしている。

```sql
-- Redshift および Oracle (同一構文)
SELECT
    sale_id,
    sale_date,
    sale_amount,
    SUM(sale_amount) OVER (PARTITION BY region_code ORDER BY sale_date
                           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
    RANK() OVER (PARTITION BY region_code ORDER BY sale_amount DESC) AS rank_in_region
FROM fact_sales;
```

---

## COPY コマンド → Oracle 外部表 / SQL*Loader

Redshift の `COPY` コマンドは、S3 等からテーブルに MPP 並列処理で直接データをロードする。Oracle では、主に 2 つの一括ロード・メカニズムを提供する。

### Redshift COPY

```sql
-- Redshift: S3 からのロード
COPY fact_sales
FROM 's3://my-bucket/data/fact_sales/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole'
FORMAT AS CSV
IGNOREHEADER 1
MAXERROR 100;
```

### Oracle 外部表 (読み取り専用、クエリベース)

外部表を使用すると、OS ファイルシステムのファイルをデータベース・テーブルであるかのように Oracle から読み取ることができる。ステージング・ロードに最適である。

```sql
-- データ・ファイルを指すディレクトリ・オブジェクトの作成
CREATE OR REPLACE DIRECTORY ext_data_dir AS '/opt/oracle/data/staging';
GRANT READ ON DIRECTORY ext_data_dir TO myuser;

-- 外部表の定義を作成
CREATE TABLE ext_fact_sales (
    sale_id      NUMBER,
    sale_date    DATE,
    product_id   NUMBER,
    customer_id  NUMBER,
    sale_amount  NUMBER(15,2),
    region_code  VARCHAR2(10)
)
ORGANIZATION EXTERNAL (
    TYPE ORACLE_LOADER
    DEFAULT DIRECTORY ext_data_dir
    ACCESS PARAMETERS (
        RECORDS DELIMITED BY NEWLINE
        SKIP 1
        FIELDS TERMINATED BY ','
        OPTIONALLY ENCLOSED BY '"'
        MISSING FIELD VALUES ARE NULL
        (
            sale_id,
            sale_date    DATE "YYYY-MM-DD",
            product_id,
            customer_id,
            sale_amount,
            region_code
        )
    )
    LOCATION ('fact_sales_*.csv')
)
REJECT LIMIT 100;

-- 外部表から物理テーブルへのロード
INSERT /*+ APPEND */ INTO fact_sales
SELECT * FROM ext_fact_sales;
COMMIT;
```

### Oracle SQL*Loader (ダイレクト・パス・ロード)

```
-- SQL*Loader コントロール・ファイル: fact_sales.ctl
OPTIONS (DIRECT=TRUE, PARALLEL=TRUE, ROWS=10000, ERRORS=100)
LOAD DATA
INFILE '/opt/oracle/data/staging/fact_sales_*.csv'
APPEND
INTO TABLE fact_sales
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
TRAILING NULLCOLS
(
    sale_id,
    sale_date    DATE "YYYY-MM-DD",
    product_id,
    customer_id,
    sale_amount,
    region_code
)
```

```bash
sqlldr userid=myuser/mypass@mydb control=fact_sales.ctl log=fact_sales.log
```

---

## ワークロード管理 (WLM) → Oracle リソース・マネージャ

Redshift の WLM は、クエリ・キュー全体にメモリーと同時実行スロットを割り当てる。Oracle の同等機能はデータベース・リソース・マネージャ (DBRM) である。

### Redshift WLM の概念

- **クエリ・キュー :** メモリー・パーセンテージと同時実行スロットが割り当てられた名前付きのキュー。
- **キューの割り当て :** ユーザー・グループまたはクエリ・グループのラベルに基づいて行われる。
- **短期間クエリの加速 (SQA) :** 実行時間が短いクエリの自動的な優先度付け。

### Oracle リソース・マネージャ 同等実装

```sql
-- リソース・マネージャ・プランの作成例
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();

    -- コンシューマ・グループの作成
    DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
        consumer_group => 'ANALYTICS_HIGH',
        comment        => 'High-priority analytics queries'
    );
    DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
        consumer_group => 'ANALYTICS_LOW',
        comment        => 'Low-priority batch analytics'
    );

    -- リソース・プランの作成
    DBMS_RESOURCE_MANAGER.CREATE_PLAN(
        plan    => 'ANALYTICS_PLAN',
        comment => 'Resource plan for analytics workload'
    );

    -- プラン・ディレクティブの設定 (CPU 割り当て)
    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan             => 'ANALYTICS_PLAN',
        group_or_subplan => 'ANALYTICS_HIGH',
        comment          => 'High priority — 60% CPU',
        cpu_p1           => 60
    );
    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan             => 'ANALYTICS_PLAN',
        group_or_subplan => 'ANALYTICS_LOW',
        comment          => 'Low priority — 20% CPU',
        cpu_p1           => 20
    );

    DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA();
    DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/

-- プランの有効化
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'ANALYTICS_PLAN';

-- ユーザーのコンシューマ・グループへの割り当て
EXEC DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP('analyst_user', 'ANALYTICS_HIGH', FALSE);
EXEC DBMS_RESOURCE_MANAGER.SET_INITIAL_CONSUMER_GROUP('analyst_user', 'ANALYTICS_HIGH');
```

---

## 列指向から行指向への移行に関する考慮事項

Redshift の列指向ストレージから Oracle の行指向ストレージに分析ワークロードを移行する場合：

### 見直すべきクエリ・パターン

1. **少数の列のみを選択する幅の広いテーブルのスキャン :** Redshift は必要な列のみを読み取るため、このケースで優れている。これに対し、Oracle の全表スキャンは行ブロック全体を読み取る。これを軽減するには：
   - Oracle インメモリ・カラム・ストアの活用 (特定のテーブル/列)
   - フィルタ条件となる列への適切な索引設定
   - パーティション・プルーニングの活用

2. **集計処理の多いクエリ :** Redshift のベクトル化された列指向集計は非常に高速である。Oracle も、IMCS を活用したパラレル・クエリ・エンジンでインメモリ・データに対しては同等の速度を実現できるが、ディスク・ベースの集計は一般に低速になる。

3. **高カーディナリティの結合 :** SQL レベルの結合処理は同様だが、実行計画は異なる。Oracle の実行計画を確認し、ハッシュ結合とネストしたループが適切に選択されているかを確認すること。

### パフォーマンス・チューニング・チェックリスト

```sql
-- IMCS が使用されているかの確認
SELECT segment_name, bytes, inmemory_size, bytes_not_populated
FROM v$im_segments
WHERE segment_name = 'FACT_SALES';

-- テーブルのパラレル度を確認
SELECT degree FROM user_tables WHERE table_name = 'FACT_SALES';

-- テーブルのパラレル度を設定
ALTER TABLE fact_sales PARALLEL (DEGREE 8);

-- ロード後に最新の統計情報を収集
EXEC DBMS_STATS.GATHER_TABLE_STATS('MYSCHEMA', 'FACT_SALES', CASCADE => TRUE);

-- 実行計画でパーティション・プルーニングを確認
EXPLAIN PLAN FOR
SELECT SUM(sale_amount) FROM fact_sales WHERE sale_date >= DATE '2024-01-01';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

---

## ベスト・プラクティス

1. **移行前に Redshift クエリをプロファイリングする。** STL_QUERY や STL_SCAN ログを書き出し、どのテーブルがどのような検索条件で最も頻繁にスキャンされているかを把握すること。これが Oracle のパーティショニングおよび IMCS 戦略のベースとなる。

2. **索引を過剰に作成しない。** Redshift はゾーンマップ（ブロック・レベルの最小/最大統計）を自動的に使用する。Oracle は B-tree 索引を使用するが、これにはメンテナンスのオーバーヘッドがある。索引は、クエリが実際に恩恵を受ける場合にのみ追加すること。

3. **大規模なテーブルには圧縮を有効にする。** アドバンスト圧縮や IMCS 圧縮を使用して、ストレージ消費と I/O を削減する：
   ```sql
   ALTER TABLE fact_sales COMPRESS FOR OLTP;  -- または読み取り専用なら COMPRESS BASIC
   ```

4. **一括ロードには Oracle のパラレル DML を使用する。**
   ```sql
   ALTER SESSION ENABLE PARALLEL DML;
   INSERT /*+ APPEND PARALLEL(t, 8) */ INTO fact_sales t SELECT ... FROM ext_fact_sales;
   ```

5. **PGA および SGA のサイズ設定を監視する。** Redshift はスライスごとにメモリーを自動管理する。Oracle では、DBA が SGA（バッファ・キャッシュ、共有プール、IMCS）および PGA（ソート領域、ハッシュ結合領域）を調整する必要がある。まずは自動メモリー管理 (AMM/ASMM) から開始すること。

---

## よくある移行の落とし穴

**落とし穴 1 — Redshift SQL が標準 SQL であるという誤認：** Redshift には、PostgreSQL 由来の拡張機能や AWS 固有の関数が多く含まれている。それらすべてが Oracle で有効な SQL とは限らず、変換が必要になる。

**落とし穴 2 — アプリケーション・コードにおける分散キーへの依存：** 分散キーを活用したルーティングを利用するアプリ・コード（高度な ETL 等）は Oracle に同等機能がないため、再設計が必要になる。

**落とし穴 3 — SUPER 型からの移行：** Redshift の SUPER 列は JSON のような半構造化データを保持する。これらは Oracle の JSON 列 (21c 以降) または IS JSON 制約付きの CLOB としてマップすること。SUPER 型に対する PartiQL クエリは、Oracle の JSON_VALUE, JSON_TABLE 等を使用して書き換える必要がある。

**落とし穴 4 — Redshift UNLOAD → Oracle 外部表：** Redshift の UNLOAD は S3 にエクスポートする。Oracle 外部表はローカル・ファイルシステムまたは NFS を読み取る。ロード前に、S3 のファイルを Oracle ホストからアクセス可能な場所にダウンロードしておく必要がある。

**落とし穴 5 — Redshift の遅延バインディング・ビュー：** Redshift は作成時に参照先テーブルが存在しなくてもよい遅延バインディング・ビューをサポートしている。Oracle はこれをサポートしておらず、ビューのコンパイル時にすべての参照オブジェクトが存在し、アクセス可能である必要がある。

**落とし穴 6 — ZEROIFNULL および EMPTYSTRING の同等表現：**
```sql
-- Redshift
SELECT ZEROIFNULL(revenue) FROM sales;

-- Oracle
SELECT NVL(revenue, 0) FROM sales;
SELECT COALESCE(revenue, 0) FROM sales;
```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — Data Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Data-Types.html)
- [Oracle Database 19c SQL Language Reference — CREATE TABLE (Partitioning)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c VLDB and Partitioning Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/vldbg/)
- [Oracle Database 19c In-Memory Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/inmem/)
- [Oracle Database 19c PL/SQL Packages Reference — DBMS_RESOURCE_MANAGER](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESOURCE_MANAGER.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [Oracle Database 19c Administrator's Guide — External Tables](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tables.html)
