# PL/SQL デバッグ (Debugging)

## 概要

PL/SQL のデバッグには、単純な出力ユーティリティから、フル機能のインタラクティブ・デバッガまで、さまざまな手法があります。それぞれの方法の長所と制限を理解することで、開発環境と本番環境の両方で迅速に問題を特定できるようになります。

---

## DBMS_OUTPUT

`DBMS_OUTPUT` は最もシンプルなデバッグ・ツールです。メッセージをサーバー側のバッファに書き込み、PL/SQL ブロックの完了後にクライアントに表示します。

### 基本的な使い方

```sql
-- SQL*Plus / SQLcl で出力を有効にする
SET SERVEROUTPUT ON SIZE UNLIMITED

-- SQL Developer では: [表示] > [DBMS出力] パネルを開き、緑の「+」をクリック

BEGIN
  DBMS_OUTPUT.PUT_LINE('処理開始');
  DBMS_OUTPUT.PUT('行の途中 ');  -- 改行なし
  DBMS_OUTPUT.PUT_LINE('ここへ続く');
  DBMS_OUTPUT.NEW_LINE;  -- 空行
  DBMS_OUTPUT.PUT_LINE('完了');
END;
/
```

### バッファ管理

```sql
-- 古いリリースではデフォルトのバッファは 20,000 バイト
-- 10g 以降、SQL*Plus では UNLIMITED に設定可能
SET SERVEROUTPUT ON SIZE UNLIMITED

-- PL/SQL 内で明示的に有効化しバッファ・サイズを設定
DBMS_OUTPUT.ENABLE(buffer_size => NULL);  -- NULL = 無制限
```

### DBMS_OUTPUT の制限事項

| 制限事項 | 詳細 |
|---|---|
| リアルタイムではない | ブロック全体が**完了した後**にのみ出力が表示される |
| バッファ・オーバーフロー | サイズ制限がある場合、バッファが一杯になると ORA-20000 が発生する |
| 本番環境で不可視 | アプリケーションが DBMS_OUTPUT を読み取ることは通常ない |
| セッション状態の変化 | ENABLE/DISABLE 操作がバッファの可用性に影響する |
| 複数セッションにまたがらない | 出力を行ったセッション内でのみ閲覧可能 |
| 高頻度ループに不向き | 呼び出しごとのパフォーマンス・オーバーヘッドがある |

```sql
-- パターン: パッケージ・フラグを使用した条件付きデバッグ出力
CREATE OR REPLACE PACKAGE debug_pkg AS
  g_debug BOOLEAN := FALSE;

  PROCEDURE enable;
  PROCEDURE disable;
  PROCEDURE log(p_msg IN VARCHAR2);
END debug_pkg;
/

CREATE OR REPLACE PACKAGE BODY debug_pkg AS
  PROCEDURE enable  IS BEGIN g_debug := TRUE;  END;
  PROCEDURE disable IS BEGIN g_debug := FALSE; END;

  PROCEDURE log(p_msg IN VARCHAR2) IS
  BEGIN
    IF g_debug THEN
      DBMS_OUTPUT.PUT_LINE(TO_CHAR(SYSTIMESTAMP, 'HH24:MI:SS.FF3') || ' | ' || p_msg);
    END IF;
  END log;
END debug_pkg;
/

-- 使用例: 一時的に有効にしてから無効化する
BEGIN
  debug_pkg.enable;
  process_orders;
  debug_pkg.disable;
END;
/
```

---

## DBMS_APPLICATION_INFO

`DBMS_APPLICATION_INFO` は、`V$SESSION` で確認できる `MODULE`、`ACTION`、`CLIENT_INFO` フィールドを設定します。これは、ビジネス・ロジックを大幅に変更することなく、本番環境で長時間実行される操作を監視するのに非常に役立ちます。

```sql
CREATE OR REPLACE PROCEDURE process_month_end_close(
  p_period_id IN NUMBER
) IS
  l_count NUMBER := 0;
BEGIN
  -- V$SESSION で確認できるモジュール名とアクション名を設定
  DBMS_APPLICATION_INFO.SET_MODULE(
    module_name => 'MONTH_END_CLOSE',
    action_name => 'INITIALIZING'
  );

  DBMS_APPLICATION_INFO.SET_CLIENT_INFO('period_id=' || p_period_id);

  -- フェーズ 1
  DBMS_APPLICATION_INFO.SET_ACTION('VALIDATE_OPEN_ITEMS');
  validate_open_items(p_period_id);

  -- フェーズ 2
  DBMS_APPLICATION_INFO.SET_ACTION('POST_JOURNAL_ENTRIES');
  FOR rec IN (SELECT * FROM pending_journals WHERE period_id = p_period_id) LOOP
    post_journal(rec.journal_id);
    l_count := l_count + 1;

    -- V$SESSION で確認できる進捗を更新
    DBMS_APPLICATION_INFO.SET_ACTION(
      'POSTING_JOURNALS (' || l_count || ' 完了)'
    );
  END LOOP;

  -- 完了時にクリア
  DBMS_APPLICATION_INFO.SET_MODULE(NULL, NULL);

END process_month_end_close;
/

-- 別のセッション (DBA 等) から進捗を監視
SELECT sid, module, action, client_info, last_call_et AS seconds_running
FROM   v$session
WHERE  module = 'MONTH_END_CLOSE';
```

### 長時間操作の追跡 (Long Operations Tracking)

```sql
-- V$SESSION_LONGOPS で確認できる長時間操作を登録
DECLARE
  l_rindex   BINARY_INTEGER;
  l_slno     BINARY_INTEGER;
  l_total    NUMBER := 1000;
  l_sofar    NUMBER := 0;
BEGIN
  l_rindex := DBMS_APPLICATION_INFO.SET_SESSION_LONGOPS_NOHINT;

  FOR i IN 1..l_total LOOP
    -- ... 処理 ...
    l_sofar := i;

    IF MOD(i, 100) = 0 THEN  -- 100行ごとに更新
      DBMS_APPLICATION_INFO.SET_SESSION_LONGOPS(
        rindex      => l_rindex,
        slno        => l_slno,
        op_name     => '従業員レコード処理',
        target      => 0,
        context     => 0,
        sofar       => l_sofar,
        totalwork   => l_total,
        target_desc => 'employees 表',
        units       => '行'
      );
    END IF;
  END LOOP;
END;
/

-- 別のセッションから監視
SELECT opname, sofar, totalwork,
       ROUND(sofar/totalwork * 100, 1) AS pct_complete,
       elapsed_seconds
FROM   v$session_longops
WHERE  sofar < totalwork
  AND  opname = '従業員レコード処理';
```

---

## SQL Developer PL/SQL デバッガ

SQL Developer は、ブレークポイント、ウォッチ式、ステップ実行が可能なインタラクティブ・デバッガを提供します。

### セットアップ

1. **デバッグ権限をユーザーに付与**:
```sql
-- デバッグの前に必要
GRANT DEBUG CONNECT SESSION TO my_dev_user;
GRANT DEBUG ANY PROCEDURE TO my_dev_user;  -- または特定のオブジェクト
```

2. **デバッグ情報付きでコンパイル** (行レベルのデバッグに必要):
```sql
-- デバッグ用シンボルを付けてコンパイル
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL = 1;  -- 正確な行追跡のため、最適化を無効化

ALTER PROCEDURE my_procedure COMPILE DEBUG;
-- または SQL Developer で：右クリック > [デバッグ用にコンパイル]
```

### デバッグの手順

1. SQL Developer のエディタでプロシージャを開く。
2. 左端の余白をクリックしてブレークポイントを設定（赤い点が表示される）。
3. プロシージャを右クリック > [実行]（またはデバッグ・アイコンをクリック）。
4. [PL/SQLの実行] ダイアログで入力パラメータの値を設定し実行。
5. ブレークポイントで実行が一時停止する。
6. デバッガ・ツールバーを使用：
   - **ステップ・イン (F7)**: 呼び出されたプロシージャの中に入る。
   - **ステップ・オーバー (F8)**: 現在の行を実行して、現在のプロシージャに留まる。
   - **ステップ・アウト (Shift+F7)**: 現在のプロシージャの最後まで実行し、呼び出し元に戻る。
   - **再開 (F9)**: 次のブレークポイントまで続行。
7. [データ] パネルで変数の内容を確認。
8. 実行中に変数の値を書き換えて条件分岐をテストする。
9. [ウォッチ] を使用して、特定の式を継続的に監視する。

---

## コンパイル警告の有効化と確認

Oracle は、PL/SQL コード内の潜在的な問題についてコンパイル時に警告を出すことができます。

### PLSQL_WARNINGS パラメータ

```sql
-- 現在のセッションですべての警告を有効にする
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL';

-- カテゴリを絞って有効にする (パフォーマンス、情報)
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:PERFORMANCE, ENABLE:INFORMATIONAL';

-- すべて有効にするが、特定の警告をエラーとして扱う
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL, ERROR:06002';

-- すべて無効にする
ALTER SESSION SET PLSQL_WARNINGS = 'DISABLE:ALL';
```

### 警告カテゴリ

| カテゴリ | コード範囲 | 説明 |
|---|---|---|
| `SEVERE` | PLW-05xxx | エラーの可能性が高い (例: 到達不能なコード) |
| `PERFORMANCE` | PLW-07xxx | パフォーマンス上の問題を引き起こす可能性のあるコード |
| `INFORMATIONAL` | PLW-06xxx | スタイルや設計上の改善提案 |

```sql
-- 有効化した後、再コンパイルして警告を確認
ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL';

CREATE OR REPLACE FUNCTION risky_function(p_val IN NUMBER) RETURN NUMBER IS
  l_result NUMBER;
BEGIN
  IF p_val > 0 THEN
    RETURN p_val * 2;
  ELSE
    RETURN 0;
  END IF;
  l_result := 99;  -- PLW-06002: 到達不能なコード
  RETURN l_result;
END risky_function;
/
-- 警告: PLW-06002: 到達不能なコード

-- コンパイル後の警告を確認
SELECT line, position, text, attribute
FROM   user_errors
WHERE  name = 'RISKY_FUNCTION'
  AND  type = 'FUNCTION'
ORDER BY line;
```

---

## UTL_FILE によるログ・ファイル出力

`UTL_FILE` はサーバー側の OS ファイルに書き込みます。DBMS_OUTPUT が実用的でないバッチ・ジョブ等のログ記録に便利です。

```sql
-- まず、DBA が OS パスを指す DIRECTORY オブジェクトを作成する必要がある
-- CREATE OR REPLACE DIRECTORY app_log_dir AS '/opt/oracle/logs';
-- GRANT READ, WRITE ON DIRECTORY app_log_dir TO my_schema;

CREATE OR REPLACE PROCEDURE write_log(
  p_message IN VARCHAR2
) IS
  l_file    UTL_FILE.FILE_TYPE;
  l_logfile VARCHAR2(50) := 'process_' || TO_CHAR(SYSDATE, 'YYYYMMDD') || '.log';
BEGIN
  l_file := UTL_FILE.FOPEN(
    location     => 'APP_LOG_DIR',  -- DIRECTORYオブジェクト名 (大文字)
    filename     => l_logfile,
    open_mode    => 'a',             -- 'a' = 追記, 'w' = 書き込み, 'r' = 読み取り
    max_linesize => 32767
  );

  UTL_FILE.PUT_LINE(
    l_file,
    TO_CHAR(SYSTIMESTAMP, 'YYYY-MM-DD HH24:MI:SS.FF3') ||
    ' | ' || SYS_CONTEXT('USERENV','SESSION_USER') ||
    ' | ' || p_message
  );

  UTL_FILE.FCLOSE(l_file);

EXCEPTION
  WHEN OTHERS THEN
    IF UTL_FILE.IS_OPEN(l_file) THEN
      UTL_FILE.FCLOSE(l_file);
    END IF;
    RAISE;
END write_log;
/
```

---

## DBMS_TRACE によるトレース

`DBMS_TRACE` は、サーバー側の PL/SQL 実行トレースを有効にします。トレース・データは `PLSQL_TRACE_EVENTS` 表と `PLSQL_TRACE_RUNS` 表に書き込まれます。

```sql
-- セットアップ (DBAによる一回限りの実行)
-- @?/rdbms/admin/tracetab.sql  -- トレース表の作成

-- トレースの開始
DBMS_TRACE.SET_PLSQL_TRACE(DBMS_TRACE.TRACE_ALL_CALLS);
-- オプション:
--   TRACE_ALL_CALLS    -- すべての呼び出しをトレース
--   TRACE_ALL_EXCEPTIONS -- すべての例外をトレース
--   TRACE_ALL_SQL      -- SQL 文をトレース

-- トレースしたいコードを実行
BEGIN
  process_orders;
END;
/

-- トレースの停止
DBMS_TRACE.CLEAR_PLSQL_TRACE;

-- トレース・データの確認
SELECT r.runid, r.run_date, r.run_comment,
       e.event_seq, e.event_kind, e.proc_name, e.line#
FROM   plsql_trace_runs  r
JOIN   plsql_trace_events e ON e.runid = r.runid
ORDER BY r.runid, e.event_seq;
```

---

## SQL トレース・ファイルの読み取り

SQL トレースを有効にすると、待機イベントやバインド変数を含む、すべての SQL および PL/SQL の実行詳細をキャプチャできます。

```sql
-- 現在のセッションでトレースを有効にする (待機イベント、バインド変数を含むレベル 12)
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- 実行
BEGIN
  process_orders;
END;
/

-- トレースを無効にする
ALTER SESSION SET EVENTS '10046 trace name context off';

-- トレース・ファイルの場所を特定
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';
```

### トレース・ファイルの処理

```bash
# データベース・サーバーの OS 上で tkprof を実行して可読化
tkprof /path/to/trace.trc output.txt explain=myuser/mypass sys=no
```

---

## デバッグのベスト・プラクティス

- 長時間実行されるすべてのプロシージャで `DBMS_APPLICATION_INFO` を使用する。コストはほぼゼロで、監視が容易になる。
- デバッグ出力には `TO_CHAR(SYSTIMESTAMP, 'HH24:MI:SS.FF3')` でタイムスタンプを付ける。
- パッケージ・レベルのデバッグ・フラグを使用して、ビジネス・ロジックを変更せずに有効/無効を切り替えられるようにする。
- 開発中は常に `DEBUG` 指定でコンパイルし、本番環境では通常の最適化でコンパイルする。
- 本番コード・パスでは `DBMS_OUTPUT` を削除または無効化し、適切なロギング機能に置き換える。
- `PLSQL_WARNINGS = 'ENABLE:ALL'` で定期的にテストし、到達不能コードやパフォーマンスの問題を早期に発見する。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| 本番環境で DBMS_OUTPUT に頼る | アプリケーションから見えず、出力量も制限される | 自律型トランザクションを使用したログ・テーブルを使用する |
| FCLOSE 前に IS_OPEN を確認しない | ファイルが開かれなかった場合に例外が発生する | 常に `IF UTL_FILE.IS_OPEN(l_file) THEN CLOSE; END IF` |
| 本番環境で DEBUG コンパイルしたままにする | パフォーマンス低下やソースコード露出のリスク | 本番用には通常コンパイルを行う |
| 終了時に DBMS_APPLICATION_INFO をクリアしない | プールされた接続の次の利用者に前のモジュール/アクションが残る | 例外ハンドラおよび正常終了時に必ずクリアする |
| タイトなループ内での大量のデバッグ出力 | メモリと CPU のオーバーヘッド | 条件を付けて N 回に 1 回だけ出力するようにする |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

---

## ソース

- [DBMS_OUTPUT (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_OUTPUT.html)
- [DBMS_APPLICATION_INFO (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_APPLICATION_INFO.html)
- [UTL_FILE (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/UTL_FILE.html)
- [DBMS_TRACE (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_TRACE.html)
- [Oracle Database SQL Trace Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/performing-application-tracing.html)
