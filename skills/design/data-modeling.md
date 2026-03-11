# Oracle Databaseのデータ・モデリング

## 概要

データ・モデリングとは、データベース内でデータがどのように整理、格納、および関連付けられるかを定義するプロセスである。これは、**概念**（ビジネス概念）、**論理**（プラットフォームに依存しない関係構造）、および**物理**（ストレージ・パラメータを含む Oracle 固有の DDL）という 3 つの抽象化レベルで実行される。このガイドでは、論理および物理モデリングの手法、データウェアハウスのスキーマ・パターン、運用データ・ストア (ODS) の設計、およびストレージ句、圧縮、パーティショニングを含む Oracle 固有の物理モデルの考慮事項について説明する。

---

## 1. モデリング・レベル

### 概念モデル (Conceptual Model)

概念モデルは、技術的な詳細を一切含まず、ビジネス・エンティティ（実体）とその高レベルな関係を把握する。これはツールに依存せず、ビジネス関係者とのコミュニケーションを目的としている。

- エンティティ: 顧客 (Customer)、注文 (Order)、製品 (Product)
- 関係: 顧客が注文を行う、注文は製品を含む
- データ型、キー、制約は含まない

### 論理モデル (Logical Model)

論理モデルはプラットフォームには依存しないが、技術的には完全である。以下の内容を定義する：

- 列名とデータ型（汎用的なもの。例：STRING, INTEGER, DECIMAL）を持つテーブルとしてのすべての実体
- 主キー (PK)、外部キー (FK)、および一意キー (候補キー)
- 適用された正規化（通常、OLTP では第3正規形 (3NF) まで、OLAP では非正規化）
- 制約とビジネス・ルール（参照整合性、NOT NULL）

### 物理モデル (Physical Model)

物理モデルは、Oracle 固有の DDL である。以下の内容を追加する：

- Oracle のデータ型 (`VARCHAR2`, `NUMBER`, `DATE`, `TIMESTAMP`, `CLOB`, `BLOB`)
- ストレージ句 (`TABLESPACE`, `STORAGE`, `PCTFREE`, `PCTUSED`)
- パーティショニング戦略
- 索引（インデックス）の定義
- 圧縮設定
- パラレル・クエリ・ヒント

---

## 2. OLTP 論理モデリング

オンライン・トランザクション処理 (OLTP) システムは、**書込みパフォーマンス**、**データ整合性**、および**並行性**を優先する。OLTP の論理モデルは正規化ルールに厳密に従う（最低でも 3NF）。

### 主な特徴

- 大量の挿入 (Insert) / 更新 (Update) / 削除 (Delete)
- 特定のデータを対象とした小規模なクエリ（単一行または狭い範囲の検索）
- 高い並行性を伴う多数の短いトランザクション
- 行レベル・ロックが極めて重要
- 冗長性を最小限に抑え、ロック競合を減らすために正規化されている

### OLTP スキーマのサンプル

```sql
-- eコマース・システム用の正規化された OLTP スキーマ

CREATE TABLE CUSTOMERS (
    customer_id   NUMBER         GENERATED ALWAYS AS IDENTITY,
    email         VARCHAR2(255)  NOT NULL,
    first_name    VARCHAR2(100)  NOT NULL,
    last_name     VARCHAR2(100)  NOT NULL,
    created_at    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT pk_customers      PRIMARY KEY (customer_id),
    CONSTRAINT uq_customers_email UNIQUE (email)
)
TABLESPACE users_data
PCTFREE 10;

CREATE TABLE ORDERS (
    order_id      NUMBER         GENERATED ALWAYS AS IDENTITY,
    customer_id   NUMBER         NOT NULL,
    order_date    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    status        VARCHAR2(20)   DEFAULT 'PENDING' NOT NULL,
    total_amount  NUMBER(14,2),
    CONSTRAINT pk_orders         PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_cust    FOREIGN KEY (customer_id) REFERENCES CUSTOMERS (customer_id),
    CONSTRAINT ck_order_status   CHECK (status IN ('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED'))
)
TABLESPACE users_data
PCTFREE 15;  -- 更新による行の拡張が予想される場合は PCTFREE を高めに設定

CREATE INDEX IX_ORDERS_CUSTOMER_ID ON ORDERS (customer_id) TABLESPACE users_idx;
CREATE INDEX IX_ORDERS_ORDER_DATE  ON ORDERS (order_date)  TABLESPACE users_idx;
```

---

## 3. データウェアハウスのディメンショナル・モデリング

データウェアハウス (DW) は、大量のデータに対する分析クエリの**読取りパフォーマンス**を優先する。**ディメンショナル・モデリング** (Ralph Kimball 手法) は、データを**ファクト表**と**ディメンション表**に構造化する。

### ファクト表 (Fact Tables)

ファクト表は、測定可能なビジネス・イベント（売上、取引、イベント）を格納する。以下が含まれる：

- ディメンション表への外部キー
- 数値メジャー（数量、金額、時間）
- 縮退ディメンション (Degenerate dimensions: 独立したディメンションではなく、ファクト内に直接格納される注文番号など)
- 通常、非常に巨大（数億から数十億行）

### ディメンション表 (Dimension Tables)

ディメンション表は、ファクトに対する記述的なコンテキスト（背景情報）を格納する。以下が含まれる：

- サロゲート主キー（ナチュラル・キーやビジネス・キーではない代替キー）
- 記述的な属性（名前、カテゴリ、階層）
- 通常、ファクト表よりもはるかに小さい
- 更新頻度は低い (徐々に変化するディメンション: SCD)

---

## 4. スター・スキーマ (Star Schema)

**スター・スキーマ**は、最もシンプルでクエリ効率の高いディメンショナル・モデルである。中心にファクト表があり、ディメンション表が星のように外側に放射状に配置される。ディメンション表間には正規化は行わず、すべての記述的な属性は単一の「幅の広い」ディメンション表に集約される。

```
                 DIM_DATE
                    |
DIM_CUSTOMER -- FACT_SALES -- DIM_PRODUCT
                    |
               DIM_STORE
```

### スター・スキーマの DDL 例

```sql
-- ディメンション: 日付 (Date)
CREATE TABLE DIM_DATE (
    date_key        NUMBER(8)    NOT NULL,  -- YYYYMMDD のサロゲート・キー
    full_date       DATE         NOT NULL,
    day_of_week     VARCHAR2(10) NOT NULL,
    day_of_month    NUMBER(2)    NOT NULL,
    month_number    NUMBER(2)    NOT NULL,
    month_name      VARCHAR2(10) NOT NULL,
    quarter_number  NUMBER(1)    NOT NULL,
    year_number     NUMBER(4)    NOT NULL,
    is_weekend      CHAR(1)      DEFAULT 'N' NOT NULL,
    is_holiday      CHAR(1)      DEFAULT 'N' NOT NULL,
    CONSTRAINT pk_dim_date PRIMARY KEY (date_key)
)
TABLESPACE dw_data
COMPRESS FOR QUERY HIGH;  -- ハイブリッド列圧縮 (HCC) — Exadata または Oracle ZFS/ODA ストレージが必要

-- ディメンション: 顧客 (Customer - 都市/州/国が非正規化され集約されている)
CREATE TABLE DIM_CUSTOMER (
    customer_key    NUMBER       GENERATED ALWAYS AS IDENTITY,
    customer_bk     VARCHAR2(50) NOT NULL,  -- ビジネス/ナチュラル・キー
    full_name       VARCHAR2(200) NOT NULL,
    email           VARCHAR2(255),
    city            VARCHAR2(100),
    state_province  VARCHAR2(100),
    country_code    CHAR(2),
    customer_segment VARCHAR2(50),
    effective_from  DATE         NOT NULL,
    effective_to    DATE,
    is_current      CHAR(1)      DEFAULT 'Y' NOT NULL,
    CONSTRAINT pk_dim_customer PRIMARY KEY (customer_key)
)
TABLESPACE dw_data
COMPRESS FOR QUERY HIGH;

-- ディメンション: 製品 (Product - カテゴリ階層が非正規化され集約されている)
CREATE TABLE DIM_PRODUCT (
    product_key       NUMBER        GENERATED ALWAYS AS IDENTITY,
    product_bk        VARCHAR2(50)  NOT NULL,
    product_name      VARCHAR2(200) NOT NULL,
    product_desc      VARCHAR2(1000),
    subcategory_name  VARCHAR2(100),
    category_name     VARCHAR2(100),
    brand_name        VARCHAR2(100),
    unit_cost         NUMBER(12,2),
    unit_price        NUMBER(12,2),
    is_active         CHAR(1)       DEFAULT 'Y' NOT NULL,
    CONSTRAINT pk_dim_product PRIMARY KEY (product_key)
)
TABLESPACE dw_data
COMPRESS FOR QUERY HIGH;

-- ファクト: 売上 (Sales - 中心となるファクト表)
CREATE TABLE FACT_SALES (
    sales_id          NUMBER        GENERATED ALWAYS AS IDENTITY,
    date_key          NUMBER(8)     NOT NULL,
    customer_key      NUMBER        NOT NULL,
    product_key       NUMBER        NOT NULL,
    store_key         NUMBER        NOT NULL,
    order_number      VARCHAR2(50),           -- 縮退ディメンション
    quantity_sold     NUMBER(10)    NOT NULL,
    unit_price        NUMBER(12,2)  NOT NULL,
    unit_cost         NUMBER(12,2)  NOT NULL,
    gross_revenue     NUMBER(14,2)  NOT NULL,
    gross_profit      NUMBER(14,2)  NOT NULL,
    discount_amount   NUMBER(12,2)  DEFAULT 0 NOT NULL,
    CONSTRAINT pk_fact_sales      PRIMARY KEY (sales_id),
    CONSTRAINT fk_fs_date         FOREIGN KEY (date_key)     REFERENCES DIM_DATE     (date_key),
    CONSTRAINT fk_fs_customer     FOREIGN KEY (customer_key) REFERENCES DIM_CUSTOMER (customer_key),
    CONSTRAINT fk_fs_product      FOREIGN KEY (product_key)  REFERENCES DIM_PRODUCT  (product_key),
    CONSTRAINT fk_fs_store        FOREIGN KEY (store_key)    REFERENCES DIM_STORE    (store_key)
)
TABLESPACE dw_data
COMPRESS FOR QUERY HIGH
PARTITION BY RANGE (date_key) (
    PARTITION p_2023 VALUES LESS THAN (20240101),
    PARTITION p_2024 VALUES LESS THAN (20250101),
    PARTITION p_2025 VALUES LESS THAN (20260101),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)
);

-- ビットマップ索引 — DW におけるカーディナリティの低い外部キー列に非常に有効
CREATE BITMAP INDEX BIX_FS_DATE     ON FACT_SALES (date_key)     LOCAL TABLESPACE dw_idx;
CREATE BITMAP INDEX BIX_FS_CUSTOMER ON FACT_SALES (customer_key) LOCAL TABLESPACE dw_idx;
CREATE BITMAP INDEX BIX_FS_PRODUCT  ON FACT_SALES (product_key)  LOCAL TABLESPACE dw_idx;
```

**注意:** ビットマップ索引は、データウェアハウスの外部キー列（低カーディナリティ、検索重視、更新頻度が低い）に最適である。書き込みが頻発する OLTP テーブルにはビットマップ索引を使用してはいけない。深刻なロック競合が発生する。

> **重要:** `COMPRESS FOR QUERY HIGH` は Oracle ハイブリッド列圧縮 (HCC) を使用する。HCC は一般的なアドバンスト圧縮機能ではなく、Exadata、Oracle ZFS Storage Appliance、Oracle Database Appliance (ODA)、またはその他の HCC 互換エンジニアド・システムが必要である。標準的なサーバー・ストレージでは、この句を指定しても列形式の圧縮は実現されない。非 Exadata 環境では、`ROW STORE COMPRESS ADVANCED` (アドバンスト行圧縮: Advanced Compression オプションが必要) または `COMPRESS BASIC` (ダイレクト・パス・インサートのみ: すべてのエディションで使用可能) を使用する。

---

## 5. スノーフレーク・スキーマ (Snowflake Schema)

**スノーフレーク・スキーマ**はディメンション表を正規化し、階層構造を別のテーブルに分割する。これによりディメンション・データのストレージ容量は削減されるが、結合 (Join) の数が増える。

```
DIM_PRODUCT_CATEGORY
        |
  DIM_PRODUCT -- FACT_SALES -- DIM_CUSTOMER -- DIM_GEOGRAPHY
                    |
               DIM_DATE
```

### スノーフレーク・スキーマの DDL 例

```sql
-- 正規化された製品ディメンション階層
CREATE TABLE DIM_PRODUCT_CATEGORY (
    category_key   NUMBER       GENERATED ALWAYS AS IDENTITY,
    category_name  VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_dim_prod_cat PRIMARY KEY (category_key)
);

CREATE TABLE DIM_PRODUCT_SUBCATEGORY (
    subcategory_key   NUMBER       GENERATED ALWAYS AS IDENTITY,
    subcategory_name  VARCHAR2(100) NOT NULL,
    category_key      NUMBER       NOT NULL,
    CONSTRAINT pk_dim_prod_subcat    PRIMARY KEY (subcategory_key),
    CONSTRAINT fk_subcat_category    FOREIGN KEY (category_key)
                                     REFERENCES DIM_PRODUCT_CATEGORY (category_key)
);

CREATE TABLE DIM_PRODUCT (
    product_key       NUMBER        GENERATED ALWAYS AS IDENTITY,
    product_bk        VARCHAR2(50)  NOT NULL,
    product_name      VARCHAR2(200) NOT NULL,
    subcategory_key   NUMBER        NOT NULL,
    unit_price        NUMBER(12,2),
    CONSTRAINT pk_dim_product     PRIMARY KEY (product_key),
    CONSTRAINT fk_prod_subcategory FOREIGN KEY (subcategory_key)
                                   REFERENCES DIM_PRODUCT_SUBCATEGORY (subcategory_key)
);
```

### スター vs スノーフレーク: 使い分けの目安

| 基準 | スター・スキーマ | スノーフレーク・スキーマ |
|---|---|---|
| クエリ・パフォーマンス | 優れている (結合が少ない) | 低下する (結合が多い) |
| ストレージ容量 | 多い (非正規化されたディメンション) | 少ない (正規化されたディメンション) |
| ETL の複雑さ | 単純 | 複雑 |
| メンテナンス | 困難 (多数の行を更新) | 容易 (ディメンション表のみ更新) |
| BI ツールとの互換性 | 極めて良好 (多くがスター推奨) | 許容範囲 |
| 最適なケース | ほとんどの DW ユースケース | 深い階層を持つ非常に巨大なディメンション |

---

## 6. Operational Data Store (ODS)

**運用データ・ストア (ODS)** は、OLTP ソース・システムとデータウェアハウスの橋渡しをする。以下を提供する：

- 準リアルタイムの統合された運用レポート
- DW ロード前のステージング / クレンジング・レイヤー
- 複数のソース・システムにわたる単一の統合ビュー

### ODS の設計原則

- **サブジェクト指向 (Subject-oriented)**: ソース・システムではなく、ビジネス上の主題（顧客、注文など）に基づいて編成される
- **統合されている (Integrated)**: 複数のソースからのデータが共通モデルに統合される
- **最新である (Current)**: 準リアルタイムの運用状態を反映する (履歴を重視する DW とは異なる)
- **更新可能 (Volatile)**: ODS データはその場で更新される (追記型の DW とは異なる)

```sql
-- ソース・システムの追跡と監査列を持つ ODS 表
CREATE TABLE ODS_CUSTOMER (
    ods_customer_id     NUMBER        GENERATED ALWAYS AS IDENTITY,
    -- ソース・システムの追跡
    source_system       VARCHAR2(50)  NOT NULL,  -- 'CRM', 'ERP', 'WEB'
    source_system_id    VARCHAR2(100) NOT NULL,
    -- ビジネス・キー (複数のソース・システム間で統一)
    customer_email      VARCHAR2(255) NOT NULL,
    -- 統合属性
    full_name           VARCHAR2(200),
    phone_number        VARCHAR2(30),
    address_line1       VARCHAR2(255),
    city                VARCHAR2(100),
    country_code        CHAR(2),
    customer_status     VARCHAR2(20),
    -- ODS 監査列
    source_created_at   TIMESTAMP,
    source_updated_at   TIMESTAMP,
    ods_loaded_at       TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
    ods_updated_at      TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
    ods_checksum        VARCHAR2(64),  -- 変更検出のための主要フィールドのハッシュ (MD5/SHA)
    CONSTRAINT pk_ods_customer       PRIMARY KEY (ods_customer_id),
    CONSTRAINT uq_ods_cust_src       UNIQUE (source_system, source_system_id)
)
TABLESPACE ods_data;

-- 一般的な ODS 検索パターンのための索引
CREATE INDEX IX_ODS_CUST_EMAIL  ON ODS_CUSTOMER (customer_email)  TABLESPACE ods_idx;
CREATE INDEX IX_ODS_CUST_LOADED ON ODS_CUSTOMER (ods_loaded_at)   TABLESPACE ods_idx;
```

---

## 7. 徐々に変化するディメンション (SCD)

SCD (Slowly Changing Dimensions) は、時間の経過に伴うディメンション・データの変更を管理する。Oracle は 3 つの一般的なタイプをすべてサポートしている。

### SCD タイプ 1 — 上書き (Overwrite)

履歴を保持しない。最も単純な手法。履歴が重要でない場合に使用される。

```sql
UPDATE DIM_CUSTOMER
SET    email       = :new_email,
       ods_updated_at = SYSTIMESTAMP
WHERE  customer_bk = :customer_bk
AND    is_current  = 'Y';
```

### SCD タイプ 2 — 新しい行の追加 (履歴全体を保持)

変更ごとに新しい行が挿入される。前の行は有効期限が設定されて閉じられる。

```sql
-- 現在のレコードを閉じる
UPDATE DIM_CUSTOMER
SET    effective_to = TRUNC(SYSDATE) - INTERVAL '1' SECOND,
       is_current   = 'N'
WHERE  customer_bk  = :customer_bk
AND    is_current   = 'Y';

-- 新しい現在のレコードを挿入
INSERT INTO DIM_CUSTOMER (
    customer_bk, full_name, email, city, state_province, country_code,
    customer_segment, effective_from, effective_to, is_current
) VALUES (
    :customer_bk, :full_name, :email, :city, :state_province, :country_code,
    :customer_segment, TRUNC(SYSDATE), NULL, 'Y'
);
```

### SCD タイプ 3 — 以前の値の列

現在の値と 1 つ前の値を格納する。履歴は限定的だがクエリは単純になる。

```sql
ALTER TABLE DIM_CUSTOMER ADD (
    previous_email     VARCHAR2(255),
    email_changed_date DATE
);

UPDATE DIM_CUSTOMER
SET    previous_email     = email,
       email_changed_date = SYSDATE,
       email              = :new_email
WHERE  customer_bk = :customer_bk;
```

---

## 8. Oracle の物理モデルに関する考慮事項

### Oracle のデータ型

ストレージ効率と正確性のために、データ型を慎重に選択すること：

```sql
CREATE TABLE PHYSICAL_MODEL_EXAMPLE (
    -- 数値型
    id              NUMBER(10)          NOT NULL,  -- 最大 10 桁の整数
    amount          NUMBER(18,4)        NOT NULL,  -- 会計用: 精度 + 位取り
    percentage      NUMBER(5,2)         NOT NULL,  -- 999.99
    
    -- 文字型
    short_code      CHAR(3)             NOT NULL,  -- 固定長: ISO コードなど
    description     VARCHAR2(4000)      NOT NULL,  -- 可変長。最大 4000 バイト
    large_text      CLOB,                          -- 4000 文字超
    json_data       CLOB CHECK (json_data IS JSON),-- JSON 検証 (12c+)

    -- 日付/時刻型
    event_date      DATE                NOT NULL,  -- 日付 + 時刻 (タイムゾーンなし)
    created_at      TIMESTAMP(6)        NOT NULL,  -- マイクロ秒の精度
    updated_at      TIMESTAMP WITH TIME ZONE,      -- グローバルアプリ用: 常に TZ を保存
    duration_days   INTERVAL DAY(3) TO SECOND(0),  -- 経過時間

    -- バイナリ型
    file_content    BLOB,                          -- バイナリ・ファイル
    thumbnail       RAW(2000),                     -- 小規模なバイナリ (2000 バイト以下)

    CONSTRAINT pk_pme PRIMARY KEY (id)
);
```

### ストレージ句

Oracle のストレージ・パラメータは、セグメント内での物理スペースの割り当てを制御する：

```sql
CREATE TABLE ORDERS (
    order_id     NUMBER        NOT NULL,
    order_data   VARCHAR2(500),
    CONSTRAINT pk_orders PRIMARY KEY (order_id)
)
TABLESPACE users_data
PCTFREE   15     -- 行の更新（拡張）用に各ブロックの 15% を予約
PCTUSED   40     -- 使用率が 40% を下回った場合に挿入対象となる (MSSMのみ)
INITRANS  4      -- ブロックあたりの初期トランザクション・スロット数 (並行 DML が多い場合に高くする)
MAXTRANS  255    -- ブロックあたりの最大並列トランザクション数
STORAGE (
    INITIAL     64K    -- 初期エクステント・サイズ
    NEXT        64K    -- 次のエクステント・サイズ (ローカル管理では無視される)
    MINEXTENTS  1      -- 最小エクステント数
    MAXEXTENTS  UNLIMITED
    PCTINCREASE 0      -- 増分率 (ローカル管理では無視される)
);
```

**PCTFREE チューニング・ガイド:**

| ワークロード | 推奨 PCTFREE | 理由 |
|---|---|---|
| 挿入のみ (追加) | 0–5 | 更新されないため。ブロック密度を最大化する |
| 挿入 + 更新の混在 | 10–20 | 行の拡張に備えてスペースを予約する |
| 大幅な更新 (行の拡張) | 25–40 | 行連鎖/行移行を防ぐ |
| データウェアハウス (検索のみ) | 0–5 | スキャン密度を最大化する |

### Oracle の圧縮

Oracle は複数の圧縮ティアを提供している：

```sql
-- 基本圧縮 (Basic Compression: 全エディション) — ダイレクト・パス・インサート時のみ圧縮される
CREATE TABLE SALES_ARCHIVE (
    sale_id     NUMBER,
    sale_date   DATE,
    amount      NUMBER(14,2)
)
COMPRESS BASIC
TABLESPACE dw_data;

-- アドバンスト行圧縮 (Advanced Row Compression: Enterprise) — すべての DML 操作で圧縮される
CREATE TABLE ORDERS (
    order_id    NUMBER,
    customer_id NUMBER,
    order_date  DATE
)
ROW STORE COMPRESS ADVANCED
TABLESPACE users_data;

-- ハイブリッド列圧縮 (HCC) — Exadata, ZFS Storage Appliance, または ODA が必要
-- 標準的なサーバー/SAN ストレージでは利用不可。警告なしで非圧縮にフォールバックされる
CREATE TABLE FACT_SALES (
    date_key    NUMBER(8),
    amount      NUMBER(14,2)
)
COMPRESS FOR QUERY HIGH
TABLESPACE dw_data;

-- インメモリー圧縮 (12c 12.1.0.2 以降) — インメモリー列形式ストア
ALTER TABLE FACT_SALES
    INMEMORY MEMCOMPRESS FOR QUERY HIGH PRIORITY CRITICAL;
```

### パラレル・クエリの構成

データウェアハウスのテーブルに対して、パラレル DML およびクエリを構成する：

```sql
-- テーブルでのパラレル・クエリを有効化 (DW ファクト表の例)
ALTER TABLE FACT_SALES PARALLEL 8;

-- セッション・レベルでパラレル DML を有効化
ALTER SESSION ENABLE PARALLEL DML;

-- ヒントを使用したパラレル・クエリ
SELECT /*+ PARALLEL(f, 8) PARALLEL(d, 4) */
       d.year_number,
       SUM(f.gross_revenue) AS total_revenue
FROM   FACT_SALES    f
JOIN   DIM_DATE      d ON f.date_key = d.date_key
GROUP  BY d.year_number
ORDER  BY d.year_number;
```

---

## 9. ベスト・プラクティス

- **OLTP サービスと DW スキーマを別の表領域に分ける** (理想的には別のデータベースや PDB)。I/O プロファイルとバックアップ戦略を分離するため。
- **ディメンション表ではサロゲート・キーを使用し**、ビジネス/ナチュラル・キーをディメンションの主キーに使用しない。ナチュラル・キーは変更される可能性があるが、サロゲート・キーは不変である。
- **慎重に事前集計を行う。** Oracle のマテリアライズド・ビューは、オプティマイザが自動的に使用（クエリ・リライト）できる事前集計サマリーとして機能する。
- **データウェアハウス表に圧縮を適用する。** Exadata (HCC) では、`COMPRESS FOR QUERY HIGH` により、ファクト表で 10 倍以上の圧縮を実現できる。非 Exadata 環境では、`ROW STORE COMPRESS ADVANCED` (オプションが必要) を使用することで、DW の一括ロードで通常 2:1 から 4:1 の圧縮率が得られる。
- **ETL パターンを考慮して設計する。** 初日からすべての DW テーブルに監査列 (`LOAD_DATE`, `SOURCE_SYSTEM`, `BATCH_ID`) を含めること。後からの追加は非常にコストがかかる。
- **DW ファクト表でのトリガー使用を避ける。** 大容量の一括ロードで単一行レベルのトリガーを使用するとパフォーマンスが壊滅的になる。ロード・プロセス内の ETL ロジックで行うこと。
- **すべてのファクト表の粒度 (Grain) を明示的に文書化する。** 粒度とは、ファクト表が記録する最も詳細なレベルのこと（例：「1日あたり、注文の1明細行あたり1行」）。

---

## 10. よくある間違いとその回避方法

### 間違い 1: 分析クエリに OLTP スキーマを使用する

正規化された OLTP スキーマに対して分析クエリを実行すると、巨大な結合ツリーと全表スキャンが発生する。分析用には別のディメンショナル・モデル（スター・スキーマなど）を維持すること。

### 間違い 2: オペレーショナルな主キーをディメンションのサロゲート・キーとして使用する

ビジネス・キー（注文番号、製品 SKU、顧客メールアドレス）は時間の経過とともに変更される可能性がある。ディメンション表には常に新しくサロゲート・キーを生成すること。

### 間違い 3: SCD 戦略が欠如している

本番開始前に SCD タイプを決定していないと、履歴の変更が警告なしに失われてしまう。最初のデータ・ロードの前に、ディメンション表ごとに SCD タイプを文書化し、実装すること。

### 間違い 4: 巨大なファクト表をパーティション化していない

パーティショニングなしで 5,000万から 1億行を超えるファクト表は、全表スキャンや非常に遅いメンテナンス操作（アーカイブ、削除、統計収集）に悩まされることになる。

```sql
-- 既存の巨大なテーブルにレンジ・パーティショニングを追加 (12.2 以降のオンライン操作)
ALTER TABLE FACT_SALES MODIFY
    PARTITION BY RANGE (date_key) INTERVAL (10000) (
        PARTITION p_initial VALUES LESS THAN (20200101)
    )
    ONLINE;
```

### 間違い 5: OLTP テーブルにビットマップ索引を使用する

ビットマップ索引は DML 操作中にビットマップ・セグメント・レベルでロックされる。ファクト表への単一の挿入や更新が、数十の並行トランザクションをブロックする可能性がある。ビットマップ索引は、メンテナンス・ウィンドウ中に一括ロードを行うデータウェアハウス表専用とすること。

### 間違い 6: 文書化せずに計算フィールドを保存する

パフォーマンスのために計算済みの値（粗利益、割引額など）を保存することは有効だが、式（計算式）を文書化しておかないと、ソースと計算値との間に不一致が生じた際に原因の特定が困難になる。可能な限り仮想列 (Virtual Column) を使用するか、列のコメントとして式を記述しておくこと。

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 23ai Concepts Guide](https://docs.oracle.com/en/database/oracle/oracle-database/23/cncpt/)
- [Oracle Database 23ai SQL Language Reference — CREATE TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/CREATE-TABLE.html)
- [Oracle Database 23ai VLDB and Partitioning Guide](https://docs.oracle.com/en/database/oracle/oracle-database/23/vldbg/)
- [Oracle Advanced Compression FAQ (oracle.com)](https://www.oracle.com/a/ocom/docs/database/advanced-compression-faq.pdf)
- [Oracle Exadata Hybrid Columnar Compression (docs.oracle.com)](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/exadata-hybrid-columnar-compression.html)
- [Oracle Database 12c R1 — In-Memory Column Store (oracle-base.com)](https://oracle-base.com/articles/12c/in-memory-column-store-12cr1)
- [Oracle Database 19c — Enabling Objects for In-Memory Population](https://docs.oracle.com/en/database/oracle/oracle-database/19/inmem/populating-objects-in-memory.html)
- [Oracle Database 23ai PL/SQL Packages and Types Reference — DBMS_SPACE](https://docs.oracle.com/en/database/oracle/oracle-database/23/arpls/DBMS_SPACE.html)
