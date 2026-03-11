# Oracle Databaseにおけるトランザクション管理

## 概要

トランザクションは、1つ以上のSQL文で構成される論理的な作業単位である。Oracleのトランザクション・モデルはリレーショナル・データベースの世界で最も堅牢なものの1つであり、マルチバージョン並行性制御（MVCC）の実装を通じて高い並行性を可能にしながら、完全なACID特性を保証する。

Oracleがどのようにトランザクションを管理するかを理解することは、正しくパフォーマンスの高いアプリケーションを書くために不可欠である。コミットされていないトランザクションを長く保持しすぎたり、セーブポイントを不適切に使用したり、自律型トランザクションを誤解したりといった微妙な間違いは、データ破損、ロック競合、およびデッドロックの最も一般的な原因となる。

---

## OracleにおけるACID特性

### 原子性 (Atomicity)

トランザクション内のすべての文は、完全に成功するか、完全に失敗するかのどちらかである。文の実行途中で失敗が発生した場合（例：一意制約違反）、Oracleは自動的にその単一の文の変更を**文レベルのロールバック**を介して取り消すが、トランザクション自体はそれ以前の変更を保持したままオープンの状態を維持する。

```sql
-- 文レベルのロールバックのデモンストレーション
INSERT INTO orders (order_id, customer_id) VALUES (1, 101);  -- 成功
INSERT INTO orders (order_id, customer_id) VALUES (1, 102);  -- 失敗: 主キーの重複
-- この時点で、最初の INSERT はまだ保留中（トランザクションはオープン）
-- 2番目の INSERT は自動的にロールバックされた
COMMIT;  -- 最初の INSERT だけがコミットされる
```

### 一貫性 (Consistency)

Oracleは（デフォルトで）トランザクションのコミット時にすべての整合性制約を適用する。トランザクション内で制約チェックを遅延させることができる：

```sql
-- コミットまで制約チェックを遅延させる
ALTER TABLE child_table
    MODIFY CONSTRAINT fk_parent DEFERRABLE INITIALLY DEFERRED;

-- セッション内で遅延チェックを有効にする
SET CONSTRAINTS ALL DEFERRED;

-- これで、同一トランザクション内で親の前に子を挿入できる
INSERT INTO child_table (id, parent_id) VALUES (1, 999);
INSERT INTO parent_table (id) VALUES (999);
COMMIT;  -- ここで制約がチェックされる。親が存在しなければロールバックされる
```

### 分離性 (Isolation)

Oracleは**MVCC (マルチバージョン並行性制御)**を通じて分離性を実装している。読取り側が書込み側をブロックすることはなく、書込み側が読取り側をブロックすることもない。各クエリは、その開始時点（またはシリアライザブル・モードの場合はトランザクション開始時点）の一貫したデータのスナップショットを参照する。

Oracleは2つの標準的な分離レベルをサポートしている：

| 分離レベル | 説明 | Oracleのデフォルト? |
|---|---|---|
| `READ COMMITTED` | 各文は、その文が開始される前にコミットされたデータを参照する | はい |
| `SERIALIZABLE` | トランザクションは開始時点のデータを参照する。開始後にデータが変更されているとエラーになる | いいえ (明示的設定が必要) |

```sql
-- 現在のトランザクションに SERIALIZABLE 分離レベルを設定
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- またはセッション全体に設定
ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;
```

注：Oracleは `READ UNCOMMITTED`（ダーティ・リード）をサポートして**いない**。これは設計によるものであり、MVCCによってその必要性が排除されているためである。

### 永続性 (Durability)

`COMMIT` が正常に返されると、Oracleはそのデータがディスク上のredoログに書き込まれたことを保証する。ログ・ライター (LGWR) は、COMMITが返る前にログ・エントリをディスクにフラッシュする必要がある。

```sql
-- 同期的なredo書込みを強制する (デフォルトの動作)
COMMIT WRITE IMMEDIATE WAIT;

-- 非同期コミット (スループットは向上するが、永続性の窓がわずかに減少する)
COMMIT WRITE IMMEDIATE NOWAIT;

-- バッチ・コミット (多少の損失が許容される大量ロードに最適)
COMMIT WRITE BATCH NOWAIT;
```

---

## 開始、コミット、およびロールバック

Oracleでは、最初のDML文でトランザクションが暗黙的に開始される。SQL*Plusやほとんどのツールにおいて、明示的な `BEGIN TRANSACTION` は存在しない（PL/SQLには `BEGIN ... END` ブロックがあるが、これはトランザクションの境界ではない）。

```sql
-- 最初のDMLでトランザクションが暗黙的に開始される
INSERT INTO accounts (account_id, balance) VALUES (1001, 5000);
UPDATE accounts SET balance = balance - 500 WHERE account_id = 1001;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2001;

-- COMMIT により変更を確定し、すべてのロックを解放する
COMMIT;

-- ROLLBACK により前回のコミット以降のすべての変更を破棄し、ロックを解放する
ROLLBACK;
```

### コミットのベスト・プラクティス

```sql
-- 良い例: 論理的な作業単位の後にコミットする
BEGIN
    -- 資金を原子的に移動
    UPDATE accounts SET balance = balance - p_amount
    WHERE  account_id = p_from_account;

    UPDATE accounts SET balance = balance + p_amount
    WHERE  account_id = p_to_account;

    INSERT INTO transfer_log (from_acct, to_acct, amount, transfer_date)
    VALUES (p_from_account, p_to_account, p_amount, SYSDATE);

    COMMIT;  -- これら3つの変更はまとめてコミットされる
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;  -- これら3つの変更はすべて破棄される
        RAISE;
END;
```

---

## セーブポイント (Savepoints)

セーブポイントを使用すると、トランザクション内で部分的なロールバックが可能になる。`ROLLBACK TO savepoint_name` は、セーブポイント確立後のすべての変更を取り消すが、トランザクションを終了させることはない。それ以前の変更とセーブポイント自体は維持される。

```sql
INSERT INTO orders (order_id, status) VALUES (1001, 'PENDING');
SAVEPOINT after_order;

INSERT INTO order_items (order_id, item_id, qty) VALUES (1001, 'WIDGET', 5);
SAVEPOINT after_first_item;

INSERT INTO order_items (order_id, item_id, qty) VALUES (1001, 'GADGET', 2);
-- このアイテムが在庫切れだと仮定し、これだけを取り消すが、注文と最初のアイテムは保持する
ROLLBACK TO after_first_item;

-- トランザクションはまだオープン。注文と最初のアイテムは保留中のまま
COMMIT;  -- 注文と最初のアイテムのみをコミット
```

### PL/SQLエラー処理におけるセーブポイント

```plpgsql
DECLARE
    e_invalid_item EXCEPTION;
BEGIN
    INSERT INTO orders (order_id, customer_id, order_date)
    VALUES (seq_orders.NEXTVAL, 42, SYSDATE);

    SAVEPOINT order_created;

    FOR item IN (SELECT * FROM staging_order_items WHERE session_id = :sid) LOOP
        BEGIN
            INSERT INTO order_items (order_id, product_id, quantity, price)
            VALUES (item.order_id, item.product_id, item.quantity,
                    get_current_price(item.product_id));
        EXCEPTION
            WHEN OTHERS THEN
                -- 不正なアイテムはスキップし、正常なものは保持する
                ROLLBACK TO order_created;
                log_error('Failed to insert item: ' || item.product_id);
        END;
    END LOOP;

    COMMIT;
END;
```

**重要:** `ROLLBACK TO savepoint` は、そのセーブポイント以降に取得された行ロックを解放するが、それ以前に取得されたロックは解放**しない**。

---

## 自律型トランザクション (Autonomous Transactions)

**自律型トランザクション**は、呼び出し元のトランザクション内から生成される独立したトランザクションである。独自のコミット/ロールバック・スコープを持ち、親トランザクションとは完全に独立している。自律型トランザクションによる変更は、呼び出し元のトランザクションがコミットされる前であっても、コミット直後に他のセッションから参照可能になる。

`PRAGMA AUTONOMOUS_TRANSACTION` コンパイラ・ディレクティブを使用して有効にする。

### 主なユースケース

1. **エラー・ログの記録** — 呼び出し元のトランザクションがロールバックされてもエラーを記録する
2. **監査証跡（Audit Trails）** — ロールバックを生き残る必要がある監査レコードを挿入する
3. **シーケンス番号の生成** — 欠番を追跡する必要がある稀なケース

```sql
-- 自律型トランザクションを使用した監査/ログ用プロシージャ
CREATE OR REPLACE PROCEDURE log_audit_event (
    p_action   IN VARCHAR2,
    p_table    IN VARCHAR2,
    p_row_id   IN VARCHAR2,
    p_details  IN VARCHAR2
) AS
    PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO audit_log (log_id, action, table_name, row_id, details, log_time, logged_by)
    VALUES (seq_audit.NEXTVAL, p_action, p_table, p_row_id, p_details, SYSTIMESTAMP, USER);

    COMMIT;  -- 必須: 自律型トランザクションは明示的にコミットまたはロールバックする必要がある
END log_audit_event;
/

-- 使用例: 外部トランザクションがロールバックされても、監査レコードは残る
BEGIN
    UPDATE sensitive_data SET value = 'changed' WHERE id = 42;
    log_audit_event('UPDATE', 'SENSITIVE_DATA', '42', 'Value changed');
    -- ここでロールバックしても、log_audit_event の INSERT は既にコミットされている
    ROLLBACK;  -- sensitive_data の変更は取り消されるが、監査レコードは残る
END;
```

### エラー・ログ表パターン

```sql
CREATE TABLE error_log (
    log_id    NUMBER GENERATED ALWAYS AS IDENTITY,
    error_msg VARCHAR2(4000),
    error_code NUMBER,
    program   VARCHAR2(100),
    log_time  TIMESTAMP DEFAULT SYSTIMESTAMP,
    CONSTRAINT pk_error_log PRIMARY KEY (log_id)
);

CREATE OR REPLACE PROCEDURE log_error (
    p_msg     IN VARCHAR2,
    p_code    IN NUMBER DEFAULT SQLCODE,
    p_program IN VARCHAR2 DEFAULT $$PLSQL_UNIT
) AS
    PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
    INSERT INTO error_log (error_msg, error_code, program)
    VALUES (SUBSTR(p_msg, 1, 4000), p_code, p_program);
    COMMIT;
END log_error;
```

### 自律型トランザクションの注意点

```sql
-- 誤り: 親からの未コミット・データを読み取ろうとする自律型トランザクション
-- 親が1行挿入したが、まだコミットしていない
-- 自律型トランザクションは親の未コミットの変更を参照できない
CREATE OR REPLACE PROCEDURE bad_autonomous AS
    PRAGMA AUTONOMOUS_TRANSACTION;
    v_count NUMBER;
BEGIN
    -- これは、親の保留中の挿入ではなく、テーブルの「コミット済み」状態を読み取る
    SELECT COUNT(*) INTO v_count FROM orders WHERE status = 'PENDING';
    COMMIT;
END;
```

---

## 分散トランザクション (XA)

Oracleは、複数のデータベースまたはリソース・マネージャ（データベース + メッセージ・キュー）にまたがる分散トランザクションのためのX/Open XAプロトコルをサポートしている。

### 2フェーズ・コミット (2PC)

Oracleは2PCにおいて、**コーディネータ**または**参加者**として動作する：

```sql
-- フェーズ 1: Prepare (準備。コーディネータが全参加者に準備を要求する)
-- 各参加者は準備完了状態を自らのredoログに書き込む

-- フェーズ 2: Commit または Rollback (コーディネータの決定)
-- 全参加者がコミットするか、全参加者がロールバックする

-- 未確定の（in-doubt）分散トランザクションの監視
SELECT local_tran_id, global_tran_id, state, mixed, advice, tran_comment
FROM   dba_2pc_pending;

-- 未確定トランザクションを強制的にコミットする（ネットワーク復旧後など）
COMMIT FORCE 'local_tran_id';

-- 未確定トランザクションを強制的にロールバックする
ROLLBACK FORCE 'local_tran_id';
```

### トランザクション内でのデータベース・リンク

```sql
-- DBリンクに触れると、トランザクションは自動的に分散トランザクションになる
UPDATE orders@remote_db SET status = 'SHIPPED' WHERE order_id = :id;
UPDATE local_inventory SET qty = qty - 1 WHERE product_id = :prod;
COMMIT;  -- Oracleは自動的に remote_db との間で2PCを実行する
```

### JDBC XA の例

```java
import javax.sql.XADataSource;
import javax.transaction.xa.XAResource;
import javax.transaction.xa.Xid;

// JTAトランザクション・マネージャ (Atomikos, Bitronix, Narayana 等) を使用したXA
XAConnection xaConn = xaDataSource.getXAConnection("user", "password");
XAResource xaResource = xaConn.getXAResource();

Xid xid = createXid();  // アプリケーション定義のトランザクションID

xaResource.start(xid, XAResource.TMNOFLAGS);
try (Connection conn = xaConn.getConnection()) {
    conn.prepareStatement("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
        .executeUpdate();
}
xaResource.end(xid, XAResource.TMSUCCESS);

int prepareResult = xaResource.prepare(xid);
if (prepareResult == XAResource.XA_OK) {
    xaResource.commit(xid, false);
}
```

---

## 長時間実行されるトランザクションの回避

長時間実行されるトランザクションは、Oracleアプリケーションにおけるパフォーマンス問題の最も一般的な原因である。これらは以下の影響を及ぼす：

- 他のセッションをブロックする行ロックを保持し続ける
- トランザクションが完了するまで保持する必要のあるundoデータを生成し続ける
- 他の長時間実行されるクエリに対して `ORA-01555: snapshot too old` を引き起こす可能性がある
- undo表領域を消費し、ORA-30036の原因となる可能性がある

### 長時間実行されるトランザクションの検出

```sql
-- オープンなトランザクションを持つセッションを特定
SELECT s.sid, s.serial#, s.username, s.status, s.program,
       t.start_time, t.used_ublk AS undo_blocks_used,
       ROUND((SYSDATE - TO_DATE(t.start_time,'MM/DD/YY HH24:MI:SS')) * 24 * 60, 1)
           AS minutes_open
FROM   v$session s
JOIN   v$transaction t ON s.taddr = t.addr
ORDER  BY minutes_open DESC;

-- undo使用状況の確認
SELECT usn, xacts, rssize/1024/1024 AS mb_used, status
FROM   v$rollstat rs
JOIN   v$rollname rn ON rs.usn = rn.usn
ORDER  BY mb_used DESC;
```

### バッチ処理パターン — バッチごとにコミット

```plpgsql
-- 誤り: 何百万行に対しても1回のコミット
BEGIN
    FOR r IN (SELECT id FROM large_table WHERE needs_processing = 'Y') LOOP
        UPDATE large_table SET processed_date = SYSDATE WHERE id = r.id;
    END LOOP;
    COMMIT;  -- ループ中ずっとすべてのロックを保持してしまう
END;

-- 正解: N行ごとにコミット
DECLARE
    v_batch_size CONSTANT PLS_INTEGER := 1000;
    v_count      PLS_INTEGER := 0;
BEGIN
    FOR r IN (SELECT id FROM large_table WHERE needs_processing = 'Y') LOOP
        UPDATE large_table SET processed_date = SYSDATE WHERE id = r.id;
        v_count := v_count + 1;

        IF MOD(v_count, v_batch_size) = 0 THEN
            COMMIT;
        END IF;
    END LOOP;
    COMMIT;  -- 最後のバッチ
END;
```

### 大量DMLのための FORALL の使用

```plpgsql
DECLARE
    TYPE id_array IS TABLE OF NUMBER;
    v_ids   id_array;
    CURSOR c_pending IS
        SELECT id FROM large_table WHERE needs_processing = 'Y';
BEGIN
    OPEN c_pending;
    LOOP
        FETCH c_pending BULK COLLECT INTO v_ids LIMIT 5000;
        EXIT WHEN v_ids.COUNT = 0;

        FORALL i IN 1..v_ids.COUNT
            UPDATE large_table
            SET    processed_date = SYSDATE
            WHERE  id = v_ids(i);

        COMMIT;
    END LOOP;
    CLOSE c_pending;
END;
```

---

## ベスト・プラクティス

- **トランザクションは可能な限り短く保つ。** すべての計算をトランザクション前に行い、DMLを実行し、直ちにコミットする。
- **トランザクション内でユーザー入力を待たない。** ユーザーが席を外すと、ロックが保持されたままになってしまう。
- **常に例外を処理し、ROLLBACK を呼び出す。** アプリケーション・コードでハンドルされない例外が発生し、ロールバックなしで切断されると、セッションが強制終了されるまでトランザクションがオープンのまま残る。
- **`COMMIT WRITE BATCH NOWAIT` は大量ロードにのみ使用する**（永続性とのトレードオフを理解した上で行う）。
- **`FORALL` やバルク操作を優先する。** 行単位のDMLよりもコミット頻度を下げつつ、トランザクションを短く保つことができる。
- **`SERIALIZABLE` 分離レベルでのテスト。** アプリケーションのロジックが複数の文にわたる一貫した読取りに依存している場合に行う。`READ COMMITTED` がトランザクション・レベルでスナップショットの一貫性を提供すると仮定してはならない。
- **ロック問題を回避するために `PRAGMA AUTONOMOUS_TRANSACTION` を使用しない。** デバッグが困難な目に見えないデータの依存関係を生み出してしまう。

---

## よくある間違い

### 間違い 1: ロールバックなしで例外を握りつぶす

```plpgsql
-- 誤り: 例外はキャッチされているが、トランザクションがロールバックされていない
BEGIN
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;
EXCEPTION
    WHEN OTHERS THEN
        -- 沈黙のうちに握りつぶされる — 最初の UPDATE がどこかでコミットされる可能性がある
        NULL;
END;

-- 正解
BEGIN
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;  -- 呼び出し元が把握できるように再スローする
END;
```

### 間違い 2: トランザクションの途中で DDL を実行する

Oracleでは、あらゆるDDL文（`CREATE`, `ALTER`, `DROP`, `TRUNCATE`）の実行前後に暗黙的な COMMIT が発行される。これにより、保留中のDMLが意図せずコミットされてしまう。

```sql
INSERT INTO temp_data VALUES (1, 'test');  -- DML
CREATE INDEX idx_temp ON temp_data(id);    -- この直前に暗黙の COMMIT が発生！
ROLLBACK;  -- 遅すぎる。INSERT は既にコミットされている
```

### 間違い 3: JDBC のオートコミットをそのままにする

JDBC接続のデフォルトは `autoCommit=true` である。すべての文が直ちにコミットされる。これは OLTP においてほとんどの場合、望ましい動作ではない。

```java
// 接続取得後、直ちにオートコミットを無効にする
connection.setAutoCommit(false);
// ... DMLを実行 ...
connection.commit();  // または catch ブロックで connection.rollback()
```

### 間違い 4: 文レベルとトランザクション・レベルのロールバックの混同

DML文が例外で失敗した場合、その文だけがロールバックされる。トランザクションはオープンのままである。例外をキャッチして何もしなければ、トランザクション内の以前の文はコミットされず、ロックも保持されたままになる。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c 概念 (CNCPT) — トランザクション](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/)
- [Oracle Database 19c アプリケーション・開発者ガイド (ADFNS)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/)
- [Oracle Database 19c PL/SQL言語リファレンス — トランザクション処理](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/)
