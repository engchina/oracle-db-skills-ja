# Oracle Databaseにおけるロックと並行性

## 概要

Oracleの並行性モデルは、他の多くのデータベースとは根本的に異なっている。マルチバージョン並行性制御（MVCC）の実装により、**「読取り側が書込み側をブロックせず、書込み側が読取り側をブロックしない」**ことを意味する。これにより、他のデータベースを悩ませる膨大な種類の競合問題が排除されるが、Oracleでも書込み同士の競合や明示的なロック・シナリオでは引き続きロックが使用される。

Oracleのロック・アーキテクチャを理解することは、デッドロックや過度な競合を発生させずに、並行ロード下でスケールするアプリケーションを作成するために不可欠である。

---

## マルチバージョン並行性制御 (MVCC)

### OracleにおけるMVCCの仕組み

行が変更される際、Oracleは元のデータを上書きしない。代わりに：

1. 新しい行バージョンがデータ・ブロックに書き込まれる
2. 古い行バージョンは**undo表領域**（ロールバック・セグメント）に格納される
3. 古いバージョンを必要とする読取り側は、必要に応じてundoデータからそれを再構成する

これにより「タイムトラベル」機能が実現する。すべての読取りは、並行する書込み側の影響を受けることなく、クエリの開始時点のSCN（システム変更番号）における**一貫したスナップショット**を参照する。

```sql
-- 現在のSCNを確認
SELECT current_scn FROM v$database;

-- 特定のSCN時点のデータを照会 (フラッシュバック・クエリ)
SELECT * FROM orders AS OF SCN 12345678;

-- 特定のタイムスタンプ時点のデータを照会
SELECT * FROM orders AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '5' MINUTE);
```

### 読取り一貫性の保証

| シナリオ | Oracleの動作 |
|---|---|
| 読取り vs. 書込み (同一行) | ブロックなし。読取り側はundoを介して変更前のデータを参照 |
| 書込み vs. 読取り (同一行) | ブロックなし。書込み側は処理を続行し、読取り側はundoを使用 |
| 書込み vs. 書込み (同一行) | 書込み側2は、書込み側1がコミットまたはロールバックするまでブロックされる |
| 長時間の読取り (undoが再利用済み) | `ORA-01555: snapshot too old` (スナップショットが古すぎます) |

### ORA-01555: スナップショットが古すぎます

このエラーは、undoデータが上書きされたために（undo保持期間を超過）、Oracleが古い行バージョンを再構成できない場合に発生する。防止策：

```sql
-- 現在の undo_retention 設定を確認
SHOW PARAMETER undo_retention;

-- undo保持期間を延長 (秒)
ALTER SYSTEM SET UNDO_RETENTION = 3600;  -- 1時間

-- undoアドバイザの推奨値を確認
SELECT d.undoblks, d.maxquerylen, d.tuned_undoretention
FROM   v$undostat d
WHERE  rownum <= 1;

-- undo保持の保証を有効化 (undoの上書きを防止)
ALTER TABLESPACE undotbs1 RETENTION GUARANTEE;
```

---

## 行レベル・ロック

Oracleは、`INSERT`、`UPDATE`、または `DELETE` されるすべての行に対して、自動的に行レベルのロックを取得する。これらのロックは：

- **排他 (Xモード)**: 変更を行っているセッションによって保持される
- **COMMIT または ROLLBACK 時にのみ解放される**
- データ・ブロック自体に格納される（ロック・テーブルを持たない）ため、ロックする行数に関わらず実質的にリソース消費がない

```sql
-- 現在の行ロックを表示
SELECT o.object_name, l.session_id, l.locked_mode,
       s.username, s.status, s.sql_id
FROM   v$locked_object l
JOIN   dba_objects o ON l.object_id = o.object_id
JOIN   v$session s ON l.session_id = s.sid
ORDER  BY o.object_name;
```

### ロック・モード

| モード・コード | 名称 | 説明 |
|---|---|---|
| 0 | None | |
| 1 | Null (N) | サブ共有。ほとんど制限なし |
| 2 | Row Share (SS) | SELECT FOR UPDATE、または実行中のDML |
| 3 | Row Exclusive (SX) | 表に対して実行中のDML |
| 4 | Share (S) | `LOCK TABLE ... IN SHARE MODE` |
| 5 | Share Row Exclusive (SSX) | |
| 6 | Exclusive (X) | `LOCK TABLE ... IN EXCLUSIVE MODE`, DDL |

---

## SELECT FOR UPDATE

`SELECT FOR UPDATE` は、DMLが実行される前に、選択された行を即座にロックする。これは、更新後の値を決定する前に行を予約するための**悲観的ロック (Pessimistic Locking)** の主要な手段である。

### 基本構文

```sql
-- 選択された全行をロックする。既にロックされている行があれば無期限に待機する
SELECT account_id, balance
FROM   accounts
WHERE  account_id IN (1001, 2001)
FOR UPDATE;

-- ロックして処理する例
DECLARE
    v_balance accounts.balance%TYPE;
BEGIN
    SELECT balance INTO v_balance
    FROM   accounts
    WHERE  account_id = 1001
    FOR UPDATE;  -- 行が排他的にロックされる

    IF v_balance >= 500 THEN
        UPDATE accounts SET balance = balance - 500 WHERE account_id = 1001;
        UPDATE accounts SET balance = balance + 500 WHERE account_id = 2001;
        COMMIT;
    ELSE
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20001, 'Insufficient funds');
    END IF;
END;
```

### NOWAIT — ロックされていれば即座に失敗させる

```sql
-- 既にロックされている行があれば、即座に ORA-00054 を発生させる
SELECT product_id, stock_qty
FROM   inventory
WHERE  product_id = 42
FOR UPDATE NOWAIT;

-- アプリケーションでの処理例
DECLARE
    v_qty NUMBER;
BEGIN
    BEGIN
        SELECT stock_qty INTO v_qty FROM inventory WHERE product_id = 42 FOR UPDATE NOWAIT;
    EXCEPTION
        WHEN resource_busy THEN  -- ORA-00054
            RAISE_APPLICATION_ERROR(-20002, 'Product is being updated by another user. Please try again.');
    END;

    IF v_qty > 0 THEN
        UPDATE inventory SET stock_qty = stock_qty - 1 WHERE product_id = 42;
        COMMIT;
    END IF;
END;
```

### WAIT n — 最大 n 秒間待機する

```sql
-- ロックを最大5秒間待機し、取得できなければ ORA-30006 を発生させる
SELECT order_id, status
FROM   orders
WHERE  order_id = 9999
FOR UPDATE WAIT 5;
```

### SKIP LOCKED — 非ブロック型のキュー処理

`SKIP LOCKED` は、ワーク・キューの実装に非常に有用である。待機する代わりに、既にロックされている行をスキップする。これにより、複数のワーカーが競合することなく並行してキューを処理できる。

```sql
-- ワーカー・プロセス: 次に利用可能な担当ジョブを取得
DECLARE
    v_job_id   NUMBER;
    v_payload  VARCHAR2(4000);
BEGIN
    -- 他のワーカーがロックしている行をスキップし、未処理のジョブを1つ取得
    SELECT job_id, payload INTO v_job_id, v_payload
    FROM   job_queue
    WHERE  status = 'PENDING'
      AND  ROWNUM = 1
    ORDER  BY created_at
    FOR UPDATE SKIP LOCKED;

    -- 処理中としてマーク
    UPDATE job_queue SET status = 'PROCESSING', started_at = SYSTIMESTAMP
    WHERE  job_id = v_job_id;

    COMMIT;

    -- ジョブを処理 (ロックの外で実行)
    process_job(v_job_id, v_payload);

    -- 完了としてマーク
    UPDATE job_queue SET status = 'DONE', completed_at = SYSTIMESTAMP
    WHERE  job_id = v_job_id;
    COMMIT;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        NULL;  -- 利用可能なジョブなし
END;
```

このワーカーの複数のインスタンスは、プロセス間調整を必要とすることなく、Oracleの行レベル・ロックによって自動的に処理が振り分けられる。

---

## デッドロックの検出と回避

**デッドロック**は、2つ以上のセッションが互いに他方の保持するロックを待機し、サイクルが発生した際に起こる。

Oracleはバックグラウンドのサイクル検出アルゴリズムを使用して、自動的にデッドロックを検出する。検出されると：
- 1つのセッションが `ORA-00060: deadlock detected while waiting for resource` (リソース待機中にデッドロックが検出されました) を受信する
- Oracleはエラーを受信した**単一の文のみ**をロールバックする（トランザクション全体ではない）
- ロールバックされたセッションは、その文を再実行するか、トランザクション全体をロールバックする必要がある

```sql
-- デッドロックのシナリオ
-- セッション 1                           セッション 2
UPDATE t SET v=1 WHERE id=1;  -- 成功
                                UPDATE t SET v=2 WHERE id=2;  -- 成功
UPDATE t SET v=3 WHERE id=2;  -- 待機
                                UPDATE t SET v=4 WHERE id=1;  -- 待機 -> デッドロック
                                -- セッション 2 が ORA-00060 を受信
```

### デッドロック・アラート・ログ

Oracleはアラート・ログとトレース・ファイルにデッドロックのトレースを記録する：

```bash
# デッドロックのトレースを検索
grep -l "deadlock" $ORACLE_BASE/diag/rdbms/*/trace/*.trc | tail -5
```

```sql
-- 最近のデッドロックをトレース・ファイルから特定
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';
```

### デッドロック回避戦略

**戦略 1: 一貫したロック順序**

すべてのコード・パスで常に同じ順序でロックを取得する：

```sql
-- 誤り: 異なる順序はデッドロックの要因となる
-- パス A: 注文1をロック、次に注文2をロック
-- パス B: 注文2をロック、次に注文1をロック

-- 正解: 常に昇順でロックする
-- 両方のパス: IDが小さい方を先にロックし、次に大きい方をロック
SELECT * FROM orders WHERE order_id IN (1, 2) ORDER BY order_id FOR UPDATE;
```

**戦略 2: トランザクション開始時にロックを取得**

増分的にではなく、必要なすべてのロックを事前に取得する：

```sql
-- 処理を開始する前に、トランザクションで必要になる全行をロック
SELECT account_id, balance
FROM   accounts
WHERE  account_id IN (:from_acct, :to_acct)
ORDER  BY account_id  -- 一貫した順序
FOR UPDATE;
```

**戦略 3: NOWAIT / 短い WAIT の使用**

待機によるデッドロックを、即座に処理可能な例外に変換する：

```sql
BEGIN
    SELECT * FROM resource_table WHERE resource_id = :id FOR UPDATE NOWAIT;
    -- ... 処理 ...
    COMMIT;
EXCEPTION
    WHEN resource_busy THEN
        -- 短い遅延の後にリトライするか、キューに入れる
        log_retry('Resource busy, retrying...');
        DBMS_SESSION.SLEEP(0.5);
        -- リトライ・ロジック
END;
```

**戦略 4: トランザクション期間の最小化**

ロックを保持する時間が長いほど、デッドロックの機会が増える。バッチ操作の場合は頻繁にコミットする。

---

## 表ロック

Oracleは、行ロックに加えて**表レベルのロック (TMロック)**を取得する。表ロックは、DML実行中に競合するDDLが発生するのを防ぐ。明示的にエスカレーションしない限り、並行するDMLを妨げることはない。

### 明示的な表ロック

```sql
-- 並行する変更を防ぐために表全体をロック
-- (他のDMLをブロックするため、控えめに使用すること)
LOCK TABLE orders IN EXCLUSIVE MODE;
LOCK TABLE orders IN EXCLUSIVE MODE NOWAIT;  -- ロックされていれば失敗

-- 共有モード: DMLは防ぐが、並行する読取りは許可する
LOCK TABLE orders IN SHARE MODE;

-- 行排他: DML中に自動的に取得されるデフォルト・モード
LOCK TABLE orders IN ROW EXCLUSIVE MODE;
```

### 表ロックが必要な場面

アプリケーション・コードで `EXCLUSIVE MODE` の表ロックが必要になることは稀である。主なユースケースは：
- 並行するDMLを完全に禁止したい大量ロード操作
- `ONLINE DDL` が利用できない場合のスキーマ変更
- ETLプロセスの明示的な同期

```sql
-- ETLパターン: 安全なスワップのためにステージング表を排他的にロック
BEGIN
    LOCK TABLE staging_orders IN EXCLUSIVE MODE NOWAIT;

    -- ステージングから本番へのマージ
    MERGE INTO production_orders p
    USING staging_orders s ON (p.order_id = s.order_id)
    WHEN MATCHED THEN UPDATE SET p.status = s.status
    WHEN NOT MATCHED THEN INSERT VALUES (s.order_id, s.status, s.created_at);

    DELETE FROM staging_orders;
    COMMIT;
EXCEPTION
    WHEN resource_busy THEN
        RAISE_APPLICATION_ERROR(-20010, 'Staging table is locked; ETL already running?');
END;
```

---

## ロックの監視クエリ

### アクティブなロックとブロックされたセッション

```sql
-- ブロックしているセッションと、ブロックされている内容を特定
SELECT
    blocker.sid         AS blocking_sid,
    blocker.serial#     AS blocking_serial,
    blocker.username    AS blocking_user,
    blocker.status      AS blocking_status,
    blocker.sql_id      AS blocking_sql_id,
    waiter.sid          AS waiting_sid,
    waiter.username     AS waiting_user,
    waiter.event        AS waiting_event,
    waiter.wait_time_micro / 1e6 AS wait_seconds,
    obj.object_name     AS locked_object,
    obj.object_type
FROM
    v$session blocker
    JOIN v$lock bl ON bl.sid = blocker.sid AND bl.block = 1
    JOIN v$lock wl ON wl.id1 = bl.id1 AND wl.id2 = bl.id2
                   AND wl.request > 0
    JOIN v$session waiter ON waiter.sid = wl.sid
    LEFT JOIN dba_objects obj ON obj.object_id = bl.id1
ORDER BY
    wait_seconds DESC;
```

### ロック待機ツリー (階層型)

```sql
-- 階層クエリを使用して、ロック待機チェーンの全容を表示
SELECT
    LPAD(' ', 2 * (LEVEL - 1)) || sid AS sid,
    username,
    status,
    osuser,
    machine,
    program,
    blocking_session,
    wait_class,
    event,
    seconds_in_wait
FROM
    v$session
WHERE
    status = 'ACTIVE'
    OR blocking_session IS NOT NULL
CONNECT BY PRIOR sid = blocking_session
START WITH blocking_session IS NULL AND status = 'ACTIVE'
ORDER SIBLINGS BY sid;
```

### ブロックされたセッションが実行しているSQLの特定

```sql
SELECT s.sid, s.blocking_session, s.event,
       sq.sql_text, s.seconds_in_wait
FROM   v$session s
JOIN   v$sql sq ON s.sql_id = sq.sql_id
WHERE  s.blocking_session IS NOT NULL;
```

### ロック履歴 (AWR — Diagnostics Pack ライセンスが必要)

```sql
-- 過去1時間のロックに関する待機イベントのトップ
SELECT event, total_waits, time_waited_micro / 1e6 AS total_wait_secs
FROM   v$system_event
WHERE  wait_class = 'Concurrency'
ORDER  BY time_waited_micro DESC;
```

---

## 楽観的ロック vs. 悲観的ロック

### 悲観的ロック (SELECT FOR UPDATE)

更新の根拠となる値を読み取る前に、即座に行をロックする。以下の場合に使用する：
- 行に対する競合が非常に激しい
- 衝突時のリトライが許容できない
- 読取りから更新までの「考慮時間（Think time）」が非常に短い

### 楽観的ロック (Optimistic Locking)

ロックせずに行を読み取る。更新時にのみ、行が変更されていないことを検証する：

```sql
-- 読取り (ロックなし)
SELECT order_id, status, last_modified, ORA_ROWSCN AS read_scn
FROM   orders
WHERE  order_id = 1001;
-- アプリケーションがデータを処理し、ユーザーが考える...

-- ORA_ROWSCN またはバージョン列を使用した衝突検出を伴う更新
UPDATE orders
SET    status = 'APPROVED', last_modified = SYSTIMESTAMP
WHERE  order_id = 1001
  AND  ORA_ROWSCN = :read_scn;  -- 読取り以降に行が変更されていれば失敗する

IF SQL%ROWCOUNT = 0 THEN
    -- 他の誰かに変更された。リトライまたは衝突エラーを発生させる
    RAISE_APPLICATION_ERROR(-20003, 'Conflict: order was modified. Please reload and retry.');
END IF;
COMMIT;
```

**バージョン列を使用した楽観的ロック:**

```sql
-- 表設計
CREATE TABLE orders (
    order_id    NUMBER PRIMARY KEY,
    status      VARCHAR2(20),
    version_no  NUMBER DEFAULT 1 NOT NULL  -- 更新のたびにインクリメント
);

-- バージョン・チェックを伴う更新
UPDATE orders
SET    status = :new_status,
       version_no = version_no + 1
WHERE  order_id = :order_id
  AND  version_no = :read_version;  -- 読み取った値と一致する必要がある

IF SQL%ROWCOUNT = 0 THEN
    RAISE_APPLICATION_ERROR(-20004, 'Stale data: please reload.');
END IF;
```

---

## ベスト・プラクティス

- **競合が少ないシナリオでは楽観的ロックを優先する。** 読取りから更新の間に並行変更がないことを確実に保証する必要がある場合にのみ、`FOR UPDATE` へのエスカレーションを行う。
- **ロック時間を可能な限り短く保つ。** ユーザーとのやり取りの開始時ではなく、DMLの直前にロックを取得する。
- **ネットワーク・ラウンドトリップやユーザー入力を挟んでロックを保持しない。** トランザクションがロックを保持したままユーザーが席を外すと、他の全員がブロックされる。
- **キュー型のワークロードには `SKIP LOCKED` を使用する。** 別のキュー基盤を必要とせず、ワーカーの水平スケーリングが可能になる。
- **一貫した順序でロックを取得し**、デッドロックを防止する。
- **本番環境で `v$lock` と `v$session` を監視し**、ブロッキング・チェーンを確認する。`seconds_in_wait` が閾値を超えた場合にアラートが出るよう設定する。
- **アプリケーション・コードでの `LOCK TABLE IN EXCLUSIVE MODE` を避ける。** これはほとんどの場合、誤った手法であり、シリアル実行のボトルネックを生じさせる。

---

## よくある間違い

### 間違い 1: 書込みによって読取りがブロックされると誤解する

SQL ServerやMySQLの経験がある開発者は、不要な `NOLOCK` ヒントや Read-uncommitted 分離レベルを追加しようとすることがある。Oracleではこれは一切不要である。読取り側が書込み側にブロックされることはない。

### 間違い 2: ORA-00060 をキャッチして無視する

アプリケーションがデッドロック・エラーをキャッチした場合、文をロールバックし（Oracleは文のロールバックを既に行っているが、トランザクションは以前の変更を保持してオープンのままである）、リトライするかトランザクション全体を中止するかを決定する必要がある。

```plpgsql
-- 誤り: 何もなかったかのように続行
EXCEPTION WHEN OTHERS THEN
    IF SQLCODE = -60 THEN NULL; END IF;  -- デッドロックを無視！

-- 正解: ロールバックして対処
EXCEPTION WHEN OTHERS THEN
    IF SQLCODE = -60 THEN
        ROLLBACK;
        retry_or_raise();
    ELSE
        ROLLBACK;
        RAISE;
    END IF;
```

### 間違い 3: 読取り専用のシナリオで SELECT FOR UPDATE を使用する

`SELECT FOR UPDATE` は排他ロックを取得する。アプリケーションがデータを確認するだけで、その後のDMLが発生しない場合、それらのロックはトランザクションが終了するまで無意味に他の書込み側をブロックし続ける。

### 間違い 4: 早まって表ロックにエスカレーションする

バッチ更新中に「念のため」`LOCK TABLE IN EXCLUSIVE MODE` を使用する開発者がいる。これはすべての処理を直列化（シリアライズ）し、並列実行によるメリットを破壊してしまう。代わりに行レベル・ロックとバッチ・コミットを使用すること。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c 概念 (CNCPT) — データの並行性と一貫性](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- [Oracle Database 19c アプリケーション・開発者ガイド (ADFNS)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/)
- [V$LOCK — Oracle Database 19c リファレンス](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOCK.html)
- [V$SESSION — Oracle Database 19c リファレンス](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SESSION.html)
