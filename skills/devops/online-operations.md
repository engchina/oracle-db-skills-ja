# Oracle DB におけるオンライン操作

## 概要

本番環境の Oracle データベースにおいて、従来の DDL 変更（構造変更）は、表のロック、DML のブロック、および読取りの制限など、何らかのダウンタイムを必要としていた。Oracle は、バックグラウンドで構造変更を行いながら、アプリケーションのトラフィックを継続して処理できる一連のオンライン操作機能を提供している。これらの機能を理解し、その制限（制約）を知ることは、ゼロ・ダウンタイムでのデプロイメントを実現するために不可欠である。

主なメカニズムは以下の通りである：

- **DBMS_REDEFINITION** — 複雑な変更（列の入れ替え、型の変更、パーティション化など）を伴う、表のオンライン再定義。
- **索引のオンライン操作** — DML をブロックせずに索引を作成または再構築する。
- **ALTER TABLE ... ONLINE** — ロックを軽減した状態で列の追加や制約の変更を行う。
- **オンライン・セグメント縮小** — 表をオフラインにすることなく領域を回収する。

これらはすべての場合に適用できるわけではなく、それぞれに前提条件、制限事項、および特有の挙動がある。このガイドでは、実用的な例を交えながら、それぞれの使用時期と使用方法について説明する。

---

## DBMS_REDEFINITION (オンライン再定義)

### 概要

`DBMS_REDEFINITION` パッケージを使用すると、テーブルに対する DML 操作を完全に継続したまま、表の構造を再定義できる。プロセスは以下の手順で進む：

1. 新しい構造を持つ**中間表 (interim table)** を作成する。
2. 同期を開始する。Oracle は、マテリアライズド・ビュー・ログを使用して、元の表に適用された DML を追跡する。
3. バックグラウンドで、既存の行を中間表にコピーする。
4. 途中で発生した増分変更を定期的に同期する。
5. 再定義を完了する。Oracle はアトミック（不可分）に表名を入れ替え、元の表を削除する。

アプリケーションからは中断が見えない。最後の入れ替え時に一瞬だけ排他ロックが発生する（通常は数ミリ秒）が、長時間のダウンタイムは発生しない。

### DBMS_REDEFINITION を使用すべきケース

以下のような変更が必要な場合に使用する：

- 列のデータ型の変更（例：`VARCHAR2(100)` から `VARCHAR2(500)`、または `DATE` から `TIMESTAMP`）
- 列の順序の変更（圧縮効率の向上やアプリケーションの互換性のため）
- 通常の表（ヒープ構成表）からパーティション表への変換
- 圧縮の追加または削除
- オンラインでの別の表領域への移動
- 通常の `ALTER TABLE` では処理できない大幅な論理構造の変更

### 前提条件の確認

```sql
-- 表に主キー（または代替の一意キー）が必要
-- 执行権限 (EXECUTE on DBMS_REDEFINITION) が必要
-- 操作中、表のサイズの約 1 倍の追加ストレージ容量が必要

-- 再定義が可能かどうかを確認
BEGIN
  DBMS_REDEFINITION.CAN_REDEF_TABLE(
    uname        => 'APP_OWNER',
    tname        => 'ORDERS',
    options_flag => DBMS_REDEFINITION.CONS_USE_PK  -- または CONS_USE_ROWID
  );
END;
/
-- 例外が発生しなければ、再定義可能
```

### 実行例：通常の表をレンジ・パーティション表に変換する

```sql
-- 手順 1: 新しい構造で中間表を作成
CREATE TABLE ORDERS_NEW (
    ORDER_ID     NUMBER(18,0)  NOT NULL,
    CUSTOMER_ID  NUMBER(18,0)  NOT NULL,
    ORDER_DATE   DATE          NOT NULL,
    STATUS_CODE  VARCHAR2(10)  NOT NULL,
    TOTAL_AMOUNT NUMBER(12,4),
    CREATED_AT   TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT PK_ORDERS_NEW PRIMARY KEY (ORDER_ID, ORDER_DATE)
)
PARTITION BY RANGE (ORDER_DATE)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
  PARTITION P_BEFORE_2024 VALUES LESS THAN (DATE '2024-01-01')
    TABLESPACE DATA_TS
);

-- 手順 2: 再定義の開始
-- 元の表に MV ログが作成され、コピーが開始される
BEGIN
  DBMS_REDEFINITION.START_REDEF_TABLE(
    uname        => 'APP_OWNER',
    orig_table   => 'ORDERS',
    int_table    => 'ORDERS_NEW',
    col_mapping  => 'ORDER_ID ORDER_ID, CUSTOMER_ID CUSTOMER_ID, ' ||
                    'ORDER_DATE ORDER_DATE, STATUS_CODE STATUS_CODE, ' ||
                    'TOTAL_AMOUNT TOTAL_AMOUNT, SYSTIMESTAMP CREATED_AT',
    options_flag => DBMS_REDEFINITION.CONS_USE_PK
  );
END;
/

-- 手順 3: 増分同期 (コピー中に 1 回以上実行)
-- 前回の同期以降に発生した DML を適用し、最終切り替え時間を短縮する
BEGIN
  DBMS_REDEFINITION.SYNC_INTERIM_TABLE('APP_OWNER', 'ORDERS', 'ORDERS_NEW');
END;
/

-- 手順 4: 依存オブジェクト（制約、索引、トリガーなど）をコピー
DECLARE
  v_num_errors PLS_INTEGER;
BEGIN
  DBMS_REDEFINITION.COPY_TABLE_DEPENDENTS(
    uname            => 'APP_OWNER',
    orig_table       => 'ORDERS',
    int_table        => 'ORDERS_NEW',
    copy_indexes     => DBMS_REDEFINITION.CONS_ORIG_PARAMS,
    copy_triggers    => TRUE,
    copy_constraints => TRUE,
    copy_privileges  => TRUE,
    ignore_errors    => FALSE,
    num_errors       => v_num_errors
  );
END;
/

-- 手順 5: 最終同期とアトミックな入れ替え
BEGIN
  DBMS_REDEFINITION.FINISH_REDEF_TABLE('APP_OWNER', 'ORDERS', 'ORDERS_NEW');
END;
/

-- 手順 6: 後片付け
-- 新しい表が正しいことを確認後、古い中間表を削除
DROP TABLE ORDERS_NEW PURGE;
```

---

## 索引のオンライン操作

### 索引のオンライン再構築 (REBUILD ONLINE)

索引は時間の経過とともに、構造的な非効率性（削除によるリーフ・ブロックの断片化、キーの偏りによる高さの増大など）が蓄積される。オンライン再構築により、アプリケーションを停止させずにこれらを修正できる。

```sql
-- オフライン再構築: 完了までテーブルへの DML がブロックされる
ALTER INDEX IDX_ORDERS_CUSTOMER REBUILD;

-- オンライン再構築: DML を継続可能。ジャーナル表を使用して変更を追跡する
-- オフラインより時間がかかり I/O 負荷も高いが、アプリケーションを停止させない
ALTER INDEX IDX_ORDERS_CUSTOMER REBUILD ONLINE;

-- 並列実行の例
ALTER INDEX IDX_ORDERS_CUSTOMER
REBUILD ONLINE
TABLESPACE IDX_TS
PARALLEL 4
COMPUTE STATISTICS;
```

### 索引のオンライン作成 (CREATE INDEX ONLINE)

大規模な本番環境の表に新しい索引を追加する場合の標準的な手法である。

```sql
CREATE INDEX IDX_ORDERS_STATUS
    ON ORDERS (STATUS_CODE, ORDER_DATE)
    ONLINE
    TABLESPACE IDX_TS
    PARALLEL 4
    NOLOGGING;  -- REDO 生成を抑える（作成後にバックアップが必要）

-- 作成完了後、ロギングを有効に戻す
ALTER INDEX IDX_ORDERS_STATUS LOGGING;
```

**索引のオンライン操作の制限:**
- **ビットマップ索引はオンラインで作成・再構築できない。**（そもそもビットマップ索引は DML が頻発する表には向かない）
- オンライン操作は、オフラインの場合よりも多くの Undo 領域と一時（Temporary）領域を消費する。

---

## ALTER TABLE ... ONLINE

Oracle 12c 以降、いくつかの `ALTER TABLE` 操作で `ONLINE` 句がサポートされ、ロック競合が大幅に軽減された。

### 列の追加

```sql
-- 12c以降: デフォルト値を持つ NOT NULL 列の追加はメタデータのみの変更となり、瞬時に完了する
ALTER TABLE ORDERS
ADD (LAST_MODIFIED TIMESTAMP DEFAULT SYSTIMESTAMP NOT NULL);

-- 明示的にオンライン操作を指定 (12c以降ではデフォルトでロックが最小化されるが、明示も可能)
ALTER TABLE ORDERS ADD (AUDIT_USER VARCHAR2(100)) ONLINE;
```

### 列を未使用 (UNUSED) に設定

列を即座に削除する（すべての行を書き換える）のではなく、まず「未使用」としてマークする。アプリケーションからは即座に見えなくなり、物理的な領域回収は後回しにできる。

```sql
-- 手順 1: アプリケーションから見えなくする (一瞬で完了)
ALTER TABLE ORDERS SET UNUSED COLUMN LEGACY_REF_NUM;

-- 手順 2: 負荷の低い時間帯などに領域を回収
ALTER TABLE ORDERS DROP UNUSED COLUMNS;
```

---

## オンライン・セグメント縮小 (SHRINK SPACE)

行の削除によって発生した表内の空き領域を回収する。表を移動させたりオフラインにしたりする必要はない。

```sql
-- 手順 1: 行移動を有効化 (縮小中に行の ROWID が変わる可能性があるため)
ALTER TABLE ORDERS ENABLE ROW MOVEMENT;

-- 手順 2: 行のコンパクション (HWM はまだ動かさない)
ALTER TABLE ORDERS SHRINK SPACE COMPACT;

-- 手順 3: ハイ・ウォーター・マーク (HWM) の調整 (短い排他ロックが発生)
ALTER TABLE ORDERS SHRINK SPACE;

-- 一括実行 (推奨)
ALTER TABLE ORDERS SHRINK SPACE CASCADE;  -- 依存する索引も縮小
```

---

## ダウンタイムを最小化する戦略のまとめ

### 追加的な変更 (Zero Downtime)

これらはアプリケーションの稼働中に適用しても安全である：

- null 許可の列の追加
- デフォルト値を持つ NOT NULL 列の追加 (12c+)
- 新しい索引の作成 (`CREATE INDEX ... ONLINE`)
- 新しい表、シーケンス、シノニムの作成
- 制約の追加 (`ENABLE NOVALIDATE` を使用)
- ビュー、パッケージ、プロシージャの作成または置換（意味的に互換性がある場合）

### 破壊的またはリスクのある変更 (計画が必要)

これらは順序立てたデプロイメント（Expand/Contract パターンなど）が必要となる：

| 変更内容 | リスク | 対応策 |
|---|---|---|
| 列の削除 | アプリケーションがまだ参照している可能性 | まず UNUSED に設定し、次期リリースで削除 |
| 列名の変更 | アプリケーションが即座にエラーになる | 新列追加、データ移行、アプリ切替、旧列削除 |
| 列型の変更 | 大規模表では長時間ロックや失敗の恐れ | DBMS_REDEFINITION を使用 |
| 通常表からパーティション化 | `ALTER TABLE` では不可 | DBMS_REDEFINITION を使用 |

---

## ベスト・プラクティス

- **再定義の前に必ず `DBMS_REDEFINITION.CAN_REDEF_TABLE` を実行する。** 大規模な表の操作の途中で失敗すると、修復が非常に困難になる。
- **`V$SESSION_LONGOPS` を監視する。** オンライン操作は数時間かかることもある。進捗の可視化は運用上不可欠である。
- **一時領域を十分に確保する。** オンライン操作は、元のオブジェクトと同等程度の追加ストレージ容量（中間表や索引作成用）を必要とする。
- **並列度 (PARALLEL) の活用。** 負荷の低い時間帯に高い並列度で実行し、完了後は `NOPARALLEL` に戻す。
- **事前に非本番環境でテストする。** LOB 列や特定の索引タイプなど、オンライン操作に制約がある場合がある。

---

## よくある間違い

**間違い: 「オンライン」＝「一瞬で終わる」と思い込む。**
オンライン操作は DML をブロックしないだけであり、完了までには時間、I/O 負荷、一時領域を消費する。500 GB の表の再定義には数時間かかることを想定しておくべきである。

**間違い: `FINISH_REDEF_TABLE` の前に同期を行わない。**
最終的な入れ替え時にも同期は行われるが、事前に `SYNC_INTERIM_TABLE` を手動で数回実行しておくことで、最終切り替え時の排他ロック時間を最短にできる。

**間違い: 縮小 (Shrink) 後に `ROW MOVEMENT` を有効なままにする。**
ROWID をキャッシュして高速検索を行うアプリケーションがある場合、縮小後に ROWID が変わっているため、誤ったデータを取得する可能性がある。縮小後は `DISABLE ROW MOVEMENT` に戻す。

**間違い: データの完全削除に `SET UNUSED` を頼る。**
`SET UNUSED` は SQL から見えなくするだけで、データ・ファイル上にはまだデータが残っている。機密データの削除が目的の場合は、UPDATE で null クリアしてから実行すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [DBMS_REDEFINITION (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_REDEFINITION.html)
- [Oracle Database Administrator's Guide 19c — Online Operations](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tables.html)
- [Oracle Database SQL Language Reference 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [Oracle ALTER TABLE SHRINK SPACE — Online Segment Shrink](https://oracle-base.com/articles/misc/alter-table-shrink-space-online)
