# メモリ・チューニング — SGA、PGA、および Oracle のメモリ管理

## 概要

Oracle データベースのメモリは、主に次の 2 つの構造に分けられる。

- **SGA (System Global Area):** すべてのセッションとバックグラウンド・プロセスによって使用される共有メモリ。バッファ・キャッシュ、共有プール、ラージ・プール、Java プール、ストリーム・プール、および REDO ログ・バッファが含まれる。
- **PGA (Program Global Area):** セッションごとのプライベート・メモリ。ソート・エリア、ハッシュ・ジョイン・エリア、ビットマップ・オペレーション、およびセッション状態に使用される。

メモリのサイジングは、最も効果の高いチューニング活動の 1 つである。適切に設定すれば多くの問題が解消するが、誤れば過度な I/O、ハード・パース、TEMP へのあふれ（スビル）に直面することになる。

Oracle には 3 つのメモリ管理モードがある。
- **手動メモリ管理:** 管理者が各 SGA コンポーネントおよび PGA ソート・エリア・パラメータを手動で設定する。
- **ASMM (Automatic Shared Memory Management):** 管理者が `SGA_TARGET` を設定し、Oracle が SGA コンポーネント間で自動的にメモリを分配する。
- **AMM (Automatic Memory Management):** 管理者が `MEMORY_TARGET` を設定し、Oracle が SGA と PGA の両方を自動的に管理する。

---

## SGA コンポーネント

### バッファ・キャッシュ (Buffer Cache)

バッファ・キャッシュは、ディスクから読み取られたデータ・ブロックのコピーを保持する。通常、SGA で最大のコンポーネントであり、パフォーマンスに最も敏感である。

```sql
-- 現在のバッファ・キャッシュ・サイズ
SELECT component,
       current_size / 1024 / 1024  AS current_mb,
       min_size / 1024 / 1024      AS min_mb,
       last_oper_type
FROM   v$sga_dynamic_components
WHERE  component = 'DEFAULT buffer cache';

-- バッファ・キャッシュ・ヒット率 (起動時からの累積)
SELECT ROUND(
         (1 - (phyrds.value / (dbgets.value + congets.value))) * 100, 2
       ) AS buffer_hit_pct
FROM   v$sysstat phyrds,
       v$sysstat dbgets,
       v$sysstat congets
WHERE  phyrds.name  = 'physical reads'
  AND  dbgets.name  = 'db block gets'
  AND  congets.name = 'consistent gets';

-- バッファ・キャッシュを増やした場合に物理読み取りがどれくらい減るか？
SELECT size_for_estimate          AS cache_size_mb,
       size_factor,
       estd_physical_reads,
       estd_physical_read_factor,
       ROUND((1 - estd_physical_read_factor) * 100, 1) AS pct_reduction
FROM   v$db_cache_advice
WHERE  name       = 'DEFAULT'
  AND  block_size = (SELECT TO_NUMBER(value) FROM v$parameter WHERE name = 'db_block_size')
ORDER  BY size_for_estimate;
```

**目安:** OLTP ワークロードではバッファ・キャッシュ・ヒット率 > 95% が望ましい。ただし、常に物理読み取りの実数も確認すること。ヒット率 99% でも、分あたり 1,000 万回の物理読み取りが発生していれば、依然として大量の I/O である。

### Keep プールと Recycle プール

特定のアクセス・パターン用の個別のバッファ・プール：

```sql
-- Keep プール: 小規模で頻繁にアクセスされる表用 (溢れさせない)
-- 例: 小さな参照表を KEEP プールに固定 (PIN) する
ALTER TABLE country_codes STORAGE (BUFFER_POOL KEEP);

-- Recycle プール: 大規模で頻繁にアクセスされないオブジェクト用 (すぐに解放)
ALTER TABLE archive_log STORAGE (BUFFER_POOL RECYCLE);

-- プールのサイズ設定
ALTER SYSTEM SET DB_KEEP_CACHE_SIZE   = 64M;
ALTER SYSTEM SET DB_RECYCLE_CACHE_SIZE = 32M;
```

### 共有プール (Shared Pool)

共有プールには以下が保持される：
- **ライブラリ・キャッシュ (Library Cache):** 解析済み SQL、PL/SQL コード、実行計画。
- **ディクショナリ・キャッシュ (Row Cache):** キャッシュされたデータ・ディクショナリのメタデータ。
- **結果キャッシュ (Result Cache, 12c+):** キャッシュされたクエリ実行結果セット。

```sql
-- 共有プールのサイズ
SELECT component, current_size / 1024 / 1024 AS mb
FROM   v$sga_dynamic_components
WHERE  component IN ('shared pool', 'large pool', 'java pool', 'streams pool');

-- ライブラリ・キャッシュの健全性
SELECT namespace,
       gets,
       gethits,
       ROUND(gethitratio * 100, 2)  AS get_hit_pct,
       pins,
       pinhits,
       ROUND(pinhitratio * 100, 2)  AS pin_hit_pct,
       reloads,
       invalidations
FROM   v$librarycache
ORDER  BY gets DESC
FETCH  FIRST 10 ROWS ONLY;
-- ピン・ヒット率 (Pin hit ratio) とゲット・ヒット率 (Get hit ratio) は共に > 99% であるべき。

-- ディクショナリ・キャッシュ (行キャッシュ) ミス率
SELECT parameter,
       gets,
       getmisses,
       ROUND(getmisses / NULLIF(gets, 0) * 100, 2) AS miss_pct
FROM   v$rowcache
WHERE  gets > 0
ORDER  BY getmisses DESC
FETCH  FIRST 15 ROWS ONLY;
-- 安定したワークロードでのミス率は < 2% であるべき。

-- 共有プールの空きメモリ
SELECT name, bytes / 1024 / 1024 AS mb
FROM   v$sgastat
WHERE  pool = 'shared pool'
  AND  name = 'free memory';
-- ほぼゼロの場合、共有プールが不足しているか、激しく断片化している可能性がある。
```

### 共有プールの断片化

時間の経過とともに、多数の小さな割り当てと解放によって共有プールが断片化し、空きメモリが合計では存在していても、`ORA-04031: unable to allocate N bytes of shared memory` エラーが発生することがある。

```sql
-- 割り当て失敗の確認
SELECT name, value
FROM   v$sysstat
WHERE  name LIKE '%shared pool%';

-- 共有プール予約領域の統計 (V$SHARED_POOL_RESERVED)
SELECT free_space,
       avg_free_size,
       free_count,
       used_space,
       avg_used_size,
       used_count,
       request_failures,
       last_failure_size
FROM   v$shared_pool_reserved;

-- オブジェクトを共有プールに固定 (PIN) して、解放を防止し断片化を軽減する
-- (頻繁に使用される PL/SQL パッケージ用)
BEGIN
  DBMS_SHARED_POOL.KEEP('HR.ORDER_PROCESSING', 'P');  -- P = Package
  DBMS_SHARED_POOL.KEEP('SYS.STANDARD', 'P');
END;
/

-- 固定されたオブジェクトの表示
SELECT owner, name, type, kept
FROM   v$db_object_cache
WHERE  kept = 'YES';
```

### ラージ・プール (Large Pool)

ラージ・プールは、共有プールを圧迫する可能性がある特定の大きな割り当てのための独立したメモリ・プールである：

- RMAN バックアップ/リカバリ操作。
- パラレル・クエリのメッセージ・バッファ。
- 共有サーバー (`UGA` メモリ)。
- Oracle XA トランザクション管理。

```sql
-- ラージ・プールの使用状況確認
SELECT name, bytes / 1024 / 1024 AS mb
FROM   v$sgastat
WHERE  pool = 'large pool'
ORDER  BY bytes DESC;

-- ラージ・プールのサイズ設定
ALTER SYSTEM SET LARGE_POOL_SIZE = 256M;
```

**目安:** ラージ・プールは、「RMAN パラレル数 × チャネル・バッファ・サイズ」+「パラレル・クエリ・スレーブ数 × メッセージ・バッファ・サイズ」に基づいてサイズを決定する。

### REDO ログ・バッファ

REDO ログ・バッファは、LGWR によってオンライン REDO ログ・ファイルに書き込まれる前に、REDO レコードを保持する SGA 内の循環バッファである。

```sql
-- REDO ログ・バッファ・サイズ
SELECT name, bytes / 1024 AS kb
FROM   v$sgainfo
WHERE  name = 'Redo Buffers';

-- REDO バッファの競合
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('redo log space requests',
                'redo log space wait time',
                'redo buffer allocation retries');
-- 'redo log space requests' > 0 は REDO バッファが小さすぎる可能性がある。

-- 現在の REDO パラメータ
SELECT name, value FROM v$parameter WHERE name = 'log_buffer';
```

**サイジングの指針:** デフォルトで十分な場合が多い（3 ～ 15 MB）。`redo log space requests > 0` の場合やヒット率が低い場合に増やす。非常に大きな値（> 64MB）が効果的であることは稀である。

---

## PGA メモリ管理

PGA はセッションごとのプライベート・メモリである。主な消費要因：

- **ソート・エリア (Sort Area):** `ORDER BY`、`GROUP BY`、`DISTINCT`、索引作成。
- **ハッシュ・エリア (Hash Area):** ハッシュ・ジョイン、ハッシュ集計。
- **ビットマップ・エリア (Bitmap Area):** ビットマップ・マージ操作。
- **セッション状態:** カーソル状態、バインド変数、実行コンテキスト。

### 自動 PGA 管理 (推奨)

```sql
-- 自動 PGA 管理を有効にする
ALTER SYSTEM SET WORKAREA_SIZE_POLICY = AUTO;     -- デフォルト
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 2G;        -- 全セッションの合計ターゲット

-- 12c+: PGA のハード制限 (暴走クエリによるインスタンスのクラッシュを防止)
ALTER SYSTEM SET PGA_AGGREGATE_LIMIT = 4G;
-- 通常、PGA_AGGREGATE_TARGET の 2 倍以上に設定する。

-- 現在の PGA 使用状況とターゲットの確認
SELECT name, value
FROM   v$pgastat
WHERE  name IN (
  'aggregate PGA target parameter',
  'aggregate PGA auto target',
  'total PGA inuse',
  'total PGA allocated',
  'maximum PGA allocated',
  'total freeable PGA memory',
  'cache hit percentage',
  'recompute count (total)'
);
-- ほとんどのワークロードでキャッシュ・ヒット率は > 90% であるべき。
```

### PGA アドバイザ

```sql
-- PGA を増やした場合に TEMP I/O がどれくらい減るか？
SELECT pga_target_for_estimate / 1024 / 1024   AS pga_target_mb,
       pga_target_factor,
       estd_pga_cache_hit_percentage,
       estd_overalloc_count   -- ゼロでない場合は PGA が小さすぎる
FROM   v$pga_target_advice
ORDER  BY pga_target_for_estimate;
-- cache_hit_pct が横ばいになり、overalloc_count = 0 になるポイントを探す。
```

### セッションごとの PGA 使用量

```sql
-- PGA 消費の上位セッション
SELECT s.sid,
       s.username,
       s.program,
       s.sql_id,
       p.pga_used_mem / 1024 / 1024     AS pga_used_mb,
       p.pga_alloc_mem / 1024 / 1024    AS pga_alloc_mb,
       p.pga_max_mem / 1024 / 1024      AS pga_max_mb
FROM   v$session s
JOIN   v$process p ON s.paddr = p.addr
WHERE  s.type = 'USER'
ORDER  BY p.pga_used_mem DESC
FETCH  FIRST 10 ROWS ONLY;
```

### TEMP へのソートあふれ (ディスク・ソート)

ソートやハッシュ・ジョインのためのワーク・エリアが不足すると、Oracle は TEMP 表領域（ディスク）へデータを書き出す。これは処理速度を著しく低下させる。

```sql
-- 現在発生しているソートあふれの検出
SELECT s.sid,
       s.username,
       s.sql_id,
       su.blocks * (SELECT TO_NUMBER(value) FROM v$parameter WHERE name = 'db_block_size')
         / 1024 / 1024 AS temp_mb
FROM   v$sort_usage su
JOIN   v$session s ON su.session_addr = s.saddr
ORDER  BY su.blocks DESC;

-- 履歴: ディスク・ソート의 発生回数
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('sorts (memory)', 'sorts (disk)');
-- 'sorts (disk)' は 'sorts (memory)' のごく一部であるべき。

-- AWR からの確認:
SELECT snap_id,
       sorts_disk,
       sorts_mem,
       ROUND(sorts_disk * 100.0 / NULLIF(sorts_disk + sorts_mem, 0), 2) AS pct_disk
FROM (
  SELECT snap_id,
          SUM(CASE WHEN stat_name = 'sorts (disk)'   THEN value END) AS sorts_disk,
          SUM(CASE WHEN stat_name = 'sorts (memory)' THEN value END) AS sorts_mem
  FROM   dba_hist_sysstat
  WHERE  stat_name IN ('sorts (disk)', 'sorts (memory)')
    AND  snap_id BETWEEN 200 AND 210
  GROUP  BY snap_id
)
ORDER  BY snap_id;
```

---

## AMM vs. ASMM

### 自動メモリ管理 (AMM)

SGA と PGA の両方を自動的に管理する。`MEMORY_TARGET` と、必要に応じて `MEMORY_MAX_TARGET` のみを設定する。

```sql
-- AMM を有効にする
ALTER SYSTEM SET MEMORY_TARGET     = 8G SCOPE = SPFILE;
ALTER SYSTEM SET MEMORY_MAX_TARGET = 12G SCOPE = SPFILE;
-- AMM が有効な場合、SGA_TARGET と PGA_AGGREGATE_TARGET はサブ・リミットになるか 0 に設定される。
-- Linux では /dev/shm (tmpfs) が十分に大きい必要がある。

-- AMM が有効か確認
SELECT name, value
FROM   v$parameter
WHERE  name IN ('memory_target', 'memory_max_target', 'sga_target', 'pga_aggregate_target');
```

**AMM の利点:** 管理負荷が最も低い。需要に応じて、Oracle が SGA と PGA の間でメモリを動的に移動させる。

**AMM の欠点:**
- Linux で HugePages をサポートしていない（大規模インスタンスでは致命的なパフォーマンス低下の原因となる）。
- 予測可能性が低い（メモリが予期せず移動することがある）。
- 大規模メモリを搭載した Linux 本番環境では推奨されない。

### 自動共有メモリ管理 (ASMM)

SGA コンポーネントを自動的に管理する。`SGA_TARGET` を設定し、必要に応じて各コンポーネントの最小サイズを設定する。PGA は `PGA_AGGREGATE_TARGET` を介して個別に管理される。

```sql
-- ASMM を有効にする (Linux 本番環境での推奨)
ALTER SYSTEM SET MEMORY_TARGET  = 0;        -- AMM を無効化
ALTER SYSTEM SET SGA_TARGET     = 16G;
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 4G;

-- オプション: 各コンポーネントの最小サイズを設定 (Oracle はこれ未満には縮小しない)
ALTER SYSTEM SET DB_CACHE_SIZE     = 4G;      -- バッファ・キャッシュの最小値
ALTER SYSTEM SET SHARED_POOL_SIZE  = 1G;      -- 共有プールの最小値
ALTER SYSTEM SET LARGE_POOL_SIZE   = 256M;    -- ラージ・プールの最小値

-- SGA アドバイザ (ASMM): Oracle は SGA_TARGET をどのように分配すべきか？
SELECT component,
       current_size / 1024 / 1024     AS current_mb,
       min_size / 1024 / 1024         AS min_mb,
       oper_count,
       last_oper_type
FROM   v$sga_dynamic_components
ORDER  BY current_size DESC;
```

### SGA_TARGET アドバイザ

```sql
-- SGA_TARGET を増やせば効果があるか？ (ASMM のみ)
SELECT sga_size          AS sga_mb,
       sga_size_factor,
       estd_db_time,
       estd_db_time_factor,
       estd_physical_reads
FROM   v$sga_target_advice
ORDER  BY sga_size;
-- SGA を 2 倍にしても改善幅が 10% 未満になる限界点を探す。
```

---

## V$SGA 要約ビュー

```sql
-- SGA の概要
SELECT name, bytes / 1024 / 1024 AS mb
FROM   v$sga
ORDER  BY bytes DESC;

-- 詳細な SGA コンポーネント
SELECT pool, name, bytes / 1024 / 1024 AS mb
FROM   v$sgastat
ORDER  BY bytes DESC
FETCH  FIRST 20 ROWS ONLY;

-- SGA 情報 (12c+)
SELECT name, bytes / 1024 / 1024 AS mb, resizeable
FROM   v$sgainfo
ORDER  BY bytes DESC;

-- インスタンス起動時からの最大 PGA 使用量
SELECT name, value / 1024 / 1024 AS max_mb
FROM   v$pgastat
WHERE  name = 'maximum PGA allocated';
```

---

## HugePages の構成 (Linux)

Linux システムでは、SGA に **HugePages** (4KB ではなく 2MB ページ) を使用することで、TLB (Translation Lookaside Buffer) の負荷を劇的に軽減し、ページ・スワップを防止できる。

```sql
-- 必要な HugePages を算出するために現在の SGA サイズを確認
SELECT SUM(bytes) / 1024 / 1024 AS total_sga_mb
FROM   v$sgainfo
WHERE  name != 'Free SGA Memory Available';
```

```bash
# Linux: 必要な HugePages の計算
# SGA (MB) / 2 = 必要な 2MB HugePages の数 (さらに ~10% のバッファを追加)
# /etc/sysctl.conf で設定する例:
# vm.nr_hugepages = <計算値>

# 現在の HugePages の確認
grep -i huge /proc/meminfo
```

**重要:** HugePages を使用する場合、Oracle の初期化パラメータで `USE_LARGE_PAGES = ONLY` を設定する。また、`MEMORY_TARGET` (AMM) は HugePages を使用**できない**ため、Linux の本番環境では ASMM が好まれる。

```sql
ALTER SYSTEM SET USE_LARGE_PAGES = ONLY SCOPE = SPFILE;
-- 起動時に十分な HugePages がない場合、インスタンスの起動は失敗する。
-- 利用できない場合に通常のページにフォールバックさせたい場合は 'TRUE' を使用する。
```

---

## メモリ・チューニングのワークフロー

```sql
-- ステップ 1: 現在のメモリ構成を確認
SELECT name, value
FROM   v$parameter
WHERE  name IN ('sga_target', 'sga_max_size', 'pga_aggregate_target',
                'pga_aggregate_limit', 'memory_target', 'db_cache_size',
                'shared_pool_size', 'large_pool_size', 'log_buffer');

-- ステップ 2: メモリ不足の指標を確認
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('free buffer waits',      -- バッファ・キャッシュ不足
                'redo log space requests', -- REDO バッファ不足
                'sorts (disk)',            -- PGA 不足
                'parse count (hard)');    -- 共有プール不足 / バインド変数問題

-- ステップ 3: アドバイザ・ビューを確認
-- バッファ・キャッシュ:
SELECT size_for_estimate, estd_physical_reads FROM v$db_cache_advice
WHERE  name = 'DEFAULT' ORDER BY size_for_estimate;
-- 共有プール:
SELECT shared_pool_size_for_estimate, estd_lc_size, estd_lc_memory_objects
FROM   v$shared_pool_advice ORDER BY shared_pool_size_for_estimate;
-- PGA:
SELECT pga_target_for_estimate, estd_pga_cache_hit_percentage FROM v$pga_target_advice
ORDER  BY pga_target_for_estimate;

-- ステップ 4: 変更を反映する (ASMM の例)
-- バッファ・キャッシュ・アドバイザで大幅な読み取り削減が見込める場合:
ALTER SYSTEM SET SGA_TARGET = 20G;  -- Oracle が再分配する
-- PGA overalloc_count > 0 の場合:
ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 6G;
```

---

## ベスト・プラクティス

- **Linux 本番環境では AMM より ASMM + HugePages を優先する。** 大規模な SGA では HugePages によるパフォーマンスのメリットが大きいが、AMM とは互換性がない。
- **Restart なしでの動的拡張を可能にするため、`SGA_MAX_SIZE` >= `SGA_TARGET` とする。**
- **頻繁に使用されるパッケージを共有プールに固定する。** `DBMS_SHARED_POOL.KEEP` を使用して断片化を軽減し、高負荷時の追い出しを防止する。
- **`free buffer waits` および `sorts (disk)` を監視する。** これらはメモリ不足の早期警告信号となる。
- **変更前にアドバイザ・ビューを活用する。** `V$DB_CACHE_ADVICE`、`V$PGA_TARGET_ADVICE`、`V$SHARED_POOL_ADVICE` を参照し、変更による期待効果を把握する。
- **OS 用の余白を残す。** RAM の 100% を Oracle に割り当ててはならない。少なくとも 10 ～ 20% は OS、ファイル・システム・キャッシュ（Direct I/O を未使用の場合）、その他のプロセスのために残しておくこと。
- **ラージ・プールを明示的にサイジングする。** RMAN パラレル・チャネルや多数のパラレル・クエリ・スレーブを使用する場合。

```sql
-- 共有プール・アドバイザの確認
SELECT shared_pool_size_for_estimate / 1024 / 1024 AS sp_mb,
       estd_lc_size / 1024 / 1024                  AS lc_mb,
       estd_lc_memory_objects,
       estd_lc_time_saved_factor
FROM   v$shared_pool_advice
ORDER  BY shared_pool_size_for_estimate;
-- estd_lc_time_saved_factor で収穫逓減のポイントを確認できる。
```

---

## よくある間違い

| 間違い | 影響 | 対策 |
|---|---|---|
| Linux で HugePages と AMM を併用する | HugePages が無視される。TLB の競合が発生 | ASMM を使用し、AMM を無効化 (`MEMORY_TARGET=0`) する。 |
| `SGA_TARGET` を全 RAM 容量に設定する | OS がリソース不足に陥り、スワップが発生 | OS 用に RAM の 20% を残す。 |
| `PGA_AGGREGATE_LIMIT` を設定しない (12c+) | 暴走クエリがすべての RAM を使い果たす可能性がある | ターゲットの 2 倍以上の制限を常に設定する。 |
| RMAN 使用時に LARGE_POOL を過小評価する | 共有プールから割り当てられ、断片化を引き起こす | チャネル数 × バッファ・サイズに基づきラージ・プールを設定する。 |
| PGA アドバイザの `estd_overalloc_count` を無視する | PGA が不足している証拠である | overalloc_count = 0 になるまで PGA を増やす。 |
| 共有プールのすべてのオブジェクトを固定する | 必要なオブジェクトを追い出し、領域を浪費する | 大規模で頻繁に無効化されるパッケージのみに絞る。 |
| ASMM 有効時に個々の SGA コンポーネント・サイズを固定する | 硬直的なサブ・リミットを生み、柔軟性を損なう | ASMM 使用時は、最小サイズとしてのガード的な設定に留める。 |
| MEMORY_TARGET 増設後に Linux の `/dev/shm` の拡張を忘れる | 起動時に ORA-00845 エラーが発生 | `/dev/shm` を `MEMORY_MAX_TARGET` 以上に増やす。 |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Performance Tuning Guide (TGDBA)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)
- [V$SGA — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SGA.html)
- [V$PGASTAT — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-PGASTAT.html)
- [V$DB_CACHE_ADVICE — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DB_CACHE_ADVICE.html)
- [V$PGA_TARGET_ADVICE — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-PGA_TARGET_ADVICE.html)
- [DBMS_SHARED_POOL — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SHARED_POOL.html)
