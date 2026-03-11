# Oracle undo管理

## 概要

undoデータは、トランザクションによって変更される前のデータのコピーである。Oracleは、トランザクションのロールバック、読取り一貫性の提供（問合せ開始時のデータの表示）、およびFlashback機能（過去のデータの参照）のためにundoを使用する。

以前のバージョンのOracleでは「ロールバック・セグメント」を手動で管理していたが、現在のすべてのバージョンでは**自動undo管理 (AUM)**を使用する。AUMでは、Oracleが専用の**undo表領域**内のundoセグメントを自動的に作成、拡張、および調整する。

---

## undoの主な目的

1. **トランザクションのロールバック:** ユーザーが `ROLLBACK` 命令を発行した際に、変更を取り消して元の状態に戻す。
2. **読取り一貫性 (Read Consistency):** あるセッションがデータを更新中でも、別のセッションがそのデータの「クリーン」な（変更前の）コピーを読み取れるようにする。
3. **トランザクションのリカバリ:** インスタンス・クラッシュ後、コミットされていないトランザクションをロールバックするために使用する。
4. **Flashback機能:** `FLASHBACK QUERY` などの機能を使用して、過去のある時点のデータを表示する。

---

## 自動undo管理 (AUM) の構成

自動undo管理を制御する主なパラメータは以下の通り。

### 初期化パラメータ

```sql
-- 現在のundo設定を確認
SHOW PARAMETER undo;

-- undo管理モード (必須: AUTO)
ALTER SYSTEM SET undo_management = AUTO SCOPE=SPFILE;

-- 使用するundo表領域の指定
ALTER SYSTEM SET undo_tablespace = UNDOTBS1 SCOPE=BOTH;

-- undoデータが上書きされずに保持される秒数 (目標値)
ALTER SYSTEM SET undo_retention = 900 SCOPE=BOTH; -- 15分
```

### undo表領域の管理

undo表領域は他の表領域と同様に管理されるが、undoデータのみを格納する。

```sql
-- 新しいundo表領域の作成 (推奨: データファイルは自動拡張に設定)
CREATE UNDO TABLESPACE undotbs_new
  DATAFILE '/oradata/undo/undotbs_new01.dbf' SIZE 2G
  AUTOEXTEND ON NEXT 512M MAXSIZE 32G;

-- アクティブなundo表領域の切り替え
ALTER SYSTEM SET undo_tablespace = undotbs_new;

-- 古いundo表領域の削除 (すべてのトランザクションが終了した後に実行可能)
DROP TABLESPACE undotbs_old INCLUDING CONTENTS AND DATAFILES;
```

---

## undo保持 (Undo Retention)

`UNDO_RETENTION` パラメータは、Oracleがコミット済みエントリを再利用する前に、どれくらいの期間（秒単位）保持しようとするかを指定する。

- **保証なし (デフォルト):** 領域が不足すると、Oracleは `UNDO_RETENTION` の期間を待たずにコミット済みデータを上書きすることがある。
- **保証あり (RETENTION GUARANTEE):** 領域不足になっても、指定された期間は絶対に上書きしない。領域がなくなると、新しいトランザクションはエラー（ORA-30036）で失敗する。

```sql
-- 保持の「保証」を有効化
ALTER TABLESPACE undotbs1 RETENTION GUARANTEE;

-- 保持の「保証」を無効化 (デフォルト)
ALTER TABLESPACE undotbs1 RETENTION NOGUARANTEE;
```

---

## 監視とトラブルシューティング

### undo使用状況の確認

```sql
-- undo表領域の使用率
SELECT tablespace_name,
       SUM(bytes)/1024/1024 size_mb,
       SUM(maxbytes)/1024/1024 max_size_mb
FROM dba_data_files
WHERE tablespace_name LIKE 'UNDO%'
GROUP BY tablespace_name;

-- アクティブなトランザクションによるundo使用量
SELECT s.username, t.used_ublk, t.used_urec, t.start_time
FROM v$session s, v$transaction t
WHERE s.saddr = t.ses_addr;
```

### ORA-01555: snapshot too old

このエラーは、問合せに必要な古いデータ（undo）が、問合せが終わる前に他のトランザクションによって上書きされた場合に発生する。

**原因:**
- undo表領域が小さすぎる。
- `UNDO_RETENTION` の設定値が短すぎる。
- 非常に長い時間がかかる問合せが大量に実行されている間に、別のセッションが大量の更新・コミットを繰り返している。

**対策:**
- undo表領域のサイズを大きくする。
- `UNDO_RETENTION` を問合せの実行時間より長く設定する。
- `RETENTION GUARANTEE` の使用を検討する。

### ORA-30036: unable to extend segment by %s in undo tablespace

これは、undo表領域がいっぱいになり、新しいトランザクションのためのスペースを確保できなかった場合に発生する。

**原因:**
- undo表領域の自動拡張がオフになっている、または最大サイズ（MAXSIZE）に達した。
- 保持期間（GUARANTEE）が長すぎて、古いデータが解放されない。
- 巨大なトランザクションが実行されている。

---

## undoアドバイザ (Undo Advisor)

Oracleは、過去の負荷に基づいて適切なundo表領域のサイズをアドバイスしてくれる。

```sql
-- V$UNDOSTAT を参照して過去7日間の履歴を確認
SELECT begin_time, end_time, undoblks, txncount, maxquerylen, unexpiredblks
FROM v$undostat
ORDER BY begin_time DESC;

-- 必要な最小サイズを見積もる (例: 保持期間1800秒の場合)
-- 式: (Max Undo Blocks Per Second * Undo Retention) * DB Block Size
```

---

## ベスト・プラクティス

- **自動拡張 (AUTOEXTEND) を使用する:** 予期せぬ巨大なトランザクションや、LOBデータの更新に対応するため。ただし、ディスク容量を使い果たさないよう `MAXSIZE` を設定すること。
- **適切なサイズ設定:** 少なくとも、ピーク時の最大問合せ実行時間 (`MAXQUERYLEN`) 以上の `UNDO_RETENTION` を確保できるサイズに設定する。
- **1つのDBに1つのUNDO表領域:** 基本的な構成では、インスタンスごとに1つのアクティブなundo表領域を割り当てる（RACの場合はインスタンスごとに1つ）。
- **LOBデータの考慮:** LOB（CLOB/BLOB）のundoは、専用の領域管理（SecureFilesなど）で行われるため、通常の `UNDO_RETENTION` とは異なる動作をすることがある。

---

## よくある間違い

- **巨大なトランザクションを1つで実行する:** 何百万行もの更新を一気にコミットすると、undo表領域を急激に消費する。必要に応じてバッチ（小分け）処理を検討する。
- **古い「ロールバック・セグメント」のスクリプトを使用する:** 現代のOracleでは手動管理（Manual Undo Management）は非推奨（将来的に廃止）である。必ず `undo_management=AUTO` を使用する。
- **監視を怠る:** ORA-01555が頻発してから対処するのではなく、`V$UNDOSTAT` を定期的にチェックして、`MAXQUERYLEN` が `UNDO_RETENTION` に近づいていないか確認する。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 19cでは「Flashback Data Archive」などの機能がundoを補完するが、23ai以降ではベクター・データ型のundo管理など、AI機能に関連する内部的な拡張が含まれる場合がある。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 管理者ガイド 19c — undoの管理](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-undo.html)
- [Oracle Database 19c リファレンス — V$UNDOSTAT](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-UNDOSTAT.html)
- [Oracle Database 23ai — New Features Guide](https://docs.oracle.com/en/database/oracle/oracle-database/23/nfcvw/)
