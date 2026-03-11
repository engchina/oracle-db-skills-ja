# Oracle Exadataの機能: スマート・スキャン、HCC、ストレージ・オフロード

## 概要

Oracle Exadataは、Oracle Databaseサーバーと、Exadata Storage Server Software (CELLSRV) を実行するインテリジェントなストレージ・セルを組み合わせた垂直統合型システム（Engineered System）である。その根本的な設計原理は、SQL処理（フィルタリング、結合、解凍など）をストレージ・レイヤーに移動（オフロード）させることで、ストレージ・ネットワークを経由してデータベース・サーバーに転送されるデータ量を劇的に削減することにある。

Exadataは、物理的なオンプレミス・システム（X9M、X10M）、OCI上のExadata Cloud Service (ExaCS)、およびExadata Cloud at Customer (ExaDB-C@C) として利用可能である。このガイドで説明するExadata固有の機能は、すべてのデプロイメント・モデルで利用できる。

---

## 1. スマート・スキャン (セル・オフロード処理)

スマート・スキャン（Smart Scan）は、Exadataの基幹となる機能である。Oracleがクエリにおいてセル側での処理が有益であると判断すると、述語（WHERE句）、プロジェクション（選択列）、およびオプションの結合フィルタ情報を各ストレージ・セルに送信する。各セルはローカル・ディスク上のデータを評価し、述語を満たす行のみを返す。これを**セル・オフロード**と呼ぶ。

### スマート・スキャンの仕組み

スマート・スキャンがない場合：
1. ストレージ・セルがディスクからすべてのブロックを読み取る
2. すべてのブロックがInfiniBand経由でDBサーバーに転送される
3. DBサーバーがWHERE句を評価し、ほとんどの行を破棄する
4. DBサーバーが少量の結果セットをセッションに返す

スマート・スキャンがある場合：
1. ストレージ・セルがディスクからすべてのブロックを読み取る
2. セルがCELLSRVプロセス内でWHERE句、プロジェクション、およびブルーム・フィルタを評価する
3. 一致する行（または行の一部）のみがInfiniBand経由で転送される
4. DBサーバーが必要に応じて残りの集計処理などを行う

データ削減率（送信バイト数 vs ディスク上のバイト数）は、**セル・オフロード効率**と呼ばれる。オフロード効率が90%であれば、データの90%はストレージ・レイヤーから出なかったことを意味する。

### スマート・スキャンの前提条件

スマート・スキャンは、以下の条件がすべて満たされた場合にのみ有効になる：

1. **フル・セグメント・スキャン**: 操作が全表スキャン（Full Table Scan）または索引高速全スキャン（Index Fast Full Scan）であること（ROWIDによるルックアップや、選択的な範囲のスキャンではないこと）。
2. **ダイレクト・パス・リード**: 操作がバッファ・キャッシュをバイパスし、ストレージから直接読み取ること。これは、セグメントがダイレクト・読み取りをトリガーするのに十分な大きさであるか、セッションで `/*+ PARALLEL */` または `/*+ FULL */` ヒントを使用している場合に発生する。
3. **Exadataストレージ**: セグメントのデータファイルが、Exadataストレージ・セルによってバックアップされたASMディスク・グループ上に存在すること。

```sql
-- 表のスマート・スキャン適格性の確認
SELECT table_name, blocks, num_rows,
       ROUND(blocks * 8192 / 1024 / 1024, 1) AS size_mb
FROM   user_tables
WHERE  table_name = 'SALES';

-- 表はダイレクト・パス・リードをトリガーするのに十分な大きさである必要がある
-- 閾値はおおよそ: セグメントサイズ > _small_table_threshold * db_block_size
SELECT name, value
FROM   v$parameter
WHERE  name = '_small_table_threshold';

-- テスト目的でスマート・スキャンを強制する（本番環境では使用しないこと）
ALTER SESSION SET "_serial_direct_read" = always;
```

### スマート・スキャン・オフロードの監視

```sql
-- システム・レベルのスマート・スキャン統計
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell physical IO interconnect bytes',
    'cell physical IO interconnect bytes returned by smart scan',
    'cell scans',
    'cell blocks processed by cache layer',
    'cell blocks processed by data layer',
    'cell blocks processed by txn layer'
)
ORDER  BY name;

-- オフロード効率（フィルタリングされたバイトの割合）の計算
SELECT ROUND(
    (1 - (bytes_returned / NULLIF(bytes_read, 0))) * 100,
    2
) AS offload_efficiency_pct
FROM (
    SELECT
        SUM(CASE WHEN name = 'cell physical IO interconnect bytes'
                 THEN value ELSE 0 END) AS bytes_returned,
        SUM(CASE WHEN name = 'cell physical IO interconnect bytes returned by smart scan'
                 THEN value ELSE 0 END) AS bytes_scan_returned
    FROM v$sysstat
    WHERE name IN (
        'cell physical IO interconnect bytes',
        'cell physical IO interconnect bytes returned by smart scan'
    )
);

-- セッション・レベル: 現在のセッションでどれだけオフロードされたか
SELECT name, value
FROM   v$mystat m JOIN v$statname n ON m.statistic# = n.statistic#
WHERE  n.name LIKE 'cell%'
ORDER  BY value DESC;

-- SQLレベル: AWR から cell_offload_efficiency を確認
SELECT sql_id,
       ROUND(AVG(cell_offload_efficiency), 2) AS avg_offload_pct,
       SUM(executions) AS total_execs,
       ROUND(SUM(elapsed_time) / 1e6, 1) AS total_elapsed_sec
FROM   dba_hist_sqlstat
WHERE  cell_offload_efficiency > 0
  AND  snap_id >= (SELECT MAX(snap_id) - 24 FROM dba_hist_snapshot)
GROUP  BY sql_id
ORDER  BY total_elapsed_sec DESC
FETCH  FIRST 20 ROWS ONLY;
```

---

## 2. ストレージ索引

ストレージ索引（Storage Index）は、完全なインメモリの最適化機能であり、CELLSRVによって自動的に維持される。ディスクの1MBのリージョンごとに、ストレージ索引は最大8つの列の最小値と最大値を追跡する。クエリにWHERE句の述語がある場合、ストレージ・セルは各1MBリージョンに一致する行が含まれる可能性があるかどうかを確認する。含まれない場合、そのリージョン全体のディスクI/Oはスキップされる。

ストレージ索引の特徴：
- **自動**: DDLや設定は不要
- **インメモリのみ**: ディスク上ではなく、ストレージ・セルのメモリー（DRAM）に格納される
- **セル再起動で消失**: データがアクセスされるにつれて、時間の経過とともに再構築される

### ストレージ索引が最も効果的な場合

ストレージ索引は以下の場合に最も効果的である：
- 述語となる列によってデータが**自然にクラスタリング**されている場合（例：時系列で挿入される注文表の `ORDER_DATE`）
- クエリが**カーディナリティ（選択性）の高い範囲述語**（`WHERE order_date BETWEEN ... AND ...`）でフィルタリングする場合
- 表がストレージ・セル間で**ランダムに分散されていない**場合

```sql
-- ストレージ索引統計 (各ストレージ・セル、CellCLI または V$ ビューから取得)
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell blocks helped by minmax predicate',   -- ストレージ索引によって削減されたブロック
    'cell blocks helped by bloom filter'        -- ブルーム・フィルタ結合によって削減されたブロック
)
ORDER  BY name;

-- ストレージ索引によるクラスタリングの恩恵を受けるクエリの特定
-- 選択的な日付/数値述語を持つ全表スキャンを探す
SELECT sql_id, sql_text, executions,
       rows_processed / NULLIF(executions, 0) AS avg_rows_per_exec,
       buffer_gets / NULLIF(executions, 0) AS avg_bufgets_per_exec
FROM   v$sql
WHERE  sql_text LIKE '%ORDER_DATE%'
  AND  operation_type = 'TABLE ACCESS FULL'
  AND  executions > 10
ORDER  BY buffer_gets DESC
FETCH  FIRST 10 ROWS ONLY;
```

---

## 3. ハイブリッド列圧縮 (HCC)

HCC（Hybrid Columnar Compression）は、Exadata固有の圧縮テクノロジーであり（OCI Object StorageやZFS Storage Applianceでも利用可能）、複数の行の列値を**圧縮ユニット**にグループ化し、まとめて圧縮する。列データはカーディナリティが低い（同じ値が繰り返される）傾向があるため、HCCは通常、行ベースの圧縮よりも10倍から50倍優れた圧縮率を実現する。

### HCCの圧縮レベル

| 圧縮レベル | 対象ユースケース | 一般的な圧縮率 | クエリ・オーバーヘッド |
|---|---|---|---|
| `QUERY LOW` | アクティブに照会される表 | 6倍–10倍 | 低 |
| `QUERY HIGH` | アクティブに照会される表 | 10倍–15倍 | 低～中 |
| `ARCHIVE LOW` | 照会頻度の低いアーカイブ | 15倍–25倍 | 中 |
| `ARCHIVE HIGH` | コールド・アーカイブ | 25倍–50倍 | 高 |

`QUERY LOW` および `QUERY HIGH` は、定期的に照会される表に適している。`ARCHIVE` レベルは、一度ロードされたらめったに照会されないデータ用である。

### HCCの適用

```sql
-- HCC 圧縮を使用した表の作成
CREATE TABLE sales_archive (
    sale_id       NUMBER        NOT NULL,
    sale_date     DATE          NOT NULL,
    product_id    NUMBER        NOT NULL,
    customer_id   NUMBER        NOT NULL,
    region_id     NUMBER(2)     NOT NULL,
    amount        NUMBER(12,2)  NOT NULL
)
COMPRESS FOR QUERY HIGH;

-- 既存の表を変更して HCC を使用する
-- 注意: これは ALTER 以降にロードされた新しいデータにのみ適用される
ALTER TABLE sales_history COMPRESS FOR QUERY HIGH;

-- 既存の行を再圧縮するには、表を移動 (MOVE) する
ALTER TABLE sales_history MOVE COMPRESS FOR QUERY HIGH ONLINE;

-- または CTAS (Create Table As Select) を使用して完全に再圧縮する
CREATE TABLE sales_history_new
COMPRESS FOR QUERY HIGH
AS SELECT * FROM sales_history;

-- 個々の表パーティションの圧縮設定の確認
SELECT table_name, partition_name, compression, compress_for,
       blocks, num_rows
FROM   user_tab_partitions
WHERE  table_name = 'SALES_HISTORY'
ORDER  BY partition_position;
```

### HCC と DML

HCCは、DML操作に対して重要な制限がある：

- **INSERT ... AS SELECT (ダイレクト・パス)**: 完全にサポートされる。HCC圧縮ユニットを作成する。
- **INSERT (従来型)**: 行は「OLTP圧縮」形式の小さな領域に格納され、HCC形式にはならない。
- **UPDATE**: 更新された行はHCC圧縮ユニットからOLTP形式に移行される（行移行）。
- **DELETE**: 削除された行は圧縮ユニット内に「穴」を残す。行移行は発生しないが、領域が無駄になる。

このため、HCCはアペンド・オンリー（追記型）またはアーカイブ用途の表に適している。読み取り/書き込みが混在するワークロードには、HCCではなく「アドバンスト行圧縮（Advanced Row Compression）」を使用すべきである。

```sql
-- 表の HCC 圧縮ユニットの確認 (SYSDBA または特定の権限が必要)
SELECT table_name, compression, compress_for,
       ROUND(blocks * 8192 / 1024 / 1024, 1) AS size_mb,
       num_rows
FROM   dba_tables
WHERE  table_name = 'SALES_ARCHIVE';

-- HCC 表での更新によって発生した行移行の確認
SELECT table_name, chain_cnt, num_rows,
       ROUND(chain_cnt / NULLIF(num_rows, 0) * 100, 2) AS migration_pct
FROM   dba_tables
WHERE  compress_for IN ('QUERY LOW', 'QUERY HIGH', 'ARCHIVE LOW', 'ARCHIVE HIGH')
  AND  chain_cnt > 0;
```

---

## 4. I/Oリソース・マネージャ (IORM)

IORM（I/O Resource Manager）は、Exadataインフラを共有するデータベース、サービス、およびコンシューマ・グループ間のI/O帯域幅の割り当てを制御するストレージ・セル側のリソース・マネージャである。DBサーバーで動作するCPUベースのDBRM（Database Resource Manager）とは異なり、IORMはCELLSRV内部の物理ディスク・レベルでI/O制限を適用する。

### IORMプラン

IORMプランは、以下の3つのレベルで構成される：
1. **データベース間プラン (Inter-database plan)**: 同じExadata上の異なるデータベース間でI/Oシェアを割り当てる。
2. **データベース内プラン (Intra-database plan)**: 単一のデータベース内のコンシューマ・グループ間でI/Oシェアを割り当てる。
3. **制限付きデータベース・プラン (Database plan with limits)**: 特定のデータベースが消費できるI/O帯域幅に上限を設ける。

```sql
-- 現在の IORM 設定の確認 (ストレージ・セル上の CellCLI から)
-- CellCLI> LIST IORMPLAN DETAIL

-- SQL*Plus から IORM を構成する (SYSDBA 権限が必要、12c以降)
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();

    -- データベースの I/O プランを作成
    DBMS_RESOURCE_MANAGER.CREATE_PLAN(
        plan    => 'EXADATA_IO_PLAN',
        comment => 'Exadata I/O resource plan'
    );

    -- コンシューマ・グループに I/O シェアを割り当てる
    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan             => 'EXADATA_IO_PLAN',
        group_or_subplan => 'OLTP_GROUP',
        mgmt_p1          => 70,
        limit_io_megabytes_per_second => 5000  -- MB/s 上限
    );

    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan             => 'EXADATA_IO_PLAN',
        group_or_subplan => 'BATCH_GROUP',
        mgmt_p2          => 30,
        limit_io_megabytes_per_second => 2000
    );

    DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/

-- プランを有効化
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'EXADATA_IO_PLAN' SCOPE = BOTH;

-- コンシューマ・グループごとの I/O スループットの監視
SELECT consumer_group_name,
       io_requests,
       io_service_time,
       io_service_waits
FROM   v$rsrc_consumer_group
ORDER  BY io_requests DESC;
```

---

## 5. Exadata固有のSQLヒント

いくつかのSQLヒントは、Exadataのオフロード機能と直接相互作用する。

### 主要なヒント

| ヒント | 効果 |
|---|---|
| `/*+ FULL(table) */` | 全表スキャンを強制し、スマート・スキャンを有効にする |
| `/*+ PARALLEL(table, degree) */` | パラレル・クエリを有効にし、ダイレクト・パス・リードとスマート・スキャンを強制する |
| `/*+ NO_PARALLEL(table) */` | 表のパラレル・クエリを無効にする |
| `/*+ CELL_FLASH_CACHE(KEEP) */` | オブジェクトを Exadata スマート・フラッシュ・キャッシュにピン留めする |
| `/*+ CELL_FLASH_CACHE(NONE) */` | オブジェクトがスマート・フラッシュ・キャッシュを消費しないようにする |
| `/*+ RESULT_CACHE */` | 結果を SQL 結果キャッシュにキャッシュする (繰り返されるスマート・スキャンのオーバーヘッドを削減) |
| `/*+ CACHE(table) */` | ブロックをデータベース・バッファ・キャッシュにキャッシュする (ダイレクト・パスを無効にする) |
| `/*+ NO_CACHE(table) */` | バッファ・キャッシュへのキャッシュを防止する (ダイレクト・パスとスマート・スキャンを維持) |
| `/*+ VECTOR_TRANSFORM */` | インメモリー集計用のベクトル型変換を有効にする (スマート・スキャンを補完) |

```sql
-- パラレル・ヒントを使用してスマート・スキャンを強制する
SELECT /*+ FULL(s) PARALLEL(s, 8) */
       region_id,
       COUNT(*),
       SUM(amount)
FROM   sales s
WHERE  sale_date >= DATE '2025-01-01'
  AND  sale_date <  DATE '2026-01-01'
GROUP  BY region_id;

-- アクセス頻度の高いルックアップ表をスマート・フラッシュ・キャッシュにピン留めし、
-- 大規模なスマート・スキャン操作によって追い出されないようにする
ALTER TABLE country_codes STORAGE (CELL_FLASH_CACHE KEEP);

-- 非常に大きなコールド（アクセス頻度の低い）表がスマート・フラッシュ・キャッシュを占有しないようにする
ALTER TABLE sales_archive_2010 STORAGE (CELL_FLASH_CACHE NONE);

-- スマート・フラッシュ・キャッシュ効率の確認
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell flash cache read hits',
    'physical read requests optimized'
)
ORDER  BY name;
```

---

## 6. Exadataオフロード効率の監視

### Exadata監視用の主要なV$ビュー

```sql
-- SQL 文ごとの全体的なセル・オフロード効率 (現在)
SELECT sql_id,
       child_number,
       io_cell_offload_eligible_bytes / 1024 / 1024 AS eligible_mb,
       io_cell_offload_returned_bytes  / 1024 / 1024 AS returned_mb,
       ROUND(
           (1 - io_cell_offload_returned_bytes /
                 NULLIF(io_cell_offload_eligible_bytes, 0)) * 100,
           2
       ) AS offload_efficiency_pct,
       io_cell_uncompressed_bytes / 1024 / 1024 AS uncompressed_mb
FROM   v$sql
WHERE  io_cell_offload_eligible_bytes > 0
ORDER  BY io_cell_offload_eligible_bytes DESC
FETCH  FIRST 20 ROWS ONLY;

-- AWR からの過去のオフロード効率
SELECT snap_id,
       sql_id,
       ROUND(AVG(cell_offload_efficiency), 2) AS avg_offload_pct,
       SUM(io_cell_offload_eligible_bytes) / 1024 / 1024 / 1024 AS eligible_gb,
       SUM(io_cell_offload_returned_bytes) / 1024 / 1024 / 1024 AS returned_gb
FROM   dba_hist_sqlstat
WHERE  cell_offload_efficiency > 0
  AND  snap_id >= (SELECT MAX(snap_id) - 24 FROM dba_hist_snapshot)
GROUP  BY snap_id, sql_id
ORDER  BY eligible_gb DESC
FETCH  FIRST 20 ROWS ONLY;

-- セル・メトリック統計 (Exadata ストレージ・セルのパフォーマンス)
SELECT metric_name,
       AVG(average_value) AS avg_value,
       MAX(maximum_value) AS max_value,
       MIN(minimum_value) AS min_value
FROM   v$cell_metric_desc cmd
JOIN   v$cell_metric      cm ON cmd.metric_id = cm.metric_id
GROUP  BY metric_name
ORDER  BY metric_name;

-- スマート・フラッシュ・キャッシュ・ヒット率の確認
SELECT name, value
FROM   v$sysstat
WHERE  name IN (
    'cell flash cache read hits',
    'physical read total IO requests',
    'physical read requests optimized'
)
ORDER  BY name;

-- スマート・フラッシュ・キャッシュ・ヒット率 (%) の計算
SELECT ROUND(
    SUM(CASE WHEN name = 'cell flash cache read hits' THEN value END) /
    NULLIF(SUM(CASE WHEN name = 'physical read total IO requests' THEN value END), 0) * 100,
    2
) AS flash_cache_hit_pct
FROM   v$sysstat
WHERE  name IN ('cell flash cache read hits', 'physical read total IO requests');
```

### 実行計画 (EXPLAIN PLAN) と Exadata オフロードの表示

```sql
-- オプティマイザがセル・オフロードの使用を計画しているかどうかの確認
EXPLAIN PLAN FOR
SELECT region_id, COUNT(*), SUM(amount)
FROM   sales
WHERE  sale_date >= DATE '2025-01-01'
GROUP  BY region_id;

-- "TABLE ACCESS FULL" かつ "Batched Disk Reads" がある場合、ダイレクト・パスを意味する
-- 実行計画内の重要なフレーズは "Batched Disk Reads" や "storage" 述語のオフロード表示である
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => '+PREDICATE'));

-- 実行計画にオフロードされる述語を表示するようにセッションを設定
ALTER SESSION SET "_cell_offload_plan_display" = ALWAYS;

EXPLAIN PLAN FOR
SELECT region_id, SUM(amount)
FROM   sales
WHERE  sale_date >= DATE '2025-01-01'
  AND  amount > 1000
GROUP  BY region_id;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- 実行計画には、どの述語と列がセルにオフロードされているかを示す
-- "Storage Filter" および "Storage Project" 注釈が表示される
```

---

## 7. セル・オフロード・メトリック・リファレンス

| 統計名 | 説明 |
|---|---|
| `cell physical IO interconnect bytes` | ストレージ・セルから DB サーバーに送信された合計バイト数 |
| `cell physical IO interconnect bytes returned by smart scan` | スマート・スキャン操作によって具体的に返されたバイト数 |
| `cell scans` | 開始されたスマート・スキャン操作の数 |
| `cell blocks processed by cache layer` | セルのキャッシュ・レイヤーによって処理されたブロック数 |
| `cell blocks helped by minmax predicate` | ストレージ索引によって削減されたブロック数 |
| `cell blocks helped by bloom filter` | ブルーム・フィルタ結合のオフロードによって削減されたブロック数 |
| `cell flash cache read hits` | HDD アクセスなしでスマート・フラッシュ・キャッシュから提供された読み取り回数 |
| `physical read requests optimized` | スマート・フラッシュ・キャッシュによって提供された物理読み取り要求 |
| `IO megabytes read total` | Exadata ストレージ・セルから読み取られた合計 MB |
| `IO megabytes written total` | Exadata ストレージ・セルに書き込まれた合計 MB |

---

## 8. ベスト・プラクティス

- **ダイレクト・パス INSERT を使用して HCC 表にデータをロードする。** 従来型の INSERT は行形式で行を書き込むため、HCC が無効化される。大規模なロードには常に `INSERT /*+ APPEND */` または `INSERT ... AS SELECT` を使用すること。
- **プライベート InfiniBand ネットワークのサイズを適切に設定する。** Exadata の InfiniBand（X9M では RoCE）ネットワークは、スマート・スキャンの結果と（RAC の）キャッシュ・フュージョンのトラフィックの両方を運ぶ。`V$CELL_METRIC` で飽和状態を監視すること。
- **IORM を使用して、大規模な分析クエリから OLTP のレイテンシを保護する。** IORM がないと、すべてのセル I/O 帯域幅を消費するパラレル・スマート・スキャンが発生した場合、OLTP クエリがその背後で待機することになる。
- **Exadata では統計を常に最新の状態に保つ。** CBO（コストベース・オプティマイザ）がスマート・スキャンをトリガーする実行計画を生成するには、全表スキャンが適切であることを認識していなければならない。古い統計により、スマート・スキャンをバイパスする索引ルックアップが選択される可能性がある。
- **巨大な Exadata 表に対して索引の使用を強制しない。** 索引範囲スキャンは小さな I/O を使用し、スマート・スキャンをバイパスする。Exadata では、5%〜10% を超える行にアクセスする範囲述語の場合、スマート・スキャンを伴うフル・スキャンの方が選択的な索引スキャンよりも性能が良いことが多い。
- **毎日ロードされるレポート用表には `QUERY HIGH` を使用し、決して更新されないデータにのみ `ARCHIVE HIGH` を使用する。** アーカイブ圧縮と更新の多いワークロードを混在させると、深刻な行移行とパフォーマンス低下が発生する。

---

## 9. よくある間違いとその回避方法

### 間違い 1: 小さな表でスマート・スキャンを期待する

スマート・スキャンは、ダイレクト・パス・リードを使用する大規模セグメントに対してのみ有効になる。表がバッファ・キャッシュに収まる場合、Oracle はキャッシュされた読み取りを使用し、スマート・スキャンは有効にならない。この閾値は `_small_table_threshold` によって制御される。

```sql
-- 閾値 (データベース・ブロック数) の確認
SELECT name, value FROM v$parameter WHERE name = '_small_table_threshold';
-- ブロックサイズを掛けてバイト単位の閾値を取得
-- このサイズを下回る表はバッファ・キャッシュの読み取りを使用し、スマート・スキャンを使用しない
```

### 間違い 2: OLTP 表での HCC の使用

HCC は頻繁な DML と根本的に互換性がない。HCC 行を UPDATE するたびに、その行は圧縮ユニットから OLTP 形式に移行される。時間が経つにつれて。表は圧縮された行と未圧縮の行が混在した状態になり、領域の浪費と読み取り速度の低下を招く。

**対策:** OLTP 表には `COMPRESS FOR OLTP` (アドバンスト行圧縮) を使用する。HCC はアペンド・オンリーまたはアーカイブ表に限定して使用すること。

### 間違い 3: システム全体でセル・オフロードを無効にする

システム・レベルで `cell_offload_processing = FALSE` を設定すると、グローバルにスマート・スキャンが無効になる。これは特定のクエリのバグ回避のために行われることがあるが、その後パラメータを戻し忘れると、Exadata が単なる通常のデータベース・サーバーとして動作することになってしまう。

```sql
-- セル・オフロードが有効かどうかの確認
SELECT name, value FROM v$parameter WHERE name = 'cell_offload_processing';
-- Exadata では TRUE である必要がある

-- セッション・レベルでバグ回避のために設定したとしても、システム全体で永続化させてはいけない
-- ALTER SESSION SET cell_offload_processing = FALSE;  -- セッションのみ、許容範囲
-- ALTER SYSTEM  SET cell_offload_processing = FALSE;  -- 絶対にやってはいけない
```

### 間違い 4: 大量ロード後のストレージ索引の無効化を無視する

ストレージ索引はデータがアクセスされる際にメモリー内に構築される。大規模な一括ロードやパーティション交換の直後は、影響を受けるリージョンのストレージ索引は「コールド」状態である。新しくロードされたデータに対する最初の数回のクエリ実行は、ストレージ索引によるスキップの恩恵を受けられない。これは期待される動作であり、パフォーマンスの低下ではない。

### 間違い 5: データベース・レイヤーのみで Exadata の診断を行う

Exadata のパフォーマンス問題は、ストレージ・セル（CELLSRV プロセス、フラッシュ・キャッシュ、ディスク I/O）に起因することが多い。セル・レベルの診断には `CellCLI` へのアクセスか、OCI コンソール経由で利用可能な Exadata メトリックが必要である。パフォーマンスの問題がクエリに関連していると結論付ける前に、必ず `V$SYSSTAT` と並行してセル・レベルのメトリックを確認すること。

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Exadata Database Machine System Overview](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/) — スマート・スキャン、ストレージ索引、HCC、IORM
- [Oracle Database SQL Language Reference 19c — COMPRESS clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html) — HCC 圧縮レベル
- [DBMS_RESOURCE_MANAGER (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESOURCE_MANAGER.html) — IORM プラン・ディレクティブ
- [Oracle Exadata System Software User's Guide](https://docs.oracle.com/en/engineered-systems/exadata-database-machine/sagug/) — CellCLI, IORM 構成
