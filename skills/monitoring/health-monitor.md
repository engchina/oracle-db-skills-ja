# ヘルス・モニターとデータベース・アドバイザ

## 概要

Oracle のヘルス・モニター (Health Monitor) は、データベース・カーネルに組み込まれたフレームワークであり、データベースの構造的な整合性に対して、自動的、オンデマンド、またはスケジュールされた診断チェックを提供する。これは、リアクティブなアラート・ログ監視に対するプロアクティブな対応策である。エラーの発生を待つのではなく、ヘルス・モニターはデータベースの内部を能動的に調査し、データ損失やダウンタイムの原因となる前に問題を表面化させる。

ヘルス・モニターには `DBMS_HM` PL/SQL パッケージを介してアクセスし、その結果は ADR に保存される。また、Oracle のアドバイザ・フレームワーク・コンポーネント（SQL チューニング・アドバイザ、セグメント・アドバイザ、メモリー・アドバイザなど）とも密接に関連している。これらは構造的な整合性ではなく、パフォーマンスや領域管理に関する実行可能な推奨事項を提供する。

---

## ヘルス・モニターのアーキテクチャ

### コンポーネント

- **ヘルス・チェック (Health Monitor Checks):** データベース・スタックの特定の層を対象とした、事前定義済みの診断ルーチン。
- **検出結果 (Findings):** 各チェックの実行出力。見つかった内容（正常、失敗、またはアドバイザリ）を記述した構造化レコード。
- **推奨事項 (Recommendations):** 検出結果に基づいて Oracle が提案するアクション（修復スクリプト、SQL、RMAN コマンド）。
- **DBMS_HM:** チェックを実行し、結果を取得するための PL/SQL API。
- **ADR 統合:** すべての検出結果と推奨事項は ADR に保存され、`V$HM_*` ビューおよび `adrci` を介してアクセス可能である。

### ヘルス・モニター用 V$ ビュー

| ビュー名 | 説明 |
|------|-------------|
| `V$HM_CHECK` | 利用可能なすべてのヘルス・チェックとその現在のステータス |
| `V$HM_CHECK_PARAM` | 各チェックが受け入れるパラメータ |
| `V$HM_RUN` | 開始/終了時間とステータスを含む、過去のチェック実行履歴 |
| `V$HM_FINDING` | 各チェック実行によって生成された検出結果 |
| `V$HM_RECOMMENDATION` | 検出結果に基づく推奨事項 |
| `V$HM_INFO` | 検出結果に関する追加の情報詳細 |

---

## 組み込みヘルス・チェック

### チェック項目のリストと説明

```sql
-- 利用可能なすべてのヘルス・チェックを表示する
SELECT name,
       description,
       internal_check,
       offline_capable
FROM   v$hm_check
ORDER BY name;
```

#### DB 構造整合性チェック (DB Structure Integrity Check)

データベースの制御ファイル、データファイル・ヘッダー、および REDO ログ・ヘッダーの整合性を検証する。制御ファイルにリストされているすべてのファイルが存在し、アクセス可能で、一貫した SCN を持っていることを確認する。

- **使用場面:** クラッシュ後、バックアップからのリストア後、制御ファイルの破損が疑われる場合
- **オフライン実行の可否:** 可 (マウント・モードで実行可能)
- **実行時間:** 短い (数秒から数分)

#### データ・ブロック整合性チェック (Data Block Integrity Check)

データ・ブロックをスキャンして、論理的な破損（不正なチェックサム、無効なブロック・タイプ、オブジェクト ID の不一致）がないかを確認する。デフォルトですべてのブロックをチェックするわけではなく、疑わしいと特定されたブロックを対象とする。

- **使用場面:** ORA-01578 (ブロック破損) 発生後、ストレージ故障後
- **オフライン実行の可否:** 否 (データベースがオープンしている必要がある)
- **実行時間:** 範囲によって異なる

#### REDO 整合性チェック (Redo Integrity Check)

REDO ログ・ファイルとアーカイブ REDO ログの完全性と読み取り可能性を検証する。リカバリを妨げる可能性のあるギャップや読み取り不能な REDO を特定する。

- **使用場面:** メディア・リカバリの前後、REDO 関連の ORA- エラーの調査時
- **オフライン実行の可否:** 可

#### UNDO セグメント整合性チェック (Undo Segment Integrity Check)

システムで参照されているすべての UNDO セグメントがアクセス可能で一貫していることを検証する。トランザクションのロールバックに関連する ORA-01578 または ORA-00600 エラーの原因となる UNDO 破損を検出する。

- **使用場面:** 予期しないシャットダウン後、ORA-01555 またはロールバック・エラーの調査時
- **オフライン実行の可否:** 否

#### トランザクション整合性チェック (Transaction Integrity Check)

特定のトランザクションを調査して、その UNDO レコードが完全で一貫しているかどうかを検証する。特定のトランザクションに破損の疑いがある場合に役立つ。

- **使用場面:** 特定のトランザクション ID の重点的な調査
- **パラメータ:** `TXN_ID` パラメータが必要
- **オフライン実行の可否:** 否

#### ディクショナリ整合性チェック (Dictionary Integrity Check)

すべてのデータベース・オブジェクトのマスター・カタログである Oracle データ・ディクショナリの内部的な一貫性を検証する。ディクショナリの破損は、すべてのデータベース操作にわたって連鎖的な障害を引き起こす可能性がある。

- **使用場面:** パッチ適用後、DDL 操作の失敗後、`dict_` 引数を含む ORA-00600 エラーの発生後
- **オフライン実行の可否:** 否
- **実行時間:** 大規模なデータベースでは時間がかかる場合がある

---

## ヘルス・チェックの実行

### 基本構文

```sql
-- 必須項目：チェック名と実行名
BEGIN
    DBMS_HM.run_check(
        check_name => 'DB Structure Integrity Check',
        run_name   => 'MY_STRUCT_CHECK_20260306'
    );
END;
/
```

### すべての主要なチェックの実行

```sql
-- DB 構造整合性チェック
BEGIN
    DBMS_HM.run_check(
        check_name => 'DB Structure Integrity Check',
        run_name   => 'STRUCT_' || TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MISS')
    );
END;
/

-- データ・ブロック整合性チェック（特定のファイルとブロックを対象）
BEGIN
    DBMS_HM.run_check(
        check_name => 'Data Block Integrity Check',
        run_name   => 'BLOCK_' || TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MISS'),
        input_params => DBMS_HM.get_run_param('FILE_ID', '5')
                     || DBMS_HM.get_run_param('BLOCK_ID', '128')
    );
END;
/

-- REDO 整合性チェック
BEGIN
    DBMS_HM.run_check(
        check_name => 'Redo Integrity Check',
        run_name   => 'REDO_' || TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MISS')
    );
END;
/

-- UNDO セグメント整合性チェック
BEGIN
    DBMS_HM.run_check(
        check_name => 'Undo Segment Integrity Check',
        run_name   => 'UNDO_' || TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MISS')
    );
END;
/

-- ディクショナリ整合性チェック
BEGIN
    DBMS_HM.run_check(
        check_name => 'Dictionary Integrity Check',
        run_name   => 'DICT_' || TO_CHAR(SYSDATE, 'YYYYMMDD_HH24MISS')
    );
END;
/
```

### パラメータの受渡し

```sql
-- 各チェックで利用可能なパラメータを確認する
SELECT c.name AS check_name,
       p.name AS param_name,
       p.description,
       p.default_value
FROM   v$hm_check c
JOIN   v$hm_check_param p ON c.id = p.check_id
ORDER BY c.name, p.name;
```

```sql
-- トランザクション整合性チェックにはトランザクション ID が必要
-- まず調査したいトランザクション ID を探す
SELECT xid, status
FROM   v$transaction;

-- パラメータを指定して実行
BEGIN
    DBMS_HM.run_check(
        check_name   => 'Transaction Integrity Check',
        run_name     => 'TXN_CHECK_20260306',
        input_params => DBMS_HM.get_run_param('TXN_ID', '0500.010.000003e8')
    );
END;
/
```

### 実行中のチェックの監視

ヘルス・チェックはデフォルトで非同期に実行される。完了を確認するにはポーリングを行う：

```sql
-- 実行ステータスの確認
-- V$HM_RUN の実行識別子カラムは NAME（RUN_NAME ではない）
-- STATUS の値：INITIAL, EXECUTING, INTERRUPTED, TIMEDOUT, CANCELLED, COMPLETED, ERROR
-- V$HM_RUN には NUM_FINDINGS カラムはないため、NUM_INCIDENT を使用する
SELECT r.name          AS run_name,
       r.check_name,
       r.status,
       r.start_time,
       r.end_time,
       ROUND((CAST(r.end_time AS DATE) - CAST(r.start_time AS DATE)) * 86400, 1) AS duration_sec,
       r.num_incident
FROM   v$hm_run r
ORDER BY r.start_time DESC
FETCH FIRST 20 ROWS ONLY;
```

---

## 検出結果と推奨事項の表示

### 検出結果

```sql
-- 直近の実行によるすべての検出結果
SELECT r.run_name,
       r.check_name,
       f.type,           -- FAILURE, WARNING, ADVISORY
       f.status,
       f.description,
       f.repair_sql
FROM   v$hm_run     r
JOIN   v$hm_finding f ON r.id = f.run_id
ORDER BY r.start_time DESC, f.type;
```

```sql
-- FAILURE（失敗）と WARNING（警告）のみ（アドバイザリを除く）
SELECT r.run_name,
       r.check_name,
       f.type,
       f.description,
       f.repair_sql
FROM   v$hm_run     r
JOIN   v$hm_finding f ON r.id = f.run_id
WHERE  f.type IN ('FAILURE', 'WARNING')
ORDER BY r.start_time DESC;
```

```sql
-- 検出結果に関連付けられた推奨事項
SELECT r.run_name,
       f.description AS finding,
       f.type        AS finding_type,
       rec.type      AS recommendation_type,
       rec.description AS recommendation,
       rec.repair_script
FROM   v$hm_run            r
JOIN   v$hm_finding        f   ON r.id = f.run_id
JOIN   v$hm_recommendation rec ON f.id = rec.finding_id
ORDER BY r.start_time DESC;
```

### HTML レポートの生成

`DBMS_HM` は人間が読みやすい HTML レポートを生成できる：

```sql
-- 特定の実行に関するレポートを生成
DECLARE
    v_report CLOB;
BEGIN
    v_report := DBMS_HM.get_run_report('MY_STRUCT_CHECK_20260306');
    -- ファイルに書き出す (UTL_FILE の設定が必要) か、または表示する
    DBMS_OUTPUT.put_line(DBMS_LOB.SUBSTR(v_report, 32767, 1));
END;
/
```

より実用的な `adrci` を使用した手法：

```bash
# adrci でヘルス・モニターの検出結果を表示
adrci> SHOW HM_FINDING

# 特定の実行に関する検出結果を表示
adrci> SHOW HM_FINDING -P "RUN_NAME = 'MY_STRUCT_CHECK_20260306'"

# 推奨事項を表示
adrci> SHOW HM_RECOMMENDATION
```

---

## Oracle アドバイザ

Oracle アドバイザ・フレームワークは、構造的なヘルス・モニター・チェックとは異なる、ワークロードに基づいた推奨事項を提供する。監視と診断に関連する主なアドバイザは、SQL チューニング・アドバイザ、セグメント・アドバイザ、およびメモリー・アドバイザである。

### SQL チューニング・アドバイザ

SQL チューニング・アドバイザは、1 つ以上の SQL ステートメントを分析し、プロファイル・ベースのチューニング、新しい索引、統計情報の収集、または SQL の再構築を推奨する。

#### SQL チューニング・アドバイザの実行

```sql
-- 特定の SQL_ID に対するチューニング・タスクを作成
DECLARE
    v_task_name VARCHAR2(100);
BEGIN
    v_task_name := DBMS_SQLTUNE.create_tuning_task(
        sql_id      => 'abc123def456g',
        scope       => DBMS_SQLTUNE.scope_comprehensive,
        time_limit  => 300,  -- 秒
        task_name   => 'TUNE_ABC123_20260306',
        description => 'AWR からの top CPU クエリのチューニング'
    );
    DBMS_OUTPUT.put_line('Task created: ' || v_task_name);
END;
/

-- タスクを実行
BEGIN
    DBMS_SQLTUNE.execute_tuning_task(task_name => 'TUNE_ABC123_20260306');
END;
/

-- ステータスを確認
SELECT task_name, status, advisor_name
FROM   dba_advisor_tasks
WHERE  task_name = 'TUNE_ABC123_20260306';

-- 推奨レポートを表示
SELECT DBMS_SQLTUNE.report_tuning_task('TUNE_ABC123_20260306') AS report
FROM   dual;
```

#### AWR トップ SQL の自動チューニング

```sql
-- AWR からのトップ SQL セット (STS) に対するチューニング・タスクを作成
DECLARE
    v_task_name VARCHAR2(100);
BEGIN
    -- STS の作成
    DBMS_SQLTUNE.create_sqlset(sqlset_name => 'TOP_AWR_SQL');

    DBMS_SQLTUNE.load_sqlset(
        sqlset_name  => 'TOP_AWR_SQL',
        populate_cursor => DBMS_SQLTUNE.select_workload_repository(
            begin_snap => 100,
            end_snap   => 120,
            basic_filter => 'elapsed_time > 10000000',  -- > 10 秒
            ranking_measure1 => 'elapsed_time',
            result_limit => 20
        )
    );

    v_task_name := DBMS_SQLTUNE.create_tuning_task(
        sqlset_name => 'TOP_AWR_SQL',
        task_name   => 'TUNE_TOP_20_AWR'
    );
    DBMS_SQLTUNE.execute_tuning_task('TUNE_TOP_20_AWR');
END;
/
```

#### SQL プロファイルの受入れ

```sql
-- チューニング・タスクからの推奨事項をすべて受け入れる
BEGIN
    DBMS_SQLTUNE.accept_sql_profile(
        task_name  => 'TUNE_ABC123_20260306',
        force_match => TRUE  -- SQL テキストがわずかに異なる場合でも適用（バインド非依存）
    );
END;
/
```

### セグメント・アドバイザ

セグメント・アドバイザは、再利用可能な領域があるセグメント（高水位線以上のブロックを再利用するために縮小できる表や索引）を特定する。

```sql
-- 特定のスキーマに対してセグメント・アドバイザを実行
DECLARE
    v_task_name  VARCHAR2(100) := 'SEG_ADV_' || TO_CHAR(SYSDATE, 'YYYYMMDD');
    v_object_id  NUMBER;
BEGIN
    DBMS_ADVISOR.create_task(
        advisor_name => 'Segment Advisor',
        task_name    => v_task_name
    );

    DBMS_ADVISOR.create_object(
        task_name    => v_task_name,
        object_type  => 'SCHEMA',
        attr1        => 'SCOTT',  -- スキーマ名
        object_id    => v_object_id
    );

    DBMS_ADVISOR.set_task_parameter(
        task_name    => v_task_name,
        parameter    => 'RECOMMEND_ALL',
        value        => 'TRUE'
    );

    DBMS_ADVISOR.execute_task(task_name => v_task_name);
END;
/

-- セグメント・アドバイザの検出結果を表示
SELECT o.owner,
       o.object_name,
       o.object_type,
       f.message,
       TO_NUMBER(f.more_info) AS reclaimable_mb
FROM   dba_advisor_objects      o
JOIN   dba_advisor_findings     f ON o.task_id = f.task_id AND o.object_id = f.object_id
JOIN   dba_advisor_tasks        t ON t.task_id = o.task_id
WHERE  t.advisor_name = 'Segment Advisor'
  AND  t.status       = 'COMPLETED'
ORDER BY TO_NUMBER(f.more_info) DESC NULLS LAST;
```

```sql
-- クイック手動チェック：無駄な領域がある可能性が高いセグメント（アドバイザを使用しない）
SELECT owner,
       segment_name,
       segment_type,
       ROUND(blocks * 8192 / 1048576, 1)       AS allocated_mb,
       ROUND(num_rows * avg_row_len / 1048576, 1) AS estimated_data_mb,
       ROUND((blocks * 8192 - num_rows * avg_row_len) / 1048576, 1) AS waste_mb
FROM   dba_tables
WHERE  owner NOT IN ('SYS','SYSTEM','DBSNMP','SYSMAN')
  AND  num_rows > 10000
  AND  blocks > 100
ORDER BY waste_mb DESC NULLS LAST
FETCH FIRST 30 ROWS ONLY;
```

### メモリー・アドバイザ

メモリー・アドバイザは、現在のワークロードに基づいた SGA および PGA のサイジングに関する推奨事項を提供する。

#### SGA ターゲット・アドバイザ

```sql
-- SGA サイズのアドバイス：異なる SGA サイズにおける推定 DB 時間
SELECT sga_size,
       sga_size_factor,
       estd_db_time,
       estd_db_time_factor,
       estd_physical_reads
FROM   v$sga_target_advice
ORDER BY sga_size;
```

```sql
-- 共有プール・アドバイザ：異なる共有プール・サイズにおけるキャッシュ・ヒット率
SELECT shared_pool_size_for_estimate AS pool_size_mb,
       shared_pool_size_factor,
       estd_lc_size               AS estd_lib_cache_mb,
       estd_lc_memory_objects,
       estd_lc_time_saved         AS time_saved_sec,
       estd_lc_time_saved_factor
FROM   v$shared_pool_advice
ORDER BY shared_pool_size_for_estimate;
```

```sql
-- バッファ・キャッシュ・アドバイザ：異なるキャッシュ・サイズにおける物理読み取り
SELECT size_for_estimate          AS cache_size_mb,
       size_factor,
       estd_physical_read_factor,
       estd_physical_reads
FROM   v$db_cache_advice
WHERE  block_size = (SELECT value FROM v$parameter WHERE name = 'db_block_size')
AND    advice_status = 'ON'
ORDER BY size_for_estimate;
```

#### PGA ターゲット・アドバイザ

```sql
-- PGA ターゲット・アドバイス：ワークロードに基づく最適な PGA サイズ
SELECT pga_target_for_estimate  AS pga_target_mb,
       pga_target_factor,
       estd_pga_cache_hit_percentage,
       estd_overalloc_count
FROM   v$pga_target_advice
ORDER BY pga_target_for_estimate;
```

```sql
-- コンポーネントごとの現在の PGA 使用状況
SELECT name,
       ROUND(value/1024/1024, 1) AS mb
FROM   v$pgastat
WHERE  name IN (
    'total PGA inuse',
    'total PGA allocated',
    'aggregate PGA target parameter',
    'total freeable PGA memory',
    'maximum PGA allocated'
)
ORDER BY name;
```

---

## ヘルス・チェックの自動スケジューリング

### DBMS_SCHEDULER の使用

```sql
-- 毎週日曜日の午前 2 時に DB 構造整合性チェックをスケジュール
BEGIN
    DBMS_SCHEDULER.create_job(
        job_name        => 'WEEKLY_HM_STRUCT_CHECK',
        job_type        => 'PLSQL_BLOCK',
        job_action      => q'[
            BEGIN
                DBMS_HM.run_check(
                    check_name => 'DB Structure Integrity Check',
                    run_name   => 'SCHED_STRUCT_' || TO_CHAR(SYSDATE, 'YYYYMMDD')
                );
            END;
        ]',
        start_date      => TRUNC(NEXT_DAY(SYSDATE, 'SUNDAY')) + INTERVAL '2' HOUR,
        repeat_interval => 'FREQ=WEEKLY; BYDAY=SUN; BYHOUR=2; BYMINUTE=0',
        enabled         => TRUE,
        comments        => '週次 DB 構造整合性ヘルス・チェック'
    );
END;
/
```

### 実行後のヘルス・チェック結果の監視

```sql
-- 最近の実行で発生した FAILURE（失敗）の検出結果に対してアラートを出す
SELECT r.run_name,
       r.check_name,
       r.start_time,
       f.type,
       f.description,
       f.repair_sql
FROM   v$hm_run     r
JOIN   f.run_id f ON r.id = f.run_id
WHERE  r.start_time > SYSDATE - 7
  AND  f.type = 'FAILURE'
ORDER BY r.start_time DESC;
```

---

## ベスト・プラクティス

1. **メンテナンス・ウィンドウ後には必ずプロアクティブにヘルス・モニター・チェックを実行すること。** パッチ適用、インポート、手動でのデータファイル操作、およびストレージ・メンテナンスは、すべて微妙な破損を引き起こす可能性がある。メンテナンス後のヘルス・モニター実行により、ユーザーが気づく前に問題を特定できる。

2. **ストレージやハードウェアの変更後には必ず `DB Structure Integrity Check` を実行すること。** これにより、制御ファイル、データファイル・ヘッダー、および REDO ログの一貫性（すべてが依存する構造的な基礎）が検証される。

3. **週次実行をスケジュールし、FAILURE 検出結果に対してアラートを出すようにすること。** `DBMS_SCHEDULER` を使用して自動化し、日次の DBA ヘルス・チェック・スクリプトの一部として `V$HM_FINDING` の失敗を照会するようにする。

4. **SHRINK（縮小）または MOVE（移動）操作の前にセグメント・アドバイザを使用すること。** どのセグメントに最も再利用可能な領域があるかを特定し、メンテナンス作業の優先順位付けと影響の見積もりを行うのに役立つ。

5. **SGA のサイズを変更する前に、バッファ・キャッシュ/共有プールのアドバイザを参照すること。** これらのアドバイザは実際のワークロードをモデル化しており、収穫逓減点を示し、スワッピングを引き起こす可能性のあるメモリーの過剰割り当てを回避するのに役立つ。

6. **ヘルス・モニターの実行名をタイムスタンプ付きで保存すること。** `CHECK_TYPE_YYYYMMDD_HH24MISS` のような一貫した命名規則により、過去の実行履歴を容易に照会でき、チェック結果が時間の経過とともに変化したかどうかを検出できる。

7. **アドバイザの検出結果を変更管理ワークフローに統合すること。** SQL チューニング・アドバイザのプロファイルや SQL 計画ベースラインは、本番環境で受け入れる前に非本番環境でテストする必要がある（たとえ Oracle の推奨事項が一般的に信頼できるものであっても）。

---

## よくある間違いと回避方法

**間違い: ピーク時間帯にヘルス・モニター・チェックを実行する。**  
ディクショナリ整合性チェックなどのチェックはリソースを大量に消費し、内部ロックを取得したり大量の読み取りを実行したりする場合がある。メンテナンス・ウィンドウやトラフィックの少ない時間帯にスケジュールすること。

**間違い: チェックによって失敗が検出された後、推奨事項を確認しない。**  
失敗を発見しても推奨事項に従わないのは、チェックを全く実行しないよりも悪い。誤った安心感を生むからである。FAILURE 検出結果には常に迅速に対応すること。

**間違い: テストせずに SQL チューニング・アドバイザのプロファイルを受け入れる。**  
SQL プロファイルは実行計画を変更する。アドバイザは包括的な分析を行うが、特に計画の一貫性が重要な複雑な OLTP クエリについては、まず非本番環境でテストすること。

**間違い: ADVISORY（アドバイザリ）タイプの検出結果を無視する。**  
`ADVISORY` は失敗ではないが、多くの場合、早期の兆候（しきい値サイズに近づいているセグメント、まだエラーを引き起こしていないディクショナリの不整合など）を示している。週次の DBA チェックの中で、これらを確認すること。

**間違い: SYS/SYSTEM スキーマに対してセグメント・アドバイザを実行する。**  
これらのスキーマには、Oracle サポートの指導なしに縮小や移動を行うべきではない Oracle 内部オブジェクトが含まれている。セグメント・アドバイザのタスクからこれらを除外すること。

**間違い: 混合ワークロードのデータベースでメモリー・アドバイザの見積もりに過度に依存する。**  
アドバイザは現在のワークロードをモデル化する。データベースに大幅なワークロードの変動（例：夜間のバッチ、日中の OLTP）がある場合、アドバイザのスナップショットは、実際に最適化が必要なワークロードを代表していない可能性がある。代表的なピーク期間中にアドバイザを実行すること。

---


## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 以降の機能は、Oracle Database 26ai 対応機能として扱う。
- 19c と 26ai では、リリース更新によってデフォルト設定や非推奨機能が異なる場合があるため、両方の環境をサポートする場合は構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Administrator's Guide — Monitoring Database Operations](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/monitoring-database-operations.html)
- [Oracle Database 19c PL/SQL Packages Reference — DBMS_HM](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_HM.html)
- [Oracle Database 19c PL/SQL Packages Reference — DBMS_SQLTUNE](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQLTUNE.html)
- [Oracle Database 19c Reference — V$HM_CHECK](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-HM_CHECK.html)
- [Oracle Database 19c Reference — V$SGA_TARGET_ADVICE](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SGA_TARGET_ADVICE.html)
