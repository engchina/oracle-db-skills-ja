# Oracle DatabaseのERD設計と正規化

## 概要

実体関連設計 (ERD: Entity Relationship Design) は、適切に構造化されたリレーショナル・データベースを構築するための基礎となるステップである。Oracle 環境において適切に設計された ERD は、保守性の高いスキーマ、効率的なクエリ、および予測可能な拡張性に直結する。このガイドでは、エンティティとリレーションシップのモデリング、すべての正規形による正規化、Oracle 固有の命名規則、およびカーディナリティ・モデリングのベスト・プラクティスなど、ERD 設計の全範囲を網羅する。

---

## 1. ERD のコア概念

### エンティティ (Entities)

**エンティティ**とは、データが格納される対象となる、明確に区別できるオブジェクト（実体）または概念を表す。Oracle では、通常、各エンティティは 1 つのテーブルに対応する。エンティティは以下の 2 つのカテゴリに分類される：

- **強実体 (Strong entities)**: 独立して存在する（例：`CUSTOMER`, `PRODUCT`）。
- **弱実体 (Weak entities)**: 存在が強実体に依存する（例：`ORDER_ITEM` は `ORDER` に依存する）。

### 属性 (Attributes)

属性は、エンティティのプロパティ（性質）を表す。Oracle 固有の考慮事項は以下の通り：

| 属性タイプ | 説明 | Oracle でのマッピング |
|---|---|---|
| 単一 (Simple) | 単一の値、不可分 | 標準的な列 |
| 複合 (Composite) | 部分に分割可能（例：フルネーム） | 複数の列に分けることが推奨される |
| 誘導 (Derived) | 他のデータから計算される | 仮想列 (Virtual column) またはビュー |
| 多値 (Multi-valued) | 複数の値を保持できる | 子テーブル（配列は避ける） |

### リレーションシップ (Relationships)

リレーションシップは、エンティティ同士がどのように関連し合うかを定義する。Oracle では、**外部キー制約**、**チェック制約**、および**トリガー**を使用してリレーションシップを適用する。

---

## 2. リレーションシップのカーディナリティ (多重度)

カーディナリティは、2 つのエンティティ間の関係の「数」を定義する。

### 1 対 1 (1:1)

実務では稀である。多くの場合、テーブルの統合が検討されるべきか、あるいはセキュリティ上の理由で分割されていることを示す。

```sql
CREATE TABLE EMPLOYEE (
    employee_id   NUMBER(10)    NOT NULL,
    full_name     VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_employee PRIMARY KEY (employee_id)
);

CREATE TABLE EMPLOYEE_SECURITY (
    employee_id   NUMBER(10)    NOT NULL,
    password_hash VARCHAR2(256) NOT NULL,
    last_login    TIMESTAMP,
    CONSTRAINT pk_emp_security  PRIMARY KEY (employee_id),
    CONSTRAINT fk_emp_security  FOREIGN KEY (employee_id)
                                REFERENCES EMPLOYEE (employee_id)
                                ON DELETE CASCADE
);
```

### 1 対 多 (1:N)

最も一般的なリレーションシップ・タイプ。「多」側に外部キーを持たせる。

```sql
CREATE TABLE DEPARTMENT (
    department_id   NUMBER(6)    NOT NULL,
    department_name VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_department PRIMARY KEY (department_id)
);

CREATE TABLE EMPLOYEE (
    employee_id     NUMBER(10)    NOT NULL,
    full_name       VARCHAR2(100) NOT NULL,
    department_id   NUMBER(6)     NOT NULL,
    hire_date       DATE          NOT NULL,
    CONSTRAINT pk_employee    PRIMARY KEY (employee_id),
    CONSTRAINT fk_emp_dept    FOREIGN KEY (department_id)
                              REFERENCES DEPARTMENT (department_id)
);
```

### 多 対 多 (M:N)

**関連表 (associative table / junction table)** を介して解消される。関連表自体が属性を持つことも多い。

```sql
CREATE TABLE STUDENT (
    student_id  NUMBER(10)   NOT NULL,
    full_name   VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_student PRIMARY KEY (student_id)
);

CREATE TABLE COURSE (
    course_id   NUMBER(6)    NOT NULL,
    course_name VARCHAR2(200) NOT NULL,
    CONSTRAINT pk_course PRIMARY KEY (course_id)
);

-- 独自の属性を持つ関連表 (中間表)
CREATE TABLE ENROLLMENT (
    student_id    NUMBER(10) NOT NULL,
    course_id     NUMBER(6)  NOT NULL,
    enrolled_date DATE       NOT NULL,
    grade         VARCHAR2(2),
    CONSTRAINT pk_enrollment  PRIMARY KEY (student_id, course_id),
    CONSTRAINT fk_enroll_stu  FOREIGN KEY (student_id) REFERENCES STUDENT (student_id),
    CONSTRAINT fk_enroll_crs  FOREIGN KEY (course_id)  REFERENCES COURSE  (course_id)
);
```

### 自己参照 (再帰) リレーションシップ

組織図や部品構成表 (BOM) などの階層データに一般的である。

```sql
CREATE TABLE CATEGORY (
    category_id        NUMBER(10)    NOT NULL,
    category_name      VARCHAR2(100) NOT NULL,
    parent_category_id NUMBER(10),              -- ルート・ノードの場合は NULL
    CONSTRAINT pk_category     PRIMARY KEY (category_id),
    CONSTRAINT fk_cat_parent   FOREIGN KEY (parent_category_id)
                               REFERENCES CATEGORY (category_id)
);
```

Oracle でこの階層をトラバース（巡回）するには、`CONNECT BY` または再帰的 CTE (11g R2 以降) を使用する：

```sql
-- Oracle 階層クエリ
SELECT category_id, category_name, LEVEL AS depth,
       SYS_CONNECT_BY_PATH(category_name, ' > ') AS full_path
FROM   CATEGORY
START WITH parent_category_id IS NULL
CONNECT BY PRIOR category_id = parent_category_id
ORDER  SIBLINGS BY category_name;
```

---

## 3. 正規化 (Normalization)

正規化は、データを適切に構造化されたテーブルに整理することで、データの冗長性を削減し、データ整合性を向上させる。各正規形は前のステップの上に成り立つ。

### 第 1 正規形 (1NF)

**ルール:**
- すべての列の値は原子性 (不可分) を持たなければならない。
- 繰り返しグループや配列を持ってはいけない。
- 各行は一意に識別可能（主キーが存在する）でなければならない。

**違反例:**

```
CUSTOMER(customer_id, name, phone1, phone2, phone3)  -- 繰り返しグループ
CUSTOMER(customer_id, name, "555-1234, 555-5678")    -- 非原子的な値
```

**1NF での解決:**

```sql
CREATE TABLE CUSTOMER (
    customer_id  NUMBER(10)   NOT NULL,
    full_name    VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_customer PRIMARY KEY (customer_id)
);

CREATE TABLE CUSTOMER_PHONE (
    customer_id  NUMBER(10)   NOT NULL,
    phone_type   VARCHAR2(20) NOT NULL,  -- MOBILE, HOME, WORK
    phone_number VARCHAR2(20) NOT NULL,
    CONSTRAINT pk_cust_phone  PRIMARY KEY (customer_id, phone_type),
    CONSTRAINT fk_cust_phone  FOREIGN KEY (customer_id)
                              REFERENCES CUSTOMER (customer_id)
);
```

### 第 2 正規形 (2NF)

**ルール:**
- 1NF であること。
- すべての非キー属性は、主キーの**全体**に対して完全に関数従属していなければならない（部分従属があってはいけない。主キーが複合キーの場合にのみ関係する）。

**違反例** (複合主キー: `order_id + product_id`):

```
ORDER_ITEM(order_id, product_id, quantity, product_name, product_price)
-- product_name と product_price は product_id のみに従属しており、全キーには従属していない
```

**2NF での解決:**

```sql
CREATE TABLE PRODUCT (
    product_id    NUMBER(10)     NOT NULL,
    product_name  VARCHAR2(200)  NOT NULL,
    unit_price    NUMBER(12,2)   NOT NULL,
    CONSTRAINT pk_product PRIMARY KEY (product_id)
);

CREATE TABLE ORDER_ITEM (
    order_id    NUMBER(10) NOT NULL,
    product_id  NUMBER(10) NOT NULL,
    quantity    NUMBER(8)  NOT NULL,
    unit_price  NUMBER(12,2) NOT NULL,  -- 注文時の価格（スナップショット）として保持
    CONSTRAINT pk_order_item  PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_oi_product  FOREIGN KEY (product_id) REFERENCES PRODUCT (product_id)
);
```

### 第 3 正規形 (3NF)

**ルール:**
- 2NF であること。
- 推移的従属がないこと：非キー属性は他の非キー属性に従属してはならない。

**違反例:**

```
EMPLOYEE(employee_id, name, department_id, department_name, department_location)
-- department_name と department_location は employee_id ではなく department_id に従属している
```

**3NF での解決:** `DEPARTMENT` を別のテーブルとして抽出する（既出の例を参照）。

### ボイス・コッド正規形 (BCNF)

3NF のより厳格なバージョン。すべての決定子が候補キーである必要がある。テーブルに複数の重複する候補キーがある場合に違反が発生する。

```sql
-- 違反例: TEACHING(student, subject, teacher)
-- 1人の講師は1つの科目を教えるが、1人の生徒は複数の講師を持つことができる場合
-- 決定子: (student, subject) -> teacher および (student, teacher) -> subject
-- teacher -> subject (teacher は候補キーではない)

-- 解決策: 2つのテーブルに分割
CREATE TABLE TEACHER_SUBJECT (
    teacher_id  NUMBER(10) NOT NULL,
    subject_id  NUMBER(10) NOT NULL,
    CONSTRAINT pk_teacher_subj PRIMARY KEY (teacher_id),  -- 各講師は1つの科目
    CONSTRAINT fk_ts_subject   FOREIGN KEY (subject_id) REFERENCES SUBJECT (subject_id)
);

CREATE TABLE STUDENT_TEACHER (
    student_id  NUMBER(10) NOT NULL,
    teacher_id  NUMBER(10) NOT NULL,
    CONSTRAINT pk_student_teacher PRIMARY KEY (student_id, teacher_id)
);
```

### 第 4 正規形 (4NF)

**ルール:**
- BCNF であること。
- 多値従属がないこと（1つのエンティティに関して、2つ以上の独立した多値の事実を同一テーブルに持たない）。

**違反例:**

```
EMPLOYEE_SKILLS_LANGUAGES(employee_id, skill, language)
-- スキルと使用言語は、従業員に関する独立した多値の事実である
```

**4NF での解決:**

```sql
CREATE TABLE EMPLOYEE_SKILL (
    employee_id  NUMBER(10)   NOT NULL,
    skill        VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_emp_skill PRIMARY KEY (employee_id, skill)
);

CREATE TABLE EMPLOYEE_LANGUAGE (
    employee_id  NUMBER(10)   NOT NULL,
    language     VARCHAR2(100) NOT NULL,
    CONSTRAINT pk_emp_lang PRIMARY KEY (employee_id, language)
);
```

### 第 5 正規形 (5NF)

**ルール:**
- 4NF であること。
- 候補キーによって暗示されない結合従属性がないこと（より小さなテーブルに無損失分解することで、元のテーブルの意味をより適切に表現できることがないこと）。

5NF は主に理論的なものであり、実務で遭遇することは稀である。ある事実が 3 つ以上のエンティティの同時組み合わせによってのみ表現できる場合に適用される。

---

## 4. Oracle の命名規則

Oracle には、ERD から DDL への変換時に遵守すべき厳格な命名規則と予約語がある。

### ハード・ルール (Oracle による強制)

- オブジェクト名: **1～128 バイト** (Oracle 12.2 以降)。以前のバージョンでは **1～30 バイト**。
- 文字で始まる必要がある（引用符で囲まない限り）。
- 使用可能な文字: 文字、数字、`_`, `$`, `#`。
- 二重引用符 (`"`) で囲まない限り、大文字小文字は区別されない。
- **二重引用符での囲みは避ける**: オブジェクト名が常に引用符を必要とするようになり、大文字小文字が区別されてしまうため。

### 推奨される命名標準

| オブジェクト | 規則 | 例 |
|---|---|---|
| テーブル | 複数形の名詞、UPPER_SNAKE_CASE | `CUSTOMERS`, `ORDER_ITEMS` |
| 列 | 記述的名詞、UPPER_SNAKE_CASE | `CUSTOMER_ID`, `CREATED_AT` |
| 主キー | `PK_<table>` | `PK_CUSTOMERS` |
| 外部キー | `FK_<child>_<parent>` | `FK_ORDERS_CUSTOMERS` |
| 一意制約 | `UQ_<table>_<columns>` | `UQ_CUSTOMERS_EMAIL` |
| チェック制約 | `CK_<table>_<column>` | `CK_EMPLOYEES_SALARY` |
| 索引 (インデックス) | `IX_<table>_<columns>` | `IX_ORDERS_ORDER_DATE` |
| シーケンス | `SEQ_<table>` | `SEQ_CUSTOMERS` |
| ビュー | `V_<name>` または `VW_<name>` | `V_ACTIVE_ORDERS` |
| トリガー | `TRG_<table>_<timing>_<event>` | `TRG_ORDERS_BI_INSERT` |

### 識別子として避けるべき Oracle 予約語

以下は、テーブル名や列名として誤用されやすい主要な Oracle 予約語である。これらを（引用符なしで）使用してはいけない：

```
ACCESS      ADD         ALL         ALTER       AND
ANY         AS          ASC         AUDIT       BETWEEN
BY          CHAR        CHECK       CLUSTER     COLUMN
COMMENT     COMPRESS    CONNECT     CREATE      CURRENT
DATE        DECIMAL     DEFAULT     DELETE      DESC
DISTINCT    DROP        ELSE        EXCLUSIVE   EXISTS
FILE        FLOAT       FOR         FROM        GRANT
GROUP       HAVING      IDENTIFIED  IMMEDIATE   IN
INCREMENT   INDEX       INITIAL     INSERT      INTEGER
INTERSECT   INTO        IS          LEVEL       LIKE
LOCK        LONG        MAXEXTENTS  MINUS       MLSLABEL
MODE        MODIFY      NOAUDIT     NOCOMPRESS  NOT
NOWAIT      NULL        NUMBER      OF          OFFLINE
ON          ONLINE      OPTION      OR          ORDER
PCTFREE     PRIOR       PRIVILEGES  PUBLIC      RAW
RENAME      RESOURCE    REVOKE      ROW         ROWID
ROWNUM      ROWS        SELECT      SESSION     SET
SHARE       SIZE        SMALLINT    START       SUCCESSFUL
SYNONYM     SYSDATE     TABLE       THEN        TO
TRIGGER     UID         UNION       UNIQUE      UPDATE
USER        VALIDATE    VALUES      VARCHAR     VARCHAR2
VIEW        WHENEVER    WHERE       WITH
```

**問題のある列名の例と修正案:**

```sql
-- 悪い例: 予約語を使用している
CREATE TABLE ORDERS (
    order_id  NUMBER,
    date      DATE,        -- DATE は予約語
    comment   VARCHAR2(500) -- COMMENT は予約語
);

-- 良い例: 記述的な代替名
CREATE TABLE ORDERS (
    order_id      NUMBER,
    order_date    DATE,
    order_comment VARCHAR2(500)
);
```

---

## 5. Oracle 固有の ERD 考慮事項

### サロゲート・キー vs ナチュラル・キー

Oracle は、**シーケンス**および **Identity 列 (ID 列)** (12c 以降) を通じてサロゲート主キー（代替キー）を強力にサポートしている。

```sql
-- Oracle 12c 以降の Identity 列 (新規開発に推奨)
CREATE TABLE CUSTOMER (
    customer_id   NUMBER        GENERATED ALWAYS AS IDENTITY,
    email         VARCHAR2(255) NOT NULL,
    created_at    TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT pk_customer  PRIMARY KEY (customer_id),
    CONSTRAINT uq_cust_email UNIQUE (email)
);

-- 12c 以前のシーケンス + トリガー・パターン
CREATE SEQUENCE SEQ_CUSTOMER START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;

CREATE OR REPLACE TRIGGER TRG_CUSTOMER_BI
BEFORE INSERT ON CUSTOMER
FOR EACH ROW
BEGIN
    IF :NEW.customer_id IS NULL THEN
        :NEW.customer_id := SEQ_CUSTOMER.NEXTVAL;
    END IF;
END;
/
```

### 制約の遅延可能性 (Deferrable Constraints)

Oracle は**遅延可能制約**をサポートしている。これは、循環的な外部キーや一括ロード操作を扱う際に重要になる。

```sql
-- 遅延可能外部キー — 親子の挿入順序が不確かなバッチ挿入に有用
ALTER TABLE ORDER_ITEM
    ADD CONSTRAINT fk_oi_order
        FOREIGN KEY (order_id) REFERENCES ORDERS (order_id)
        DEFERRABLE INITIALLY DEFERRED;
```

### 誘導属性としての仮想列

計算値を保存する代わりに、Oracle の仮想列 (Virtual column) を使用することで、アプリケーション・ロジックなしでデータの一貫性を保つことができる。

```sql
CREATE TABLE PRODUCT (
    product_id    NUMBER(10)   NOT NULL,
    unit_price    NUMBER(12,2) NOT NULL,
    tax_rate      NUMBER(5,4)  NOT NULL,
    price_with_tax NUMBER(12,2) GENERATED ALWAYS AS (unit_price * (1 + tax_rate)) VIRTUAL,
    CONSTRAINT pk_product PRIMARY KEY (product_id)
);
```

### 不可視列 (Invisible Columns: 12c 以降)

スキーマ移行中に有用。既存の `SELECT *` クエリを壊すことなく新しい列を追加できる。

```sql
ALTER TABLE CUSTOMER ADD (
    legacy_system_id VARCHAR2(50) INVISIBLE
);
```

---

## 6. ベスト・プラクティス

- **すべてのテーブルに主キーを定義すること。** Oracle は自動的に一意索引を作成する。
- **すべての制約に明示的な名前を付けること。** 名前を付けない制約にはシステム生成名（例：`SYS_C001234`）が付けられ、保守、エラー・メッセージの理解、および移行が極めて困難になる。
- **必須属性にはデータベース・レベルで NOT NULL を適用すること。** アプリケーション・レイヤーだけに頼ってはいけない。
- **日付のみのデータには `DATE` を、タイムゾーンをまたぐ日時データには `TIMESTAMP WITH TIME ZONE` を使用すること。** 日付を `VARCHAR2` として保存してはいけない。
- **任意のリレーションシップを慎重にモデリングすること。** Null 許容の外部キーは適切だが、必須のリレーションシップでは外部キー列に `NOT NULL` を適用すべきである。
- **関連表（中間表）はシンプルに保つこと。** 関連表の目的は M:N の解消である。リレーションシップに付随するビジネス属性（日付、数量、ステータスなど）を含めるのは適切だが、過剰な負荷をかけないようにすること。
- **ビジネス・コンテキストとともに ERD を文書化すること。** Oracle の列コメントはスキーマの一部であり、将来の開発者にとって非常に貴重である。

```sql
COMMENT ON TABLE  CUSTOMER IS 'アカウント作成を完了した登録済み顧客';
COMMENT ON COLUMN CUSTOMER.customer_id IS 'Identity列によって生成されるサロゲート主キー';
COMMENT ON COLUMN CUSTOMER.email       IS '一意のログイン用メールアドレス。保存前に小文字化される。';
```

---

## 7. よくある間違いとその回避方法

### 間違い 1: 1 つの列に複数の値を格納する

```sql
-- 悪い例: カンマ区切り値 (CSV) を列に格納
CREATE TABLE PROJECT (
    project_id   NUMBER,
    team_members VARCHAR2(4000)  -- "101,102,103" — 検索も保守も不可能
);

-- 良い例: 正確な子テーブル（交差表）の作成
CREATE TABLE PROJECT_MEMBER (
    project_id   NUMBER NOT NULL,
    employee_id  NUMBER NOT NULL,
    CONSTRAINT pk_project_member PRIMARY KEY (project_id, employee_id)
);
```

### 間違い 2: ROWNUM または ROWID を主キーとして使用する

`ROWNUM` と `ROWID` は擬似列である。一括操作、パーティションの移動、テーブルの再編成などによって変化する。常に適切なサロゲート・キーまたはナチュラル・キーを使用すること。

### 間違い 3: 外部キー索引を忘れる

Oracle は**外部キー列に索引を自動作成しない**。これらの索引がないと、親行の削除時に子テーブルでロックのエスカレーションや全表スキャンが発生する。

```sql
-- 外部キー作成後、常に外部キー列に索引を追加すること
CREATE INDEX IX_ORDER_ITEMS_ORDER_ID ON ORDER_ITEMS (order_id);
CREATE INDEX IX_ORDER_ITEMS_PRODUCT_ID ON ORDER_ITEMS (product_id);
```

### 間違い 4: パフォーマンス重視の OLTP での過度な正規化

OLTP の目標は 3NF だが、数十個の小さなテーブルに過度に正規化すると、クエリ実行時の結合が増え、パフォーマンス低下の要因になる場合がある。3NF を超えて分割する前にパフォーマンステストを行うこと。

### 間違い 5: 一意制約における NULL セマンティクスを無視する

Oracle では、一意制約において `NULL` は**他の NULL も含め、すべての値と異なる**ものとして扱われる。そのため、一意制約のある列でも複数の行に `NULL` を入れることができる。「一意または NULL」の動作で明示的な NULL 管理が必要な場合は、一意のファンクション索引を使用すること。

```sql
-- 複数の NULL を許可しつつ、非 NULL 値の一意性を強制する
CREATE UNIQUE INDEX UX_EMP_NATIONAL_ID
    ON EMPLOYEE (CASE WHEN national_id IS NOT NULL THEN national_id END);
```

### 間違い 6: 固定フォーマットのコードに VARCHAR2 を使用する

ISO 国コードや州の略称など、真に固定長のコードには `CHAR` を使用して、ストレージを節約し一貫した比較を保証する。

```sql
country_code  CHAR(2)      NOT NULL,  -- 'US', 'JP', 'GB'
status_code   CHAR(1)      NOT NULL   -- 'A'ctive, 'I'nactive, 'S'uspended
```

---

## 8. Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 19c と 26ai の両方を運用する場合は、正確な RU (Release Update) レベルでのデフォルト動作を検証すること。

| 機能 | 導入バージョン |
|---|---|
| Identity 列 | 12c (12.1) |
| 不可視列 (Invisible columns) | 12c (12.1) |
| インメモリー列ストア | 12c (12.1.0.2) |
| 多相表関数 (Polymorphic table functions) | 18c |
| 自動索引作成 | 19c |
| ブロックチェーン表 | 21c (19.10以降にバックポート) |
| オブジェクト名 128バイト | 12.2 |
| `WITH FUNCTION` インライン SQL 関数 | 12c (12.1) |

12c より前のシステムでは、Identity 列をシーケンス + BEFORE INSERT トリガーに置き換え、オブジェクト名は最大 30 文字に制限すること。

---

## ソース

- [Oracle Database 23ai SQL Language Reference — Database Object Names and Qualifiers](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/Database-Object-Names-and-Qualifiers.html)
- [Oracle Database 23ai Concepts — Tables and Table Clusters](https://docs.oracle.com/en/database/oracle/oracle-database/23/cncpt/tables-and-table-clusters.html)
- [Oracle Database 12c R1 — Identity Columns (oracle-base.com)](https://oracle-base.com/articles/12c/identity-columns-in-oracle-12cr1)
- [Oracle Database 12c R1 — Invisible Columns (oracle-base.com)](https://oracle-base.com/articles/12c/invisible-columns-12cr1)
- [Oracle Database 12c R1 — In-Memory Column Store (oracle-base.com)](https://oracle-base.com/articles/12c/in-memory-column-store-12cr1)
- [Oracle Database 12c R1 — WITH Clause Enhancements (oracle-base.com)](https://oracle-base.com/articles/12c/with-clause-enhancements-12cr1)
- [Oracle Database 18c — Polymorphic Table Functions (oracle-base.com)](https://oracle-base.com/articles/18c/polymorphic-table-functions-18c)
- [Oracle Database 19c — Automatic Indexing (oracle-base.com)](https://oracle-base.com/articles/19c/automatic-indexing-19c)
- [Oracle Database 21c — Blockchain Tables (oracle-base.com)](https://oracle-base.com/articles/21c/blockchain-tables-21c)
- [Oracle Database 11g R2 — Recursive Subquery Factoring (oracle-base.com)](https://oracle-base.com/articles/11g/recursive-subquery-factoring-11gr2)
