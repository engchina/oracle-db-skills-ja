# Oracleの表領域設計

## 概要

**表領域 (Tablespace)** は、Oracle の論理的なストレージ・コンテナである。これは、割り当て、管理、およびバックアップの単位となる、1 つ以上のデータ・ファイルのグループに名前を付けたものである。すべての Oracle オブジェクト（表、索引、クラスタ、LOB）は、いずれかの表領域に属する。適切な表領域設計を行うことで、以下が可能になる：

- ワークロードのタイプごとに異なるストレージ・デバイスに I/O を分散・分離する
- バックアップとリカバリの簡素化（表領域レベルでの RMAN バックアップ）
- 領域の拡張と割当て動作の制御
- オブジェクト・グループの独立したトランスポート（移動）やオフライン管理
- 拡張時のデータ・ファイル同士の干渉防止

このガイドでは、表領域のタイプ、サイズ指定の戦略、ASSM と MSSM の選択、および本番環境の Oracle データベースに推奨されるマルチ表領域構成について説明する。

---

## 1. 表領域のタイプ

### 永続表領域 (Permanent Tablespaces)

表、索引、ビュー（のセグメント）、クラスタ、LOB などの永続的なデータベース・オブジェクトを格納する。表領域設計の大部分は、この永続表領域に焦点を当てる。

### 一時表領域 (Temporary Tablespaces)

セッション・レベルの一時セグメント（ソート処理、ハッシュ結合のワーク・エリア、グローバル一時表のデータ、一時 LOB など）を格納する。一時表領域はバックアップの必要がない。データベースの作成時や再作成時に常に再構築される。

```sql
-- 専用の一時表領域を作成
CREATE TEMPORARY TABLESPACE temp_01
    TEMPFILE '/u02/oradata/ORCL/temp01.dbf' SIZE 2G AUTOEXTEND ON NEXT 512M MAXSIZE 20G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

-- デフォルトの一時表領域として割り当て
ALTER DATABASE DEFAULT TEMPORARY TABLESPACE temp_01;

-- 一時表領域グループ (パラレル・クエリ用に複数の表領域をグループ化)
CREATE TABLESPACE GROUP temp_group;
ALTER TABLESPACE temp_01 TABLESPACE GROUP temp_group;
ALTER TABLESPACE temp_02 TABLESPACE GROUP temp_group;
ALTER DATABASE DEFAULT TEMPORARY TABLESPACE temp_group;
```

### Undo 表領域 (Undo Tablespace)

トランザクションのロールバック、読取り一貫性、および Oracle Flashback 機能に使用される Undo セグメントを格納する。`UNDO_MANAGEMENT = AUTO` の場合、Oracle によって自動的に管理される (AUM — 自動 Undo 管理)。

```sql
CREATE UNDO TABLESPACE undo_01
    DATAFILE '/u02/oradata/ORCL/undo01.dbf' SIZE 4G AUTOEXTEND ON NEXT 1G MAXSIZE 50G;

ALTER SYSTEM SET UNDO_TABLESPACE = undo_01;
ALTER SYSTEM SET UNDO_RETENTION = 900;  -- 15分 (秒単位で指定)
```

Undo サイズの見積もりの目安：

```
Undoサイズ (バイト) = UNDO_RETENTION (秒) x (DBブロックサイズ x 1秒あたりの変更ブロック数)
                   = UNDO_RETENTION x (SELECT value FROM v$parameter WHERE name='db_block_size')
                     x (SELECT undoblks/((last_analyzed - first_analyzed)*86400) FROM v$undostat)
```

---

## 2. Bigfile 表領域 vs Smallfile 表領域

### Smallfile 表領域 (従来型、デフォルト)

- 1 つの表領域につき最大 **1,022 個のデータ・ファイル**
- 各データ・ファイルは最大 **400万ブロック** (8K ブロックで 32 GB、32K ブロックで 128 GB)
- 表領域の最大サイズ: **約 32 PB** (1,022 ファイル x 各 32 TB (32K ブロック時))
- ファイル番号はデータベース全体の制限リソース (データベースあたり最大 65,534 ファイル)
- ファイル単位でのきめ細かなバックアップが可能

```sql
CREATE TABLESPACE users_data
    DATAFILE '/u01/oradata/ORCL/users_data01.dbf' SIZE 2G
             AUTOEXTEND ON NEXT 256M MAXSIZE 32G,
            '/u01/oradata/ORCL/users_data02.dbf' SIZE 2G
             AUTOEXTEND ON NEXT 256M MAXSIZE 32G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;
```

### Bigfile 表領域 (10g 以降)

- 1 つの表領域につき**ちょうど 1 つのデータ・ファイル**
- データ・ファイルは最大 **40億ブロック** (8K ブロックで 32 TB、32K ブロックで最大 128 TB)
- 表領域ごとに消費されるファイル番号は 1 つのみ
- 非常に巨大な表領域の管理を簡素化できる
- Bigfile 表領域の RMAN バックアップは 1 つの非常に巨大なファイルのバックアップとなる（I/O パラレル度が低下する可能性がある）
- 特定のストレージ構成での Oracle Managed Files (OMF) で必要とされる

```sql
CREATE BIGFILE TABLESPACE dw_facts_data
    DATAFILE '/u03/oradata/ORCL/dw_facts_data01.dbf' SIZE 100G
    AUTOEXTEND ON NEXT 10G MAXSIZE 10T
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;

-- Bigfile 表領域のサイズ変更 (データ・ファイルではなく表領域を直接指定可能)
ALTER TABLESPACE dw_facts_data RESIZE 200G;
```

### Bigfile vs Smallfile 選択ガイド

| 項目 | Smallfile | Bigfile |
|---|---|---|
| ファイル数 | 表領域あたり最大 1,022 | 表領域あたり 1 つ固定 |
| 単一ファイル・サイズ | 最大 約 128 TB | 最大 約 128 TB |
| ファイル番号の消費 | 表領域ごとに複数 | 表領域ごとに 1 つ |
| RMAN バックアップ並列度 | 高い (複数ファイルを並行してバックアップ) | 低い (単一ファイル) |
| 管理の容易さ | 追跡すべきファイルが多い | シンプル (1ファイルのみ) |
| 最適な用途 | OLTP、中規模テーブル、柔軟性 | 巨大な DW ファクト表、ASM 環境 |
| ASM との互換性 | 両方可能 | ASM 上で推奨 (ASM がストライピングを行うため) |

---

## 3. エクステント管理 (Extent Management)

エクステントは、セグメントに割り当てられる連続した Oracle ブロックのグループである。エクステント管理は、表領域内でエクステントをどのように追跡・割当てするかを決定する。

### ローカル管理表領域 (LMT) — 常にこれを使用すること

Oracle は、SYSTEM 表領域のデータ・ディクショナリではなく、表領域のデータ・ファイル自体に格納されたビットマップでエクステントの割当てを追跡する。これによりデータ・ディクショナリへの競合が排除される。Oracle 9i 以降、すべての表領域のデフォルトである。

```sql
-- UNIFORM SIZE: すべてのエクステントを同じサイズにする (DW やサイズが均一なオブジェクトに推奨)
CREATE TABLESPACE dw_idx
    DATAFILE '/u02/oradata/ORCL/dw_idx01.dbf' SIZE 10G AUTOEXTEND ON NEXT 1G MAXSIZE 100G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 8M;  -- すべてのエクステントが正確に 8M

-- AUTOALLOCATE: Oracle がエクステント・サイズを選択 (64K -> 1M -> 8M -> 64M...)
-- オブジェクト・サイズが混在する OLTP に推奨
CREATE TABLESPACE users_data
    DATAFILE '/u01/oradata/ORCL/users_data01.dbf' SIZE 4G AUTOEXTEND ON NEXT 512M MAXSIZE 50G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE;
```

**UNIFORM SIZE のガイドライン:**

| オブジェクト・タイプ | 推奨される UNIFORM SIZE |
|---|---|
| OLTP データ | 1M または AUTOALLOCATE |
| OLTP 索引 | 1M または AUTOALLOCATE |
| DW ファクト表 | 8M ～ 32M |
| DW ディメンション表 | 1M ～ 4M |
| LOB セグメント | 8M ～ 64M |
| Undo | 1M または AUTOALLOCATE |
| 一時 (Temporary) | 1M |

### ディクショナリ管理表領域 (DMT) — 廃止、使用厳禁

Oracle は、SYSTEM 表領域のデータ・ディクショナリ・表でエクステントの割当てを追跡する。`UET$` および `FET$` 表で深刻な競合が発生する。すべての新しい表領域はローカル管理にする必要がある。

---

## 4. セグメント領域管理: ASSM vs MSSM

セグメント領域管理は、セグメント内の個々のデータベース・ブロック**内部**の空き領域（新しい行を挿入する余地があるブロックはどれか）を Oracle がどのように追跡するかを制御する。

### 自動セグメント領域管理 (ASSM)

セグメント自体の内部にある複数レベルのビットマップを使用して、ブロックの空き領域を追跡する。Oracle 9i 以降のデフォルトであり、ほとんどの場合、これが正しい選択である。

```sql
CREATE TABLESPACE users_data
    DATAFILE '/u01/oradata/ORCL/users_data01.dbf' SIZE 4G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;  -- ASSM (デフォルト、推奨)
```

**ASSM の利点:**
- 高並行挿入が行われる表での「フリーリスト」競合を排除する
- `PCTUSED` パラメータは無視される（Oracle が自動的に管理する）
- `SHRINK SPACE` やオンライン・セグメント再編成をサポートする
- Oracle In-Memory、アドバンスト圧縮、およびその他の多くの機能で必須となる

### 手動セグメント領域管理 (MSSM)

挿入可能なブロックを追跡するために、空き領域のあるブロックのリンク・リストである**フリーリスト (freelists)** を使用する。各セグメントには、構成可能な数のフリーリスト (`FREELIST GROUPS`) がある。

```sql
CREATE TABLESPACE legacy_data
    DATAFILE '/u01/oradata/ORCL/legacy01.dbf' SIZE 1G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT MANUAL;  -- MSSM (レガシー)

-- MSSM では、PCTUSED と FREELISTS が重要となる：
CREATE TABLE LEGACY_TABLE (
    id   NUMBER,
    data VARCHAR2(100)
)
TABLESPACE legacy_data
PCTFREE   10  -- 更新用に 10% を予約
PCTUSED   60  -- 使用率が 60% を下回った時にブロックがフリーリストに戻る
STORAGE (FREELISTS 4 FREELIST GROUPS 2);  -- 4つのフリーリスト、2つのフリーリスト・グループ (RAC)
```

**MSSM が依然として現れる可能性のあるケース:**
- Oracle 8i/9i から移行された非常に古いデータベース
- MSSM の動作を必要とする特定のサードパーティ・アプリケーション
- 一時表領域 (ソート・セグメント用に内部で常に MSSM を使用する)

### ASSM vs MSSM の比較

| 機能 | ASSM | MSSM |
|---|---|---|
| 並行性 | 非常に優れている (ビットマップにより競合なし) | フリーリストで競合が発生する可能性がある |
| PCTUSED | 無視される | 有効なパラメータ |
| PCTFREE | 尊重される | 尊重される |
| SHRINK SPACE | サポートされる | サポートされない |
| アドバンスト圧縮 | サポートされる | サポートされない |
| インメモリー (In-Memory) | サポートされる | サポートされない |
| ブロック使用率の可視化 | `DBMS_SPACE.SPACE_USAGE` | `DBMS_SPACE.FREE_BLOCKS` |
| 推奨 | はい (すべての新規作成) | いいえ (レガシーのみ) |

---

## 5. 推奨される本番用マルチ表領域構成

適切に設計された本番用 Oracle データベースは、I/O プロファイル、成長率、およびリカバリ要件によってオブジェクトを分離する。以下の構成は、OLTP およびデータウェアハウス・ワークロードをカバーする。

### OLTP 本番構成例

```sql
-- ============================================================
-- SYSTEM および SYSAUX: Oracle によって管理される。ユーザー・オブジェクトを置いてはいけない
-- ============================================================

-- ============================================================
-- UNDO: データベース・インスタンスごとに 1 つ (RAC の場合はインスタンス数分用意)
-- ============================================================
CREATE UNDO TABLESPACE undo_01
    DATAFILE '/u02/oradata/ORCL/undo_01_01.dbf' SIZE 8G
             AUTOEXTEND ON NEXT 1G MAXSIZE 100G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE;

-- ============================================================
-- TEMP: データベースごとに 1 つ (または RAC/パラレル・クエリ用の一時表領域グループ)
-- ============================================================
CREATE TEMPORARY TABLESPACE temp_01
    TEMPFILE '/u02/oradata/ORCL/temp_01_01.dbf' SIZE 4G
             AUTOEXTEND ON NEXT 512M MAXSIZE 50G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

ALTER DATABASE DEFAULT TEMPORARY TABLESPACE temp_01;

-- ============================================================
-- アプリケーション・データ: スキーマまたはアプリケーション・モジュールごとに 1 つの表領域
-- 更新頻度の高い表は、個別のバックアップを可能にするため表領域を分ける
-- ============================================================
CREATE TABLESPACE app_data
    DATAFILE '/u03/oradata/ORCL/app_data_01.dbf' SIZE 10G
             AUTOEXTEND ON NEXT 1G MAXSIZE 500G,
            '/u03/oradata/ORCL/app_data_02.dbf' SIZE 10G
             AUTOEXTEND ON NEXT 1G MAXSIZE 500G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;

-- LOB セグメント用表領域 — 行データとは分離
CREATE TABLESPACE app_lob
    DATAFILE '/u04/oradata/ORCL/app_lob_01.dbf' SIZE 20G
             AUTOEXTEND ON NEXT 2G MAXSIZE 1T
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 8M  -- LOB の効率性のために UNIFORM を使用
    SEGMENT SPACE MANAGEMENT AUTO;

-- ============================================================
-- アプリケーション索引: I/O 分離のため、常にデータとは別にする
-- ============================================================
CREATE TABLESPACE app_idx
    DATAFILE '/u05/oradata/ORCL/app_idx_01.dbf' SIZE 5G
             AUTOEXTEND ON NEXT 512M MAXSIZE 200G,
            '/u05/oradata/ORCL/app_idx_02.dbf' SIZE 5G
             AUTOEXTEND ON NEXT 512M MAXSIZE 200G
    EXTENT MANAGEMENT LOCAL AUTOALLOCATE
    SEGMENT SPACE MANAGEMENT AUTO;

-- ============================================================
-- 読取り専用 (Read-Only): 変更されることのない履歴/アーカイブ・データ
-- ============================================================
CREATE TABLESPACE app_archive
    DATAFILE '/u06/oradata/ORCL/app_archive_01.dbf' SIZE 50G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 8M
    SEGMENT SPACE MANAGEMENT AUTO;
-- データのロード完了後:
ALTER TABLESPACE app_archive READ ONLY;  -- 誤った変更を防ぎ、リカバリ対象から外す
```

### データウェアハウス (DW) 構成例

```sql
-- ============================================================
-- DW データ: ファクト表とディメンション表で表領域を分ける
-- ============================================================
CREATE BIGFILE TABLESPACE dw_fact_data
    DATAFILE '/u07/oradata/ORCL/dw_fact_data_01.dbf' SIZE 200G
             AUTOEXTEND ON NEXT 20G MAXSIZE 10T
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 32M  -- 巨大なファクト・セグメント用に大きな UNIFORM サイズ
    SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE dw_dim_data
    DATAFILE '/u07/oradata/ORCL/dw_dim_data_01.dbf' SIZE 10G
             AUTOEXTEND ON NEXT 1G MAXSIZE 100G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 4M
    SEGMENT SPACE MANAGEMENT AUTO;

-- ============================================================
-- DW 索引: ファクト表に対するビットマップ索引や B-tree 索引
-- ============================================================
CREATE TABLESPACE dw_idx
    DATAFILE '/u08/oradata/ORCL/dw_idx_01.dbf' SIZE 20G
             AUTOEXTEND ON NEXT 2G MAXSIZE 500G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 8M
    SEGMENT SPACE MANAGEMENT AUTO;

-- ============================================================
-- ステージング (Staging): ETL ステージング・エリア — バックアップ不要（再作成可能）
-- ============================================================
CREATE TABLESPACE etl_stage
    DATAFILE '/u09/oradata/ORCL/etl_stage_01.dbf' SIZE 50G
             AUTOEXTEND ON NEXT 5G MAXSIZE 1T
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 16M
    SEGMENT SPACE MANAGEMENT AUTO;

-- ============================================================
-- DW TEMP: 分析クエリ用の独立した巨大な一時表領域
-- ============================================================
CREATE TEMPORARY TABLESPACE dw_temp
    TEMPFILE '/u09/oradata/ORCL/dw_temp_01.dbf' SIZE 50G
             AUTOEXTEND ON NEXT 5G MAXSIZE 500G
    EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1M;

-- DW 分析ユーザーに割り当て
ALTER USER dw_analyst_user  TEMPORARY TABLESPACE dw_temp;
ALTER USER etl_process_user TEMPORARY TABLESPACE dw_temp;
```

---

## 6. 表領域のサイズ指定 (Sizing)

### 初期サイズ指定の戦略

最初から表領域を将来の最大予測サイズで作成してはいけない。`AUTOEXTEND` を、現実的な `MAXSIZE` 制限とともに使用する：

```sql
-- 良いパターン: 小さな初期サイズ、制御された自動拡張、明確な上限設定
DATAFILE '/u01/oradata/ORCL/app_data_01.dbf'
    SIZE 1G                    -- 初期サイズ: すぐに割り当てられフォーマットされる
    AUTOEXTEND ON              -- 拡張を許可
    NEXT 512M                  -- 1回につき 512M ずつ拡張
    MAXSIZE 100G               -- 1ファイルあたり 100G を超えない
```

### サイズ見積もり式

**表セグメントのサイズ見積もり:**

```sql
-- 作成前にセグメント・サイズを見積もる
-- DBMS_SPACE.CREATE_TABLE_COST は OUT パラメータを持つプロシージャであるため、
-- SELECT 文ではなく PL/SQL ブロック経由で呼び出す。
DECLARE
    v_used_bytes  NUMBER;
    v_alloc_bytes NUMBER;
BEGIN
    DBMS_SPACE.CREATE_TABLE_COST(
        tablespace_name => 'USERS_DATA',
        avg_row_size    => 250,      -- 1行あたりの平均バイト数
        row_count       => 10000000, -- 予測行数
        pct_free        => 10,
        used_bytes      => v_used_bytes,
        alloc_bytes     => v_alloc_bytes
    );
    DBMS_OUTPUT.PUT_LINE('Used bytes:  ' || v_used_bytes);
    DBMS_OUTPUT.PUT_LINE('Alloc bytes: ' || v_alloc_bytes);
END;
/
```

**既存の表領域使用率の監視:**

```sql
SELECT
    ts.tablespace_name,
    ts.block_size,
    ROUND(SUM(df.bytes)          / 1073741824, 2) AS total_gb,
    ROUND(SUM(fs.free_space)     / 1073741824, 2) AS free_gb,
    ROUND((1 - SUM(fs.free_space) / SUM(df.bytes)) * 100, 1) AS used_pct,
    ts.status,
    ts.contents,
    ts.extent_management,
    ts.segment_space_management
FROM
    dba_tablespaces ts
    JOIN dba_data_files df ON ts.tablespace_name = df.tablespace_name
    LEFT JOIN (
        SELECT tablespace_name, SUM(bytes) AS free_space
        FROM   dba_free_space
        GROUP  BY tablespace_name
    ) fs ON ts.tablespace_name = fs.tablespace_name
GROUP BY
    ts.tablespace_name, ts.block_size, ts.status, ts.contents,
    ts.extent_management, ts.segment_space_management
ORDER BY
    used_pct DESC NULLS LAST;
```

---

## 7. 表領域のメンテナンス

### データ・ファイルの追加

```sql
-- 既存の表領域に新しいデータ・ファイルを追加
ALTER TABLESPACE app_data ADD DATAFILE
    '/u03/oradata/ORCL/app_data_03.dbf' SIZE 10G
    AUTOEXTEND ON NEXT 1G MAXSIZE 500G;
```

### データ・ファイルのサイズ変更

```sql
-- 特定のデータ・ファイルのサイズを変更
ALTER DATABASE DATAFILE '/u03/oradata/ORCL/app_data_01.dbf' RESIZE 20G;

-- 自動拡張の設定変更
ALTER DATABASE DATAFILE '/u03/oradata/ORCL/app_data_01.dbf'
    AUTOEXTEND ON NEXT 2G MAXSIZE 200G;
```

### 領域の回収 — セグメントの縮小 (Shrink)

```sql
-- 行移動を有効化 (SHRINK のために必須)
ALTER TABLE APP_SCHEMA.LARGE_TABLE ENABLE ROW MOVEMENT;

-- セグメントを圧縮し、HWM (High Water Mark) をリセット (2フェーズ方式)
ALTER TABLE APP_SCHEMA.LARGE_TABLE SHRINK SPACE COMPACT;  -- フェーズ1: オンラインで行を移動
ALTER TABLE APP_SCHEMA.LARGE_TABLE SHRINK SPACE;           -- フェーズ2: HWM をリセット

-- または一括で実行 (HWM リセット時に短いロックが発生する)
ALTER TABLE APP_SCHEMA.LARGE_TABLE SHRINK SPACE CASCADE;  -- 表とすべての索引を縮小
```

### テーブルを別の表領域に移動

```sql
-- オンライン移動 (12c 以降、ENABLE ROW MOVEMENT が必要)
ALTER TABLE APP_SCHEMA.ORDERS
    MOVE TABLESPACE new_tablespace
    ONLINE;

-- オフライン移動 (索引が無効化されるが高速)
ALTER TABLE APP_SCHEMA.ORDERS MOVE TABLESPACE new_tablespace;

-- オフライン移動後に無効化された索引を再構築
SELECT 'ALTER INDEX ' || owner || '.' || index_name || ' REBUILD TABLESPACE app_idx;'
FROM   dba_indexes
WHERE  table_name = 'ORDERS'
AND    status = 'UNUSABLE';
```

---

## 8. デフォルト表領域の割当て

```sql
-- スキーマ・レベルでのデフォルト設定
ALTER USER app_owner
    DEFAULT   TABLESPACE app_data
    TEMPORARY TABLESPACE temp_01
    QUOTA UNLIMITED ON app_data
    QUOTA UNLIMITED ON app_idx
    QUOTA UNLIMITED ON app_lob;

-- クォータ (割当て制限) 管理
ALTER USER app_owner QUOTA 50G ON app_data;  -- 50G に制限
ALTER USER app_owner QUOTA 0   ON users;      -- USERS 表領域の使用を禁止
```

---

## 9. ベスト・プラクティス

- **データと索引を分離する。** 索引の I/O パターン（ランダム・アクセス）は表の I/O パターン（シーケンシャル・スキャン）と異なるため、これらを別の表領域（理想的には別のストレージ、または ASM ディスク・グループ）に分けることでスループットが向上する。
- **OLTP データと DW データを分離する。** 圧縮設定、エクステント・サイズ、バックアップ頻度が異なるため、分離は必須である。
- **SYSTEM や SYSAUX にアプリケーション・オブジェクトを格納しない。** SYSTEM の破損や領域不足はデータベース全体を停止させる。
- **AUTOEXTEND には現実的な MAXSIZE を設定する。** ファイル・システム上で無制限に自動拡張させると、ディスクがいっぱいになった際にデータベースがクラッシュする。ファイル・システムの空き容量に少なくとも 20% の余裕を持たせるように `MAXSIZE` を設定する。
- **DW 表領域には UNIFORM SIZE を使用する。** 均等なエクステント・サイズはフル・スキャン・パフォーマンス（連続 I/O）を向上させ、領域管理を簡素化する。オブジェクト・サイズが大きく異なる OLTP 表領域には `AUTOALLOCATE` を使用する。
- **LOB セグメントは専用の表領域に配置する。** LOB は行データとは独立して成長するため、分離することで領域競合を防ぎ、監視が容易になる。
- **履歴・アーカイブ・データには読取り専用表領域を作成する。** 読取り専用表領域は、最終的なデータ・ファイルのチェック・ポイント移行はバックアップが不要になり、バックアップ時間を劇的に短縮できる。
- **表領域の使用率をプロアクティブに監視する。** Oracle のしきい値やカスタム・アラートを 75% や 85% に設定する。`ORA-01653: 表を拡張できません` が発生するまで待ってはいけない。

```sql
-- 組み込みの領域アラートしきい値管理
EXEC DBMS_SERVER_ALERT.SET_THRESHOLD(
    metrics_id        => DBMS_SERVER_ALERT.TABLESPACE_PCT_FULL,
    warning_operator  => DBMS_SERVER_ALERT.OPERATOR_GE,
    warning_value     => '75',
    critical_operator => DBMS_SERVER_ALERT.OPERATOR_GE,
    critical_value    => '90',
    observation_period => 1,
    consecutive_occurrences => 1,
    instance_name     => NULL,
    object_type       => DBMS_SERVER_ALERT.OBJECT_TYPE_TABLESPACE,
    object_name       => 'APP_DATA'
);
```

---

## 10. よくある間違いとその回避方法

### 間違い 1: ユーザー・オブジェクトを SYSTEM や USERS に格納する

新しいスキーマのデフォルト表領域は、明示的に設定しない限り `USERS` になる。`USERS` 表領域、特に `SYSTEM` 表領域にアプリケーション・データを置いてはいけない。

### 間違い 2: MAXSIZE なしの無制限 AUTOEXTEND

単一の暴走クエリが過剰な一時セグメントを生成したり、無限インサート・ループが発生したりすると、ファイル・システムを使い果たし、データベース・インスタンス全体がクラッシュする。必ず `MAXSIZE` を設定すること。

### 間違い 3: ディクショナリ管理表領域 (DMT) を使用する

DMT は `UET$` および `FET$` システム表でのシリアル化（競合）を引き起こす。すべての新しい表領域はローカル管理にする必要がある。古い DMT が残っている場合は移行すること：

```sql
EXEC DBMS_SPACE_ADMIN.TABLESPACE_MIGRATE_TO_LOCAL('OLD_DMT_TABLESPACE');
```

### 間違い 4: 高並行システムで新規テーブルに MSSM を使用する

MSSM のフリーリスト競合は、スケーラビリティのボトルネックとして知られている。新しい表領域には常に ASSM (`SEGMENT SPACE MANAGEMENT AUTO`) を使用すること。

### 間違い 5: 断片化 (Fragmentation) の放置

頻繁な削除/作成サイクルを繰り返すと、空き領域が表領域内で断片化する。断片化を監視し、隣接する空きエクステントを結合するには `ALTER TABLESPACE app_data COALESCE` を実行する。

### 間違い 6: インデックスをテーブルと同じ表領域に配置する

インデックス検索を伴うフル表スキャンが発生すると、Oracle は表と索引の両方を読み取る必要がある。これらが同じデータ・ファイルを共有していると、I/O がボトルネックになる。データと索引の表領域を異なるストレージ・パスに分離すること。

### 間違い 7: Undo 表領域のサイズ不足

サイズ不足の Undo 表領域は、長時間実行クエリにおける `ORA-01555: snapshot too old` エラーの原因となり、Flashback Query が目的の期間機能しなくなる。

---

## 11. Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。

| 機能 | バージョン |
|---|---|
| ローカル管理表領域のデフォルト化 | 9i |
| ASSM のデフォルト化 | 9i |
| Bigfile 表領域 | 10g |
| オンライン・セグメント縮小 | 10g |
| Undo アドバイザ | 10g |
| 表領域の暗号化 (TDE) | 10g R2 |
| オンラインでの表移動 | 12c R2 (12.2) |
| 自動 Bigfile 設定 (Autonomous DB) | 19c (ADB) |
| インメモリー表領域 (IM_IMCU_POOL) | 12c |

---

## ソース

- [Oracle Database 23ai Administrator's Guide — Managing Tablespaces](https://docs.oracle.com/en/database/oracle/oracle-database/23/admin/managing-tablespaces.html)
- [Oracle Database 23ai Concepts — Logical Storage Structures](https://docs.oracle.com/en/database/oracle/oracle-database/23/cncpt/logical-storage-structures.html)
- [Oracle Database 19c Administrator's Guide — Managing Tablespaces](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tablespaces.html)
- [Oracle Database 23ai PL/SQL Packages and Types Reference — DBMS_SPACE](https://docs.oracle.com/en/database/oracle/oracle-database/23/arpls/DBMS_SPACE.html)
- [Oracle Database 19c PL/SQL Packages and Types Reference — DBMS_SERVER_ALERT](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SERVER_ALERT.html)
- [Oracle Bigfile Tablespace (oracle-faq.com)](https://www.orafaq.com/wiki/Bigfile_tablespace)
- [Oracle Database 12c R2 — Online Table Move (oracle-base.com)](https://oracle-base.com/articles/12c/online-move-table-12cr2)
- [Oracle ALTER TABLE SHRINK SPACE — Online Segment Shrink (oracle-base.com)](https://oracle-base.com/articles/misc/alter-table-shrink-space-online)
