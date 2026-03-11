# PL/SQL コレクション (Collections)

## 概要

コレクションは、PL/SQL においてメモリ内でデータを扱うための主要なデータ構造です。コレクションには 3 つの異なるタイプがあり、それぞれストレージのセマンティクス、SQL での可視性、および初期化の要件が異なります。それぞれのタイプをいつ使用すべきか、またコレクション・メソッドをどのように安全に使用するかを理解することは、正確性とパフォーマンスの両面で不可欠です。

---

## コレクション・タイプの比較

| 機能 | 結合配列 (Associative Array) | ネストした表 (Nested Table) | 可変長配列 (Varray) |
|---|---|---|---|
| **別名** | 索引付き表 (Index-by table) | 無制限配列 | 可変サイズ配列 |
| **索引の型** | `PLS_INTEGER` または `VARCHAR2` | 整数 (1 から開始) | 整数 (1 から開始) |
| **最大サイズ** | 無制限 | 無制限 | 宣言時に固定 |
| **疎 (sparse) の可否** | 可 | 可 (DELETE 後) | 不可 |
| **初期化の要件** | 不要 | 必要 (使用前) | 必要 (使用前) |
| **NULL の可否** | NULL にできない | NULL にできる (アトミック NULL) | NULL にできる (アトミック NULL) |
| **DB への保存** | 不可 (PL/SQL のみ) | 可 (列の型として) | 可 (列の型として) |
| **SQL TABLE() アクセス** | 不可 | 可 | 可 |
| **オブジェクト型内へのネスト** | 不可 | 可 | 可 |
| **多次元ネスト** | 不可 | 可 | 可 |
| **主なユースケース** | メモリ内の検索/キャッシュ、文字列索引のマップ | 集合演算、SQL 連携、柔軟な DML | 固定サイズ配列、データベース・ストレージ |

---

## 結合配列 (Associative Arrays)

`INDEX BY` を指定して宣言します。初期化は不要です。文字列または整数のキーを使用できます。

```sql
DECLARE
  -- 整数索引
  TYPE t_salary_map IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  l_salaries t_salary_map;

  -- 文字列索引 (ルックアップ表/ハッシュ・マップ・パターン)
  TYPE t_country_map IS TABLE OF VARCHAR2(100) INDEX BY VARCHAR2(3);
  l_countries t_country_map;
BEGIN
  -- 整数索引のデータ設定
  l_salaries(1)   := 50000;
  l_salaries(2)   := 60000;
  l_salaries(1000) := 75000;  -- 疎 (sparse): ギャップがあっても問題なし


  -- 文字列索引のデータ設定 (ディクショナリ/マップ・パターン)
  l_countries('USA') := 'United States';
  l_countries('GBR') := 'United Kingdom';
  l_countries('DEU') := 'Germany';

  -- 検索
  IF l_countries.EXISTS('USA') THEN
    DBMS_OUTPUT.PUT_LINE(l_countries('USA'));
  END IF;

  -- FIRST/NEXT を使用した反復処理
  DECLARE
    l_key VARCHAR2(3) := l_countries.FIRST;
  BEGIN
    WHILE l_key IS NOT NULL LOOP
      DBMS_OUTPUT.PUT_LINE(l_key || ': ' || l_countries(l_key));
      l_key := l_countries.NEXT(l_key);
    END LOOP;
  END;
END;
/
```

**ユースケース**: 参照データのキャッシュ、メモリ内ハッシュ・マップの構築、文字列キーが必要な場合。

---

## ネストした表 (Nested Tables)

使用前に初期化する必要があります。すべてのコレクション・メソッドと SQL の `TABLE()` 演算子をサポートしています。

```sql
-- スキーマ・レベルの型 (DB に保存可能、SQL で使用可能)
CREATE OR REPLACE TYPE t_name_list AS TABLE OF VARCHAR2(100);
/

-- パッケージ・レベルの型 (PL/SQL のみ)
DECLARE
  TYPE t_employee_ids IS TABLE OF employees.employee_id%TYPE;

  -- 使用前に初期化が必要
  l_ids t_employee_ids := t_employee_ids();  -- 空のネストした表
  l_names t_name_list  := t_name_list('Alice', 'Bob', 'Carol');
BEGIN
  -- 値を割り当てる前に EXTEND が必要
  l_ids.EXTEND;
  l_ids(1) := 101;

  l_ids.EXTEND(3);  -- 3 つの空のスロットを追加
  l_ids(2) := 102;
  l_ids(3) := 103;
  l_ids(4) := 104;

  -- DELETE により疎 (sparse) な表になる
  l_ids.DELETE(2);  -- l_ids(2) は存在しなくなるが、l_ids(3) は 103 のまま

  -- 疎な表を安全に走査する
  FOR i IN 1..l_ids.LAST LOOP
    IF l_ids.EXISTS(i) THEN
      DBMS_OUTPUT.PUT_LINE(l_ids(i));
    END IF;
  END LOOP;

  -- TABLE() 経由での SQL アクセス
  SELECT column_value FROM TABLE(l_names);
END;
/
```

---

## 可変長配列 (Varrays)

作成時に最大サイズを指定します。疎にすることはできません。初期化が必要です。

```sql
-- 最大 12 個の要素を持つ Varray 型
CREATE OR REPLACE TYPE t_month_names AS VARRAY(12) OF VARCHAR2(20);
/

DECLARE
  l_months t_month_names := t_month_names(
    'January', 'February', 'March', 'April',
    'May', 'June', 'July', 'August',
    'September', 'October', 'November', 'December'
  );
BEGIN
  -- 最大 12 要素まで保持可能 (型で定義)
  DBMS_OUTPUT.PUT_LINE('Month 3: ' || l_months(3));  -- March
  DBMS_OUTPUT.PUT_LINE('Count: ' || l_months.COUNT);  -- 12
  DBMS_OUTPUT.PUT_LINE('Limit: ' || l_months.LIMIT);  -- 12

  -- Varray から個別の要素を DELETE することはできない
  -- l_months.DELETE(1);  -- これはエラーになる

  -- 末尾から TRIM することは可能
  l_months.TRIM;    -- 最後の要素を削除
  l_months.TRIM(2); -- 最後の 2 要素を削除
END;
/
```

**ユースケース**: 最大サイズが既知で固定されている場合（月、曜日、データベース列に保存される固定サイズの参照用配列）。

---

## コレクション・メソッド・リファレンス

| メソッド | 結合配列 | ネストした表 | Varray | 説明 |
|---|---|---|---|---|
| `COUNT` | 可 | 可 | 可 | 現在コレクション内にある要素数 |
| `FIRST` | 可 | 可 | 可 | 最小の索引 (空の場合は NULL) |
| `LAST` | 可 | 可 | 可 | 最大の索引 (空の場合は NULL) |
| `NEXT(n)` | 可 | 可 | 可 | n の次の索引 (存在しない場合は NULL) |
| `PRIOR(n)` | 可 | 可 | 可 | n の前の索引 (存在しない場合は NULL) |
| `EXISTS(n)` | 可 | 可 | 可 | 索引 n に要素が存在すれば TRUE |
| `DELETE` | 可 (全削除) | 可 (全削除) | 不可 | すべての要素を削除 |
| `DELETE(n)` | 可 | 可 | 不可 | 索引 n の要素を削除 |
| `DELETE(m,n)` | 可 | 可 | 不可 | 索引 m から n の要素を削除 |
| `EXTEND` | 否 | 可 | 可 | NULL 要素を 1 つ追加 |
| `EXTEND(n)` | 否 | 可 | 可 | NULL 要素を n 個追加 |
| `EXTEND(n,i)` | 否 | 可 | 可 | 索引 i の要素を n 個コピーして追加 |
| `TRIM` | 否 | 可 | 可 | 最後の要素を削除 |
| `TRIM(n)` | 否 | 可 | 可 | 最後の n 個の要素を削除 |
| `LIMIT` | 否 | 否 | 可 | 許容される最大要素数 |

```sql
DECLARE
  TYPE t_numbers IS TABLE OF NUMBER;
  l_nums t_numbers := t_numbers(10, 20, 30, 40, 50);
BEGIN
  DBMS_OUTPUT.PUT_LINE('COUNT: ' || l_nums.COUNT);   -- 5
  DBMS_OUTPUT.PUT_LINE('FIRST: ' || l_nums.FIRST);   -- 1
  DBMS_OUTPUT.PUT_LINE('LAST: '  || l_nums.LAST);    -- 5
  DBMS_OUTPUT.PUT_LINE('NEXT(3): ' || l_nums.NEXT(3)); -- 4
  DBMS_OUTPUT.PUT_LINE('PRIOR(3): '|| l_nums.PRIOR(3)); -- 2

  l_nums.DELETE(3);  -- 索引 3 の要素を削除
  DBMS_OUTPUT.PUT_LINE('COUNT after DELETE(3): ' || l_nums.COUNT); -- 4
  DBMS_OUTPUT.PUT_LINE('EXISTS(3): ' || CASE WHEN l_nums.EXISTS(3) THEN 'Y' ELSE 'N' END); -- N
  DBMS_OUTPUT.PUT_LINE('NEXT(2): ' || l_nums.NEXT(2)); -- 4 (削除された 3 をスキップ)

  l_nums.EXTEND(2);  -- 索引 6 と 7 に 2 つの NULL 要素を追加
  l_nums(l_nums.LAST) := 60;  -- 拡張された最後の要素に代入

  l_nums.TRIM;  -- 最後の要素を削除
  DBMS_OUTPUT.PUT_LINE('COUNT after TRIM: ' || l_nums.COUNT);
END;
/
```

---

## コレクションを使用したバルク操作

コレクションは、`FORALL` によるバルク DML および `BULK COLLECT` によるバルク SELECT を行うための手段です。

```sql
DECLARE
  TYPE t_emp_id_tab  IS TABLE OF employees.employee_id%TYPE;
  TYPE t_dept_id_tab IS TABLE OF employees.department_id%TYPE;
  TYPE t_sal_tab     IS TABLE OF employees.salary%TYPE;

  l_emp_ids  t_emp_id_tab;
  l_dept_ids t_dept_id_tab;
  l_salaries t_sal_tab;

  CURSOR c_dept10 IS
    SELECT employee_id, department_id, salary
    FROM   employees
    WHERE  department_id = 10;
BEGIN
  -- パラレル・コレクションへの一括フェッチ
  OPEN c_dept10;
  FETCH c_dept10 BULK COLLECT INTO l_emp_ids, l_dept_ids, l_salaries LIMIT 500;
  CLOSE c_dept10;

  -- PL/SQL 内での変更
  FOR i IN 1..l_salaries.COUNT LOOP
    l_salaries(i) := l_salaries(i) * 1.05;  -- 5% の昇給
  END LOOP;

  -- 単一の FORALL ですべての DML を一度に送信
  FORALL i IN 1..l_emp_ids.COUNT
    UPDATE employees
    SET    salary = l_salaries(i)
    WHERE  employee_id = l_emp_ids(i);

  DBMS_OUTPUT.PUT_LINE('Updated: ' || SQL%ROWCOUNT || ' rows');
  COMMIT;
END;
/
```

**%ROWTYPE を使用したより簡潔なコード**:

```sql
DECLARE
  TYPE t_emp_tab IS TABLE OF employees%ROWTYPE;
  l_employees t_emp_tab;
BEGIN
  SELECT * BULK COLLECT INTO l_employees
  FROM   employees
  WHERE  department_id = 20
  LIMIT 1000;

  FOR i IN 1..l_employees.COUNT LOOP
    DBMS_OUTPUT.PUT_LINE(l_employees(i).last_name);
  END LOOP;
END;
/
```

---

## SQL アクセスのための TABLE() と CAST()

スキーマ・レベルのネストした表および Varray 型は、`TABLE()` 演算子を使用して SQL 内で使用できます。

```sql
-- SQL の FROM 句でネストした表を使用
CREATE OR REPLACE TYPE t_number_list AS TABLE OF NUMBER;
/

-- SQL 内でのインライン・コレクション
SELECT column_value AS id
FROM   TABLE(t_number_list(10, 20, 30, 40));

-- PL/SQL から SQL へコレクションを渡す
DECLARE
  l_dept_ids t_number_list := t_number_list(10, 20, 30);
BEGIN
  -- コレクションからの IN リスト
  FOR rec IN (
    SELECT employee_id, last_name
    FROM   employees
    WHERE  department_id IN (SELECT column_value FROM TABLE(l_dept_ids))
  ) LOOP
    DBMS_OUTPUT.PUT_LINE(rec.last_name);
  END LOOP;
END;
/

-- 型変換のための CAST() と MULTISET
SELECT d.department_name,
       CAST(MULTISET(
         SELECT e.last_name
         FROM   employees e
         WHERE  e.department_id = d.department_id
       ) AS t_name_list) AS employee_names
FROM   departments d;
```

---

## コレクションの例外

| 例外 | ORA コード | 発生条件 |
|---|---|---|
| `COLLECTION_IS_NULL` | ORA-06531 | 未初期化 (アトミック NULL) のネストした表または Varray に対してメソッドが呼び出された |
| `NO_DATA_FOUND` | ORA-01403 | サブスクリプトが削除された要素または存在しない要素を参照した |
| `SUBSCRIPT_BEYOND_COUNT` | ORA-06533 | サブスクリプトが COUNT を超えた |
| `SUBSCRIPT_OUTSIDE_LIMIT` | ORA-06532 | Varray のサブスクリプトが 1 未満または LIMIT を超えた |
| `VALUE_ERROR` | ORA-06502 | 整数演算索引のコレクションに対して整数以外のサブスクリプトを使用した |

```sql
DECLARE
  TYPE t_nums IS TABLE OF NUMBER;
  l_nums t_nums;  -- 未初期化 = NULL
BEGIN
  -- COLLECTION_IS_NULL: l_nums は単に空なだけでなく NULL である
  l_nums.EXTEND;  -- ORA-06531: 未初期化コレクションへの参照

EXCEPTION
  WHEN COLLECTION_IS_NULL THEN
    DBMS_OUTPUT.PUT_LINE('最初にコンストラクタで初期化してください: l_nums := t_nums()');
END;
/

DECLARE
  TYPE t_nums IS TABLE OF NUMBER;
  l_nums t_nums := t_nums(1, 2, 3);
BEGIN
  l_nums.DELETE(2);

  -- NO_DATA_FOUND: 要素 2 は削除されている
  DBMS_OUTPUT.PUT_LINE(l_nums(2));  -- ORA-01403

  -- SUBSCRIPT_BEYOND_COUNT: 2 つの要素のみが残っている (インデックス 1 と 3)
  DBMS_OUTPUT.PUT_LINE(l_nums(10)); -- ORA-06533

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('要素が存在しません — 最初に EXISTS() を使用してください');
  WHEN SUBSCRIPT_BEYOND_COUNT THEN
    DBMS_OUTPUT.PUT_LINE('インデックスが COUNT を超えています');
END;
/
```

**防御的なパターン**: 索引によるアクセスを行う前、特に `DELETE` の後は、常に `EXISTS()` をチェックしてください。

---

## 多次元コレクション

ネストした表には、他のコレクション・タイプを含めることができます（ネストした表のネストした表）。

```sql
-- 多次元: 表の表
CREATE OR REPLACE TYPE t_string_list AS TABLE OF VARCHAR2(100);
/

CREATE OR REPLACE TYPE t_string_matrix AS TABLE OF t_string_list;
/

DECLARE
  l_matrix t_string_matrix := t_string_matrix();
  l_row1   t_string_list   := t_string_list('Q1', 'Q2', 'Q3', 'Q4');
  l_row2   t_string_list   := t_string_list('Jan', 'Feb', 'Mar');
BEGIN
  l_matrix.EXTEND(2);
  l_matrix(1) := l_row1;
  l_matrix(2) := l_row2;

  -- アクセス: outer(row)(column)
  DBMS_OUTPUT.PUT_LINE(l_matrix(1)(2));  -- Q2
  DBMS_OUTPUT.PUT_LINE(l_matrix(2)(1));  -- Jan

  -- 反復処理
  FOR i IN 1..l_matrix.COUNT LOOP
    FOR j IN 1..l_matrix(i).COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(i || ',' || j || ': ' || l_matrix(i)(j));
    END LOOP;
  END LOOP;
END;
/
```

---

## ベスト・プラクティス

- PL/SQL におけるすべての一時的なメモリ内ストレージには **結合配列** を使用します。初期化のオーバーヘッドがありません。
- SQL アクセス (`TABLE()`)、バルク DML、または集合演算 (`MULTISET UNION`, `MULTISET INTERSECT`) が必要な場合は、**ネストした表** を使用します。
- 最大サイズが真に固定されており、その型をデータベース・列に保存する必要がある場合のみ **Varray** を使用します。
- 削除された可能性のある要素にアクセスする前には、必ず `EXISTS()` を呼び出してください。
- 疎になる可能性があるコレクションを走査する場合は、数値 FOR ループではなく `FIRST`/`NEXT` を使用します。
- ループ内では `BULK COLLECT ... LIMIT` を使用してください。件数が無制限の表に対して `LIMIT` なしの `BULK COLLECT` は絶対に使用しないでください。
- スキーマ変更に対応できるよう、コレクション要素の型は `%TYPE` を使用して列定義に固定します。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| 未初期化のネストした表へのアクセス | `ORA-06531: COLLECTION_IS_NULL` | コンストラクタで初期化: `l_tab := t_tab()` |
| `EXISTS()` なしで削除された要素へのアクセス | `ORA-01403` | 常に `IF l_tab.EXISTS(i)` をチェック |
| 疎な表に対して `FOR i IN 1..l_tab.COUNT` でループ | `EXISTS` チェックをスキップし、`COUNT` は削除分を含む | `FIRST`/`NEXT` ループを使用 |
| `LIMIT` なしの `BULK COLLECT` | 全結果セットを PGA にロード | 常に `LIMIT n` を追加 |
| `DELETE` を使用した後に `TRIM` を使用 | `TRIM` はギャップに関係なく末尾から削除 | `DELETE` か `TRIM` のどちらか一貫して使用 |
| ネストした表に NULL を代入 | アトミック NULL になり、全メソッド・コールが失敗 | NULL ではなく空のコンストラクタを代入 |
| 疎なコレクションに対する `FORALL` | ORA-22160: index に要素が存在しない | `FORALL i IN INDICES OF l_tab` を使用 |

```sql
-- 疎なコレクションに対する FORALL: INDICES OF を使用
DECLARE
  TYPE t_ids IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
  l_ids t_ids;
BEGIN
  l_ids(1)  := 101;
  l_ids(5)  := 105;  -- ギャップ: 2,3,4 は存在しない
  l_ids(10) := 110;

  -- FORALL i IN 1..l_ids.LAST はギャップがあるため失敗する
  -- INDICES OF を使用して存在する要素のみを処理
  FORALL i IN INDICES OF l_ids
    UPDATE employees SET status = 'PROCESSED' WHERE employee_id = l_ids(i);
END;
/
```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **Oracle 9i以降**: `BULK COLLECT`、`FORALL`、および `TABLE()` 演算子は成熟しており、完全にサポートされています。
- **Oracle 10g以降**: 文字列索引の結合配列 (`INDEX BY VARCHAR2(n)`) が完全にサポートされています。
- **Oracle 12c以降**: `ACCESSIBLE BY` により、パッケージ・レベルのコレクション型を使用できるユニットを制限できます。
- **Oracle 21c以降**: ドキュメント・スタイルのデータに対する、コレクションと JSON の統合が向上しました。

---

## ソース

- [Oracle Database PL/SQL Language Reference 19c — Collection Types](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-collections-and-records.html) — 結合配列、ネストした表、Varray、BULK COLLECT、FORALL、INDICES OF
- [Oracle Database PL/SQL Language Reference 19c — FORALL Statement](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/FORALL-statement.html) — INDICES OF、VALUES OF、疎なコレクションの処理
