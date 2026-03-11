# 待機イベント (Wait Events) — 診断と根本原因の特定

## 概要

Oracle の待機イベント・インフラストラクチャは、パフォーマンス診断の基盤である。セッションが処理を続行できず、何らかのリソースのために一時停止する必要があるたびに、イベント名、パラメータ、および待機時間が記録される。これらのイベントは、リアルタイム (`V$SESSION_WAIT`, `V$SESSION`) および履歴の集計 (`V$SYSTEM_EVENT`, AWR, ASH) の両方で追跡される。

待機イベントは、いくつかの **待機クラス (Wait Classes)** に分類される：

| 待機クラス | 説明 |
|---|---|
| `User I/O` | 物理 I/O の待機 (表や索引の読み取り) |
| `System I/O` | バックグラウンド・プロセスによる I/O の待機 (DBWR, ARCH など) |
| `Commit` | コミット後の REDO ログ書き込み待ち |
| `Concurrency` | データベース内部リソースの競合 |
| `Configuration` | パラメータ設定に起因する待機 |
| `Network` | SQL*Net 通信の待機 |
| `Administrative` | 管理操作 (ALTER TABLE, rebuild など) による待機 |
| `Application` | アプリケーション・レベルのロックや待機 |
| `Cluster` | RAC インスタンス間の通信待機 |
| `Other` | その他諸々 |
| `Idle` | セッションがアイドル状態 (分析対象から除外する) |

---

## 待機イベントの主要なビュー

### V$SESSION_WAIT — 現在の待機

```sql
-- アクティブなセッションが現在何を待機しているかを確認
SELECT sw.sid,
       s.serial#,
       s.username,
       s.program,
       sw.event,
       sw.wait_class,
       sw.seconds_in_wait,
       sw.state,
       sw.p1text, sw.p1,
       sw.p2text, sw.p2,
       sw.p3text, sw.p3
FROM   v$session_wait sw
JOIN   v$session s ON sw.sid = s.sid
WHERE  sw.wait_class != 'Idle'
ORDER  BY sw.seconds_in_wait DESC;
```

### V$SESSION — セッション情報と待機情報の統合

```sql
-- アクティブ・セッションの包括的なビュー
SELECT s.sid,
       s.serial#,
       s.username,
       s.status,
       s.sql_id,
       s.event,
       s.wait_class,
       s.seconds_in_wait,
       s.blocking_session,
       s.module,
       s.action,
       s.program,
       s.machine
FROM   v$session s
WHERE  s.status    = 'ACTIVE'
  AND  s.wait_class != 'Idle'
  AND  s.type       = 'USER'
ORDER  BY s.seconds_in_wait DESC;
```

### V$SYSTEM_EVENT — システム全体の累積待機

```sql
-- インスタンス起動時 (または統計のリセット以降) の上位待機イベント
SELECT event,
       wait_class,
       total_waits,
       total_timeouts,
       time_waited_micro / 1e6              AS time_waited_sec,
       ROUND(average_wait * 10, 3)          AS avg_wait_ms,  -- AVERAGE_WAIT は 1/100 秒単位。10 倍して ms に変換。
       ROUND(time_waited_micro / 1e6 /
             NULLIF(total_waits, 0) * 1000, 3) AS avg_wait_ms_v2
FROM   v$system_event
WHERE  wait_class != 'Idle'
ORDER  BY time_waited_micro DESC
FETCH  FIRST 20 ROWS ONLY;
```

### V$EVENT_HISTOGRAM — 待機の分布

```sql
-- 特定のイベントの待機時間の分布を確認
SELECT event,
       wait_time_milli,
       wait_count,
       ROUND(100 * wait_count / SUM(wait_count) OVER (PARTITION BY event), 2) AS pct
FROM   v$event_histogram
WHERE  event = 'db file sequential read'
ORDER  BY wait_time_milli;
```

---

## db file sequential read

### 概要

シングルブロックの物理 I/O。セッションがディスクからバッファ・キャッシュへ、ちょうど 1 つのデータ・ブロックが読み込まれるのを待っている状態。以下のような場合に発生する標準的な待機である：

- 索引レンジ・スキャンによる行の取得 (各 rowid ごとに 1 ブロック・フェッチ)。
- 非常に小さな表の全表スキャン。
- UNDO セグメントの読み取り。

```sql
-- 診断: 現在、どのオブジェクトが最も多くのシーケンシャル・リードを引き起こしているか？
-- P1 = ファイル番号 (絶対番号), P2 = ブロック番号, P3 = ブロック数
-- DBA_EXTENTS を介して、ファイル番号とブロック番号をセグメントに紐付ける。
SELECT o.owner,
       o.object_name,
       o.object_type,
       COUNT(*) AS waits
FROM   v$session_wait sw
JOIN   v$session s ON sw.sid = s.sid
JOIN   dba_extents e ON e.file_id  = sw.p1
                    AND sw.p2 BETWEEN e.block_id AND e.block_id + e.blocks - 1
JOIN   dba_objects o ON o.object_id = e.object_id
WHERE  sw.event = 'db file sequential read'
GROUP  BY o.owner, o.object_name, o.object_type
ORDER  BY waits DESC;

-- AWR から: どのオブジェクトの I/O が最も多かったか？
SELECT owner, object_name, object_type,
       physical_reads_total
FROM   dba_hist_seg_stat_obj
WHERE  snap_id BETWEEN 200 AND 201
ORDER  BY physical_reads_total DESC
FETCH  FIRST 20 ROWS ONLY;
```

### 対策

- **不適切な索引の選択:** 実行計画が最適な索引を使用しているか確認する。統計情報が古いと、間違ったアクセス・パスが選択されることがある。
- **クラスタ化係数 (Clustering Factor) が高すぎる:** 表の行が散らばっており、索引ルックアップのたびに新しいブロックをフェッチする必要がある。表の再編成や IOT の使用を検討する。
- **IOPS 要求が高い:** ストレージの高速化 (SSD/NVMe)、バッファ・キャッシュ・サイズの拡大、または Exadata Smart Scan の活用を検討する。
- **索引の不足:** 別の索引スキャンの後にフル行フェッチが発生している。カバリング・インデックスの作成を検討する。

---

## db file scattered read

### 概要

マルチブロックの物理 I/O。連続する複数のブロック (最大 `DB_FILE_MULTIBLOCK_READ_COUNT` 分) を 1 回の I/O で読み込む。以下のような場合に発生する：

- 全表スキャン (Full Table Scan)。
- 索引全スキャン (Fast Full Scan)。
- ダイレクト・パス・リード (パラレル・クエリ、一括ロード)。

```sql
-- マルチブロック読み取り数の確認
SELECT name, value FROM v$parameter WHERE name = 'db_file_multiblock_read_count';

-- 全表スキャンを引き起こしている SQL の特定 (AWR から)
SELECT sql_id,
       ROUND(elapsed_time_total / 1e6, 2) AS elapsed_sec,
       disk_reads_total,
       executions_total,
       SUBSTR(sql_text, 1, 80) AS sql_text
FROM   dba_hist_sqlstat
JOIN   dba_hist_sqltext USING (sql_id, dbid)
WHERE  snap_id BETWEEN 200 AND 201
  AND  disk_reads_total > 100000
ORDER  BY disk_reads_total DESC
FETCH  FIRST 10 ROWS ONLY;
```

### 対策

- **不要な全表スキャン:** OLTP クエリに対して適切な索引を追加する。
- **意図的な全表スキャン (分析系):** 大規模な分析スキャンの場合は許容されるが、パラレル・クエリの使用を検討する。
- **キャッシュ:** 頻繁にアクセスされる小さな表をキャッシュする (`ALTER TABLE t CACHE`)。
- **パーティショニング:** パーティション・プルーニングによりスキャン範囲を制限する。

---

## log file sync

### 概要

セッションが `COMMIT` を発行するたびに、Oracle はログ・ライター (LGWR) が REDO バッファをオンライン REDO ログ・ファイルに書き込み、完了報告を待機する必要がある。`log file sync` は、コミットを発行したセッションが LGWR の応答を待っている時間である。

```sql
-- 現在の log file sync 待機
SELECT sid, serial#, event, seconds_in_wait, state
FROM   v$session
WHERE  event = 'log file sync'
ORDER  BY seconds_in_wait DESC;

-- REDO ログのパフォーマンス統計
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('redo writes', 'redo write time', 'user commits',
                'redo size', 'redo synch writes', 'redo synch time');

-- 平均 log file sync 時間 (ms)
SELECT ROUND(sw.time_waited_micro / sw.total_waits / 1000, 2) AS avg_sync_ms
FROM   v$system_event sw
WHERE  event = 'log file sync';
-- > 5ms は注意が必要。 > 20ms は深刻な問題。
```

### 対策

- **REDO ログ・ストレージの低速:** REDO ログをより高速なストレージ（専用 SSD、データファイルとは分離）に移動する。
- **コミット回数が多すぎる:** コミットをバッチ化する（1 行ごとではなく N 行ごとにコミット）。ETL や行ベースのループ処理でよく見られる。
- **REDO ログ・ファイルが小さい:** 頻繁なログ・スイッチが LGWR をビジーにする。ログ・ファイルを適切なサイズに設定する。
- **非同期コミット (永続性のトレードオフが許容される場合):**

```sql
-- 非同期コミット: セッションは LGWR の書き込み完了を待機しない
-- 注意: クラッシュ時に、最後にコミットされたトランザクションが失われる可能性がある。
-- 特定のユースケース (重要度の低い一時データなど) でのみ使用すること。
COMMIT WRITE NOWAIT;
-- またはセッション・レベルで設定：
ALTER SESSION SET COMMIT_WRITE = 'BATCH,NOWAIT';
```

- **REDO ログのマルチプレックス:** ソフトウェア・レベルではなくハードウェア・レベルのミラーリングを使用することで、同期オーバーヘッドを抑えつつ単一故障点を回避する。

---

## buffer busy waits

### 概要

複数のセッションが同時に、バッファ・キャッシュ内の同じバッファを読み取りまたは修正しようとしている状態。1 つのセッションが、他者がアクセスできない状態でバッファを保持している。主な原因：

- **ホットなセグメント・ヘッダ・ブロック:** 同じセグメントへの多数の並列挿入が、セグメント・ヘッダ（空き領域を管理）での競合を引き起こす。
- **ホットなデータ・ブロック:** 多数のセッションが同時に読み取りまたは更新を行う、頻繁にアクセスされるブロック。
- **右側に成長する索引:** シーケンスなどを使用した単調増加するキーへの挿入が、最右翼のリーフ・ブロックで競合を引き起こす。

```sql
-- ホットなブロックの特定 (待機パラメータからファイル番号とブロック番号を取得)
SELECT sw.p1 AS file_num,
       sw.p2 AS block_num,
       sw.p3 AS class_num,
       o.owner,
       o.object_name,
       o.object_type,
       COUNT(*) AS waiters
FROM   v$session_wait sw
JOIN   dba_extents e ON e.file_id  = sw.p1
                    AND sw.p2 BETWEEN e.block_id AND e.block_id + e.blocks - 1
JOIN   dba_objects o ON o.object_id = e.object_id
WHERE  sw.event = 'buffer busy waits'
GROUP  BY sw.p1, sw.p2, sw.p3, o.owner, o.object_name, o.object_type
ORDER  BY waiters DESC;
```

### 対策

- **セグメント・ヘッダの競合:** 複数のフリーリスト・グループを持つ自動セグメント領域管理 (ASSM) を使用する。
- **索引ブロックの競合 (右側成長):** **反転キー索引 (Reverse-Key Index)** を使用して、挿入箇所を多数のリーフ・ブロックに分散させる。

```sql
-- 反転キー索引の作成 (キーのバイト順を反転させ、挿入を分散させる)
CREATE INDEX orders_id_rk ON orders (order_id) REVERSE;
-- 注意: 反転キー索引はレンジ・スキャンには使用できず、等価検索のみ可能。
```

- **ホットなデータ・ブロック:** 同じ行への同時アクセスを減らすためのアプリケーション設計の見直し。パーティショニングも有効。

---

## enqueue waits

### 概要

エンキュー待機は、Oracle のキュー・ベースのロックに関連する。待機イベント名は `enq: XX - description` の形式で、XX はエンキューの種類を表す。

| エンキュー | 説明 | 主な原因 |
|---|---|---|
| `TX - row lock contention` | 行レベル・ロック | 2 つのセッションが同じ行を更新しようとしている |
| `TM - contention` | 表レベル・ロック | DDL 実行中の DML、または索引のない外部キー |
| `HW - contention` | ハイウォーター・マーク拡張 | 多数のセッションが同じセグメントを拡張しようとしている |
| `ST - space transaction` | 領域管理 | 辞書管理表領域での領域管理の競合 |
| `CF - contention` | コントロール・ファイル | 激しい RAC 操作やアーカイブ・ログ活動 |
| `US - contention` | UNDO セグメント | 激しい UNDO セグメントの使用 |

```sql
-- 現在のエンキュー待機
SELECT sw.sid,
       s.username,
       sw.event,
       sw.seconds_in_wait,
       sw.p1raw,    -- ロック・モード + 種類が P1 にエンコードされている
       sw.p2,       -- オブジェクト ID またはトランザクション・スロット
       sw.p3,
       s.blocking_session
FROM   v$session_wait sw
JOIN   v$session s ON sw.sid = s.sid
WHERE  sw.event LIKE 'enq:%'
ORDER  BY sw.seconds_in_wait DESC;

-- TX 待機のブロッキング・セッションの特定
SELECT s.sid, s.serial#, s.username, s.sql_id, s.program,
       l.type, l.lmode, l.request
FROM   v$lock l
JOIN   v$session s ON l.sid = s.sid
WHERE  l.type IN ('TX', 'TM')
  AND  l.block  = 1   -- 1 = このセッションが他者をブロックしている
ORDER  BY l.type, l.sid;

-- ロックされている行の表を特定
SELECT do.object_name,
       row_wait_file#,
       row_wait_block#,
       row_wait_row#
FROM   v$session s
JOIN   dba_objects do ON do.object_id = s.row_wait_obj#
WHERE  s.sid = :blocked_sid;
```

### TX - 行ロック競合の対策

- **原因の特定とセッションの処理:** ブロッキング・セッションがハングしている、または不要な場合は終了させる。
- **アプリケーションの修正:** トランザクション保持時間の短縮、コミット、ロック保持中のユーザー入力を避ける。
- **楽観的ロック・パターンの採用:** `SELECT ... FOR UPDATE NOWAIT` を使用して、待機せずに即座にエラーを返させる。
- **直列化問題:** 多数のセッションが同じ行を更新する場合、アクセスを直列化するようにアプリケーションを再設計する。

---

## library cache waits

### 概要

ライブラリ・キャッシュ関連の待機 (`library cache: mutex X`, `library cache lock`, `library cache pin`) は、共有プール・カーソルでの競合を示す。セッションが以下を巡って競合している：

- 同じ SQL の解析、またはハード・パース。
- カーソルの無効化 (DDL や統計収集時)。
- カーソル・メタデータへのアクセス。

```sql
-- ハード・パース率 (健全な OLTP ではほぼゼロであるべき)
SELECT name, value
FROM   v$sysstat
WHERE  name IN ('parse count (hard)', 'parse count (total)',
                'parse count (failures)', 'execute count');

-- ライブラリ・キャッシュの統計
SELECT namespace,
       gets,
       gethits,
       ROUND(gethitratio * 100, 2) AS get_hit_pct,
       pins,
       pinhits,
       ROUND(pinhitratio * 100, 2) AS pin_hit_pct,
       reloads,
       invalidations
FROM   v$librarycache
WHERE  namespace = 'SQL AREA'
ORDER  BY gets DESC;

-- バインド変数を使用していない SQL (述語内にリテラルを含んでいる)
-- これらはハード・パースとライブラリ・キャッシュの競合を引き起こす
SELECT force_matching_signature,
       COUNT(*)               AS distinct_plans,
       COUNT(DISTINCT sql_id) AS sql_count,
       MAX(SUBSTR(sql_text, 1, 100)) AS sample_text
FROM   v$sql
WHERE  parsing_user_id > 0   -- SYS/SYSTEM ユーザーを除外
GROUP  BY force_matching_signature
HAVING COUNT(*) > 50          -- 同じクエリ・パターンのバリエーションが多い
ORDER  BY COUNT(*) DESC
FETCH  FIRST 20 ROWS ONLY;
```

### 対策

- **バインド変数の使用:** すべてのアプリケーション SQL でバインド変数を使用する。これが最も効果的な対策である。
- **`CURSOR_SHARING = FORCE`** (緊急対策): リテラルをシステム生成のバインド変数に強制的に置換する。ただし、ヒストグラムが使用されなくなったり、最適でない計画になる副作用がある。

```sql
-- システム・レベルの緊急修正 (慎重に行うこと)
ALTER SYSTEM SET CURSOR_SHARING = FORCE;

-- 推奨: アプリケーション側を修正し、修正後は EXACT に戻す
ALTER SYSTEM SET CURSOR_SHARING = EXACT;
```

- **`SHARED_POOL_SIZE` の拡大:** pin/get ヒット率が 99% 未満の場合。
- **`CURSOR_SPACE_FOR_TIME = TRUE`** (12c 以降は非推奨) — 保持されているカーソルをより長くピン留めし、リロードを減らす。

---

## その他の一般的な待機イベント

### free buffer waits

バッファ・キャッシュが小さすぎるか、DBWR がダーティ・バッファを書き出す速度が追いついていない。

```sql
-- バッファ・キャッシュのサイズ推奨値の確認
SELECT size_for_estimate,
       size_factor,
       estd_physical_reads,
       estd_physical_read_factor
FROM   v$db_cache_advice
WHERE  name      = 'DEFAULT'
  AND  block_size = (SELECT value FROM v$parameter WHERE name = 'db_block_size')
ORDER  BY size_for_estimate;
```

**対策:** `DB_CACHE_SIZE` を増やすか、AMM/ASMM を有効にする。DBWR プロセスを増やして書き出しを高速化する (`DB_WRITER_PROCESSES`)。

### direct path read / direct path read temp

バッファ・キャッシュをバイパスして、パラレル全スキャンや大規模なソートを行っているセッション。

- パラレル・クエリ中の `direct path read` は正常な動作。
- `direct path read temp` は、ソートやハッシュ・ジョインがディスク (TEMP) にあふれたことを示す。

```sql
-- TEMP 領域を使用しているセッションの特定
SELECT s.sid, s.username, s.sql_id,
       su.tablespace, su.contents, su.blocks * 8192 / 1024 / 1024 AS mb_used
FROM   v$sort_usage su
JOIN   v$session s ON su.session_addr = s.saddr;
```

**TEMP あふれの対策:** `PGA_AGGREGATE_TARGET` を増やすか、中間結果セットを減らすように SQL を最適化する。

### read by other session

あるセッションがディスクから読み取ろうとしている同じブロックを、すでに別のセッションが読み取っているのを待っている状態。重複した I/O を避けるための待機。

**診断:** `buffer busy waits` と同様にホット・ブロックが原因であることが多い。待機対象のブロックを特定する。

### latch: cache buffers chains

バッファ・キャッシュのハッシュ・バケット・ラッチの競合。極めてホットなブロック（1 秒間に数千回アクセスされるブロック）によって引き起こされる。

**対策:** ホット・ブロックを特定し（待機 P1/P2 から）、所属するセグメントを確認し、アクセス・パターンの見直しやパーティショニングによる分散を検討する。

---

## 待機イベント診断のワークフロー

```sql
-- ステップ 1: 現在の上位待機イベントを特定
SELECT event, COUNT(*) AS sessions
FROM   v$session
WHERE  wait_class != 'Idle'
  AND  status      = 'ACTIVE'
GROUP  BY event
ORDER  BY sessions DESC;

-- ステップ 2: 上位イベントのパラメータ詳細を取得
SELECT sid, event, p1text, p1, p2text, p2, p3text, p3, seconds_in_wait
FROM   v$session_wait
WHERE  event = :top_event
ORDER  BY seconds_in_wait DESC;

-- ステップ 3: 待機セッションが実行している SQL を特定
SELECT s.sid, s.sql_id, q.sql_text, s.event, s.seconds_in_wait
FROM   v$session s
JOIN   v$sql q ON s.sql_id = q.sql_id
WHERE  s.event     = :top_event
  AND  s.wait_class != 'Idle'
ORDER  BY s.seconds_in_wait DESC;

-- ステップ 4: ブロッキング・チェーンを追跡 (誰が誰をブロックしているか)
SELECT l.sid,
       s.username,
       l.type,
       l.lmode,
       l.request,
       l.block,
       s.blocking_session,
       s.event
FROM   v$lock l
JOIN   v$session s ON l.sid = s.sid
WHERE  l.request > 0 OR l.block > 0
ORDER  BY l.block DESC, l.sid;
```

---

## ベスト・プラクティス

- **分析にはアイドル待機イベントを含めないこと。** 常に `wait_class != 'Idle'` でフィルタリングする。
- **時間重み付け (Time-Weighted) 分析を行うこと。** 100 万回の待機であっても合計が 1ms なら重要ではない。100 回の待機であっても合計が 100 秒なら致命的である。
- **待機イベントと SQL を関連付けること。** 待機イベントは「症状」を示し、SQL_ID が「根本原因」を指し示す。
- **パラメータ (`P1`, `P2`, `P3`) を確認すること。** 各イベントにおいて、これらが何を意味するのか（ファイル番号、ブロック番号、ロック・モードなど）を確認する。
- **ASH を活用すること。** 継続的な監視がなかった過去の待機イベントを振り返って分析するのに有効。
- **平均待機時間のトレンドを監視すること。** ストレージの遅延の増加など、徐々に進行する劣化を検出するためにトレンドを確認する。

---

## よくある間違い

| 間違い | 影響 | 対策 |
|---|---|---|
| 分析にアイドル・イベントを含めてしまう | 本当の待機が見えなくなる | 常に `wait_class != 'Idle'` でフィルタリングする。 |
| 待機回数のみを主要指標として扱う | 回数は多いが影響の小さいイベントに惑わされる | `total_waits` ではなく `time_waited` でソートする。 |
| 根本原因を突き止めずにブロッキング・セッションを強制終了する | 同じ問題が再発する | なぜブロックが発生していたのかを突き止める。 |
| P1/P2/P3 パラメータを無視する | オブジェクト・レベルの診断ができなくなる | Oracle のマニュアルで各パラメータの意味を確認する。 |
| `db file sequential read` を常に「悪」とみなす | インデックス・ベースの OLTP では正常な動作である | まず、アクセス・パスが正しいかどうかを確認する。 |
| `log file sync` をストレージの問題とだけ捉える | コミット頻度の問題である可能性もある | まずは Load Profile から 1 秒あたりのコミット数を確認する。 |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Performance Tuning Guide (TGDBA)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)
- [V$SESSION_WAIT — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SESSION_WAIT.html)
- [V$SYSTEM_EVENT — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SYSTEM_EVENT.html)
- [V$EVENT_HISTOGRAM — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-EVENT_HISTOGRAM.html)
- [V$LOCK — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOCK.html)
