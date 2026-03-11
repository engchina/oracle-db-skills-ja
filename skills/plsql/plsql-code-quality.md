# PL/SQL コード品質 (Code Quality)

## 概要

一貫性があり、読みやすく、保守性の高い PL/SQL を作成するには、合意された命名規則、文書化されたアンチパターンの回避、および自動化された静的解析が必要です。このガイドでは、スタイル基準、アンチパターンの検出、コード・レビューの実践、およびツールについて説明します。

---

## 命名規則

一貫した命名は、読みやすいコードの基礎です。最も広く採用されている規約は、Oracle 社独自の内部標準や Trivadis PL/SQL コーディング・ガイドラインに基づいています。

### 変数の接頭辞 (Prefix)

| 接頭辞 | スコープ/種類 | 例 |
|---|---|---|
| `l_` | ローカル変数 (Local) | `l_employee_id`, `l_salary` |
| `g_` | パッケージ・グローバル変数 (Global) | `g_debug_enabled`, `g_config_loaded` |
| `p_` | パラメータ (Parameter) | `p_customer_id`, `p_result` |
| `c_` | ローカル定数 (Constant) | `c_max_retries`, `c_default_currency` |
| `gc_` | パッケージ・グローバル定数 | `gc_app_name`, `gc_max_batch_size` |
| `e_` | 例外変数 (Exception) | `e_order_not_found` |
| `r_` | レコード変数 (Record) | `r_employee`, `r_order` |
| `t_` | 型定義 (Type) | `t_id_list`, `t_order_tab` |
| `cur_` または `c_` | カーソル (Cursor) | `cur_employees`, `c_pending_orders` |

### オブジェクトの命名規則

| オブジェクト型 | 規約 | 例 |
|---|---|---|
| 表 (Table) | 複数形の名詞、snake_case | `employees`, `order_items` |
| パッケージ | ドメイン + `_pkg` | `order_mgmt_pkg`, `customer_api_pkg` |
| プロシージャ | 動詞 + 名詞 | `create_order`, `validate_customer` |
| ファンクション | 戻り値を表す名詞。`get_` や `is_`/`has_` | `get_tax_rate`, `is_valid_email` |
| トリガー | 表名 + `_trg` または `_trigger` | `employees_audit_trg` |
| シーケンス | 表名 + `_seq` | `orders_seq`, `employees_seq` |
| 索引 | `idx_` + 表名 + 列名 | `idx_orders_customer_id` |
| 型 (Type) | `t_` + 説明 | `t_employee_list`, `t_id_tab` |

### 命名規則を適用した完全な例

```sql
CREATE OR REPLACE PACKAGE order_mgmt_pkg AS
  -- パッケージ定数 (gc_ 接頭辞)
  gc_max_order_items CONSTANT PLS_INTEGER := 500;
  gc_default_currency CONSTANT VARCHAR2(3) := 'USD';

  -- 公開型 (t_ 接頭辞)
  TYPE t_order_summary IS RECORD (
    order_id       orders.order_id%TYPE,
    customer_name  VARCHAR2(100),
    total_amount   NUMBER
  );

  -- 公開例外 (e_ 接頭辞)
  e_order_not_found EXCEPTION;

  -- 公開プロシージャ/ファンクション
  FUNCTION get_order_summary(
    p_order_id IN orders.order_id%TYPE  -- 引数は p_ 接頭辞
  ) RETURN t_order_summary;

END order_mgmt_pkg;
/

CREATE OR REPLACE PACKAGE BODY order_mgmt_pkg AS

  -- 非公開グローバル変数 (g_ 接頭辞)
  g_cache_loaded BOOLEAN := FALSE;

  -- 非公開型
  TYPE t_cache_map IS TABLE OF t_order_summary INDEX BY PLS_INTEGER;
  g_cache t_cache_map;

  FUNCTION get_order_summary(
    p_order_id IN orders.order_id%TYPE
  ) RETURN t_order_summary IS
    -- ローカル変数 (l_ 接頭辞)
    l_summary   t_order_summary;
    -- ローカル定数 (c_ 接頭辞)
    c_not_found CONSTANT VARCHAR2(50) := '注文が見つかりません: ';
  BEGIN
    SELECT o.order_id, c.customer_name, o.total_amount
    INTO   l_summary.order_id, l_summary.customer_name, l_summary.total_amount
    FROM   orders    o
    JOIN   customers c ON c.customer_id = o.customer_id
    WHERE  o.order_id = p_order_id;

    RETURN l_summary;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      RAISE_APPLICATION_ERROR(-20001, c_not_found || p_order_id);
  END get_order_summary;

END order_mgmt_pkg;
/
```

---

## スタイル・ガイドライン (Oracle / Trivadis / PL/SQL Cop)

[Trivadis PL/SQL and SQL Coding Guidelines](https://trivadis.github.io/plsql-and-sql-coding-guidelines/) は、最も包括的な公開スタイル基準です。主なルールは以下の通りです：

### フォーマット

```sql
-- G-1010: 意味のある名前を使用する (ループカウンタを除き、1文字の名前は避ける)
-- 悪い例:
FOR i IN 1..l_count LOOP
  IF x > 0 THEN d := p * r; END IF;
END LOOP;

-- 良い例:
FOR idx IN 1..l_count LOOP
  IF l_amount > 0 THEN
    l_discount := l_price * l_rate;
  END IF;
END LOOP;

-- G-2130: 変数宣言を整列させる
l_employee_id   employees.employee_id%TYPE;
l_salary        employees.salary%TYPE;
l_department_id employees.department_id%TYPE;

-- G-4130: インデントを一貫させる (2 または 4 スペース)
BEGIN
  IF condition THEN
    do_something;
    IF nested_condition THEN
      do_nested_thing;
    END IF;
  END IF;
END;
```

### キーワードと識別子

```sql
-- G-1020: キーワードは大文字、識別子は小文字にする
-- 悪い例: select employee_id FROM Employees WHERE Department_Id = 10;
-- 良い例:
SELECT employee_id FROM employees WHERE department_id = 10;

-- G-2180: Oracle の予約語を識別子として使用しない
-- 悪い例: DECLARE date DATE; BEGIN ... END;
-- 良い例: DECLARE l_hire_date DATE; BEGIN ... END;

-- G-2230: 列に依存する変数宣言には %TYPE を使用する
l_salary employees.salary%TYPE;     -- 列の型変更に追従できる
-- 悪い例:
l_salary NUMBER(8,2);               -- 列の精度が変わるとエラーになる可能性がある
```

---

## アンチパターンのリファレンス

### WHEN OTHERS THEN NULL

```sql
-- 決して行ってはいけない例
EXCEPTION
  WHEN OTHERS THEN NULL;

-- 有害な理由:
-- 1. すべての例外を黙って破棄する
-- 2. 呼び出し側は操作が失敗したことを知ることができない
-- 3. データが不整合な状態になる可能性がある
-- 4. 本番環境での診断が不可能になる

-- 正しいパターン: 常にログを記録して再発生させる
EXCEPTION
  WHEN OTHERS THEN
    error_logger_pkg.log_error('MY_PKG', 'MY_PROC');
    RAISE;
```

### PL/SQL 内での SELECT *

```sql
-- 避けるべき例: SELECT *
DECLARE l_emp employees%ROWTYPE;
BEGIN
  SELECT * INTO l_emp FROM employees WHERE employee_id = 100;
  -- 問題: 列が追加、並び替え、または削除された場合、誤った値がマップされたり（%ROWTYPEがキャッシュされている場合）、実行時エラーが発生したりする可能性がある

-- 推奨される例: 列を明示的に指定する
DECLARE
  l_emp_id   employees.employee_id%TYPE;
  l_emp_name employees.last_name%TYPE;
BEGIN
  SELECT employee_id, last_name INTO l_emp_id, l_emp_name
  FROM   employees WHERE employee_id = 100;
```

**例外**: `%ROWTYPE` を使用し、かつ本当にすべての列が必要な場合に限っては `SELECT * ... INTO l_%ROWTYPE` は許容されます。

### スキーマ名のハードコード

```sql
-- 避けるべき例: スキーマ名のハードコード
SELECT * FROM hr.employees;  -- 別のスキーマにデプロイすると壊れる
INSERT INTO finance.accounts VALUES ...;

-- 推奨される例: シノニムまたは現行スキーマの参照を使用する
SELECT * FROM employees;     -- シノニムまたは現行スキーマ経由で解決される
```

### マジック・ナンバー

```sql
-- 避けるべき例: 説明のない数値リテラル
IF l_status_code = 3 THEN ...;        -- 「3」とは何か？
IF l_retry_count > 5 THEN ...;        -- なぜ「5」なのか？

-- 推奨される例: 名前付き定数を使用する
DECLARE
  c_status_shipped   CONSTANT PLS_INTEGER := 3;
  c_max_retries      CONSTANT PLS_INTEGER := 5;
BEGIN
  IF l_status_code = c_status_shipped THEN ...;
  IF l_retry_count > c_max_retries THEN ...;
```

---

## コード・レビュー・チェックリスト

- [ ] `WHEN OTHERS THEN NULL` が存在しないか
- [ ] すべての例外が `FORMAT_ERROR_BACKTRACE` 等でログに記録されているか
- [ ] DML の直後に `SQL%ROWCOUNT` が取得されているか
- [ ] 例外ハンドラ内でカーソルが閉じられているか（`%ISOPEN` チェック）
- [ ] WHERE 句で暗黙の型変換が行われていないか
- [ ] 日付リテラルに `DATE 'YYYY-MM-DD'` または書式指定の `TO_DATE` が使用されているか
- [ ] カーソル・ループ内での DML を避け、`FORALL` を使用しているか
- [ ] `BULK COLLECT` に `LIMIT` 句が使用されているか
- [ ] 動的 SQL で文字列連結ではなくバインド変数を使用しているか
- [ ] 動的 SQL 内の表名/列名が `DBMS_ASSERT` で検証されているか
- [ ] 変数名が接頭辞規則（l_, p_, g_, c_）に従っているか
- [ ] マジック・ナンバーを避け、定数が使用されているか
- [ ] 型宣言に `%TYPE` が使用されているか

---

## 静的解析ツール

### PL/SQL Cop (Trivadis)

PL/SQL Cop は、Trivadis のガイドラインに基づいた商業的な静的解析ツールです。

- **G-2150**: NULL との比較を避け、`IS NULL` / `IS NOT NULL` を使用すること
- **G-5030**: 仕様部にロジックを書かず、本体（Body）に記述すること
- **G-7810**: `RAISE` なしの `WHEN OTHERS` を使用しないこと

### SQL Developer コード分析

SQL Developer にはコード分析ツールが組み込まれています。
1. **[ツール] > [コード分析] > [分析を実行]**
2. ルール（PL/SQL ベスト・プラクティス、セキュリティなど）を選択。
3. 結果がコード分析パネルに行番号付きで表示されます。

---

## McCabe の循環的複雑度 (Cyclomatic Complexity)

循環的複雑度は、コード内の線形的に独立したパスの数を測定します。複雑度が高いほど、テストと保守が困難になります。

**計算式**: 複雑度 = 判定ポイントの数 + 1

判定ポイント: `IF`, `ELSIF`, `CASE WHEN`, `LOOP`, `WHILE`, `FOR`, `EXCEPTION WHEN`, 条件内の `AND`, `OR`。

| 複雑度 | 評価 | 推奨アクション |
|---|---|---|
| 1–5 | 低 | 良好 — 理解しやすく、テストも容易 |
| 6–10 | 中 | 許容範囲 — 注意深くレビューすること |
| 11–15 | 高 | リファクタリングを検討すべき |
| 16–25 | 非常に高い | リファクタリングを強く推奨 |
| > 25 | 極めて高い | コード・レビュー承認前にリファクタリングが必須 |

---

## プロシージャの長さのガイドライン

| 項目 | ガイドライン |
|---|---|
| プロシージャ本体の最大行数 | 60〜80行 (宣言部を除く) |
| パッケージ本体の最大行数 | 分割を検討する前に 1000〜1500行 |
| ネストの深さ | `IF`/`LOOP` のネストは 3〜4レベルまで |
| パラメータ数 | 7〜10個まで。それ以上の場合はレコード型を使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

---

## ソース

- [Trivadis PL/SQL and SQL Coding Guidelines](https://trivadis.github.io/plsql-and-sql-coding-guidelines/)
- [Oracle Database PL/SQL Language Reference 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/)
- [Oracle Database Reference 19c — USER_ERRORS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/USER_ERRORS.html)
