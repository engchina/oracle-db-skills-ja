# Oracle ユーザー管理

## 概要

Oracle Databaseのユーザー管理は、セキュリティとアクセスの基盤である。すべてのセッションは特定のユーザーとして接続する必要があり、各ユーザーはオブジェクトに対する権限、リソース制限（プロファイル）、およびストレージ制限（割当て制限）を持つ。

Oracle 12c以降のマルチテナント・アーキテクチャでは、「共通ユーザー（Common Users）」と「ローカル・ユーザー（Local Users）」の区別を理解することが重要である。

---

## ユーザーの作成と管理

### 基本的なユーザーの作成

```sql
-- 基本的なユーザー作成 (パスワード認証)
CREATE USER app_user
  IDENTIFIED BY "Complex_Password_123"
  DEFAULT TABLESPACE users
  TEMPORARY TABLESPACE temp;

-- ユーザー情報の変更
ALTER USER app_user IDENTIFIED BY "New_Password_456";

-- ユーザーの削除 (スキーマ内のオブジェクトも削除する場合は CASCADE が必要)
DROP USER app_user CASCADE;
```

### マルチテナント環境におけるユーザー (CDB/PDB)

- **共通ユーザー (Common User):** CDB$ROOTおよびすべてのPDBに存在するユーザー。名前は `C##` または `c##` で始める必要がある。
- **ローカル・ユーザー (Local User):** 特定の1つのPDB内にのみ存在するユーザー。

```sql
-- 共通ユーザーの作成 (CDB$ROOTに接続して実行)
CREATE USER c##admin_user IDENTIFIED BY password CONTAINER=ALL;

-- ローカル・ユーザーの作成 (特定のPDBに接続して実行)
CREATE USER sales_user IDENTIFIED BY password CONTAINER=CURRENT;
```

---

## 認証方法

Oracleは複数のユーザー認証方法をサポートしている。

1. **パスワード認証:** データベース内にハッシュ化されたパスワードを保持する（最も一般的）。
2. **外部認証:** OSのユーザー名を使用してログインする。
3. **グローバル認証:** LDAPやActive Directoryなどの外部ディレクトリを使用する。
4. **プロキシ認証:** あるユーザーが別のユーザーとしてログインすることを許可する（スキーマ所有者へのアクセスを制限するのに有用）。

```sql
-- プロキシ認証の例: app_server ユーザーが app_owner として接続することを許可
ALTER USER app_owner GRANT CONNECT THROUGH app_server;
```

---

## ユーザー・プロファイル (リソース制限)

プロファイルは、パスワードポリシー（有効期限、複雑さ）やリソース制限（CPU時間、アイドル時間、セッション数）のセットである。

### プロファイルの作成と割り当て

```sql
-- パスワードポリシーの設定例
CREATE PROFILE app_profile LIMIT
  FAILED_LOGIN_ATTEMPTS 5      -- ログイン失敗5回でロック
  PASSWORD_LIFE_TIME 90        -- 90日でパスワード期限切れ
  PASSWORD_GRACE_TIME 7        -- 期限切れ後の猶予期間7日
  PASSWORD_REUSE_TIME 365      -- 1年間は同じパスワードを再利用不可
  IDLE_TIME 30                 -- 30分アイドルで接続切断
  SESSIONS_PER_USER 3;         -- 1ユーザーあたり最大3セッション

-- プロファイルをユーザーに割り当てる
ALTER USER app_user PROFILE app_profile;
```

### ユーザーのロックとロック解除

```sql
-- ユーザーを手動でロック
ALTER USER app_user ACCOUNT LOCK;

-- ユーザーのロックを解除
ALTER USER app_user ACCOUNT UNLOCK;
```

---

## 割当て制限 (Tablespace Quota)

ユーザーが特定の表領域にどれだけのデータを格納できるかを制御する。デフォルトでは、ユーザーは表領域に対して0バイトの権限しか持っていない。

```sql
-- 特定の表領域に 100MB の割当て制限を付与
ALTER USER app_user QUOTA 100M ON users;

-- 表領域に対して無制限の権限を付与
ALTER USER app_user QUOTA UNLIMITED ON users;

-- すべての表領域に対して無制限の権限を付与 (権限の付与が必要)
GRANT UNLIMITED TABLESPACE TO app_user;
```

---

## 権限とロール

ユーザーにアクションを実行させるには、**権限 (Privileges)** を付与する必要がある。

### システム権限

特定のデータベース操作を実行する権利（例: `CREATE TABLE`, `SELECT ANY TABLE`）。

```sql
GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW TO app_user;
```

### オブジェクト権限

特定のオブジェクト（テーブル、ビュー、パッケージなど）に対する権利。

```sql
GRANT SELECT, INSERT, UPDATE ON hr.employees TO app_user;
GRANT EXECUTE ON sys.dbms_crypto TO app_user;
```

### ロール (Roles)

権限のグループ化されたセット。個別の権限ではなくロールをユーザーに付与するのがベストプラクティスである。

```sql
-- ロールの作成
CREATE ROLE app_developer;

-- 権限をロールに付与
GRANT CREATE SESSION, CREATE TABLE, CREATE PROCEDURE TO app_developer;

-- ロールをユーザーに付与
GRANT app_developer TO app_user;

-- 標準的な組み込みロール
-- CONNECT: 接続に最低限必要な権限
-- RESOURCE: 一般的な開発者が必要な権限
-- DBA: すべてのシステム権限
```

---

## ユーザー情報の確認 (データ・ディクショナリ)

```sql
-- ユーザーの一覧とステータス、デフォルト表領域
SELECT username, account_status, default_tablespace, created
FROM dba_users
ORDER BY username;

-- ユーザーに割り当てられたプロファイル
SELECT username, profile FROM dba_users;

-- ユーザーの割当て制限
SELECT * FROM dba_ts_quotas WHERE username = 'APP_USER';

-- ユーザーに付与されたロール
SELECT * FROM dba_role_privs WHERE grantee = 'APP_USER';

-- ユーザーに付与された直接のシステム権限
SELECT * FROM dba_sys_privs WHERE grantee = 'APP_USER';

-- ユーザーへのオブジェクト権限
SELECT * FROM dba_tab_privs WHERE grantee = 'APP_USER';
```

---

## ベスト・プラクティス

1. **最小権限の原則 (Least Privilege):** ユーザーには業務に必要な最小限の権限のみを付与する。`DBA` ロールや `ANY` 系統の権限（例: `SELECT ANY TABLE`）の乱用を避ける。
2. **デフォルト表領域の設定:** `SYSTEM` 表領域にユーザー・データが書き込まれないよう、必ずデフォルト表領域を明示的に指定する。
3. **ロールによる管理:** 個々のユーザーに直接 `GRANT` するのではなく、役割に応じたロールを作成して管理する。
4. **パスワードプロファイルの使用:** パスワードの有効期限、失敗時のロック、複雑さの検証を強制する。
5. **スキーマ所有者とアプリケーション・ユーザーの分離:** オブジェクトを所有するユーザー（スキーマ）と、実際に処理を実行するユーザーを分けることで、不適切なDDL実行を防ぐ。
6. **デフォルトユーザーのロック:** `SYS`, `SYSTEM` 以外のインストール時に作成されるデフォルトユーザー（`SCOTT`, `HR` など）は、使用しない場合はロックし、パスワードを変更しておく。

---

## よくある間違い

- **`DEFAULT TABLESPACE` を指定しない:** 指定しない場合、12c以降では `USERS` になることが多いが、古い環境や設定によっては `SYSTEM` 表領域にデータが入ってしまい、システム全体のパフォーマンスや安定性に影響を及ぼす。
- **権限付与時の `WITH ADMIN OPTION` / `WITH GRANT OPTION` の無分別の付与:** 付与されたユーザーが他人にその権限をまた貸しできてしまうため、管理がコントロール不能になるリスクがある。
- **`PUBLIC` への権限付与:** `GRANT ... TO PUBLIC` を実行すると、全ユーザーに権限が付与されてしまう。セキュリティホールになりやすいため、避けるべきである。
- **プロキシ認証を使用せずにパスワードを共有する:** 複数の個人やアプリが一つのパスワードを共有すると、監査 (`AUDIT`) 時に「誰が操作したか」を特定できなくなる。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 23ai以降では、「スキーマ・レベルの権限（Schema Privileges）」により、スキーマ内のすべてのオブジェクトに対して一括で権限を付与できるようになり、管理が簡素化されている。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database セキュリティ・ガイド 19c — ユーザー権限の管理](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/managing-user-privileges.html)
- [Oracle Database 管理者ガイド 19c — ユーザーとリソースの管理](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-users-and-resources.html)
- [Oracle Database 23ai — User Management Enhancements](https://docs.oracle.com/en/database/oracle/oracle-database/23/nfcvw/user-management.html)
