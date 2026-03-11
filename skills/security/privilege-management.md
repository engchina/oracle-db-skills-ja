# Oracle 権限管理 (Privilege Management)

## 概要

権限管理は、Oracle データベース・セキュリティの基盤である。Oracle は、ユーザーに対して操作の実行やオブジェクトへのアクセスに関する明示的な権利を付与する、任意アクセス制御 (DAC) モデルを使用している。権限管理を正しく行うことは、Oracle のデプロイメントにおいて最も影響力のあるセキュリティ制御である。過度に寛大な権限付与は、データベース侵害や内部脅威の大多数の根本原因となっている。

Oracle の権限は、主に 2 つのカテゴリに分けられる。**システム権限**（データベース内のどこでも特定の種類の操作を実行する権利）と、**オブジェクト権限**（特定のオブジェクトに対して特定の操作を実行する権利）である。どちらもユーザーに直接付与することも、管理を容易にするために**ロール**にまとめて付与することもできる。

---

## システム権限 (System Privileges)

システム権限は、データベース全体、または任意のスキーマ内のオブジェクトに影響を与えるアクションを実行する能力を付与する。Oracle には 200 以上のシステム権限がある。最も危険なのは `ANY` キーワードを含む権限である。

### 注意すべき危険なシステム権限

| 権限 | リスク |
|---|---|
| `DBA` ロール | データベースのほぼ完全な制御 |
| `SYSDBA` / `SYSOPER` | OS 認証をバイパスする管理者アクセス |
| `CREATE ANY TABLE` | 任意のスキーマにオブジェクトを作成 |
| `DROP ANY TABLE` | 任意のスキーマのデータを破壊 |
| `SELECT ANY TABLE` | データベース内の任意のデータを読み取り |
| `EXECUTE ANY PROCEDURE` | 任意のストアド・コードを実行 |
| `ALTER ANY TABLE` | 任意の表の構造を変更 |
| `GRANT ANY PRIVILEGE` | 誰に対しても権限を再付与 |
| `GRANT ANY ROLE` | 誰に対してもロールを割り当て |
| `BECOME USER` | 任意のユーザーになりすます |

### システム権限の付与

```sql
-- 特定のシステム権限をユーザーに付与
GRANT CREATE SESSION TO app_user;
GRANT CREATE TABLE TO app_user;
GRANT CREATE SEQUENCE TO app_user;

-- WITH ADMIN OPTION を付けて付与すると、付与されたユーザーがさらに他者へ再付与できるようになる
-- これは権限の無制限な拡散を招くため、慎重に使用すること
GRANT CREATE TABLE TO schema_owner WITH ADMIN OPTION;

-- ロールに付与
GRANT CREATE SESSION TO app_readonly_role;
```

### システム権限の取り消し

```sql
-- システム権限を取り消す
REVOKE CREATE TABLE FROM app_user;

-- 注: WITH ADMIN OPTION で付与されたシステム権限を取り消しても、
-- そのユーザーによって再付与された権限は連鎖的に取り消されない。
-- これは、オブジェクト権限の取り消し動作とは異なる。
REVOKE CREATE SESSION FROM schema_owner;
```

---

## オブジェクト権限 (Object Privileges)

オブジェクト権限は、表、ビュー、順序、プロシージャなどの特定のデータベース・オブジェクトに対してユーザーができることを制御する。システム権限よりもきめ細かく、常にこちらを優先して使用すべきである。

### 一般的なオブジェクト権限

```sql
-- 表/ビューの権限
GRANT SELECT ON hr.employees TO reporting_user;
GRANT INSERT ON hr.employees TO data_entry_user;
GRANT UPDATE (salary, job_id) ON hr.employees TO hr_manager;  -- 列レベルの付与
GRANT DELETE ON hr.employees TO hr_admin;
GRANT REFERENCES ON hr.departments TO app_user;  -- 外部キー制約に必要

-- 順序の権限
GRANT SELECT ON hr.emp_seq TO app_user;

-- プロシージャ/ファンクション/パッケージの権限
GRANT EXECUTE ON hr.process_payroll TO payroll_app;

-- すべてのユーザーに付与（機密オブジェクトには避けること）
GRANT SELECT ON hr.public_holidays TO PUBLIC;  -- 注意して使用！
```

### 列レベルの付与

Oracle では、UPDATE および REFERENCES 権限を列レベルで指定できる。これは最小権限の原則を実現するための強力なツールである。

```sql
-- 特定の列の更新のみを許可
GRANT UPDATE (phone_number, email) ON hr.employees TO help_desk_role;

-- 列レベルの付与を確認
SELECT grantee, owner, table_name, column_name, privilege
FROM dba_col_privs
WHERE owner = 'HR'
ORDER BY table_name, column_name;
```

---

## ロール (Roles)

ロールは、権限の管理を簡素化するために、権限を名前付きでまとめたものである。各ユーザーに個別に数十の権限を付与するのではなく、ロールを一括して付与する。ロールにはシステム権限とオブジェクト権限の両方を含めることができ、ロールを他のロールに付与することもできる。

### ロールの作成と管理

```sql
-- シンプルなロールの作成
CREATE ROLE app_read_role;
CREATE ROLE app_write_role;
CREATE ROLE app_admin_role;

-- パスワード保護されたロール（明示的に有効化する必要がある）
CREATE ROLE sensitive_data_role IDENTIFIED BY "R0leP@ssw0rd!";

-- ロールへの権限付与
GRANT SELECT ON orders.customers TO app_read_role;
GRANT SELECT ON orders.order_lines TO app_read_role;
GRANT SELECT ON orders.products TO app_read_role;

GRANT app_read_role TO app_write_role;  -- ロールの階層化
GRANT INSERT, UPDATE, DELETE ON orders.customers TO app_write_role;

-- ユーザーへのロール付与
GRANT app_read_role TO reporting_svc;
GRANT app_write_role TO webapp_svc;

-- WITH ADMIN OPTION を付けて付与（ロールの再付与を許可）
GRANT app_read_role TO team_lead WITH ADMIN OPTION;
```

### パスワード保護されたロールの有効化

```sql
-- アプリケーション・コードまたはセッション・セットアップ内
SET ROLE sensitive_data_role IDENTIFIED BY "R0leP@ssw0rd!";

-- 現在のセッションで特定のロール以外を無効化
SET ROLE ALL EXCEPT sensitive_data_role;

-- すべてのロールを再度有効化
SET ROLE ALL;
```

### 事前定義された Oracle ロール（細心の注意が必要）

```sql
-- DBA ロール: WITH ADMIN OPTION を含むほぼすべてのシステム権限を付与
-- アプリケーション・アカウントには絶対に付与しないこと
GRANT DBA TO scott;  -- BAD PRACTICE

-- CONNECT ロール: 12c 以降では CREATE SESSION のみを付与
-- RESOURCE ロール: CREATE TABLE, SEQUENCE, PROCEDURE などを付与
-- これらは開発用スキーマには許容されるが、アプリケーション・サービス・アカウントには不適切

-- 事前定義ロールの中身を確認する
SELECT privilege, admin_option
FROM dba_sys_privs
WHERE grantee = 'DBA'
ORDER BY privilege;
```

---

## 最小権限の原則 (Least Privilege Principle)

最小権限の原則とは、ユーザー、アプリケーション、またはプロセスが、その機能を実行するために必要な最小限のアクセス権のみを持ち、それ以上の権利は持たないようにすべきであるという考え方である。

### 最小権限を考慮したスキーマ設計

```sql
-- ステップ 1: スキーマ所有者とアプリケーション・ユーザーを分離
-- スキーマ所有者はオブジェクトを作成するが、本番環境で接続することはない
CREATE USER orders_schema IDENTIFIED BY "SchemaP@ss!" ACCOUNT LOCK;

-- ステップ 2: 不要な権限を持たないアプリケーション・サービス・アカウントを作成
CREATE USER orders_app IDENTIFIED BY "AppP@ss!"
  DEFAULT TABLESPACE users
  TEMPORARY TABLESPACE temp
  PROFILE app_profile;

-- ステップ 3: 必要なものだけを付与
GRANT CREATE SESSION TO orders_app;
GRANT SELECT, INSERT, UPDATE ON orders_schema.orders TO orders_app;
GRANT SELECT ON orders_schema.customers TO orders_app;
GRANT SELECT ON orders_schema.products TO orders_app;
GRANT EXECUTE ON orders_schema.process_order TO orders_app;

-- ステップ 4: 読み取り専用のレポート作成用アカウントを作成
CREATE USER orders_report IDENTIFIED BY "ReportP@ss!";
GRANT CREATE SESSION TO orders_report;
GRANT SELECT ON orders_schema.orders TO orders_report;
GRANT SELECT ON orders_schema.order_lines TO orders_report;
```

---

## DBMS_PRIVILEGE_CAPTURE による権限分析

Oracle 12c 以降には `DBMS_PRIVILEGE_CAPTURE` パッケージが含まれており、特定の期間内にユーザーが実際にどの権限を使用したかをキャプチャできる。これは、権限を適切なサイズに調整するための究極のツールである。

### 権限キャプチャの実行

```sql
-- ステップ 1: キャプチャ・ポリシーを作成
-- キャプチャ・タイプのオプション: G_DATABASE (すべて), G_ROLE, G_CONTEXT, G_ROLE_AND_CONTEXT
BEGIN
  DBMS_PRIVILEGE_CAPTURE.CREATE_CAPTURE(
    name        => 'app_privs_capture',
    description => 'Capture privileges used by the orders app',
    type        => DBMS_PRIVILEGE_CAPTURE.G_DATABASE
  );
END;
/

-- ステップ 2: キャプチャを有効化
EXEC DBMS_PRIVILEGE_CAPTURE.ENABLE_CAPTURE('app_privs_capture');

-- ステップ 3: アプリケーション・ワークロードを実行（すべてのコード・パスを実行させる）

-- ステップ 4: キャプチャを無効化
EXEC DBMS_PRIVILEGE_CAPTURE.DISABLE_CAPTURE('app_privs_capture');

-- ステップ 5: 分析結果を生成
EXEC DBMS_PRIVILEGE_CAPTURE.GENERATE_RESULT('app_privs_capture');

-- ステップ 6: 結果をクエリ
-- 使用された（WERE used）権限
SELECT username, sys_priv, object_owner, object_name, object_type
FROM dba_used_privs
WHERE capture = 'APP_PRIVS_CAPTURE'
ORDER BY username, sys_priv;

-- 使用されなかった（NOT used）権限（取り消し候補）
SELECT username, sys_priv, object_owner, object_name, object_type
FROM dba_unused_privs
WHERE capture = 'APP_PRIVS_CAPTURE'
ORDER BY username, sys_priv;

-- ステップ 7: クリーンアップ
EXEC DBMS_PRIVILEGE_CAPTURE.DROP_CAPTURE('app_privs_capture');
```

---

## 権限データ・ディクショナリの照会

### システム権限のクエリ

```sql
-- ユーザーに付与されているすべてのシステム権限 (直接付与分)
SELECT grantee, privilege, admin_option
FROM dba_sys_privs
WHERE grantee = 'ORDERS_APP'
ORDER BY privilege;

-- ロールに付与されているすべてのシステム権限
SELECT grantee, privilege, admin_option
FROM dba_sys_privs
WHERE grantee = 'APP_WRITE_ROLE'
ORDER BY privilege;

-- DBA ロールを持つすべてのユーザー（直接または間接）を特定
SELECT grantee, granted_role, admin_option, default_role
FROM dba_role_privs
WHERE granted_role = 'DBA'
ORDER BY grantee;

-- ロール経由を含め、ユーザーのすべてのシステム権限を再帰的に検索
WITH role_tree (role_or_user) AS (
  SELECT granted_role
  FROM dba_role_privs
  WHERE grantee = 'ORDERS_APP'
  UNION ALL
  SELECT rp.granted_role
  FROM dba_role_privs rp
  JOIN role_tree rt ON rp.grantee = rt.role_or_user
)
SELECT sp.grantee, sp.privilege, sp.admin_option
FROM dba_sys_privs sp
WHERE sp.grantee IN (
  SELECT role_or_user FROM role_tree
  UNION ALL SELECT 'ORDERS_APP' FROM dual
)
ORDER BY sp.grantee, sp.privilege;
```

### オブジェクト権限のクエリ

```sql
-- ユーザーに付与されているすべてのオブジェクト権限
SELECT owner, table_name, grantee, privilege, grantable, hierarchy
FROM dba_tab_privs
WHERE grantee = 'ORDERS_APP'
ORDER BY owner, table_name, privilege;

-- 特定の機密表に誰がアクセスできるかを確認
SELECT grantee, privilege, grantable
FROM dba_tab_privs
WHERE owner = 'HR' AND table_name = 'EMPLOYEES'
ORDER BY grantee;

-- 列レベルの権限
SELECT owner, table_name, column_name, grantee, privilege
FROM dba_col_privs
WHERE owner = 'HR'
ORDER BY table_name, column_name, grantee;

-- PUBLIC に付与されているものを検索 (高リスク)
SELECT owner, table_name, privilege, grantable
FROM dba_tab_privs
WHERE grantee = 'PUBLIC'
ORDER BY owner, table_name;

SELECT privilege, admin_option
FROM dba_sys_privs
WHERE grantee = 'PUBLIC'
ORDER BY privilege;
```

### ロール・メンバーシップのクエリ

```sql
-- ユーザーに付与されているロール
SELECT granted_role, admin_option, default_role
FROM dba_role_privs
WHERE grantee = 'ORDERS_APP'
ORDER BY granted_role;

-- ロールの全メンバー
SELECT grantee, admin_option, default_role
FROM dba_role_privs
WHERE granted_role = 'APP_WRITE_ROLE'
ORDER BY grantee;

-- ロール階層の確認
SELECT role, privilege, admin_option
FROM role_sys_privs
WHERE role = 'APP_WRITE_ROLE'
ORDER BY privilege;

SELECT role, owner, table_name, privilege, grantable
FROM role_tab_privs
WHERE role = 'APP_WRITE_ROLE'
ORDER BY owner, table_name;
```

---

## PUBLIC への付与の回避

`PUBLIC` に権限を付与すると、データベース内のすべてのユーザー（将来作成されるユーザーを含む）がその権限を利用できるようになる。これは、機密オブジェクトにとって非常に危険である。

```sql
-- 危険: アプリケーションの表に対して絶対に行わないこと
GRANT SELECT ON payroll.salary_data TO PUBLIC;
GRANT EXECUTE ON sys.utl_file TO PUBLIC;  -- ファイル・システムへのアクセスを許可してしまう

-- 現在の PUBLIC への付与状況を監査
SELECT owner, table_name, privilege
FROM dba_tab_privs
WHERE grantee = 'PUBLIC'
  AND owner NOT IN ('SYS', 'SYSTEM', 'XDB', 'APEX_PUBLIC_USER',
                    'FLOWS_FILES', 'CTXSYS', 'MDSYS', 'ORDSYS')
ORDER BY owner, table_name;

-- 過度な PUBLIC への付与を取り消す
REVOKE EXECUTE ON utl_file FROM PUBLIC;
REVOKE EXECUTE ON utl_http FROM PUBLIC;
REVOKE EXECUTE ON utl_tcp FROM PUBLIC;
REVOKE EXECUTE ON utl_smtp FROM PUBLIC;
REVOKE EXECUTE ON dbms_advisor FROM PUBLIC;
```

---

## ユーザー・アカウントのセキュリティ設定

```sql
-- アプリケーション・ユーザー向けのセキュアなプロファイルを作成
CREATE PROFILE app_profile LIMIT
  SESSIONS_PER_USER          5
  CPU_PER_SESSION            UNLIMITED
  CPU_PER_CALL               3000
  CONNECT_TIME               60
  IDLE_TIME                  15
  LOGICAL_READS_PER_SESSION  DEFAULT
  LOGICAL_READS_PER_CALL     1000000
  PRIVATE_SGA                15K
  FAILED_LOGIN_ATTEMPTS      5
  PASSWORD_LIFE_TIME         90
  PASSWORD_REUSE_TIME        365
  PASSWORD_REUSE_MAX         10
  PASSWORD_VERIFY_FUNCTION   ora12c_strong_verify_function
  PASSWORD_LOCK_TIME         1/24
  PASSWORD_GRACE_TIME        7;

-- プロファイルをユーザーに割り当て
ALTER USER app_user PROFILE app_profile;

-- 直接ログインすべきでないアカウントをロック
ALTER USER schema_owner ACCOUNT LOCK;

-- デフォルト・パスワードのアカウントを確認 (Oracle 12c 以降)
SELECT username, account_status
FROM dba_users_with_defpwd
ORDER BY username;
```

---

## ベスト・プラクティス

1.  **スキーマ分離を使用する**: オブジェクトを所有するアカウントはロックし、アプリケーション接続には決して使用しない。アプリケーション用サービス・アカウントには、必要なオブジェクトに対する権限だけを付与する。

2.  **アプリケーション・アカウントに DBA を付与しない**: DBA ロールはデータベース内のほぼすべての権限を付与する。アプリケーション用サービス・アカウントがこれを持つことはあってはならない。

3.  **直接付与ではなくロールを使用する**: ロールを使用することで権限管理が監査可能になり、取り消しも容易になる。権限をロールに付与し、ロールをユーザーに付与する。

4.  **`WITH ADMIN OPTION` および `WITH GRANT OPTION` を避ける**: これらは管理者以外の制御が及ばない場所での権限拡散を許してしまう。どうしても必要な場合にのみ、DBA アカウントに対して付与する。

5.  **PUBLIC に付与しない**: PUBLIC に付与されたものは誰でも利用できる。一見無害に見えるパッケージでも、SQL インジェクションやデータの持ち出しに悪用される可能性がある。

6.  **メジャー・リリース前に DBMS_PRIVILEGE_CAPTURE を実行する**: ステージング環境で使用された権限をキャプチャし、本番環境のアカウントにそれらの権限のみが付与されていることを確認する。

7.  **DBA_SYS_PRIVS を定期的に監査する**: 新しいシステム権限の付与状況を週次または月次でレポートするジョブをスケジュールし、レビューを行う。

8.  **職務分掌の原則を使用する**: データを作成するユーザーと、データを削除またはエクスポートできるユーザーを分ける。

---

## よくある間違いとその回避方法

### 間違い 1: サービス・アカウントに RESOURCE ロールを付与する

`RESOURCE` ロールには `CREATE TABLE`, `CREATE PROCEDURE`, `CREATE SEQUENCE` などが含まれる。アプリケーション用サービス・アカウントが自らオブジェクトを作成すべきではない。

```sql
-- 不適切
GRANT RESOURCE TO webapp_svc;

-- 適切: 必要な特定のオブジェクト権限のみを付与
GRANT SELECT, INSERT, UPDATE ON app_schema.orders TO webapp_svc;
GRANT EXECUTE ON app_schema.order_pkg TO webapp_svc;
```

### 間違い 2: 特定の権限の代わりに `SELECT ANY TABLE` を使用する

```sql
-- 不適切: データベース全体のすべての表を読み取れてしまう
GRANT SELECT ANY TABLE TO reporting_user;

-- 適切: ビューを作成するか、特定の表に対する権限を付与する
GRANT SELECT ON sales.orders TO reporting_user;
GRANT SELECT ON sales.order_lines TO reporting_user;
-- さらに良い方法: 読み取り専用ロールを作成して付与
GRANT reporting_role TO reporting_user;
```

### 間違い 3: デフォルトで PUBLIC に付与されているパッケージを取り消し忘れる

```sql
-- 多くの危険なパッケージがデフォルトで PUBLIC に付与されている
-- これらを取り消し、必要なアカウントにのみ付与し直す
REVOKE EXECUTE ON sys.dbms_backup_restore FROM PUBLIC;
REVOKE EXECUTE ON sys.utl_file FROM PUBLIC;
GRANT EXECUTE ON sys.utl_file TO etl_process_account;
```

### 間違い 4: 権限付与の履歴を監査していない

権限付与に関する統合監査を有効にしていないと、誰が何を付与したかの記録が残らない。常に `GRANT` および `REVOKE` 文を監査対象にすること（`auditing.md` を参照）。

### 間違い 5: 期限なしに権限を付与する

Oracle にはネイティブな権限期限切れ機能はないが、以下のように実装できる。

```sql
-- 一時的な権限付与を自動的に取り消すジョブを作成
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'REVOKE_TEMP_ACCESS',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'REVOKE SELECT ON hr.employees FROM contractor_user;',
    start_date      => SYSTIMESTAMP + INTERVAL '30' DAY,
    enabled         => TRUE,
    comments        => 'Auto-revoke temporary contractor access'
  );
END;
/
```

---

## コンプライアンスの考慮事項

### SOX (サーベンス・オクスリー法)
-   開発者、DBA、ビジネス・ユーザー間の職務分掌を要求
-   財務データ表には文書化されたアクセス制御リストが必要
-   特権アクセスは少なくとも四半期ごとにレビューが必要
-   すべての DBA レベルのアクションは監査が必要

### PCI-DSS
-   要件 7: 会員データへのアクセスを、業務上知る必要のある範囲に制限する
-   要件 8: システム・コンポーネントへのアクセスを識別および認証する
-   会員データにアクセスするユーザーには、必要最小限の権限のみを付与する
-   アクセス制御リストは少なくとも 6 か月ごとにレビューが必要

### HIPAA
-   最小必要標準：ユーザーの役割に必要な PHI（個人健康情報）へのアクセスのみを許可
-   技術的保護手段により、承認されたユーザーのみが ePHI にアクセスできるようにする
-   アクセスは個人ごとに固有である必要がある（人間による共有サービス・アカウントの使用禁止）
-   PHI 表へのアクセスの監査ログを保持する必要がある

```sql
-- コンプライアンス・クエリ: 機密スキーマへの広範なアクセス権を持つユーザーを特定
SELECT DISTINCT grantee
FROM dba_tab_privs
WHERE owner IN ('HR', 'PAYROLL', 'FINANCE', 'HEALTH')
  AND grantee NOT IN (
    SELECT role FROM dba_roles  -- 結果からロール名を除外
  )
ORDER BY grantee;

-- 行レベル・セキュリティをバイパスできる可能性のあるシステム権限を持つユーザーを検索
SELECT grantee, privilege
FROM dba_sys_privs
WHERE privilege IN (
  'SELECT ANY TABLE',
  'INSERT ANY TABLE',
  'UPDATE ANY TABLE',
  'DELETE ANY TABLE',
  'EXEMPT ACCESS POLICY'  -- VPD をバイパスする - 極めて機密性が高い
)
ORDER BY grantee, privilege;
```

> **重要**: `EXEMPT ACCESS POLICY` 権限は、すべての Virtual Private Database (VPD) 行レベル・セキュリティ・ポリシーをバイパスする。特定の文書化された ETL または DBA アカウントを除き、いかなるユーザーにも付与すべきではなく、その使用状況は厳格に監査される必要がある。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

-   このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効である。
-   21c、23c、または 23ai と記された機能は、Oracle Database 26ai 対応機能として扱う。バージョンが混在する環境では、19c 互換の代替案を保持すること。
-   リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンをサポートする環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

## ソース

-   [Oracle Database 19c Security Guide (DBSEG)](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/)
-   [Oracle Database 19c SQL Language Reference — GRANT](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/GRANT.html)
-   [DBA_SYS_PRIVS — Oracle Database 19c Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_SYS_PRIVS.html)
-   [DBMS_PRIVILEGE_CAPTURE — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_PRIVILEGE_CAPTURE.html)
