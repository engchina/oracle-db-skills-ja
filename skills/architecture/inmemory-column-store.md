# Oracleインメモリー列ストア

## 概要

Oracle Database In-Memory (DBIM) オプションは、Oracle 12c Release 1 (12.1.0.2) で導入され、従来の行形式のバッファ・キャッシュに加えて、列形式のメモリー・ストアを追加する機能である。これは行ストアを置き換えるものではない。Oracleは両方の形式を同時に維持し、特定の操作に対してどちらがより効率的であるかに基づいて使い分ける。OLTPのDMLは引き続き行ストアに対して実行され、表の大部分をスキャンする分析クエリは列形式のインメモリー・ストアから読み取る。

主なメリットは分析クエリの高速化である：列形式ストレージ、ベクトル化されたSIMD（Single Instruction Multiple Data）CPU命令、およびインメモリー圧縮の組み合わせにより、行形式でディスクから同じデータを読み取る場合と比較して、全表スキャンの分析クエリを10倍から100倍高速化できる。

DBIMはOracle Databaseのオプションであり、Oracle Exadata（含まれている）やOracle Autonomous Database（含まれており自動管理される）を使用していない場合は、別途ライセンスが必要となる。

---

## 1. アーキテクチャの概要

### デュアル形式ストレージ

Oracleは、インメモリーにポピュレートされたオブジェクトに対して2つのデータ表現を維持する：

| 形式 | 場所 | 用途 |
|---|---|---|
| 行形式 | バッファ・キャッシュ / ディスク | DML (INSERT, UPDATE, DELETE)、PK/索引ルックアップ、小規模な結果セット |
| 列形式 | インメモリー列ストア (IMCS) | 全表スキャン分析クエリ、集計、範囲述語 |

IMCSは、SGAから割り当てられる独立したメモリー・プールである。`INMEMORY_SIZE` 初期化パラメータで構成され、`DB_CACHE_SIZE` とは独立して存在する。

### インメモリー圧縮ユニット (IMCU)

IMCSはデータを**IMCU (In-Memory Compression Units)** に格納する。各IMCUは：

- 1つの列セグメントからの連続した行範囲を含む
- デフォルトで約512,000行を保持する（調整可能）
- IMCUパーティションごとに1つの列を格納する（列形式レイアウト）
- IMCU内の各列に対して**最小値/最大値索引**を維持する（Exadataのストレージ索引と同様の概念）
- インメモリー圧縮形式のいずれかを使用して圧縮される

### インメモリー・ワーカー・プロセス

オブジェクトをIMCSにポピュレート（読み込み）する処理は、バックグラウンドのワーカー・プロセス（`Wnnn`）によって実行される。これらのプロセスはディスク（またはバッファ・キャッシュ）からセグメントを読み取り、非同期でIMCSにポピュレートする。ワーカー・プロセスの数は `INMEMORY_MAX_POPULATE_SERVERS` によって制御される。

---

## 2. インメモリー列ストアの構成

### DBIMの有効化

```sql
-- IMCS サイズの設定 (最小 100 MB、実用的には 1 GB 以上)
-- まだ設定されていない場合はデータベースの再起動が必要
ALTER SYSTEM SET INMEMORY_SIZE = 16G SCOPE = SPFILE;

-- 12.1 では再起動が必要。12.2 以降では動的に増やすことが可能
-- (ただし、再起動なしで減らすことはできない)
ALTER SYSTEM SET INMEMORY_SIZE = 24G;  -- 増やすだけであれば 12.2 以降は再起動不要

-- IMCS がアクティブであることを確認
SELECT name, value
FROM   v$parameter
WHERE  name IN ('inmemory_size', 'inmemory_max_populate_servers',
                'inmemory_query', 'inmemory_force')
ORDER  BY name;

-- IMCS メモリー領域の確認
SELECT pool, alloc_bytes / 1024 / 1024 / 1024 AS alloc_gb,
       used_bytes  / 1024 / 1024 / 1024 AS used_gb,
       populate_status
FROM   v$inmemory_area;
```

### インメモリー初期化パラメータ

| パラメータ | デフォルト | 説明 |
|---|---|---|
| `INMEMORY_SIZE` | 0 (無効) | IMCS のサイズ。インメモリーを有効にするために設定 |
| `INMEMORY_MAX_POPULATE_SERVERS` | CPU 数 / 2 | バックグラウンド・ポピュレート・ワーカーの最大数 |
| `INMEMORY_QUERY` | ENABLE | インスタンス・レベルでのインメモリー・クエリの有効/無効 (ENABLE / DISABLE) |
| `INMEMORY_FORCE` | DEFAULT | すべての表を強制的に IMCS に入れる (PMEM, OFF, DEFAULT) |
| `INMEMORY_VIRTUAL_COLUMNS` | MANUAL | AUTO: 仮想列もポピュレートする。MANUAL: ユーザ指定 |
| `INMEMORY_CLAUSE_DEFAULT` | (空) | すべての新規オブジェクトのデフォルト INMEMORY 属性 |
| `INMEMORY_DEEP_VECTORIZATION` | TRUE | 高度な SIMD ベクトル化を有効にする |
| `HEAT_MAP` | OFF | INMEMORY での自動データ最適化 (ADO) に必要 |

---

## 3. オブジェクトのIMCSへのポピュレート

### 表レベルの INMEMORY 句

```sql
-- 表にデフォルト設定で INMEMORY を有効化 (MEMCOMPRESS FOR QUERY LOW)
ALTER TABLE sales INMEMORY;

-- 特定の圧縮レベルを指定して有効化
ALTER TABLE sales INMEMORY MEMCOMPRESS FOR QUERY HIGH;

-- 分析ワークロード用に有効化 (高い圧縮率、許容可能なクエリ・オーバーヘッド)
ALTER TABLE sales INMEMORY MEMCOMPRESS FOR CAPACITY HIGH;

-- 優先順位（Priority）は、他のオブジェクトと比較していつポピュレートされるかを制御する
-- CRITICAL > HIGH > MEDIUM > LOW > NONE (NONE = 初回アクセス時のみポピュレート)
ALTER TABLE sales INMEMORY
    MEMCOMPRESS FOR QUERY LOW
    PRIORITY HIGH;

-- 特定の表のインメモリーを無効化
ALTER TABLE sales NO INMEMORY;
```

### インメモリー圧縮レベル

| MEMCOMPRESS レベル | 圧縮方法 | 一般的な圧縮率 | CPU コスト |
|---|---|---|---|
| `FOR DML` | 圧縮なし | 1倍 | なし |
| `FOR QUERY LOW` | LZ4 (高速解凍) | 2倍–4倍 | 極めて低い |
| `FOR QUERY HIGH` | ZLIB (高速解凍) | 4倍–8倍 | 低 |
| `FOR CAPACITY LOW` | ZLIB (より高い圧縮率) | 8倍–15倍 | 中 |
| `FOR CAPACITY HIGH` | BZIP2 相当 | 15倍–25倍 | 高 |

`FOR QUERY LOW` が本番環境で最も一般的な選択肢である。意味のある圧縮を実現しつつ、スキャン中の解凍オーバーヘッドがほぼゼロである。

### 列の選択的なポピュレート

すべての列を IMCS に入れる必要はない。頻繁に更新される列や、めったに照会されない列を除外することで、IMCS のフットプリントを削減できる。

```sql
-- 特定の列のみを IMCS に含める
ALTER TABLE sales
    INMEMORY MEMCOMPRESS FOR QUERY LOW
    (sale_date, region_id, product_id, amount, quantity)
    NO INMEMORY (customer_comments, audit_timestamp, last_updated_by);

-- 列レベルの INMEMORY 設定の確認
SELECT table_name, column_name, inmemory_compression
FROM   v$im_column_level
WHERE  table_name = 'SALES'
ORDER  BY column_name;
```

### パーティション・レベルの INMEMORY

非常に大きなパーティション表の場合、アクセス頻度の高い（Hot な）パーティションのみを IMCS にポピュレートする。

```sql
-- 今月と先月のパーティションのみに INMEMORY を適用
ALTER TABLE sales MODIFY PARTITION sales_202502 INMEMORY PRIORITY HIGH;
ALTER TABLE sales MODIFY PARTITION sales_202503 INMEMORY PRIORITY CRITICAL;

-- メモリー節約のため、古いパーティションを IMCS から除外
ALTER TABLE sales MODIFY PARTITION sales_2024q1 NO INMEMORY;
ALTER TABLE sales MODIFY PARTITION sales_2024q2 NO INMEMORY;
ALTER TABLE sales MODIFY PARTITION sales_2024q3 NO INMEMORY;
ALTER TABLE sales MODIFY PARTITION sales_2024q4 NO INMEMORY;

-- パーティション・レベルのインメモリー・ステータスの確認
SELECT table_name, partition_name, inmemory,
       inmemory_priority, inmemory_compression
FROM   dba_tab_partitions
WHERE  table_name = 'SALES'
ORDER  BY partition_position DESC
FETCH  FIRST 10 ROWS ONLY;
```

### 即時ポピュレートの強制

デフォルトでは、`PRIORITY NONE` のオブジェクトは初回アクセス時にのみポピュレートされる。他の優先順位を持つオブジェクトは、DB起動後にバックグラウンド・ワーカーによってポピュレートされる。即座にポピュレートするには：

```sql
-- 表の即時ポピュレートを強制 (完了までブロックされる)
EXEC DBMS_INMEMORY.POPULATE(schema_name => 'SCOTT', table_name => 'SALES');

-- 特定のパーティションの即時ポピュレートを強制
EXEC DBMS_INMEMORY.POPULATE(
    schema_name    => 'SCOTT',
    table_name     => 'SALES',
    subobject_name => 'SALES_202503'
);

-- 再ポピュレート (大量 DML の後にセグメントが古くなった場合に有用)
EXEC DBMS_INMEMORY.REPOPULATE(schema_name => 'SCOTT', table_name => 'SALES', force => FALSE);
```

---

## 4. V$IM_SEGMENTS による監視

`V$IM_SEGMENTS` は、インメモリー列ストアを監視するための主要なビューである。

```sql
-- 現在ポピュレートされているすべてのセグメントの表示
SELECT owner,
       segment_name,
       partition_name,
       segment_type,
       populate_status,
       bytes         / 1024 / 1024 AS disk_mb,
       inmemory_size / 1024 / 1024 AS imcs_mb,
       bytes_not_populated / 1024 / 1024 AS not_populated_mb,
       ROUND(inmemory_size / NULLIF(bytes, 0) * 100, 1) AS imcs_to_disk_pct
FROM   v$im_segments
ORDER  BY inmemory_size DESC;

-- ポピュレーション・ステータスの確認
-- STARTED: ワーカーが処理を開始
-- COMPLETED: 完全にポピュレート済み
-- FAILED: 処理中にエラーが発生
-- OUT OF MEMORY: IMCS が一杯
SELECT owner, segment_name, partition_name, populate_status
FROM   v$im_segments
WHERE  populate_status != 'COMPLETED'
ORDER  BY owner, segment_name;

-- IMCS 使用状況のサマリー
SELECT COUNT(*)                              AS populated_segments,
       SUM(bytes)         / 1024 / 1024 / 1024 AS total_disk_gb,
       SUM(inmemory_size) / 1024 / 1024 / 1024 AS total_imcs_gb,
       SUM(bytes_not_populated) / 1024 / 1024   AS not_populated_mb
FROM   v$im_segments;

-- INMEMORY 設定されているが、まだストアに未ロードのオブジェクト
SELECT owner, segment_name, partition_name, priority
FROM   v$im_segments_detail
WHERE  populate_status = 'NOT POPULATED'
  AND  priority != 'NONE'
ORDER  BY priority DESC, owner, segment_name;
```

### IMCS の追い出し (Eviction) と再ポピュレート

IMCSは、領域が一杯になるとLRU（Least Recently Used）に近い追い出しポリシーを使用する。優先順位の高いオブジェクトのために領域を空けるべく、優先順位の低いオブジェクトが追い出される。

```sql
-- IMCS 負荷の監視
SELECT pool,
       alloc_bytes / 1024 / 1024 / 1024 AS alloc_gb,
       used_bytes  / 1024 / 1024 / 1024 AS used_gb,
       ROUND(used_bytes / NULLIF(alloc_bytes, 0) * 100, 1) AS pct_used
FROM   v$inmemory_area;

-- 追い出しの確認
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'IM populate segments requested',
    'IM populate segments completed',
    'IM repopulate (trickle) total',
    'IM repopulate (trickle) failed',
    'IM segments evicted',
    'IM scans'
)
ORDER  BY name;
```

---

## 5. 分析クエリの高速化

インメモリー列ストアは、以下のようなパターンのクエリを劇的に高速化する：

### 全表スキャンを伴う集計

```sql
-- このクエリは IMCS の列形式スキャンの恩恵を直接受ける
-- Oracle は必要な列 (region_id, sale_date, amount) のみを読み取り、
-- 連続したメモリー上で SIMD 命令を使用して処理する
SELECT region_id,
       TRUNC(sale_date, 'MONTH') AS sale_month,
       COUNT(*)                  AS num_sales,
       SUM(amount)               AS total_amount,
       AVG(amount)               AS avg_amount,
       MAX(amount)               AS max_amount
FROM   sales
WHERE  sale_date >= DATE '2025-01-01'
GROUP  BY region_id, TRUNC(sale_date, 'MONTH')
ORDER  BY sale_month, region_id;

-- 実行計画で IMCS が使用されたことを確認
EXPLAIN PLAN FOR
SELECT region_id, SUM(amount) FROM sales GROUP BY region_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- キーワード: TABLE ACCESS INMEMORY FULL
-- また: VECTOR GROUP BY (インメモリー集計) を探す
```

### 実行計画におけるインメモリー・アクセスの検証

```sql
-- セッション・レベル: IMCS が使用されたことの確認
ALTER SESSION SET STATISTICS_LEVEL = ALL;

SELECT region_id, SUM(amount) FROM sales GROUP BY region_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT => 'ALLSTATS LAST'));
-- 実行計画の主要インジケータ:
-- Operation "TABLE ACCESS INMEMORY FULL" = IMCS スキャン
-- Operation "VECTOR GROUP BY"            = インメモリー集計
-- 下部の Note 欄に "In-Memory Scan" 
```

### IMCS による結合の高速化

インメモリー列ストアは、IMCS 内部でブルーム・フィルタを作成および適用することで、ハッシュ結合も高速化する。これは12.2で導入された**インメモリー結合グループ**機能である。

```sql
-- 結合グループは結合列のハッシュ値を事前に計算することで、
-- 行を実体化させることなく IMCS 内部での結合条件の評価を可能にする

-- 頻繁に結合される列ペアに結合グループを作成
CREATE INMEMORY JOIN GROUP jg_sales_products
    (sales(product_id), products(product_id));

-- 結合グループの確認
SELECT join_group_name, table_name, column_name
FROM   dba_joingroups
ORDER  BY join_group_name, table_name;

-- product_id で結合するクエリは、IMCS レベルでの結合処理の恩恵を受ける
SELECT p.product_name,
       SUM(s.amount) AS total_revenue
FROM   sales s
JOIN   products p ON s.product_id = p.product_id
WHERE  s.sale_date >= DATE '2025-01-01'
GROUP  BY p.product_name
ORDER  BY total_revenue DESC;
```

---

## 6. インメモリー集計 (Vector Group By)

インメモリー集計 (IMA: In-Memory Aggregation) は、GROUP BY 操作を IMCU データに対する SIMD ベクトル操作として実行する 12c 以降の機能である。Oracle は行を1つずつ処理する代わりに、ベクトル化された CPU 命令を使用して IMCU 全体の集計を溜め込む。

### ベクトル・グループ・バイの仕組み

1. IMCS スキャンが IMCU から一連の列ベクトルを生成する
2. Oracle は IMCU ごとにベクトル化されたパスで GROUP BY を適用し、IMCU ごとの部分集計を生成する
3. 最終集計で、すべての IMCU からの部分的な結果をマージする

これは行データに対する従来のハッシュ集計よりも桁違いに高速である。理由は以下の通り：
- 行の再構成が不要（必要な列のみが読み取られる）
- SIMD: 1つの CPU 命令で 8 つ以上の値を同時に処理する
- キャッシュ・フレンドリー: 連続したメモリー・アクセス・パターン

```sql
-- ベクトル・グループ・バイが使用されているかの確認
SELECT operation, options, object_name, cardinality, bytes, cost
FROM   plan_table
WHERE  operation IN ('VECTOR GROUP BY', 'HASH GROUP BY', 'TABLE ACCESS')
ORDER  BY id;

-- インメモリー集計の有効化/無効化
ALTER SESSION SET "_inmemory_vector_aggregation" = TRUE;  -- デフォルト

-- VECTOR_TRANSFORM ヒントによる強制
SELECT /*+ VECTOR_TRANSFORM */
       sale_year, region_id, SUM(amount)
FROM   (SELECT EXTRACT(YEAR FROM sale_date) AS sale_year,
               region_id, amount
        FROM   sales)
GROUP  BY sale_year, region_id;
```

---

## 7. IMCS のサイジングと容量計画

### IMCS の必要量の見積もり

```sql
-- INMEMORY を有効化する前に表の IMCS サイズを見積もる
-- シミュレーションを実行して投影サイズを取得する
SELECT
    table_name,
    SUM(bytes) / 1024 / 1024 AS on_disk_mb,
    -- 経験則: IMCS サイズ = ディスクサイズ * 圧縮率
    -- FOR QUERY LOW: ディスクサイズの約 0.3-0.5 倍 (圧縮 + 列指向)
    ROUND(SUM(bytes) / 1024 / 1024 * 0.4, 0) AS estimated_imcs_mb_low,
    ROUND(SUM(bytes) / 1024 / 1024 * 0.2, 0) AS estimated_imcs_mb_high
FROM   dba_segments
WHERE  owner = 'SCOTT'
  AND  segment_name IN ('SALES', 'ORDERS', 'PRODUCTS')
GROUP  BY table_name;

-- 実際にポピュレートされた後の IMCS サイズの測定
SELECT SUM(inmemory_size) / 1024 / 1024 / 1024 AS total_imcs_gb
FROM   v$im_segments;

-- IMCS の割り当て量に対する使用量を確認
SELECT pool,
       alloc_bytes / 1024 / 1024 / 1024 AS alloc_gb,
       used_bytes  / 1024 / 1024 / 1024 AS used_gb
FROM   v$inmemory_area;
```

### IMCS サイジングのガイドライン

| シナリオ | サイジングの推奨事項 |
|---|---|
| すべての分析表を IMCS に | (表のディスクサイズ * 圧縮係数) の合計 + 20% のオーバーヘッド |
| Hot パーティションのみ | 今月 + 前月分などのパーティション合計サイズ |
| OLTP と分析の混在 | アクティブな分析データセットの 25–50%。優先順位で追い出しを管理 |
| Autonomous Database (ATP) | インメモリーは自動管理される。ユーザ操作は不要 |

### 19c 以降の自動インメモリー (AIM)

Oracle 19c で導入された自動インメモリー (AIM) は、データベースのヒート・マップ（Heat Map）を使用して、実際のアクセス・パターンに基づいて IMCS からのオブジェクトのポピュレートと追い出しを自動的に行う。

```sql
-- ヒート・マップの有効化 (AIM の前提条件)
ALTER SYSTEM SET HEAT_MAP = ON SCOPE = BOTH;

-- 自動インメモリー管理の有効化
ALTER SYSTEM SET INMEMORY_AUTOMATIC_LEVEL = MEDIUM SCOPE = BOTH;
-- レベル: OFF (手動), LOW (明示的な INMEMORY オブジェクトのみ管理),
--         MEDIUM (熱量に基づいてポピュレート/追い出しを行う),
--         HIGH (INMEMORY 設定がないオブジェクトも自動的に対象とする)

-- AIM 動作の確認
SELECT object_name, action_taken, timestamp, reason
FROM   v$im_adg_log
ORDER  BY timestamp DESC
FETCH  FIRST 50 ROWS ONLY;

-- ヒート・マップ・データの確認
SELECT object_name, owner, track_time,
       full_scan, lookup_scan, n_full_scans
FROM   v$heat_map_segment
ORDER  BY full_scan DESC
FETCH  FIRST 20 ROWS ONLY;
```

---

## 8. ベスト・プラクティス

- **スキャン頻度が最も高く、列の選択性が最も高い表から開始する。** IMCS は、クエリが多くの列のうちの一部のみにアクセスするような「横に長い（列数が多い）」表で最大の効果を発揮する。5列しかないような「細い」表ですべての列にアクセスする場合は、メリットが少ない。
- **SLAにとって重要な表には `PRIORITY CRITICAL` を使用する。** データベース起動時、ポピュレート・ワーカーは優先順位の高い順に IMCS を埋める。CRITICAL オブジェクトが最初にポピュレートされ、ユーザの負荷がかかる前に準備が整うようになる。
- **大規模な表ではパーティショニングと組み合わせる。** 時間ベースでパーティション化し、Hot なパーティション（今月、過去3ヶ月分など）のみを IMCS に保持する。これにより IMCS のフットプリントを予測可能にし、追い出しのプレッシャーを避けることができる。
- **UPDATE アクティビティが激しい表には INMEMORY を有効にしない。** IMCS は IMCU ジャーナルを介して行ストアとの整合性を最終的に保つ。DML 発生率が非常に高いと、IMCU ジャーナルが一杯になり、再ポピュレートが頻発してワーカーのリソースを消費する。DML が低～中程度の表に使用すること。
- **結合グループを積極的に活用する。** 分析クエリの 50% 以上で同じ2つの表が結合される場合、結合グループによってハッシュ値を事前計算することで、DBサーバーへの行転送を伴わない IMCS レベルでの結合処理が可能になる。
- **`V$IM_SEGMENTS` の `bytes_not_populated` を監視する。** この値が 0 でない場合、IMCS の容量が不足してセグメントを完全にポピュレートできなかったことを意味する。ポピュレートされていないエクステントに対するクエリはディスク読み取りにフォールバックする。`INMEMORY_SIZE` を増やすか、インメモリー対象のオブジェクトを減らす必要がある。

---

## 9. よくある間違いとその回避方法

### 間違い 1: オブジェクト追加後に INMEMORY_SIZE を増やさない

`INMEMORY_SIZE` を増やさずに多くの表に `INMEMORY` を設定すると、IMCS が一杯になる。新しいオブジェクトはポピュレートに失敗し（または既存のものが追い出され）、クエリは通知なしにディスク読み取りにフォールバックする（`populate_status = 'OUT OF MEMORY'` フラグのみが原因を示す）。

```sql
-- OUT OF MEMORY による未ポピュレートの確認
SELECT owner, segment_name, partition_name, populate_status
FROM   v$im_segments
WHERE  populate_status = 'OUT OF MEMORY';

-- 必要に応じて INMEMORY_SIZE を動的に増やす (12.2 以降、増やす方向のみ)
ALTER SYSTEM SET INMEMORY_SIZE = 32G;
```

### 間違い 2: パラレル・クエリを有効にせずに INMEMORY を使用する

非常に大きな表の場合、IMCS スキャンもパラレル・クエリ（`PARALLEL` ヒントや表の `PARALLEL` 句）の恩恵を受ける。パラレル・クエリがない場合、100GB の表に対するシリアルな IMCS スキャンは依然としてシングルスレッドの操作となる。IMCS と適切な並列度 (DOP) を組み合わせて使用すること。

```sql
-- インメモリー・スキャンのための表レベル並列度の設定
ALTER TABLE sales PARALLEL 8;

-- またはクエリ・レベルのヒントを使用
SELECT /*+ PARALLEL(s, 8) */
       region_id, SUM(amount)
FROM   sales s
GROUP  BY region_id;
```

### 間違い 3: IMCS が同期的に更新されると思い込む

IMCS はすべての DML 操作に対してリアルタイムに更新されるわけではない。Oracle は IMCU ジャーナリングを使用しており、DML による変更は IMCU 内の小さなジャーナル領域に記録され、定期的な「トリクル（少量ずつ）・再ポピュレート」によってジャーナルが IMCU にマージされる。クエリはジャーナルのマージを通じて最新（コミット済み）のデータを見るが、DML の同時実行性が極めて高い表では、わずかな CPU コストが追加で発生する。

厳密な変更追跡が必要な場合は、IMCS オブジェクトに対して `SELECT ... AS OF TIMESTAMP` を使用せず、行ストアのクエリ・パスを使用するようにすること。

### 間違い 4: 頻繁に照会される表に CAPACITY 圧縮を使用する

`MEMCOMPRESS FOR CAPACITY HIGH` は最大の圧縮率を実現するが、解凍アルゴリズムが遅い。1時間に数百回もスキャンされる表では、解凍にかかる CPU コストが無視できなくなる。Hot な分析用の表には `FOR QUERY LOW` または `FOR QUERY HIGH` を使用すること。

### 間違い 5: 大量データ・ロード後の再ポピュレートを忘れる

大規模な `INSERT /*+ APPEND */` やパーティション交換操作の直後、そのセグメントの IMCS コピーは古くなっている。Oracle はバックグラウンドで再ポピュレートを行うが、それが完了するまでの間、新しいデータに対するクエリはディスクから読み取られる。時間の制約が厳しいワークロードでは、ロード直後に明示的な再ポピュレートをトリガーすること。

```sql
-- 夜間バッチ完了後、明示的に再ポピュレートする
EXEC DBMS_INMEMORY.REPOPULATE(
    schema_name    => 'DW_OWNER',
    table_name     => 'FACT_SALES',
    subobject_name => 'SALES_202503'
);

-- 完了の監視
SELECT segment_name, partition_name, populate_status, bytes_not_populated
FROM   v$im_segments
WHERE  segment_name = 'FACT_SALES'
  AND  partition_name = 'SALES_202503';
```

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database In-Memory Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/inmem/) — IMCS アーキテクチャ、MEMCOMPRESS レベル、ポピュレーション、監視
- [DBMS_INMEMORY (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_INMEMORY.html) — POPULATE および REPOPULATE プロシージャ (パラメータは `subobject_name` であり `partition_name` ではない)
- [DBMS_INMEMORY_ADMIN (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_INMEMORY_ADMIN.html) — AIM パラメータ、FastStart, IME_CAPTURE_EXPRESSIONS
- [Oracle Database Reference 19c — V$IM_SEGMENTS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-IM_SEGMENTS.html) — インメモリー・セグメント監視ビュー
