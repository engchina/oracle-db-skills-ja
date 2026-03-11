# 領域管理

## 概要

領域管理 (Space Management) は、DBA の責任の中でも最も重要かつ継続的なものの 1 つである。表領域の不足は、アプリケーションの停止を引き起こす最も一般的な原因の 1 つであるが、適切な監視と事前のアクティブな管理によってほぼ完全に防ぐことが可能である。表領域の不足を防ぐだけでなく、効果的な領域管理には、削除された行による無駄な領域の再利用、Oracle の高水位線 (High Water Mark) の仕組みの理解、一時領域の管理、および LOB 列の独自の領域特性の処理が含まれる。

このガイドでは、Oracle データベースにおける領域の監視と管理のための主要なビュー、クエリ、ツール、およびテクニックについて説明する。

---

## 主要な領域監視ビュー

### DBA_TABLESPACE_USAGE_METRICS

表領域の監視において最も重要なビューである。自動拡張の可能性を含め、各表領域のすべてのデータファイル全体で、空き領域と使用済み領域を集計する。

```sql
-- 包括的な表領域使用状況レポート
SELECT m.tablespace_name,
       ROUND(m.tablespace_size    * t.block_size / 1073741824, 2) AS total_gb,
       ROUND(m.used_space         * t.block_size / 1073741824, 2) AS used_gb,
       ROUND(m.free_space         * t.block_size / 1073741824, 2) AS free_gb,
       ROUND(m.used_percent, 1)                                    AS used_pct,
       t.contents,
       t.extent_management,
       t.segment_space_management
FROM   dba_tablespace_usage_metrics m
JOIN   dba_tablespaces              t ON m.tablespace_name = t.tablespace_name
ORDER BY m.used_percent DESC;
```

```sql
-- 使用率が 80% を超えている表領域を検索する (アラートしきい値)
SELECT tablespace_name,
       ROUND(used_percent, 1) AS used_pct,
       ROUND(free_space * 8192 / 1073741824, 2) AS free_gb
FROM   dba_tablespace_usage_metrics
WHERE  used_percent > 80
ORDER BY used_percent DESC;
```

### DBA_DATA_FILES と DBA_FREE_SPACE

データファイル・レベルでのより詳細な分析用:

```sql
-- 自動拡張情報を含むデータファイル・レベルの領域レポート
SELECT df.tablespace_name,
       df.file_name,
       ROUND(df.bytes       / 1073741824, 2) AS file_size_gb,
       ROUND(df.maxbytes    / 1073741824, 2) AS max_size_gb,
       df.autoextensible,
       ROUND(NVL(fs.free_bytes, 0) / 1073741824, 2) AS free_gb,
       ROUND((df.bytes - NVL(fs.free_bytes, 0)) / df.bytes * 100, 1) AS used_pct
FROM   dba_data_files df
LEFT JOIN (
    SELECT file_id, SUM(bytes) AS free_bytes
    FROM   dba_free_space
    GROUP BY file_id
) fs ON df.file_id = fs.file_id
ORDER BY used_pct DESC;
```

### DBA_SEGMENTS による監視

どのオブジェクトが最も領域を消費しているかを理解するため:

```sql
-- サイズの大きいセグメント上位 30 件
SELECT owner,
       segment_name,
       segment_type,
       tablespace_name,
       ROUND(bytes / 1073741824, 3) AS size_gb,
       extents
FROM   dba_segments
WHERE  owner NOT IN ('SYS','SYSTEM','DBSNMP','SYSMAN','OUTLN','XDB')
ORDER BY bytes DESC
FETCH FIRST 30 ROWS ONLY;
```

```sql
-- セグメント・タイプごとの消費領域
SELECT segment_type,
       COUNT(*)                           AS object_count,
       ROUND(SUM(bytes)/1073741824, 2)    AS total_gb
FROM   dba_segments
WHERE  owner NOT IN ('SYS','SYSTEM','DBSNMP','SYSMAN')
GROUP BY segment_type
ORDER BY total_gb DESC;
```

```sql
-- 行数および平均行サイズを含む、最大の表
SELECT s.owner,
       s.segment_name,
       ROUND(s.bytes / 1048576, 1)   AS alloc_mb,
       t.num_rows,
       t.avg_row_len,
       ROUND(t.num_rows * t.avg_row_len / 1048576, 1) AS data_mb,
       ROUND((s.bytes - t.num_rows * t.avg_row_len) / 1048576, 1) AS waste_mb
FROM   dba_segments s
JOIN   dba_tables   t ON s.owner = t.owner AND s.segment_name = t.table_name
WHERE  s.segment_type = 'TABLE'
  AND  s.owner NOT IN ('SYS','SYSTEM','DBSNMP','SYSMAN')
  AND  t.num_rows > 0
ORDER BY waste_mb DESC NULLS LAST
FETCH FIRST 30 ROWS ONLY;
```

---

## 高水位線 (High Water Mark) の概念

### 高水位線 (HWM) とは

**高水位線 (HWM)** は、そのセグメントでこれまでに使用された最大のブロックを追跡する、セグメント内の内部マーカーである。Oracle の全表スキャンは、その下に現在どれだけのデータがあるかに関係なく、HWM までのブロックを読み取る。つまり、かつて 1,000 万行あった表のすべての行を削除しても、全表スキャンでは満杯だった時と同じ数のブロックをスキャンすることになる。

HWM には 2 つのタイプがある。

- **High HWM:** これまでにフォーマットされ、使用された可能性のある絶対的に最大のブロック。全表スキャンはこのポイントまで読み取る。
- **Low HWM (ASSM のみ):** その下のすべてのブロックがフォーマットされていることが保証されているポイント。Low HWM より下の領域は、新しい挿入に優先的に使用される。

### HWM がパフォーマンスに与える影響

```
全表スキャンは、実際のデータに関係なく、ブロック 1 → HWM を読み取る
│
│ ████████████░░░░░░░░░░░░░░░░░░░░░░░░░
│ [データ・ブロック] [HWM マーカーより上の空きブロック]
│
│ 大量削除後：全表スキャンのコストは変わらず、大部分が空きブロックの読み取りになる
```

### HWM と再利用可能な領域の確認

```sql
-- 割り当てられたブロックが、データに必要なブロックを大幅に上回っている表
-- (HWM 再利用の可能性が高い)
SELECT t.owner,
       t.table_name,
       t.num_rows,
       s.blocks             AS allocated_blocks,
       t.blocks             AS hwm_blocks,         -- HWM までのブロック
       t.empty_blocks,                              -- HWM より上の空きブロック
       ROUND(s.bytes/1048576, 1) AS alloc_mb,
       CASE WHEN t.num_rows > 0
            THEN ROUND(t.num_rows * t.avg_row_len / 8192)
            ELSE 0
       END AS estimated_needed_blocks
FROM   dba_tables   t
JOIN   dba_segments s ON t.owner = s.owner AND t.table_name = s.segment_name
WHERE  s.segment_type = 'TABLE'
  AND  t.owner NOT IN ('SYS','SYSTEM','DBSNMP','SYSMAN')
  AND  t.num_rows IS NOT NULL
  AND  s.blocks > 500
ORDER BY (t.blocks - CASE WHEN t.num_rows > 0
                           THEN ROUND(t.num_rows * t.avg_row_len / 8192)
                           ELSE 0
                       END) DESC NULLS LAST
FETCH FIRST 30 ROWS ONLY;
```

**注意:** `DBA_TABLES.BLOCKS` および `NUM_ROWS` には、最新の統計情報が必要である。正確性を期すために、まず `DBMS_STATS.gather_table_stats` を実行すること。

---

## 領域の再利用

### SHRINK SPACE (オンライン、非破壊)

`ALTER TABLE ... SHRINK SPACE` は、行を HWM より下に移動させることで表データを圧縮し、HWM をリセットする。これはオンラインで動作し (DML を継続可能)、表領域が自動セグメント領域管理 (ASSM) を使用している必要がある。

```sql
-- 前提条件：表領域が ASSM を使用しており、表で行移動が有効であること
-- ステップ 1: 行移動を有効化
ALTER TABLE sales ENABLE ROW MOVEMENT;

-- ステップ 2: データの圧縮（行を移動するが、HWM はまだリセットしない）
ALTER TABLE sales SHRINK SPACE COMPACT;
-- 営業時間内は COMPACT を使用する：デフラグを行うが、領域はまだ解放されない

-- ステップ 3: HWM のリセットと領域の解放（短時間の排他ロック）
ALTER TABLE sales SHRINK SPACE;
-- 通常の SHRINK (COMPACT なし) は、1 つの操作で両方のステップを実行する

-- カスケード：関連するすべての索引も縮小する
ALTER TABLE sales SHRINK SPACE CASCADE;
```

```sql
-- 縮小後、影響を受けた索引を再構築（または縮小）する
-- 縮小のアプローチ:
ALTER INDEX sales_idx SHRINK SPACE;

-- または再構築（オフラインだが、より徹底的）:
ALTER INDEX sales_idx REBUILD ONLINE;
```

**SHRINK を使用すべき場面:**
- 表が ASSM 表領域にある
- オンライン操作が必要（メンテナンス・ウィンドウがない）
- 表に対して大量の削除または更新が行われた（初期ロード直後ではない）

**SHRINK を使用すべきでない場面:**
- 表が IOT (索引構成表) 構造である
- 表領域がディクショナリ管理エクステントを使用している (レガシー)
- 行移動によって変更される列に、ファンクション索引が作成されている

### MOVE (オフライン、より徹底的)

`ALTER TABLE ... MOVE` は、表全体を新しいセグメント (同じ表領域または異なる表領域) に物理的に再配置し、HWM を完全にリセットしてストレージをデフラグする。

```sql
-- 表を同じ表領域に移動（HWM をリセットしデフラグする）
ALTER TABLE sales MOVE;

-- 表を別の表領域に移動
ALTER TABLE sales MOVE TABLESPACE data_ts;

-- オンラインで移動 (Oracle 12.2 以降)
ALTER TABLE sales MOVE ONLINE;

-- MOVE 後、すべての索引を再構築する（索引は UNUSABLE となるため）
-- 使用不可能になった索引を検索して再構築する
SELECT 'ALTER INDEX ' || owner || '.' || index_name || ' REBUILD;' AS rebuild_stmt
FROM   dba_indexes
WHERE  table_owner = 'SCOTT'
  AND  table_name  = 'SALES'
  AND  status      = 'UNUSABLE';
```

**MOVE を使用すべき場面:**
- 最大限の領域再利用が必要
- メンテナンス・ウィンドウが許容される (ONLINE オプションを使用しない場合)
- 表領域間でデータを移動する（例：低速ディスクから高速 SSD へ）
- 表が MSSM (手動セグメント領域管理) 表領域にある

### SHRINK と MOVE の比較

| 機能 | SHRINK SPACE | MOVE |
|---------|-------------|------|
| オンライン？ | はい | ONLINE 句を使用した場合のみ (12.2+) |
| 索引の無効化？ | いいえ | はい（再構築が必要） |
| ASSM 必須？ | はい | いいえ |
| 行移動の有効化必須？ | はい | いいえ |
| 再利用の完全性 | 非常に良好 | 完全 |
| 表領域への領域返却 | はい | はい |

---

## 領域再利用のためのセグメント・アドバイザ

Oracle のセグメント・アドバイザは、セグメントを分析し、縮小する価値のあるものを推奨する。アドバイザの実行については、`health-monitor.md` も参照すること。

```sql
-- リポジトリにある既存のアドバイザ結果をクイック・チェック
SELECT f.task_id,
       o.owner,
       o.object_name,
       o.object_type,
       f.message,
       f.more_info
FROM   dba_advisor_findings   f
JOIN   dba_advisor_objects    o ON f.task_id = o.task_id AND f.object_id = o.object_id
JOIN   dba_advisor_tasks      t ON f.task_id = t.task_id
WHERE  t.advisor_name = 'Segment Advisor'
  AND  t.status       = 'COMPLETED'
  AND  o.owner NOT IN ('SYS','SYSTEM')
ORDER BY f.task_id DESC, o.owner, o.object_name;
```

```sql
-- セグメント・アドバイザの推奨事項から SHRINK 文を自動生成
SELECT 'ALTER TABLE ' || o.owner || '.' || o.object_name ||
       ' ENABLE ROW MOVEMENT;' || CHR(10) ||
       'ALTER TABLE ' || o.owner || '.' || o.object_name ||
       ' SHRINK SPACE CASCADE;'   AS shrink_stmt
FROM   dba_advisor_findings   f
JOIN   dba_advisor_objects    o ON f.task_id = o.task_id AND f.object_id = o.object_id
JOIN   dba_advisor_tasks      t ON f.task_id = t.task_id
WHERE  t.advisor_name  = 'Segment Advisor'
  AND  t.status        = 'COMPLETED'
  AND  o.object_type   = 'TABLE'
  AND  o.owner NOT IN ('SYS','SYSTEM')
ORDER BY o.owner, o.object_name;
```

---

## LOB の領域管理

LOB (Large Objects — BLOB, CLOB, NCLOB, BFILE) は、通常の表の行とは根本的に異なる独自のストレージ・メカニズムを持っている。

### LOB ストレージの仕組み

- **インライン・ストレージ:** 小さな LOB (CLOB/NCLOB は 4000 バイト未満、BLOB は 2000 バイト未満) は、表の行内にインラインで保存される場合がある。
- **アウトオブライン・ストレージ:** 大きな LOB は専用の LOB セグメントに保存され、表の行にはポインタ (ロケータ) が保持される。
- **SecureFiles と BasicFiles:** SecureFiles (11g で導入) は現代的な LOB ストレージ形式である。重複排除、圧縮、暗号化をサポートしている。BasicFiles はレガシーな形式である。
- **LOB 索引:** 各アウトオブライン LOB 列には、関連する LOB 索引セグメントがある。

```sql
-- LOB セグメントとそのサイズを確認する
SELECT l.owner,
       l.table_name,
       l.column_name,
       l.segment_name,
       l.tablespace_name,
       l.securefile,
       ROUND(s.bytes / 1073741824, 3) AS lob_gb
FROM   dba_lobs     l
JOIN   dba_segments s ON l.owner = s.owner AND l.segment_name = s.segment_name
WHERE  l.owner NOT IN ('SYS','SYSTEM','DBSNMP')
ORDER BY s.bytes DESC
FETCH FIRST 30 ROWS ONLY;
```

### LOB 領域の問題と再利用

LOB には専用の HWM があり、表の `SHRINK` 操作の恩恵を受けない。LOB 領域を再利用するには、別のテクニックが必要である。

```sql
-- LOB セグメント領域と実際のデータを比較
SELECT l.owner,
       l.table_name,
       l.column_name,
       ROUND(s.bytes / 1048576, 1) AS alloc_mb,
       -- 統計情報から実際の LOB データ・サイズを見積もる
       ROUND(NVL(t.num_rows, 0) *
             (SELECT AVG(DBMS_LOB.getlength(lob_col))
              FROM   some_table) / 1048576, 1) AS est_data_mb
FROM   dba_lobs     l
JOIN   dba_segments s ON l.owner = s.owner AND l.segment_name = s.segment_name
JOIN   dba_tables   t ON l.owner = t.owner AND l.table_name = t.table_name;
```

**LOB 領域の再利用:**

```sql
-- SecureFiles の場合: 縮小を有効化 (オンライン)
ALTER TABLE documents MODIFY LOB (document_content) (SHRINK SPACE);

-- BasicFiles の場合: MOVE を使用する必要がある
ALTER TABLE documents MOVE LOB (document_content) STORE AS (TABLESPACE lob_ts);

-- あるいは、完全な再利用のために、ストレージ句を再定義する
-- (これにより新しい LOB セグメントが作成され、データがコピーされる)
ALTER TABLE documents MODIFY (document_content CLOB
    STORE AS SECUREFILE (
        TABLESPACE lob_ts
        ENABLE STORAGE IN ROW
        CHUNK 8192
        COMPRESS HIGH
    )
);
```

### LOB の保持と UNDO

LOB (特に BasicFiles) は、標準の UNDO 表領域を使用せず、LOB セグメント内に独自の UNDO メカニズムを持っている。これは以下のことを意味する。

```sql
-- LOB の保持設定を確認する
SELECT l.owner,
       l.table_name,
       l.column_name,
       l.securefile,
       l.retention_type,
       l.retention_value
FROM   dba_lobs l
WHERE  l.owner NOT IN ('SYS','SYSTEM')
ORDER BY l.owner, l.table_name;
```

BasicFiles LOB の場合、`RETENTION` 句は古い LOB バージョンをどれ期間保持するかを制御する。過度な保持は、LOB セグメントが予期せず肥大化する原因となる可能性がある。

---

## 一時領域の監視

一時表領域 (Temporary Tablespace) は、ソート操作、ハッシュ・ジョブ、グローバル一時表、およびパラレル・クエリ操作に使用される。永続表領域とは異なり、各操作の後に再利用されるが、セッションが強制終了されたり内部エラーが発生したりすると、「スタック」した状態になることがある。

### 現在の一時領域の使用状況

```sql
-- セッションごとの現在の一時領域使用状況
SELECT s.sid,
       s.serial#,
       s.username,
       s.program,
       s.status,
       ROUND(t.blocks * t.block_size / 1048576, 1) AS temp_mb,
       t.tablespace
FROM   v$sort_usage  t
JOIN   v$session     s ON t.session_addr = s.saddr
ORDER BY t.blocks DESC;
```

```sql
-- 使用済み vs 割り当て済みの一時領域の合計
SELECT tablespace_name,
       ROUND(total_blocks    * 8192 / 1073741824, 2) AS total_gb,
       ROUND(used_blocks     * 8192 / 1073741824, 2) AS used_gb,
       ROUND(free_blocks     * 8192 / 1073741824, 2) AS free_gb,
       ROUND(used_blocks * 100.0 / NULLIF(total_blocks, 0), 1) AS used_pct
FROM   v$temp_space_header
ORDER BY tablespace_name;
```

```sql
-- 現在、どのアクションが最も多く一時領域を使用しているか
SELECT s.sid,
       s.username,
       s.sql_id,
       ROUND(SUM(t.blocks * t.block_size) / 1048576, 1) AS temp_mb,
       MAX(t.sql_id) AS current_sql_id
FROM   v$tempseg_usage t
JOIN   v$session       s ON t.session_addr = s.saddr
GROUP BY s.sid, s.username, s.sql_id
ORDER BY temp_mb DESC;
```

```sql
-- 一時表領域のサイジング：過去の最大使用量を確認する (AWR が利用可能な場合)
SELECT snap_id,
       tablespace_name,
       ROUND(tablespace_size   / 1073741824, 2) AS total_gb,
       ROUND(tablespace_usedsize / 1073741824, 2) AS used_gb
FROM   dba_hist_tbspc_space_usage
WHERE  tablespace_name IN (
    SELECT tablespace_name FROM dba_tablespaces WHERE contents = 'TEMPORARY'
)
ORDER BY snap_id DESC
FETCH FIRST 100 ROWS ONLY;
```

### スタックした一時領域の再利用

セッションが切断された後も一時領域が解放されない場合 (特定のバグや RAC シナリオにおける既知の問題):

```sql
-- 孤立した一時セグメントの検索（対応するアクティブ・セッションがないもの）
SELECT t.tablespace, t.blocks, t.sql_id
FROM   v$sort_usage t
WHERE  NOT EXISTS (
    SELECT 1 FROM v$session s WHERE s.saddr = t.session_addr
);
```

```sql
-- 一時領域がスタックしている場合、一時表領域を縮小する (12c 以降)
ALTER TABLESPACE temp SHRINK SPACE KEEP 1G;

-- または、領域を強制的に再利用するためにリサイズする（最終手段）:
ALTER DATABASE TEMPFILE '/u01/oradata/orcl/temp01.dbf' RESIZE 2G;
```

```sql
-- 深刻なスタック状況の場合、一時表領域を削除して再作成する
-- (事前に一時領域を使用しているアクティブ・セッションがないことを確認すること)
CREATE TEMPORARY TABLESPACE temp2
TEMPFILE '/u01/oradata/orcl/temp02.dbf' SIZE 4G AUTOEXTEND ON NEXT 512M MAXSIZE 20G;

ALTER DATABASE DEFAULT TEMPORARY TABLESPACE temp2;

DROP TABLESPACE temp INCLUDING CONTENTS AND DATAFILES;

-- 必要に応じて temp2 を temp にリネームして戻す
```

---

## プロアクティブな領域管理：まとめ

### 日次領域チェック・スクリプト

```sql
-- マスター領域監視クエリ：毎日、または監視ツールで実行する
WITH ts_usage AS (
    SELECT m.tablespace_name,
           ROUND(m.tablespace_size    * t.block_size / 1073741824, 2) AS total_gb,
           ROUND(m.used_space         * t.block_size / 1073741824, 2) AS used_gb,
           ROUND(m.used_percent, 1) AS used_pct,
           t.contents,
           CASE WHEN EXISTS (
               SELECT 1 FROM dba_data_files d
               WHERE  d.tablespace_name = m.tablespace_name
               AND    d.autoextensible  = 'YES'
           ) THEN 'YES' ELSE 'NO' END AS has_autoextend
    FROM   dba_tablespace_usage_metrics m
    JOIN   dba_tablespaces              t ON m.tablespace_name = t.tablespace_name
)
SELECT tablespace_name,
       total_gb,
       used_gb,
       used_pct,
       contents,
       has_autoextend,
       CASE
           WHEN used_pct >= 95 THEN 'CRITICAL'
           WHEN used_pct >= 85 THEN 'WARNING'
           WHEN used_pct >= 75 THEN 'WATCH'
           ELSE 'OK'
       END AS status
FROM   ts_usage
ORDER BY used_pct DESC;
```

### 表領域の成長トレンド (AWR)

```sql
-- 過去 3 か月間の、週ごとの表領域成長
SELECT tablespace_name,
       TRUNC(snap_time, 'IW')                                     AS week_start,
       ROUND(MAX(tablespace_usedsize) * 8192 / 1073741824, 2)     AS max_used_gb,
       ROUND(MIN(tablespace_usedsize) * 8192 / 1073741824, 2)     AS min_used_gb,
       ROUND((MAX(tablespace_usedsize) - MIN(tablespace_usedsize))
             * 8192 / 1073741824, 3)                               AS growth_gb
FROM   dba_hist_tbspc_space_usage  u
JOIN   dba_hist_snapshot            s ON u.snap_id = s.snap_id AND u.dbid = s.dbid
WHERE  s.snap_time > SYSDATE - 90
GROUP BY tablespace_name, TRUNC(snap_time, 'IW')
ORDER BY tablespace_name, week_start;
```

---

## ベスト・プラクティス

1. **`DBA_TABLESPACE_USAGE_METRICS` を毎日監視し、使用率 80% でアラートを出すようにすること。** 95% になるまで待ってはいけない。自動拡張ファイルが最大サイズに達している可能性があり、大量ロードや暴走プロセスによって表領域は数分で満杯になることがある。

2. **現在の使用量だけでなく、自動拡張の制限を理解すること。** 使用率 50% でデータファイルの `MAXSIZE` が極めて小さい表領域は、使用率 85% で大きな自動拡張の余地がある表領域よりも、空き領域不足になるリスクが高い。

3. **HWM の無駄を分析する前に、必ず表の統計情報を収集すること。** `DBA_TABLES.NUM_ROWS` および `AVG_ROW_LEN` は、前回の `DBMS_STATS` 実行時の最新の状態に基づいている。古い統計情報は、無駄な領域の誤った計算につながる。

4. **営業時間内にはまず `SHRINK SPACE COMPACT` を実行し、トラフィックが少ない時間帯に `SHRINK SPACE` を実行すること。** COMPACT フェーズは、最小限のロックで重い処理 (行の移動) を行う。最終的な HWM リセット・フェーズは短時間で済む。

5. **`MOVE` 後には必ず索引を再構築すること。** これを忘れることは、本番環境での一般的なインシデントの原因である。使用不可能になった索引は、その表に対する DML で ORA-01502 エラーを引き起こす。

6. **一時表領域は、平均ではなくピーク時に合わせてサイジングすること。** AWR 履歴 (`DBA_HIST_TBSPC_SPACE_USAGE`) を使用して過去のピーク使用量を確認し、それに応じてサイズを決定する。月末のレポート処理などで一時表領域が満杯になると、操作が失敗する。

7. **本番環境で一時領域の問題を解決するために `DROP TABLESPACE` を使用するのは避けること。** まず、アクティブな一時セッションがないか常に確認すること。セッションが使用中に一時表領域を削除すると、インスタンスが不安定になる可能性がある。

---

## よくある間違いと回避策

**間違い: 使用済みサイズではなく、割り当て済みの総サイズを監視する。**
`DBA_DATA_FILES.BYTES` は割り当てられたファイル・サイズを示すが、これは実際のデータ量よりもはるかに大きい場合がある。正確な使用量と空き領域の数値を得るには、`DBA_TABLESPACE_USAGE_METRICS` を使用するか、`DBA_DATA_FILES` と `DBA_FREE_SPACE` を結合して確認すること。

**間違い: `DELETE` によって領域が即座に解放されると思い込む。**
`DELETE` は行を論理的に削除するが、HWM を下げたり表領域に領域を解放したりはしない。適切な場合は `TRUNCATE` (HWM をリセットする) を使用するか、大量削除後に `SHRINK/MOVE` を実行すること。

**間違い: 行移動の影響を考慮せずに表を縮小する。**
`ENABLE ROW MOVEMENT` は、縮小中に Oracle が行の ROWID を変更することを許可する。物理的な ROWID を外部でキャッシュしているアプリケーションは正常に動作しなくなる。行移動を有効にする前に、アプリケーションが ROWID を保持していないか確認すること。

**間違い: 領域予測において LOB セグメントを考慮に入れない。**
LOB セグメントは親表のセグメントの数倍のサイズになることがあり、独立して成長する。表領域の成長予測には、必ず LOB セグメントを含めること。

**間違い: 本番環境の自動拡張データファイルに `MAXSIZE UNLIMITED` を設定する。**
無制限の自動拡張は、ファイル・システム全体を埋め尽くし、OS とデータベースの両方を停止させる可能性がある。OS や他のアプリケーションのために余裕を持たせた `MAXSIZE` を常に設定すること。Oracle だけでなく、ファイル・システムも監視すること。

**間違い: UNDO 表領域を無視する。**
UNDO 表領域は動作が異なる。Oracle は保持期間を自動的に管理するが、非常に長時間実行されるトランザクションによって UNDO 表領域が急激に肥大化することがある。UNDO 表領域を領域監視に含め、適切な `UNDO_RETENTION` を設定すること。

---


## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 19c と 26ai では、リリース更新によってデフォルト設定や非推奨機能が異なる場合があるため、両方の環境をサポートする場合は構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database Administrator's Guide 19c — Managing Tablespaces](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tablespaces.html)
- [Oracle Database 19c Reference — DBA_TABLESPACE_USAGE_METRICS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_TABLESPACE_USAGE_METRICS.html)
- [Oracle Database 19c Reference — V$SORT_USAGE](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SORT_USAGE.html)
- [Oracle Database 19c SQL Language Reference — ALTER TABLE (SHRINK SPACE)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/ALTER-TABLE.html)
