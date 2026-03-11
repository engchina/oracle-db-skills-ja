# ORDS メタデータ・カタログと OpenAPI ドキュメント

## 概要

ORDS は、すべての REST モジュールおよび AutoREST 有効化されたオブジェクトに対して、OpenAPI 3.0 仕様ドキュメントを自動的に生成する。このマシン・リーダブルな API 記述は、Swagger UI や Oracle Database Actions などの対話型ドキュメント・ツールの基盤となり、クライアント SDK の生成を可能にし、利用者に対して自己記述的な API カタログを提供する。このメタデータのアクセス方法、カスタマイズ方法、および活用方法を理解することは、機能的で、かつ文書化の行き届いた API を構築するために不可欠である。

---

## 自動生成される OpenAPI エンドポイント

ORDS は、複数の URL パスでメタデータを公開している。

### スキーマ・レベルのメタデータ・カタログ

スキーマ内で公開されているすべてのモジュールをカバーする OpenAPI 3.0 ドキュメントを返す。

```
GET /ords/{schema_alias}/metadata-catalog/
```

例:

```http
GET /ords/hr/metadata-catalog/ HTTP/1.1
Host: myserver.example.com
Accept: application/json
```

レスポンス: HR スキーマ内のすべての公開済み REST モジュールおよび AutoREST オブジェクトを網羅する、完全な OpenAPI 3.0 JSON ドキュメント。

### モジュール・レベルの OpenAPI ドキュメント

単一のモジュールに対する OpenAPI 3.0 ドキュメントを返す。

```
GET /ords/{schema_alias}/metadata-catalog/{module_name}/
```

例:

```http
GET /ords/hr/metadata-catalog/hr.employees/ HTTP/1.1
```

### AutoREST オブジェクトのメタデータ

個別の AutoREST 有効化されたオブジェクトに対する OpenAPI ドキュメントを返す。

```
GET /ords/{schema_alias}/metadata-catalog/{object_alias}/
```

例:

```http
GET /ords/hr/metadata-catalog/employees/ HTTP/1.1
```

### OpenAPI JSON 形式の強制用 URL

```
GET /ords/{schema_alias}/metadata-catalog/?format=json
GET /ords/{schema_alias}/open-api-catalog/
```

---

## OpenAPI 3.0 出力例

単純な `GET /employees` ハンドラーの場合、ORDS は以下を生成する。

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "HR REST API",
    "description": "従業員管理 API",
    "version": "1.0.0",
    "contact": {
      "name": "HR Platform Team"
    }
  },
  "servers": [
    {
      "url": "https://myserver.example.com/ords/hr/v1"
    }
  ],
  "paths": {
    "/employees/": {
      "get": {
        "summary": "従業員の一覧表示",
        "description": "オプションのフィルタリングを伴う、ページ分けされた従業員リストを返す",
        "operationId": "getEmployees",
        "parameters": [
          {
            "name": "offset",
            "in": "query",
            "schema": { "type": "integer", "default": 0 }
          },
          {
            "name": "limit",
            "in": "query",
            "schema": { "type": "integer", "default": 25 }
          },
          {
            "name": "q",
            "in": "query",
            "description": "JSON フィルタ・クエリ",
            "schema": { "type": "string" }
          }
        ],
        "responses": {
          "200": {
            "description": "成功レスポンス",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/EmployeeCollection"
                }
              }
            }
          }
        }
      },
      "post": {
        "summary": "新しい従業員の作成",
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": { "$ref": "#/components/schemas/Employee" }
            }
          }
        },
        "responses": {
          "201": { "description": "従業員作成完了" }
        }
      }
    },
    "/employees/{id}": {
      "get": {
        "summary": "ID による従業員の取得",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "schema": { "type": "string" }
          }
        ],
        "responses": {
          "200": { "description": "従業員発見" },
          "404": { "description": "従業員が見つかりません" }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "Employee": {
        "type": "object",
        "properties": {
          "employee_id": { "type": "integer" },
          "first_name":  { "type": "string" },
          "last_name":   { "type": "string" },
          "email":       { "type": "string" },
          "salary":      { "type": "number" }
        }
      }
    },
    "securitySchemes": {
      "OAuth2": {
        "type": "oauth2",
        "flows": {
          "clientCredentials": {
            "tokenUrl": "https://myserver.example.com/ords/hr/oauth/token",
            "scopes": {
              "hr.employees.read": "従業員データの読み取り"
            }
          }
        }
      }
    }
  }
}
```

---

## コメントによる API メタデータのカスタマイズ

ORDS は、`DEFINE_MODULE`、`DEFINE_TEMPLATE`、および `DEFINE_HANDLER` の `p_comments` パラメータを使用して、OpenAPI の `description`（説明）および `summary`（概要）フィールドを入力する。

### モジュール・レベルの説明

```sql
BEGIN
  ORDS.DEFINE_MODULE(
    p_module_name => 'hr.employees',
    p_base_path   => '/v1/',
    p_status      => 'PUBLISHED',
    p_comments    => 'HR 従業員管理 API v1。従業員レコードの CRUD 操作を提供。書き込み操作には認証が必要。'
  );
END;
/
```

### テンプレートのコメント (パスの説明にマップ)

```sql
BEGIN
  ORDS.DEFINE_TEMPLATE(
    p_module_name => 'hr.employees',
    p_pattern     => 'employees/',
    p_comments    => '従業員コレクション・リソース。q パラメータによるフィルタ、limit/offset によるページネーション、orderby パラメータによる順序付けをサポート。'
  );
END;
/
```

### ハンドラーのコメント (操作の概要/説明にマップ)

```sql
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.employees',
    p_pattern     => 'employees/',
    p_method      => 'GET',
    p_source_type => ORDS.source_type_collection_feed,
    p_comments    => 'ページ分けされた従業員リストを返す。?dept_id=N を使用して部門でフィルタ可能。任意の列による順序付けをサポート。',
    p_source      => 'SELECT * FROM employees'
  );
END;
/
```

---

## ORDS.DEFINE_PARAMETER によるパラメータの文書化

カスタム定義された REST モジュールの場合、ORDS では明示的なパラメータの文書化が可能であり、これにより OpenAPI の出力が充実する。

```sql
BEGIN
  -- dept_id クエリ・パラメータの文書化
  ORDS.DEFINE_PARAMETER(
    p_module_name        => 'hr.employees',
    p_pattern            => 'employees/',
    p_method             => 'GET',
    p_name               => 'dept_id',
    p_bind_variable_name => 'dept_id',
    p_source_type        => 'HEADER',   -- URI, HEADER, または RESPONSE
    p_param_type         => 'INT',      -- OpenAPI の型情報として使用
    p_access_method      => 'IN',
    p_comments           => '部門 ID による従業員のフィルタ'
  );

  -- id パス・パラメータの文書化
  ORDS.DEFINE_PARAMETER(
    p_module_name        => 'hr.employees',
    p_pattern            => 'employees/:id',
    p_method             => 'GET',
    p_name               => 'id',
    p_bind_variable_name => 'id',
    p_source_type        => 'URI',
    p_param_type         => 'INT',
    p_access_method      => 'IN',
    p_comments           => '一意の従業員識別子'
  );

  COMMIT;
END;
/
```

---

## OpenAPI 仕様書の Swagger UI での利用

### オプション 1: ORDS スタンドアロン経由の Swagger UI

ORDS スタンドアロンは、その `doc_root` ディレクトリから Swagger UI の静的ファイルを提供できる。

```shell
# Swagger UI ディストリビューションをダウンロード
curl -L https://github.com/swagger-api/swagger-ui/archive/v5.x.x.tar.gz | tar xz
cp -r swagger-ui-5.x.x/dist/* /opt/oracle/ords/config/ords/standalone/doc_root/swagger/
```

ORDS の doc root にリダイレクト・ページを作成する:

```html
<!-- /opt/oracle/ords/config/ords/standalone/doc_root/api-docs/index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>HR API Documentation</title>
  <meta charset="utf-8"/>
  <link rel="stylesheet" href="/swagger/swagger-ui.css">
</head>
<body>
  <div id="swagger-ui"></div>
  <script src="/swagger/swagger-ui-bundle.js"></script>
  <script>
    SwaggerUIBundle({
      url: "/ords/hr/metadata-catalog/",
      dom_id: '#swagger-ui',
      presets: [SwaggerUIBundle.presets.apis],
      layout: "BaseLayout"
    });
  </script>
</body>
</html>
```

アクセス URL: `https://myserver.example.com/api-docs/`

### オプション 2: Oracle Database Actions (標準機能)

Database Actions (旧 SQL Developer Web) が有効な場合、Rest Workshop が組み込まれており、以下の場所で自動 API ドキュメントを確認できる。

```
https://myserver.example.com/ords/sql-developer
```

「REST」→「モジュール」に移動すると、対話型 UI で REST API の表示、テスト、管理が可能である。

### オプション 3: Postman へのインポート

OpenAPI 仕様書をエクスポートし、Postman に直接インポートする。

```shell
# OpenAPI 仕様書のダウンロード
curl -o hr-api.json \
  https://myserver.example.com/ords/hr/metadata-catalog/hr.employees/

# Postman CLI を介したインポート
postman import -f hr-api.json
```

または Postman UI の「File」→「Import」から JSON ファイルを選択する。Postman は、すべてのエンドポイント、サンプル・リクエスト、およびドキュメントを含むコレクションを生成する。

---

## ORDS メタデータ REST エンドポイント

OpenAPI 以外にも、ORDS は独自のメタデータを REST 経由で公開している。

### すべてのモジュールの一覧表示

```http
GET /ords/_/db-api/stable/metadata-catalog/ HTTP/1.1
Authorization: Bearer <admin-token>
```

### モジュールの詳細

```http
GET /ords/hr/metadata-catalog/hr.employees/ HTTP/1.1
```

### ORDS のステータス確認

```http
GET /ords/_/db-api/stable/database/ HTTP/1.1
```

### 有効化されたスキーマ

```sql
-- SQL 経由 (DBA 権限が必要)
SELECT schema, url_mapping_pattern, auto_rest_auth
FROM   dba_ords_enabled_schemas
ORDER  BY schema;

-- REST API 経由 (Database API が有効な場合)
GET /ords/_/db-api/stable/database/pdbs/
```

---

## SQL による ORDS メタデータのクエリ

ORDS は、データ・ディクショナリ・ビューを通じてメタデータを公開している。

```sql
-- REST 有効化されているすべてのスキーマをリスト表示
SELECT schema, url_mapping_pattern, auto_rest_auth
FROM   dba_ords_enabled_schemas;

-- 現在のスキーマ内のすべてのモジュールをリスト表示
SELECT name, uri_prefix, status, items_per_page, comments
FROM   user_ords_modules
ORDER  BY name;

-- モジュールのすべてのテンプレートをリスト表示
SELECT module_id, uri_template, priority, etag_type
FROM   user_ords_templates
WHERE  module_id = (SELECT id FROM user_ords_modules WHERE name = 'hr.employees');

-- ソースとともにすべてのハンドラーをリスト表示
SELECT t.uri_template, h.method, h.source_type, h.source
FROM   user_ords_handlers h
JOIN   user_ords_templates t ON t.id = h.template_id
JOIN   user_ords_modules   m ON m.id = t.module_id
WHERE  m.name = 'hr.employees'
ORDER  BY t.uri_template, h.method;

-- スキーマをまたいだ検査用の DBA ビュー
SELECT s.schema, m.name, t.uri_template, h.method
FROM   dba_ords_enabled_schemas s
JOIN   dba_ords_modules         m ON m.schema = s.schema
JOIN   dba_ords_templates       t ON t.module_id = m.id
JOIN   dba_ords_handlers        h ON h.template_id = t.id
ORDER  BY s.schema, m.name;
```

---

## REST 定義のエクスポートとインポート

### SQL スクリプトによるエクスポート

```sql
-- 現在のスキーマ内のすべての REST 定義に対する、再実行可能な SQL スクリプトを生成
SELECT DBMS_METADATA.GET_DDL('ORDS', m.name)
FROM   user_ords_modules m;
```

### ORDS CLI によるエクスポート

```shell
# 指定したスキーマの REST 定義を SQL ファイルにエクスポート
ords --config /opt/oracle/ords/config rest export \
  --schema HR \
  --output /opt/backup/hr-rest-export.sql

# REST 定義のインポート
ords --config /opt/oracle/ords/config rest import \
  --schema HR \
  --input /opt/backup/hr-rest-export.sql
```

これにより、必要な `ORDS.DEFINE_MODULE`、`ORDS.DEFINE_TEMPLATE`、および `ORDS.DEFINE_HANDLER` の呼び出しを含むスクリプトが生成される。これは以下の用途に使用される。
- REST API 定義のバージョン管理
- 開発(DEV)からテスト(TEST)、本番(PROD)環境への移行
- ソース管理システムでの REST API 変更の文書化

---

## ベスト・プラクティス

- **すべての ORDS オブジェクトに `p_comments` を提供する**: 短い説明であっても、自動生成される OpenAPI 仕様書の品質が大幅に向上する。期待される入力、出力、および動作を文書化すること。
- **名前とパスにモジュールのバージョン管理を含める**: `hr.employees.v1` や `/v1/` とすることで、メタデータと OpenAPI 仕様書の両方でバージョンの変更が明確になる。
- **CI/CD での OpenAPI 仕様書のダウンロードを自動化する**: ORDS から稼働中のメタデータを取得し、コミット済みの仕様書と比較（diff）を行う。ライブの仕様書が期待値と異なる場合にアラートを出すようにする。
- **OpenAPI 仕様書をバリデーターでテストする**: `openapi-generator validate` や `spectral lint` などのツールを使用して、利用者が使用する前に仕様書の問題を特定する。
- **内部 API の metadata-catalog エンドポイントは保護しておく**: 公開されているメタデータ URL は、API 構造のすべてをさらけ出すことになる。機密性の高い API については、ORDS 権限を使用して `/metadata-catalog/` へのアクセスを制限すること。
- **すべてのクエリ・パラメータとパス・パラメータに `ORDS.DEFINE_PARAMETER` を使用する**: これにより、文書化されたパラメータに依存する Postman や Swagger UI の利用者に対して、正確な OpenAPI 仕様を提供できる。

## よくある間違い

- **metadata-catalog が未公開のモジュールも表示すると仮定する**: メタデータ・カタログに表示されるのは、`p_status => 'PUBLISHED'` が設定されたモジュールのみである。開発中で `NOT_PUBLISHED` に設定されているモジュールは（意図通り）表示されない。
- **`ORDS.DEFINE_PARAMETER` のあとにコミットを忘れる**: 他の ORDS メタデータと同様に、パラメータ定義はデータベースの表に保存されるため、明示的な COMMIT が必要である。
- **OpenAPI JSON を直接編集する**: metadata-catalog エンドポイントは、ORDS_METADATA 表から動的に生成される。JSON 出力への直接の編集は、次のリクエスト時に破棄される。代わりに元の定義を編集すること。
- **metadata-catalog の URL 内のモジュール名の大文字小文字を間違える**: `metadata-catalog/HR.Employees/` と `metadata-catalog/hr.employees/` では異なる結果が返される場合がある。モジュール名は定義された通り（多くの場合小文字）に保存される。一貫性を保つこと。
- **メタデータ・エンドポイントをヘルス・チェックとして使用する**: メタデータ・カタログはデータベースへのアクセスを必要とする。ヘルス・チェックには、レスポンスが単純でオーバーヘッドが少ない `/ords/_/db-api/stable/database/` を代わりに使用すること。

---

## ソース

- [ORDS Developer's Guide — OpenAPI and Metadata Catalog](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/developing-oracle-rest-data-services-applications.html)
- [Oracle REST Data Services PL/SQL API Reference — ORDS.DEFINE_PARAMETER](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orrst/index.html)
- [ORDS CLI Reference — rest export and rest import](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/ords-command-line-interface.html)
