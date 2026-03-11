# Oracle マテリアライズド・ビュー

## 概要

**マテリアライズド・ビュー (MV)** は、クエリの結果をディスク上に物理的に保存し、必要に応じて基となる表（マスター表）の変更を反映（リフレッシュ）するデータベース・オブジェクトである。通常のビューは参照されるたびにクエリを再実行するが、マテリアライズド・ビューは定義済みの結果セットであるため、再計算のコストなしに高速に読み取ることができる。

Oracle におけるマテリアライズド・ビューの主な目的は以下の 2 つである：

1. **クエリ・リライト (Query Rewrite)** — オブジェクト・レベルでの透過的な最適化。ユーザーのクエリを、高コストなマスター表ではなく、事前集計済みの MV を読み取るようにオプティマイザが自動的に書き換える。アプリケーション・コードを変更する必要はない。
2. **データ・レプリケーション (Data Replication)** — データの要約やフィルタリングされたコピーを別のスキーマやデータベースに配布し、独立して利用可能にする。

マテリアライズド・ビューは Oracle 8i で導入され、従来の「スナップショット」機能を置き換えた。23ai に至るまで継続的に強化されている。

---

## コア・コンセプト

### リフレッシュ・モード

| モード | 説明 | ユースケース |
|---|---|---|
| `COMPLETE` | MV データを一度全削除し、クエリを再実行して全件を再投入する | あらゆるクエリで利用可能。大規模データでは低速 |
| `FAST` | 前回のリフレッシュ以降の変更のみを MV ログから反映する | 大規模な表に対して増分変更が少ない場合。高速 |
| `FORCE` | 可能であれば FAST を行い、不可であれば COMPLETE を行う | デフォルト推奨。FAST が可能か不明な場合に有効 |
| `NEVER` | 自動リフレッシュを行わない。手動でのみ更新可能 | 静的なスナップショット、データ移行用 |

### リフレッシュ・タイミング

| タイミング | 説明 |
|---|---|
| `ON COMMIT` | マスター表に対する DML トランザクションがコミットされたとき、自動的にリフレッシュする |
| `ON DEMAND` | `DBMS_MVIEW.REFRESH` を明示的に呼び出したときのみリフレッシュする |
| `ON STATEMENT` | 12.1 以降：コミット前であっても、各 DML 文の直後にリフレッシュする |

### マテリアライズド・ビュー・ログ (MV ログ)

**マテリアライズド・ビュー・ログ (MV ログ)** は、マスター表に作成される変更追跡用の表である。マスター表への INSERT, UPDATE, DELETE がすべて記録される。**高速リフレッシュ (FAST refresh)** は、マスター表を再スキャンする代わりに、このログのみを読み取ることで実行される。

---

## マテリアライズド・ビュー・ログの作成

高速リフレッシュを可能にするには、事前に関連するすべてのマスター表に MV ログを作成しておく必要がある。

```sql
-- 基本的な MV ログの作成
CREATE MATERIALIZED VIEW LOG ON sales
WITH ROWID, SEQUENCE (sale_date, product_id, region_id, amount)
INCLUDING NEW VALUES;
```

---

## マテリアライズド・ビューの作成

### COMPLETE リフレッシュ — 単純集計 MV

```sql
CREATE MATERIALIZED VIEW mv_monthly_sales_summary
BUILD IMMEDIATE           -- 作成時にすぐデータを投入
REFRESH COMPLETE
ON DEMAND
AS
SELECT  TRUNC(sale_date, 'MM') AS sale_month,
        region_id,
        COUNT(*)               AS num_sales,
        SUM(amount)            AS total_revenue
FROM    sales
GROUP   BY TRUNC(sale_date, 'MM'), region_id;
```

### FAST リフレッシュ — 集計 MV

集計 MV で高速リフレッシュを行うには、クエリ内に `COUNT(*)` と、集計対象の `COUNT(col)` が含まれている必要がある。

```sql
CREATE MATERIALIZED VIEW mv_sales_by_product
REFRESH FAST ON DEMAND
ENABLE QUERY REWRITE      -- クエリ・リライトを有効化
AS
SELECT  product_id,
        COUNT(*)        AS cnt,         -- 必須
        SUM(amount)     AS sum_amount,
        COUNT(amount)   AS cnt_amount   -- SUM の FAST リフレッシュに必須
FROM    sales
GROUP   BY product_id;
```

---

## クエリ・リライト (Query Rewrite)

クエリ・リライトは、ユーザーがマスター表を参照するクエリを投げた際、オプティマイザが自動的に MV を使うように書き換える機能である。

```sql
-- セッション・レベルでの有効化
ALTER SESSION SET query_rewrite_enabled = TRUE;

-- 整合性モードの設定
ALTER SESSION SET query_rewrite_integrity = ENFORCED;
-- ENFORCED: MV が最新 (FRESH) であることが確実な場合のみリライト
-- TRUSTED:  リレーションシップ（外部キーなど）を信頼してリライト
-- STALE_TOLERATED: データが古くても (STALE) リライトを許可
```

---

## 手動リフレッシュ (DBMS_MVIEW)

```sql
-- 1つの MV をリフレッシュ
EXEC DBMS_MVIEW.REFRESH('MV_MONTHLY_SALES_SUMMARY', method => 'C');

-- 複数の MV を依存関係順にリフレッシュ
EXEC DBMS_MVIEW.REFRESH('MV_SALES_LOG, MV_SALES_SUMMARY', method => 'F');

-- 特定のマスター表に依存するすべての MV をリフレッシュ
EXEC DBMS_MVIEW.REFRESH_DEPENDENT('SALES');
```

---

## ベスト・プラクティス

- **MV ログを忘れずに作成する:** `REFRESH FAST` を指定する場合、マスター表側に適切なオプション付きの MV ログが存在しなければエラーになる。
- **`ON COMMIT` リフレッシュは軽量な処理に限定する:** ユーザーのコミット処理の一部として実行されるため、集計が重いとアプリケーションの応答性に直接悪影響を与える。重い集計は `ON DEMAND` でジョブ実行することを推奨する。
- **`atomic_refresh => FALSE` の活用:** 大規模な `COMPLETE` リフレッシュを行う際、 undo 領域の不足を避けるためにこれを FALSE に設定することを検討する（内部的に TRUNCATE が使用されるようになる）。
- **ステータスの監視:** `USER_MVIEWS` の `STALENESS` 列を確認し、MV のデータが古くなっていないか、リフレッシュが失敗していないかを定期的に監視する。

---

## よくある間違い

**間違い: 高速リフレッシュを指定したのに、実際には完全リフレッシュが走っている。**
`DBMS_MVIEW.REFRESH` で `method => '?'` (FORCE) を指定していると、FAST が失敗してもエラーが出ずに COMPLETE が実行される。`USER_MVIEWS` の `LAST_REFRESH_TYPE` を確認する習慣を付けること。

**間違い: クエリ・リライトが効かない。**
MV が `STALE`（マスター表の更新後にリフレッシュされていない）状態だと、デフォルトの整合性モードではリライト対象から外れる。リフレッシュの間隔と、許容されるデータの古さを適切に設計する必要がある。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [Oracle Database Data Warehousing Guide: Basic Materialized Views](https://docs.oracle.com/en/database/oracle/oracle-database/19/dwhsg/basic-materialized-views.html)
- [DBMS_MVIEW — Oracle Database PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_MVIEW.html)
