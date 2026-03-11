# Oracle Database 監査 (Auditing)

## 概要

データベース監査は、「誰が、いつ、どこから、何をしたか」という不変の記録を作成する。これは、規制コンプライアンス、フォレンジック調査、および内部脅威の検出のための基本的な制御である。Oracle はいくつかの監査メカニズムを提供しているが、Oracle 12c 以降の主要で推奨されるアプローチは**統合監査 (Unified Auditing)** である。

統合監査は、以前の断片化された監査システム（標準監査、ファイングレイン監査、OS 監査、SYS 監査）を、単一の一貫したフレームワークに置き換える。すべてのソースからのすべての監査レコードは `UNIFIED_AUDIT_TRAIL` ビューに格納され、共通のコマンド・セットを通じて管理される。

### 統合監査 vs 従来の監査

| 機能 | 従来の監査 (12c 未満) | 統合監査 (12c 以降) |
|---|---|---|
| ストレージ | `AUD$` 表または OS ファイル | `AUDSYS` スキーマ (セキュア) |
| 構成 | 複数のパラメータ | `CREATE AUDIT POLICY` |
| ファイングレイン監査 | 個別の `DBMS_FGA` | 統合済み |
| SYS 監査 | 個別 | 統合済み |
| クエリ参照 | 複数のビュー | `UNIFIED_AUDIT_TRAIL` |
| 耐改ざん性 | 限定的 | 強化（個別スキーマ） |

### 監査モードの確認

```sql
-- 純粋統合監査がアクティブかどうかを確認
SELECT value FROM v$option WHERE parameter = 'Unified Auditing';
-- 純粋統合監査の場合は 'TRUE' が返される

-- 現在の監査証跡設定を確認（従来の監査）
SHOW PARAMETER audit_trail;
-- 純粋統合監査では、このパラメータは無視される

-- 有効になっている監査機能を確認
SELECT * FROM v$unified_audit_trail_exists;
```

---

## 純粋統合監査の有効化

デフォルトでは、Oracle 12c データベースは「混合モード」で動作し、従来の監査と統合監査の両方が共存している。純粋統合監査を使用するには、Oracle 実行可能ファイルの再リンクが必要である。

```bash
# データベース・サーバー OS 上で（oracle OS ユーザーとして実行）:
cd $ORACLE_HOME/rdbms/lib
make -f ins_rdbms.mk uniaud_on ioracle

# 再リンク後、データベースを再起動
sqlplus / as sysdba
SHUTDOWN IMMEDIATE;
STARTUP;
```

---

## 統合監査ポリシーの作成

監査ポリシーは、何を監査するかを定義する。これらは一度作成すれば、すべてのユーザーまたは特定のユーザーに対して有効にできる。

### 基本的なポリシー作成構文

```sql
CREATE AUDIT POLICY policy_name
  [PRIVILEGES privilege_list]
  [ACTIONS action_list]
  [ACTIONS COMPONENT = component action_list]
  [WHEN 'condition_expression' EVALUATE PER SESSION|INSTANCE]
  [CONTAINER = CURRENT|ALL];
```

### 権限使用の監査

```sql
-- 強力なシステム権限の使用を監査
CREATE AUDIT POLICY audit_dba_privs
  PRIVILEGES CREATE USER, DROP USER, ALTER USER,
             GRANT ANY PRIVILEGE, GRANT ANY ROLE,
             CREATE ANY TABLE, DROP ANY TABLE,
             AUDIT SYSTEM;

-- すべてのユーザーに対して有効化
AUDIT POLICY audit_dba_privs;

-- 特定のユーザーに対してのみ有効化
AUDIT POLICY audit_dba_privs BY hr_admin, sys_admin;

-- 失敗のみを監査（不正アクセスの試みの検出に有用）
AUDIT POLICY audit_dba_privs WHENEVER NOT SUCCESSFUL;

-- 成功と失敗の両方を監査
AUDIT POLICY audit_dba_privs WHENEVER SUCCESSFUL;
AUDIT POLICY audit_dba_privs WHENEVER NOT SUCCESSFUL;
```

### オブジェクト・アクションの監査

```sql
-- 特定の表に対するすべての DML を監査
CREATE AUDIT POLICY audit_salary_changes
  ACTIONS SELECT, INSERT, UPDATE, DELETE ON hr.employees;

AUDIT POLICY audit_salary_changes;

-- スキーマに対する DDL を監査
CREATE AUDIT POLICY audit_schema_ddl
  ACTIONS CREATE TABLE, ALTER TABLE, DROP TABLE,
          CREATE INDEX, DROP INDEX,
          CREATE VIEW, DROP VIEW,
          CREATE PROCEDURE, ALTER PROCEDURE, DROP PROCEDURE
  ON hr.employees;

-- 機密性の高いパッケージの EXECUTE を監査
CREATE AUDIT POLICY audit_payroll_pkg
  ACTIONS EXECUTE ON payroll.process_payroll_pkg;

AUDIT POLICY audit_payroll_pkg;

-- ログオンとログオフを監査
CREATE AUDIT POLICY audit_connections
  ACTIONS LOGON, LOGOFF;

AUDIT POLICY audit_connections;
```

### 条件付き監査 (WHEN 句)

`WHEN` 句を使用すると、特定の条件が真の場合にのみポリシーを起動できる。これにより、関心のあるケースのみにフィルタリングし、監査ボリュームを劇的に削減できる。

```sql
-- 営業時間外の給与データへのアクセスのみを監査
CREATE AUDIT POLICY audit_after_hours_salary
  ACTIONS SELECT ON hr.employees
  WHEN 'TO_NUMBER(TO_CHAR(SYSDATE,''HH24'')) NOT BETWEEN 8 AND 18'
  EVALUATE PER SESSION;

AUDIT POLICY audit_after_hours_salary;

-- 社内ネットワーク以外からのアクセスを監査
CREATE AUDIT POLICY audit_external_access
  ACTIONS SELECT, INSERT, UPDATE, DELETE ON payments.transactions
  WHEN 'SYS_CONTEXT(''USERENV'',''IP_ADDRESS'') NOT LIKE ''10.%'''
  EVALUATE PER SESSION;

AUDIT POLICY audit_external_access;

-- 特定のアプリーケーション接続を監査
CREATE AUDIT POLICY audit_third_party_access
  ACTIONS SELECT ON hr.employees
  WHEN 'SYS_CONTEXT(''USERENV'',''MODULE'') NOT LIKE ''HR_APP%'''
  EVALUATE PER SESSION;

AUDIT POLICY audit_third_party_access BY APP_READ_USER;
```

---

## 特権ユーザー (SYS および SYSDBA) の監査

`SYS` のような特権ユーザーは、標準的なセキュリティ制御をバイパスできる。Oracle は、これらのユーザーであっても監査するメカニズムを提供している。

```sql
-- SYS レベルの監査を有効化（OS ファイルまたは AUDSYS に書き込まれる）
-- 初期化パラメータ・ファイルで設定:
ALTER SYSTEM SET audit_sys_operations = TRUE SCOPE = SPFILE;
-- 再起動が必要

-- 統合監査で、特権ユーザー用のポリシーを作成
CREATE AUDIT POLICY audit_sysdba_actions
  ACTIONS ALL
  WHEN 'SYS_CONTEXT(''USERENV'',''ISDBA'') = ''TRUE'''
  EVALUATE PER SESSION;

AUDIT POLICY audit_sysdba_actions;

-- 指定された DBA アカウントによるすべてのアクションを監査
CREATE AUDIT POLICY audit_named_dbas
  ACTIONS ALL;

AUDIT POLICY audit_named_dbas BY dba_user1, dba_user2, dba_user3;
```

---

## ファイングレイン監査 (DBMS_FGA)

ファイングレイン監査 (FGA) を使用すると、特定の列にアクセスする SELECT 文を監査し、オプションで実際に実行されたクエリをキャプチャできる。統合監査モードでも `DBMS_FGA` ポリシーは引き続きサポートされ、そのレコードは `UNIFIED_AUDIT_TRAIL` に表示される。

```sql
-- クエリ・キャプチャを伴う機密列への SELECT を監査
-- 注: audit_trail パラメータは Oracle 23ai で非推奨となった。
-- すべての FGA レコードは自動的に UNIFIED_AUDIT_TRAIL に書き込まれる。
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema    => 'HR',
    object_name      => 'EMPLOYEES',
    policy_name      => 'AUDIT_SALARY_ACCESS',
    audit_column     => 'SALARY,COMMISSION_PCT',  -- これらが参照されたときのみ起動
    audit_condition  => NULL,  -- NULL = 常に監査
    statement_types  => 'SELECT',
    enable           => TRUE
  );
END;
/

-- 条件付き FGA: 給与が 100,000 を超える場合のみ監査
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'AUDIT_HIGH_SALARY_ACCESS',
    audit_column    => 'SALARY',
    audit_condition => 'SALARY > 100000',
    statement_types => 'SELECT'
  );
END;
/

-- ハンドラ・プロシージャを伴う FGA（アクセス時にアラートを出力）
CREATE OR REPLACE PROCEDURE security_alert(
  p_schema    IN VARCHAR2,
  p_object    IN VARCHAR2,
  p_policy    IN VARCHAR2
) AS
BEGIN
  INSERT INTO security.alerts (schema_name, object_name, policy_name,
                                db_user, os_user, ip_address, alert_time)
  VALUES (p_schema, p_object, p_policy,
          SYS_CONTEXT('USERENV', 'SESSION_USER'),
          SYS_CONTEXT('USERENV', 'OS_USER'),
          SYS_CONTEXT('USERENV', 'IP_ADDRESS'),
          SYSTIMESTAMP);
  COMMIT;
END;
/

BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema    => 'FINANCE',
    object_name      => 'WIRE_TRANSFERS',
    policy_name      => 'ALERT_WIRE_ACCESS',
    handler_schema   => 'SECURITY',
    handler_module   => 'SECURITY_ALERT',  -- ポリシー起動時に呼び出される
    enable           => TRUE
  );
END;
/

-- FGA ポリシーの管理
EXEC DBMS_FGA.ENABLE_POLICY('HR', 'EMPLOYEES', 'AUDIT_SALARY_ACCESS');
EXEC DBMS_FGA.DISABLE_POLICY('HR', 'EMPLOYEES', 'AUDIT_SALARY_ACCESS');
EXEC DBMS_FGA.DROP_POLICY('HR', 'EMPLOYEES', 'AUDIT_SALARY_ACCESS');
```

---

## UNIFIED_AUDIT_TRAIL のクエリ

`UNIFIED_AUDIT_TRAIL` ビューは、すべての監査レコードの主要な検索ポイントである。

### UNIFIED_AUDIT_TRAIL の主な列

| 列名 | 説明 |
|---|---|
| `EVENT_TIMESTAMP` | イベントが発生した日時 |
| `DBUSERNAME` | データベース・ユーザー名 |
| `OS_USERNAME` | OS のユーザー名 |
| `USERHOST` | クライアントのホスト名 |
| `UNIFIED_AUDIT_POLICIES` | このレコードをトリガーした監査ポリシー |
| `ACTION_NAME` | SQL アクション (SELECT, INSERT, LOGON など) |
| `OBJECT_SCHEMA` | アクセスされたオブジェクトのスキーマ |
| `OBJECT_NAME` | アクセスされたオブジェクトの名前 |
| `SQL_TEXT` | 実際の SQL ステートメント (CLOB として保存) |
| `RETURN_CODE` | Oracle エラー・コード (0 = 成功) |
| `AUTHENTICATION_TYPE` | 認証方法とクライアント・アドレスの詳細（`(CLIENT ADDRESS=((PROTOCOL=...)(HOST=ip)(PORT=port)))` 形式でクライアント IP が含まれる） |
| `SYSTEM_PRIVILEGE_USED` | 監査対象のアクションを実行するために使用されたシステム権限（例: `SYSDBA`） |

> **注:** `UNIFIED_AUDIT_TRAIL` には `CLIENT_IP` 列はない。クライアント IP は `AUTHENTICATION_TYPE` 文字列の内部に含まれている。また、`AUTHENTICATION_PRIVILEGE` 列も存在しない。SYSDBA/SYSOPER の使用を確認するには `SYSTEM_PRIVILEGE_USED` を使用する。

### 一般的な監査証跡クエリ

```sql
-- 最近のログイン失敗（ブルートフォース攻撃の検出）
-- 注: UNIFIED_AUDIT_TRAIL には CLIENT_IP 列はない。クライアント IP は AUTHENTICATION_TYPE 内にある
SELECT event_timestamp, dbusername, userhost, return_code, authentication_type
FROM unified_audit_trail
WHERE action_name = 'LOGON'
  AND return_code != 0
  AND event_timestamp > SYSDATE - 1  -- 過去 24 時間
ORDER BY event_timestamp DESC;

-- 本日給与データにアクセスしたユーザー
SELECT event_timestamp, dbusername, userhost, action_name, sql_text
FROM unified_audit_trail
WHERE object_name = 'EMPLOYEES'
  AND object_schema = 'HR'
  AND unified_audit_policies LIKE '%SALARY%'
  AND event_timestamp > TRUNC(SYSDATE)
ORDER BY event_timestamp DESC;

-- 過去 7 日間の DDL 変更
SELECT event_timestamp, dbusername, userhost, action_name,
       object_schema, object_name, sql_text
FROM unified_audit_trail
WHERE action_name IN ('CREATE TABLE', 'DROP TABLE', 'ALTER TABLE',
                       'CREATE INDEX', 'DROP INDEX', 'TRUNCATE TABLE')
  AND event_timestamp > SYSDATE - 7
ORDER BY event_timestamp DESC;

-- 権限付与 (SOX: 誰がアクセス権を変更したか？)
SELECT event_timestamp, dbusername, userhost, action_name, sql_text
FROM unified_audit_trail
WHERE action_name IN ('GRANT', 'REVOKE', 'CREATE ROLE', 'DROP ROLE',
                       'CREATE USER', 'DROP USER', 'ALTER USER')
  AND event_timestamp > SYSDATE - 30
ORDER BY event_timestamp DESC;

-- 過去 1 週間以内の SYS または SYSDBA によるすべてのアクション
-- SYSTEM_PRIVILEGE_USED を使用して SYSDBA ログインを検出する（AUTHENTICATION_PRIVILEGE 列はない）
SELECT event_timestamp, dbusername, os_username, userhost,
       action_name, object_name, sql_text, system_privilege_used
FROM unified_audit_trail
WHERE (dbusername = 'SYS' OR system_privilege_used LIKE '%SYSDBA%')
  AND event_timestamp > SYSDATE - 7
ORDER BY event_timestamp DESC;

-- アクセス試行の失敗（SQL インジェクションまたは不正アクセスの可能性）
SELECT event_timestamp, dbusername, userhost, action_name,
       object_name, return_code, sql_text
FROM unified_audit_trail
WHERE return_code NOT IN (0, 1403)  -- NOT FOUND (1403) を除外
  AND event_timestamp > SYSDATE - 1
ORDER BY event_timestamp DESC, dbusername;

-- 最もアクティブなユーザー TOP 20（監査イベント数順）
SELECT dbusername, COUNT(*) event_count,
       COUNT(DISTINCT action_name) distinct_actions
FROM unified_audit_trail
WHERE event_timestamp > SYSDATE - 7
GROUP BY dbusername
ORDER BY event_count DESC
FETCH FIRST 20 ROWS ONLY;
```

---

## 監査ポリシーの管理

```sql
-- 定義されているすべての監査ポリシーを一覧表示
SELECT policy_name, enabled_option, entity_name, entity_type,
       success, failure
FROM audit_unified_enabled_policies
ORDER BY policy_name, entity_name;

-- すべてのポリシー（無効なものを含む）を一覧表示
SELECT policy_name, policy_type, object_schema, object_name,
       condition_eval_opt
FROM audit_unified_policies
ORDER BY policy_name;

-- 監査ポリシーを無効化
NOAUDIT POLICY audit_salary_changes;

-- 特定のユーザーに対してのみ無効化
NOAUDIT POLICY audit_salary_changes BY specific_user;

-- ポリシーを永久に削除
DROP AUDIT POLICY audit_salary_changes;

-- 古い監査レコードのパージ（AUDIT_ADMIN ロールまたは DBA が必要）
-- CLEAN_AUDIT_TRAIL には delete_timestamp パラメータはない。
-- 90 日より古いレコードをパージするには、まずアーカイブ・タイムスタンプを設定してから CLEAN を呼び出す。
BEGIN
  DBMS_AUDIT_MGMT.SET_LAST_ARCHIVE_TIMESTAMP(
    audit_trail_type  => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    last_archive_time => SYSTIMESTAMP - INTERVAL '90' DAY
  );
END;
/

-- その後、アーカイブ・タイムスタンプより古いレコードをパージする
BEGIN
  DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
    audit_trail_type        => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    use_last_arch_timestamp => TRUE   -- 上記で設定したタイムスタンプより古いレコードを削除
  );
END;
/

BEGIN
  DBMS_AUDIT_MGMT.CREATE_PURGE_JOB(
    audit_trail_type           => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    audit_trail_purge_interval => 24,          -- 24 時間ごとに実行
    audit_trail_purge_name     => 'DAILY_AUDIT_PURGE',
    use_last_arch_timestamp    => TRUE
  );
END;
/
```

---

## 監査証跡のアーキテクチャとサイジング

統合監査証跡は、デフォルトで `SYSAUX` 表領域の `AUDSYS` スキーマに格納される。監査ボリュームが大きい大規模なデータベースの場合は、専用の表領域に移動することを検討する。

```sql
-- 現在の監査証跡表領域を確認
SELECT tablespace_name
FROM dba_segments
WHERE owner = 'AUDSYS'
FETCH FIRST 1 ROWS ONLY;

-- 監査証跡のサイズを確認
SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) AS size_gb
FROM dba_segments
WHERE owner = 'AUDSYS';

-- 監査証跡を専用の表領域に移動（AUDIT_ADMIN ロールが必要）
BEGIN
  DBMS_AUDIT_MGMT.SET_AUDIT_TRAIL_LOCATION(
    audit_trail_type    => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    audit_trail_location_value => 'AUDIT_TBS'  -- 事前に作成しておく必要がある
  );
END;
/

-- 監査証跡の増加傾向を確認
SELECT TRUNC(event_timestamp, 'DD') AS audit_day,
       COUNT(*) AS event_count,
       ROUND(COUNT(*) * 500 / 1024 / 1024, 2) AS approx_mb  -- ~500 バイト/レコード
FROM unified_audit_trail
WHERE event_timestamp > SYSDATE - 30
GROUP BY TRUNC(event_timestamp, 'DD')
ORDER BY audit_day;
```

---

## ベスト・プラクティス

1.  **認証イベントを常に監査する**: ログオン失敗は、ブルートフォース攻撃や不正アクセスの試みを発見するための主要な指標である。ログオン成功は、異常検出のベースラインを確立する。

2.  **すべての DDL を監査する**: スキーマの変更は、監査証跡の破棄、表の削除、またはバックドアの作成につながる可能性がある。アプリケーション・スキーマにおけるすべての `CREATE`, `ALTER`, `DROP`, `TRUNCATE` は監査する必要がある。

3.  **すべての権限とロールの付与を監査する**: これらはアクセス制御モデルにおける変更イベントである。SOX や PCI の下では、これらはレビューの対象となる必要がある。

4.  **機密列へのアクセスには `DBMS_FGA` を使用する**: 給与データにどのクエリがアクセスしたかを正確に知る必要がある場合、FGA が適切なツールである。SQL テキストは `UNIFIED_AUDIT_TRAIL.SQL_TEXT` にキャプチャされる。注: `audit_trail` パラメータ（DBMS_FGA.DB + DBMS_FGA.EXTENDED）は Oracle 23ai で非推奨となり、すべての FGA レコードは自動的に統合監査証跡に格納される。

5.  **監査対象のデータベースから監査レコードを分離して保存する**: 理想的には、監査レコードをリアルタイムで SIEM または別の監査データベースに送信する。監査レコードを削除できる DBA は、自分の行動を隠蔽できてしまう可能性がある。

6.  **頻繁にアクセスされる表の SELECT を過剰に監査しない**: 1 日に数百万回の読み取りがある表のすべての SELECT を監査すると、監査表領域が急速にいっぱいになり、パフォーマンスに影響を与える可能性がある。条件付きの FGA を使用して、重要なものだけをターゲットにする。

7.  **監査ポリシーを四半期ごとに見直す**: 6 か月前に適切だったポリシーが、現在は適用されない可能性がある。未使用のポリシーを削除し、アプリケーションの進化に合わせて新しいポリシーを追加する。

8.  **ポリシーが起動するかテストする**: ポリシー作成後、監査対象のアクションを実行し、`UNIFIED_AUDIT_TRAIL` をクエリして、レコードが生成されることを確認する。

---

## よくある間違いとその回避方法

### 間違い 1: デフォルトですべてを監査する

```sql
-- 不適切: 負荷の高い OLTP システムでは 1 時間に数百万のレコードが生成される
AUDIT ALL STATEMENTS;

-- 適切: 権限の使用、DDL、および機密データへのアクセスを戦略的に監査する
CREATE AUDIT POLICY targeted_audit
  ACTIONS SELECT ON finance.wire_transfers,
          DELETE ON finance.wire_transfers,
          INSERT ON finance.wire_transfers,
          UPDATE ON finance.wire_transfers;
AUDIT POLICY targeted_audit;
```

### 間違い 2: 監視なしに DBA が監査レコードをパージできるようにする

```sql
-- 通常の DBA が持たない個別の AUDIT_ADMIN ロールを作成する
-- DBA は自分自身のアクションの監査レコードをパージすることはできない
-- パージ操作には二重制御を要求する

-- 監査証跡を管理できるユーザーを確認
SELECT grantee FROM dba_sys_privs
WHERE privilege = 'AUDIT SYSTEM'
UNION
SELECT grantee FROM dba_role_privs
WHERE granted_role = 'AUDIT_ADMIN';
```

### 間違い 3: パージ前にアーカイブしない

```sql
-- パージする前に、常に監査レコードを外部システムにアーカイブする
-- Oracle Advanced Queuing、ロギング DB へのデータベース・リンク、または SIEM へのエクスポートを使用する

-- 例: パージ前に DB リンク経由でアーカイブ
INSERT INTO audit_archive.unified_audit_archive@audit_db_link
SELECT * FROM unified_audit_trail
WHERE event_timestamp < SYSTIMESTAMP - INTERVAL '90' DAY;
COMMIT;

-- その後、設定したアーカイブ・タイムスタンプより古いレコードをパージする
BEGIN
  DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
    audit_trail_type        => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    use_last_arch_timestamp => TRUE
  );
END;
/
```

### 間違い 4: ポリシーに SQL テキストを含めない

```sql
-- 不適切: ユーザーが表にアクセスしたことは分かるが、どのようなクエリを実行したかは分からない
CREATE AUDIT POLICY audit_emp ACTIONS SELECT ON hr.employees;

-- 適切: FGA を使用して実際の SQL テキストをキャプチャする（UNIFIED_AUDIT_TRAIL.SQL_TEXT に保存される）
-- audit_trail パラメータは 23ai で非推奨となり、すべてのレコードが UNIFIED_AUDIT_TRAIL に格納される
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema => 'HR', object_name => 'EMPLOYEES',
    policy_name   => 'FGA_EMP_SELECT'
  );
END;
/
```

---

## コンプライアンスの考慮事項

### SOX (サーベンス・オクスリー法)
- 財務データ表へのすべてのアクセスをログに記録する必要がある
- すべての権限付与とロールの割り当てをログに記録する必要がある
- 監査ログは 7 年間保持する必要がある
- DBA の活動は監査され、独立してレビューされる必要がある
- 監査レコードは改ざんから保護される必要がある

### PCI-DSS
- 要件 10: ネットワーク・リソースおよび会員データへのすべてのアクセスを追跡および監視する
- 要件 10.2: すべてのシステム・コンポーネントに対して監査ログを実装する
- 要件 10.3: 日時、ユーザー、イベントの種類、および結果をキャプチャする
- 要件 10.5: 監査証跡が変更されないように保護する
- 要件 10.7: 監査証跡の履歴を少なくとも 1 年間保持する

```sql
-- PCI 準拠の会員データ・アクセス用ポリシー
CREATE AUDIT POLICY pci_cardholder_audit
  ACTIONS SELECT, INSERT, UPDATE, DELETE
    ON payments.card_data,
  ACTIONS SELECT, INSERT, UPDATE, DELETE
    ON payments.transactions;

AUDIT POLICY pci_cardholder_audit;

-- クエリ・テキストをキャプチャするための補完的な FGA（レコードは UNIFIED_AUDIT_TRAIL.SQL_TEXT に格納される）
-- audit_trail パラメータは 23ai で非推奨となっている
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema => 'PAYMENTS', object_name => 'CARD_DATA',
    policy_name   => 'PCI_FGA_CARD_DATA'
  );
END;
/
```

### HIPAA
- 45 CFR 164.312(b): PHI（個人健康情報）を含む情報システムにおける活動を記録および調査するためのハードウェア、ソフトウェア、および手順メカニズムを実装する
- 監査ログには、ユーザー ID、日付、時刻、およびアクセスの種類を含める必要がある
- ログは定期的に（通常は毎週/毎月）確認する必要がある
- 監査ログは少なくとも 6 年間保持する必要がある

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効である。
- 21c、23c、または 23ai と記された機能は、Oracle Database 26ai 対応機能として扱う。バージョンが混在する環境では、19c 互換の代替案を保持すること。
- リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンをサポートする環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database Security Guide 19c — Introduction to Auditing](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/introduction-to-auditing.html)
- [Oracle Database Security Guide 19c — Administering the Audit Trail](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/administering-the-audit-trail.html)
- [Oracle Database Reference 19c — UNIFIED_AUDIT_TRAIL](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/UNIFIED_AUDIT_TRAIL.html)
- [Oracle PL/SQL Packages Reference 19c — DBMS_FGA](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_FGA.html)
- [Oracle PL/SQL Packages Reference 19c — DBMS_AUDIT_MGMT](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_AUDIT_MGMT.html)
