# SQLcl組み込みのLiquibase

## 概要

SQLclには、Liquibaseが標準で組み込まれています。個別のLiquibaseのインストール、Javaクラスパスの構成、またはOracle JDBCドライバのセットアップは必要ありません。SQLclにはすでにOracleドライバとLiquibaseが同梱され、統合されています。`lb` コマンド（`liquibase` としてもアクセス可能）は、SQLclプロンプトおよびコマンドラインから使用できます。

SQLclのLiquibase統合は、Oracle専用に構築されており、スタンドアロンのLiquibase CLIには標準で備わっていないOracle固有の拡張機能が含まれています。最も注目すべきは、**既存のOracleスキーマから変更ログ（Changelog）を生成**できる機能です。これにより、データベース・カタログから表、ビュー、パッケージ、トリガー、シーケンス、索引、シノニムなどのオブジェクト・タイプを直接キャプチャできます。

これにより、SQLcl Liquibaseは以下に最適です：
- 既存のデータベース・スキーマのベースラインをバージョン管理にキャプチャする
- 時間の経過に伴う段階的なスキーマ変更を追跡する
- 環境間（開発 → テスト → 本番）で変更をデプロイする
- データベースの変更をCI/CDパイプラインに統合する

---

## 主要な概念

### 変更ログ (Changelog)

変更ログは、順序付けられた **チェンジセット（Changeset）** のリストを含むファイル（XML, YAML, JSON, または SQL）です。Liquibaseは、`DATABASECHANGELOG` という管理表を使用して、どのチェンジセットがデータベースに適用済みかを追跡します。

### チェンジセット (Changeset)

チェンジセットは、`id`、`author`、および `filename` の一意の組み合わせによって識別される、変更のアトミックな単位です。一度適用されたチェンジセットは（明示的にロールバックされない限り）再適用されません。チェンジセットには、DDL、DML、またはストアド・プロシージャの定義を含めることができます。

### DATABASECHANGELOG 表

Liquibaseは、変更が適用されるスキーマにこの表を自動的に作成します。以下の内容を記録します。
- `ID` — チェンジセットID
- `AUTHOR` — チェンジセットの作成者
- `FILENAME` — 変更ログのファイル・パス
- `DATEEXECUTED` — 変更が適用された日時
- `MD5SUM` — チェンジセット内容のチェックサム
- `EXECTYPE` — EXECUTED, FAILED, RERAN など

### DATABASECHANGELOGLOCK 表

同時実行のLiquibase操作による競合を防ぐためのロック管理表です。

---

## 既存スキーマからの変更ログ（Changelog）の生成

これはSQLcl Liquibaseの最も強力な機能であり、既存のOracleスキーマをリバース・エンジニアリングしてLiquibaseの変更ログを生成できます。

### スキーマ全体の変更ログ生成

```sql
-- 最初に対象のスキーマに接続
CONNECT hr/hr@localhost:1521/FREEPDB1

-- スキーマ全体の変更ログを生成
lb generate-schema -split
```

`-split` フラグは、オブジェクトごとに1つのファイルを作成します（バージョン管理に推奨）。`-split` なしの場合は、すべてのDDLが1つのファイルに配置されます。

`-split` 使用時の出力ディレクトリ構造：

```
controller.xml          (すべてのサブ・ファイルをリファレンスするマスター・変更ログ)
table/
  employees.xml
  departments.xml
  ...
view/
  emp_details_view.xml
  ...
procedure/
  add_job_history.xml
  ...
index/
  ...
sequence/
  ...
trigger/
  ...
```

### 特定のオブジェクト・タイプの変更ログ生成

```sql
-- 表のみ
lb generate-schema -object-type table

-- 表とビューのみ
lb generate-schema -object-type table -object-type view

-- 特定のオブジェクト・タイプ（カンマ区切りはサポートされていません。複数のフラグを使用してください）
```

### 単一オブジェクトの変更ログ生成

```sql
-- 特定の表の変更ログを生成
lb generate-object -object-type table -object-name EMPLOYEES

-- ビューの変更ログを生成
lb generate-object -object-type view -object-name EMP_DETAILS_VIEW

-- パッケージの変更ログを生成
lb generate-object -object-type package -object-name MY_PKG

-- 出力ファイル名を指定
lb generate-object -object-type table -object-name EMPLOYEES -file employees_changelog.xml
```

### 出力フォーマット

SQLcl Liquibaseは、XML（デフォルト）、YAML、JSON、および SQL フォーマットをサポートしています。

```sql
-- YAML フォーマットで生成
lb generate-schema -format yaml

-- JSON フォーマットで生成
lb generate-schema -format json

-- SQL フォーマットで生成（LiquibaseのSQLチェンジセットでラップされたプレーンなDDL文）
lb generate-schema -format sql
```

---

## lb update による変更の適用

### すべての保留中のチェンジセットを適用

```sql
lb update -changelog-file controller.xml
```

Liquibaseは変更ログを読み取り、`DATABASECHANGELOG` をチェックしてすでに適用済みのチェンジセットを確認し、新しいものだけを実行します。

### 変更のプレビュー (updateSQL)

適用する前に、実行されるSQLを実際に実行せずに生成・確認します。

```sql
lb update-sql -changelog-file controller.xml
```

これはSQLを画面に出力します。`SPOOL` でファイルにリダイレクトできます。

```sql
SPOOL /tmp/pending_changes.sql
lb update-sql -changelog-file controller.xml
SPOOL OFF
```

### タグまで適用

```sql
-- 最初に現在の状態をタグ付け
lb tag -tag v1.2.0

-- 後で、特定のタグまでのみアップデート
lb update -changelog-file controller.xml -tag v1.2.0
```

### 指定した件数のチェンジセットを適用

```sql
lb update-count -count 3 -changelog-file controller.xml
```

---

## 変更のロールバック

### タグまでロールバック

```sql
lb rollback -tag v1.1.0 -changelog-file controller.xml
```

Liquibaseは、指定されたタグ以降に適用されたすべてのチェンジセットを元に戻します。自動的なロールバックを機能させるには、各チェンジセットに `<rollback>` 要素を含める必要があります（`ALTER TABLE` などの基本的なDDLの場合、Liquibaseがロールバックを自動生成できることもよくあります）。

### 指定した件数のチェンジセットをロールバック

```sql
lb rollback-count -count 2 -changelog-file controller.xml
```

### 日時までロールバック

```sql
lb rollback-to-date -date "2025-01-15 00:00:00" -changelog-file controller.xml
```

### ロールバックSQLのプレビュー

```sql
lb rollback-sql -tag v1.1.0 -changelog-file controller.xml
```

---

## チェンジセット（Changeset）のフォーマット例

### XML チェンジセット (DDL)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="001" author="jdoe">
        <createTable tableName="PRODUCT">
            <column name="PRODUCT_ID" type="NUMBER(10)">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="PRODUCT_NAME" type="VARCHAR2(100)">
                <constraints nullable="false"/>
            </column>
            <column name="PRICE" type="NUMBER(10,2)"/>
            <column name="CREATED_DATE" type="DATE" defaultValueComputed="SYSDATE"/>
        </createTable>
        <rollback>
            <dropTable tableName="PRODUCT"/>
        </rollback>
    </changeSet>

    <changeSet id="002" author="jdoe">
        <addColumn tableName="PRODUCT">
            <column name="CATEGORY_ID" type="NUMBER(5)"/>
        </addColumn>
        <rollback>
            <dropColumn tableName="PRODUCT" columnName="CATEGORY_ID"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

### XML チェンジセット (PL/SQL)

ストアド・プロシージャ、パッケージ、トリガーの場合は、`<sql>` を使用し、`splitStatements="false"` と `endDelimiter` を指定します。

```xml
<changeSet id="003" author="jdoe" runOnChange="true">
    <sql splitStatements="false" endDelimiter="/">
CREATE OR REPLACE PROCEDURE update_product_price (
    p_product_id IN NUMBER,
    p_new_price  IN NUMBER
) IS
BEGIN
    UPDATE product SET price = p_new_price WHERE product_id = p_product_id;
    COMMIT;
END update_product_price;
/
    </sql>
    <rollback>
        <sql>DROP PROCEDURE update_product_price;</sql>
    </rollback>
</changeSet>
```

### YAML チェンジセット

```yaml
databaseChangeLog:
  - changeSet:
      id: "001"
      author: jdoe
      changes:
        - createTable:
            tableName: PRODUCT
            columns:
              - column:
                  name: PRODUCT_ID
                  type: NUMBER(10)
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: PRODUCT_NAME
                  type: VARCHAR2(100)
                  constraints:
                    nullable: false
      rollback:
        - dropTable:
            tableName: PRODUCT
```

### SQL フォーマットのチェンジセット

```sql
--liquibase formatted sql

--changeset jdoe:001
CREATE TABLE product (
    product_id   NUMBER(10) PRIMARY KEY,
    product_name VARCHAR2(100) NOT NULL,
    price        NUMBER(10,2)
);

--rollback DROP TABLE product;

--changeset jdoe:002
ALTER TABLE product ADD (category_id NUMBER(5));

--rollback ALTER TABLE product DROP COLUMN category_id;
```

### マスター・変更ログ (include)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <!-- サブ変更ログを順番にインクルード -->
    <include file="table/product.xml"/>
    <include file="table/category.xml"/>
    <include file="view/product_catalog_v.xml"/>
    <include file="procedure/update_product_price.xml"/>

</databaseChangeLog>
```

---

## status および diff コマンド

```sql
-- 保留中の（まだ適用されていない）チェンジセットを表示
lb status -changelog-file controller.xml

-- 詳細なステータスを表示
lb status --verbose -changelog-file controller.xml

-- 変更ログとデータベースの差異を確認
lb diff -changelog-file controller.xml

-- 2つのデータベース間の差異の変更ログを生成
lb diff-changelog -reference-url jdbc:oracle:thin:hr/hr@prod:1521/PROD -changelog-file diff.xml
```

---

## トラッキングと履歴

```sql
-- DATABASECHANGELOG の内容を表示
SELECT id, author, filename, dateexecuted, exectype FROM databasechangelog ORDER BY dateexecuted;

-- 特定のチェンジセットが適用されたかを確認
SELECT * FROM databasechangelog WHERE id = '001' AND author = 'jdoe';

-- チェンジセットを実行済みとしてマーク（実際には実行しない）
lb changelog-sync -changelog-file controller.xml
```

---

## CI/CDとの統合

### GitHub Actions の例

```yaml
name: Deploy Database Changes

on:
  push:
    branches: [main]
    paths:
      - 'db/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install SQLcl
        run: |
          wget -q https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip
          unzip -q sqlcl-latest.zip -d /opt/sqlcl
          echo "/opt/sqlcl/sqlcl/bin" >> $GITHUB_PATH

      - name: Set up wallet
        run: |
          mkdir -p /tmp/wallet
          echo "${{ secrets.WALLET_ZIP_B64 }}" | base64 -d > /tmp/wallet.zip
          unzip /tmp/wallet.zip -d /tmp/wallet
        env:
          TNS_ADMIN: /tmp/wallet

      - name: Apply Liquibase Changes
        run: |
          cd db
          sql -S "${{ secrets.DB_USER }}/${{ secrets.DB_PASSWORD }}@${{ secrets.DB_SERVICE }}" <<'EOF'
          lb update -changelog-file controller.xml
          exit
          EOF
        env:
          TNS_ADMIN: /tmp/wallet
```

### GitLab CI の例

```yaml
deploy_db:
  stage: deploy
  image: container-registry.oracle.com/database/sqlcl:latest
  script:
    - cd db
    - |
      sql -S "${DB_USER}/${DB_PASSWORD}@${DB_SERVICE}" <<'EOF'
      lb update -changelog-file controller.xml
      exit
      EOF
  only:
    - main
```

### ヘッドレス・ワンライナー

```shell
echo "lb update -changelog-file controller.xml\nexit" | sql -S user/pass@service
```

またはラッパー・スクリプト（`deploy.sql`）を使用：

```sql
lb update -changelog-file controller.xml
exit
```

```shell
sql -S user/pass@service @deploy.sql
```

---

## スタンドアロンのLiquibase CLIとの違い

| 機能 | SQLcl `lb` | スタンドアロン Liquibase CLI |
|---|---|---|
| Oracle JDBC ドライバ | 同梱済み | ダウンロードと構成が必要 |
| `generate-schema` コマンド | 組み込み | 利用不可 (Pro機能) |
| `generate-object` コマンド | 組み込み | 利用不可 |
| インストール | SQLclの一部 | 個別インストール |
| Oracle Wallet サポート | ネイティブ | 手動のドライバ構成が必要 |
| PL/SQL サポート | ネイティブのOracleパース | 拡張機能なしでは標準SQLのみ |
| 変更ログのフォーマット | XML, YAML, JSON, SQL | XML, YAML, JSON, SQL |
| 主要なLiquibase操作 | 完全（update, rollback, diff, など） | 完全 |
| Liquibase Pro 機能 | 含まれていない | ライセンスがあれば利用可能 |

---

## ベスト・プラクティス

- スキーマの変更ログを生成する際は、`-split` を使用してください。単一ファイルの変更ログは管理が困難になり、バージョン管理で大規模なマージ競合を引き起こす原因になります。
- PL/SQLオブジェクト（パッケージ、プロシージャ、ファンクション、トリガー、ビュー）のチェンジセットには `runOnChange="true"` を使用してください。これにより、ソースが変更されたときにチェンジセットIDを変更することなく、Liquibaseがチェンジセットを再適用できるようになります。
- チェンジセットには常に `<rollback>` 要素を含めてください。Liquibaseは `createTable` や `addColumn` などの単純なDDLについてはロールバックSQLを自動生成できますが、任意のSQLやPL/SQLについては生成できません。
- 本番環境に変更を適用する前に、`lb tag` でデータベースの状態をタグ付けしてください。これにより、何か問題が発生した場合にクリーンなロールバック・ターゲットが確保されます。
- 変更ログをアプリケーション・コードと同じリポジトリに保存し、データベースの変更が依存するコードとともにバージョン管理されるようにしてください。
- `DATABASECHANGELOG` 表を手動で変更しないでください。チェンジセットを実行せずに適用済みとしてマークしたい場合（Liquibase採用前に手動作成されたオブジェクトなど）は、`lb changelog-sync` を使用してください。
- CI/CDパイプラインで `lb update` を実行する前に `lb status` を実行し、適用される予定のチェンジセットの数を確認してください。変更が期待されているのに件数がゼロの場合は、パイプラインを失敗させてください。

---

## よくある間違いと回避策

**間違い：すでに適用済みのチェンジセットを変更してしまう**
Liquibaseは各チェンジセットのチェックサムを保存しています。変更すると、次回の実行時にチェックサムの不一致で失敗します。PL/SQLオブジェクトの変更には `runOnChange="true"` を使用してください。データの修正には、新しいチェンジセットを作成してください。

**間違い：異なるファイルで同じチェンジセットIDを使用してしまう**
チェンジセットの一意性は `id + author + filename` の組み合わせで決まります。2つのチェンジセットはファイルが異なれば同じ `id` を共有できますが、混乱の原因になります。一貫したID命名規則（連番やタイムスタンプなど）を使用してください。

**間違い：クリーンではないスキーマで `lb generate-schema` を実行してしまう**
ターゲット・スキーマに変更ログにキャプチャされていない手動の変更がある場合、`lb update` はすでに存在するオブジェクトで失敗します。最初のベースライン生成後に `lb changelog-sync` を実行して、既存のすべてのオブジェクトを適用済みとしてマークしてください。

**間違い：`splitStatements="false"` なしでPL/SQLをインライン記述してしまう**
Liquibaseはデフォルトでセミコロン（;）をSQLの区切りとして扱います。PL/SQLの本体にはコード内にセミコロンが含まれています。PL/SQLを含むチェンジセットでは、常に `splitStatements="false"` を使用してください。

**間違い：`DATABASECHANGELOG` をバージョン管理にコミットしようとする**
`DATABASECHANGELOG` はデータベース内に存在し、バージョン管理するものではありません。コミットするのは変更ログ・ファイル（Changelogファイル）です。データベース内の表は、Liquibaseのランタイム状態です。

---

## 参考資料

- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLcl 19.2 での Liquibase の使用 — Oracle Docs](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/19.2/sqcug/using-liquibase-sqlcl.html)
- [SQLcl Liquibase によるデプロイメントの自動化 — ORACLE-BASE](https://oracle-base.com/articles/misc/sqlcl-automating-your-database-deployments-using-sqlcl-and-liquibase)
- [Oracle SQLcl リリース・インデックス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/index.html)
