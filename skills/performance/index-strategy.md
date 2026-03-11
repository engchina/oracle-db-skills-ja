# インデックス戦略 (Index Strategy) — Oracle のインデックスの種類と活用法

## 概要

インデックスは、表全体をスキャンすることなく、効率的に行を特定するための Oracle の主要なメカニズムである。適切なインデックスの種類、構造、および列の順序を選択することは、クエリのパフォーマンスにとって極めて重要である。不適切なインデックス戦略は、過度な全表スキャン（インデックス不足）を引き起こすか、あるいは DML パフォーマンスの低下やストレージの浪費（インデックスの過剰または不適切）を招く。

このガイドでは、B-tree インデックス、ビットマップ・インデックス、関数ベースのインデックス、複合インデックスの設計、不可視インデックス、監視、およびメンテナンスについて解説する。

---

## B-Tree インデックス

**B-tree (Balanced Tree)** は、デフォルトかつ最も汎用性の高いインデックス・タイプである。インデックス値をバランスの取れたツリー構造内にソートされた状態で保持し、O(log n) のルックアップを可能にする。

### 使用すべきケース

- 高カーディナリティの列（個別値が多い）：主キー、一意識別子、タイムスタンプ。
- `WHERE` 句で等価検索や範囲検索によく使用される列。
- 外部キー列（親の `DELETE` 時の表全体のロックを防止する）。
- `ORDER BY` や `GROUP BY` で事前にソートされたアクセスが有益な列。

### 使用を避けるべきケース

- 極めてカーディナリティが低い列（例：Y/N フラグ、性別） — 代わりにビットマップ・インデックスを検討。
- ほとんどの場合、全表スキャンの一部としてアクセスされる列。
- DML の負荷が非常に高く、インデックスによるオーバーヘッドがクエリのメリットを上回る列。

```sql
-- シンプルな B-tree インデックス
CREATE INDEX emp_salary_ix ON employees (salary);

-- 一意 B-tree インデックス (一一意性を強制し、UNIQUE SCAN を可能にする)
CREATE UNIQUE INDEX emp_email_uk ON employees (email);

-- インデックスが作成されたか確認
SELECT index_name, index_type, uniqueness, status
FROM   user_indexes
WHERE  table_name = 'EMPLOYEES';

-- インデックスが設定されている列を確認
SELECT index_name, column_position, column_name, descend
FROM   user_ind_columns
WHERE  table_name = 'EMPLOYEES'
ORDER  BY index_name, column_position;
```

---

## ビットマップ・インデックス (Bitmap Indexes)

ビットマップ・インデックスは、個別値ごとに 1 行あたり 1 ビットを格納する。低カーディナリティの列に対して極めてコンパクトであり、複数の述語を持つアドホックな分析クエリに対して非常に効率的である。

### 使用すべきケース

- **低カーディナリティ**の列（個別値が 2 ～ 100 程度）：ステータス・コード、地域、ブール値フラグ。
- **データ・ウェアハウス**や、読み取り（`SELECT`）が中心で挿入/更新/削除が少ないレポート環境。
- 複数の低カーディナリティ・フィルタ（`AND`/`OR`）を組み合わせるクエリ — Oracle はビットマップ演算を使用してこれらを結合できる。

### 使用を避けるべきケース

- **OLTP 環境** — ビットマップ・インデックスは DML 中にビットマップ全体をロックするため、激しい競合を引き起こす。
- 高カーディナリティの列 — ストレージの無駄であり、B-tree の方が適している。
- 頻繁に更新される列 — 1 つの DML が同じ値を持つすべての行のビットマップをロックする。

```sql
-- 低カーディナリティのステータス列に対するビットマップ・インデックス
CREATE BITMAP INDEX orders_status_bix ON orders (status);

-- 複数列のフィルタ・クエリで威力を発揮する
-- Oracle は表アクセス前にビットマップの論理積/和（AND/OR）を計算する
SELECT COUNT(*) FROM sales
WHERE  region  = 'WEST'
  AND  quarter = 'Q1'
  AND  channel = 'ONLINE';
-- region, quarter, channel にビットマップ・インデックスがある場合:
-- 3 つのビットマップ・スキャン → ビットマップ AND 演算 → COUNT (表アクセスが不要になることもある)

-- ビットマップ・インデックスの確認
SELECT index_name, index_type
FROM   user_indexes
WHERE  table_name = 'SALES'
  AND  index_type = 'BITMAP';
```

### ビットマップ vs B-Tree の比較

| 特徴 | B-Tree | ビットマップ |
|---|---|---|
| 最適なカーディナリティ | 高 | 低 (個別値 100 未満) |
| DML パフォーマンス | 1 行あたりのオーバーヘッドは中程度 | 激しい競合。行レベル・ロックが拡大 |
| ストレージ | 値ごとのエントリー | 低カーディナリティでは極めてコンパクト |
| 結合された述語 | 個別のインデックス・ルックアップ | ビット演算。極めて効率的 |
| 最適なワークロード | OLTP + OLAP | データ・ウェアハウス / 読み取り主体の OLAP |
| NULL の格納 | 格納されない (IS NULL 検索はインデックスを使えない) | NULL も独自のビットマップを持つ |

---

## 関数ベースのインデックス (FBI)

**関数ベースのインデックス (Function-Based Index)** は、関数や式の計算結果を事前に求めてインデックス化する。これにより、インデックス付きの列に関数が適用された場合でも、インデックス・アクセスが可能になる（通常、関数の適用はインデックスの使用を抑制する）。

```sql
-- FBI がない場合: LAST_NAME 上のインデックスは使用されない
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';

-- 式に対して FBI を作成
CREATE INDEX emp_upper_lname_fix ON employees (UPPER(last_name));

-- これにより Oracle はインデックスを使用できるようになる
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';

-- 大文字小文字を区別しないメール検索用の FBI
CREATE INDEX emp_lower_email_fix ON employees (LOWER(email));
SELECT * FROM employees WHERE LOWER(email) = 'john.doe@example.com';

-- 日付の切り捨て用 FBI (日付部分のみでフィルタするレポート・クエリ用)
CREATE INDEX orders_order_date_trunc ON orders (TRUNC(order_date));
SELECT * FROM orders WHERE TRUNC(order_date) = DATE '2026-03-01';

-- 複数の列を組み合わせた式に対する FBI
CREATE INDEX emp_annual_sal_ix ON employees (salary * 12);
SELECT * FROM employees WHERE salary * 12 > 120000;
```

### 重要な注意点

- 関数は **決定論的 (deterministic)** である必要がある（同じ入力に対して常に同じ出力を返すこと）。
- FBI で使用されるユーザー定義関数は、`DETERMINISTIC` として宣言されている必要がある。
- Oracle が FBI を使用するには、`QUERY_REWRITE_ENABLED` が `TRUE`（デフォルト）に設定されている必要がある。

```sql
-- QUERY_REWRITE_ENABLED の確認
SELECT name, value FROM v$parameter WHERE name = 'query_rewrite_enabled';
```

---

## 複合 (複数列) インデックス

複合インデックスは、2 つ以上の列をカバーする。列の順序は極めて重要であり、クエリのアクセス・パターンと一致していなければならない。

### 列順序のルール

**ルール 1: インデックスがアクセス（範囲または等価スキャン）に使用されるためには、先頭列がクエリ述語に含まれている必要がある。** 先頭列をスキップするクエリは、「インデックス・スキップ・スキャン」しか使用できず、これは先頭列のカーディナリティが極めて低い場合にのみ効率的である。

**ルール 2: 等価検索に使用される列を先に配置し、その後に範囲検索に使用される列を配置する。**

```sql
-- (DEPT_ID, SALARY) 上のインデックス
CREATE INDEX emp_dept_sal_ix ON employees (department_id, salary);

-- インデックスを使用する (先頭列が述語に含まれる)
SELECT * FROM employees WHERE department_id = 50 AND salary > 5000;
-- アクセス: department_id=50 で INDEX RANGE SCAN を行い、salary>5000 でフィルタ。

-- インデックスを使用する (先頭列のみ)
SELECT * FROM employees WHERE department_id = 50;
-- アクセス: INDEX RANGE SCAN

-- インデックスを効率的に使用できない (先頭列が含まれない)
SELECT * FROM employees WHERE salary > 5000;
-- アクセス: INDEX SKIP SCAN または TABLE ACCESS FULL (カーディナリティに依存)

-- 範囲述語では列の順序が重要:
-- インデックス (DEPT_ID, HIRE_DATE) — 適しているケース: WHERE dept=X AND hire_date BETWEEN...
-- インデックス (HIRE_DATE, DEPT_ID) — 適しているケース: WHERE hire_date=X AND dept=Y
--   ただし、DEPT_ID が等価検索でない場合、HIRE_DATE の範囲スキャンを効率的に行えないことがある。
```

### 複合インデックスが個別のインデックスよりも優れているケース

- クエリが両方の列でフィルタする場合 → 2 つの別個のスキャンとビットマップ・マージを行う代わりに、単一のインデックス範囲スキャンで済む。
- インデックスが必要なすべての列をカバーしている場合 → **インデックス・オンリー・スキャン** (表へのアクセスが不要になる)。
- `ORDER BY` や `GROUP BY` がインデックスの順序を利用できる場合。

### カバリング・インデックス (インデックス・オンリー・スキャン)

```sql
-- カバリング・インデックスがない場合: インデックス・スキャン + 表の行取得 (1 行あたり 2 回の I/O)
-- カバリング・インデックスがある場合: インデックス・スキャンのみ (1 行あたり 1 回の I/O)

-- クエリ: SELECT last_name, salary FROM employees WHERE department_id = 60
-- 選択される列とフィルタされる列をすべて含むカバリング・インデックスを作成:
CREATE INDEX emp_dept_cover_ix ON employees (department_id, last_name, salary);

-- EXPLAIN PLAN で確認 — "INDEX FAST FULL SCAN" または表アクセス・ステップがないことを確認
EXPLAIN PLAN FOR
SELECT last_name, salary FROM employees WHERE department_id = 60;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

---

## 不可視インデックス (Invisible Indexes)

不可視インデックスは、DML によってメンテナンスされるが、デフォルトでは **オプティマイザから無視される**。これにより、本番クエリに影響を与えることなく新しいインデックスをテストしたり、削除前に安全に無効化したりすることができる。

```sql
-- 不可視インデックスの作成
CREATE INDEX emp_job_id_ix ON employees (job_id) INVISIBLE;

-- 既存のインデックスを不可視にする
ALTER INDEX emp_job_id_ix INVISIBLE;

-- 効果をテストする: 自セッションでのみ不可視インデックスの使用を許可
ALTER SESSION SET OPTIMIZER_USE_INVISIBLE_INDEXES = TRUE;

-- クエリをテスト
EXPLAIN PLAN FOR SELECT * FROM employees WHERE job_id = 'IT_PROG';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());

-- 有効であれば、すべてのユーザーに対して可視にする
ALTER INDEX emp_job_id_ix VISIBLE;

-- 不可視インデックスの表示
SELECT index_name, visibility, status
FROM   user_indexes
WHERE  table_name = 'EMPLOYEES';
```

### 不可視インデックスの活用シーン

1. 他のセッションに影響を与えずに **新しいインデックスをテストする**。
2. **安全な廃止:** しばらく不可視にして問題が発生しないことを確認してから削除する。
3. **インデックス削除のテスト:** 既存のインデックスを不可視にし、ワークロードを実行して影響を評価する。

---

## インデックスの監視

Oracle は、インデックスが使用されたかどうか（クエリ実行中にオプティマイザによってアクセスされたか）を追跡できる。これにより、不要なインデックスを特定して削除し、DML のオーバーヘッドを軽減できる。

### 12c 以降 (DBA_INDEX_USAGE による自動監視)

Oracle Database 12c Release 2 以降、インデックスの使用状況は明示的に監視を有効にしなくても自動的に追跡される。統計情報は `V$INDEX_USAGE_INFO` から `DBA_INDEX_USAGE` へ、およそ 15 分ごとにフラッシュされる。

```sql
-- インデックス使用統計の確認 (12cR2+)
-- V$INDEX_USAGE_INFO ではなく DBA_INDEX_USAGE をクエリする
SELECT name            AS index_name,
       total_access_count,
       total_exec_count,
       last_used
FROM   dba_index_usage
WHERE  owner = 'HR'
  AND  name IN (
    SELECT index_name FROM user_indexes WHERE table_name = 'EMPLOYEES'
  );
```

### 12c 以前の監視 (ALTER INDEX MONITORING USAGE)

```sql
-- 監視を有効にする
ALTER INDEX emp_salary_ix MONITORING USAGE;

-- ワークロードを実行...

-- 使用されたか確認
SELECT index_name, monitoring, used, start_monitoring, end_monitoring
FROM   v$object_usage
WHERE  index_name = 'EMP_SALARY_IX';

-- 監視を無効にする
ALTER INDEX emp_salary_ix NOMONITORING USAGE;
```

**注意:** 12c 以前の監視機能は、「使用されたかどうか（TRUE/FALSE）」を記録するだけであり、使用頻度は記録されない。1 か月に 1 回使われたインデックスも、100 万回使われたインデックスも、表示上は同じになる。

---

## 再構築 (Rebuild) vs. 結合 (Coalesce)

時間の経過とともに、B-tree インデックスにはデフラグメンテーションや未使用領域が発生することがある。これに対処するための 2 つのメンテナンス・オプションがある。

### 結合 (COALESCE)

既存のブランチ内のリーフ・ブロックをマージ（結合）する。インデックスの高さ（Height）は **減少しない**。I/O オーバーヘッドは最小限であり、軽度の断片化に適している。

```sql
ALTER INDEX emp_salary_ix COALESCE;
```

### 再構築 (REBUILD)

表データを使用してインデックスをゼロから作り直す。高さの問題の解消、クラスタリング・ファクタの改善、ストレージ・パラメータの変更が可能である。オーバーヘッドは大きい（表データ全体を読み取るため）。

```sql
-- オンラインで再構築 (DML をブロックしない)
ALTER INDEX emp_salary_ix REBUILD ONLINE;

-- 別の表領域に再構築
ALTER INDEX emp_salary_ix REBUILD TABLESPACE idx_tbs ONLINE;

-- キーを圧縮して再構築 (先頭値が繰り返される複合インデックスに有効)
ALTER INDEX emp_dept_sal_ix REBUILD COMPRESS 1;
```

### 再構築 vs 結合の使い分け

| シナリオ | 推奨アクション |
|---|---|
| 大量の削除によりリーフ・ブロックに無駄がある | 結合 (高速で安全) |
| 中小規模の表でインデックスの高さが 4 以上になった | 再構築 |
| クラスタリング・ファクタが著しく低下した | 再構築 (または表自体の MOVE による再配置) |
| インデックスを別の表領域に移動する | 再構築 |
| 健全なインデックスに対する定期的な「メンテナンス」 | 不要 (副作用の方が大きいこともある) |

**重要:** Oracle 10g 以降、インデックスの定期的な再構築（月次など）は、ほとんどの場合不要である。統計情報が適切で正常に動作しているインデックスを再構築する必要はほとんどない。決定する前に、必ず `ANALYZE INDEX ... VALIDATE STRUCTURE` で状況を確認すること。

```sql
-- インデックスの構造を分析し、断片化の状態を確認
ANALYZE INDEX emp_salary_ix VALIDATE STRUCTURE;

-- 分析結果の確認
SELECT name,
       height,
       blocks,
       lf_rows,     -- リーフ行 (実際のエントリー)
       lf_blks,     -- リーフ・ブロック
       del_lf_rows, -- 削除されたリーフ行
       ROUND(del_lf_rows / NULLIF(lf_rows, 0) * 100, 2) AS pct_deleted
FROM   index_stats;
-- pct_deleted が 20-30% を超える場合、再構築が有効な場合がある。
```

---

## 外部キー上のインデックス

頻繁に見落とされるのが、子テーブルの **外部キー列** 上のインデックスである。これがない場合：

- 親から子へ移動する際に全表スキャンが発生する。
- 親行の削除や更新時に、子テーブルに対して **表レベルの排他ロック** が発生する（カスケード/整合性チェックが完了するまで）。

```sql
-- インデックスのない外部キーを特定する
SELECT ac.table_name,
       ac.constraint_name,
       acc.column_name
FROM   all_constraints ac
JOIN   all_cons_columns acc
  ON   ac.constraint_name = acc.constraint_name
  AND  ac.owner           = acc.owner
WHERE  ac.constraint_type = 'R'  -- 参照整合性 (FK)
  AND  ac.owner           = 'HR'
  AND  NOT EXISTS (
    SELECT 1
    FROM   all_ind_columns aic
    WHERE  aic.table_name  = ac.table_name
      AND  aic.owner       = ac.owner
      AND  aic.column_name = acc.column_name
      AND  aic.column_position = 1
  );
```

---

## ベスト・プラクティス

- **選択的にインデックスを作成する:** 使用されるという証拠（実行計画、ASH、AWR）がある場合にのみ、インデックスを追加すること。
- **定期的に不要なインデックスを監視する:** `DBA_INDEX_USAGE` (12cR2+) を使用し、本当に使われていない場合は削除する。不要なインデックスは領域を浪費し、DML を遅くする。
- **外部キーには必ずインデックスを作成する:** ロックの競合や不要な全表スキャンを回避するためである。
- **高頻度の OLTP クエリにはカバリング・インデックスを優先する:** 表の行取得ステップを排除するためである。
- **不可視インデックスを活用する:** 全ユーザーに公開する前に安全にテストを行う。
- **過度なキー圧縮を避ける:** B-tree インデックスのキー圧縮は、先頭値が繰り返される複合インデックスには有効だが、CPU オーバーヘッドが発生する。
- **本番環境ではオンラインで再構築する:** DML のブロッキングを避けるために `REBUILD ONLINE` を使用すること。
- **大規模なバッチ削除の後**、影響を受けた表のインデックスを結合または再構築することを検討する。

---

## よくある間違い

| 間違い | 影響 | 対策 |
|---|---|---|
| Y/N フラグ列に B-tree インデックスを作成する | ほとんど使用されず、DML のオーバーヘッドのみが発生 | DW ならビットマップ、OLTP ならインデックスなし。 |
| 複合インデックスの列順序が不適切 | 代表的なクエリでインデックスが使用されない | 等価検索の列を先に、範囲検索の列を後にする。 |
| 外部キー列へのインデックス作成漏れ | 親の DML 時にロック競合が発生。結合が遅くなる | 外部キーには必ずインデックスを作成する。 |
| FTS の結果を見て「インデックスは不要」と結論づける | FBI の不足や型不一致が原因かもしれない | 述語情報をチェックし、関数適用や型変換を修正する。 |
| スケジュールに基づいてすべてのインデックスを再構築する | メンテナンス・ウィンドウの浪費。実質的なメリットなし | 断片化が確認された場合にのみ再構築する。 |
| テストせずにインデックスを削除する | パフォーマンス低下を引き起こす可能性がある | まず不可視にし、テストしてから削除する。 |
| 重複したインデックスを作成する | DML オーバーヘッド。ストレージの浪費 | 新規作成前に既存のインデックスを確認する。 |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Tuning Guide (TGSQL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [Oracle Database 19c Performance Tuning Guide (TGDBA)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)
- [USER_INDEXES / DBA_INDEXES — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/USER_INDEXES.html)
- [DBA_INDEX_USAGE — Oracle Database 19c Reference (12cR2+)](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_INDEX_USAGE.html)
- [V$OBJECT_USAGE — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-OBJECT_USAGE.html)
