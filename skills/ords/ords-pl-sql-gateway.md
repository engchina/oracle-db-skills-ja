# ORDS PL/SQL ゲートウェイ: ストアド・プロシージャの呼び出しとカスタム結果の返却

## 概要

ORDS PL/SQL ゲートウェイは、HTTP リクエストを Oracle PL/SQL ストアド・プロシージャ、パッケージ、または匿名ブロックに直接接続する。これは、以前 Oracle Application Server で `mod_plsql` ゲートウェイとともに使用されていた Oracle HTTP Server (OHS) を置き換えるものである。ORDS REST API における `plsql/block` ソース・タイプを使用すると、パッケージの呼び出し、複雑なビジネス・ロジックの実行、カスタム JSON の返却、適切な HTTP ステータス・コードによるエラー処理、および CLOB/BLOB コンテンツのストリーミングなど、Oracle PL/SQL の機能を最大限に活用できる。

各ソース・タイプをいつ使用すべきか、またパラメータ、結果セット、およびエラーをどのように適切に処理するかを理解することは、堅牢な PL/SQL ベースの REST サービスを構築するために不可欠である。

---

## PL/SQL 用のハンドラー・ソース・タイプ

ORDS は、ハンドラー内の SQL/PL/SQL がどのように実行され、結果がどのようにシリアル化（直列化）されるかを決定する、複数のソース・タイプをサポートしている。

| ソース・タイプ | 定数 | 説明 |
|---|---|---|
| `plsql/block` | `ORDS.source_type_plsql` | 匿名 PL/SQL ブロック。手動での結果処理が必要。 |
| `query` / `collection/feed` | `ORDS.source_type_collection_feed` | SQL SELECT を実行し、ページ分けされた JSON コレクションを返す。 |
| `query/one_row` / `collection/item` | `ORDS.source_type_collection_item` | SQL SELECT を実行し、単一の JSON オブジェクトを返す。 |
| `dml` | `ORDS.source_type_dml` | SQL INSERT/UPDATE/DELETE を実行。暗黙的なコミットが行われる。 |
| `query/resultset` | N/A (plsql を使用) | PL/SQL からの暗黙的な結果 (Implicit Results) を使用する。 |

---

## REST ハンドラーからのストアド・プロシージャの呼び出し

### 単純なプロシージャ呼び出し

HR スキーマにおけるストアド・プロシージャの例:

```sql
CREATE OR REPLACE PROCEDURE hr.give_raise(
  p_employee_id IN  employees.employee_id%TYPE,
  p_percentage  IN  NUMBER,
  p_new_salary  OUT employees.salary%TYPE,
  p_message     OUT VARCHAR2
) AS
  l_current_salary employees.salary%TYPE;
BEGIN
  SELECT salary INTO l_current_salary
  FROM   employees
  WHERE  employee_id = p_employee_id
  FOR UPDATE;

  UPDATE employees
  SET    salary = salary * (1 + p_percentage / 100)
  WHERE  employee_id = p_employee_id
  RETURNING salary INTO p_new_salary;

  p_message := 'Raise applied successfully';
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_new_salary := NULL;
    p_message    := 'Employee not found';
    RAISE_APPLICATION_ERROR(-20001, 'Employee ' || p_employee_id || ' not found');
END;
/
```

このプロシージャを呼び出す ORDS REST ハンドラー:

```sql
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.api',
    p_pattern     => 'employees/:id/raise',
    p_method      => 'POST',
    p_source_type => ORDS.source_type_plsql,
    p_source      => q'[
      DECLARE
        l_new_salary  employees.salary%TYPE;
        l_message     VARCHAR2(500);
      BEGIN
        hr.give_raise(
          p_employee_id => :id,            -- URI パラメータ
          p_percentage  => :percentage,    -- JSON ボディ・パラメータ
          p_new_salary  => l_new_salary,
          p_message     => l_message
        );

        -- 手動で JSON レスポンスを構築
        HTP.P('{"employee_id":' || :id ||
              ',"new_salary":' || l_new_salary ||
              ',"message":"' || l_message || '"}');
        :status_code := 200;
      END;
    ]'
  );
  COMMIT;
END;
/
```

---

## IN/OUT パラメータ・バインディング

### リクエストからの IN パラメータ

ORDS は、URI テンプレート、クエリ文字列、および JSON ボディの 3 つのソースからパラメータをバインドし、それらをすべて「名前付きバインド変数」として扱う。

```
URL: POST /ords/hr/v1/employees/101/raise
Body: {"percentage": 10}

利用可能なバインド変数:
  :id          → "101" (URI テンプレートの :id から)
  :percentage  → 10    (JSON ボディの "percentage" フィールドから)
```

すべてのバインド変数は、ソースに関係なく VARCHAR2 として渡される。必要に応じて明示的にキャストすること。

```sql
l_percentage := TO_NUMBER(:percentage);
l_emp_id     := TO_NUMBER(:id);
l_hire_date  := TO_DATE(:hire_date, 'YYYY-MM-DD"T"HH24:MI:SS"Z"');
```

### 暗黙的なパラメータを介した OUT パラメータ

ORDS は PL/SQL の OUT パラメータを自動的にシリアル化しない。以下の方法を使用すること。

1. **`:status_code`** や `:forward_location` に値を代入してレスポンスを制御する。
2. **`HTP.P`** / **`HTP.PRN`** を使用して生のレスポンス・コンテンツを書き込む。
3. **`APEX_JSON`** を使用して構造化された JSON を出力する。
4. **`DBMS_OUTPUT`** (非推奨。特別な構成が必要)。

---

## 暗黙的な結果 (Implicit Results) による結果セットの返却

Oracle 12c 以降は **暗黙的な結果** (`DBMS_SQL.RETURN_RESULT`) をサポートしており、これにより PL/SQL は ref cursor を返すことができる。ORDS はこれを検出し、自動的に JSON コレクションとしてシリアル化する。

```sql
CREATE OR REPLACE PROCEDURE hr.get_dept_employees(
  p_dept_id IN departments.department_id%TYPE
) AS
  l_cursor SYS_REFCURSOR;
BEGIN
  OPEN l_cursor FOR
    SELECT employee_id, first_name, last_name, salary
    FROM   employees
    WHERE  department_id = p_dept_id
    ORDER  BY last_name;

  DBMS_SQL.RETURN_RESULT(l_cursor);
END;
/
```

ORDS ハンドラーからの呼び出し（ソース・タイプは `plsql/block` である必要がある）:

```sql
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.api',
    p_pattern     => 'departments/:dept_id/employees/',
    p_method      => 'GET',
    p_source_type => ORDS.source_type_plsql,
    p_source      => q'[
      BEGIN
        hr.get_dept_employees(p_dept_id => :dept_id);
      END;
    ]'
  );
  COMMIT;
END;
/
```

ORDS は暗黙的な結果カーソルを検出し、JSON コレクションとしてシリアル化する。これは、直接 SQL を公開することなく、パッケージから結果セットを返却するための最もクリーンな方法である。

---

## REF CURSOR の返却

もう一つの方法は、バインド変数を通じて REF CURSOR を返却することである。

```sql
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.api',
    p_pattern     => 'employees/by-dept',
    p_method      => 'GET',
    p_source_type => ORDS.source_type_plsql,
    p_source      => q'[
      BEGIN
        OPEN :results FOR
          SELECT employee_id, first_name, last_name, salary
          FROM   employees
          WHERE  department_id = :dept_id;
      END;
    ]'
  );
  COMMIT;
END;
/
```

注: `:results` は SYS_REFCURSOR 型の OUT バインド変数である。ORDS はカーソル・バインドを認識し、自動的にシリアル化する。

---

## カスタム JSON 出力用の APEX_JSON の使用

APEX がインストールされている場合、`APEX_JSON` は PL/SQL から JSON を生成するためのクリーンな API を提供する。

```sql
DECLARE
  CURSOR c_emps IS
    SELECT e.employee_id, e.first_name, e.last_name,
           e.salary, d.department_name
    FROM   employees e
    JOIN   departments d ON d.department_id = e.department_id
    WHERE  e.department_id = :dept_id;
BEGIN
  APEX_JSON.OPEN_OBJECT;
  APEX_JSON.WRITE('department_id', :dept_id);

  APEX_JSON.OPEN_ARRAY('employees');
  FOR r IN c_emps LOOP
    APEX_JSON.OPEN_OBJECT;
    APEX_JSON.WRITE('employee_id',   r.employee_id);
    APEX_JSON.WRITE('name',          r.first_name || ' ' || r.last_name);
    APEX_JSON.WRITE('salary',        r.salary);
    APEX_JSON.WRITE('department',    r.department_name);
    APEX_JSON.CLOSE_OBJECT;
  END LOOP;
  APEX_JSON.CLOSE_ARRAY;

  APEX_JSON.CLOSE_OBJECT;
END;
```

これにより以下が生成される。

```json
{
  "department_id": "60",
  "employees": [
    {"employee_id": 103, "name": "Alexander Hunold", "salary": 9000, "department": "IT"},
    {"employee_id": 104, "name": "Bruce Ernst", "salary": 6000, "department": "IT"}
  ]
}
```

---

## 生の出力のための HTP パッケージの使用

APEX がない場合は、Oracle の `HTP` (Hypertext Procedures) パッケージを使用して、生の HTTP 出力を行うことができる。

```sql
DECLARE
  l_result CLOB;
BEGIN
  -- JSON 文字列の構築
  SELECT JSON_OBJECT(
           'employee_id' VALUE e.employee_id,
           'name'        VALUE e.first_name || ' ' || e.last_name,
           'salary'      VALUE e.salary
           RETURNING CLOB
         )
  INTO l_result
  FROM employees e
  WHERE employee_id = :id;

  -- レスポンス・ヘッダーの書き込み
  OWA_UTIL.MIME_HEADER('application/json', FALSE);
  HTP.P('Cache-Control: no-cache');
  OWA_UTIL.HTTP_HEADER_CLOSE;

  -- ボディの書き込み
  HTP.PRN(l_result);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    :status_code := 404;
END;
```

---

## エラー処理と HTTP ステータス・コード

### `:status_code` の使用

HTTP レスポンス・ステータスを制御するには、`:status_code` を設定する。

```sql
BEGIN
  -- 操作の試行
  DELETE FROM employees WHERE employee_id = :id;

  IF SQL%ROWCOUNT = 0 THEN
    :status_code := 404;  -- Not Found
  ELSE
    :status_code := 204;  -- No Content (削除成功、ボディなし)
  END IF;
EXCEPTION
  WHEN OTHERS THEN
    :status_code := 500;
    -- エラーのログ記録 (内部詳細は外部に公開しない)
    INSERT INTO api_error_log (error_time, error_msg, sql_code)
    VALUES (SYSDATE, SQLERRM, SQLCODE);
    COMMIT;
END;
```

### アプリケーション・エラーの発生

ORDS は `RAISE_APPLICATION_ERROR` を HTTP 400 レスポンスに変換する。

```sql
BEGIN
  IF :salary IS NULL OR TO_NUMBER(:salary) <= 0 THEN
    RAISE_APPLICATION_ERROR(-20100, 'Salary must be a positive number');
  END IF;

  IF TO_NUMBER(:salary) > 100000 THEN
    RAISE_APPLICATION_ERROR(-20101, 'Salary exceeds maximum allowed value');
  END IF;

  UPDATE employees SET salary = :salary WHERE employee_id = :id;
END;
```

ORDS は以下を返す。

```json
{
  "code": "Bad Request",
  "message": "Salary must be a positive number",
  "type": "tag:oracle.com,2020:error/Bad_Request",
  "instance": "tag:oracle.com,2020:ecid/..."
}
```

### JSON ボディを含むカスタム・エラー・レスポンス

エラー構造をより細かく制御する場合、例外をキャッチしてカスタム JSON を書き込む。

```sql
BEGIN
  UPDATE employees SET salary = :salary WHERE employee_id = :id;

  IF SQL%ROWCOUNT = 0 THEN
    :status_code := 404;
    HTP.PRN('{"error":"NOT_FOUND","message":"Employee ' || :id || ' does not exist"}');
    RETURN;
  END IF;

  :status_code := 200;
EXCEPTION
  WHEN VALUE_ERROR THEN
    :status_code := 400;
    HTP.PRN('{"error":"VALIDATION_ERROR","message":"Invalid salary value: must be numeric"}');
  WHEN OTHERS THEN
    :status_code := 500;
    -- 本番環境では SQLERRM をクライアントに公開してはならない
    HTP.PRN('{"error":"INTERNAL_ERROR","message":"An unexpected error occurred"}');
END;
```

---

## パラメータ映射 (Mapping Parameters)

パラメータを明示的に定義することで、期待されるデータ型や、JSON/HTTP ヘッダーへのアクセス方法を指定できる。

```sql
BEGIN
  ORDS.DEFINE_PARAMETER(
    p_module_name        => 'hr.api',
    p_pattern            => 'employees/:id',
    p_method             => 'GET',
    p_name               => 'If-None-Match',
    p_bind_variable_name => 'etag_in',
    p_source_type        => 'HEADER',
    p_access_method      => 'IN'
  );
END;
/
```

- `p_source_type`: `HEADER`, `RESPONSE`, `URI`, `QUERY`。
- `p_access_method`: `IN`, `OUT`, `INOUT`。

---

## ベスト・プラクティス

### 1. 公開スキーマではなく、アプリケーション・スキーマを使用する
ORDS ハンドラーを、データを持つ実際のスキーマ（例: `HR`）に直接定義する。プロキシ・ユーザー（`ORDS_PUBLIC_USER`）ではなく、実際のスキーマの権限でコードを実行できる。

### 2. ビジネス・ロジックを PL/SQL パッケージに置く
ハンドラー内に長い SQL コードを直接書くのではなく、パッケージ化された関数を呼び出すようにする。

```sql
-- 推奨されるハンドラー・ソース
BEGIN
  :new_id := hr_service_pkg.create_employee(
               p_first_name => :first_name,
               p_last_name  => :last_name,
               p_email      => :email
             );
  :status_code := 201;
END;
```

### 3. バージョン管理されたパス・プレフィックス
API の互換性のために、バージョン番号（例: `/v1/`, `/v2/`）をモジュールのベース・パスに含める。

### 4. 適切な HTTP メソッドの遵守
- `GET`: 情報の取得（副作用なし）
- `POST`: 新規リソースの作成
- `PUT`: 既存リソースの置換（べき等）
- `PATCH`: リソースの部分的な更新
- `DELETE`: リソースの削除

### 5. ページネーションの活用
大量のデータを返す GET エンドポイントには、常に `p_items_per_page` を設定し、`collection/feed` を使用して、リンク・ヘッダーや JSON 内のナビゲーション・リンクを提供すること。

---

## よくある間違い

- **モジュールの末尾のスラッシュを忘れる**: `p_base_path => 'hr/v1'` とすると、URL が `.../hr/v1employees/` のように崩れる可能性がある。常に `'hr/v1/'` のように指定すること。
- **データが見つからない場合の 200 OK**: 単一リソースの検索 (`collection/item`) でデータがない場合、ORDS は自動的に 404 を返すが、PL/SQL ハンドラーでは明示的に `:status_code := 404` を設定する必要がある。
- **バインド変数の型を指定しない**: すべてのバインド変数はデフォルトで VARCHAR2 として扱われる。数値や日付の計算に使用する場合は、明示的に `TO_NUMBER` や `TO_DATE` を使用すること。
- **セキュリティの欠如**: モジュールやテンプレートに対して privileges（権限）をマップしていない。すべての作成/更新エンドポイントは、必ず認証が必要なように権限をマップすること。
- **エラー・メッセージの不備**: `RAISE_APPLICATION_ERROR` を使用して、クライアントに具体的なエラー原因（例: "Invalid email format"）を伝えること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替手段を維持すること。
- 19c と 26ai の両方をサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [ORDS Developer's Guide — Developing Oracle REST Data Services Applications](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/developing-oracle-rest-data-services-applications.html)
- [Defining RESTful Services with PL/SQL API](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/restful-services-api.html)
- [ORDS PL/SQL Mapping Parameters Reference](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/implicit-parameters.html)
