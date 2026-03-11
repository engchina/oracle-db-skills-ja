# アラート・ログの分析

## 概要

Oracle のアラート・ログは、データベース・インスタンスの主要な診断記録である。起動と停止のイベント、管理操作、構造的な変更、そして最も重要な点として、通常の運用中に発生したエラーを記録する。すべての Oracle DBA は、トラブルシューティング・ワークフローの最初のステップとして、アラート・ログのナビゲート、読み取り、および監視に習熟している必要がある。

アラート・ログを理解するということは、それが自動診断リポジトリ (ADR) のどこに存在するか、XML 形式とテキスト形式の違いは何か、一般的な ORA- エラーをどのように解釈するか、そしてユーザーが報告する前に問題を表面化させるために監視をどのように自動化するかを理解することを意味する。

---

## ADR とアラート・ログの場所

### 自動診断リポジトリ (ADR)

Oracle 11g で導入された**自動診断リポジトリ (ADR)** は、すべての Oracle 製品の診断データを格納する統一されたファイルベースのリポジトリである。各データベース・インスタンスは、`DIAGNOSTIC_DEST` 初期化パラメータによって制御されるルート・ディレクトリの下に、独自の ADR ホームを保持する。

```sql
-- ADR ベースとホームを確認する
SELECT name, value
FROM   v$diag_info
ORDER BY name;
```

返される主な行:

| 名前 | 説明 |
|------|-------------|
| `ADR Base` | ADR ツリーのルート (通常は `$ORACLE_BASE`) |
| `ADR Home` | このインスタンスの ADR ホームへのフルパス |
| `Diag Alert` | XML アラート・ログを含むディレクトリ |
| `Diag Trace` | トレース・ファイルとテキスト形式アラート・ログを含むディレクトリ |

### アラート・ログのファイル形式

Oracle はアラート・ログを **2 つの形式で同時に** 保持する。

1. **XML 形式** — `$ADR_HOME/alert/` に配置される。ファイル名は `log.xml` である。これは `adrci` や Enterprise Manager が使用する正式な形式である。構造化されたメタデータ、タイムスタンプ、およびインシデントの相互参照が含まれている。

2. **テキスト形式** — `$ADR_HOME/trace/` に配置される。ファイル名は `alert_<SID>.log` である。これは、ほとんどの DBA が直接使用する人間が読める形式である。XML 形式と並行して生成され、より単純なレイアウトで同じ情報が含まれている。

```sql
-- テキスト形式アラート・ログへの正確なパスを確認する
SELECT value
FROM   v$diag_info
WHERE  name = 'Diag Trace';
-- フルパスを取得するには、末尾に /alert_<SID>.log を追加する
```

```sql
-- XML 形式アラート・ログへの正確なパスを確認する
SELECT value
FROM   v$diag_info
WHERE  name = 'Diag Alert';
-- このディレクトリ内の log.xml が XML ログである
```

### レガシーな場所 (11g より前)

ADR より前は、アラート・ログは `$ORACLE_BASE/admin/<DB_NAME>/bdump/alert_<SID>.log` または `BACKGROUND_DUMP_DEST` で指定されたディレクトリに書き込まれていた。古いデータベースを扱う場合は、そのパラメータを確認すること。

```sql
SELECT name, value
FROM   v$parameter
WHERE  name = 'background_dump_dest';
```

---

## XML 形式とテキスト形式の比較

### テキスト形式

テキスト形式のアラート・ログは、任意のテキスト・エディタや Unix コマンドライン・ツールで簡単に読み取ることができる。エントリは以下のようになる。

```
Thu Mar 06 14:23:15 2026
ORA-01555 caused by SQL statement below (Query Duration=0 sec, SCN: 0x0003.abcd1234):
SELECT * FROM sales WHERE sale_date > :1
```

**長所:** 人間が読める、grep が容易、シェル・スクリプトで解析可能。
**短所:** 構造化されたメタデータがない、プログラムによる照会が困難、トレース・ファイルへの直接的な相互参照がない。

### XML 形式

XML 形式のアラート・ログは、各エントリを構造化された要素で囲む。

```xml
<msg time='2026-03-06T14:23:15.123+00:00' org_id='oracle' comp_id='rdbms'
     msg_id='12345' type='INCIDENT_ERROR' group='generic,orcl'
     level='1' host_id='myserver' host_addr='192.168.1.10' pid='23456'>
  <txt>ORA-00600: internal error code, arguments: [kcbz_check_objd_typ_3], ...</txt>
</msg>
```

**長所:** `adrci` を介して照会可能、時間範囲やメッセージ・タイプによるフィルタリングをサポート、インシデント ID と相互リンクされている。
**短所:** 冗長である、直接読むのには適さない。

### SQL からの XML アラート・ログの照会

Oracle は、`V$DIAG_ALERT_EXT` ビューを介してアクセス可能な外部表を通じて、XML アラート・ログを公開している。

```sql
-- SQL からアラート・ログの最新 50 件を取得（ファイル・アクセスは不要）
SELECT originating_timestamp,
       message_text
FROM   v$diag_alert_ext
ORDER BY originating_timestamp DESC
FETCH FIRST 50 ROWS ONLY;
```

```sql
-- 過去 24 時間に発生したすべての ORA- エラーを検索する
SELECT originating_timestamp,
       message_text
FROM   v$diag_alert_ext
WHERE  originating_timestamp > SYSTIMESTAMP - INTERVAL '24' HOUR
AND    message_text LIKE 'ORA-%'
ORDER BY originating_timestamp DESC;
```

---

## アラート・ログ内の一般的な ORA- エラー

### 重大なエラー (即時の対応が必要)

**ORA-00600: 内部エラー・コード**
最も深刻な Oracle エラー。Oracle が検出した内部的な不整合を示す。常に、すべての引数リストと関連するトレース・ファイルを添えて、サポート・リクエストを提出すること。
- 引数は角括弧内に示される: `ORA-00600: [kcbz_check_objd_typ_3], [0], [1], ...`
- 各引数の組み合わせは、特定のバグや問題に対応している
- プロセス ID とタイムスタンプが一致するトレース・ファイルを `$ADR_HOME/trace/` 内で探すこと

**ORA-07445: 例外が発生しました**
ORA-00600 と同様だが、オペレーティング・システムの例外 (セグメンテーション・フォールト、シグナル) によってトリガーされる。これもサポート・リクエストが必要である。

**ORA-01578 / ORA-01110: データ・ブロックの破損**
データ・ブロックの破損を示す。直ちにハードウェアの問題を確認し、RMAN の `VALIDATE DATABASE` を実行すること。

```sql
-- 既知の破損ブロックを確認する
SELECT file#, block#, blocks, corruption_type
FROM   v$database_block_corruption;
```

**ORA-04031: 共有メモリーを割り当てることができません**
共有プールまたは別の SGA コンポーネントが枯渇している。メモリーの圧迫またはメモリー・リークの可能性を示す。

```sql
-- 共有プールの圧迫を診断する
SELECT pool, name, bytes/1024/1024 AS mb
FROM   v$sgastat
WHERE  pool = 'shared pool'
ORDER BY bytes DESC
FETCH FIRST 20 ROWS ONLY;
```

**ORA-00257: アーカイバ・エラー**
アーカイバ・プロセス (ARCn) がアーカイブ・ログを書き込めない。アーカイブ・ログの出力先のディスク空き容量を確認すること。

```sql
-- アーカイブ・ログの出力先を確認する
SELECT dest_id, status, target, archiver, error
FROM   v$archive_dest
WHERE  status != 'INACTIVE';
```

### 警告レベルのエラー (速やかに調査)

**ORA-01555: スナップショットが古すぎます**
長時間実行されているクエリの読み取りが完了する前に、UNDO データが上書きされた。通常は、UNDO 表領域のサイズ不足、または非常に長いトランザクションが原因である。

```sql
-- UNDO 保持と表領域サイズを確認する
SELECT tablespace_name, retention
FROM   dba_tablespaces
WHERE  contents = 'UNDO';

SELECT a.tablespace_name,
       ROUND(a.bytes/1024/1024, 1) AS alloc_mb,
       ROUND(b.bytes/1024/1024, 1) AS used_mb
FROM   dba_tablespaces a
JOIN   (SELECT tablespace_name, SUM(bytes) bytes
        FROM   dba_segments GROUP BY tablespace_name) b
  ON   a.tablespace_name = b.tablespace_name
WHERE  a.contents = 'UNDO';
```

**ORA-01652: 一時セグメントを拡張できません**
一時表領域がいっぱいである。大規模なソートやハッシュ操作、あるいは強制終了されたセッションによる一時領域のリークを示している。

**ORA-00060: デッドロックが検出されました**
2 つのセッションが互いのロックを待機している。Oracle は一方の文を強制終了することで解決する。トレース・ファイルにはデッドロック・グラフが含まれているため、必ず確認すること。

**ORA-03113 / ORA-03114: 通信チャネルでファイル末尾が検出されました**
クライアントがサーバーへの接続を失った。通常はサーバー・プロセスのクラッシュを示しており、一致するトレース・ファイルを探すこと。

**ORA-12514 / ORA-12541: TNS エラー**
リスナー関連の問題。`listener.log` を確認し、`lsnrctl status` を実行すること。

### 情報メッセージ (ベースラインの把握)

- `Thread 1 advanced to log sequence` — REDO ログのスイッチ。頻度を監視すること。
- `Completed checkpoint up to RBA` — チェックポイントが完了。正常。
- `Starting ORACLE instance` / `Shutting down instance` — インスタンスの生涯サイクル・イベント。
- `ALTER TABLESPACE ... ADD DATAFILE` — 記録された管理 DDL。
- アラート・ログ内の `ORA-00942: 表またはビューが存在しません` — 通常は起動スクリプトの問題。

---

## 注目すべきパターン

### REDO ログ・スイッチの頻度

頻繁なログ・スイッチ (15〜20 分に 1 回以上) は、REDO ログのサイズが不足していることを示し、パフォーマンスのオーバーヘッドや I/O の増加につながる。

```sql
-- 過去 7 日間の 1 時間ごとのログ・スイッチ頻度
SELECT TRUNC(first_time, 'HH24') AS switch_hour,
       COUNT(*)                  AS switches
FROM   v$log_history
WHERE  first_time > SYSDATE - 7
GROUP BY TRUNC(first_time, 'HH24')
ORDER BY switch_hour;
```

### チェックポイントが未完了

アラート・ログ内の `checkpoint not complete` というメッセージは、チェックポイントによるダーティ・バッファのディスクへの書き込みが完了していないため、Oracle が REDO ログ・グループを再利用できないことを意味する。これによりログ・ライターが待機状態となり、すべての DML が停止する。

```
Thread 1 cannot allocate new log, sequence 9823
Checkpoint not complete
  Current log# 3 seq# 9823 mem# 0: /u01/oradata/orcl/redo03.log
```

修正策: REDO ログ・グループの追加、REDO ログ・ファイルのサイズ拡大、またはチェックポイント頻度の調整 (`FAST_START_MTTR_TARGET`)。

### ORA-00600 / ORA-07445 の頻度

いかなる発生も調査すべきである。同じエラー・コードと引数が繰り返し発生する場合は、特定のバグを指している可能性がある。引数のシグネチャを使用して、My Oracle Support (MOS) を検索すること。

### 増加するトレース・ファイル・ディレクトリ

```sql
-- ADR インシデント数を確認する
SELECT COUNT(*) FROM v$diag_incident;

-- 問題キーごとのインシデント（同様のエラーをグループ化）
SELECT problem_key, COUNT(*) AS incident_count
FROM   v$diag_incident
GROUP BY problem_key
ORDER BY incident_count DESC;
```

---

## adrci によるエラーの検索

`adrci` (ADR Command Interpreter) は、アラート・ログを含む ADR コンテンツを操作するためのコマンドライン・ツールである。詳細は、関連ガイドの `adrci-usage.md` を参照すること。

アラート・ログ操作のクイック・リファレンス:

```bash
# adrci を起動
adrci

# ADR ホームを設定 (複数のホームが存在する場合)
adrci> SET HOMEPATH diag/rdbms/orcl/orcl

# アラート・ログの最後の 20 行を表示 (text の tail に相当)
adrci> SHOW ALERT -TAIL 20

# エラーのみを含むアラート・ログを表示
adrci> SHOW ALERT -P "MESSAGE_TEXT LIKE '%ORA-%'"

# 時間帯を指定してアラート・ログを表示
adrci> SHOW ALERT -P "ORIGINATING_TIMESTAMP > TIMESTAMP '2026-03-06 00:00:00'"

# すべての問題 (グループ化されたインシデント) を表示
adrci> SHOW PROBLEM

# インシデントを表示
adrci> SHOW INCIDENT
```

---

## アラート・ログ監視の自動化

### シェル・スクリプトによるアプローチ

定石として、テキスト形式アラート・ログの最後に読み取った位置を追跡し、実行のたびに新しいコンテンツのみをスキャンする方法がある。

```bash
#!/bin/bash
# monitor_alert.sh — 新しい ORA- エラーがないか Oracle アラート・ログをスキャン
# cron を使用して 5〜10 分おきに実行

ORACLE_SID=orcl
ALERT_LOG="/u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_${ORACLE_SID}.log"
STATE_FILE="/tmp/alert_log_offset_${ORACLE_SID}"
EMAIL="dba@example.com"

# 現在のファイル・サイズを取得
CURRENT_SIZE=$(wc -c < "$ALERT_LOG")

# 最後に読み取ったオフセットを取得 (デフォルトは 0)
LAST_OFFSET=0
[ -f "$STATE_FILE" ] && LAST_OFFSET=$(cat "$STATE_FILE")

# ファイルが増分している場合のみ処理
if [ "$CURRENT_SIZE" -gt "$LAST_OFFSET" ]; then
    # 新しいコンテンツを抽出し、エラーを検索
    NEW_ERRORS=$(tail -c +$((LAST_OFFSET + 1)) "$ALERT_LOG" | \
                 grep -E "ORA-[0-9]+" | \
                 grep -v "ORA-00000")

    if [ -n "$NEW_ERRORS" ]; then
        echo "$NEW_ERRORS" | mail -s "Oracle Alert: $ORACLE_SID でエラーを検出" "$EMAIL"
    fi

    # オフセットを更新
    echo "$CURRENT_SIZE" > "$STATE_FILE"
fi
```

crontab への追加:
```
*/5 * * * * /opt/scripts/monitor_alert.sh
```

### SQL ベースの監視 (Cloud Control / カスタム)

`V$DIAG_ALERT_EXT` を使用すると、任意のスケジューラ (DBMS_SCHEDULER、Enterprise Manager、JDBC を使用した Grafana など) が実行できる SQL ベースの監視クエリを構築できる。

```sql
-- 監視ウィンドウ内のエラー (必要に応じてパラメータ化)
SELECT originating_timestamp AS error_time,
       message_text
FROM   v$diag_alert_ext
WHERE  originating_timestamp > SYSTIMESTAMP - INTERVAL '10' MINUTE
AND    (message_text LIKE 'ORA-%'
        OR message_text LIKE '%error%'
        OR message_text LIKE '%checkpoint not complete%')
ORDER BY originating_timestamp;
```

### DBMS_SCHEDULER ベースのアプローチ

```sql
-- アラート・ログを確認し、結果を記録するプロシージャを作成
CREATE OR REPLACE PROCEDURE check_alert_log AS
    v_count NUMBER;
BEGIN
    SELECT COUNT(*)
    INTO   v_count
    FROM   v$diag_alert_ext
    WHERE  originating_timestamp > SYSTIMESTAMP - INTERVAL '15' MINUTE
    AND    message_text LIKE 'ORA-00060%';  -- デッドロック

    IF v_count > 0 THEN
        -- 監視テーブルへの挿入、APEX 通知の送信など
        INSERT INTO dba_alert_log_events (event_time, event_type, event_count)
        VALUES (SYSTIMESTAMP, 'DEADLOCK', v_count);
        COMMIT;
    END IF;
END;
/

-- 15 分おきに実行するようにスケジュール
BEGIN
    DBMS_SCHEDULER.create_job(
        job_name        => 'CHECK_ALERT_LOG_JOB',
        job_type        => 'STORED_PROCEDURE',
        job_action      => 'CHECK_ALERT_LOG',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=MINUTELY; INTERVAL=15',
        enabled         => TRUE
    );
END;
/
```

### Oracle Enterprise Manager

OEM / Cloud Control では、以下のメトリックしきい値を構成する。
- **Generic Alert Log Error Status** — ORA- エラーが発生したときにアラート
- **Incident** メトリック — インシデント数が増加したときにアラート
- **Archive Area Used (%)** — アーカイブ領域が不足する前にアラート

---

## ベスト・プラクティス

1. **アラート・ログを直接削除または切り捨てないこと。** Oracle は必要に応じて自動的にローテートする。サイズが懸念される場合は、テキスト・ログを手動で削除するのではなく、`adrci purge` を使用して古いインシデントや古い XML ログ・セグメントを削除すること。

2. **本番環境ではアラート・ログを継続的に監視すること。** 5〜10 分の間隔でポーリングすることで、ユーザーが報告してくる前に問題を把握できる。重大なエラー (ORA-00600, ORA-07445, ORA-01578) に対しては、準リアルタイムのアラートを目指す。

3. **常に関連するトレース・ファイルを確認すること。** アラート・ログがトレース・ファイルを参照している場合 (通常は明示的に名前が挙げられる)、MOS を検索する前にまずそのファイルを確認すること。トレース・ファイルには、完全なコンテキスト、スタック・トレース、そして多くの場合、根本原因が含まれている。

4. **システム・イベントと関連付けること。** アラート・ログのタイムスタンプは出発点に過ぎない。同じ時間帯の OS メトリック (CPU, I/O, メモリー) と関連付けて、Oracle のバグとリソースの圧迫を区別すること。

5. **ベースラインを確立すること。** 自身のアラート・ログにおける「正常」な状態を知っておく。ログ・スイッチはどのくらいの頻度で発生するか？ 定期的なチェックポイントの警告はあるか？ ベースラインからの逸脱が、多くの場合、トラブルの最初の兆候である。

6. **本番環境のアラート・ワークフローには `adrci` を使用すること。** XML 形式に対するフィルタリング機能は、特にインシデントをトレース・ファイルと関連付ける際に、テキスト・ファイルに対する grep よりも正確で高速である。

7. **アラート・ログを定期的にアーカイブすること。** コンプライアンスや事後分析のために、現在の実効アラート・ログをアーカイブ場所 (ファイル名にタイムスタンプを付加) に、月次または週次でコピーすることを検討する。

---

## よくある間違いと回避策

**間違い: データベースが安定しているように見えるため、ORA-00600 や ORA-07445 を無視する。**
これらの内部エラーは、Oracle 内部で予期しないことが起こったことを常に示している。その後データベースが正常に見えても、高負荷時に再発する可能性がある。必ずサポート・リクエストを開き、トレース・ファイルを収集すること。

**間違い: "ORA-" だけを grep し、他の重要なメッセージを見逃す。**
すべての重大なエントリに "ORA-" が含まれているわけではない。`checkpoint not complete`、`db_block_checking detected an error`、`WARNING: Heavy swapping observed` などのメッセージも同様に重要である。これらの文字列を含む監視パターンを構築すること。

**間違い: データベース・ホストとは異なるサーバーでテキスト・アラート・ログ・ファイルを監視する。**
アラート・ログが共有 NFS マウント上にあり、マウントが不安定になった場合、監視ツールが新しいエントリを検知できなくなってもエラーが発生しない可能性がある。監視ツールが実際にファイルを読み取れることを常に確認すること。

**間違い: メンテナンスの前後でアラート・ログの状態を記録しない。**
メンテナンス・ウィンドウの前に、アラート・ログの最後の行またはタイムスタンプを控えておく。終了後、ウィンドウ中に書き込まれたすべての内容を確認し、目に見える問題として表面化しなかったエラーを見逃さないようにすること。

**間違い: すべての ORA-01555 エラーをチューニングの問題として扱う。**
ORA-01555 は、アプリケーションのバグ (コミットされていないトランザクションの保持)、UNDO の削除と競合するスケジュール・ジョブ内のクエリ、または不適切な UNDO サイズ設定によっても発生する。単に `UNDO_RETENTION` を増やす前に、完全なコンテキストを確認すること。

**間違い: XML 形式とテキスト形式のアラート・ログが常に同期していると思い込む。**
まれなケース (システム・クラッシュやディスクの問題) では、それらが乖離することがある。XML ログが正式なソースであり、精度が重要な場合は `adrci` を使用すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 19c と 26ai では、リリース更新によってデフォルト設定や非推奨機能が異なる場合があるため、両方の環境をサポートする場合は構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Administrator's Guide — Diagnosing and Resolving Problems](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/diagnosing-and-resolving-problems.html)
- [Oracle Database 19c Reference — V$DIAG_ALERT_EXT](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DIAG_ALERT_EXT.html)
- [Oracle Database 19c Reference — V$DIAG_INFO](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DIAG_INFO.html)
