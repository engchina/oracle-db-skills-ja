# Oracle 仮想列 (Virtual Columns)

## 概要

**仮想列**とは、ディスク上に物理的にデータが保存されない列のことである。代わりに、決定論的な式（確定的な式）として定義され、クエリ、索引、制約、またはパーティション・キーでその列が参照されるたびに、Oracle によって動的に計算される。

仮想列は Oracle 11g Release 1 で導入された。これにより、ストレージのオーバーヘッド、トリガーによるメンテナンスの負担、あるいは複雑なインライン式の記述なしに、派生データをあたかも通常の列であるかのように扱うことができる。

**仮想列が役立つケース:**
- 頻繁に使用される計算式を名前付きの列として公開する
- 関数ベースの索引を直接書かずに、複雑な式に対して索引を作成する
- 計算された値をパーティション・キーとして使用する
- 派生値に対してチェック制約を適用し、ビジネス・ルールを強制する
- 基底のロジックが変わっても、ビューやアプリケーションに対して安定したインターフェースを提供する

---

## 仮想列の定義

### 基本構文

```sql
column_name [data_type] [GENERATED ALWAYS] AS (expression) [VIRTUAL]
```

- `GENERATED ALWAYS AS (expression)` は必須の構文である。
- `VIRTUAL` キーワードはオプションだが、明示することが推奨される。
- データ型は省略可能。Oracle が式から型を推論する。

### 例：基本的な仮想列を持つ表の作成

```sql
CREATE TABLE employees (
    employee_id   NUMBER(6)     NOT NULL,
    first_name    VARCHAR2(50)  NOT NULL,
    last_name     VARCHAR2(50)  NOT NULL,
    salary        NUMBER(10,2)  NOT NULL,
    commission_pct NUMBER(3,2),

    -- 表示用の姓名連結列（データは保存されない）
    full_name     VARCHAR2(101) GENERATED ALWAYS AS (first_name || ' ' || last_name) VIRTUAL,

    -- 手当を含めた年収計算
    annual_comp   NUMBER        GENERATED ALWAYS AS (
        salary * 12 * NVL(1 + commission_pct, 1)
    ) VIRTUAL,

    CONSTRAINT pk_employees PRIMARY KEY (employee_id)
);
```

---

## 関数ベースの仮想列

仮想列は、**決定論的 (DETERMINISTIC)** と宣言された PL/SQL 関数を呼び出すことができる。

```sql
CREATE OR REPLACE FUNCTION fiscal_year(p_date IN DATE)
RETURN NUMBER DETERMINISTIC AS
BEGIN
    -- 4月1日を年度の開始とする例
    RETURN CASE
        WHEN EXTRACT(MONTH FROM p_date) >= 4
        THEN EXTRACT(YEAR FROM p_date)
        ELSE EXTRACT(YEAR FROM p_date) - 1
    END;
END fiscal_year;
/

CREATE TABLE sales_orders (
    order_id       NUMBER         NOT NULL,
    order_date     DATE           NOT NULL,
    -- 決定論的関数を使用した仮想列
    fiscal_yr      NUMBER         GENERATED ALWAYS AS (fiscal_year(order_date)) VIRTUAL
);
```

**重要:** 関数は必ず `DETERMINISTIC` として宣言されている必要がある。そうでない場合、仮想列に対して索引を作成した際に不正確な結果を招く可能性がある。

---

## 仮想列への索引作成

仮想列の最も重要なユースケースの一つは、仮想列に対して直接 B 樹索引を作成できることである。これは関数ベースの索引と同じ効果を持つが、より直感的に管理できる。

```sql
-- 仮想列に対して索引を作成
CREATE INDEX idx_emp_full_name ON employees (full_name);

-- オプティマイザはこの索引を使用して検索を高速化できる
SELECT * FROM employees WHERE full_name = '田中 太郎';
```

---

## パーティション・キーとしての仮想列

仮想列をパーティション・キーとして使用することで、データを非正規化することなく、派生した値（年度や特定のコードなど）に基づいたパーティショニングが可能になる。

```sql
CREATE TABLE financial_transactions (
    txn_id        NUMBER         NOT NULL,
    txn_date      DATE           NOT NULL,
    -- 仮想列をパーティション・キーにする
    txn_fiscal_yr NUMBER         GENERATED ALWAYS AS (fiscal_year(txn_date)) VIRTUAL
)
PARTITION BY RANGE (txn_fiscal_yr) (
    PARTITION p_fy2024 VALUES LESS THAN (2025),
    PARTITION p_fy2025 VALUES LESS THAN (2026)
);
```

---

## 制限事項と注意点

- **DML 操作の禁止:** 仮想列に対して直接 INSERT や UPDATE を行うことはできない（エラー ORA-54013 が発生する）。
- **式の決定論性:** 式は決定論的である必要がある。つまり、同じ入力に対して常に同じ結果を返す必要がある。`SYSDATE` や他の表の値を参照する式は使用できない。
- **同一行内の参照:** 式は同じ行内の他の列を参照できるが、他の仮想列を参照できるかどうかは Oracle のバージョンに依存する。
- **統計情報:** 仮想列に対しても統計情報を収集できる。オプティマイザはこれを利用して最適な実行計画を作成する。

---

## ベスト・プラクティス

- **複雑なロジックは関数化する:** 定義内に複雑な式を直接書くのではなく、決定論的な関数として定義することで、再利用性と可読性が向上する。
- **検索条件になる仮想列には索引を貼る:** 索引がない場合、検索のたびに式が評価されるため、大規模な表ではパフォーマンスが低下する。
- **名前から仮想列と分かるようにする:** 物理列と区別するために、接尾辞（例：`_V`）を付けるなどの命名ルールを検討する。
- **解説をコメントに残す:** `COMMENT ON COLUMN` を使用して、その列がどのような計算式に基づいているかを記述しておく。

---

## よくある間違い

**間違い: 仮想列に値を INSERT しようとする。**
アプリケーション・コードで `INSERT INTO table VALUES (... , virtual_val)` のように記述するとエラーになる。仮想列は INSERT 対象から除外しなければならない。

**間違い: 関数の中身を変更した後、索引を再構築しない。**
仮想列が参照する関数のロジックを変更した場合、既存の索引データは古いロジックに基づいたままである。関数変更後は必ず関連する索引を `REBUILD` すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [Oracle Database SQL Language Reference: Virtual Columns](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database Administrator's Guide: Managing Tables](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-tables.html)
