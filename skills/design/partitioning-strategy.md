# Oracleのパーティショニング戦略

## 概要

Oracle の表パーティショニングは、巨大な表や索引を**パーティション**と呼ばれる小さく管理しやすいセグメントに論理的に分割する機能である。各パーティションは物理的には独立したセグメントだが、ユーザーは表名を通じて透過的にアクセスできる。パーティショニングには主に 3 つのメリットがある：

1. **パーティション・プルーニング (Partition Pruning)**: オプティマイザがクエリ・実行計画から不要なパーティションを除外し、I/O を劇的に削減する。
2. **パーティション単位の操作 (Partition-wise Operations)**: パラレル操作（結合、集計など）をパーティションの境界に沿って分割して実行できる。
3. **パーティションのメンテナンス**: 個別のパーティションのアーカイブ、削除、移動をアトミックな DDL 操作として実行できる。これは行レベルの削除よりもはるかに高速である。

パーティショニングの使用には、**Oracle Partitioning オプション** (Enterprise Edition) が必要である。使用可能かどうかは以下で確認できる：

```sql
SELECT value FROM v$option WHERE parameter = 'Partitioning';
```

---

## 1. レンジ・パーティショニング (Range Partitioning)

**レンジ・パーティショニング**は、列の値が指定された範囲内にあるかどうかに基づいて、行をパーティションに割り当てる。最も一般的なタイプであり、時系列データ（日付、タイムスタンプ、連続する ID）に最適である。

### 構文

```sql
CREATE TABLE SALES (
    sale_id       NUMBER         NOT NULL,
    sale_date     DATE           NOT NULL,
    customer_id   NUMBER         NOT NULL,
    product_id    NUMBER         NOT NULL,
    amount        NUMBER(14,2)   NOT NULL,
    CONSTRAINT pk_sales PRIMARY KEY (sale_id, sale_date)  -- 主キーにはパーティション・キーを含める必要がある
)
TABLESPACE sales_data
COMPRESS FOR QUERY HIGH
PARTITION BY RANGE (sale_date) (
    PARTITION p_2022_q1 VALUES LESS THAN (DATE '2022-04-01') TABLESPACE sales_2022,
    PARTITION p_2022_q2 VALUES LESS THAN (DATE '2022-07-01') TABLESPACE sales_2022,
    PARTITION p_2022_q3 VALUES LESS THAN (DATE '2022-10-01') TABLESPACE sales_2022,
    PARTITION p_2022_q4 VALUES LESS THAN (DATE '2023-01-01') TABLESPACE sales_2022,
    PARTITION p_2023_q1 VALUES LESS THAN (DATE '2023-04-01') TABLESPACE sales_2023,
    PARTITION p_future  VALUES LESS THAN (MAXVALUE)          TABLESPACE sales_data
);
```

### インターバル・パーティショニング (レンジの拡張)

Oracle 11g 以降では**インターバル・パーティショニング**をサポートしている。これは、既存の境界を超えるデータが挿入された際に新しいパーティションを自動的に作成する機能であり、手動でパーティションを追加する手間を省くことができる。

```sql
CREATE TABLE ORDERS (
    order_id    NUMBER        NOT NULL,
    order_date  DATE          NOT NULL,
    customer_id NUMBER        NOT NULL,
    total       NUMBER(14,2)  NOT NULL,
    CONSTRAINT pk_orders PRIMARY KEY (order_id, order_date)
)
TABLESPACE orders_data
PARTITION BY RANGE (order_date)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))  -- 1ヶ月ごとにパーティションを自動生成
(
    -- 少なくとも1つのパーティションを手動でシードとして定義する必要がある
    PARTITION p_before_2023 VALUES LESS THAN (DATE '2023-01-01')
        TABLESPACE orders_archive
);
```

インターバル・パーティションの格納先（表領域）は `SET STORE IN` で指定できる：

```sql
CREATE TABLE LOG_EVENTS (
    event_id    NUMBER     NOT NULL,
    event_time  TIMESTAMP  NOT NULL,
    severity    VARCHAR2(10),
    message     VARCHAR2(4000)
)
PARTITION BY RANGE (event_time)
INTERVAL (INTERVAL '1' DAY)
STORE IN (log_ts_1, log_ts_2, log_ts_3)  -- 表領域をラウンドロビンで使用
(
    PARTITION p_before_start VALUES LESS THAN (TIMESTAMP '2023-01-01 00:00:00')
);
```

### レンジ・パーティション・プルーニングの例

```sql
-- このクエリは p_2022_q1 と p_2022_q2 のみに絞り込まれる
SELECT sale_id, amount
FROM   SALES
WHERE  sale_date BETWEEN DATE '2022-01-01' AND DATE '2022-06-30';

-- 実行計画でプルーニングを確認する
EXPLAIN PLAN FOR
SELECT sale_id, amount FROM SALES
WHERE  sale_date BETWEEN DATE '2022-01-01' AND DATE '2022-06-30';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'PARTITION'));
-- 実行計画の出力にある "Pstart" と "Pstop" 列を確認する
```

---

## 2. リスト・パーティショニング (List Partitioning)

**リスト・パーティショニング**は、列の不連続で列挙可能な値に基づいて、行をパーティションに割り当てる。地域、ステータス、国コード、ビジネス・ユニットなどのカテゴリ・データに最適である。

### 構文

```sql
CREATE TABLE CUSTOMERS (
    customer_id   NUMBER         NOT NULL,
    region        VARCHAR2(20)   NOT NULL,  -- パーティション・キー
    full_name     VARCHAR2(200)  NOT NULL,
    email         VARCHAR2(255)  NOT NULL,
    CONSTRAINT pk_customers PRIMARY KEY (customer_id, region)
)
PARTITION BY LIST (region) (
    PARTITION p_north_america VALUES ('US', 'CA', 'MX'),
    PARTITION p_europe         VALUES ('GB', 'DE', 'FR', 'IT', 'ES', 'NL'),
    PARTITION p_apac           VALUES ('AU', 'JP', 'SG', 'IN', 'CN', 'KR'),
    PARTITION p_latam          VALUES ('BR', 'AR', 'CL', 'CO'),
    PARTITION p_other          VALUES (DEFAULT)  -- キャッチオール (11g以降: DEFAULT キーワード)
);
```

**注意:** Oracle 11g 以降で `DEFAULT` パーティションを指定せずに、マッピングされていない値を挿入しようとすると、`ORA-14400: 挿入されたパーティション・キーはどのパーティションにもマップされません` エラーが発生する。常に `DEFAULT` または `MAXVALUE` を含めることが推奨される。

### 自動リスト・パーティショニング (Oracle 12c R2 以降)

```sql
CREATE TABLE TRANSACTIONS (
    txn_id       NUMBER       NOT NULL,
    payment_type VARCHAR2(30) NOT NULL,
    amount       NUMBER(14,2) NOT NULL
)
PARTITION BY LIST (payment_type) AUTOMATIC  -- 新しいパーティションをオンザフライで作成
(
    PARTITION p_credit_card VALUES ('VISA', 'MASTERCARD', 'AMEX')
);
-- 'PAYPAL' が最初に挿入されると、パーティション p_sys_p_xxx が自動的に作成される
```

---

## 3. ハッシュ・パーティショニング (Hash Partitioning)

**ハッシュ・パーティショニング**は、パーティション・キーに適用される内部ハッシュ関数を使用して、行を固定数のパーティションに均等に分散させる。値の範囲やリストによるパーティション・プルーニングはサポート**しない**。主に **I/O 分散**と**パラレル結合パフォーマンス**のために使用される。

### ハッシュ・パーティショニングを使用する場合

- 自然な範囲やカテゴリの境界が存在しない場合
- ストレージ・デバイス間での I/O 分散を均等にすることが目的の場合
- **パーティション単位の結合 (Partition-wise Join)** を有効にする場合（2 つのハッシュ・パーティション表を同じキー、同じパーティション数で結合する場合）
- 高並行性の挿入が行われる表でのホット・ブロック競合を削減する場合

### 構文

```sql
-- 均等に分散させるため、パーティション数は必ず 2 のべき乗にすること
CREATE TABLE SESSION_EVENTS (
    event_id     NUMBER        NOT NULL,
    session_id   VARCHAR2(64)  NOT NULL,  -- ハッシュ・キー: 均等に分散される
    event_type   VARCHAR2(50)  NOT NULL,
    event_time   TIMESTAMP     NOT NULL,
    payload      CLOB,
    CONSTRAINT pk_session_events PRIMARY KEY (event_id)
)
PARTITION BY HASH (session_id)
PARTITIONS 16  -- 2 のべき乗: 2, 4, 8, 16, 32, 64...
STORE IN (hash_ts_1, hash_ts_2, hash_ts_3, hash_ts_4);  -- 16個のパーティションを4つの表領域に分散
```

### パーティション単位の結合 (ハッシュ・パーティショニングのメリット)

```sql
-- 両方の表が同じキー、同じパーティション数でハッシュ・パーティション化されている場合
CREATE TABLE ORDERS_H (
    order_id    NUMBER NOT NULL,
    customer_id NUMBER NOT NULL  -- ハッシュ・パーティション・キー
)
PARTITION BY HASH (customer_id) PARTITIONS 32;

CREATE TABLE ORDER_ITEMS_H (
    item_id     NUMBER NOT NULL,
    order_id    NUMBER NOT NULL,
    customer_id NUMBER NOT NULL  -- 同じハッシュ・パーティション・キー
)
PARTITION BY HASH (customer_id) PARTITIONS 32;

-- Oracle は完全なパーティション単位の結合を実行可能: 各パーティション・ペアが独立して結合される
SELECT /*+ PQ_DISTRIBUTE(i PARTITION NONE) */ o.order_id, i.item_id
FROM   ORDERS_H o
JOIN   ORDER_ITEMS_H i ON o.customer_id = i.customer_id AND o.order_id = i.order_id;
```

---

## 4. コンポジット・パーティショニング (Composite Partitioning)

**コンポジット・パーティショニング**（複合パーティショニング）は、2 つのパーティショニング戦略を組み合わせる。第 1 の戦略（パーティション・レベル）と、第 2 の戦略（サブパーティション・レベル）である。これにより、最上位キーによるパーティション・プルーニングと、各パーティション内での均等分散やカテゴリ化の両方が可能になる。

### レンジ - ハッシュ コンポジット

時系列データであり、かつ各期間内での I/O 分散も必要な場合に最適である。

```sql
CREATE TABLE CLICKSTREAM (
    event_id      NUMBER        NOT NULL,
    event_date    DATE          NOT NULL,   -- レンジ・パーティション・キー
    user_id       NUMBER        NOT NULL,   -- ハッシュ・サブパーティション・キー
    page_url      VARCHAR2(2000),
    event_type    VARCHAR2(50),
    CONSTRAINT pk_clickstream PRIMARY KEY (event_id, event_date)
)
PARTITION BY RANGE (event_date)
SUBPARTITION BY HASH (user_id) SUBPARTITIONS 8
(
    PARTITION p_2023_q1 VALUES LESS THAN (DATE '2023-04-01')
        STORE IN (cs_ts_1, cs_ts_2, cs_ts_3, cs_ts_4),
    PARTITION p_2023_q2 VALUES LESS THAN (DATE '2023-07-01')
        STORE IN (cs_ts_1, cs_ts_2, cs_ts_3, cs_ts_4),
    PARTITION p_2023_q3 VALUES LESS THAN (DATE '2023-10-01'),
    PARTITION p_future  VALUES LESS THAN (MAXVALUE)
);
```

### レンジ - リスト コンポジット

時系列データであり、かつ地域やカテゴリによるセグメント化も必要な場合に最適である。

```sql
-- サブパーティション・テンプレート: すべてのレンジ・パーティションに一貫したサブパーティション構造を適用する
CREATE TABLE REGIONAL_SALES (
    sale_id     NUMBER         NOT NULL,
    sale_date   DATE           NOT NULL,
    region      VARCHAR2(20)   NOT NULL,
    amount      NUMBER(14,2)   NOT NULL,
    CONSTRAINT pk_reg_sales PRIMARY KEY (sale_id, sale_date, region)
)
PARTITION BY RANGE (sale_date)
SUBPARTITION BY LIST (region)
SUBPARTITION TEMPLATE (                          -- テンプレートがすべてのレンジ・パーティションに適用される
    SUBPARTITION sp_na    VALUES ('US', 'CA', 'MX'),
    SUBPARTITION sp_eu    VALUES ('GB', 'DE', 'FR'),
    SUBPARTITION sp_apac  VALUES ('AU', 'JP', 'SG'),
    SUBPARTITION sp_other VALUES (DEFAULT)
)
(
    PARTITION p_2023 VALUES LESS THAN (DATE '2024-01-01'),
    PARTITION p_2024 VALUES LESS THAN (DATE '2025-01-01'),
    PARTITION p_future VALUES LESS THAN (MAXVALUE)
);

-- 両方のレベルで同時にプルーニングが機能する
SELECT * FROM REGIONAL_SALES
WHERE  sale_date >= DATE '2023-01-01'
AND    sale_date <  DATE '2024-01-01'
AND    region IN ('US', 'CA');
-- p_2023 パーティションの sp_na サブパーティションのみに絞り込まれる
```

### リスト - ハッシュ コンポジット

最上位レベルではカテゴリで分割し、各カテゴリ内でサブパーティションによって均等分散させる場合に最適である。

```sql
CREATE TABLE PRODUCT_INVENTORY (
    inventory_id  NUMBER       NOT NULL,
    warehouse_id  NUMBER       NOT NULL,  -- ハッシュ・サブパーティション・キー
    category      VARCHAR2(50) NOT NULL,  -- リスト・パーティション・キー
    product_id    NUMBER       NOT NULL,
    quantity      NUMBER       NOT NULL
)
PARTITION BY LIST (category)
SUBPARTITION BY HASH (warehouse_id) SUBPARTITIONS 4
(
    PARTITION p_electronics VALUES ('PHONES', 'LAPTOPS', 'TABLETS'),
    PARTITION p_clothing     VALUES ('SHIRTS', 'PANTS', 'SHOES'),
    PARTITION p_food         VALUES ('FRESH', 'FROZEN', 'DRY'),
    PARTITION p_other        VALUES (DEFAULT)
);
```

---

## 5. パーティション・プルーニング (Partition Pruning)

パーティション・プルーニングは、クエリ実行時に不要なパーティションを除外するオプティマイザの機能である。これはパーティショニングによる最大のパフォーマンス上のメリットであり、パーティション・キーが `WHERE` 句に**定数またはバインド変数**の述語として含まれている場合にのみ機能する。

### プルーニングの要件

```sql
-- 良い例: 定数リテラル — コンパイル時にプルーニングされる
SELECT * FROM ORDERS WHERE order_date >= DATE '2024-01-01';

-- 良い例: バインド変数 — 実行時にプルーニングされる（これでも効率的）
SELECT * FROM ORDERS WHERE order_date >= :start_date;

-- 悪い例: パーティション・キーに関数を適用している — プルーニングが無効になる
SELECT * FROM ORDERS WHERE TRUNC(order_date) >= DATE '2024-01-01';
-- 対策: あらかじめ切り捨てられた日付列を別途保存するか、以下のように記述する：
-- order_date >= DATE '2024-01-01' AND order_date < DATE '2024-01-02'

-- 悪い例: 暗黙の型変換 — プルーニングが無効になる可能性がある
SELECT * FROM ORDERS WHERE order_date >= '2024-01-01';  -- 文字列との比較
-- 対策: 常に明示的な DATE リテラルまたは TO_DATE() を使用する
```

### 実行計画でのプルーニングの確認

```sql
-- 方法1: DBMS_XPLAN を PARTITION フォーマットで使用
EXPLAIN PLAN FOR
SELECT * FROM SALES WHERE sale_date BETWEEN DATE '2024-01-01' AND DATE '2024-03-31';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(NULL, NULL, 'ALL'));
-- 出力結果を確認: "Pstart=1 Pstop=1"（単一パーティション）ならOK。"Pstart=1 Pstop=4"なら複数にまたがっている。

-- 方法2: 実行中のクエリに対して V$SQL_PLAN を確認
SELECT partition_start, partition_stop, partition_id, operation, options
FROM   v$sql_plan
WHERE  sql_id = :sql_id
ORDER  BY id;
```

### パーティション統計情報

```sql
-- テーブルのパーティション・レベルの統計情報を確認
SELECT table_name, partition_name, num_rows, blocks, last_analyzed
FROM   user_tab_partitions
WHERE  table_name = 'SALES'
ORDER  BY partition_position;
```

---

## 6. ローカル索引 vs グローバル索引

### ローカル索引 (Local Indexes)

**ローカル索引**は、基になる表と同じ戦略でパーティション化される。各索引パーティションは、対応する表パーティションと正確に 1 対 1 で対応する。パーティション表では通常、ローカル索引が推奨される。

```sql
-- ローカル非一意索引
CREATE INDEX IX_SALES_CUSTOMER ON SALES (customer_id)
LOCAL  -- 表パーティションごとに索引パーティションを作成
TABLESPACE sales_idx;

-- ローカル一意索引 (一意性を保証するためパーティション・キーを含める必要がある)
CREATE UNIQUE INDEX UX_SALES_ORDER_LINE ON SALES (order_id, line_id, sale_date)
LOCAL
TABLESPACE sales_idx;
```

**ローカル索引の特徴:**
- パーティション操作（分割、統合、削除、交換）時に自動的にメンテナンスされる
- 表のパーティション・プルーニングが機能すると、索引も自動的にプルーニングされる
- `UNUSABLE` 状態は影響を受けたパーティションのみに限定され、索引全体に波及しない
- 一意ローカル索引を作成するには、一意キーの中にパーティション・キーを含める必要がある

### グローバル索引 (Global Indexes)

**グローバル索引**は、すべての表パーティションを単一の索引構造（または表とは異なるパーティション構造）でカバーする。パーティション・キーに依存せず、一意性を適用できる。

```sql
-- グローバル・パーティション索引 (索引キーに基づいてレンジ・パーティション化される)
CREATE UNIQUE INDEX UX_ORDERS_ORDER_NUM ON ORDERS (order_number)
GLOBAL
PARTITION BY RANGE (order_number) (
    PARTITION p_low    VALUES LESS THAN (1000000),
    PARTITION p_mid    VALUES LESS THAN (9000000),
    PARTITION p_high   VALUES LESS THAN (MAXVALUE)
)
TABLESPACE orders_idx;

-- グローバル非パーティション索引（すべてのパーティションをカバーする古典的な単一構造の索引）
CREATE INDEX IX_ORDERS_EMAIL ON ORDERS (customer_email)
GLOBAL  -- 記述を省略した場合（非パーティション索引）もデフォルトでこれになる
TABLESPACE orders_idx;
```

**グローバル索引の特徴:**
- パーティション・キーを含まない列に対して一意制約を適用可能
- `UPDATE GLOBAL INDEXES` を指定しない限り、パーティションのメンテナンス操作後に再構築（または UNUSABLE 指定）が必要になる
- パーティション DDL 操作中のメンテナンス・コストが高くなる

```sql
-- グローバル索引の更新を伴うパーティション削除
ALTER TABLE ORDERS DROP PARTITION p_2020
UPDATE GLOBAL INDEXES;  -- グローバル索引を有効に保つ。低速だが再構築を回避できる

-- あるいはグローバル索引を一度 UNUSABLE にしてから再構築する
-- （索引句を指定しない場合、デフォルトで UNUSABLE になる）
ALTER TABLE ORDERS DROP PARTITION p_2020;
ALTER INDEX UX_ORDERS_ORDER_NUM REBUILD;
```

### ローカル vs グローバルの決定基準

| 基準 | ローカル索引 | グローバル索引 |
|---|---|---|
| パーティション・メンテナンスの影響 | 最小限（影響を受けた分のみ） | 全体の再構築が必要（`UPDATE GLOBAL INDEXES` を使用しない場合） |
| パーティション・プルーニングのサポート | 表と索引の両方で機能 | 索引レベルのプルーニングのみ（グローバルが分割されている場合） |
| パーティション・キーなしの一意制約 | 不可能 | サポートされている |
| 非パーティション・キーでの範囲スキャン | 効率が低い（全索引パーティションが必要） | 効率が高い |
| 最適な用途 | DW のファクト表、時系列データ | OLTP の一意識別検索、複数のパーティションをまたぐ範囲スキャン |

---

## 7. パーティションのメンテナンス操作

```sql
-- レンジ・パーティション表に新しいパーティションを追加
ALTER TABLE SALES ADD PARTITION p_2025_q1
    VALUES LESS THAN (DATE '2025-04-01')
    TABLESPACE sales_2025;

-- 古いパーティション（およびそのデータ！）を削除 — 元に戻せない
ALTER TABLE SALES DROP PARTITION p_2022_q1
UPDATE GLOBAL INDEXES;

-- パーティションの切り捨て (パーティション内の全行を削除 — DELETE よりも高速)
ALTER TABLE SALES TRUNCATE PARTITION p_2022_q1;

-- ステージング表を使用したパーティションの交換 (Zero-copy ETL)
-- 1. 同一構造のステージング表を作成 (パーティショニングなし)
CREATE TABLE SALES_STAGING AS SELECT * FROM SALES WHERE 1=0;

-- 2. ステージング表にデータをロード (bulk insert 等)
INSERT /*+ APPEND */ INTO SALES_STAGING SELECT ...;
COMMIT;

-- 3. 交換: パーティションとステージング表をアトミックに入れ替える
ALTER TABLE SALES EXCHANGE PARTITION p_2024_q4
WITH TABLE SALES_STAGING
INCLUDING INDEXES
WITHOUT VALIDATION;  -- パフォーマンスのため行検証をスキップ

-- 1つのパーティションを2つに分割
ALTER TABLE SALES SPLIT PARTITION p_future
    AT (DATE '2026-01-01')
    INTO (PARTITION p_2025_q4, PARTITION p_future)
UPDATE GLOBAL INDEXES;

-- 2つの隣接するパーティションを統合
ALTER TABLE SALES MERGE PARTITIONS p_2022_q1, p_2022_q2
    INTO PARTITION p_2022_h1
UPDATE GLOBAL INDEXES;
```

---

## 8. パーティショニング・タイプの使い分け

| パーティショニング・タイプ | 最適なユースケース | 避けるべきケース |
|---|---|---|
| **レンジ (Range)** | 時系列データ、ログ表、過去アーカイブ。ローリング・ウィンドウ（古いものを捨て、新しいものを足す）が可能。 | データに自然な範囲の順序がない場合。 |
| **インターバル (Interval)** | レンジと同じだが、パーティションが自動作成される。大量の追記型テーブル。 | パーティション作成のタイミングを厳密に制御する必要がある場合。 |
| **リスト (List)** | 地域データ、ステータス・コード、ビジネス・ユニット。値が不連続で列挙可能な場合。 | 異なる値の種類が多すぎる場合。値が頻繁に変わる場合。 |
| **自動リスト (Automatic List)** | リストと同じだが、将来の値が予見できない場合。マルチテナント・スキーマ。 | パーティション名や場所を予測可能にしておく必要がある場合。 |
| **ハッシュ (Hash)** | 均等な I/O 分散。ホット・スポットの解消。パーティション単位の結合。 | 値の範囲によるプルーニングが必要な場合。キーによる連続スキャンの場合。 |
| **レンジ - ハッシュ** | 期間ごとに、かつ期間内での均等分散が必要な時系列データ。 | 並行性の問題がない単純な時系列データ。 |
| **レンジ - リスト** | 期間ごとに、かつ地域やカテゴリによる分割が必要な時系列データ。 | リストのカテゴリが不安定、または多すぎる場合。 |
| **リスト - ハッシュ** | カテゴリごとに、かつカテゴリ内での均等分散が必要なデータ。 | スキュー（偏り）がない単純なカテゴリ・データ。 |

---

## 9. ベスト・プラクティス

- **一意ローカル索引を使用する場合、主キーや一意キーにパーティション・キーを必ず含めること。** Oracle の制約上、これは必須である。
- **データ量とプルーニング・パターンに基づいてパーティションの粒度を選択する。** 月間 1,000万行程度なら月次、それ以下なら四半期や年次を検討する。
- **インターバル・パーティショニングを活用する。** 追記型の時系列データに使用することで、メンテナンスの手間（手動追加）を排除できる。
- **パーティション・レベルで統計情報を収集する。** 正確なカーディナリティの見積もりのため、パーティションごとの最新の統計情報をオプティマイザに提供すること。
  ```sql
  EXEC DBMS_STATS.GATHER_TABLE_STATS(
      ownname          => 'SCHEMA_OWNER',
      tabname          => 'SALES',
      partname         => 'P_2024_Q4',
      granularity      => 'PARTITION',
      estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE
  );
  ```
- **DW のファクト表パーティションには `COMPRESS FOR QUERY HIGH` を使用する。** 過去データの再圧縮を避けるため、パーティション単位で適用すること。
- **DW のテーブルにはローカル索引を基本とし、グローバル索引は最小限にする。** パーティションをまたいだ一意性や、パーティション・キー以外での範囲スキャンが必要な場合にのみグローバル索引を検討する。

---

## 10. よくある間違いとその回避方法

### 間違い 1: WHERE 句でパーティション・キーに関数を適用する

```sql
-- 悪い例: TRUNC() によってパーティション・プルーニングが機能しなくなる
SELECT * FROM ORDERS WHERE TRUNC(order_date, 'MM') = DATE '2024-01-01';

-- 良い例: パーティション・キーに対して直接範囲指定を行う
SELECT * FROM ORDERS
WHERE  order_date >= DATE '2024-01-01'
AND    order_date <  DATE '2024-02-01';
```

### 間違い 2: ハッシュ・パーティション数を 2 のべき乗にしない

均等な分散を保証するためには、ハッシュ・パーティション数は 2 のべき乗である必要がある（2, 4, 8, 16...）。Oracle は任意の数を指定できるが、分散が不均等になると I/O バランシングのメリットが薄れる。

### 間違い 3: パーティション DDL で UPDATE GLOBAL INDEXES を忘れる

```sql
-- 悪い例: グローバル索引が UNUSABLE 状態のままになる
ALTER TABLE ORDERS DROP PARTITION p_old;

-- 良い例: グローバル索引の有効性を維持する
ALTER TABLE ORDERS DROP PARTITION p_old UPDATE GLOBAL INDEXES;
```

### 間違い 4: 小さなテーブルをパーティション化する

数百万行以下のテーブルをパーティション化しても、メタデータの管理や統計情報収集のコストが増えるだけで、メリットはほとんどない。パーティション・プルーニングによって I/O が有意に削減される規模のテーブルのみを対象とすること。

### 間違い 5: パーティション・キーを索引付けしていない

パーティショニングされていても、パーティション・キー以外の列で絞り込むクエリが多い場合は、それらの列に索引が必要である。パーティション・プルーニングは、述語にパーティション・キーが含まれている場合にのみ機能する。

### 間違い 6: 不適切なパーティション・キーの選択

パーティション・キーは、最も頻繁に使用されるクエリ・フィルタと一致させる必要がある。`LOAD_DATE` でパーティション化されているが、実際には `TRANSACTION_DATE` でクエリされる場合、プルーニングは機能しない。本番環境でこの決定を覆すのは困難なため、実際のクエリ・パターンを慎重に分析すること。

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 23ai VLDB and Partitioning Guide — Partition Concepts](https://docs.oracle.com/en/database/oracle/oracle-database/23/vldbg/partition-concepts.html)
- [Oracle Database 23ai VLDB and Partitioning Guide — Partition Administration](https://docs.oracle.com/en/database/oracle/oracle-database/23/vldbg/)
- [Oracle Database 23ai VLDB and Partitioning Guide — Partition Pruning](https://docs.oracle.com/en/database/oracle/oracle-database/21/vldbg/partition-pruning.html)
- [Oracle Database 12c R2 — Automatic List Partitioning (oracle-base.com)](https://oracle-base.com/articles/12c/automatic-list-partitioning-12cr2)
- [Oracle Database 11g R1 — Interval Partitioning (oracle-base.com)](https://oracle-base.com/articles/11g/partitioning-enhancements-11gr1)
- [Oracle Database 12c R2 — Partitioning Enhancements (oracle-base.com)](https://oracle-base.com/articles/12c/partitioning-enhancements-12cr2)
- [Oracle Database 12c R1 — Asynchronous Global Index Maintenance (oracle-base.com)](https://oracle-base.com/articles/12c/asynchronous-global-index-maintenance-for-drop-and-truncate-partition-12cr1)
- [Oracle Database 23ai SQL Language Reference — ALTER TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/)
- [Oracle Database Partitioning Option licensing](https://www.oracle.com/database/technologies/partitioning.html)
