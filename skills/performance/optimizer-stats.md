# オプティマイザ統計 — 収集、管理、およびチューニング

## 概要

Oracle のコストベース・オプティマイザ (CBO) は、さまざまな実行計画のコストを見積もるために、すべて統計情報に依存している。正確な統計情報があれば、オプティマイザは効率的な計画を選択できるが、統計情報が古い、あるいは存在しない場合、誤ったカーディナリティの見積もり、間違った結合順序、そして不適切な実行計画の選択を招くことになる。

統計情報には以下が含まれる：
- **表統計:** 行数、ブロック数、平均行長
- **列統計:** 個別の値の数 (NDV)、NULL 数、最小値/最大値、ヒストグラム
- **索引統計:** クラスタ化係数、リーフ・ブロック数、索引の高さ
- **システム統計:** CPU 速度、I/O スループット (マルチブロック読み取りコスト、シングルブロック読み取りコスト)

統計情報を管理するための主要なツールは `DBMS_STATS` パッケージである。

---

## DBMS_STATS による統計情報の収集

### 表統計の収集

```sql
-- 自動サンプル・サイズを使用した基本的な収集 (推奨されるデフォルト)
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname   => 'HR',
    tabname   => 'EMPLOYEES',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,  -- Oracle がサンプル量を決定
    method_opt       => 'FOR ALL COLUMNS SIZE AUTO',   -- 自動ヒストグラム
    cascade          => TRUE,                          -- 索引統計も収集
    no_invalidate    => FALSE                          -- カーソルを即座に無効化
  );
END;
/
```

### スキーマ統計の収集

```sql
-- スキーマ内のすべての表を収集
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS(
    ownname          => 'HR',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
    cascade          => TRUE,
    degree           => 4,          -- 大規模なスキーマ向けのパラレル次数
    options          => 'GATHER'    -- GATHER, GATHER STALE, GATHER EMPTY, または LIST STALE
  );
END;
/
```

### データベース統計の収集

```sql
-- すべてのスキーマを収集 (DBA 権限が必要)
BEGIN
  DBMS_STATS.GATHER_DATABASE_STATS(
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    method_opt       => 'FOR ALL COLUMNS SIZE AUTO',
    cascade          => TRUE,
    options          => 'GATHER STALE'  -- 統計が古いオブジェクトのみ
  );
END;
/
```

### 索引統計の収集

```sql
-- 特定の索引の統計を収集
BEGIN
  DBMS_STATS.GATHER_INDEX_STATS(
    ownname  => 'HR',
    indname  => 'EMP_SALARY_IX',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE
  );
END;
/
```

---

## 現在の統計情報の表示

```sql
-- 表レベルの統計
SELECT table_name,
       num_rows,
       blocks,
       avg_row_len,
       last_analyzed,
       stale_stats
FROM   dba_tab_statistics
WHERE  owner = 'HR'
ORDER  BY table_name;

-- 列レベルの統計
SELECT column_name,
       num_distinct,
       num_nulls,
       density,
       low_value,
       high_value,
       histogram,
       num_buckets,
       last_analyzed
FROM   dba_tab_col_statistics
WHERE  owner      = 'HR'
  AND  table_name = 'ORDERS';

-- 索引統計
SELECT index_name,
       clustering_factor,
       num_rows,
       leaf_blocks,
       blevel,           -- 索引の高さ (深さ)
       last_analyzed
FROM   dba_ind_statistics
WHERE  owner      = 'HR'
  AND  table_name = 'ORDERS';
```

### クラスタ化係数 (Clustering Factor)

クラスタ化係数は、表の行の並び順が索引の並び順とどの程度一致しているかを測定する尺度である。低い値（ブロック数に近い値）が理想的である。高い値（行数に近い値）は、その索引を使用すると大量のランダム I/O が発生することを意味する。

```sql
-- クラスタ化係数が高い場合、索引に対する行の並び順が悪いことを示す
-- clustering_factor >> blocks の場合、表のリオーグ (CREATE TABLE AS SELECT ... ORDER BY indexed_col)
-- または IOT (索引構成表) の使用を検討する
SELECT i.index_name,
       i.clustering_factor,
       t.blocks,
       t.num_rows,
       ROUND(i.clustering_factor / NULLIF(t.blocks, 0), 1) AS cf_to_blocks_ratio
FROM   dba_indexes i
JOIN   dba_tables  t ON i.owner = t.owner AND i.table_name = t.table_name
WHERE  i.owner = 'HR'
  AND  i.table_name = 'ORDERS'
ORDER  BY i.clustering_factor DESC;
```

---

## 古い統計情報 (Stale Stats) の検出

Oracle は、前回の収集以降に行の 10% 以上が変更された場合、統計情報を「古い (STALE)」とマークする。

```sql
-- 統計情報が古い、または存在しない表をリストアップ
SELECT owner,
       table_name,
       num_rows,
       last_analyzed,
       stale_stats,
       stattype_locked
FROM   dba_tab_statistics
WHERE  owner        = 'HR'
  AND  (stale_stats = 'YES' OR last_analyzed IS NULL)
ORDER  BY last_analyzed NULLS FIRST;

-- 鮮度評価のための DML 変更回数の確認
SELECT table_owner,
       table_name,
       inserts,
       updates,
       deletes,
       timestamp,
       ROUND((inserts + updates + deletes) / NULLIF(num_rows, 0) * 100, 2) AS pct_modified
FROM   dba_tab_modifications m
JOIN   dba_tables t ON m.table_owner = t.owner AND m.table_name = t.table_name
WHERE  table_owner = 'HR'
ORDER  BY pct_modified DESC;

-- 注意: DBA_TAB_MODIFICATIONS はメモリから遅延フラッシュされる。強制フラッシュするには：
EXEC DBMS_STATS.FLUSH_DATABASE_MONITORING_INFO;
```

---

## ヒストグラム (Histograms)

ヒストグラムは、値が偏っている列のデータ分布をキャプチャする。ヒストグラムがない場合、オプティマイザは均一な分布を想定するため、偏りのある列に対して誤った見積もりを行ってしまう。

### ヒストグラムを使用すべき場合

- 列に **データの偏り (Data Skew)** がある（特定の値が他の値よりはるかに頻繁に出現する）。
- 列が `WHERE` 述語に使用されている。
- 列のカーディナリティが低～中程度で、分布が不均一である。

**例:** ステータス・コード (90% 'COMPLETED', 5% 'PENDING', 5% 'FAILED')、州コード、製品カテゴリなど。

### ヒストグラムの種類

| 種類 | 使用場面 | 説明 |
|---|---|---|
| `NONE` | ヒストグラムなし | 均一な分布を想定 |
| `FREQUENCY` | NDV <= 254 | 個別の値ごとに 1 つのバケット。正確な頻度 |
| `TOP-FREQUENCY` | NDV > 254 だが上位 254 値が閾値以上を占める | 最も頻繁に出現する値を格納 |
| `HEIGHT-BALANCED` | レガシー (12c 以前) | 等高バケットの境界値を格納 |
| `HYBRID` | NDV > 254 (12c+) | 頻度と等高の両方の性質を組み合わせる |

```sql
-- 特定の列にヒストグラムを指定して統計を収集
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname   => 'HR',
    tabname   => 'ORDERS',
    method_opt => 'FOR COLUMNS SIZE 254 STATUS, REGION SIZE AUTO'
    -- SIZE 254: 最大 254 バケットの頻度ヒストグラムを作成
    -- SIZE AUTO: ヒストグラムが必要かどうか Oracle が決定
    -- SIZE SKEWONLY: データに偏りがある場合のみヒストグラムを作成
  );
END;
/

-- ヒストグラム・データの表示
SELECT endpoint_number,
       endpoint_value,
       endpoint_actual_value
FROM   dba_histograms
WHERE  owner       = 'HR'
  AND  table_name  = 'ORDERS'
  AND  column_name = 'STATUS'
ORDER  BY endpoint_number;

-- どの列にヒストグラムがあるか確認
SELECT column_name,
       histogram,
       num_buckets
FROM   dba_tab_col_statistics
WHERE  owner      = 'HR'
  AND  table_name = 'ORDERS'
  AND  histogram  != 'NONE';
```

### ヒストグラムを使用すべきでない場合

- 列の値が真に均一である（偏りがない）場合 — ヒストグラムはメリットなしにオーバーヘッドだけを追加する。
- 列が `WHERE` 述語で一度も使用されない。
- 列が常にバインド変数でアクセスされ、バインド・ビープ (Bind Variable Peeking) に依存している場合（ヒストグラムはバインド・ピークと不整合を起こすことがある。`CURSOR_SHARING` の挙動に注意すること）。

---

## 拡張統計 (Extended Statistics)

拡張統計を使用すると、オプティマイザは述語における **列間の相関 (Correlated Columns)** や **式 (Expressions)** を考慮できるようになる。これらがないと、オプティマイザは個々の列の選択度を単純に乗算するため、相関がある場合にカウントを過小または過大に見積もってしまう。

### 列グループ統計 (相関のある列)

```sql
-- 問題: 相関のある 2 つの列 (MAKE, MODEL) が一緒にフィルタリングされる
-- SELECT * FROM cars WHERE make = 'TOYOTA' AND model = 'CAMRY'
-- 拡張統計なし: 選択度 = P(make=TOYOTA) * P(model=CAMRY) [誤り]
-- 拡張統計あり: オプティマイザは結合分布を使用する [正しい]

-- 列グループ拡張統計の作成
DECLARE
  l_cg_name VARCHAR2(30);
BEGIN
  l_cg_name := DBMS_STATS.CREATE_EXTENDED_STATS(
    ownname    => 'SALES',
    tabname    => 'CARS',
    extension  => '(MAKE, MODEL)'
  );
  DBMS_OUTPUT.PUT_LINE('作成完了: ' || l_cg_name);
END;
/

-- 統計の収集 (拡張統計作成後は再収集が必要)
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname   => 'SALES',
    tabname   => 'CARS',
    method_opt => 'FOR ALL COLUMNS SIZE AUTO FOR COLUMNS (MAKE, MODEL) SIZE AUTO'
  );
END;
/

-- 拡張統計の表示
SELECT extension_name, extension
FROM   dba_stat_extensions
WHERE  owner      = 'SALES'
  AND  table_name = 'CARS';
```

### 式の統計 (仮想列の統計)

```sql
-- オプティマイザは式に対する統計を持つこともできる
-- 関数ベースの索引が存在する場合に便利

-- 式の拡張統計を作成
DECLARE
  l_cg_name VARCHAR2(30);
BEGIN
  l_cg_name := DBMS_STATS.CREATE_EXTENDED_STATS(
    ownname   => 'HR',
    tabname   => 'EMPLOYEES',
    extension => '(UPPER(LAST_NAME))'
  );
END;
/
```

---

## 統計のロック (Locking Statistics)

統計をロックすることで、特定のオブジェクトに対して、自動または手動の再収集によって既知の良好な統計が上書きされるのを防ぐ。これは、ステージング・データや、特定の計画を維持しなければならない場合に有効である。

```sql
-- 表の統計をロック (再収集を防止)
BEGIN
  DBMS_STATS.LOCK_TABLE_STATS(
    ownname  => 'HR',
    tabname  => 'REFERENCE_DATA'
  );
END;
/

-- スキーマ内のすべての統計をロック
BEGIN
  DBMS_STATS.LOCK_SCHEMA_STATS(ownname => 'HR');
END;
/

-- アンロック (通常の収集に戻す)
BEGIN
  DBMS_STATS.UNLOCK_TABLE_STATS(
    ownname  => 'HR',
    tabname  => 'REFERENCE_DATA'
  );
END;
/

-- ロックされている統計の表示
SELECT table_name, stattype_locked
FROM   dba_tab_statistics
WHERE  owner         = 'HR'
  AND  stattype_locked IS NOT NULL;
```

---

## 統計のインポートとエクスポート

統計のエクスポートにより、以下のことが可能になる：
- 本番の統計をテスト/開発環境にコピーし、現実的なテストを行う。
- 再収集の前に既知の良好な統計を保存する（ロールバック機能）。
- 異なるデータベース・バージョン間で統計を移行する。

```sql
-- 統計保存用のステージング表を作成
BEGIN
  DBMS_STATS.CREATE_STAT_TABLE(
    ownname   => 'HR',
    stattab   => 'SAVED_STATS'
  );
END;
/

-- 表の統計をステージング表にエクスポート
BEGIN
  DBMS_STATS.EXPORT_TABLE_STATS(
    ownname  => 'HR',
    tabname  => 'ORDERS',
    stattab  => 'SAVED_STATS',
    statid   => 'PRE_UPGRADE_STATS'  -- このエクスポートのラベル
  );
END;
/

-- スキーマ全体の統計をエクスポート
BEGIN
  DBMS_STATS.EXPORT_SCHEMA_STATS(
    ownname  => 'HR',
    stattab  => 'SAVED_STATS',
    statid   => 'SCHEMA_BACKUP_20260306'
  );
END;
/

-- 統計のインポート (以前の統計にロールバック)
BEGIN
  DBMS_STATS.IMPORT_TABLE_STATS(
    ownname  => 'HR',
    tabname  => 'ORDERS',
    stattab  => 'SAVED_STATS',
    statid   => 'PRE_UPGRADE_STATS',
    no_invalidate => FALSE
  );
END;
/

-- 保留中の統計の表示 (12c+: 統計を保留状態にしてテストし、その後公開できる)
SELECT table_name, last_analyzed
FROM   dba_tab_pending_stats
WHERE  owner = 'HR';

-- 保留中の統計を公開
BEGIN
  DBMS_STATS.PUBLISH_PENDING_STATS(ownname => 'HR', tabname => 'ORDERS');
END;
/
```

---

## 自動統計収集 (Automatic Statistics Collection)

Oracle の自動メンテナンス・ジョブ (Autotask) は、メンテナンス・ウィンドウ中に、統計が古い、あるいは欠落しているオブジェクトに対して統計を収集する。

```sql
-- 自動メンテナンス・ジョブのステータス表示
SELECT client_name, status, consumer_group, mean_job_duration
FROM   dba_autotask_client
WHERE  client_name = 'auto optimizer stats collection';

-- メンテナンス・ウィンドウの表示
SELECT window_name, enabled, duration, repeat_interval
FROM   dba_scheduler_windows
WHERE  enabled = 'TRUE'
ORDER  BY window_name;

-- 自動統計収集を無効化 (慎重に行うこと)
BEGIN
  DBMS_AUTO_TASK_ADMIN.DISABLE(
    client_name  => 'auto optimizer stats collection',
    operation    => NULL,
    window_name  => NULL
  );
END;
/
```

---

## 保留中の統計 (Pending Statistics - 非破壊テスト)

Oracle 11g 以降では、新しい統計情報を「保留 (pending)」状態に設定し、他のセッションに影響を与えずにテストした後、公開または破棄することができる。

```sql
-- プリファレンス設定: 新しい統計を保留にし、即座に公開しないようにする
BEGIN
  DBMS_STATS.SET_TABLE_PREFS(
    ownname  => 'HR',
    tabname  => 'ORDERS',
    pname    => 'PUBLISH',
    pval     => 'FALSE'
  );
END;
/

-- 統計の収集 (保留状態になり、ライブには反映されない)
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(ownname => 'HR', tabname => 'ORDERS');
END;
/

-- 自分のセッションで保留中の統計をテストする
ALTER SESSION SET OPTIMIZER_USE_PENDING_STATISTICS = TRUE;

-- 実行計画の確認やクエリのテストを行う...

-- 満足できれば公開する
BEGIN
  DBMS_STATS.PUBLISH_PENDING_STATS(ownname => 'HR', tabname => 'ORDERS');
END;
/

-- 満足できなければ保留中の統計を破棄する
BEGIN
  DBMS_STATS.DELETE_PENDING_STATS(ownname => 'HR', tabname => 'ORDERS');
END;
/
```

---

## ベスト・プラクティス

- **`AUTO_SAMPLE_SIZE` を使用する** — Oracle 11g 以降の増分統計アルゴリズムは、最小限のオーバーヘッドで正確な統計を作成する。固定サンプル・サイズ (例: 10%) が優れていることは稀である。
- **パーティション表には増分統計 (Incremental Statistics) を使用する** — 変更されていないパーティションの全スキャンを回避できる。

```sql
BEGIN
  DBMS_STATS.SET_TABLE_PREFS(
    ownname  => 'HR',
    tabname  => 'SALES_PART',
    pname    => 'INCREMENTAL',
    pval     => 'TRUE'
  );
END;
/
```

- **クエリ実行前に、一括ロード後の統計を収集する** — 数百万行を挿入するバッチ処理は、即座に統計を古くさせる。
- **大幅な変更 (アップグレード、スキーマ変更) の前に統計をエクスポートする** — 実行計画が劣化した際に、以前の状態に復元できるようにするため。
- **`METHOD_OPT => 'FOR ALL COLUMNS SIZE AUTO'` をデフォルトとして使用する** — Oracle がワークロード・フィードバックに基づき、どの列にヒストグラムが必要かを決定する。
- **大規模なバッチ・ジョブの前に `STALE_STATS` を確認する** — 古い統計に対して 2 時間のバッチを走らせることはよくあるが、回避可能な事態である。
- **ほとんど変更されないが多くの結合で使用される、参照/ルックアップ表の統計をロックする。**

---

## よくある間違い

| 間違い | 影響 | 対策 |
|---|---|---|
| 新しい表の統計を一度も収集しない | オプティマイザが動的サンプリングを使用し、不正確になる可能性がある | 初回ロード後に統計を収集する |
| 固定サンプル・サイズ (例: 1%) を使用する | 偏りのあるデータに対して不正確になる | `AUTO_SAMPLE_SIZE` を使用する |
| 常に `NO_INVALIDATE=TRUE` で統計を収集する | 新しい統計が収集されても古い計画が残り続ける | `FALSE` に設定するか、デフォルト (遅延無効化) を使用する |
| 拡張統計作成後の再収集を忘れる | 拡張統計は存在するがデータがない状態になる | `CREATE_EXTENDED_STATS` 後は必ず収集する |
| 代替手段なしに自動統計収集を無効化する | 統計が古くなり、時間とともに計画が劣化する | カスタム・ジョブで置き換えるか、無効化だけして放置しない |
| 小さなルックアップ表の統計をロックしない | 統計が予期せず変わり、オプティマイザが結合順序を変えてしまう | 安定した小さな表の統計をロックする |
| バインド変数を使用する列にヒストグラムを作成する | バインド・ピーク + ヒストグラムが計画の不安定さを招く | `SIZE 1` (ヒストグラムなし) にするか、`CURSOR_SHARING=FORCE` を慎重に使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Tuning Guide (TGSQL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [DBMS_STATS — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_STATS.html)
- [DBMS_AUTO_TASK_ADMIN — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_AUTO_TASK_ADMIN.html)
- [DBA_TAB_STATISTICS — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_TAB_STATISTICS.html)
- [DBA_HISTOGRAMS — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HISTOGRAMS.html)
