# Oracle DB におけるエディショニングによる再定義 (EBR)

## 概要

エディショニングによる再定義 (Edition-Based Redefinition: EBR) は、PL/SQL コード、ビュー、シノニムなどの変更を、ダウンタイムなしで本番稼働中のデータベースにデプロイするための Oracle の仕組みである。これにより、これらのオブジェクトの複数のバージョンを「**エディション (edition)**」と呼ばれる名前付きのコンテナ内でデータベース内に同時に共存させることができる。データベース・セッションは特定のエディションに関連付けられるため、古いアプリケーション・インスタンスと新しいアプリケーション・インスタンスが同じデータベースに対して同時に実行され、それぞれが自身のバージョンのコードを参照することができる。

EBR は Oracle 11g Release 2 で導入され、データベース層でのホット・ロールオーバー（ブルー/グリーン・デプロイメントやローリング・デプロイメント）を実現するための Oracle の標準的な手法である。単にパッケージを置き換えるよりもはるかに強力であり、後方互換性のあるビューの進化やエディション間でのデータ同期など、アプリケーション・スキーマのバージョン・ライフサイクル全体を適切に処理できる。

---

## コア・コンセプト

### エディション (Editions)

エディションは、エディション化可能なオブジェクトを格納する、名前付きのスキーマから独立したコンテナである。エディションは、デフォルトのエディション (`ORA$BASE`) をルートとするツリー構造を形成する。子エディションは親エディションからすべてのオブジェクトを継承する。子エディションで行われた変更は、そのエディションで実行されているセッションにおいて親のバージョンを上書きする。

```
ORA$BASE (ルート・エディション)
  └── V1 (初期の本番環境)
        └── V2 (デプロイ中の次期バージョン)
              └── V3 (その次のデプロイ)
```

```sql
-- データベース内のすべてのエディションを一覧表示
SELECT EDITION_NAME, PARENT_EDITION_NAME, USABLE
FROM   DBA_EDITIONS
ORDER BY EDITION_NAME;

-- セッションの現在のエディションを確認
SELECT SYS_CONTEXT('USERENV', 'CURRENT_EDITION_NAME') FROM DUAL;

-- データベースのデフォルトのエディションを確認
SELECT PROPERTY_VALUE FROM DATABASE_PROPERTIES
WHERE  PROPERTY_NAME = 'DEFAULT_EDITION';
```

### エディション化可能なオブジェクト・タイプ

すべてのデータベース・オブジェクトがエディション化できるわけではない。以下のオブジェクト・タイプのみが、エディション固有のバージョンを持つことができる。

| エディション化可能 | エディション化不可 |
|---|---|
| プロシージャ (PROCEDURE) | 表 (TABLE) |
| 関数 (FUNCTION) | 索引 (INDEX) |
| パッケージ (仕様部 + 本体) | シーケンス (SEQUENCE) |
| トリガー (TRIGGER) | マテリアライズド・ビュー (MATERIALIZED VIEW) |
| タイプ (仕様部 + 本体) | 権限 (GRANT) |
| ビュー (VIEW) | データベース・リンク (DATABASE LINK) |
| シノニム (SYNONYM) | |
| ライブラリ (LIBRARY) | |
| SQL 変換プロファイル | |

表、索引、シーケンスはすべてのエディションで共有される。これは設計上の基本原則である。EBR は「コード」のバージョン管理を行うものであり、「データ」のバージョン管理を行うものではない。

### スキーマでのエディションの有効化

```sql
-- EBR を使用する各スキーマに対してエディションを有効にする必要がある
-- ALTER USER 権限が必要
ALTER USER app_owner ENABLE EDITIONS;

-- 確認
SELECT USERNAME, EDITIONS_ENABLED
FROM   DBA_USERS
WHERE  USERNAME = 'APP_OWNER';
```

### エディションの作成と使用

```sql
-- 新しいエディションを作成 (CREATE EDITION システム権限が必要)
CREATE EDITION v2 AS CHILD OF v1;

-- データベースのデフォルト・エディションを設定 (新しいセッションはこのエディションを使用する)
ALTER DATABASE DEFAULT EDITION = v2;

-- 特定のセッションのエディションを設定
ALTER SESSION SET EDITION = v2;

-- エディションの使用権限をユーザーに付与
GRANT USE ON EDITION v2 TO app_user;
```

---

## エディショニング・ビュー (Editioning Views)

表はエディション化できないため、EBR ではコード（エディション化可能）とデータ（エディション化不可）の境界として**エディショニング・ビュー**を導入している。アプリケーション・コードはベースとなる表を直接クエリするのではなく、このエディショニング・ビューをクエリする。デプロイ中、ベースとなる表に新旧両方のスキーマ用のデータが含まれている間、新しいエディションでエディショニング・ビューを再定義して、異なる列レイアウトを公開することができる。

### エディショニング・ビューの作成

```sql
-- ベース・表には、現在および移行状態のすべての列が含まれる
CREATE TABLE CUSTOMERS_T (
    CUSTOMER_ID    NUMBER(18,0)     NOT NULL,
    -- 旧バージョンの列 (V1 で使用)
    FULL_NAME      VARCHAR2(200),
    -- 新バージョンの列 (V2 デプロイ用に追加)
    FIRST_NAME     VARCHAR2(100),
    LAST_NAME      VARCHAR2(100),
    EMAIL          VARCHAR2(320)    NOT NULL,
    STATUS_CODE    VARCHAR2(10)     DEFAULT 'ACTIVE' NOT NULL,
    CREATED_AT     TIMESTAMP        DEFAULT SYSTIMESTAMP NOT NULL,
    CONSTRAINT PK_CUSTOMERS_T PRIMARY KEY (CUSTOMER_ID)
);

-- V1 用エディショニング・ビュー: 旧バージョンの列構成を公開
-- (V1 エディションに接続して実行)
CREATE OR REPLACE EDITIONING VIEW CUSTOMERS AS
SELECT
    CUSTOMER_ID,
    FULL_NAME,
    EMAIL,
    STATUS_CODE,
    CREATED_AT
FROM CUSTOMERS_T;
```

```sql
-- V2 用エディショニング・ビュー: 新バージョンの列構成を公開
-- (V2 エディションに接続して実行)
ALTER SESSION SET EDITION = v2;

CREATE OR REPLACE EDITIONING VIEW CUSTOMERS AS
SELECT
    CUSTOMER_ID,
    FIRST_NAME,
    LAST_NAME,
    EMAIL,
    STATUS_CODE,
    CREATED_AT
FROM CUSTOMERS_T;
```

エディション `v1` のセッションには `FULL_NAME` のレイアウトが見え、エディション `v2` のセッションには `FIRST_NAME`, `LAST_NAME` のレイアウトが見える。両方のセッションが同じ物理的な `CUSTOMERS_T` 表をクエリしている。

---

## クロスエディション・トリガー (Crossedition Triggers)

両方のエディションが同じベース・表に書き込むため、異なる列レイアウト間でデータの整合性を保つ仕組みが必要である。**クロスエディション・トリガー**は、一方のエディションの列レイアウトへの書き込みをもう一方へ伝播させる。

### 順方向クロスエディション・トリガー (Forward Crossedition Trigger)

順方向クロスエディション・トリガーは旧エディションで発生し、変更を新しい列に伝播させる。これにより、旧エディションのセッションによって書き込まれたデータが、新エディションのセッションからも見えるようになる。

```sql
-- V1 エディションとして接続し、順方向トリガーを作成
ALTER SESSION SET EDITION = v1;

CREATE OR REPLACE TRIGGER TRG_CUST_FORWARD
BEFORE INSERT OR UPDATE ON CUSTOMERS_T
FOR EACH ROW
FORWARD CROSSEDITION
DISABLE
BEGIN
  -- 旧バージョンのコードが FULL_NAME に書き込んだ際、FIRST_NAME と LAST_NAME に分割する
  IF :NEW.FULL_NAME IS NOT NULL THEN
    :NEW.FIRST_NAME := REGEXP_SUBSTR(:NEW.FULL_NAME, '^\S+');
    :NEW.LAST_NAME  := REGEXP_SUBSTR(:NEW.FULL_NAME, '\S+$');
  END IF;
END;
/

-- V2 デプロイメントがトラフィックを受け入れる準備ができたらトリガーを有効化
ALTER TRIGGER TRG_CUST_FORWARD ENABLE;
```

### 逆方向クロスエディション・トリガー (Reverse Crossedition Trigger)

逆方向クロスエディション・トリガーは新エディションで発生し、変更を古い列に書き戻す。これにより、新エディションの稼働中も、並行して動いている旧エディションのセッションで整合性が保たれる。

```sql
-- V2 エディションとして接続し、逆方向トリガーを作成
ALTER SESSION SET EDITION = v2;

CREATE OR REPLACE TRIGGER TRG_CUST_REVERSE
BEFORE INSERT OR UPDATE ON CUSTOMERS_T
FOR EACH ROW
REVERSE CROSSEDITION
DISABLE
BEGIN
  -- 新バージョンのコードが FIRST_NAME と LAST_NAME に書き込んだ際、FULL_NAME を再構築する
  IF :NEW.FIRST_NAME IS NOT NULL OR :NEW.LAST_NAME IS NOT NULL THEN
    :NEW.FULL_NAME := TRIM(:NEW.FIRST_NAME || ' ' || :NEW.LAST_NAME);
  END IF;
END;
/

ALTER TRIGGER TRG_CUST_REVERSE ENABLE;
```

---

## ホット・ロールオーバー・デプロイメントのワークフロー

EBR を使用した無停止デプロイメントのプロセスは、以下の手順に従う。

### フェーズ 1: 新エディションの準備

```sql
-- 1. 新しいエディションを作成
CREATE EDITION v2 AS CHILD OF v1;

-- 2. ベース・表に新しい列を追加 (非破壊的な追加)
ALTER TABLE CUSTOMERS_T ADD (
    FIRST_NAME VARCHAR2(100),
    LAST_NAME  VARCHAR2(100)
);

-- 3. 新エディションに切り替えて新しいコードをデプロイ
ALTER SESSION SET EDITION = v2;

-- 4. 新しいエディショニング・ビューを作成 (V2 レイアウト)
CREATE OR REPLACE EDITIONING VIEW CUSTOMERS AS
SELECT CUSTOMER_ID, FIRST_NAME, LAST_NAME, EMAIL, STATUS_CODE, CREATED_AT
FROM   CUSTOMERS_T;

-- 5. 更新された PL/SQL パッケージを V2 エディションにデプロイ
CREATE OR REPLACE PACKAGE PKG_CUSTOMERS AS
  PROCEDURE create_customer(
    p_first_name IN VARCHAR2,
    p_last_name  IN VARCHAR2,
    p_email      IN VARCHAR2
  );
END PKG_CUSTOMERS;
/
```

### フェーズ 2: クロスエディション・トリガーの有効化

```sql
-- 旧エディション (V1) の書き込みを新エディションの列に伝播させる (順方向)
ALTER SESSION SET EDITION = v1;
ALTER TRIGGER TRG_CUST_FORWARD ENABLE;

-- 新エディション (V2) の書き込みを旧エディションの列に伝播させる (逆方向)
ALTER SESSION SET EDITION = v2;
ALTER TRIGGER TRG_CUST_REVERSE ENABLE;
```

### フェーズ 3: 既存データのバックフィル

```sql
-- トリガーが有効化される前に挿入された行に対して、新しい列のデータを埋める
ALTER SESSION SET EDITION = v2;

UPDATE CUSTOMERS_T
SET
    FIRST_NAME = REGEXP_SUBSTR(FULL_NAME, '^\S+'),
    LAST_NAME  = REGEXP_SUBSTR(FULL_NAME, '\S+$')
WHERE
    FULL_NAME IS NOT NULL
    AND (FIRST_NAME IS NULL OR LAST_NAME IS NULL);

COMMIT;
```

### フェーズ 4: トラフィックの切り替え

```sql
-- 新しいセッションのデフォルト・エディションとして V2 を設定
-- (V1 の既存セッションは中断されることなく継続される)
ALTER DATABASE DEFAULT EDITION = v2;
```

この時点で、新しいアプリケーション・インスタンスは V2 を使用して接続する。V1 で実行されている古いインスタンスも引き続き動作する。両方のインスタンスが同じデータを共有し、クロスエディション・トリガーが両方の列レイアウトを同期させる。

### フェーズ 5: 旧エディションの廃棄

V1 を使用しているすべてのアプリケーション・インスタンスが停止された後：

```sql
-- クロスエディション・トリガーを無効化 (不要になったため)
ALTER TRIGGER TRG_CUST_FORWARD DISABLE;
ALTER TRIGGER TRG_CUST_REVERSE DISABLE;

-- 旧バージョンの列を削除 (V1 が廃棄されたため)
ALTER TABLE CUSTOMERS_T DROP COLUMN FULL_NAME;

-- 旧エディションを削除 (セッションが残っているエディションやデフォルトのエディションは削除できない)
DROP EDITION v1 CASCADE;
```

---

## 運用上の EBR 管理

### エディションごとのオブジェクトの確認

```sql
-- 各オブジェクトがどのエディションに属しているか
SELECT OBJECT_NAME, OBJECT_TYPE, EDITION_NAME, STATUS
FROM   USER_OBJECTS_AE  -- AE = All Editions (すべてのエディション)
WHERE  OBJECT_TYPE IN ('PACKAGE', 'PACKAGE BODY', 'VIEW', 'PROCEDURE', 'FUNCTION')
ORDER BY OBJECT_NAME, EDITION_NAME;
```

### 接続文字列でのエディションの指定

```shell
# JDBC 接続文字列での指定
jdbc:oracle:thin:@//host:1521/service?oracle.jdbc.editionName=V2

# Python (oracledb) での指定
conn = oracledb.connect(
    user="app_user",
    password=password,
    dsn="host:1521/service",
    edition="V2"
)
```

---

## ユースケースと制限事項

### 最適なユースケース

- **ローリング・デプロイメント** — 旧バージョンを実行しつつ新しいバージョンをデプロイし、双方を同じデータベースに接続させる。
- **PL/SQL を駆使したアプリケーション** — パッケージ内にビジネス・ロジックが多く含まれる場合、パッケージを独立してバージョン管理できる EBR のメリットが最大化される。
- **複雑な列名の変更や型変更** — エディショニング・ビュー ＋ クロスエディション・トリガーのパターンにより、アプリケーションを停止させることなく名前変更が可能になる。

### 制限事項

- **表、索引、シーケンスはエディション化できない。** 構造的な変更には、依然として後方互換性のある設計（拡張/縮小）が必要になる。
- **デプロイメントの複雑さが増す。** すべてのオブジェクトを正しいエディションで作成する必要がある。デプロイ中にエディション・コンテキストを忘れると、意図しないエディションにオブジェクトが作成され、トラブルシューティングが困難になる。
- **コネクション・プール管理が重要。** コネクション・プールは正しいエディションを指定するように構成される必要がある。指定がない場合、データベースのデフォルト・エディションが使用される。

---

## ベスト・プラクティス

- **エディションは線形チェーンとしてモデル化する。** Oracle は分岐するエディション・ツリーをサポートしているが、線形のチェーン (v1 → v2 → v3) の方が理解しやすく、運用も容易である。
- **デプロイ・スクリプト内では常に明示的にエディション・コンテキストを設定する。** セッションのデフォルトに頼ってはいけない。スクリプトの冒頭で `ALTER SESSION SET EDITION = target_edition;` を実行し、確実に想定通りのエディションであることを確認する。
- **アクティブなエディションの数は少なく保つ（最大 2 ～ 3）。** 複数の古いエディションを保持し続けると、クロスエディション・トリガーの複雑さと検証コストが指数関数的に増大する。
- **エディションの作成とクリーンアップをパイプラインで自動化する。** DBA による手動の手順に頼らないようにする。

---

## よくある間違い

**間違い: オブジェクト作成前にエディション・コンテキストを設定し忘れる。**
`ALTER SESSION SET EDITION = v2` を発行せずに `CREATE OR REPLACE VIEW` を実行すると、現在のセッションのエディション（古い方かもしれない）にオブジェクトが作成されてしまう。

**間違い: ベース・表に対する通常のトリガーが「すべての」エディションで発生することを忘れる。**
通常の（クロスエディションではない）トリガーは、セッションのエディションに関係なく発生する。監査トリガーや不整合チェック・トリガーは、すべてのエディションからの DML を捕捉する。

**間違い: 旧エディションのセッションが残っている状態で列を削除する。**
V1 セッションがまだアクティブな間に `FULL_NAME` 列を削除すると、それらのセッションは即座にエラーで停止する。列削除の前に、古いエディションを使用しているセッションがないことを必ず確認すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。
- 19c と 26ai が混在する環境では、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database Development Guide 19c — Using Edition-Based Redefinition](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/editions.html)
- [Oracle Database SQL Language Reference 19c — CREATE EDITION](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-EDITION.html)
- [Oracle Database Reference 19c — DBA_EDITIONS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_EDITIONS.html)
