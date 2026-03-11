# Oracle 行レベル・セキュリティ (Virtual Private Database)

## 概要

Virtual Private Database (VPD) は、きめ細かなアクセス制御 (FGAC: Fine-Grained Access Control) とも呼ばれ、データベース・カーネル・レベルで透過的に行レベル・セキュリティを強制するための Oracle のメカニズムである。アプリケーション・レイヤーでのフィルタリング（アドホック・クエリ、ETL ツール、レポート作成アプリケーションなどによってバイパスされる可能性がある）とは異なり、VPD ポリシーは、クエリがどのようにデータベースに到達したかに関わらず、Oracle クエリ・エンジン自体によって強制される。

VPD ポリシーが表に適用されると、Oracle はその表に触れるすべての SQL 文を自動的に書き換え、PL/SQL ポリシー・関数によって生成された `WHERE` 句を追加する。これは、オプティマイザがクエリを参照する前に行われる。その結果、ユーザーはアクセス許可のない行を読み取ったり、更新したり、削除したりすることが文字通り不可能になる。

VPD は、**Oracle Database Enterprise Edition** に含まれるライセンス機能である。マルチテナント SaaS アプリケーション、医療システム (HIPAA)、金融システム (SOX, PCI-DSS) などで広く使用されている。

---

## アーキテクチャの概要

```
アプリケーションが送信する SQL:
  SELECT * FROM hr.employees

Oracle VPD がこれをインターセプトして書き換えた SQL:
  SELECT * FROM hr.employees
  WHERE department_id IN (
    SELECT department_id FROM hr.user_dept_access
    WHERE username = SYS_CONTEXT('USERENV', 'SESSION_USER')
  )

実際に実行されるのは、この書き換えられたクエリである。
```

VPD 実装の構成要素は以下の通り。

1.  **アプリケーション・コンテキスト**: 現在のユーザーのセキュリティ・コンテキスト（例: テナント ID、ユーザー・ロール、部門）を保持するために使用される、セッション・レベルの属性（キーと値のペア）の名前付きセット。
2.  **ポリシー・ファンクション**: `VARCHAR2` 形式の述語（プレディケート）文字列を返す PL/SQL 関数。これは Oracle が追加する `WHERE` 句の断片となる。
3.  **ポリシー**: `DBMS_RLS.ADD_POLICY` を介して登録され、表/ビューをポリシー・ファンクションに関連付け、いつ実行されるかを構成する。

---

## アプリケーション・コンテキスト (Application Contexts)

アプリケーション・コンテキストは、ポリシー・ファンクションが読み取るセッション固有のセキュリティ属性を格納するための主要なメカニズムである。名前空間を持ち、指定された信頼できる（trusted）パッケージによってのみ設定可能である。

### アプリケーション・コンテキストの作成

```sql
-- コンテキスト名前空間の作成
-- USING 句で、このコンテキストに値を設定する権限を持つパッケージを指定する
CREATE OR REPLACE CONTEXT hr_security_ctx USING hr.set_context_pkg;

-- グローバルにアクセス可能なコンテキスト（すべてのセッションで共有）の場合
CREATE OR REPLACE CONTEXT hr_security_ctx USING hr.set_context_pkg ACCESSED GLOBALLY;
```

### コンテキスト設定パッケージの作成

```sql
CREATE OR REPLACE PACKAGE hr.set_context_pkg AS
  PROCEDURE set_user_context;
END set_context_pkg;
/

CREATE OR REPLACE PACKAGE BODY hr.set_context_pkg AS

  PROCEDURE set_user_context IS
    v_dept_id    NUMBER;
    v_user_role  VARCHAR2(30);
    v_tenant_id  NUMBER;
  BEGIN
    -- 現在のユーザーの部門とロールを検索
    SELECT e.department_id, e.job_id
    INTO v_dept_id, v_user_role
    FROM hr.employees e
    WHERE e.email = SYS_CONTEXT('USERENV', 'SESSION_USER');

    -- このセッションのコンテキスト値を設定
    DBMS_SESSION.SET_CONTEXT('hr_security_ctx', 'dept_id',   v_dept_id);
    DBMS_SESSION.SET_CONTEXT('hr_security_ctx', 'user_role', v_user_role);

  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      -- 従業員でないユーザーにはアクセスを許可しない — 部門を -1 に設定
      DBMS_SESSION.SET_CONTEXT('hr_security_ctx', 'dept_id',   '-1');
      DBMS_SESSION.SET_CONTEXT('hr_security_ctx', 'user_role', 'NONE');
  END set_user_context;

END set_context_pkg;
/
```

### ログオン・トリガーによるコンテキスト値の設定

コンテキストに値を入力する最も明白で確実な方法は、セッションが確立されるたびに自動的に実行されるログオン・トリガーを使用することである。

```sql
CREATE OR REPLACE TRIGGER hr.set_logon_context
  AFTER LOGON ON DATABASE
BEGIN
  -- HR アプリケーションへの非 DBA セッションに対してのみコンテキストを設定
  IF SYS_CONTEXT('USERENV', 'SESSION_USER') NOT IN ('SYS', 'SYSTEM') THEN
    hr.set_context_pkg.set_user_context;
  END IF;
EXCEPTION
  WHEN OTHERS THEN
    NULL;  -- コンテキストのエラーでログインを妨げないようにする（代わりにログを記録するなどの対応を検討）
END;
/
```

### コンテキスト値の読み取り

```sql
-- ポリシー・ファンクションおよび SQL クエリ内
SYS_CONTEXT('hr_security_ctx', 'dept_id')
SYS_CONTEXT('hr_security_ctx', 'user_role')

-- 組み込みの USERENV 名前空間の値（常に利用可能）
SYS_CONTEXT('USERENV', 'SESSION_USER')    -- データベース・ユーザー名
SYS_CONTEXT('USERENV', 'OS_USER')         -- OS ユーザー名
SYS_CONTEXT('USERENV', 'IP_ADDRESS')      -- クライアント IP
SYS_CONTEXT('USERENV', 'MODULE')          -- アプリケーション・モジュール名
SYS_CONTEXT('USERENV', 'CLIENT_INFO')     -- アプリケーションのクライアント情報
SYS_CONTEXT('USERENV', 'HOST')            -- クライアント・ホスト名
SYS_CONTEXT('USERENV', 'DB_NAME')         -- データベース名
SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA')  -- 現在のスキーマ
SYS_CONTEXT('USERENV', 'ISDBA')           -- DBA としてログインしている場合は 'TRUE'
```

---

## ポリシー・ファンクション (Policy Functions)

ポリシー・ファンクションは特定のシグネチャを持つ必要がある。スキーマ名とオブジェクト名をパラメータとして受け取り、`VARCHAR2` 形式の述語（プレディケート）文字列を返す。`NULL` または空の文字列を返した場合、制限は適用されない（すべての行が表示される）。`'1=0'` を返した場合、どの行も表示されない。

### シンプルな部門ベースのポリシー

```sql
CREATE OR REPLACE FUNCTION hr.emp_dept_policy(
  p_schema  IN VARCHAR2,
  p_object  IN VARCHAR2
) RETURN VARCHAR2 AS
  v_predicate VARCHAR2(4000);
  v_dept_id   VARCHAR2(10);
  v_role      VARCHAR2(30);
BEGIN
  v_dept_id := SYS_CONTEXT('hr_security_ctx', 'dept_id');
  v_role    := SYS_CONTEXT('hr_security_ctx', 'user_role');

  -- HR Manager は自分の部門を表示できる
  -- HR Director はすべての部門を表示できる
  -- それ以外のユーザーは自分のレコードのみを表示できる
  IF v_role = 'HR_MGR' THEN
    v_predicate := 'department_id = ' || v_dept_id;
  ELSIF v_role = 'HR_DIR' THEN
    v_predicate := NULL;  -- 制限なし（全行表示）
  ELSE
    -- 一般従業員は自分の行のみ
    v_predicate := 'email = SYS_CONTEXT(''USERENV'', ''SESSION_USER'')';
  END IF;

  RETURN v_predicate;
END emp_dept_policy;
/
```

### マルチテナント分離ポリシー

```sql
CREATE OR REPLACE FUNCTION saas.tenant_isolation_policy(
  p_schema  IN VARCHAR2,
  p_object  IN VARCHAR2
) RETURN VARCHAR2 AS
BEGIN
  -- コンテキストが設定されていない場合 (DBA によるバイパスや ETL 時)、不可能な述語を返す
  IF SYS_CONTEXT('saas_ctx', 'tenant_id') IS NULL THEN
    -- DBA には無制限のアクセスを許可
    IF SYS_CONTEXT('USERENV', 'ISDBA') = 'TRUE' THEN
      RETURN NULL;
    END IF;
    RETURN '1=0';  -- コンテキストなし = 行なし
  END IF;

  RETURN 'tenant_id = ' || SYS_CONTEXT('saas_ctx', 'tenant_id');
END tenant_isolation_policy;
/
```

---

## DBMS_RLS によるポリシーの登録

### 基本的なポリシーの登録

```sql
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'EMP_DEPT_VISIBILITY',
    function_schema => 'HR',
    policy_function => 'EMP_DEPT_POLICY',
    statement_types => 'SELECT, INSERT, UPDATE, DELETE',
    update_check    => TRUE,   -- UPDATE によって行が範囲外に移動するのを防ぐ
    enable          => TRUE
  );
END;
/
```

### ポリシー・パラメータの解説

| パラメータ | 説明 |
|---|---|
| `object_schema` | 保護対象の表/ビューのスキーマ |
| `object_name` | 保護対象の表、ビュー、またはシノニムの名前 |
| `policy_name` | このポリシーの一意の名前 |
| `function_schema` | ポリシー・ファンクションが存在するスキーマ |
| `policy_function` | ポリシー・ファンクションの名前 |
| `statement_types` | カンマ区切り: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `INDEX` |
| `update_check` | TRUE の場合、ポリシーの述語から外れるような内容での INSERT/UPDATE を防ぐ |
| `enable` | ポリシーを即座に有効にするかどうか |
| `static_policy` | TRUE の場合、関数は一度だけ呼び出され結果がキャッシュされる（最高パフォーマンス） |
| `policy_type` | `static_policy` よりも詳細なキャッシュ・ポリシー定数を指定可能 |
| `long_predicate` | TRUE の場合、最大 32KB までの述語を許可する |
| `sec_relevant_cols` | カンマ区切りの列。これらの列が参照された場合にのみポリシーが起動する |
| `sec_relevant_cols_opt` | `DBMS_RLS.ALL_ROWS` に設定すると、制限された行を非表示にする代わりに、機密列を NULL にして行を返す |

---

## ポリシー・タイプ (Policy Types)

Oracle は、ポリシー・ファンクションが呼び出されるタイミングと結果がキャッシュされるかどうかを制御するいくつかのポリシー・タイプを提供している。適切なタイプを選択することはパフォーマンスにとって非常に重要である。

```sql
-- DBMS_RLS におけるポリシー・タイプ定数:
-- DBMS_RLS.STATIC             -- 関数は一度だけ呼び出され、ポリシー有効期間中は結果がキャッシュされる
-- DBMS_RLS.SHARED_STATIC      -- すべてのユーザー間で共有される 1 つのキャッシュ済み述語
-- DBMS_RLS.CONTEXT_SENSITIVE  -- アプリケーション・コンテキストが変更されたときに再評価される
-- DBMS_RLS.SHARED_CONTEXT_SENSITIVE -- 共有された述語。コンテキスト変更時に再評価される
-- DBMS_RLS.DYNAMIC            -- クエリごとに毎回呼び出される (デフォルト)

-- スタティック（静的）ポリシー: 最高パフォーマンス。ユーザーごとに述語が変わらない場合
-- 例: 'status != ''DELETED''' のようなグローバルなフィルタリング
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'APP',
    object_name     => 'ORDERS',
    policy_name     => 'HIDE_DELETED_ORDERS',
    function_schema => 'APP',
    policy_function => 'ACTIVE_RECORDS_POLICY',
    statement_types => 'SELECT',
    policy_type     => DBMS_RLS.STATIC
  );
END;
/

-- コンテキスト・センシティブ・ポリシー: アプリケーション・コンテキスト変更時に再評価
-- アプリケーション・コンテキストに基づくマルチテナント分離に最適
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'SAAS',
    object_name     => 'CUSTOMER_DATA',
    policy_name     => 'TENANT_ISOLATION',
    function_schema => 'SAAS',
    policy_function => 'TENANT_ISOLATION_POLICY',
    statement_types => 'SELECT, INSERT, UPDATE, DELETE',
    update_check    => TRUE,
    policy_type     => DBMS_RLS.CONTEXT_SENSITIVE
  );
END;
/

-- ダイナミック（動的）ポリシー: クエリを実行するたびに呼び出される
-- クエリ実行時のデータ検索に述語が依存する場合に使用する
-- 柔軟性は高いが低速。高頻度でアクセスされる表には避けるべきである
BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema   => 'HR',
    object_name     => 'SALARY_HISTORY',
    policy_name     => 'SALARY_ACCESS',
    function_schema => 'HR',
    policy_function => 'SALARY_POLICY',
    statement_types => 'SELECT',
    policy_type     => DBMS_RLS.DYNAMIC
  );
END;
/
```

---

## 列マスキング・ポリシー (sec_relevant_cols_opt)

行全体を非表示にするのではなく、権限のないユーザーに対しては機密列の値を NULL にして行を返すことができる。

```sql
CREATE OR REPLACE FUNCTION hr.salary_col_policy(
  p_schema  IN VARCHAR2,
  p_object  IN VARCHAR2
) RETURN VARCHAR2 AS
BEGIN
  -- HR および Payroll ロールは給与を表示可能。その他は NULL にする。
  IF SYS_CONTEXT('hr_security_ctx', 'user_role') IN ('HR_MGR', 'PAYROLL') THEN
    RETURN NULL;  -- 制限なし
  END IF;
  RETURN '1=0';  -- sec_relevant_cols_opt = ALL_ROWS によってオーバーライドされる
END salary_col_policy;
/

BEGIN
  DBMS_RLS.ADD_POLICY(
    object_schema          => 'HR',
    object_name            => 'EMPLOYEES',
    policy_name            => 'MASK_SALARY',
    function_schema        => 'HR',
    policy_function        => 'SALARY_COL_POLICY',
    statement_types        => 'SELECT',
    sec_relevant_cols      => 'SALARY,COMMISSION_PCT',
    sec_relevant_cols_opt  => DBMS_RLS.ALL_ROWS  -- 給与を NULL にして行自体は返す
  );
END;
/

-- HR/Payroll 以外のロールの従業員には以下のように見える:
-- EMPLOYEE_ID | LAST_NAME | SALARY | COMMISSION_PCT
-- 100         | King      | (null) | (null)
-- 101         | Kochhar   | (null) | (null)
```

---

## ポリシーの管理

```sql
-- 無効になっているポリシーを有効にする
EXEC DBMS_RLS.ENABLE_POLICY('HR', 'EMPLOYEES', 'EMP_DEPT_VISIBILITY', TRUE);

-- ポリシーを無効にする (メンテナンス時の DBA バイパスなど)
EXEC DBMS_RLS.ENABLE_POLICY('HR', 'EMPLOYEES', 'EMP_DEPT_VISIBILITY', FALSE);

-- ポリシーの削除
EXEC DBMS_RLS.DROP_POLICY('HR', 'EMPLOYEES', 'EMP_DEPT_VISIBILITY');

-- スタティックまたはコンテキスト・センシティブ・ポリシーのリフレッシュ (キャッシュされた述語を再評価させる)
EXEC DBMS_RLS.REFRESH_POLICY('HR', 'EMPLOYEES', 'EMP_DEPT_VISIBILITY');

-- 表のすべてのポリシーをリフレッシュ
EXEC DBMS_RLS.REFRESH_POLICY('HR', 'EMPLOYEES');

-- 特定の表に適用されているすべてのポリシーを表示
SELECT object_owner, object_name, policy_name, function, sel, ins, upd, del,
       enable, static_policy, policy_type, long_predicate
FROM dba_policies
WHERE object_owner = 'HR'
  AND object_name  = 'EMPLOYEES'
ORDER BY policy_name;

-- データベース内のすべてのポリシーを表示
SELECT object_owner, object_name, policy_name, function, enable, policy_type
FROM dba_policies
ORDER BY object_owner, object_name;
```

---

## ポリシー・グループ (Groups of Policies)

同じオブジェクトに対して複数のポリシーを適用すると、それらは AND で組み合わされる。排他的なポリシー（一度に 1 つのグループのみ有効）を使用したい場合は、ポリシー・グループを利用できる。

```sql
-- 名前付きポリシー・グループの作成
EXEC DBMS_RLS.CREATE_POLICY_GROUP('HR', 'EMPLOYEES', 'INTERNAL_GROUP');
EXEC DBMS_RLS.CREATE_POLICY_GROUP('HR', 'EMPLOYEES', 'EXTERNAL_GROUP');

-- 各グループにポリシーを追加
BEGIN
  DBMS_RLS.ADD_GROUPED_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_group    => 'INTERNAL_GROUP',
    policy_name     => 'INTERNAL_FILTER',
    function_schema => 'HR',
    policy_function => 'INTERNAL_EMP_POLICY'
  );
END;
/

-- このオブジェクトのポリシー・グループを制御する「駆動」アプリケーション・コンテキストを関連付ける。
-- Oracle は実行時にコンテキスト値を読み取り、アクティブなポリシー・グループを決定する。
-- どのコンテキスト属性がグループ選択を駆動するかを登録するには DBMS_RLS.ADD_POLICY_CONTEXT を使用する:
EXEC DBMS_RLS.ADD_POLICY_CONTEXT('HR', 'EMPLOYEES', 'HR_APP_CTX', 'POLICY_GROUP');

-- セッション時に、コンテキスト値を目的のグループ名に設定する:
EXEC DBMS_SESSION.SET_CONTEXT('HR_APP_CTX', 'POLICY_GROUP', 'INTERNAL_GROUP');

-- DBMS_RLS.SET_CONTEXT というプロシージャは存在しない。ポリシー・グループの有効化は、
-- ADD_POLICY_CONTEXT で登録されたアプリケーション・コンテキスト値によって駆動される。
```

---

## VPD のテストとデバッグ

```sql
-- 現在のセッションでポリシーがどのような述語を生成するかテストする
-- (DBMS_RLS への EXECUTE 権限または DBA ロールが必要)
DECLARE
  v_predicate VARCHAR2(4000);
BEGIN
  v_predicate := hr.emp_dept_policy('HR', 'EMPLOYEES');
  DBMS_OUTPUT.PUT_LINE('Predicate: [' || v_predicate || ']');
END;
/

-- 現在のセッションに設定されているコンテキスト値を確認
SELECT namespace, attribute, value
FROM session_context
ORDER BY namespace, attribute;

-- SQL トレースを有効にして、実際に書き換えられたクエリを確認する
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';
SELECT * FROM hr.employees;
ALTER SESSION SET EVENTS '10046 trace name context off';
-- トレース・ファイルを確認する。述語は PARSE セクションに表示される。

-- 一時的に VPD なしでテストする (EXEMPT ACCESS POLICY 権限が必要。細心の注意を払うこと)
-- この権限を持つユーザーのみが VPD をバイパスできる。
-- GRANT EXEMPT ACCESS POLICY TO dba_test_user;  -- テスト目的のみに使用し、本番環境では決して付与しないこと。
```

---

## ベスト・プラクティス

1.  **コンテキストの設定には必ずログオン・トリガーを使用する**: アプリケーションがクエリの前に `DBMS_SESSION.SET_CONTEXT` を呼び出すことに依存すると、コンテキストがない間にクエリが実行され、行が表示されない（あるいはポリシーによっては全行表示される）といった隙が生まれる可能性がある。ログオン・トリガーはこのギャップをなくす。

2.  **`CONTEXT_SENSITIVE` または `STATIC` ポリシー・タイプを使用する**: `DYNAMIC` ポリシーはすべてのクエリで再評価されるため、負荷の高い表では重大なパフォーマンス・ボトルネックになる可能性がある。述語がセッション内で動的に変更される必要がある場合にのみ使用すること。

3.  **コンテキストが欠落している場合は、NULL ではなく `'1=0'` を返す**: コンテキスト値が設定されていないときに NULL を返すと、ポリシーの述語が削除され、すべての行が露出してしまう。`'1=0'` を返すことで、デフォルトを「すべて拒否 (deny-all)」にできる。

4.  **`update_check => TRUE` を設定する**: これにより、ユーザーが自分の表示範囲外となるような INSERT または UPDATE を行うことを防ぐ。そうしないと、そのユーザーが二度とアクセスや変更ができない孤立した行が作成されてしまう可能性がある。

5.  **ポリシー・ファンクション内で DML を実行しない**: ポリシー・ファンクションは、クエリのパース中に Oracle カーネルによって呼び出される。ポリシー・ファンクション内での DML は再帰的な SQL やエラーの原因となる。

6.  **単純なケースではビューを使用する**: フィルタリング・ロジックが静的であるか単純な結合に基づいている場合は、`WHERE` 句を備えたビューの方が VPD よりもシンプルで高速な場合がある。

7.  **すべてのポリシーを文書化する**: 強制されているビジネス・ルールをポリシー・ファンクション内のコメントとして含めること。VPD はクエリ作成者からは見えないため、十分な文書化が必要である。

---

## よくある間違いとその回避方法

### 間違い 1: コンテキストが欠落しているときに NULL を返してしまう

```sql
-- 不適切: コンテキストが設定されていないときに NULL を返し、フィルタが削除されてしまう
CREATE OR REPLACE FUNCTION bad_policy(p_schema VARCHAR2, p_object VARCHAR2)
RETURN VARCHAR2 AS
BEGIN
  RETURN 'tenant_id = ' || SYS_CONTEXT('app_ctx', 'tenant_id');
  -- tenant_id が NULL の場合、これは 'tenant_id = ' となり SQL エラーになる。
  -- またはポリシー・ファンクションが例外を投げた場合、Oracle は全行を公開してしまう可能性がある。
END;
/

-- 適切: フェイルセーフを用いた明示的な NULL チェック
CREATE OR REPLACE FUNCTION good_policy(p_schema VARCHAR2, p_object VARCHAR2)
RETURN VARCHAR2 AS
  v_tenant VARCHAR2(10) := SYS_CONTEXT('app_ctx', 'tenant_id');
BEGIN
  IF v_tenant IS NULL THEN
    RETURN '1=0';  -- デフォルト拒否
  END IF;
  RETURN 'tenant_id = ' || TO_NUMBER(v_tenant);  -- インジェクション防止のため数値にキャスト
END;
/
```

### 間違い 2: ポリシー述語における SQL インジェクション

ポリシー・ファンクションは動的な SQL 述語を構築する。ユーザーが提供した値をそのまま連結すると、WHERE 句に SQL インジェクションが入り込む可能性がある。

```sql
-- 潜在的に安全: SYS_CONTEXT からの値のみを信頼する (ユーザー入力を直接使わない)
-- SYS_CONTEXT の値は信頼できるパッケージによってのみ設定されるため、安全である
RETURN 'dept_id = ' || SYS_CONTEXT('app_ctx', 'dept_id');  -- 安全: コンテキストは信頼されている

-- 危険: 未加工のユーザー入力から述語を構築してはいけない
RETURN 'dept_id = ' || v_user_supplied_param;  -- 絶対に行わないこと
```

### 間違い 3: DYNAMIC ポリシーによるパフォーマンスへの影響

```sql
-- VPD ポリシーのパフォーマンス・プロファイルを確認
SELECT sql_text, executions, elapsed_time/1000000 elapsed_sec,
       rows_processed
FROM v$sql
WHERE sql_text LIKE '%EMPLOYEES%'
  AND parsing_user_id != 0
ORDER BY elapsed_time DESC;

-- ポリシー・ファンクションが遅い場合は、コンテキスト検索表に索引が不足していないか確認する
-- DYNAMIC の代わりにログオン・トリガーを併用した CONTEXT_SENSITIVE を検討する
```

### 間違い 4: ビューやシノニムにもポリシーが適用されることを忘れる

VPD ポリシーは実表（ベース・テーブル）に関連付けられる。その表に対するビューやシノニムは、自動的にそのポリシーを継承する。しかし、ビューに直接ポリシーを追加することも可能であり、慎重に設計しないと「二重フィルタリング」の原因になる可能性がある。

---

## コンプライアンスの考慮事項

### HIPAA
VPD は、HIPAA の「最小必要 (minimum necessary)」標準を実現するための理想的な制御手段である。患者レコードへのアクセスを制限し、臨床医には担当の患者のみを表示させ、請求担当には請求に関連する項目のみを表示させ、事務スタッフには基本属性のみを表示させるといった制御が可能になる。

### PCI-DSS
PCI 要件 7 (業務上の必要性に基づいて会員データへのアクセスを制限する) は、VPD で直接対応可能である。会員データ表に、決済処理アプリケーションのセッション・コンテキストにのみ表示を制限するポリシーを適用できる。

### SOX
複数の法人データが混在する財務データベースにおいて、ある事業部門のユーザーが別の事業部門の財務記録を参照できないように VPD を使用して強制できる。これにより、アプリケーション・レイヤーではなくデータ・レイヤーで職務分掌の要件を満たすことができる。

```sql
-- 例: SOX 準拠のエンティティ（法人）レベルの分離
CREATE OR REPLACE FUNCTION finance.entity_isolation_policy(
  p_schema  IN VARCHAR2,
  p_object  IN VARCHAR2
) RETURN VARCHAR2 AS
  v_entity_id VARCHAR2(10);
  v_is_auditor VARCHAR2(5);
BEGIN
  v_entity_id  := SYS_CONTEXT('finance_ctx', 'entity_id');
  v_is_auditor := SYS_CONTEXT('finance_ctx', 'is_auditor');

  -- 外部監査人は法人をまたいだ可視性が必要
  IF v_is_auditor = 'TRUE' THEN
    RETURN NULL;
  END IF;

  IF v_entity_id IS NULL THEN
    RETURN '1=0';
  END IF;

  RETURN 'legal_entity_id = ' || TO_NUMBER(v_entity_id);
END entity_isolation_policy;
/
```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

-   このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効である。
-   21c、23c、または 23ai と記された機能は、Oracle Database 26ai 対応機能として扱う。バージョンが混在する環境では、19c 互換の代替案を保持すること。
-   リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンをサポートする環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

## ソース

-   [Oracle Database Security Guide 19c — Using Oracle VPD to Control Data Access](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/using-oracle-vpd-to-control-data-access.html)
-   [Oracle PL/SQL Packages Reference 19c — DBMS_RLS](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_RLS.html)
-   [Oracle Database Reference 19c — DBA_POLICIES](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_POLICIES.html)
