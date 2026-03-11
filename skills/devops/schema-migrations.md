# Oracle DBのスキーマ・マイグレーション

## 概要

スキーマ・マイグレーションは、アプリケーション・コードとともにデータベース構造を進化させるための仕組みである。CI/CD パイプラインにおいて、表の作成、列の追加、索引の定義、制約の変更といったすべての DDL 変更は、追跡、バージョン管理され、反復可能かつ監査可能な方法でデプロイされなければならない。本番データベースに対して直接実行されるアドホックな DDL は、環境の乖離 (環境ドリフト)、デプロイの失敗、および停止を引き起こす主な原因の 1 つである。

Oracle の文字セット、豊富なデータ型、および手続き型拡張（PL/SQL、シーケンス、トリガー、パッケージ）は、MySQL 中心の手法では見落とされがちなマイグレーション特有の課題をもたらす。このガイドでは、Oracle 用に構成された Liquibase と Flyway、マイグレーション戦略、および最新の CI/CD パイプラインにおける DDL ライフサイクルについて説明する。

---

## Oracle での Liquibase

### コア・コンセプト

Liquibase は**チェンジログ (changelog)** を通じてマイグレーションを追跡する。これは個別の**チェンジセット (changeset)** を参照するマスター・ファイルである。各チェンジセットは `id` + `author` + `file` の 3 つの組み合わせで識別され、Liquibase が初回実行時に作成する `DATABASECHANGELOG` 表に記録される。また、`DATABASECHANGELOGLOCK` 表によって同時実行が防止される。

```
DATABASECHANGELOG
  ID           VARCHAR2(255)
  AUTHOR       VARCHAR2(255)
  FILENAME     VARCHAR2(255)
  DATEEXECUTED TIMESTAMP
  ORDEREXECUTED NUMBER
  EXECTYPE     VARCHAR2(10)   -- EXECUTED, FAILED, SKIPPED, RERAN, MARK_RAN
  MD5SUM       VARCHAR2(35)
  DESCRIPTION  VARCHAR2(255)
  COMMENTS     VARCHAR2(255)
  TAG          VARCHAR2(255)
  LIQUIBASE    VARCHAR2(20)
  CONTEXTS     VARCHAR2(255)
  LABELS       VARCHAR2(255)
  DEPLOYMENT_ID VARCHAR2(10)
```

### プロジェクト構造

適切に整理された Liquibase プロジェクトでは、オブジェクト・タイプごとに責務を分ける：

```
db/
  liquibase.properties
  changelog-root.xml
  changes/
    001-initial-schema.xml
    002-add-customer-status.xml
    003-orders-partitioning.xml
  procedures/
    pkg_orders_body.sql
    pkg_orders_spec.sql
  seeds/
    001-reference-data.xml
```

### liquibase.properties (Oracle 設定例)

```properties
# liquibase.properties
url=jdbc:oracle:thin:@//dbhost:1521/ORCLPDB1
username=APP_OWNER
password=${DB_PASSWORD}
driver=oracle.jdbc.OracleDriver
changeLogFile=db/changelog-root.xml
logLevel=INFO
defaultSchemaName=APP_OWNER
liquibaseCatalogName=APP_OWNER
```

Oracle Wallet を使用する場合（CI/CD で推奨）：

```properties
url=jdbc:oracle:thin:@mydb_high?TNS_ADMIN=/opt/wallet
username=${DB_USER}
password=${DB_PASSWORD}
```

### チェンジセットの例

```xml
<!-- db/changelog-root.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

  <include file="changes/001-initial-schema.xml" relativeToChangelogFile="true"/>
  <include file="changes/002-add-customer-status.xml" relativeToChangelogFile="true"/>
  <include file="changes/003-orders-partitioning.xml" relativeToChangelogFile="true"/>

</databaseChangeLog>
```

```xml
<!-- db/changes/002-add-customer-status.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

  <changeSet id="002" author="jane.smith" labels="customer,status" context="!test">
    <comment>Add STATUS column to CUSTOMERS with lookup table</comment>

    <!-- Oracle固有: BOOLEAN ではなく NUMBER(1) を使用 -->
    <addColumn tableName="CUSTOMERS">
      <column name="STATUS_CODE" type="VARCHAR2(10)" defaultValue="ACTIVE">
        <constraints nullable="false"/>
      </column>
    </addColumn>

    <addColumn tableName="CUSTOMERS">
      <column name="CREATED_AT" type="TIMESTAMP" defaultValueComputed="SYSTIMESTAMP">
        <constraints nullable="false"/>
      </column>
    </addColumn>

    <createTable tableName="CUSTOMER_STATUS_CODES">
      <column name="CODE"        type="VARCHAR2(10)"><constraints primaryKey="true" nullable="false"/></column>
      <column name="DESCRIPTION" type="VARCHAR2(100)"><constraints nullable="false"/></column>
      <column name="IS_ACTIVE"   type="NUMBER(1,0)" defaultValueNumeric="1"><constraints nullable="false"/></column>
    </createTable>

    <insert tableName="CUSTOMER_STATUS_CODES">
      <column name="CODE"        value="ACTIVE"/>
      <column name="DESCRIPTION" value="Active customer"/>
      <column name="IS_ACTIVE"   valueNumeric="1"/>
    </insert>

    <insert tableName="CUSTOMER_STATUS_CODES">
      <column name="CODE"        value="SUSPENDED"/>
      <column name="DESCRIPTION" value="Account suspended"/>
      <column name="IS_ACTIVE"   valueNumeric="1"/>
    </insert>

    <addForeignKeyConstraint
        baseTableName="CUSTOMERS"
        baseColumnNames="STATUS_CODE"
        referencedTableName="CUSTOMER_STATUS_CODES"
        referencedColumnNames="CODE"
        constraintName="FK_CUST_STATUS"/>

    <rollback>
      <dropForeignKeyConstraint baseTableName="CUSTOMERS" constraintName="FK_CUST_STATUS"/>
      <dropColumn tableName="CUSTOMERS" columnName="STATUS_CODE"/>
      <dropColumn tableName="CUSTOMERS" columnName="CREATED_AT"/>
      <dropTable tableName="CUSTOMER_STATUS_CODES"/>
    </rollback>
  </changeSet>

</databaseChangeLog>
```

### Liquibase における Oracle 固有のデータ型

Liquibase の汎用的なデータ型は Oracle にうまくマップされない場合がある。常に明示的な Oracle ネイティブ型を使用すること：

| 汎用型 (Liquibase) | Oracle での実態 | 代わりに使用すべき型 |
|---|---|---|
| `BOOLEAN` | Oracle には存在しない | `NUMBER(1,0)` + CHECK (col IN (0,1)) |
| `BIGINT` | `NUMBER(19,0)` にマップされる | `NUMBER(18,0)` または `NUMBER(19,0)` を明示 |
| `DATETIME` | `DATE` にマップされる (秒未満が欠落) | `TIMESTAMP` または `TIMESTAMP WITH TIME ZONE` |
| `TEXT` | `CLOB` にマップされる | 必要に応じて `VARCHAR2(4000)` または `CLOB` |
| `INT` | `NUMBER(10,0)` にマップされる | `NUMBER(10,0)` を明示 |

```xml
<!-- 明示的な Oracle 型の使用を推奨 -->
<column name="PRICE"      type="NUMBER(12,4)"/>
<column name="CREATED_AT" type="TIMESTAMP WITH LOCAL TIME ZONE"/>
<column name="NOTES"      type="CLOB"/>
<column name="IS_ENABLED" type="NUMBER(1,0)"/>
```

### Liquibase でのシーケンスとトリガー

Oracle 12c 未満ではアイデンティティ列が存在しなかったため、多くのコードベースでシーケンス＋トリガーのパターンが使用されている。

```xml
<!-- Oracle 12c+ アイデンティティ列 -->
<changeSet id="010" author="dev">
  <createTable tableName="ORDERS">
    <column name="ORDER_ID" type="NUMBER(18,0)" autoIncrement="true"
            generationType="ALWAYS">
      <constraints primaryKey="true" nullable="false"/>
    </column>
    <column name="ORDER_DATE" type="DATE"/>
  </createTable>
</changeSet>

<!-- 12c 未満: 明示的なシーケンス + トリガー -->
<changeSet id="010b" author="dev" dbms="oracle">
  <createSequence sequenceName="SEQ_ORDERS"
                  startValue="1"
                  incrementBy="1"
                  minValue="1"
                  maxValue="9999999999999999999"
                  cycle="false"
                  ordered="false"
                  cacheSize="20"/>

  <!-- 複雑な PL/SQL には sqlFile を使用 -->
  <sqlFile path="triggers/trg_orders_bi.sql"
           relativeToChangelogFile="false"
           splitStatements="false"
           endDelimiter="/"
           stripComments="false"/>

  <rollback>
    <sql>DROP TRIGGER TRG_ORDERS_BI</sql>
    <dropSequence sequenceName="SEQ_ORDERS"/>
  </rollback>
</changeSet>
```

```sql
-- triggers/trg_orders_bi.sql
CREATE OR REPLACE TRIGGER TRG_ORDERS_BI
BEFORE INSERT ON ORDERS
FOR EACH ROW
WHEN (NEW.ORDER_ID IS NULL)
BEGIN
  :NEW.ORDER_ID := SEQ_ORDERS.NEXTVAL;
END;
/
```

### Liquibase の実行コマンド

```shell
# 実行せずに変更内容をプレビュー (SQL を標準出力に生成)
liquibase updateSQL

# 保留中のチェンジセットを適用
liquibase update

# 特定のタグまでのチェンジセットのみを適用
liquibase updateToTag v2.3.0

# 現在の状態にタグを付ける
liquibase tag v2.3.0

# タグまでロールバック
liquibase rollback v2.3.0

# 指定した数のチェンジセットをロールバック
liquibase rollbackCount 3

# 現在のステータスを確認
liquibase status --verbose

# チェンジログの構文を検証
liquibase validate
```

---

## Oracle での Flyway

### Liquibase との主な違い

Flyway は、よりシンプルなファイル名主導のバージョン管理スキーマを使用する。マイグレーション・ファイルは、バージョン管理されたマイグレーションには `V{version}__{description}.sql`、繰り返し可能なマイグレーションには `R__{description}.sql` という名前を付ける。XML DSL はなく、すべて生の SQL または Java コールバックである。

```
db/migration/
  V1__initial_schema.sql
  V2__add_customer_status.sql
  V2.1__customer_status_fk.sql
  V3__orders_table.sql
  R__vw_active_customers.sql        -- 繰り返し可能 (ビュー)
  R__pkg_order_processing.sql       -- 繰り返し可能 (PL/SQL パッケージ)
```

### flyway.toml (Oracle 設定例)

```toml
[environments.default]
url = "jdbc:oracle:thin:@//dbhost:1521/ORCLPDB1"
user = "${DB_USER}"
password = "${DB_PASSWORD}"
schemas = ["APP_OWNER"]

[flyway]
locations = ["filesystem:db/migration"]
defaultSchema = "APP_OWNER"
validateOnMigrate = true
outOfOrder = false
# / で終わる PL/SQL ブロックに必要
sqlMigrationSuffixes = [".sql"]
placeholderReplacement = false
```

### Flyway での Oracle PL/SQL

Flyway のデフォルトの文区切り文字は `;` である。PL/SQL ブロックには終端文字として `/` が必要である。特別なコメント・アノテーションを使用して、マイグレーションごとにこれを構成する：

```sql
-- V4__create_order_package.sql
-- flyway:delimiter=/

CREATE OR REPLACE PACKAGE PKG_ORDERS AS
  PROCEDURE process_order(p_order_id IN NUMBER);
  FUNCTION  get_order_total(p_order_id IN NUMBER) RETURN NUMBER;
END PKG_ORDERS;
/

CREATE OR REPLACE PACKAGE BODY PKG_ORDERS AS

  PROCEDURE process_order(p_order_id IN NUMBER) IS
    v_status VARCHAR2(10);
  BEGIN
    SELECT STATUS_CODE INTO v_status
    FROM   ORDERS
    WHERE  ORDER_ID = p_order_id;

    IF v_status = 'PENDING' THEN
      UPDATE ORDERS
      SET    STATUS_CODE  = 'PROCESSING',
             PROCESSED_AT = SYSTIMESTAMP
      WHERE  ORDER_ID = p_order_id;
    END IF;

    COMMIT;
  END process_order;

  FUNCTION get_order_total(p_order_id IN NUMBER) RETURN NUMBER IS
    v_total NUMBER;
  BEGIN
    SELECT SUM(LINE_TOTAL)
    INTO   v_total
    FROM   ORDER_LINES
    WHERE  ORDER_ID = p_order_id;
    RETURN NVL(v_total, 0);
  END get_order_total;

END PKG_ORDERS;
/
```

### 繰り返し可能なマイグレーション (Repeatable Migrations)

繰り返し可能なマイグレーションは、チェックサムが変更されるたびに再実行される。これらは、増分変更（ALTER）ではなく、完全に入れ替えたいオブジェクト（ビュー、パッケージ、シノニム、権限付与など）に最適である。

```sql
-- R__vw_active_customers.sql
-- ファイルの内容が変更されるたびに再実行される

CREATE OR REPLACE VIEW VW_ACTIVE_CUSTOMERS AS
SELECT
    c.CUSTOMER_ID,
    c.FIRST_NAME,
    c.LAST_NAME,
    c.EMAIL,
    c.STATUS_CODE,
    cs.DESCRIPTION  AS STATUS_DESC,
    c.CREATED_AT
FROM
    CUSTOMERS       c
JOIN
    CUSTOMER_STATUS_CODES cs ON cs.CODE = c.STATUS_CODE
WHERE
    c.STATUS_CODE != 'CLOSED';
```

### Flyway の実行コマンド

```shell
# マイグレーション状況を確認
flyway info

# 保留中のマイグレーションを適用
flyway migrate

# チェックサムを検証 (適用済みマイグレーションへの手動編集を検知)
flyway validate

# チェックサムの不一致を修復 (壊れた環境のみで慎重に使用)
flyway repair

# 最後のバージョン管理されたマイグレーションを元に戻す (Flyway Teams が必要)
flyway undo

# スキーマをクリーンアップ (本番では絶対に使用しない — すべてのオブジェクトを削除)
flyway clean
```

---

## バージョン管理 vs 繰り返し可能なマイグレーション

| 項目 | バージョン管理 (V プレフィックス) | 繰り返し可能 (R プレフィックス) |
|---|---|---|
| 実行 | 1回のみ、バージョン順 | チェックサムの変更のたびに実行 |
| ロールバック | 明示的な undo マイグレーションが必要 | ファイルの内容を戻すだけ |
| ユースケース | 表/列の DDL、データ移行 | ビュー、パッケージ、シノニム、権限 |
| 順序 | 連続性、強制される | すべてのバージョン指定の後に実行 |
| 衝突 | 同じバージョンのファイルが2つあるとエラー | 再実行は等価 (idempotent) であるべき |

---

## CI/CD パイプラインにおける DDL 管理

### パイプライン設計例

```yaml
# .github/workflows/db-deploy.yml
name: Database Deploy

on:
  push:
    branches: [main]
    paths:
      - 'db/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate Liquibase Changelog
        run: |
          liquibase \
            --url="${{ secrets.DB_DEV_URL }}" \
            --username="${{ secrets.DB_USER }}" \
            --password="${{ secrets.DB_PASSWORD }}" \
            validate

  deploy-dev:
    needs: validate
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - name: Generate migration SQL (review artifact)
        run: |
          liquibase \
            --url="${{ secrets.DB_DEV_URL }}" \
            --username="${{ secrets.DB_USER }}" \
            --password="${{ secrets.DB_PASSWORD }}" \
            updateSQL > migration-dev.sql
      - name: Apply migrations to DEV
        run: |
          liquibase \
            --url="${{ secrets.DB_DEV_URL }}" \
            --username="${{ secrets.DB_USER }}" \
            --password="${{ secrets.DB_PASSWORD }}" \
            update

  deploy-prod:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Apply migrations to PROD
        run: |
          liquibase \
            --url="${{ secrets.DB_PROD_URL }}" \
            --username="${{ secrets.DB_USER }}" \
            --password="${{ secrets.DB_PASSWORD }}" \
            update
```

### ロックと並行性

Liquibase も Flyway も、マイグレーションの同時実行を防ぐためにロック表を使用する。デプロイが失敗した場合などに古いロックが残ることがある：

```sql
-- Liquibase: 古いロックを確認して解放する
SELECT * FROM DATABASECHANGELOGLOCK;

UPDATE DATABASECHANGELOGLOCK
SET    LOCKED      = 0,
       LOCKGRANTED = NULL,
       LOCKEDBY    = NULL
WHERE  ID = 1;

COMMIT;
```

---

## マイグレーションでのシーケンス管理

シーケンスは構成 DDL とは異なり、現在の値という「状態」を持つため、ロールバック時に注意が必要である。

```sql
-- Oracle のベストプラクティスに従ったシーケンス作成
CREATE SEQUENCE SEQ_INVOICE_ID
    START WITH     1
    INCREMENT BY   1
    MINVALUE       1
    MAXVALUE       9999999999999999999
    NOCYCLE
    CACHE          100    -- パフォーマンスと欠番のバランスを考慮して 100 をキャッシュ
    NOORDER;              -- RAC で厳密な順序が必要な場合以外は NOORDER
```

### シーケンスのリセット (開発環境など)

```sql
-- 18c 以降であれば RESTART が可能
ALTER SEQUENCE SEQ_INVOICE_ID RESTART START WITH 1;
```

---

## ベスト・プラクティス

- **適用済みのマイグレーション・ファイルを編集しない。** 修正が必要な場合は、新しい「修正用」マイグレーションを作成する。
- **DDL 所有者とアプリケーション・ユーザーを分ける。** マイグレーションはスキーマ所有者（または DBA 相当）のアカウントで実行する。アプリケーションの実行ユーザーには DML 権限のみを付与し、不慮の DDL 実行を防ぐ。
- **チェンジセットを小さく保つ。** 1 つのチェンジセットには 1 つの論理的な変更のみを含める。巨大なチェンジセットは診断が難しく、DDL ロックを長時間保持してしまう。
- **常にロールバック・ブロックを書く。** Liquibase は Oracle の一部の DDL 操作に対してロールバック SQL を自動生成しない。将来使う予定がなくても、ドキュメントとして `<rollback>` を記述すること。
- **コンテキストとラベルを使用して、実行環境を制御する。** シード・データやテスト用データ、パフォーマンス負荷の高い処理などは本番環境から除外する。
- **本番環境では `update` の前に `updateSQL` を実行する。** 生成された SQL ファイルは、DBA によるレビュー、監査ログ、および障害分析のためのアーティファクトとなる。

---

## よくある間違い

**間違い: `BOOLEAN` を列型として使用する。**
Oracle SQL には BOOLEAN 型が存在しない（PL/SQL にはある）。Liquibase は自動的に `NUMBER(1,0)` にマップするが、意図が不明確になる。明示的に `NUMBER(1,0)` と CHECK 制約を使用すること。

**間違い: PL/SQL で `splitStatements=true` を指定したままコミットする。**
Liquibase はデフォルトで `;` で文を分割するが、PL/SQL ブロックには多くのセミコロンが含まれる。PL/SQL ファイルでは常に `splitStatements="false"` と `endDelimiter="/"` を設定すること。

**間違い: バージョン管理されたマイグレーションに `GRANT` 文を含める。**
一度きりのマイグレーションで権限付与を行うと、ユーザーが再作成された際に権限が失われる。繰り返し可能なマイグレーションを使用するか、等価（idempotent）な権限付与スクリプトを別途用意すること。

**間違い: アプリケーションのデプロイと同時にマイグレーションを実行する。**
高可用性が求められる環境では、アプリと DB のバージョンが一時的にずれることがある。新しい列を null 許可で追加し、古いアプリが完全に停止してから不要な列を削除するなど、後方互換性を持たせた設計にすること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。
- 環境に 19c と 26ai が混在する場合、デフォルト設定や非推奨機能が異なる可能性があるため、両方のバージョンでテストを行うこと。

## ソース

- [Liquibase Documentation](https://docs.liquibase.com/) 
- [Flyway Documentation](https://documentation.red-gate.com/flyway)
- [Oracle Database SQL Language Reference 19c — CREATE SEQUENCE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-SEQUENCE.html)
- [Oracle Database Development Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/)
