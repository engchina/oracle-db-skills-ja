# Oracle DBMS_SCHEDULER

## 概要

`DBMS_SCHEDULER` は Oracle のエンタープライズ・ジョブ・スケジューリング・フレームワークである。Oracle 10g で導入され、従来の `DBMS_JOB` パッケージに代わるものとして設計された。PL/SQL コード、ストアド・プロシージャ、実行ファイル、およびスクリプトを、時間ベースの式、外部イベントへの応答、または依存関係チェーンの一部として実行するための豊富な機能を提供する。

従来の `DBMS_JOB` に対する主な利点:
- 名前付きオブジェクトとして管理され、`DBA_SCHEDULER_*` ビューから照会可能
- カレンダーベースの強力な反復式 (cron よりも柔軟)
- リソース管理やログ制御のためのジョブ・クラス
- メンテナンス期間を制御するための時間ウィンドウとウィンドウ・グループ
- 依存関係を持つワークフローを実現するジョブ・チェーン
- キュー・イベントやファイル到着などのイベント駆動型トリガー
- OS 実行ファイルを実行する外部ジョブ
- 電子メールやアラートによる通知フレームワークの組み込み

---

## コア・オブジェクト・モデル

```
SCHEDULES         — 再利用可能な実行スケジュールの定義
PROGRAMS          — 再利用可能な実行アクション（「何を実行するか」）の定義
JOBS              — 実行の単位。プログラムとスケジュールを組み合わせて構成
JOB CLASSES       — リソース管理やログ・ポリシーを一括適用するためのジョブ・グループ
WINDOWS           — リソース・計画を有効化する特定の時間帯
CHAINS            — ジョブ・ステップ間の依存関係を表すグラフ
```

---

## スケジュール (Schedules)

**スケジュール**は「いつ」実行するかを定義する。複数のジョブから参照可能である。

```sql
-- 毎日午前 2:00 に実行するスケジュール
BEGIN
    DBMS_SCHEDULER.CREATE_SCHEDULE(
        schedule_name   => 'DAILY_2AM',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY;BYHOUR=2;BYMINUTE=0;BYSECOND=0',
        comments        => '毎晩 02:00 に実行'
    );
END;
/

-- 平日の営業時間内、15分ごとに実行
BEGIN
    DBMS_SCHEDULER.CREATE_SCHEDULE(
        schedule_name   => 'WEEKDAY_BUSINESS_HOURS_15MIN',
        repeat_interval => 'FREQ=MINUTELY;INTERVAL=15;BYDAY=MON,TUE,WED,THU,FRI;BYHOUR=8,9,10,11,12,13,14,15,16,17',
        comments        => '平日の営業時間中 15分ごと'
    );
END;
/
```

### カレンダー式リファレンス

| 式 | 意味 |
|---|---|
| `FREQ=SECONDLY;INTERVAL=30` | 30秒ごと |
| `FREQ=MINUTELY;INTERVAL=5` | 5分ごと |
| `FREQ=DAILY;BYHOUR=0` | 毎日深夜 0:00 |
| `FREQ=MONTHLY;BYMONTHDAY=1` | 毎月 1 日 |
| `FREQ=MONTHLY;BYMONTHDAY=-1` | 毎月末日 |
| `FREQ=DAILY;BYDAY=MON,TUE,WED,THU,FRI` | 毎週月〜金曜日 |

---

## プログラム (Programs)

**プログラム**は実行するアクションをカプセル化する。

```sql
-- プロシージャを呼び出すプログラム
BEGIN
    DBMS_SCHEDULER.CREATE_PROGRAM(
        program_name   => 'REFRESH_SUMMARY_PROG',
        program_type   => 'STORED_PROCEDURE',
        program_action => 'etl_pkg.refresh_summary_tables',
        enabled        => TRUE
    );
END;
/
```

---

## ジョブ (Jobs)

**ジョブ**はプログラムとスケジュールを結びつける実行単位である。

### シンプルなインライン・ジョブ

```sql
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'NIGHTLY_STATS_JOB',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN DBMS_STATS.GATHER_SCHEMA_STATS(''APP''); END;',
        repeat_interval => 'FREQ=DAILY;BYHOUR=1',
        enabled         => TRUE,
        auto_drop       => FALSE,  -- 実行完了後もジョブ定義を保持
        comments        => '毎晩 1:00 に統計情報を収集'
    );
END;
/
```

### ジョブの管理操作

```sql
-- ジョブの無効化（一時停止）
EXEC DBMS_SCHEDULER.DISABLE('NIGHTLY_STATS_JOB');

-- ジョブの即時実行 (スケジュールに関わらず今すぐ実行)
EXEC DBMS_SCHEDULER.RUN_JOB('NIGHTLY_STATS_JOB');

-- 実行中のジョブを停止
EXEC DBMS_SCHEDULER.STOP_JOB('NIGHTLY_STATS_JOB');

-- ジョブの属性変更 (例: 最大失敗回数を設定)
EXEC DBMS_SCHEDULER.SET_ATTRIBUTE('NIGHTLY_STATS_JOB', 'max_failures', 3);
```

---

## ジョブ・チェーン (Job Chains)

複数のジョブ間に依存関係を持たせ、順番に実行したり、前のジョブの結果に基づいて分岐したりするための機能である。

```sql
BEGIN
    -- チェーンの作成
    DBMS_SCHEDULER.CREATE_CHAIN(chain_name => 'ETL_CHAIN');

    -- ステップの定義 (既存のプログラムをステップとして登録)
    DBMS_SCHEDULER.DEFINE_CHAIN_STEP('ETL_CHAIN', 'S1_EXTRACT', 'EXTRACT_PROG');
    DBMS_SCHEDULER.DEFINE_CHAIN_STEP('ETL_CHAIN', 'S2_LOAD', 'LOAD_PROG');

    -- ルールの定義 (S1 が成功したら S2 を開始)
    DBMS_SCHEDULER.DEFINE_CHAIN_RULE('ETL_CHAIN', 'S1_SUCCESS', 'S1_EXTRACT COMPLETED SUCCESSFULLY', 'START S2_LOAD');
    DBMS_SCHEDULER.DEFINE_CHAIN_RULE('ETL_CHAIN', 'S2_SUCCESS', 'S2_LOAD COMPLETED SUCCESSFULLY', 'END');

    DBMS_SCHEDULER.ENABLE('ETL_CHAIN');
END;
/
```

---

## 監視とログ

ジョブの実行状況を確認するには、以下のデータ・ディクショナリ・ビューを使用する。

```sql
-- 現在のジョブの状態と次の実行予定
SELECT job_name, state, last_start_date, next_run_date
FROM   dba_scheduler_jobs;

-- 過去の実行履歴（成功、失敗、実行時間など）
SELECT job_name, log_date, status, run_duration, error#
FROM   dba_scheduler_job_run_details
ORDER  BY log_date DESC;

-- 現在実行中のジョブ
SELECT job_name, session_id, elapsed_time
FROM   dba_scheduler_running_jobs;
```

---

## ベスト・プラクティス

- **明確な命名規則:** `<システム名>_<目的>_<頻度>_JOB` のような名前を付け、監視しやすくする。
- **`auto_drop => FALSE` を設定する:** 定期的なジョブの場合、これを TRUE にすると一度実行された後に定義が消えてしまうため注意が必要である。
- **名前付きスケジュール/プログラムを利用する:** インラインで書かずに各オブジェクトを定義することで、スケジュールの変更などが一括で行えるようになる。
- **リソース・マネージャとの連携:** `JOB_CLASS` を作成し、バッチ・ジョブがオンライン業務の CPU を使い切らないように制限をかける。
- **時刻指定に注意:** `start_date` にはタイムゾーンを含めることが推奨される（夏時間の影響を避けるため）。

---

## よくある間違い

**間違い: 古い `DBMS_JOB` を使い続ける。**
`DBMS_JOB` はレガシーなインターフェースであり、機能が制限されている。新規開発では必ず `DBMS_SCHEDULER` を使用すること。

**間違い: エラー・ハンドリングを行わずにジョブを作成する。**
ジョブ内の PL/SQL で例外が発生した場合、`RAISE` しないとスケジューラ側で「成功」と見なされることがある。適切に例外を捕捉し、必要に応じて再試行やアラート送信を行うように設計する。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [DBMS_SCHEDULER — Oracle Database PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SCHEDULER.html)
- [Oracle Database Administrator's Guide: Scheduling Jobs](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/scheduling-jobs-with-oracle-scheduler.html)
