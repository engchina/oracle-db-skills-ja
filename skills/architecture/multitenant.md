# Oracleマルチテナント・アーキテクチャ: CDBおよびPDBの管理

## 概要

Oracle 12c で導入された Oracle Multitenant は、データの分離を維持しながら複数のデータベースを単一のデータベース・エンジンに統合するアーキテクチャ・パラダイムである。**コンテナ・データベース (CDB)** は外殻であり、1つの Oracle インスタンス、1セットのバックグラウンド・プロセス、および共有 SGA で構成される。その内部には **プラガブル・データベース (PDB)** が存在し、アプリケーションからは完全に独立した Oracle データベースとして見える。

マルチテナント・アーキテクチャは、従来の Oracle 統合における3つの歴史的な問題を解決する：
1. **リソースの浪費**: 50 個の独立したシングルテナント・データベースを実行すると、たとえデータベースが小さくても 50 個の SGA、50 セットのバックグラウンド・プロセス、50 セットのデータファイルが必要になる。
2. **パッチ適用の複雑さ**: 非 CDB 構成では、各データベースに独自のパッチ適用ライフサイクルが必要。CDB では、CDB にパッチを適用することで、すべての PDB に同時に適用できる。
3. **プロビジョニングの速度**: テンプレートから PDB をクローン作成するのは、数時間ではなく数秒で完了する。

---

## 1. CDB の構造とコンポーネント

### ルート・コンテナ (CDB$ROOT)

ルート・コンテナは、CDB の管理ハブである。以下が含まれる：
- Oracle 提供のメタデータ (データ・ディクショナリ、プロシージャ、パッケージ)
- 共通ユーザー (デフォルトでは接頭辞 `C##` が付く) および共通ロール
- 共通 UNDO 表領域 (12.2 以降の共有 UNDO モード時)
- CDB 自体の SYSTEM および SYSAUX 表領域

ルート・コンテナは、ユーザー・データのコンテナでは**ない**。CDB$ROOT にアプリケーション・スキーマを作成してはいけない。

### PDB$SEED

PDB$SEED は、読み取り専用のテンプレート PDB である。`CREATE PLUGGABLE DATABASE ... FROM` で新しい PDB を作成する場合、ソースを明示しない限り、デフォルトで PDB$SEED からクローン作成される。PDB$SEED は読み書きモードで開いたり変更したりすることはできない。

### ユーザー PDB

ユーザー PDB は、アプリケーション・データが格納されるテナント・コンテナである。各 PDB は以下を持つ：
- 独自の SYSTEM 表領域 (ローカル・データ・ディクショナリ拡張を含む)
- 独自の SYSAUX 表領域
- 独自の USERS (デフォルト) 表領域
- 独自の TEMP 表領域
- 独自の UNDO 表領域 (推奨されるローカル UNDO モード時)

### CDB アーキテクチャの確認クエリ

```sql
-- すべてのコンテナとそのステータスの表示
SELECT con_id, name, open_mode, restricted, con_uid
FROM   v$pdbs
ORDER  BY con_id;

-- 現在接続しているコンテナの確認
SELECT sys_context('USERENV', 'CON_NAME') AS current_container,
       sys_context('USERENV', 'CON_ID')   AS container_id
FROM   dual;

-- PDB への切り替え (CDB ルートから、DBA 権限が必要)
ALTER SESSION SET CONTAINER = pdb_name;

-- CDB ルートへの切り戻し
ALTER SESSION SET CONTAINER = CDB$ROOT;
```

---

## 2. PDB の作成と管理

### シードからの PDB 作成

```sql
-- 最もシンプルな形式: デフォルト設定で PDB$SEED から作成
CREATE PLUGGABLE DATABASE pdb_app1
    ADMIN USER pdb_admin IDENTIFIED BY "SecurePass#1"
    ROLES = (DBA)
    DEFAULT TABLESPACE users
        DATAFILE '/oradata/MYDB/pdb_app1/users01.dbf'
            SIZE 500M AUTOEXTEND ON NEXT 100M MAXSIZE 10G
    PATH_PREFIX = '/oradata/MYDB/pdb_app1/'
    FILE_NAME_CONVERT = ('/oradata/MYDB/pdbseed/', '/oradata/MYDB/pdb_app1/');

-- 新しい PDB を開く
ALTER PLUGGABLE DATABASE pdb_app1 OPEN;

-- CDB 起動時に自動で開くように設定
ALTER PLUGGABLE DATABASE pdb_app1 SAVE STATE;
```

### ストレージ制限付きの PDB 作成

```sql
CREATE PLUGGABLE DATABASE pdb_reporting
    ADMIN USER pdb_admin IDENTIFIED BY "ReportPass#2"
    STORAGE (MAXSIZE 50G MAXSHARED_TEMP_SIZE 5G)
    DEFAULT TABLESPACE app_data
        DATAFILE '/oradata/MYDB/pdb_reporting/app_data01.dbf'
            SIZE 1G AUTOEXTEND ON
    FILE_NAME_CONVERT = ('/oradata/MYDB/pdbseed/',
                         '/oradata/MYDB/pdb_reporting/');

ALTER PLUGGABLE DATABASE pdb_reporting OPEN;
ALTER PLUGGABLE DATABASE pdb_reporting SAVE STATE;
```

### PDB のオープンとクローズ

```sql
-- 単一の PDB を読み書きモードで開く
ALTER PLUGGABLE DATABASE pdb_app1 OPEN READ WRITE;

-- すべての PDB を一度に開く
ALTER PLUGGABLE DATABASE ALL OPEN;

-- 1つを除いてすべての PDB を開く
ALTER PLUGGABLE DATABASE ALL EXCEPT pdb_maintenance OPEN;

-- PDB を閉じる (デフォルトでは、アクティブなセッションがないことが必要)
ALTER PLUGGABLE DATABASE pdb_app1 CLOSE IMMEDIATE;

-- すべての PDB の現在のオープン・モードを確認
SELECT name, open_mode, restricted
FROM   v$pdbs
ORDER  BY name;
```

---

## 3. PDB のクローン作成

クローニング（クローン作成）は、最も強力な PDB 操作の1つである。PDB は、同じ CDB 内でのローカル・クローン、別の CDB からのリモート・クローン、またはソースを閉じずに実行するホット・クローンが可能である。

### ローカル・クローン (同一 CDB)

```sql
-- pdb_app1 を新しい PDB pdb_app1_test にクローン作成
-- コールド・クローンの場合、ソース PDB は READ ONLY モードである必要がある
ALTER PLUGGABLE DATABASE pdb_app1 CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE pdb_app1 OPEN READ ONLY;

CREATE PLUGGABLE DATABASE pdb_app1_test
    FROM pdb_app1
    FILE_NAME_CONVERT = ('/oradata/MYDB/pdb_app1/',
                         '/oradata/MYDB/pdb_app1_test/');

-- ソースを読み書きモードで再オープン
ALTER PLUGGABLE DATABASE pdb_app1 CLOSE IMMEDIATE;
ALTER PLUGGABLE DATABASE pdb_app1 OPEN READ WRITE;

-- クローンを開く
ALTER PLUGGABLE DATABASE pdb_app1_test OPEN;
```

### ホット・クローン (ソースのダウンタイムなし、12.2 以降)

```sql
-- ホット・クローンには、ソースがローカル UNDO モードであること、
-- および CDB で Archivelog が有効であることが必要

-- クローン作成中もソースは読み書きモードで開いたままにできる
CREATE PLUGGABLE DATABASE pdb_app1_hotclone
    FROM pdb_app1
    FILE_NAME_CONVERT = ('/oradata/MYDB/pdb_app1/',
                         '/oradata/MYDB/pdb_app1_hotclone/')
    SNAPSHOT COPY;  -- SNAPSHOT COPY は、即時クローンのためにスパース・ファイル (ASM/ACFS) を使用する
```

### リモート・クローン (別の CDB から)

```sql
-- ソース CDB を指すデータベース・リンクを作成
CREATE DATABASE LINK source_cdb_link
    CONNECT TO system IDENTIFIED BY "SourcePass#"
    USING 'SOURCE_CDB_TNSALIAS';

-- リモート PDB をクローン作成
CREATE PLUGGABLE DATABASE pdb_migrated
    FROM pdb_source@source_cdb_link
    FILE_NAME_CONVERT = ('/oradata/SOURCECDB/pdb_source/',
                         '/oradata/MYDB/pdb_migrated/');
```

---

## 4. PDB のプラグインとアンプラグ

アンプラグ・プラグインは、CDB 間での PDB の移行や PDB のアーカイブに使用されるメカニズムである。

### PDB のアンプラグ (切断)

```sql
-- ステップ 1: PDB を閉じる
ALTER PLUGGABLE DATABASE pdb_app1 CLOSE IMMEDIATE;

-- ステップ 2: XML マニフェスト・ファイルにアンプラグ
ALTER PLUGGABLE DATABASE pdb_app1
    UNPLUG INTO '/tmp/pdb_app1_manifest.xml';

-- ステップ 3: 現在の CDB から PDB を削除 (データファイルは保持)
DROP PLUGGABLE DATABASE pdb_app1 KEEP DATAFILES;
```

### 新しい CDB への PDB のプラグイン (接続)

```sql
-- ステップ 1: プラグイン前に互換性を検証
-- このチェックは、ターゲット CDB で SYSDBA として実行する必要がある
DECLARE
    l_result VARCHAR2(4000);
BEGIN
    l_result := DBMS_PDB.CHECK_PLUG_COMPATIBILITY(
        pdb_descr_file => '/tmp/pdb_app1_manifest.xml',
        pdb_name       => 'pdb_app1'
    );
END;
/

-- 互換性チェックの結果を確認
SELECT name, cause, type, message, status
FROM   PDB_PLUG_IN_VIOLATIONS
WHERE  name = 'PDB_APP1'
  AND  status = 'PENDING';

-- ステップ 2: PDB をプラグイン
CREATE PLUGGABLE DATABASE pdb_app1
    USING '/tmp/pdb_app1_manifest.xml'
    COPY                              -- COPY, NOCOPY, または MOVE
    FILE_NAME_CONVERT = ('/oradata/OLDCDB/pdb_app1/',
                         '/oradata/NEWCDB/pdb_app1/')
    STORAGE UNLIMITED
    TEMPFILE REUSE;

-- ステップ 3: PDB を開く (バージョンが異なる場合は catcon.pl の実行がトリガーされる場合がある)
ALTER PLUGGABLE DATABASE pdb_app1 OPEN
    UPGRADE;  -- ソース CDB のバージョンが低かった場合は UPGRADE を使用
```

---

## 5. PDB 間のリソース管理

CDB コンテキストにおける Oracle Database Resource Manager (DBRM) は、すべての PDB にわたる CPU と I/O の割り当てを管理し、1つのテナントがリソースを独占することを防ぐ。

### CDB リソース・プラン

```sql
-- CDB レベルのリソース・プランを作成
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();

    -- CDB プランの作成
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN(
        plan    => 'CDB_PLAN_PROD',
        comment => 'Production CDB resource plan'
    );

    -- 個々の PDB にシェア（比重）と制限を割り当てる
    -- shares: 相対的な CPU の重み (高いほど競合発生時に優先される)
    -- utilization_limit: 絶対的な CPU キャップ (CDB の全 CPU に対する %)
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN_DIRECTIVE(
        plan                  => 'CDB_PLAN_PROD',
        pluggable_database    => 'PDB_APP1',
        shares                => 4,
        utilization_limit     => 80,
        parallel_server_limit => 60
    );

    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN_DIRECTIVE(
        plan                  => 'CDB_PLAN_PROD',
        pluggable_database    => 'PDB_REPORTING',
        shares                => 2,
        utilization_limit     => 40,
        parallel_server_limit => 30
    );

    -- 他のすべての PDB 用のデフォルト・ディレクティブ
    DBMS_RESOURCE_MANAGER.CREATE_CDB_PLAN_DIRECTIVE(
        plan                  => 'CDB_PLAN_PROD',
        pluggable_database    => 'ORA$DEFAULT_PDB_DIRECTIVE',
        shares                => 1,
        utilization_limit     => 20,
        parallel_server_limit => 10
    );

    DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA();
    DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/

-- CDB プランの有効化
ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'CDB_PLAN_PROD' SCOPE = BOTH;
```

### PDB ごとのリソース・プラン

各 PDB 内で、その PDB 内のセッションにのみ適用される標準的な DBRM コンシューマ・グループ・プランを定義することもできる。

```sql
-- PDB に接続
ALTER SESSION SET CONTAINER = pdb_app1;

-- PDB 内で PDB レベルのリソース・プランを作成
BEGIN
    DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();

    DBMS_RESOURCE_MANAGER.CREATE_PLAN('PDB_APP1_PLAN', 'Internal PDB plan');

    DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('OLTP_GROUP',   'OLTP users');
    DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('REPORT_GROUP', 'Report users');

    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan           => 'PDB_APP1_PLAN',
        group_or_subplan => 'OLTP_GROUP',
        mgmt_p1        => 80
    );

    DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
        plan             => 'PDB_APP1_PLAN',
        group_or_subplan => 'REPORT_GROUP',
        mgmt_p2          => 20
    );

    DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/

ALTER SYSTEM SET RESOURCE_MANAGER_PLAN = 'PDB_APP1_PLAN' SCOPE = BOTH;
```

### PDB ごとのメモリー分離 (19c 以降)

```sql
-- PDB ごとに SGA および PGA のメモリー制限を設定 (19c マルチテナント・ライセンスが必要)
ALTER PLUGGABLE DATABASE pdb_app1
    SET SGA_TARGET = 8G SGA_MIN_SIZE = 4G;

ALTER PLUGGABLE DATABASE pdb_reporting
    SET SGA_TARGET = 4G;

-- メモリー割り当ての確認
SELECT con_id, name,
       sga_target / 1024 / 1024          AS sga_target_mb,
       sga_min_size / 1024 / 1024        AS sga_min_mb
FROM   v$pdbs
ORDER  BY con_id;
```

---

## 6. 共通ユーザーとローカル・ユーザー

CDB におけるユーザー・タイプは、マルチテナントで最も混乱を招きやすい要素の1つである。この区別を誤解すると、セキュリティや接続性の問題につながる。

### 共通ユーザー

共通ユーザーは CDB$ROOT で作成され、すべてのコンテナに（同じ ID で）存在する。Oracle によって強制される `C##` 接頭辞によって区別される。

```sql
-- SYSDBA として CDB$ROOT に接続
ALTER SESSION SET CONTAINER = CDB$ROOT;

-- 共通 DBA の作成
CREATE USER c##dba_admin IDENTIFIED BY "AdminPass#1"
    CONTAINER = ALL;

-- すべてのコンテナにわたる共通権限の付与
GRANT DBA TO c##dba_admin CONTAINER = ALL;

-- 特定の 1 つのコンテナ内でのみ権限を付与
GRANT READ ANY TABLE TO c##dba_admin CONTAINER = CURRENT;
```

### ローカル・ユーザー

ローカル・ユーザーは、ちょうど 1 つの PDB に存在し、他の PDB や CDB$ROOT から見ることも、そこから参照することもできない。

```sql
-- まずターゲット PDB に接続
ALTER SESSION SET CONTAINER = pdb_app1;

-- ローカル・ユーザーの作成 (C## 接頭辞は不要)
CREATE USER app_owner IDENTIFIED BY "AppPass#1";
GRANT CONNECT, RESOURCE TO app_owner;
GRANT UNLIMITED TABLESPACE TO app_owner;
```

### 共通ロールとローカル・ロール

ロールについても同様の区別がある。Oracle 提供のロール (DBA, SYSDBA など) は共通ロールである。アプリケーション固有のロールは、その PDB 内でローカル・ロールとして作成すべきである。

```sql
-- 共通ロール (CDB$ROOT で作成、どこでも利用可能)
ALTER SESSION SET CONTAINER = CDB$ROOT;
CREATE ROLE c##common_readonly CONTAINER = ALL;
GRANT SELECT ANY TABLE TO c##common_readonly CONTAINER = ALL;

-- ローカル・ロール (PDB 内部)
ALTER SESSION SET CONTAINER = pdb_app1;
CREATE ROLE app_readonly;
GRANT SELECT ON app_owner.orders TO app_readonly;
GRANT SELECT ON app_owner.customers TO app_readonly;
```

---

## 7. アプリケーション・コンテナ (19c)

アプリケーション・コンテナは、CDB の内側にある第2レベルの封じ込め（Containment）である。アプリケーション・コンテナは、一連の**アプリケーション PDB** のルートとして機能し、アプリケーション固有のマスター・データ (シード・データ、参照表、PL/SQL オブジェクトなど) をすべてのアプリケーション PDB 間で共有できるようにする。

```
CDB$ROOT
├── Application Container: APP_MASTER
│   ├── APP_SEED         (読み取り専用テンプレート)
│   ├── APP_PDB_TENANT1  (テナント・データ + 継承されたアプリ・オブジェクト)
│   ├── APP_PDB_TENANT2
│   └── APP_PDB_TENANT3
└── PDB_OTHER            (通常のPDB、アプリケーション・コンテナには含まれない)
```

### アプリケーション・コンテナの作成

```sql
-- ステップ 1: CDB$ROOT にアプリケーション・コンテナを作成
ALTER SESSION SET CONTAINER = CDB$ROOT;

CREATE PLUGGABLE DATABASE saas_app_root
    AS APPLICATION CONTAINER
    ADMIN USER saas_admin IDENTIFIED BY "SaasAdmin#1"
    FILE_NAME_CONVERT = ('/oradata/MYDB/pdbseed/',
                         '/oradata/MYDB/saas_app_root/');

ALTER PLUGGABLE DATABASE saas_app_root OPEN;

-- ステップ 2: アプリケーション・コンテナに接続してアプリケーションをインストール
ALTER SESSION SET CONTAINER = saas_app_root;

ALTER PLUGGABLE DATABASE APPLICATION saas_crm BEGIN INSTALL '1.0';

-- 共有アプリケーション・オブジェクトの作成 (すべての アプリ PDB から参照可能)
CREATE TABLE app_config (
    config_key   VARCHAR2(100) NOT NULL,
    config_value VARCHAR2(4000),
    CONSTRAINT pk_app_config PRIMARY KEY (config_key)
) SHARING = DATA;   -- DATA 共有: メタデータと行の両方を共有

CREATE TABLE product_catalog (
    product_id   NUMBER NOT NULL,
    product_name VARCHAR2(200) NOT NULL,
    CONSTRAINT pk_product PRIMARY KEY (product_id)
) SHARING = EXTENDED DATA;  -- EXTENDED DATA: 共有される行 + テナント独自の行を追加可能

ALTER PLUGGABLE DATABASE APPLICATION saas_crm END INSTALL '1.0';
```

### アプリケーション・コンテナにおける表共有オプション

| SHARING 句 | メタデータの共有 | データの共有 | テナントによる行の追加 |
|---|---|---|---|
| `METADATA` | はい | いいえ | はい (プライベートな行) |
| `DATA` | はい | はい | いいえ |
| `EXTENDED DATA` | はい | はい | はい (テナントからのみ見える独自の行) |
| `NONE` | いいえ | いいえ | N/A (テナント・ローカル表) |

---

## 8. ベスト・プラクティス

- **常にローカル UNDO モードを使用する (12.2 以降)。** 共有 UNDO (12.1 のデフォルト) では、すべての PDB のすべての UNDO が 1 つの UNDO 表領域に置かれるため、PDB レベルでのポイント・イン・タイム・リカバリが極めて困難になる。ローカル UNDO は各 PDB の UNDO を独自の UNDO 表領域に配置する。
- **自動オープンすべき PDB には `SAVE STATE` を使用する。** `SAVE STATE` なしでは、CDB インスタンスの再起動時に PDB は閉じられ、手動で開くまで閉じられたままになる。
- **早い段階で PDB にストレージ制限を設定する。** `STORAGE (MAXSIZE ...)` なしでは、1 つの暴走した PDB が共有ディスクをすべて消費し、他のすべての PDB に影響を及ぼす可能性がある。
- **初日から CDB レベルのリソース・プランを使用する。** リソース・プランがないと CPU 割り当ては早い者勝ちになり、重いバッチ PDB が OLTP PDB のリソースを奪う可能性がある。
- **パッチ適用は CDB レベルで行う。** 1 つの `opatch apply` ですべての PDB に一度にパッチが適用される。これは非 CDB 環境に対する大きな運用的メリットである。
- **CDB$ROOT をクリーンに保つ。** アプリケーション・データやアプリケーション・スキーマを置かないこと。CDB$ROOT には共通の管理オブジェクトのみを格納すべきである。
- **ライフステージごとに CDB を分ける。** DEV PDB は DEV CDB に、PROD PDB は PROD CDB に配置する。CDB レベルのパッチ・レベルやパラメータ設定は、含まれるすべての PDB に適用される。

---

## 9. よくある間違いとその回避方法

### 間違い 1: CDB$ROOT にアプリケーション・スキーマを作成する

CDB$ROOT はデフォルトで開いており SYSDBA として接続しやすいため、開発者がルート・コンテナにアプリケーション表を作成してしまうことがある。これらのオブジェクトは共有メタデータ・オブジェクトとなり、動作が予測不能でクリーンアップがほぼ不可能になる。

**対策:** アプリケーション・スキーマは指定された PDB の内部にのみ作成するというポリシーを徹底する。`PDB_LOCKDOWN` プロファイルを使用して、CDB$ROOT への非管理者接続を防止する。

### 間違い 2: 共通権限を付与する際に `CONTAINER = ALL` を使用し忘れる

CDB$ROOT にログインしているときに `CONTAINER = CURRENT` で権限を付与された共通ユーザーは、その権限を CDB$ROOT 内でのみ持ち、どの PDB でも持たない。これは、共通ユーザーが PDB に直接接続した際に「権限が付与されていない」というエラーが発生する一般的な原因である。

```sql
-- 誤り: 権限は CDB$ROOT にのみ適用される
GRANT DBA TO c##dba_admin;

-- 正解: 権限は現在および将来のすべてのコンテナに適用される
GRANT DBA TO c##dba_admin CONTAINER = ALL;
```

### 間違い 3: 各 PDB の内部で `LOCAL_LISTENER` の設定を忘れる

PDB が異なるリスナー (例: アプリケーションごとに異なるポート) に登録する必要がある場合、各 PDB は独自の `LOCAL_LISTENER` パラメータを設定する必要がある。

```sql
ALTER SESSION SET CONTAINER = pdb_app1;
ALTER SYSTEM SET LOCAL_LISTENER = '(ADDRESS=(PROTOCOL=TCP)(HOST=myhost)(PORT=1521))' SCOPE = BOTH;
```

### 間違い 4: PDB を指定せずに IMPDP/EXPDP を使用する

OS から実行すると、Data Pump はデフォルトで CDB ルートを対象とする。常に PDB サービス経由で接続するか、適切な接続文字列を指定すること。

```bash
# 誤り: CDB$ROOT にインポートされる
# impdp system/pwd directory=DATA_PUMP_DIR dumpfile=mydata.dmp

# 正解: 特定の PDB にインポートされる
# impdp system/pwd@//host:1521/pdb_app1 directory=DATA_PUMP_DIR dumpfile=mydata.dmp
```

### 間違い 5: 互換性レポートを確認せずに PDB をクローン作成する

`DBMS_PDB.CHECK_PLUG_COMPATIBILITY` を実行して違反を解決せずに、下位バージョンの CDB から上位バージョンの CDB に PDB をプラグインすると、オープンの失敗やエラーが発生する。新しくプラグインした PDB を開く前に、必ず `PDB_PLUG_IN_VIOLATIONS` を確認すること。

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Multitenant Administrator's Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/multi/) — CDB/PDB アーキテクチャ, PDB の作成と管理, クローニング, プラグイン/アンプラグ, 共通ユーザー, アプリケーション・コンテナ
- [DBMS_PDB (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_PDB.html) — CHECK_PLUG_COMPATIBILITY プロシージャ
- [DBMS_RESOURCE_MANAGER (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RESOURCE_MANAGER.html) — CREATE_CDB_PLAN, CREATE_CDB_PLAN_DIRECTIVE プロシージャ
