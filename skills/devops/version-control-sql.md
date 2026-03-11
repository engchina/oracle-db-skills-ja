# Oracle SQL およびスキーマ・オブジェクトのバージョン管理

## 概要

Oracle スキーマをバージョン管理下に置くことは、単に DDL をバックアップすることではない。それは、データベース構造に関する「**信頼できる唯一の情報源 (Single Source of Truth: SSOT)**」を作成することであり、コード・レビュー、ロールバック、環境比較、自動デプロイ、および変更履歴の監査を可能にすることである。適切に運用されれば、スキーマ内のすべてのオブジェクトは Git 内に対応する定義ファイルを持ち、リポジトリからデータベースを完全に再構成できるようになる。

Oracle 特有の課題として、ツールで生成される DDL が冗長で環境依存しやすいこと、オブジェクト間の依存関係によりデプロイ順序が重要であること、権限 (Grant) やシノニムが見落とされがちであることなどが挙げられる。このガイドでは、DDL を正確に抽出し、Git 内で整理し、権限やシノニムをスクリプト化し、SQL Developer のソース管理機能と統合する方法について解説する。

---

## DBMS_METADATA による DDL の抽出

`DBMS_METADATA` は、データ・ディクショナリから DDL を生成するための Oracle 標準のパッケージである。Oracle が内部で使用している表現を直接読み取るため、サードパーティ製ツールのリバース・エンジニアリングよりもはるかに信頼性が高い。

### 基本的な使用方法

```sql
-- テーブルの DDL を抽出
SELECT DBMS_METADATA.GET_DDL('TABLE', 'ORDERS', 'APP_OWNER') FROM DUAL;

-- 索引の DDL を抽出
SELECT DBMS_METADATA.GET_DDL('INDEX', 'IDX_ORDERS_CUSTOMER', 'APP_OWNER') FROM DUAL;

-- パッケージ仕様部と本体の DDL 抽出
SELECT DBMS_METADATA.GET_DDL('PACKAGE',      'PKG_ORDERS', 'APP_OWNER') FROM DUAL;
SELECT DBMS_METADATA.GET_DDL('PACKAGE_BODY', 'PKG_ORDERS', 'APP_OWNER') FROM DUAL;

-- サポートされている主なオブジェクト・タイプ:
-- TABLE, INDEX, VIEW, SEQUENCE, PROCEDURE, FUNCTION,
-- PACKAGE, PACKAGE_BODY, TRIGGER, TYPE, TYPE_BODY,
-- SYNONYM, DB_LINK, CONSTRAINT, GRANT
```

### 出力フォーマットの構成 (トランスフォーム)

デフォルトの DDL 出力には、ストレージ句や表領域名など、バージョン管理において「ノイズ」となる環境依存の情報が含まれる。これらを除去してクリーンな DDL を取得する設定を行う：

```sql
BEGIN
  -- ストレージ句 (STORAGE (...)) を除外
  DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM, 'STORAGE', FALSE);

  -- 表領域の指定を除外
  DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM, 'TABLESPACE', FALSE);

  -- セグメント属性 (PCTFREE, INITRANS など) を除外
  DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM, 'SEGMENT_ATTRIBUTES', FALSE);

  -- 各文の最後にセミコロン（SQL ターミネータ）を付与
  DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM, 'SQLTERMINATOR', TRUE);

  -- 読みやすく整形（プリティ・プリント）
  DBMS_METADATA.SET_TRANSFORM_PARAM(DBMS_METADATA.SESSION_TRANSFORM, 'PRETTY', TRUE);
END;
/
```

### 権限とシノニムの抽出

```sql
-- スキーマ・オーナーによって付与されたオブジェクト権限をすべて抽出
SELECT DBMS_METADATA.GET_DDL('OBJECT_GRANT', OBJECT_NAME, GRANTOR)
FROM (
  SELECT DISTINCT OBJECT_NAME, GRANTOR
  FROM   DBA_TAB_PRIVS
  WHERE  GRANTOR = 'APP_OWNER'
);

-- 指定したスキーマを指すパブリック・シノニムを抽出
SELECT DBMS_METADATA.GET_DDL('SYNONYM', SYNONYM_NAME, 'PUBLIC')
FROM   DBA_SYNONYMS
WHERE  TABLE_OWNER = 'APP_OWNER' AND OWNER = 'PUBLIC';
```

---

## Git におけるスキーマ・オブジェクトの整理

### 推奨されるディレクトリ構造

```
schema/
  README.md
  install.sql               -- マスター・インストール・スクリプト (順序制御)
  tables/
    customers.sql
    orders.sql
  indexes/
    idx_orders_customer.sql
  sequences/
    seq_order_id.sql
  views/
    vw_order_summary.sql
  packages/
    pkg_orders.pks            -- パッケージ仕様部 (.pks 拡張子)
    pkg_orders.pkb            -- パッケージ本体 (.pkb 拡張子)
  synonyms/
    public_synonyms.sql
  grants/
    grants_to_app_user.sql
```

### マスター・インストール・スクリプトの作成

`install.sql` は、依存関係の順序に従ってスキーマ全体をゼロから再構築するためのスクリプトである。これは、新しい環境のプロビジョニングや CI で使用される。

```sql
-- schema/install.sql
WHENEVER SQLERROR EXIT FAILURE ROLLBACK

PROMPT シーケンスの作成...
@@sequences/seq_customer_id.sql
@@sequences/seq_order_id.sql

PROMPT テーブルの作成...
@@tables/customers.sql
@@tables/orders.sql

PROMPT 索引の作成...
@@indexes/idx_orders_customer.sql

PROMPT パッケージの作成...
@@packages/pkg_orders.pks
@@packages/pkg_orders.pkb

PROMPT 権限とシノニムの設定...
@@grants/grants_to_app_user.sql
@@synonyms/public_synonyms.sql

PROMPT スキーマのインストールが完了しました。
```

---

## SQL Developer のソース管理統合

Oracle SQL Developer は、Git および SVN とのネイティブな統合をサポートしている。

- **リポジトリへの接続:** 「チーム」メニューから Git プロジェクトをクローンし、ローカル・ワーキング・コピーを管理できる。
- **DDL のエクスポート設定:** 「ツール」>「プリファレンス」>「データベース」>「オブジェクト・ビューア」>「DDL」で、「スキーマを含める」のチェックを外し、「ターミネータを含める」をチェックすることで、バージョン管理に適したクリーンな DDL が生成されるようになる。
- **差分の比較:** 「データベースの差分 (Database Diff)」ツールを使用すると、開発環境と Git 上の定義、あるいは開発環境と本番環境を比較し、同期用のスクリプトを生成できる。

---

## スキーマの乖離 (Drift) の検出

データベースがリポジトリを介さずに直接変更された場合、環境間で「乖離 (Drift)」が発生する。これを定期的に検出することが重要である。

```sql
-- 前回デプロイ以降に変更されたオブジェクトを特定
SELECT
    OBJECT_NAME,
    OBJECT_TYPE,
    LAST_DDL_TIME
FROM
    DBA_OBJECTS
WHERE
    OWNER = 'APP_OWNER'
    AND LAST_DDL_TIME > TO_DATE('2025-01-01', 'YYYY-MM-DD')
ORDER BY
    LAST_DDL_TIME DESC;
```

---

## ベスト・プラクティス

- **1 オブジェクト 1 ファイル:** 1 つのファイルに複数のテーブルや索引をまとめない。ファイル単位で変更をレビューし、差分を管理できるようにする。
- **パッケージの仕様部と本体を分ける:** 仕様部 (Spec) の変更は依存オブジェクトへの影響が大きいが、本体 (Body) の変更は影響が少ない。これらを別ファイルにすることで、不要な再コンパイルを防ぎ、デプロイを最適化できる。
- **物理的な属性を除去してコミットする:** ストレージ句、PCTFREE、表領域名などは、環境（開発・テスト・本番）によって異なる。これらを `DBMS_METADATA` のトランスフォーム機能で除去し、論理的な構造のみをコミットする。
- **PR ごとに DDL の構文チェックを行う:** CI パイプラインで SQLcl や SQL*Plus を使用し、空のスキーマに対して DDL を適用させてみることで、構文エラーを早期に発見する。
- **権限とシノニムを忘れない:** テーブルやパッケージだけでなく、それらに対する `GRANT` や `SYNONYM` もリポジトリに含める。これらがないと、新しい環境を構築した際にアプリケーションが正常に動作しない。

---

## よくある間違い

**間違い: ストレージ句を含めたままコミットする。**
環境ごとに表領域名や初期サイズが異なると、エクスポートするたびにファイルが変更されたと見なされ、Git の履歴がノイズで埋まってしまう。

**間違い: DDL ファイルに「スキーマ名.オブジェクト名」を含める。**
`CREATE TABLE APP_OWNER.ORDERS` のように記述すると、別のユーザー名で構築したい場合にエラーになる。スキーマ修飾を外した DDL を作成し、接続セッションのコンテキストに任せる。

**間違い: パッケージの最新ソースを移行（マイグレーション）スクリプト内にしか持たない。**
`V10__update_logic.sql` のような変更スクリプトの中にしかコードがないと、現在の最新版パッケージがどうなっているかを確認するのが難しくなる。常に現在の「正」となるソースを `packages/` ディレクトリに保持すべきである。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [DBMS_METADATA (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_METADATA.html)
- [Oracle Database Reference 19c — DBA_OBJECTS](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_OBJECTS.html)
- [SQL Developer User's Guide — Version Control](https://docs.oracle.com/en/database/oracle/sql-developer/19.2/sduug/version-control.html)
