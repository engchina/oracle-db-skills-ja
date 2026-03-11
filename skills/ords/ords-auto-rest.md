# ORDS AutoREST: 表、ビュー、およびプロシージャの自動 REST 有効化

## 概要

AutoREST は、Oracle Database オブジェクトを REST エンドポイントとして公開するための、ORDS のコード不要（ゼロコード）のアプローチである。単一の PL/SQL 呼び出しを行うだけで、ORDS は表やビューに対して、コレクションの GET（フィルタリング、ページネーション、順序付けを含む）、個別のアイテムの GET、POST、PUT、および DELETE を含む、完全な CRUD エンドポイントを自動的に生成する。AutoREST は、ラピッド・プロトタイピング、内部ツール、および表に対する標準的な CRUD 操作で十分な状況に最適である。複雑なビジネス・ロジックが必要な場合は、カスタム REST API (`ORDS.DEFINE_MODULE`) の方が適している。

---

## スキーマでの AutoREST の有効化

個々のオブジェクトで AutoREST を有効にする前に、スキーマ自体を REST 有効化する必要がある。これにより、スキーマが ORDS に登録され、URL の別名（エイリアス）が作成される。

```sql
-- スキーマ所有者または DBA として接続
BEGIN
  ORDS.ENABLE_SCHEMA(
    p_enabled             => TRUE,
    p_schema              => 'HR',          -- DB ユーザー名 (大文字小文字を区別しない)
    p_url_mapping_type    => 'BASE_PATH',
    p_url_mapping_pattern => 'hr',          -- URL の別名: /ords/hr/
    p_auto_rest_auth       => FALSE         -- FALSE = デフォルトでパブリック(公開)
  );
  COMMIT;
END;
/
```

パラメータ:
- `p_url_mapping_pattern`: URL 内のパス・セグメント。`hr` の場合、`/ords/hr/` となる。
- `p_auto_rest_auth`: TRUE の場合、デフォルトですべての AutoREST エンドポイントに認証が必要になる。FALSE の場合、権限が明示的に付与されない限り、エンドポイントは公開される。

スキーマを無効化する場合:

```sql
BEGIN
  ORDS.ENABLE_SCHEMA(
    p_enabled  => FALSE,
    p_schema   => 'HR'
  );
  COMMIT;
END;
/
```

---

## 各オブジェクトでの AutoREST の有効化

### 表とビュー

```sql
-- EMPLOYEES 表で AutoREST を有効化
BEGIN
  ORDS.ENABLE_OBJECT(
    p_enabled        => TRUE,
    p_schema         => 'HR',
    p_object         => 'EMPLOYEES',
    p_object_type    => 'TABLE',           -- または 'VIEW'
    p_object_alias   => 'employees',       -- URL パス・セグメント
    p_auto_rest_auth => FALSE
  );
  COMMIT;
END;
/
```

この呼び出しの後、ORDS は直ちに以下のエンドポイントを提供し始める（スキーマの別名が `hr` の場合）:

```
GET    /ords/hr/employees/          → コレクションを返す (ページネーションあり)
GET    /ords/hr/employees/{id}      → 主キーにより単一のアイテムを返す
POST   /ords/hr/employees/          → 新しい行を挿入する
PUT    /ords/hr/employees/{id}      → 行の完全な更新 (置換)
DELETE /ords/hr/employees/{id}      → 行を削除する
```

アイテム URL 内の `{id}` は、表の主キー列に対応する。複合主キーの場合、ORDS はそれらを URL 内でエンコードする。

### ビュー (Views)

ビューに対する AutoREST は、GET（コレクションおよびアイテム）エンドポイントを提供する。POST/PUT/DELETE は、そのビューが更新可能であるか、`INSTEAD OF` トリガーが定義されている場合にのみ機能する。

```sql
BEGIN
  ORDS.ENABLE_OBJECT(
    p_enabled        => TRUE,
    p_schema         => 'HR',
    p_object         => 'EMP_DETAILS_VIEW',
    p_object_type    => 'VIEW',
    p_object_alias   => 'emp-details',
    p_auto_rest_auth => FALSE
  );
  COMMIT;
END;
/
```

### プロシージャと関数

AutoREST は、PL/SQL プロシージャや関数を REST エンドポイントとして公開できる。

```sql
BEGIN
  ORDS.ENABLE_OBJECT(
    p_enabled        => TRUE,
    p_schema         => 'HR',
    p_object         => 'GET_EMPLOYEE_DETAILS',
    p_object_type    => 'PROCEDURE',
    p_object_alias   => 'get-emp-details',
    p_auto_rest_auth => FALSE
  );
  COMMIT;
END;
/
```

これにより、`/ords/hr/get-emp-details/` に POST エンドポイントが作成され、JSON ボディがプロシージャのパラメータとして渡される。

---

## 生成されるエンドポイントの URL パターン

スキーマの別名 `hr` の下で、別名 `employees` が付与された `HR.EMPLOYEES` 表の場合:

| 操作 | HTTP メソッド | URL | 説明 |
|---|---|---|---|
| 全件取得 | GET | `/ords/hr/employees/` | ページ分けされたコレクション |
| 1件取得 | GET | `/ords/hr/employees/101` | 主キーによる単一行 |
| 挿入 | POST | `/ords/hr/employees/` | 新しい行を作成 |
| 更新 | PUT | `/ords/hr/employees/101` | 既存の行を置換 |
| 削除 | DELETE | `/ords/hr/employees/101` | 行を削除 |
| メタデータ | GET | `/ords/hr/metadata-catalog/employees/` | このオブジェクトの OpenAPI 定義 |

---

## HTTP リクエストとレスポンスの例

### コレクションの取得 (GET)

```http
GET /ords/hr/employees/ HTTP/1.1
Host: myserver.example.com
Accept: application/json
```

```json
{
  "items": [
    {
      "employee_id": 100,
      "first_name": "Steven",
      "last_name": "King",
      "email": "SKING",
      "hire_date": "1987-06-17T00:00:00Z",
      "salary": 24000,
      "links": [
        {
          "rel": "self",
          "href": "https://myserver.example.com/ords/hr/employees/100"
        }
      ]
    },
    {
      "employee_id": 101,
      "first_name": "Neena",
      "last_name": "Yang",
      "email": "NYANG",
      "hire_date": "1989-09-21T00:00:00Z",
      "salary": 17000,
      "links": [...]
    }
  ],
  "hasMore": true,
  "limit": 25,
  "offset": 0,
  "count": 25,
  "links": [
    { "rel": "self",  "href": "https://myserver.example.com/ords/hr/employees/" },
    { "rel": "first", "href": "https://myserver.example.com/ords/hr/employees/?offset=0&limit=25" },
    { "rel": "next",  "href": "https://myserver.example.com/ords/hr/employees/?offset=25&limit=25" }
  ]
}
```

### 単一アイテムの取得 (GET)

```http
GET /ords/hr/employees/101 HTTP/1.1
Host: myserver.example.com
```

```json
{
  "employee_id": 101,
  "first_name": "Neena",
  "last_name": "Yang",
  "salary": 17000,
  "department_id": 90,
  "links": [
    { "rel": "self", "href": "https://myserver.example.com/ords/hr/employees/101" },
    { "rel": "edit", "href": "https://myserver.example.com/ords/hr/employees/101" },
    { "rel": "delete", "href": "https://myserver.example.com/ords/hr/employees/101" },
    { "rel": "collection", "href": "https://myserver.example.com/ords/hr/employees/" }
  ]
}
```

### 挿入 (POST)

```http
POST /ords/hr/employees/ HTTP/1.1
Host: myserver.example.com
Content-Type: application/json

{
  "employee_id": 210,
  "first_name": "Alice",
  "last_name": "Chen",
  "email": "ACHEN",
  "hire_date": "2024-01-15T00:00:00Z",
  "job_id": "IT_PROG",
  "salary": 9000,
  "department_id": 60
}
```

```http
HTTP/1.1 201 Created
Location: https://myserver.example.com/ords/hr/employees/210
Content-Type: application/json

{
  "employee_id": 210,
  "first_name": "Alice",
  ...
}
```

### 更新 (PUT)

```http
PUT /ords/hr/employees/210 HTTP/1.1
Content-Type: application/json

{
  "salary": 9500
}
```

### 削除 (DELETE)

```http
DELETE /ords/hr/employees/210 HTTP/1.1
```

```http
HTTP/1.1 200 OK
```

---

## `q` パラメータによるフィルタリング (JSON フィルタ・構文)

AutoREST は、`q` クエリ・パラメータを介した、JSON ベースの強力なクエリ構文をサポートしている。

### 基本的な一致 (Equality)

```http
GET /ords/hr/employees/?q={"department_id":90}
```

URL エンコードした場合:

```
/ords/hr/employees/?q=%7B%22department_id%22%3A90%7D
```

### 比較演算子

```json
// より大きい (Greater than)
{"salary": {"$gt": 10000}}

// 以下 (Less than or equal)
{"salary": {"$lte": 15000}}

// 等しくない (Not equal)
{"job_id": {"$ne": "IT_PROG"}}

// 範囲指定 (10000 から 20000 の間)
{"salary": {"$between": [10000, 20000]}}
```

### 文字列一致

```json
// LIKE パターン (% ワイルドカード)
{"last_name": {"$like": "K%"}}

// 大文字小文字を区別しない LIKE
{"last_name": {"$ilike": "k%"}}

// IN リスト
{"department_id": {"$in": [60, 90, 110]}}
```

### 論理演算子

```json
// AND (複数のキーがある場合のデフォルト)
{"department_id": 60, "salary": {"$gt": 5000}}

// 明示的な AND
{"$and": [{"department_id": 60}, {"salary": {"$gt": 5000}}]}

// OR
{"$or": [{"department_id": 60}, {"department_id": 90}]}

// NOT
{"$not": {"job_id": "AD_PRES"}}
```

### 例: 複雑なフィルタ

```http
GET /ords/hr/employees/?q={"$and":[{"department_id":{"$in":[60,90]}},{"salary":{"$gt":8000}}]}&orderby=salary%20DESC&limit=10
```

---

## ページネーション (Pagination)

AutoREST では、`limit` と `offset` クエリ・パラメータを使用したオフセット・ベースのページネーションを使用する。

```http
# ページ 1: 最初の 10 レコード
GET /ords/hr/employees/?limit=10&offset=0

# ページ 2: 次の 10 レコード
GET /ords/hr/employees/?limit=10&offset=10

# ページ 3
GET /ords/hr/employees/?limit=10&offset=20
```

レスポンスには常に以下が含まれる。
- `"hasMore"`: 現在のページ以降にレコードが存在する場合に `true`
- `"count"`: 現在のレスポンス内のアイテム数
- `"limit"`: 適用されたリミット（上限）
- `"offset"`: 適用されたオフセット（開始位置）
- `"links"`: `self`、`first`、`next` (`hasMore` が true の場合)、`prev` (`offset` > 0 の場合) へのリンク

最大リミット: ORDS のデフォルトは 1 回のリクエストにつき最大 10,000 行。必要に応じて ORDS 構成で変更可能。

### ページネーション・レスポンスの一部（例）

```json
{
  "items": [...],
  "hasMore": true,
  "limit": 25,
  "offset": 0,
  "count": 25,
  "links": [
    { "rel": "self",  "href": "/ords/hr/employees/?limit=25&offset=0" },
    { "rel": "first", "href": "/ords/hr/employees/?limit=25&offset=0" },
    { "rel": "next",  "href": "/ords/hr/employees/?limit=25&offset=25" }
  ]
}
```

---

## 並べ替え (Ordering)

ソート順を制御するには、`orderby` クエリ・パラメータを使用する。

```http
# 単一列の昇順
GET /ords/hr/employees/?orderby=last_name

# 単一列の降順
GET /ords/hr/employees/?orderby=salary%20DESC

# 複数列の指定
GET /ords/hr/employees/?orderby=department_id%20ASC,salary%20DESC
```

---

## 公開する列の制御

デフォルトでは、AutoREST は表またはビューのすべての列を公開する。列を制限するには、主に 2 つのオプションがある。

### オプション 1: 表の代わりにビューを公開する

必要な列のみを含むビューを作成し、そのビューを AutoREST 有効化する。

```sql
CREATE VIEW hr.employees_public AS
  SELECT employee_id, first_name, last_name, email,
         hire_date, job_id, department_id
  FROM hr.employees;
-- salary、commission_pct などは含まれない

BEGIN
  ORDS.ENABLE_OBJECT(
    p_enabled     => TRUE,
    p_schema      => 'HR',
    p_object      => 'EMPLOYEES_PUBLIC',
    p_object_type => 'VIEW',
    p_object_alias => 'employees'
  );
  COMMIT;
END;
/
```

### オプション 2: カスタム REST ハンドラー (きめ細かな制御に推奨)

明示的な SELECT リストを指定して `ORDS.DEFINE_HANDLER` を使用する（詳細は `ords-rest-api-design.md` を参照）。

---

## オブジェクト表の AutoREST

AutoREST はオブジェクト表（Oracle のオブジェクト・リレーショナル型）をサポートしている。JSON 表現はネストされたオブジェクト構造に従う。深くネストされた型や複雑な型の場合、クリーンな JSON 出力を得るためにカスタム・ハンドラーが必要になる場合がある。

---

## AutoREST ステータスの確認

```sql
-- 現在のスキーマで AutoREST 有効化されているオブジェクトを表示
SELECT object_name, object_type, object_alias, auto_rest_auth
FROM user_ords_enabled_objects
ORDER BY object_name;

-- REST 有効化されているスキーマを表示 (DBA ビュー)
SELECT schema, url_mapping_pattern, auto_rest_auth
FROM dba_ords_enabled_schemas;
```

---

## ベスト・プラクティス

- **スキーマ名と異なるスキーマ別名を使用する**: 公開 URL を内部の DB ユーザー名から切り離すことができる。スキーマ名が変更されても、URL の安定性を維持できる。
- **公開データ以外では `p_auto_rest_auth => TRUE` を設定する**: オブジェクトに対するすべての AutoREST エンドポイントに認証を強制する。特定のアクセスを許可するには、ORDS 権限を追加する。
- **表の直接公開よりもビューの使用を優先する**: ビューを使用すると、列の制御、行レベルのフィルタ適用、データの結合、API 用のクリーンな列名への変更などが可能になり、REST 固有のコードを書く必要がない。
- **内部ツールやプロトタイピングに AutoREST を使用する**: カスタムのビジネス・ロジック、バリデーション（検証）、または複雑な変換を伴う本番用 API には、明示的に REST ハンドラーを定義すること。
- **フィルタ・クエリは最初に SQL でテストする**: `q` パラメータは内部的に SQL の WHERE 句に変換される。クライアント・コードに実装する前に、IDE で同等の SQL をテストして結果を確認すること。
- **クライアント・コードで適切な `limit` のデフォルトを設定する**: ORDS のデフォルトは 1 ページあたり 25 アイテムである。クライアント側で常にページネーションを処理し、1 つのレスポンスですべてのレコードが返されると仮定してはならない。

## よくある間違い

- **`ORDS.ENABLE_OBJECT` の後にコミットを忘れる**: ORDS メタデータの表は標準的なデータベースの表である。コミットしない限り、変更はロールバックされ、エンドポイントは表示されない。
- **読み取りアクセスのみが必要な場合に、表に対して直接 AutoREST を有効化する**: これにより、POST/PUT/DELETE エンドポイントが作成され、脆弱性になる可能性がある。ビュー（通常は更新不可）を使用するか、権限を付与するか、`p_auto_rest_auth => TRUE` を設定すること。
- **`q` パラメータの URL エンコードの問題**: クエリ文字列内の JSON は URL エンコードされている必要がある。波括弧、引用符、コロンは特殊文字である。クライアント・コードでは常に `q` の値を URL エンコードすること。
- **`{id}` が常に列名と一致すると仮定する**: URL セグメントは列名の `id` ではなく、主キーの値にマップされる。主キー列が `employee_id` の場合、正しい URL は `/employees/101` であり、`/employees/?employee_id=101` ではない。
- **AutoREST を介して機密性の高い列 (パスワード、PII など) を含む表を公開する**: デフォルトですべての列が公開される。機密データを含む表で AutoREST を有効化する前に、表の構造を監査すること。
- **主キーが NULL の場合を考慮していない**: 主キーが NULL の行にはアイテム URL でアクセスできない。表に `NOT NULL` の主キー制約があることを確認すること。

---

## ソース

- [ORDS Developer's Guide — AutoREST](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/developing-oracle-rest-data-services-applications.html)
- [Oracle REST Data Services PL/SQL API Reference — ORDS.ENABLE_OBJECT](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orrst/index.html)
- [ORDS REST API Collection Query Syntax (q parameter)](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/implicit-parameters.html)
