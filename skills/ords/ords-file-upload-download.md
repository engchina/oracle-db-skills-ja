# ORDS へのファイルのアップロードとロード: REST による BLOB 処理

## 概要

ORDS は、HTTP を介したファイルのアップロードとダウンロードをネイティブにサポートしている。ファイルは Oracle Database の BLOB 列に保存され、適切な MIME タイプ・ヘッダーとともに REST エンドポイントを介して提供される。このパターンは、ドキュメント管理システム、画像リポジトリ、レポート・アーカイブ、およびリレーショナル・メタデータとともにバイナリ・コンテンツを保存および取得する必要があるすべてのアプリケーションで使用されている。

ORDS は BLOB データを透過的に処理する。アップロードされたファイル・コンテンツは、暗黙的なパラメータである `:body` を介して BLOB として PL/SQL ハンドラーにバインドされ、ダウンロードされるコンテンツは、適切な `Content-Type` ヘッダーとともにデータベースから直接ストリーミングされる。

---

## データベースのセットアップ: ファイル保存用の表

```sql
-- 汎用ドキュメント保存用の表
CREATE TABLE hr.documents (
  document_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  employee_id    NUMBER          REFERENCES hr.employees(employee_id),
  filename       VARCHAR2(255)   NOT NULL,
  content_type   VARCHAR2(100)   NOT NULL,
  file_size      NUMBER,
  file_content   BLOB,
  description    VARCHAR2(1000),
  uploaded_by    VARCHAR2(100),
  upload_date    TIMESTAMP WITH TIME ZONE DEFAULT SYSTIMESTAMP,
  CONSTRAINT uk_docs_emp_file UNIQUE (employee_id, filename)
);

-- パフォーマンス向上のための SECUREFILE LOB ストレージの使用
-- (新しい表の作成時に指定する)
CREATE TABLE hr.document_store (
  doc_id         NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  filename       VARCHAR2(255) NOT NULL,
  content_type   VARCHAR2(100) NOT NULL,
  file_content   BLOB,
  created_at     TIMESTAMP DEFAULT SYSTIMESTAMP
)
LOB (file_content) STORE AS SECUREFILE (
  ENABLE STORAGE IN ROW
  COMPRESS MEDIUM
  DEDUPLICATE
  CACHE
);
```

---

## REST を介したファイルのアップロード

### `:body` と `:content_type` の理解

HTTP クライアントがバイナリ・ボディを含むリクエストを送信した場合:
- **`:body`** — 生のリクエスト・ボディ（BLOB 形式）。これがファイル・コンテンツになる。
- **`:body_text`** — 生のリクエスト・ボディ（CLOB 形式、テキスト・ペイロード用）。
- **`:content_type`** — `Content-Type` リクエスト・ヘッダーの値。

これらは、すべての `plsql/block` ハンドラーで使用可能な ORDS の暗黙的なバインド・パラメータである。

### 単純なバイナリ・アップロード (生のボディを使用した PUT/POST)

最も単純なアップロード・パターン: HTTP ボディそのものがファイルである場合。

```sql
-- アップロード・ハンドラーの定義
BEGIN
  ORDS.DEFINE_MODULE(
    p_module_name => 'hr.docs',
    p_base_path   => '/docs/',
    p_status      => 'PUBLISHED'
  );

  ORDS.DEFINE_TEMPLATE(
    p_module_name => 'hr.docs',
    p_pattern     => 'employees/:employee_id/documents/:filename'
  );

  -- PUT: ドキュメントのアップロード/置換
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.docs',
    p_pattern     => 'employees/:employee_id/documents/:filename',
    p_method      => 'PUT',
    p_source_type => ORDS.source_type_plsql,
    p_mimes_allowed => '',  -- すべてのコンテンツ・タイプを受け入れる
    p_source      => q'[
      DECLARE
        l_doc_id   hr.documents.document_id%TYPE;
        l_file_size NUMBER := DBMS_LOB.GETLENGTH(:body);
      BEGIN
        -- ファイル・サイズの検証 (10MB 制限)
        IF l_file_size > 10 * 1024 * 1024 THEN
          :status_code := 413;  -- Payload Too Large
          RETURN;
        END IF;

        -- ドキュメントのアップサート (挿入または更新)
        BEGIN
          INSERT INTO hr.documents (
            employee_id, filename, content_type,
            file_size, file_content, uploaded_by, upload_date
          ) VALUES (
            :employee_id, :filename, :content_type,
            l_file_size, :body,
            :current_user, SYSTIMESTAMP
          )
          RETURNING document_id INTO l_doc_id;

          :status_code := 201;
        EXCEPTION
          WHEN DUP_VAL_ON_INDEX THEN
            UPDATE hr.documents
            SET    file_content  = :body,
                   content_type  = :content_type,
                   file_size     = l_file_size,
                   uploaded_by   = :current_user,
                   upload_date   = SYSTIMESTAMP
            WHERE  employee_id = :employee_id
            AND    filename    = :filename
            RETURNING document_id INTO l_doc_id;

            :status_code := 200;
        END;

        COMMIT;
        :forward_location := 'employees/' || :employee_id ||
                             '/documents/' || :filename;
      END;
    ]'
  );
  COMMIT;
END;
/
```

curl を使用したアップロード・リクエストの例:

```shell
# PDF ドキュメントのアップロード
curl -X PUT \
  https://myserver.example.com/ords/hr/docs/employees/101/documents/resume.pdf \
  -H "Content-Type: application/pdf" \
  -H "Authorization: Bearer <token>" \
  --data-binary @/path/to/resume.pdf

# 画像のアップロード
curl -X PUT \
  https://myserver.example.com/ords/hr/docs/employees/101/documents/photo.jpg \
  -H "Content-Type: image/jpeg" \
  --data-binary @/path/to/photo.jpg
```

### マルチパート・フォーム・データによるアップロード (Multipart Form Data)

HTML フォーム・ベースのアップロード（ブラウザが `multipart/form-data` を送信する場合）、ハンドラーはマルチパート・ボディの全データを `:body` で受信する。ORDS はネイティブではマルチパートの境界解析を行わないため、`APEX_WEB_SERVICE` やカスタムの PL/SQL パーサーを使用するか、最新のクライアントからは（上述の）直接的なバイナリ・アップロードを使用することが推奨される。

APEX ベースのファイル・アップロードの場合、APEX は独自のフレームワークを介してマルチパート解析を自動的に処理する。

```sql
-- マルチパート・アップロード用のハンドラー (APEX またはカスタム解析が必要)
ORDS.DEFINE_HANDLER(
  p_module_name   => 'hr.docs',
  p_pattern       => 'upload/',
  p_method        => 'POST',
  p_source_type   => ORDS.source_type_plsql,
  p_mimes_allowed => 'multipart/form-data',
  p_source        => q'[
    DECLARE
      l_body_text CLOB  := :body_text;
      l_body_blob BLOB  := :body;
      l_ctype     VARCHAR2(200) := :content_type;
      -- コンテンツ・タイプから境界 (boundary) を抽出
      l_boundary  VARCHAR2(100);
    BEGIN
      -- コンテンツ・タイプは次のような形式: multipart/form-data; boundary=----WebKitFormBoundary...
      l_boundary := SUBSTR(l_ctype, INSTR(l_ctype, 'boundary=') + 9);

      -- 本番環境では、APEX_WEB_SERVICE.PARSE_MULTIPART または
      -- カスタムのマルチパート・パーサー・パッケージを使用すること
      -- 以下は簡易的な例証:
      :status_code := 200;
    END;
  ]'
);
```

**推奨**: ブラウザ・ベースのファイル・アップロードには APEX を使用すること。サービスからのプログラムによるアップロードには、ファイルを生のボディとして送信する直接的なバイナリ PUT/POST アプローチを使用すること。

---

## REST を介したファイル (BLOB) のダウンロード

### `source_type_media` の使用

`media` ソース・タイプは、バイナリ・コンテンツを返すために特別に設計されている。SQL は以下を返す必要がある。
1. BLOB または CLOB 列 (ファイル・コンテンツ)
2. オプション: `content_type` 列 (MIME タイプ)
3. オプション: `content_filename` 列 (Content-Disposition 用)
4. オプション: `etag` 列 (HTTP キャッシュ用)

```sql
BEGIN
  ORDS.DEFINE_TEMPLATE(
    p_module_name => 'hr.docs',
    p_pattern     => 'employees/:employee_id/documents/:filename/content'
  );

  -- GET: ファイル・コンテンツのダウンロード
  ORDS.DEFINE_HANDLER(
    p_module_name => 'hr.docs',
    p_pattern     => 'employees/:employee_id/documents/:filename/content',
    p_method      => 'GET',
    p_source_type => ORDS.source_type_media,
    p_source      => q'[
      SELECT d.file_content  AS blob,
             d.content_type  AS "Content-Type",
             d.filename      AS "Content-Disposition-Filename",
             d.upload_date   AS "Last-Modified"
      FROM   hr.documents d
      WHERE  d.employee_id = :employee_id
      AND    d.filename    = :filename
    ]'
  );
  COMMIT;
END;
/
```

ORDS は以下を自動的に行う。
- `content_type` 列から `Content-Type` ヘッダーを設定
- `Content-Length` ヘッダーを追加
- BLOB ボディをクライアントにストリーミング
- クエリが行を返さない場合、404 を返却

ダウンロードの例:

```shell
# ドキュメントのダウンロード
curl -O -J \
  https://myserver.example.com/ords/hr/docs/employees/101/documents/resume.pdf/content \
  -H "Authorization: Bearer <token>"
```

`-J` フラグは、サーバーから提供された Content-Disposition 内のファイル名を使用するように curl に指示する。

### ダウンロード用の PL/SQL ハンドラーの使用 (より詳細な制御が必要な場合)

カスタム・ヘッダー、条件付きダウンロード、または変換が必要なシナリオの場合:

```sql
ORDS.DEFINE_HANDLER(
  p_module_name => 'hr.docs',
  p_pattern     => 'employees/:employee_id/documents/:filename/download',
  p_method      => 'GET',
  p_source_type => ORDS.source_type_plsql,
  p_source      => q'[
    DECLARE
      l_blob         BLOB;
      l_ctype        VARCHAR2(100);
      l_filename     VARCHAR2(255);
      l_size         INTEGER;
      l_offset       INTEGER := 1;
      l_chunk_size   INTEGER := 32767;
      l_chunk        RAW(32767);
      l_amount       INTEGER;
    BEGIN
      BEGIN
        SELECT d.file_content, d.content_type, d.filename
        INTO   l_blob, l_ctype, l_filename
        FROM   hr.documents d
        WHERE  d.employee_id = :employee_id
        AND    d.filename    = :filename;
      EXCEPTION
        WHEN NO_DATA_FOUND THEN
          :status_code := 404;
          RETURN;
      END;

      l_size := DBMS_LOB.GETLENGTH(l_blob);

      -- レスポンス・ヘッダーの設定
      OWA_UTIL.MIME_HEADER(l_ctype, FALSE);
      HTP.P('Content-Length: ' || l_size);
      HTP.P('Content-Disposition: attachment; filename="' || l_filename || '"');
      HTP.P('Cache-Control: private, max-age=3600');
      OWA_UTIL.HTTP_HEADER_CLOSE;

      -- 大きなファイルを処理するために BLOB をチャンクに分けてストリーミング
      WHILE l_offset <= l_size LOOP
        l_amount := LEAST(l_chunk_size, l_size - l_offset + 1);
        DBMS_LOB.READ(l_blob, l_amount, l_offset, l_chunk);
        WPG_DOCLOAD.DOWNLOAD_FILE(l_chunk);
        l_offset := l_offset + l_amount;
      END LOOP;

    END;
  ]'
);
```

---

## ドキュメントのリスト表示 (コンテンツを含まないメタデータ)

```sql
ORDS.DEFINE_TEMPLATE(
  p_module_name => 'hr.docs',
  p_pattern     => 'employees/:employee_id/documents/'
);

ORDS.DEFINE_HANDLER(
  p_module_name    => 'hr.docs',
  p_pattern        => 'employees/:employee_id/documents/',
  p_method         => 'GET',
  p_source_type    => ORDS.source_type_collection_feed,
  p_items_per_page => 25,
  p_source         => q'[
    SELECT d.document_id,
           d.filename,
           d.content_type,
           d.file_size,
           d.description,
           d.uploaded_by,
           d.upload_date,
           -- ダウンロード・リンクの構築
           '/ords/hr/docs/employees/' || d.employee_id
             || '/documents/' || d.filename || '/content' AS download_url
    FROM   hr.documents d
    WHERE  d.employee_id = :employee_id
    ORDER  BY d.upload_date DESC
  ]'
);
```

---

## MIME タイプの処理

BLOB と一緒に MIME タイプをデータベースに保存する。アップロード時にファイル拡張子を MIME タイプにマッピングする:

```sql
CREATE OR REPLACE FUNCTION hr.get_mime_type(p_filename IN VARCHAR2) RETURN VARCHAR2 AS
  l_ext VARCHAR2(20);
BEGIN
  l_ext := LOWER(SUBSTR(p_filename, INSTR(p_filename, '.', -1) + 1));

  RETURN CASE l_ext
    WHEN 'pdf'  THEN 'application/pdf'
    WHEN 'doc'  THEN 'application/msword'
    WHEN 'docx' THEN 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
    WHEN 'xls'  THEN 'application/vnd.ms-excel'
    WHEN 'xlsx' THEN 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    WHEN 'jpg'  THEN 'image/jpeg'
    WHEN 'jpeg' THEN 'image/jpeg'
    WHEN 'png'  THEN 'image/png'
    WHEN 'gif'  THEN 'image/gif'
    WHEN 'txt'  THEN 'text/plain'
    WHEN 'csv'  THEN 'text/csv'
    WHEN 'json' THEN 'application/json'
    WHEN 'xml'  THEN 'application/xml'
    WHEN 'zip'  THEN 'application/zip'
    ELSE 'application/octet-stream'
  END;
END;
/
```

アップロード・ハンドラーでは、クライアントから提供された Content-Type を使用するが、提供されない場合はファイル名ベースの検出にフォールバックする:

```sql
l_mime := NVL(
  NULLIF(:content_type, 'application/octet-stream'),
  hr.get_mime_type(:filename)
);
```

---

## Content-Disposition: インライン表示 vs 添付ファイル

ブラウザがファイルをインラインで表示するか、ダウンロードを促すかを制御する:

```sql
-- 画像や PDF の場合 (ブラウザ内でインライン表示)
HTP.P('Content-Disposition: inline; filename="' || l_filename || '"');

-- その他のすべてのファイルの場合 (ダウンロードを強制)
HTP.P('Content-Disposition: attachment; filename="' || l_filename || '"');

-- 非 ASCII 文字のファイル名用の RFC 5987 エンコーディング
HTP.P('Content-Disposition: attachment; filename*=UTF-8'''' ||
      UTL_URL.ESCAPE(l_filename, TRUE, 'UTF-8'));
```

---

## 大容量ファイルのストリーミング

メモリ容量を超える大容量ファイルの場合は、常にチャンク（分割）してストランキングを行う。上述の `DBMS_LOB.READ` と `WPG_DOCLOAD.DOWNLOAD_FILE` を使用したチャンク方式は、これを正しく処理する。`source_type_media` アプローチの場合、ORDS が自動的にストランキングを行う。

大容量アップロードのための ORDS 構成（JVM メモリとリクエスト制限の調整）:

```shell
# ORDS CLI を使用してページネーションとリクエスト制限を調整
ords --config /opt/oracle/ords/config config set misc.pagination.maxRows 1000
```

Jetty スタンドアロンでは、デフォルトの最大リクエスト・サイズは 200MB である。より大きなファイルの場合、分割アップロード（クライアントがファイルを分割してアップロードし、サーバー側で組み立てる）を検討すること。

---

## ドキュメントの削除

```sql
ORDS.DEFINE_HANDLER(
  p_module_name => 'hr.docs',
  p_pattern     => 'employees/:employee_id/documents/:filename',
  p_method      => 'DELETE',
  p_source_type => ORDS.source_type_plsql,
  p_source      => q'[
    BEGIN
      DELETE FROM hr.documents
      WHERE  employee_id = :employee_id
      AND    filename    = :filename;

      IF SQL%ROWCOUNT = 0 THEN
        :status_code := 404;
      ELSE
        COMMIT;
        :status_code := 204;  -- No Content
      END IF;
    END;
  ]'
);
```

---

## ベスト・プラクティス

- **可能な限りダウンロードには `source_type_media` を使用する**: 最もクリーンで効率的なアプローチである。ORDS がすべてのストランキングとヘッダー管理を自動的に処理する。
- **保存前に常にファイル・サイズを検証する**: `DBMS_LOB.GETLENGTH(:body)` を使用して、許容される最大サイズを超えていないか確認する。サイズ超過のアップロードに対しては 413 (Payload Too Large) を返すこと。
- **MIME タイプをデータベースに保存する**: ダウンロード時にファイル拡張子のみに頼らないこと。アップロード・リクエスト時の `Content-Type` を保存し、ダウンロード時にそれを使用すること。
- **SECUREFILE LOB ストレージを使用する**: SECUREFILE は、LOB 列に対して圧縮、重複排除、および暗号化を提供する。大規模なファイル保存において BASICFILE よりも大幅に効率的である。
- **適切なキャッシュ・ヘッダーを設定する**: めったに変更されない静的コンテンツ（ロゴ、PDF など）には `Cache-Control: max-age=86400` を追加する。動的なドキュメントには `Cache-Control: private, no-cache` を使用する。
- **許可されるファイル・タイプを制限する**: アップロード時に `content_type` を許可リストに対して検証する。危険なタイプ（`application/x-executable` や、ユーザー・アップロード・コンテンツとしての `text/html` など）は拒否すること。

## よくある間違い

- **バイナリ・ファイルに `:body_text` を使用する**: バイナリ・データ（画像、PDF など）には `:body` (BLOB) を使用する必要がある。バイナリ・データに `:body_text` (CLOB) を使用すると、文字セット変換によってデータが破損する。
- **ダウンロード時に `Content-Type` を設定していない**: Content-Type ヘッダーがないと、ブラウザはタイプを推測するため、間違ったビューアーで表示されたり、ファイルを開くための間違ったアプリケーションが促されたりする可能性がある。
- **アップロード・ハンドラーでのコミット忘れ**: PL/SQL ハンドラーは自動コミットされない。COMMIT を行わずに BLOB の INSERT/UPDATE を行うと、行がロックされたままになり、他のセッションからデータが参照できなくなる。
- **BLOB 全体をメモリにロードする**: `SELECT file_content INTO l_blob FROM ...` は BLOB ポインタ全体をロードするが、実際のデータはアクセスされるまで完全には読み込まれない。OOM（メモリ不足）エラーを避けるため、大容量ファイルには `DBMS_LOB.READ` をループ内で使用してストランキングすること。
- **ヘッダー内のファイル名に特殊文字を含める**: Content-Disposition ヘッダー内のファイル名にスペース、アクセント記号、または非 ASCII 文字が含まれる場合は、RFC 5987 エンコードが必要である。単純に `filename="résumé.pdf"` と記述すると、一部のクライアントで文字化けする。
- **新しい表に BASICFILE LOB を使用する**: BASICFILE はレガシーである。新しい表の BLOB 列には、より優れたパフォーマンスとストレージ・オプションを利用できる `STORE AS SECUREFILE` を常に指定すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替手段を維持すること。
- 19c と 26ai の両方をサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [ORDS Developer's Guide — Handling BLOBs and Binary Data](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/developing-oracle-rest-data-services-applications.html)
- [Oracle Database SecureFiles and Large Objects Developer's Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/adlob/index.html)
- [ORDS Implicit Parameters Reference (:body, :content_type)](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/implicit-parameters.html)
