# Oracle データのマスキングとリダクション (Data Masking and Redaction)

## 概要

データのマスキングとリダクションは、機密データが不正に開示されるのを防ぐための、補完的なセキュリティ制御である。これらはそれぞれ異なる目的で使用される。

-   **Oracle Data Redaction** (`DBMS_REDACT`): クエリ実行時に、データベース・エンジンからデータが出る前に、インメモリーで動的にデータをマスクする。ディスク上のデータは変更されない。これにより、不正な SQL アクセスやアプリケーション層での露出を防ぐ。
-   **Oracle Data Safe / Advanced Masking**: 非本番環境のコピーにおいて、データを恒久的に変換する。一度マスクされると、元の値は失われる。これにより、開発、テスト、および UAT（ユーザー受け入れテスト）環境における機密データを保護する。

どちらの機能も、「開発者、テスター、アナリスト、およびサードパーティ・アプリケーションに対して機密データ（PII、PCI、PHI）を公開する必要があるが、実際の値は見せてはならない」という根本的な課題に対応する。

Oracle Data Redaction には **Oracle Database Enterprise Edition 12c 以降**が必要である（12c では Advanced Security Option に含まれ、19c 以降では EE に含まれる）。Oracle Data Safe はクラウド・サービスである（OCI で無料枠が利用可能）。

---

## Oracle Data Redaction (DBMS_REDACT)

Data Redaction は、`EXEMPT REDACTION POLICY` 権限を持たないユーザーに対して、クエリ結果を透過的に変更する。変更はクエリの実行後、結果がクライアントに返される前に行われるため、索引、クエリ、および DML には影響を与えない。

### リダクション・タイプ

| タイプ | 定数 | 説明 |
|---|---|---|
| 完全 (Full) | `DBMS_REDACT.FULL` | 値全体を、データ型に応じたデフォルト値に置き換える（数値は 0、文字列は空白、日付はエポック） |
| 部分 (Partial) | `DBMS_REDACT.PARTIAL` | 値の一部をマスクし、形式を維持する |
| 正規表現 (Regular Expression) | `DBMS_REDACT.REGEXP` | 正規表現に一致するパターンを置き換える |
| ランダム (Random) | `DBMS_REDACT.RANDOM` | クエリごとに異なるランダムな値を返す |
| なし (None) | `DBMS_REDACT.NONE` | ポリシーは存在するがリダクションは行われない（ポリシーの一時停止に使用） |

---

## 完全リダクション (Full Redaction)

完全リダクションは、値全体をデフォルト値に置き換える。数値は `0`、文字列は単一の空白 `' '`、日付は `01-JAN-01` になる。

```sql
-- 列全体をリダクション
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'HR',
    object_name         => 'EMPLOYEES',
    column_name         => 'SALARY',
    policy_name         => 'REDACT_SALARY_FULL',
    function_type       => DBMS_REDACT.FULL,
    expression          => '1=1'   -- 常に適用。条件付きの場合は SYS_CONTEXT を使用
  );
END;
/

-- EXEMPT REDACTION POLICY 権限を持たないユーザーの結果:
-- SELECT salary FROM hr.employees WHERE employee_id = 100;
-- SALARY
-- 0

-- 日付列をリダクション
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'PATIENTS',
    object_name         => 'MEDICAL_RECORDS',
    column_name         => 'DATE_OF_BIRTH',
    policy_name         => 'REDACT_DOB',
    function_type       => DBMS_REDACT.FULL,
    expression          => '1=1'
  );
END;
/
```

---

## 部分リダクション (Partial Redaction)

部分リダクションは、元の値の形式と一部の文字を維持し、機密部分のみをマスクする。これは、クレジットカード番号、社会保障番号 (SSN)、電話番号などに最適である。

```sql
-- クレジットカード: 下 4 桁のみを表示 (XXXX-XXXX-XXXX-1234)
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'PAYMENTS',
    object_name         => 'TRANSACTIONS',
    column_name         => 'CREDIT_CARD_NUMBER',
    policy_name         => 'REDACT_CC_PARTIAL',
    function_type       => DBMS_REDACT.PARTIAL,
    function_parameters => 'VVVVFVVVVFVVVVFLLL,VVVV-VVVV-VVVV-LLLL,*,1,12',
    -- 形式: 入力形式, 出力形式, マスク文字, 開始位置, 終了位置
    expression          => '1=1'
  );
END;
/
-- 結果: ****-****-****-1234

-- 社会保障番号 (SSN): 下 4 桁のみを表示
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'HR',
    object_name         => 'EMPLOYEES',
    column_name         => 'SSN',
    policy_name         => 'REDACT_SSN',
    function_type       => DBMS_REDACT.PARTIAL,
    function_parameters => 'VVVFVVFVVVV,VVV-VV-VVVV,*,1,6',
    -- 1文字目から6文字目までをマスクし、7文字目から11文字目（下4桁）を表示
    expression          => '1=1'
  );
END;
/
-- 結果: ***-**-1234

-- メールアドレス: ローカル部分をマスクし、ドメインを表示
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'CUSTOMERS',
    object_name         => 'ACCOUNTS',
    column_name         => 'EMAIL_ADDRESS',
    policy_name         => 'REDACT_EMAIL',
    function_type       => DBMS_REDACT.PARTIAL,
    function_parameters => 'VVVVVVVVVVVVVVVVVVVV@VVVVVVVVVVVVVVVVVVVV,VVVV@VVVVVVVVVVVVVVVVVVVV,X,5,20',
    expression          => '1=1'
  );
END;
/
-- 結果: johnXXXXXXXXXXXX@example.com (おおよそのイメージ)

-- 電話番号: 中間の数字をマスク
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'HR',
    object_name         => 'EMPLOYEES',
    column_name         => 'PHONE_NUMBER',
    policy_name         => 'REDACT_PHONE',
    function_type       => DBMS_REDACT.PARTIAL,
    function_parameters => 'VVVFVVVFVVVV,VVV-VVV-VVVV,*,5,7',
    expression          => '1=1'
  );
END;
/
-- 結果: 515-***-1234
```

### 部分リダクションの形式パラメータの説明

`PARTIAL` リダクションの `function_parameters` 文字列は、5 つの要素で構成される。
```
'入力形式, 出力形式, マスク文字, 開始位置, 終了位置'
```
-   形式内の `V`: 可変位置（データ依存の文字）
-   形式内の `F`: 固定区切り文字
-   形式内の `L`: そのまま保持するリテラル文字（マスクされない）
-   `開始位置` と `終了位置`: マスクする範囲（1 から始まる位置）

---

## 正規表現リダクション (Regular Expression Redaction)

正規表現リダクションは最も柔軟なオプションであり、`REGEXP_REPLACE` スタイルのパターマッチングを使用してデータを動的にマスクする。

```sql
-- フリーテキストのメモ列に含まれるすべてのメールアドレスをリダクション
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'SUPPORT',
    object_name         => 'TICKETS',
    column_name         => 'DESCRIPTION',
    policy_name         => 'REDACT_EMAIL_IN_TEXT',
    function_type       => DBMS_REDACT.REGEXP,
    regexp_pattern      => '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}',
    regexp_replace_string => '***@***.***',
    regexp_position     => 1,      -- 文字列内の開始位置
    regexp_occurrence   => 0,      -- 0 = かなべての一致箇所
    regexp_match_parameter => 'i', -- 大文字小文字を区別しない
    expression          => '1=1'
  );
END;
/

-- 米国の SSN パターン (###-##-####) をリダクション
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema         => 'HR',
    object_name           => 'EMPLOYEE_NOTES',
    column_name           => 'NOTE_TEXT',
    policy_name           => 'REDACT_SSN_PATTERN',
    function_type         => DBMS_REDACT.REGEXP,
    regexp_pattern        => '\d{3}-\d{2}-\d{4}',
    regexp_replace_string => '***-**-****',
    regexp_occurrence     => 0,
    expression            => '1=1'
  );
END;
/

-- ログ表内の IP アドレスをリダクション
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema         => 'APP',
    object_name           => 'ACCESS_LOG',
    column_name           => 'CLIENT_IP',
    policy_name           => 'REDACT_IP',
    function_type         => DBMS_REDACT.REGEXP,
    regexp_pattern        => '\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}',
    regexp_replace_string => '*.*.*.*',
    regexp_occurrence     => 0,
    expression            => '1=1'
  );
END;
/
```

---

## ランダム・リダクション (Random Redaction)

ランダム・リダクションは、クエリごとにランダムに生成された値を返す。データ型は維持される。これは、本物らしく見えるが捏造されたデータを見せても構わない場合に有用である。

```sql
-- ランダムな給料値（SELECT ごとに異なる）
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    column_name     => 'SALARY',
    policy_name     => 'REDACT_SALARY_RANDOM',
    function_type   => DBMS_REDACT.RANDOM,
    expression      => '1=1'
  );
END;
/
-- SELECT を実行するたびに、異なるランダムな数値が返される
```

---

## 条件付きリダクション (選択的適用)

`expression` パラメータは、行単位（およびセッション単位）で評価される SQL 条件である。これが TRUE と評価された場合にのみ、リダクションが適用される。これがロールベースまたはコンテキストベースのリダクションを実現するメカニズムである。

```sql
-- セッション・ユーザーが HR ロールを持っていない場合のみ給料をリダクション
-- アプリケーション・コンテキストを事前にセットアップする必要がある (row-level-security.md を参照)
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'HR',
    object_name         => 'EMPLOYEES',
    column_name         => 'SALARY',
    policy_name         => 'CONDITIONAL_SALARY_REDACT',
    function_type       => DBMS_REDACT.FULL,
    -- ユーザーが HR_MGR または PAYROLL ロールでない場合にリダクションを適用
    expression          => 'SYS_CONTEXT(''hr_ctx'', ''user_role'') NOT IN (''HR_MGR'', ''PAYROLL'')'
  );
END;
/

-- データベース・セッション・ユーザーに基づくリダクション（USERENV を使用した簡単なアプローチ）
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'PAYMENTS',
    object_name         => 'TRANSACTIONS',
    column_name         => 'CREDIT_CARD_NUMBER',
    policy_name         => 'CC_REDACT_BY_USER',
    function_type       => DBMS_REDACT.PARTIAL,
    function_parameters => 'VVVVFVVVVFVVVVFLLL,VVVV-VVVV-VVVV-LLLL,*,1,12',
    -- payment_processor サービス・アカウント以外の全員に適用
    expression          => 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') != ''PAYMENT_PROCESSOR'''
  );
END;
/
```

---

## リダクション・ポリシーの管理

```sql
-- 既存のポリシーに列を追加
BEGIN
  DBMS_REDACT.ALTER_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'REDACT_SALARY_FULL',
    action          => DBMS_REDACT.ADD_COLUMN,
    column_name     => 'COMMISSION_PCT',
    function_type   => DBMS_REDACT.FULL
  );
END;
/

-- 既存の列のリダクション・タイプを変更
BEGIN
  DBMS_REDACT.ALTER_POLICY(
    object_schema       => 'HR',
    object_name         => 'EMPLOYEES',
    policy_name         => 'REDACT_SALARY_FULL',
    action              => DBMS_REDACT.MODIFY_COLUMN,
    column_name         => 'SALARY',
    function_type       => DBMS_REDACT.PARTIAL,
    function_parameters => 'NNNNNNNNNN,NNNNNNNNNN,*,1,5'
  );
END;
/

-- ポリシーから特定の列を削除
BEGIN
  DBMS_REDACT.ALTER_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'REDACT_SALARY_FULL',
    action          => DBMS_REDACT.DROP_COLUMN,
    column_name     => 'COMMISSION_PCT'
  );
END;
/

-- ポリシーを無効化（リダクションを一時停止）
-- DBMS_REDACT.DISABLE_POLICY は独立したプロシージャであり、ALTER_POLICY のアクション定数ではない
BEGIN
  DBMS_REDACT.DISABLE_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'REDACT_SALARY_FULL'
  );
END;
/

-- ポリシーを再度有効化
BEGIN
  DBMS_REDACT.ENABLE_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'REDACT_SALARY_FULL'
  );
END;
/

-- ポリシーを完全に削除
BEGIN
  DBMS_REDACT.DROP_POLICY(
    object_schema   => 'HR',
    object_name     => 'EMPLOYEES',
    policy_name     => 'REDACT_SALARY_FULL'
  );
END;
/
```

### リダクション・ポリシーのクエリ

```sql
-- すべてのリダクション・ポリシーとその対象列
SELECT object_schema, object_name, policy_name, column_name,
       function_type, function_parameters, regexp_pattern,
       expression, enable
FROM redaction_policies
ORDER BY object_schema, object_name, policy_name;

-- 別のビュー（旧式）
SELECT object_schema, object_name, policy_name, column_name,
       function_type, expression, policy_enable
FROM dba_redaction_policies
ORDER BY object_schema, object_name;

-- 特定のスキーマ内のリダクション対象列をすべて検索
SELECT object_name, column_name, function_type, expression
FROM redaction_policies
WHERE object_schema = 'HR'
ORDER BY object_name, column_name;
```

---

## 非本番環境向けの恒久的なデータ・マスキング

非本番環境のコピーでは、開発者、テスター、またはサードパーティーに提供する前に、データを恒久的に変換する必要がある。Oracle はいくつかの方法を提供している。

### Oracle Data Safe (クラウド)

Oracle Data Safe は、OCI でホストされているデータベース向けの無料クラウド・サービスであり、オンプレミスのデータベース向けのアドオンでもある。以下の機能を提供する。
- 機密情報の発見（PII、PCI、PHI を自動的に検索）
- 50 以上の定義済みマスキング形式を使用したデータ・マスキング
- アクティビティ監査
- セキュリティ・アセスメント

### 手動による恒久的なマスキング・テクニック

Oracle Data Safe が利用できない場合、リフレッシュ・サイクル中に SQL で恒久的なマスキングを実装できる。

```sql
-- ステップ 1: 機密列を特定する
-- ステップ 2: 非本番環境でマスクされたコピーを作成する

-- 値をシャッフルする（分布は維持するが、再識別を阻止する）
UPDATE hr.employees e1
SET salary = (
  SELECT e2.salary
  FROM hr.employees e2
  WHERE ROWNUM = 1
  ORDER BY DBMS_RANDOM.VALUE
);
COMMIT;

-- 名前をルックアップ表からのランダムな名前に置き換える
UPDATE hr.employees
SET first_name = (SELECT name FROM name_lookup ORDER BY DBMS_RANDOM.VALUE FETCH FIRST 1 ROWS ONLY),
    last_name  = (SELECT surname FROM surname_lookup ORDER BY DBMS_RANDOM.VALUE FETCH FIRST 1 ROWS ONLY);
COMMIT;

-- メールアドレスをハッシュ化する（一貫性を持たせる: 同じ入力からは常に同じ出力が生成される）
UPDATE customers
SET email = LOWER(RAWTOHEX(DBMS_CRYPTO.HASH(
  UTL_RAW.CAST_TO_RAW(LOWER(email)),
  DBMS_CRYPTO.HASH_SH256
))) || '@masked.example.com';
COMMIT;

-- 範囲内で日付をランダム化する（相対的な順序を維持する）
UPDATE patients
SET date_of_birth = date_of_birth + TRUNC(DBMS_RANDOM.VALUE(-365, 365));
COMMIT;

-- 非常に機密性の高いフィールドを完全に NULL にする
UPDATE hr.employees
SET national_id     = NULL,
    passport_number = NULL,
    biometric_data  = NULL;
COMMIT;

-- クレジットカード番号を一貫性のある形式維持した値に置き換える
UPDATE payments
SET credit_card_number = '4111' || LPAD(TRUNC(DBMS_RANDOM.VALUE(0, 999999999999)), 12, '0');
COMMIT;
```

### 非本番リフレッシュ向けのサブセット化およびマスキング・パイプライン

```sql
-- 1. 本番データの一部のみをエクスポート（PII を含まない表にはマスキング不要）
-- 2. 機密表の場合は、INSERT...SELECT を使用してインラインでマスキングを行う:
INSERT INTO nonprod.employees (
  employee_id, first_name, last_name, email, phone_number,
  hire_date, job_id, salary, department_id
)
SELECT
  employee_id,
  'First' || employee_id                              AS first_name,
  'Last'  || employee_id                              AS last_name,
  'user'  || employee_id || '@test.example.com'       AS email,
  '555-' || LPAD(employee_id, 3, '0') || '-0000'     AS phone_number,
  hire_date,   -- 非機密。そのまま保持
  job_id,      -- 非機密。そのまま保持
  ROUND(DBMS_RANDOM.VALUE(30000, 150000))             AS salary,
  department_id
FROM prod.employees;
COMMIT;
```

---

## 機密列の発見

マスキングを行う前に、どの列に機密データが含まれているかを知る必要がある。Oracle Data Safe はこれを自動化するが、手動でパターンベースの検索を行うこともできる。

```sql
-- すべての表を対象に、機密データと思われる列名を検索
SELECT owner, table_name, column_name, data_type
FROM dba_tab_columns
WHERE REGEXP_LIKE(column_name,
  'SSN|SOCIAL.SECURITY|NATIONAL.ID|PASSPORT|CREDIT.CARD|CARD.NUMBER|' ||
  'CVV|CVC|ACCOUNT.NUMBER|ROUTING|SALARY|WAGE|COMMISSION|BONUS|' ||
  'DATE.OF.BIRTH|DOB|BIRTH.DATE|GENDER|RACE|ETHNICITY|RELIGION|' ||
  'DIAGNOSIS|MEDICAL|HEALTH|PATIENT|PRESCRIPTION|' ||
  'PASSWORD|SECRET|TOKEN|API.KEY|PRIVATE.KEY|' ||
  'EMAIL|PHONE|MOBILE|ADDRESS|ZIP|POSTAL',
  'i'
)
AND owner NOT IN ('SYS', 'SYSTEM', 'CTXSYS', 'MDSYS', 'ORDSYS', 'XDB')
ORDER BY owner, table_name, column_name;
```

---

## ベスト・プラクティス

1.  **可能な限り低いレベルでリダクションを行う**: ポリシーはビューではなくベース表に適用する。リダクション対象の表に基づくビューは、自動的にリダクションを継承する。

2.  **ロールベースのリダクションには条件式を使用する**: `expression => '1=1'` のポリシーは常にリダクションを行う。`SYS_CONTEXT` 式を使用して、特権ロールが実際のデータを見られるようにする。

3.  **リダクションのみに頼らない**: リダクションは多層防御の一環である。直接エクスポート・ツール（Data Pump、SQL*Plus SPOOL）やバックアップへのアクセス権を持つユーザーは、リダクションを回避できる可能性がある。権限管理および監査と組み合わせること。

4.  **本番環境の実際のデータを非本番環境で使用しない**: 本番環境でリダクションが行われていても、非本番環境のコピーには恒久的にマスクされたデータを使用すべきである。ログ、エラー・メッセージ、デバッグ・ツールなどを通じて機密データが漏洩する経路は多すぎる。

5.  **リダクションされた列へのアクセスを監査する**: リダクション・ポリシーがある表に対してファイングレイン監査（`auditing.md` を参照）を有効にし、異常なアクセス・パターンを検出できるようにする。

6.  **マスキング戦略を文書化する**: コンプライアンスのために、特定の表の特定の列がマスクされていることを示し、使用されているマスキング・テクニックを説明できる必要がある。

7.  **特権のないアカウントでリダクションをテストする**: マスクされたデータを見るべきユーザーとして接続し、マスキングの動作が正しいことを常に確認する。

---

## よくある間違いとその回避方法

### 間違い 1: エクスポート・ツールがリダクションを尊重すると想定する

Data Pump (`expdp`)、SQL*Plus `COPY`、データベース・リンク、および GoldenGate はすべて、DBA レベルの権限で実行されるか、直接パス・アクセスを使用するため、Data Redaction をバイパスできる。

```sql
-- EXEMPT REDACTION POLICY を持っている（すべてのデータを非マスクで見ることができる）ユーザーを確認
SELECT grantee, privilege
FROM dba_sys_privs
WHERE privilege = 'EXEMPT REDACTION POLICY'
ORDER BY grantee;

-- EXP_FULL_DATABASE または DBA ロールを持つユーザーも、Data Pump 経由でリダクションをバイパスする。
-- エクスポート結果は暗号化で保護すること（encryption.md を参照）
```

### 間違い 2: リダクションは推論を防げない

列が完全にリダクションされていても、その列の索引を使用したバイナリ検索が可能な場合、攻撃者は二分探索を使用して実際の値を推論できる。リダクションをアクセス制御と組み合わせること。

### 間違い 3: expression パラメータをテストしていない

式が正しく書かれていないと、ポリシーが適用されなかったり（全員が本物のデータを見ることができる）、常に適用されたり（特権ユーザーもマスクされたデータしか見られない）することがある。

```sql
-- 式のロジックを直接テストする
SELECT
  CASE
    WHEN SYS_CONTEXT('hr_ctx', 'user_role') NOT IN ('HR_MGR', 'PAYROLL')
    THEN 'REDACTED'
    ELSE 'VISIBLE'
  END AS salary_visibility
FROM dual;
```

---

## コンプライアンスの考慮事項

### PCI-DSS
-   要件 3.3: 表示されるときに PAN（プライマリ・アカウント番号）をマスクする。最低限、最初の 6 桁と最後の 4 桁のみを表示できる。
-   要件 3.4: PAN が保存される場所はどこでも読み取り不能にする。
-   Data Redaction は表示時のマスキング（要件 3.3）を直接満たす。
-   要件 3.4 には、ストレージ保護のための暗号化（`encryption.md` を参照）が必要である。

```sql
-- PCI 準拠のクレジットカード表示マスキング
BEGIN
  DBMS_REDACT.ADD_POLICY(
    object_schema       => 'PAYMENTS',
    object_name         => 'CARD_DATA',
    column_name         => 'PAN',
    policy_name         => 'PCI_PAN_MASK',
    function_type       => DBMS_REDACT.PARTIAL,
    -- 最初 6 桁と下 4 桁を表示し、真ん中 6 桁をマスク
    function_parameters => 'VVVVVVFVVVVVVVFVVVV,VVVVVVFVVVVVVVFVVVV,*,7,12',
    expression          => 'SYS_CONTEXT(''USERENV'',''SESSION_USER'') != ''PCI_PROCESSOR'''
  );
END;
/
```

### HIPAA
-   Safe Harbor 脱識別標準では、18 個の特定の識別子（名前、住所、日付、電話番号、SSN、医療記録番号など）を削除またはマスクする必要がある。
-   Data Redaction は、クエリ時のアクセス制御としてこれを満たす。
-   外部に共有されるデータについては、恒久的なマスキングが必要である。

### GDPR
-   個人データは、その規定された目的のために必要な範囲を超えて利用可能であってはならない。
-   Data Redaction は、データベース層での目的制限の実施に役立つ。
-   消去権については、動的なリダクションではなく、恒久的な削除または恒久的なマスキングが必要である。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

-   このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効である。
-   21c、23c、または 23ai と記された機能は、Oracle Database 26ai 対応機能として扱う。バージョンが混在する環境では、19c 互換の代替案を保持すること。
-   リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンをサポートする環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

## ソース

-   [Oracle PL/SQL Packages Reference 19c — DBMS_REDACT](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_REDACT.html)
-   [Oracle Database Advanced Security Guide 19c — Configuring Oracle Data Redaction Policies](https://docs.oracle.com/en/database/oracle/oracle-database/19/asoag/configuring-oracle-data-redaction-policies.html)
-   [Oracle Database Reference 19c — REDACTION_POLICIES](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/REDACTION_POLICIES.html)
-   [Oracle Data Safe Documentation](https://docs.oracle.com/en/cloud/paas/data-safe/index.html)
