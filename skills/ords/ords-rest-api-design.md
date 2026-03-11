# ORDS REST API 設計: モジュール、テンプレート、ハンドラー、および完全な API 構築

## 概要

AutoREST はテーブルやビューに対してコードなしでエンドポイントを提供できるが、`ORDS.DEFINE_MODULE`、`ORDS.DEFINE_TEMPLATE`、および`ORDS.DEFINE_HANDLER` を使用したカスタム REST API 設計では、URL 構造、SQL/PL/SQL ロジック、入力検証、出力形式、および HTTP セマンティクスを完全に制御できる。これは、複雑なビジネス・ロジック、複数テーブルのアクティビティ、計算されたレスポンス、またはカスタムの認証/認可動作を伴う本番環境の API に適した手法である。

---

## 3 レベルの API 階層

```
ORDS.DEFINE_MODULE     → API 名前空間とベース URL パス
  └── ORDS.DEFINE_TEMPLATE  → URL パターン (オプションで URI パラメータを含む)
        └── ORDS.DEFINE_HANDLER  → HTTP メソッド + SQL または PL/SQL ソース
```

これら 3 つの定義はすべて `ORDS_METADATA` に保存され、特定のスキーマに関連付けられる。

---

## ステップ 1: REST モジュールの定義

モジュールは、関連する複数のテンプレートを論理的にグループ化したものである。

```sql
BEGIN
  ORDS.DEFINE_MODULE(
    p_module_name    => 'hr.api',
    p_base_path      => 'hr/v1/',       -- URL: /ords/hr/v1/
    p_items_per_page => 25,
    p_status         => 'PUBLISHED',
    p_comments       => 'Human Resources REST API'
  );
  COMMIT;
END;
/
```

- `p_base_path`: スキーマ・マッピングの後の URL の一部。常に末尾にスラッシュをつけること。
- `p_status`: `PUBLISHED` または `NOT_PUBLISHED`。

---

## ステップ 2: URL テンプレートの定義

テンプレートは、モジュールのベース・パスに相対的な URL パターンを定義する。

```sql
BEGIN
  ORDS.DEFINE_TEMPLATE(
    p_module_name => 'hr.api',
    p_pattern     => 'employees/',      -- URL: /ords/hr/v1/employees/
    p_priority    => 0,
    p_etag_type   => 'HASH',            -- 自動的な並行性制御
    p_comments    => 'List of all employees'
  );

  -- URI パラメータを持つテンプレート
  ORDS.DEFINE_TEMPLATE(
    p_module_name => 'hr.api',
    p_pattern     => 'employees/:id',   -- URL: /ords/hr/v1/employees/101
    p_comments    => 'Individual employee lookup'
  );
  COMMIT;
END;
/
```

- `:id`: 名前付きバインド変数としてハンドラー内でアクセス可能な URI パラメータ。
- `@` 文字（例: `employees/@me`）は、優先順位を制御するために静的な特殊パスとして使用できる。

---

## ステップ 3: HTTP ハンドラーの定義

各テンプレートには、1 つ以上の HTTP メソッド（GET, POST, PUT, DELETE）を持たせることができる。

### GET ハンドラー (JSON の返却)

```sql
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.api',
    p_pattern     => 'employees/',
    p_method      => 'GET',
    p_source_type => ORDS.source_type_collection_feed, -- 自動 JSON シリアル化
    p_source      => 'SELECT * FROM employees ORDER BY employee_id',
    p_items_per_page => 10
  );
  COMMIT;
END;
/
```

### POST ハンドラー (データの作成)

```sql
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.api',
    p_pattern     => 'employees/',
    p_method      => 'POST',
    p_source_type => ORDS.source_type_plsql,
    p_source      => q'[
      BEGIN
        INSERT INTO employees (first_name, last_name, email, hire_date, job_id, salary)
        VALUES (:first_name, :last_name, :email, SYSDATE, :job_id, :salary)
        RETURNING employee_id INTO :id;

        :status_code      := 201;
        :forward_location := :id; -- 作成されたリソースへの URL を返す
      END;
    ]'
  );
  COMMIT;
END;
/
```

---

## バインド変数の使用

ORDS は、複数のソースから透過的にバインド変数を受け入れる。

1. **URI パラメータ**: テンプレート・パターンの `:id`。
2. **クエリ・パラメータ**: URL の `?dept_id=60` は、ハンドラー内の `:dept_id` としてアクセス可能。
3. **JSON ボディ**: POST/PUT リクエストの `{"salary": 5000}` は、`:salary` としてアクセス可能。
4. **暗黙的なパラメータ**:
   - `:status_code`: HTTP 応答コードを設定。
   - `:body`: PL/SQL ブロックから生の応答ボディを送信。
   - `:current_user`: 認証されたユーザー名。
   - `:forward_location`: `Location` ヘッダーを設定。

---

## 戻り値の型 (Source Types)

| ソース・タイプ | 説明 |
|---|---|
| `collection/feed` | SQL クエリを実行し、ページ分けされた JSON 階層 (`items`, `links`, `hasMore`) を返す。 |
| `collection/item` | SQL クエリの単一行を JSON オブジェクトとして返す。行が見つからない場合は 404 を返す。 |
| `plsql/block` | PL/SQL コードを実行。`HTP` パッケージ、`APEX_JSON`、または暗黙的な結果 (`RETURN_RESULT`) を使用してデータを返す場合に適している。 |
| `media` | BLOB または CLOB を生のコンテンツ（画像、PDF、CSV など）として返す。 |

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
