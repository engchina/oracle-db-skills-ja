# 実行計画 (Explain Plan) — 実行計画の分析

## 概要

**実行計画 (Execution Plan)** とは、Oracle が SQL 文を処理するために使用する一連の操作手順のことである。オプティマイザは複数の候補プランを評価し、推定コストが最も低いものを選択する。実行計画を生成し、読み、解釈する方法を理解することは、Oracle パフォーマンス・チューニングにおける最も基本的なスキルである。

実行計画はいくつかの方法で取得できる：
- **EXPLAIN PLAN** — クエリを実行せずにプランを推定する。
- **DBMS_XPLAN.DISPLAY_CURSOR** — 最近実行された文から実際のプランを取得する。
- **DBMS_XPLAN.DISPLAY_AWR** — AWR に保存されている履歴プランを取得する。
- **AUTOTRACE** — SQL*Plus において EXPLAIN PLAN と実際の実行統計を組み合わせて表示する。

---

## EXPLAIN PLAN

`EXPLAIN PLAN FOR` は SQL を解析し、推定プランを `PLAN_TABLE`（自動的に作成されるセッション・レベルの一時表）に書き込む。クエリ自体は **実行されない**。

```sql
-- 基本的な使い方
EXPLAIN PLAN FOR
SELECT e.last_name, d.department_name
FROM   employees e
JOIN   departments d ON e.department_id = d.department_id
WHERE  e.salary > 10000;

-- 結果の表示
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY());
```

### ステートメント ID の使用 (複数のプランを扱う場合)

```sql
EXPLAIN PLAN SET STATEMENT_ID = 'MY_QUERY' FOR
SELECT * FROM orders WHERE status = 'PENDING';

SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY(
    table_name   => 'PLAN_TABLE',
    statement_id => 'MY_QUERY',
    format       => 'TYPICAL'
  )
);
```

### EXPLAIN PLAN の制限事項

`EXPLAIN PLAN` は、実行時とは異なるバインド変数の覗き見（Peeking）ルールを使用する場合がある。そのため、実際に実行されるプランとは異なるプランが表示される可能性がある。特に以下の場合に注意が必要である：
- バインド変数が使用されている（推定プランではデフォルト値が代入される）。
- 適応型プラン（Adaptive Plans）が関与している。
- 統計情報が古い。

**実際に実行されたプランが必要な場合は、常に `DISPLAY_CURSOR` を優先して使用すること。**

---

## DBMS_XPLAN.DISPLAY_CURSOR

最近実行されたステートメントの実際のプランを共有プール（Shared Pool）から取得する。これは、Oracle が実際に何を選択したかを確認するための最も信頼できる方法である。

```sql
-- 自セッションで直前に実行された文のプランを表示
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR());

-- 特定の SQL_ID のプランを表示 (最後の子カーソル)
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    sql_id        => 'abc123xyz789',
    cursor_child_no => NULL,   -- NULL = 最新の子
    format        => 'TYPICAL'
  )
);

-- 最近のクエリの SQL_ID を探す
SELECT sql_id, plan_hash_value, sql_text
FROM   v$sql
WHERE  sql_text LIKE '%orders%'
  AND  sql_text NOT LIKE '%v$sql%'
ORDER  BY last_active_time DESC
FETCH  FIRST 10 ROWS ONLY;
```

### フォーマット・オプション

| フォーマット文字列 | 表示内容 |
|---|---|
| `'BASIC'` | 操作とオブジェクトのみを表示。 |
| `'TYPICAL'` | 標準出力 (デフォルト) — 操作、コスト、行数、バイト数。 |
| `'ALL'` | 述語情報、列プロジェクションを含む詳細情報。 |
| `'ADVANCED'` | ALL に加えてアウトライン、バインド、リモート SQL 情報。 |
| `'+IOSTATS LAST'` | 直近の実行における実際の行数を追加。 |
| `'+MEMSTATS'` | ソート/ハッシュ操作のメモリ使用量。 |
| `'+ROWSTATS LAST'` | 行ソース統計を追加 (`STATISTICS_LEVEL=ALL` またはヒントが必要)。 |

```sql
-- デバッグに最も役立つフォーマット: プラン + 実際の行数
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_CURSOR(
    sql_id  => 'abc123xyz789',
    format  => 'TYPICAL +IOSTATS LAST +PEEKED_BINDS'
  )
);
```

### 行ソース統計（Row Source Statistics）収集の有効化

操作あたりの実際の行数（最も価値のある診断データ）を取得するには、統計収集を有効にする必要がある：

```sql
-- オプション 1: セッション・レベル (セッションを制御できる場合)
ALTER SESSION SET STATISTICS_LEVEL = ALL;
-- クエリの実行...
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT => 'ALLSTATS LAST'));

-- オプション 2: クエリ・レベルのヒント (ALTER SESSION は不要)
SELECT /*+ GATHER_PLAN_STATISTICS */
       e.last_name, e.salary
FROM   employees e
WHERE  e.department_id = 60;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(FORMAT => 'ALLSTATS LAST'));
```

---

## DBMS_XPLAN.DISPLAY_AWR

AWR に保存されている履歴プランを取得する。プランが過去に使用されたが、現在は共有プールに存在しない場合に有用である。

> 注: Oracle Database 23c 以降、`DISPLAY_AWR` は非推奨。代わりに、同じ目的を持つ `DBMS_XPLAN.DISPLAY_WORKLOAD_REPOSITORY` が推奨される（引数の順序や名前が若干異なる。`dbid` パラメータが最後であり、`db_id` ではなく `dbid` という名前になっている）。`DISPLAY_AWR` は後方互換性のために引き続き機能するが、新しいコードでは `DISPLAY_WORKLOAD_REPOSITORY` を優先すること。

```sql
-- AWR から特定の SQL_ID のすべてのプランを取得 (23c+ では DISPLAY_WORKLOAD_REPOSITORY を推奨)
SELECT * FROM TABLE(
  DBMS_XPLAN.DISPLAY_AWR(
    sql_id          => 'abc123xyz789',
    plan_hash_value => NULL,  -- NULL はすべてのプランを表示
    db_id           => NULL,  -- NULL = 現行データベース
    format          => 'TYPICAL'
  )
);

-- AWR から SQL のすべてのプラン・ハッシュを表示
SELECT sql_id,
       plan_hash_value,
       MIN(begin_interval_time) AS first_seen,
       MAX(end_interval_time)   AS last_seen,
       SUM(executions_delta)    AS total_executions,
       ROUND(SUM(elapsed_time_delta) / NULLIF(SUM(executions_delta),0) / 1e6, 3) AS avg_elapsed_sec
FROM   dba_hist_sqlstat s
JOIN   dba_hist_snapshot sn USING (snap_id, dbid, instance_number)
WHERE  sql_id = 'abc123xyz789'
GROUP  BY sql_id, plan_hash_value
ORDER  BY first_seen;
```

---

## プラン出力の読み方

一般的なプランは次のようになる：

```
Plan hash value: 1234567890

----------------------------------------------------------------------------------
| Id  | Operation                    | Name         | Rows  | Bytes | Cost (%CPU)|
----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |              |   500 |  25K  |   142   (2)|
|   1 |  HASH JOIN                   |              |   500 |  25K  |   142   (2)|
|   2 |   TABLE ACCESS FULL          | DEPARTMENTS  |    27 |   432 |     3   (0)|
|*  3 |   TABLE ACCESS BY INDEX ROWID| EMPLOYEES    |   500 |  17K  |   139   (1)|
|*  4 |    INDEX RANGE SCAN          | EMP_DEPT_IX  |   503 |       |     2   (0)|
----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   3 - filter("E"."SALARY">10000)
   4 - access("E"."DEPARTMENT_ID"="D"."DEPARTMENT_ID")
```

### 各列の意味

| 列名 | 意味 |
|---|---|
| `Id` | ステップ番号。アスタリスク (*) は述語が適用されているステップを示す。 |
| `Operation` | Oracle が使用するアルゴリズム (TABLE ACCESS FULL, INDEX RANGE SCAN, HASH JOIN など)。 |
| `Name` | 関連するオブジェクト名 (表または索引)。 |
| `Rows` | オプティマイザの推定行数 (カーディナリティ)。 |
| `Bytes` | 推定データ量 (行数 × 平均行サイズ)。 |
| `Cost` | オプティマイザのコスト (相対値。I/O + CPU モデルに基づく)。 |
| `(%CPU)` | コストのうち CPU が占める割合。 |
| `Time` | 推定経過時間 (あくまで概算)。 |

### 実際の行数を含めた読み方 (ALLSTATS フォーマット)

```
| Id  | Operation            | Name    | Starts | E-Rows | A-Rows |   A-Time   | Buffers |
|   0 | SELECT STATEMENT     |         |      1 |        |     50 |00:00:02.14 |   84321 |
|*  1 |  TABLE ACCESS FULL   | ORDERS  |      1 |  500K  |     50 |00:00:02.14 |   84321 |
```

- `Starts` — この操作が開始された回数。
- `E-Rows` — 推定行数（オプティマイザの予測）。
- `A-Rows` — 実際に返された行数。
- `A-Time` — このステップまでにかかった累積経過時間（それ以下のステップを含む）。
- `Buffers` — このステップでの論理 I/O（バッファ取得数）。

**最も重要な診断ポイント:** `E-Rows` と `A-Rows` を比較すること。10 倍以上の大きな乖離がある場合、カーディナリティの推定が誤っており、それが不適切なプラン選択の原因となっている。

### プラン・ツリーの順序

プランは **「内側から外側へ」**（最も字下げされているものから）、そして一連のブランチの中では **「下から上へ」** 読まれる：

```
|   0 | SELECT STATEMENT    |          -- 最後: 結果をマージ
|   1 |  SORT ORDER BY      |          -- ステップ 4: ソート
|   2 |   HASH JOIN         |          -- ステップ 3: 結合
|   3 |    INDEX RANGE SCAN | IDX_A    -- ステップ 1: 索引をスキャン
|   4 |    TABLE ACCESS FULL| TABLE_B  -- ステップ 2: 表をスキャン
```

---

## 一般的なプランの操作とその意味

### アクセス・パス (Access Paths)

| 操作 | 説明 | 適しているケース |
|---|---|---|
| `TABLE ACCESS FULL` | 表の全ブロックを読み込む。 | 返される行が全体に対して少ない、または表の大部分が必要な場合。 |
| `TABLE ACCESS BY INDEX ROWID` | 索引ルックアップ後に ROWID で行を取得。 | 索引を使用した選択性の高い抽出。 |
| `INDEX UNIQUE SCAN` | 索引の単一エントリーを検索。 | 主キーまたは一意制約でのルックアップ。 |
| `INDEX RANGE SCAN` | 索引エントリーの範囲をスキャン。 | 範囲指定またはカーディナリティの低い等価検索。 |
| `INDEX FAST FULL SCAN` | 全索引ブロックを FTS のように読み込む。 | 必要な列が索引ですべてカバーされている場合（表アクセスを回避）。 |
| `INDEX SKIP SCAN` | 複合索引の先頭列をスキップして検索。 | 先頭列のカーディナリティが低い場合。 |

### 結合メソッド (Join Methods)

| 操作 | 説明 | 適しているケース |
|---|---|---|
| `NESTED LOOPS` | 外部行ごとに内部行をプローブ。 | 外部が小さく、内部に選択性の高い索引がある場合。 |
| `HASH JOIN` | 小さい側からハッシュ表を作成し、大きい側を走査。 | 大規模なデータセット、有効な索引がない場合。 |
| `MERGE JOIN` | 両方の入力をソートしてマージ結合。 | 入力が事前ソートされている場合、等価結合。 |
| `NESTED LOOPS ANTI` / `SEMI` | アンチ結合/セミ結合の最適化。 | NOT IN / EXISTS サブクエリ。 |

---

## SQL*Plus における AUTOTRACE

AUTOTRACE は、クエリ実行後にプランと実行統計を同時に表示する機能である。

```sql
-- セットアップ (ユーザーごとに 1 回、DBA 権限が必要)
-- プラン表と autotrace ロールへのアクセス権を付与
GRANT SELECT ON v_$session TO your_user;
GRANT SELECT ON v_$sql_plan TO your_user;
-- または単に:
GRANT PLUSTRACE TO your_user;

-- autotrace の有効化
SET AUTOTRACE ON            -- 結果 + プラン + 統計情報を表示
SET AUTOTRACE TRACEONLY     -- 結果を抑制し、プラン + 統計情報を表示
SET AUTOTRACE TRACEONLY EXPLAIN -- プランのみ表示 (実行はしない)
SET AUTOTRACE TRACEONLY STATISTICS -- 統計情報のみ表示 (実行はする)
SET AUTOTRACE OFF           -- 無効化

-- 実行例
SET AUTOTRACE TRACEONLY
SELECT * FROM employees WHERE department_id = 60;
```

Autotrace の出力には以下が含まれる：

```
Statistics
----------------------------------------------------------
          45  recursive calls
           0  db block gets
         182  consistent gets          <-- 論理読み込み
           3  physical reads           <-- ディスク読み込み
           0  redo size
        1423  bytes sent via SQL*Net
         608  bytes received via SQL*Net
           3  SQL*Net roundtrips
           1  sorts (memory)
           0  sorts (disk)
           6  rows processed
```

**重要な統計情報:**

- `consistent gets` — 論理読み込み（バッファ・キャッシュのルックアップ数）。
- `physical reads` — 実際のディスク読み取り。
- `sorts (disk)` — PGA のソート領域を超え、一時表領域（TEMP）を使用した回数。
- `recursive calls` — 内部的な SQL（データ・ディクショナリ参照、トリガーなど）。高い値は要調査。

---

## 悪いプランの特定方法

### 症状 1: カーディナリティの不一致 (Cardinality Mismatch)

```sql
-- GATHER_PLAN_STATISTICS を付けて実行した場合:
-- E-Rows = 5, A-Rows = 500,000
-- オプティマイザは 5 行と予測して NESTED LOOPS を選択してしまった。
-- 対策: 正確な統計情報の収集、拡張統計の検討、SQL ヒントの使用。
```

### 症状 2: 不適切な結合順序 (Wrong Join Order)

オプティマイザが、小さな表から開始せずに大きな表同士を先に結合している。通常、統計情報の不足や古さが原因である。

### 症状 3: インデックスを期待した箇所での全表スキャン

```sql
-- 一般的な原因:
-- 1. 索引付きの列に関数が適用されている（索引が使用されない）
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';
-- 対策: 関数ベースの索引を作成する
CREATE INDEX emp_upper_lname ON employees (UPPER(last_name));

-- 2. 暗黙のデータ型変換
SELECT * FROM employees WHERE employee_id = '100';  -- VARCHAR vs NUMBER
-- 対策: データ型を一致させる

-- 3. 前方ワイルドカード
SELECT * FROM employees WHERE last_name LIKE '%SMITH';
-- 対策: Oracle Text の使用、またはアプリケーションの再設計を検討。

-- 4. オプティマイザが FTS の方が安いと判断 (小規模な表、または大部分の行が返される)
-- これは正しい判断である場合もある。実際の行数で検証すること。
```

### 症状 4: 非効率なサブクエリ (Unnest されていない)

```sql
-- 外部行ごとに 1 回ずつ実行される相関サブクエリ
SELECT * FROM orders o
WHERE  total > (SELECT AVG(total) FROM orders WHERE customer_id = o.customer_id);
-- プラン上でサブクエリを伴う "FILTER" 操作を確認 — 多くの場合遅い。
-- 対策: インライン・ビューとの結合 (JOIN) への書き換え、または分析関数の使用。
SELECT * FROM (
  SELECT o.*, AVG(total) OVER (PARTITION BY customer_id) AS avg_total
  FROM   orders o
)
WHERE  total > avg_total;
```

### 症状 5: 大規模な表でのハッシュ結合ではなくソート・マージ結合

通常、システム統計（System Statistics）の収集が不足しているか古いことを示している。

```sql
-- テストのために手動でハッシュ結合をヒント指定してみる
SELECT /*+ USE_HASH(e d) */ e.last_name, d.department_name
FROM   employees e, departments d
WHERE  e.department_id = d.department_id;
```

---

## ベスト・プラクティス

- デバッグ時には常に **`GATHER_PLAN_STATISTICS`** ヒントまたは `STATISTICS_LEVEL=ALL` を使用し、推定行数と実際の行数を比較すること。
- **ステップごとに `E-Rows` と `A-Rows` を比較する。** 乖離が始まる最初のステップが修正の焦点となる。
- 本番環境の SQL デバッグには `EXPLAIN PLAN` ではなく **`DISPLAY_CURSOR`** を使用すること。`EXPLAIN PLAN` はバインド変数が含まれる場合に不正確なことがある。
- フォーマット文字列で **`+PEEKED_BINDS`** を指定し、プラン生成時に Oracle がどのバインド値を使用したかを確認すること。
- 統計情報の変更やアップグレード後のプラン退行を防ぐために、**SQL プラン管理 (SPM)** で良いプランを固定することを検討する。
- **「述語情報（Predicate Information）」セクションを必ず確認する。** フィルタ (`filter`) が期待した場所で適用されており、アクセス述語 (`access`) が索引構造と一致していることを確認する。

```sql
-- 良いプランを固定するために SQL プラン・ベースラインを作成する
DECLARE
  l_plans PLS_INTEGER;
BEGIN
  l_plans := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
    sql_id          => 'abc123xyz789',
    plan_hash_value => 1234567890
  );
  DBMS_OUTPUT.PUT_LINE('Plans loaded: ' || l_plans);
END;
/
```

---

## よくある間違い

| 間違い | 影響 | 対策 |
|---|---|---|
| バインド変数を含む SQL に EXPLAIN PLAN を使用する | 実行時のプランと異なる場合がある | `DISPLAY_CURSOR` と `+PEEKED_BINDS` を使用する |
| A-Rows と E-Rows の比較を行わない | カーディナリティ推定の問題を見逃す | 常に `GATHER_PLAN_STATISTICS` ヒントを使用する |
| コストが低い ＝ クエリが速いと思い込む | コストはオプティマイザの相対的なモデル値であり、実時間ではない | 実際の経過時間を測定する |
| 本番用コードに直接ヒントを記述する | 脆弱になり、オブジェクト変更時に壊れやすい | 統計情報を修正、索引を追加する。または SPM を使用する |
| 述語情報（Predicate Information）を無視する | filter と access の区別を見逃す | 述語セクションを必ず読む |
| 累積的な A-Time を混同する | ステップの時間は子ステップを含んでいる | 親の A-Time から子の A-Time を引いたものがそのステップの時間。 |
| FTS をすべて回避するために過剰に索引を作成する | DML パフォーマンスの低下、領域消費の増大 | FTS が本当にボトルネックか、まずは検証すること |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替策を保持すること。
- 19c と 26ai の両方をサポートする環境では、リリース更新によってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL Tuning Guide (TGSQL)](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/)
- [DBMS_XPLAN — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_XPLAN.html)
- [V$SQL — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQL.html)
- [DBMS_SPM — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SPM.html)
- [PLAN_TABLE — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/PLAN_TABLE.html)
racle-database/19/refrn/PLAN_TABLE.html)
