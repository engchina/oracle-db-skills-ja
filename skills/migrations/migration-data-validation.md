# 移行データの検証

## 概要

データの検証は、データベース移行において最も見落とされやすいフェーズである。スキーマが自動的に変換され、エラーなくデータがロードされたとしても、行の欠落、数値の切り捨て、タイムゾーンの違いによる日時のずれ、あるいは NULL の扱いの違い（移行元と Oracle の差異）などが原因で、アプリケーションが静かに誤った結果を返す可能性がある。

本ガイドでは、SQL クエリ・ベースの包括的なデータ検証フレームワークを提供する。これらのパターンを使用して、初期ロード直後、各増分同期後、および切り替え後の継続的なドリフト（差異）検出を行っていただきたい。

---

## 行数検証

最初に行うべき、最もシンプルな検証は、移行されたすべての表にわたる行数の比較である。

### 基本的な行数クエリ

移行元と移行先の両方で以下を実行し、結果を比較する。

```sql
-- 移行元 (PostgreSQL)
SELECT 'customers'::text AS table_name, COUNT(*) AS row_count FROM customers
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'products', COUNT(*) FROM products
ORDER BY table_name;
27: 
-- 移行先 Oracle (同様のパターン)
SELECT 'customers' AS table_name, COUNT(*) AS row_count FROM customers
UNION ALL SELECT 'orders', COUNT(*) FROM orders
UNION ALL SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL SELECT 'products', COUNT(*) FROM products
ORDER BY table_name;
```

### PL/SQL による行数比較の自動化

以下のプロシージャは、移行元から期待される行数を保存し、Oracle 側の件数と比較する。

```sql
CREATE TABLE migration_row_counts (
    table_name       VARCHAR2(128)  NOT NULL,
    source_count     NUMBER         NOT NULL,
    oracle_count     NUMBER,
    captured_at      TIMESTAMP      DEFAULT SYSTIMESTAMP,
    validation_at    TIMESTAMP,
    difference       NUMBER GENERATED ALWAYS AS (oracle_count - source_count) VIRTUAL,
    status           VARCHAR2(10)   GENERATED ALWAYS AS
                         (CASE WHEN oracle_count = source_count THEN 'PASS'
                               WHEN oracle_count IS NULL THEN 'PENDING'
                               ELSE 'FAIL' END) VIRTUAL
);

-- 移行元の件数をロード (移行元データベースのクエリ結果から)
INSERT INTO migration_row_counts (table_name, source_count) VALUES
    ('CUSTOMERS',   150000),
    ('ORDERS',      1200000),
    ('ORDER_ITEMS', 4800000),
    ('PRODUCTS',    5000);

-- 移行後に Oracle の件数で更新
BEGIN
    FOR t IN (SELECT table_name FROM migration_row_counts WHERE oracle_count IS NULL) LOOP
        EXECUTE IMMEDIATE
            'UPDATE migration_row_counts
             SET oracle_count = (SELECT COUNT(*) FROM ' || t.table_name || '),
                 validation_at = SYSTIMESTAMP
             WHERE table_name = ''' || t.table_name || '''';
    END LOOP;
    COMMIT;
END;
/

-- 結果の確認
SELECT table_name, source_count, oracle_count, difference, status
FROM migration_row_counts
ORDER BY status DESC, ABS(difference) DESC;
```

### パーティション表の行数検証

パーティション表の場合は、パーティションごとの件数も検証する。

```sql
-- Oracle: パーティションごとの行数
SELECT table_name, partition_name, num_rows
FROM user_tab_partitions
WHERE table_name = 'FACT_SALES'
ORDER BY partition_position;

-- 必要に応じて、事前に統計情報を明示的に収集する
EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'FACT_SALES', GRANULARITY => 'ALL');
```

---

## ORA_HASH を使用したハッシュ・ベースの比較

行数は「量」を確認できるが、「内容」は確認できない。ハッシュ・ベースの検証では、実際のデータ値を比較する。

### ORA_HASH の概要

`ORA_HASH(expr, max_bucket, seed)` は、任意のスカラー式に対して決定論的なハッシュ・バケット番号（0 から `max_bucket`）を返す。すべての列値を連結して各行をハッシュ化し、それらを合計（SUM）または排他的論理和（XOR）することで、表全体の内容の「フィンガープリント（指紋）」を作成できる。

```sql
-- Oracle: 表レベルのハッシュ・フィンガープリントを計算
SELECT
    SUM(ORA_HASH(
        customer_id || '|' || first_name || '|' || last_name || '|' ||
        email || '|' || TO_CHAR(created_date, 'YYYY-MM-DD HH24:MI:SS'),
        4294967295  -- max bucket = 32ビット符号なし整数の最大値
    )) AS table_hash,
    COUNT(*) AS row_count
FROM customers;
```

Oracle での結果は、移行元データベースで計算された対応するハッシュと一致する必要がある。ハッシュ関数はプラットフォームによって異なるため、同じハッシュ・アルゴリズムを使用するか、中間結果を比較する必要がある。

### クロス・プラットフォームでのハッシュ比較戦略

`ORA_HASH` は Oracle 固有であるため、プラットフォームに依存しないアプローチをとる。個別の行をハッシュ化して文字列にし、それらを比較する。

**ステップ 1：移行元 (PostgreSQL) からキーとハッシュを出力する**
```sql
-- PostgreSQL: 各行の MD5 を計算し、主キーでソート
SELECT
    customer_id,
    MD5(COALESCE(first_name, '') || '|' ||
        COALESCE(last_name, '')  || '|' ||
        COALESCE(email, '')      || '|' ||
        COALESCE(TO_CHAR(created_date, 'YYYY-MM-DD'), ''))
    AS row_hash
FROM customers
ORDER BY customer_id
LIMIT 1000;  -- 最初はサンプルで検証
```

**ステップ 2：Oracle で同じハッシュを計算する**
```sql
-- Oracle: DBMS_CRYPTO を使用して各行の MD5 を計算
SELECT
    customer_id,
    LOWER(RAWTOHEX(
        DBMS_CRYPTO.HASH(
            UTL_I18N.STRING_TO_RAW(
                NVL(first_name, '') || '|' ||
                NVL(last_name, '')  || '|' ||
                NVL(email, '')      || '|' ||
                NVL(TO_CHAR(created_date, 'YYYY-MM-DD'), ''),
                'AL32UTF8'
            ),
            DBMS_CRYPTO.HASH_MD5
        )
    )) AS row_hash
FROM customers
WHERE customer_id <= 1000
ORDER BY customer_id;
```

**ステップ 3：結果を比較する** — 両方の結果セットを CSV に出力して diff をとるか、Oracle 側の比較用表にロードして結合（JOIN）する。

```sql
-- 移行元のハッシュをステージング表にロード
CREATE TABLE hash_staging_src (
    customer_id NUMBER,
    row_hash    VARCHAR2(32)
);

-- SQL*Loader や INSERT で移行元のハッシュをロードした後：
-- Oracle で計算したハッシュと比較
SELECT
    o.customer_id,
    s.row_hash AS source_hash,
    o.row_hash AS oracle_hash,
    CASE WHEN s.row_hash = o.row_hash THEN 'MATCH' ELSE 'MISMATCH' END AS result
FROM (
    SELECT
        customer_id,
        LOWER(RAWTOHEX(DBMS_CRYPTO.HASH(
            UTL_I18N.STRING_TO_RAW(
                NVL(first_name,'') || '|' || NVL(last_name,'') || '|' ||
                NVL(email,'') || '|' || NVL(TO_CHAR(created_date,'YYYY-MM-DD'),''),
                'AL32UTF8'
            ), DBMS_CRYPTO.HASH_MD5
        ))) AS row_hash
    FROM customers
) o
JOIN hash_staging_src s ON o.customer_id = s.customer_id
WHERE s.row_hash != o.row_hash
ORDER BY o.customer_id;
```

---

## データ・サンプリング戦略

初期検証において、数十億行の表に対して完全なハッシュ検証を行うのは現実的ではない。層化サンプリングを使用して、短時間で統計的な信頼性を得る。

### ランダム・サンプリング検証

```sql
-- Oracle: 大規模な表の 0.1% をランダムに抽出して検証
SELECT customer_id, email, created_date, status
FROM customers
SAMPLE(0.1)
ORDER BY customer_id;
-- SAMPLE(n) は Oracle 固有：行の約 n% を抽出する
```

### 境界値サンプリング

値の範囲の極端なケースは必ず検証する。

```sql
-- 主要な列における最小値/最大値の検証
SELECT
    'order_amount' AS column_name,
    MIN(total_amount) AS min_val,
    MAX(total_amount) AS max_val,
    AVG(total_amount) AS avg_val,
    STDDEV(total_amount) AS stddev_val,
    COUNT(*) AS total_rows,
    COUNT(CASE WHEN total_amount IS NULL THEN 1 END) AS null_count
FROM orders

UNION ALL

SELECT
    'customer_id',
    MIN(customer_id), MAX(customer_id), AVG(customer_id), STDDEV(customer_id),
    COUNT(*), COUNT(CASE WHEN customer_id IS NULL THEN 1 END)
FROM orders

UNION ALL

SELECT
    'order_date',
    TO_NUMBER(TO_CHAR(MIN(order_date),'YYYYMMDD')),
    TO_NUMBER(TO_CHAR(MAX(order_date),'YYYYMMDD')),
    NULL, NULL, COUNT(*),
    COUNT(CASE WHEN order_date IS NULL THEN 1 END)
FROM orders;
```

### 日付範囲による層化サンプリング

```sql
-- 月ごとの行数検証 — 移行元と完全に一致するはず
SELECT
    TO_CHAR(order_date, 'YYYY-MM') AS order_month,
    COUNT(*) AS row_count,
    SUM(total_amount) AS total_revenue,
    MIN(total_amount) AS min_order,
    MAX(total_amount) AS max_order
FROM orders
WHERE order_date >= DATE '2020-01-01'
GROUP BY TO_CHAR(order_date, 'YYYY-MM')
ORDER BY order_month;
```

### Top-N 検証

最大の値や最も重要なレコードが正しく移行されているか確認する。

```sql
-- 金額の大きい上位100件の注文を比較
SELECT order_id, customer_id, total_amount, status, order_date
FROM orders
ORDER BY total_amount DESC
FETCH FIRST 100 ROWS ONLY;
```

---

## 日付と数値の精度チェック

### 日付の「ずれ（オフセット）」の検出

移行で最も多いバグの1つは、移行中に特定のタイムゾーン・オフセットが適用されてしまい、すべての時刻が数時間ずれてしまうことである。

```sql
-- 日付分布の比較：体系的なずれがないか確認
SELECT
    TRUNC(order_date, 'HH24') AS hour_of_day,
    COUNT(*) AS order_count
FROM orders
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'
GROUP BY TRUNC(order_date, 'HH24')
ORDER BY hour_of_day;
-- この分布を移行元と比較する。同一である必要がある。
```

```sql
-- 不自然に「午前0時（00:00:00）」に集中していないか確認 (DATE 精度の欠落を示唆)
SELECT
    CASE WHEN TO_CHAR(order_date, 'HH24:MI:SS') = '00:00:00' THEN 'Midnight' ELSE 'Non-midnight' END AS time_class,
    COUNT(*) AS row_count
FROM orders
GROUP BY CASE WHEN TO_CHAR(order_date, 'HH24:MI:SS') = '00:00:00' THEN 'Midnight' ELSE 'Non-midnight' END;
-- すべてが Midnight であり、移行元では実際の時刻情報があった場合、データが切り捨てられている。
```

```sql
-- TIMESTAMP 精度の確認
SELECT
    order_id,
    order_timestamp,
    TO_CHAR(order_timestamp, 'YYYY-MM-DD HH24:MI:SS.FF6') AS ts_with_microseconds
FROM orders
WHERE EXTRACT(SECOND FROM order_timestamp) != TRUNC(EXTRACT(SECOND FROM order_timestamp))
FETCH FIRST 10 ROWS ONLY;
-- 行が返ってくれば、秒の小数部が保持されている。
```

### 数値精度の検証

```sql
-- 小数点以下の列での切り捨てを確認
SELECT
    order_id,
    original_amount,  -- 移行元から（ステージングにロードしたもの）
    oracle_amount,    -- Oracle 表内の値
    original_amount - oracle_amount AS difference
FROM (
    SELECT
        o.order_id,
        s.amount  AS original_amount,
        o.amount  AS oracle_amount
    FROM orders o
    JOIN orders_source_staging s ON o.order_id = s.order_id
)
WHERE ABS(original_amount - oracle_amount) > 0.0001  -- 浮動小数比較の許容誤差
ORDER BY ABS(original_amount - oracle_amount) DESC;
```

```sql
-- 指数表記の問題を確認 (BINARY_DOUBLE vs DECIMAL)
SELECT order_id, amount, TO_CHAR(amount, 'FM9999999999.999999') AS formatted_amount
FROM orders
WHERE amount != ROUND(amount, 4)  -- 小数点以下4桁を超える金額を探す
FETCH FIRST 20 ROWS ONLY;
```

---

## NULL 处理の差異

### 空文字 vs NULL (PostgreSQL → Oracle)

NULL に関する最も一般的な問題は、空文字から NULL への変換である。Oracle 21c 以前では `''` は NULL として扱われる。PostgreSQL では `''` は NULL とは異なる空文字として扱われる。

```sql
-- Oracle: 期待される非 null 値が NULL になっている行を探す
-- (移行元での空文字が変換された可能性がある)
SELECT
    COUNT(*) AS total_rows,
    COUNT(CASE WHEN email IS NULL THEN 1 END) AS null_email_count,
    COUNT(CASE WHEN email = '' THEN 1 END) AS empty_email_count,  -- Oracle では常に 0 になるはず
    COUNT(CASE WHEN LENGTH(email) = 0 THEN 1 END) AS zero_len_email_count  -- これも 0
FROM customers;

-- NULL の割合を移行元と比較
-- 移行元の NULL% = Oracle の NULL% + 空文字% になっているか確認
```

```sql
-- NOT NULL 列における予期しない NULL の調査
DECLARE
    v_count NUMBER;
BEGIN
    FOR col IN (
        SELECT table_name, column_name
        FROM user_tab_columns
        WHERE nullable = 'N'
          AND table_name IN ('CUSTOMERS', 'ORDERS', 'PRODUCTS')
    ) LOOP
        EXECUTE IMMEDIATE
            'SELECT COUNT(*) FROM ' || col.table_name ||
            ' WHERE ' || col.column_name || ' IS NULL'
        INTO v_count;
        IF v_count > 0 THEN
            DBMS_OUTPUT.PUT_LINE('UNEXPECTED NULL: ' || col.table_name ||
                                 '.' || col.column_name || ' = ' || v_count);
        END IF;
    END LOOP;
END;
/
```

### BOOLEAN 値の検証

```sql
-- BOOLEAN 列が正しく 0/1 に移行されているか確認
SELECT
    COUNT(*) AS total,
    COUNT(CASE WHEN is_active = 1 THEN 1 END) AS active_count,
    COUNT(CASE WHEN is_active = 0 THEN 1 END) AS inactive_count,
    COUNT(CASE WHEN is_active NOT IN (0, 1) THEN 1 END) AS invalid_count,
    COUNT(CASE WHEN is_active IS NULL THEN 1 END) AS null_count
FROM customers;
-- invalid_count と null_count は共に 0 である必要がある
```

---

## 照合報告（リコンシリエーション・レポート）の作成

### マスター検証レポート・クエリ

すべての検証済み表を対象に、1つの要約レポートを作成するクエリ。

```sql
-- 検証結果保存用の表を作成
CREATE TABLE migration_validation_results (
    check_id        NUMBER GENERATED ALWAYS AS IDENTITY,
    table_name      VARCHAR2(128),
    check_type      VARCHAR2(50),
    check_detail    VARCHAR2(500),
    expected_value  NUMBER,
    actual_value    NUMBER,
    difference      NUMBER GENERATED ALWAYS AS (actual_value - expected_value) VIRTUAL,
    status          VARCHAR2(10),
    checked_at      TIMESTAMP DEFAULT SYSTIMESTAMP
);

-- 検証結果の挿入
INSERT INTO migration_validation_results (table_name, check_type, check_detail, expected_value, actual_value, status)
-- 行数 (事前に移行元からロードしておいた期待値を使用)
SELECT
    mrc.table_name,
    'ROW_COUNT',
    'Total row count',
    mrc.source_count,
    mrc.oracle_count,
    CASE WHEN mrc.source_count = mrc.oracle_count THEN 'PASS' ELSE 'FAIL' END
FROM migration_row_counts mrc;

COMMIT;

-- 要約レポートの生成
SELECT
    check_type,
    COUNT(*) AS total_checks,
    SUM(CASE WHEN status = 'PASS' THEN 1 ELSE 0 END) AS passed,
    SUM(CASE WHEN status = 'FAIL' THEN 1 ELSE 0 END) AS failed,
    SUM(CASE WHEN status = 'WARN' THEN 1 ELSE 0 END) AS warnings,
    ROUND(SUM(CASE WHEN status = 'PASS' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS pass_pct
FROM migration_validation_results
GROUP BY check_type
ORDER BY failed DESC, check_type;
```

### エグゼクティブ・サマリー・レポート

```sql
SELECT
    'Migration Validation Summary' AS report_title,
    SYSTIMESTAMP AS report_time,
    COUNT(*) AS total_checks,
    SUM(CASE WHEN status = 'PASS' THEN 1 ELSE 0 END) AS total_passed,
    SUM(CASE WHEN status = 'FAIL' THEN 1 ELSE 0 END) AS total_failed,
    ROUND(SUM(CASE WHEN status = 'PASS' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS overall_pass_pct,
    CASE
        WHEN SUM(CASE WHEN status = 'FAIL' THEN 1 ELSE 0 END) = 0 THEN 'READY FOR CUTOVER'
        WHEN SUM(CASE WHEN status = 'FAIL' THEN 1 ELSE 0 END) <= 5 THEN 'REVIEW REQUIRED'
        ELSE 'NOT READY — FAILURES MUST BE RESOLVED'
    END AS overall_status
FROM migration_validation_results;
```

---

## 継続的なドリフト検出

切り替え後、フォールバックのために移行元データベースがオンラインのまま維持されている場合、あるいはパラレル・ライト（並列書き込み）モードで実行している場合、移行元と Oracle 間のドリフトを継続的に検出する。

### ドリフト検出フレームワーク

```sql
-- ドリフト監視：行数のハイウォーターマークを追跡
CREATE TABLE drift_monitor (
    table_name      VARCHAR2(128)  NOT NULL,
    check_timestamp TIMESTAMP      DEFAULT SYSTIMESTAMP,
    oracle_count    NUMBER,
    expected_count  NUMBER,
    delta           NUMBER GENERATED ALWAYS AS (oracle_count - expected_count) VIRTUAL,
    drift_pct       NUMBER GENERATED ALWAYS AS
                        (ROUND((oracle_count - expected_count) / NULLIF(expected_count, 0) * 100, 4)) VIRTUAL
);

-- スケジュール実行されるドリフト・チェック・プロシージャ
CREATE OR REPLACE PROCEDURE run_drift_check AS
    v_count NUMBER;
BEGIN
    FOR t IN (SELECT DISTINCT table_name FROM migration_row_counts) LOOP
        EXECUTE IMMEDIATE 'SELECT COUNT(*) FROM ' || t.table_name INTO v_count;
        INSERT INTO drift_monitor (table_name, oracle_count, expected_count)
        SELECT t.table_name, v_count, MAX(oracle_count)
        FROM migration_row_counts
        WHERE table_name = t.table_name;
    END LOOP;
    COMMIT;
END;
/

-- ジョブのスケジュール (1時間おき)
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'DRIFT_CHECK_JOB',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN run_drift_check; END;',
        repeat_interval => 'FREQ=HOURLY',
        enabled         => TRUE
    );
END;
/
```

### 重大なドリフトの警告

```sql
-- 0.01% を超えるドリフトが発生している表を特定
SELECT table_name, check_timestamp, oracle_count, expected_count, drift_pct
FROM drift_monitor
WHERE drift_pct > 0.01  -- 0.01% 以上の行数差
  AND check_timestamp > SYSTIMESTAMP - INTERVAL '24' HOUR
ORDER BY ABS(drift_pct) DESC;
```

### チェックサムによるドリフト検出

重要な財務系表などの場合は、スケジュール・ベースでチェックサムによるドリフト検出を実行する。

```sql
-- 重要な表の集計チェックサムを追跡
CREATE TABLE checksum_monitor (
    table_name       VARCHAR2(128),
    check_timestamp  TIMESTAMP DEFAULT SYSTIMESTAMP,
    row_count        NUMBER,
    sum_of_key_col   NUMBER,  -- 例：売上合計 SUM(total_amount)
    max_of_key_col   NUMBER,
    min_of_key_col   NUMBER
);

-- 注文表に対する財務整合性チェック
INSERT INTO checksum_monitor (table_name, row_count, sum_of_key_col, max_of_key_col, min_of_key_col)
SELECT 'ORDERS', COUNT(*), SUM(total_amount), MAX(total_amount), MIN(total_amount)
FROM orders;
COMMIT;

-- 現在の状態をベースラインと比較
SELECT
    c.check_timestamp,
    c.row_count,
    b.row_count AS baseline_row_count,
    c.sum_of_key_col,
    b.sum_of_key_col AS baseline_sum,
    c.sum_of_key_col - b.sum_of_key_col AS revenue_drift
FROM checksum_monitor c
CROSS JOIN (
    SELECT row_count, sum_of_key_col
    FROM checksum_monitor
    WHERE table_name = 'ORDERS'
    ORDER BY check_timestamp ASC
    FETCH FIRST 1 ROW ONLY
) b
WHERE c.table_name = 'ORDERS'
ORDER BY c.check_timestamp DESC;
```

---

## 制約の検証

移行後、Oracle の制約が実際に有効であり、すべてのデータがそれに適合していることを確認する。

```sql
-- Oracle の VALIDATE 機能を使用して制約違反を確認
-- 既存のデータを既存の制約に照らして再検証する
ALTER TABLE customers ENABLE VALIDATE CONSTRAINT pk_customers;
ALTER TABLE orders ENABLE VALIDATE CONSTRAINT fk_orders_customer;

-- ORA-02293 が発生した場合、データが制約を満たしていない。
-- 違反している行を特定する：
SELECT o.order_id, o.customer_id
FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM customers c WHERE c.customer_id = o.customer_id
);

-- NOT NULL 違反の確認 (正しくロードされていれば 0 になるはず)
SELECT COUNT(*) FROM customers WHERE customer_id IS NULL;
SELECT COUNT(*) FROM orders WHERE customer_id IS NULL;
```

---

## ベスト・プラクティス

1. **初日から検証を自動化する。** 検証クエリを1つのコマンドで実行できるスクリプトに組み込むこと。手動の検証はミスが発生しやすく、時間が迫ると省略されがちである。

2. **最後だけでなく段階的に検証する。** 各表のロード後に行数チェックを実行する。すべての表をロードし終えるまで問題を放置してはいけない。

3. **移行元の行数をステージング表に保持する。** 移行開始前に移行元の行数を抽出し、Oracle 内に保存しておく。これにより、移行元データベースが廃止された後でも比較のベースラインとして機能する。

4. **境界ケースのデータを明示的にテストする。** NULL 値、最大長の文字列、Unix エポック境界付近の日付、負の数、ゼロ値などを含む行をサンプル検証クエリに含めること。

5. **財務データには集計チェックサムを比較する。** 金額を含む表の場合は、移行元と Oracle で `SUM(amount)` や `SUM(quantity)` などを比較すること。行数が一致していても金額が正しいとは限らない。

6. **すべての検証失敗と解決策を記録する。** 何が失敗し、なぜ失敗し、どう解決したかのログを保存する。これは、Go/No-Go 判断を下す際の検証承諾ドキュメントとなる。

---

## よくある検証の落とし穴

**落とし穴 1 — プラットフォーム間での FLOAT/DOUBLE 精度の比較：**
IEEE 754 浮動小数点演算は、ハードウェアによってわずかに異なる結果を生むことがある。データベース間で float 列を比較する場合は、許容誤差（tolerance）を使用する。
```sql
-- 誤り：正確な一致を比較
WHERE ABS(source_val - oracle_val) = 0

-- 正解：許容誤差に基づく比較
WHERE ABS(source_val - oracle_val) > 0.00001
```

**落とし穴 2 — TIMESTAMP 比較におけるタイムゾーン：**
セッション設定によっては、UTC のタイムゾーンが Oracle でローカル時刻として表示されたり、その逆が発生したりする。異なるシステムのタイムゾーンを比較する場合は、常に `AT TIME ZONE 'UTC'` を使用すること。

**落とし穴 3 — 照合順序（Collation）の違いによる「行の欠落」：**
移行元が格文字を区別しない照合順序を使用しており、値が異なる形式で正規化されていた場合（例：'ALICE' と 'alice'）、データは存在するが、ハッシュ結果が変わってしまう。文字列のハッシュ計算では `UPPER()` による正規化を使用すること。

**落とし穴 4 — 行数は一致しているがデータが誤っている：**
行数の一致は、正しい「数」の行が存在することしか保証しない。行の順序が逆転していたり、列の値が入れ替わっていたり、データが破損している可能性がある。重要な表については、必ずハッシュ・ベースまたは集計ベースの検証で補完すること。

**落とし穴 5 — パーティション表の統計情報の遅延：**
Oracle の `ALL_TAB_PARTITIONS.NUM_ROWS` は統計情報から計算されており、リアルタイムの件数ではない。ロード直後の検証には、NUM_ROWS ではなく `COUNT(*)` を使用すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c PL/SQL Packages Reference — DBMS_CRYPTO (for hash functions)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_CRYPTO.html)
- [Oracle Database 19c Reference — DBA_TABLES (NUM_ROWS, AVG_ROW_LEN)](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_TABLES.html)
- [Oracle Database 19c SQL Language Reference — DBMS_SQLHASH](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQLHASH.html)
