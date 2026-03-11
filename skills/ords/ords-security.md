# ORDS セキュリティ: 強固な保護、CORS、レート制限、および多層防御

## 概要

ORDS のセキュリティ保護には、HTTPS 通信の強制、ブラウザ・クライアント向けの CORS ポリシーの構成、OAuth2 権限によるエンドポイントの保護、バインド変数の正しい使用による SQL インジェクションの防止、データベース資格証明の安全な管理、および不審なアクティビティの監視など、複数のレイヤーが含まれる。ORDS は Oracle Database の機能を HTTP 経由で公開するため、重要なセキュリティ境界となる。誤った構成は機密データの漏洩や不正なデータ変更を招く可能性がある。

このガイドでは、Oracle Database のセキュリティ機能（VPD、Vault、監査）を補完する、ORDS 固有のセキュリティ制御について説明する。

---

## HTTPS の強制

開発環境以外では、決して ORDS を平文の HTTP で公開してはならない。HTTP リクエストを拒否またはリダイレクトするように ORDS を構成する。

### ORDS 構成での HTTPS 強制

```shell
# すべてのリクエストに HTTPS を要求
ords --config /opt/oracle/ords/config config set security.forceHTTPS true
```

`security.forceHTTPS = true` の場合、ORDS は以下を行う。
- HTTP リクエストを HTTPS にリダイレクトする (301)
- レスポンスに `Strict-Transport-Security` ヘッダーを追加する

### ORDS スタンドアロンでの HTTPS (本番用証明書)

Let's Encrypt や商用 CA の証明書を使用する場合:

```shell
# PEM 証明書を Java キーストア用の PKCS12 に変換
openssl pkcs12 -export \
  -in /etc/ssl/certs/api.example.com.crt \
  -inkey /etc/ssl/private/api.example.com.key \
  -certfile /etc/ssl/certs/chain.crt \
  -out /opt/oracle/ords/config/ords/standalone/api.p12 \
  -name ords-ssl \
  -passout pass:changeit

# ORDS での使用構成
ords --config /opt/oracle/ords/config config set standalone.https.port 443
ords --config /opt/oracle/ords/config config set standalone.https.cert \
  /opt/oracle/ords/config/ords/standalone/api.p12
ords --config /opt/oracle/ords/config config set \
  standalone.https.cert.secret changeit
```

### TLS 最小バージョンと暗号スイート

JVM オプションを設定して、廃止された TLS バージョンを無効化する。

```shell
# /etc/systemd/system/ords.service
[Service]
Environment="JAVA_OPTS=-Djdk.tls.disabledAlgorithms=SSLv3,TLSv1,TLSv1.1,RC4,DES,MD5withRSA,DH keySize<2048"
```

または Nginx (推奨されるリバース・プロキシ手法) で設定する。

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
add_header Strict-Transport-Security "max-age=63072000" always;
```

---

## CORS 構成

CORS (Cross-Origin Resource Sharing) は、どのブラウザ・オリジンが ORDS エンドポイントを呼び出すことを許可するかを制御する。CORS の誤設定（認証済みエンドポイントでのワイルドカード `*` など）は、一般的な脆弱性である。

### ORDS での CORS 構成

CORS 設定は ORDS CLI を介して行う。

```shell
# 特定のオリジンを許可
ords --config /opt/oracle/ords/config config set security.requestValidationFunction ords_util.authorize_plsql_gateway

# 許可されたオリジンの設定 (カンマ区切り。認証済みエンドポイントにはワイルドカードを使用しないこと)
ords --config /opt/oracle/ords/config config set security.allowedCORSOrigins \
  "https://app.example.com,https://admin.example.com"
```

ORDS CLI ですべての CORS パラメータを設定する。

```shell
ords --config /opt/oracle/ords/config config set \
  security.allowedCORSOrigins "https://app.example.com,https://admin.example.com"
ords --config /opt/oracle/ords/config config set \
  security.allowedCORSHeaders "Authorization,Content-Type,X-Requested-With"
ords --config /opt/oracle/ords/config config set \
  security.allowedCORSMethods "GET,POST,PUT,DELETE,OPTIONS"
ords --config /opt/oracle/ords/config config set security.maxAge 3600
```

### CORS のベスト・プラクティス

```shell
# 間違い: 認証済み API でのワイルドカード — あらゆるオリジンが資格証明を送信することを許可してしまう
ords config set security.allowedCORSOrigins "*"

# 正解: 明示的に信頼されたオリジンのみを指定
ords config set security.allowedCORSOrigins "https://myapp.example.com"

# 完全に公開された読み取り専用 API の場合はワイルドカードも許容されるが、
# 書き込み操作はすべて OAuth2 で保護すること
ords config set security.allowedCORSOrigins "*"
```

`security.forceHTTPS = true` の場合、CORS は HTTPS オリジンのみを許可する。HTTP/HTTPS が混在するオリジンは拒否される。

---

## 権限保護されたハンドラー vs 公開ハンドラー

すべての ORDS REST ハンドラーは、公開（認証不要）または保護（Bearer トークンが必要）のいずれかである。

### 公開ハンドラー (権限設定なし)

```sql
-- このエンドポイントは認証なしでアクセス可能
BEGIN
  ORDS.DEFINE_HANDLER(
    p_module_name => 'public.api',
    p_pattern     => 'status/',
    p_method      => 'GET',
    p_source_type => ORDS.source_type_collection_item,
    p_source      => 'SELECT ''OK'' AS status, SYSDATE AS timestamp FROM DUAL'
  );
  COMMIT;
END;
/
```

### 保護されたハンドラー (権限が必要)

```sql
BEGIN
  -- 権限の定義
  ORDS.DEFINE_PRIVILEGE(
    p_privilege_name => 'hr.employees.read',
    p_label          => 'Read HR Employee Data',
    p_description    => 'Required for accessing HR employee endpoints'
  );

  -- モジュールの保護
  ORDS.PRIVILEGE_MAP_MODULE(
    p_privilege_name => 'hr.employees.read',
    p_module_name    => 'hr.employees'
  );
  COMMIT;
END;
/
```

有効なトークンなしで `hr.employees` モジュールにアクセスすると、以下が返される。

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="hr"

{
  "code": "Unauthorized",
  "message": "Unauthorized"
}
```

### ハンドラー内での現在ユーザーの確認

```sql
-- :current_user には認証済みユーザー名が含まれる (未認証の場合は NULL)
WHERE employee_id = :id
AND   owner_username = :current_user  -- 行レベルのアクセス制御
```

---

## SQL インジェクションの防止

SQL インジェクションに対する ORDS の主要な防御策は **バインド・パラメータ** である。ORDS はユーザーから提供された値を SQL 文字列に結合することは決してなく、すべてのリクエスト・パラメータを変数としてバインドする。

### 正解: バインド・パラメータ

```sql
-- 安全: employee_id はバインド変数
SELECT * FROM employees WHERE employee_id = :id

-- 安全: VARCHAR2 バインド変数
SELECT * FROM employees WHERE last_name = :last_name

-- 安全: PL/SQL での使用
UPDATE employees SET salary = :salary WHERE employee_id = :id
```

### 間違い: 動的 SQL の文字列結合

ORDS ハンドラーで以下のようなことは決して行ってはならない。

```sql
-- 危険: :last_name が動的 SQL に注入される
EXECUTE IMMEDIATE 'SELECT * FROM employees WHERE last_name = ''' || :last_name || '''';

-- 危険: バインド変数を使わずに WHERE 句を動的に構築する
l_sql := 'SELECT * FROM employees WHERE ' || :filter_column || ' = ' || :filter_value;
EXECUTE IMMEDIATE l_sql;
```

### DBMS_SQL を使用した安全な動的 SQL

動的 SQL が真に必要な場合:

```sql
DECLARE
  l_cursor   INTEGER;
  l_col_name VARCHAR2(30);
  l_val      VARCHAR2(100);
  l_result   SYS_REFCURSOR;
BEGIN
  -- 許可リストに対するカラム名の検証 (重要!)
  IF :column_name NOT IN ('department_id', 'job_id', 'manager_id') THEN
    RAISE_APPLICATION_ERROR(-20001, 'Invalid filter column');
  END IF;

  l_col_name := :column_name;  -- 許可リスト確認後は安全
  l_val      := :value;

  -- 動的 SQL 内でバインド変数を使用
  OPEN l_result FOR
    'SELECT * FROM employees WHERE ' || l_col_name || ' = :val'
    USING l_val;

  DBMS_SQL.RETURN_RESULT(l_result);
END;
```

### カラム名/テーブル名の許可リスト化

カラム名やテーブル名はバインド変数としてパラメータ化できない。厳格な許可リストに対して検証すること。

```sql
FUNCTION validate_column_name(p_col IN VARCHAR2) RETURN VARCHAR2 AS
  l_allowed DBMS_UTILITY.LNAME_ARRAY;
BEGIN
  -- 許可されたカラムを明示的に定義
  IF p_col NOT IN (
    'employee_id', 'department_id', 'job_id',
    'hire_date', 'salary', 'manager_id'
  ) THEN
    RAISE_APPLICATION_ERROR(-20001, 'Column not allowed for filtering: ' || p_col);
  END IF;
  RETURN DBMS_ASSERT.SIMPLE_SQL_NAME(p_col);  -- 追加の検証
END;
```

---

## データベース資格証明のシークレット管理

### ORDS ウォレットベースのシークレット・ストレージ (デフォルトの仕組み)

ORDS のすべてのパスワードは、ORDS 構成の `credentials/` ディレクトリにある **Oracle ウォレット**（資格証明ストア）に保存される。パスワードが構成ファイルに現れることはない。`ords config secret set` を使用して資格証明を設定またはローテーションする。

```shell
# DB パスワードの設定 — Oracle ウォレットにのみ保存
ords --config /opt/oracle/ords/config config secret set db.password \
  --password-stdin <<< "MySecurePassword123!"

# パスワードのローテーション — 新しい値で再実行
ords --config /opt/oracle/ords/config config secret set db.password \
  --password-stdin <<< "NewSecurePassword456!"

# ウォレットの存在確認 (パスワードは構成ファイルからは読み取れない)
ls /opt/oracle/ords/config/credentials/
```

`credentials/` ディレクトリは OS レベルの権限 (`chmod 700`) で保護し、構成ディレクトリの他の部分とともにバックアップ・手順に含めること。

### OCI Vault との統合

クラウド・デプロイメントでは、起動時に OCI Vault からシークレットを取得し、中間ファイルを経由せずに直接ウォレットにパイプする。

```shell
OCI_SECRET=$(oci secrets secret-bundle get \
  --secret-id ocid1.vaultsecret.oc1... \
  --query 'data."secret-bundle-content".content' \
  --raw-output | base64 --decode)

ords --config /opt/oracle/ords/config config secret set db.password \
  --password-stdin <<< "$OCI_SECRET"
```

---

## IP 許可リスト

ORDS はネイティブでは IP 許可リストをサポートしていない。ネットワーク・レイヤーで実装する。

### Linux `iptables` の使用

```shell
# 特定の IP 範囲のみ ORDS ポートへの到達を許可
iptables -A INPUT -p tcp --dport 8443 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 8443 -j DROP
```

### Nginx をリバース・プロキシとして使用

```nginx
location /ords/ {
    # 社内ネットワークの許可
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    # 特定のパートナー IP の許可
    allow 203.0.113.45;
    # その他すべてを拒否
    deny all;

    proxy_pass http://localhost:8080/ords/;
}
```

### ORDS リクエスト検証関数

ORDS は、IP チェックを含むカスタム・アクセス制御ロジックのための PL/SQL リクエスト検証関数をサポートしている。

```sql
CREATE OR REPLACE FUNCTION hr.ords_request_validator(
  p_method        IN VARCHAR2,
  p_path          IN VARCHAR2,
  p_content_type  IN VARCHAR2,
  p_payload       IN BLOB,
  p_env           IN OWA.vc_arr,
  p_val           IN OWA.vc_arr
) RETURN BOOLEAN AS
  l_remote_addr VARCHAR2(50);
BEGIN
  -- OWA 環境からクライアント IP を取得
  FOR i IN 1..p_env.COUNT LOOP
    IF UPPER(p_env(i)) = 'REMOTE_ADDR' THEN
      l_remote_addr := p_val(i);
    END IF;
  END LOOP;

  -- 既知の不正 IP をブロック
  IF l_remote_addr IN (
    SELECT ip_address FROM security.blocked_ips WHERE active = 'Y'
  ) THEN
    RETURN FALSE;
  END IF;

  RETURN TRUE;
END;
/
```

ORDS での構成:

```shell
ords --config /opt/oracle/ords/config config set \
  security.requestValidationFunction hr.ords_request_validator
```

---

## レート制限

ORDS にはレート制限機能が組み込まれていない。リバース・プロキシまたは API ゲートウェイ・レイヤーで実装する。

### Nginx のレート制限

```nginx
# レート制限ゾーンの定義: IP あたり 毎分 100 リクエスト
limit_req_zone $binary_remote_addr zone=ords_limit:10m rate=100r/m;

server {
    location /ords/ {
        limit_req zone=ords_limit burst=20 nodelay;
        limit_req_status 429;

        proxy_pass http://localhost:8080/ords/;
    }
}
```

### API ゲートウェイのレート制限 (OCI)

OCI デプロイメントでは、ORDS の前に OCI API Gateway を使用する。
- クライアントごとのリクエスト制限を含む使用プランを構成する
- API キーまたは OAuth トークンに基づいてクライアント別のレート制限を適用する
- 制限されたリクエストについてログを記録しアラートを出す

---

## Oracle Database Vault との統合

Oracle Database Vault (DBV) は、特権アカウントからでも機密スキーマへのアクセスを制限する。ORDS と組み合わせることで、万が一 ORDS が侵害された場合でも、ORDS 接続アカウントが必要以上のデータにアクセスすることを防ぐ。

```sql
-- 例: HR スキーマ用の DBV レルムを作成
EXEC DVSYS.DBMS_MACADM.CREATE_REALM(
  realm_name   => 'HR Protected Realm',
  description  => 'Restricts access to HR sensitive tables',
  enabled      => DVSYS.DBMS_MACUTL.G_YES,
  audit_options => DVSYS.DBMS_MACUTL.G_REALM_AUDIT_FAIL
);

-- HR スキーマ・オブジェクトをレルムに追加
EXEC DVSYS.DBMS_MACADM.ADD_OBJECT_TO_REALM(
  realm_name   => 'HR Protected Realm',
  object_owner => 'HR',
  object_name  => 'SALARY_DETAILS',  -- 機密テーブル
  object_type  => 'TABLE'
);

-- 特定のユーザーのみを承認
EXEC DVSYS.DBMS_MACADM.ADD_AUTH_TO_REALM(
  realm_name   => 'HR Protected Realm',
  grantee      => 'HR_COMPENSATION_APP',  -- アプリケーション・ユーザー (ORDS_PUBLIC_USER ではない)
  rule_set_name => NULL,
  auth_options => DVSYS.DBMS_MACUTL.G_REALM_AUTH_OWNER
);
```

これにより、攻撃者が ORDS ハンドラーの SQL を操作したとしても、`ORDS_PUBLIC_USER`（および一般的な ORDS 接続）からは `SALARY_DETAILS` にアクセスできなくなる。

---

## セキュリティ HTTP ヘッダー

セキュリティ・ヘッダーを返すように ORDS を構成する。

```shell
# Nginx 経由 (推奨)
add_header X-Content-Type-Options    "nosniff" always;
add_header X-Frame-Options           "DENY" always;
add_header X-XSS-Protection          "1; mode=block" always;
add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy   "default-src 'self'" always;
add_header Permissions-Policy        "geolocation=(), microphone=()" always;
```

---

## ベスト・プラクティス

- **最小権限の原則を徹底する**: `ORDS_PUBLIC_USER` に必要なのは `CREATE SESSION` のみである。REST を実行するスキーマには、使用するオブジェクトのみの権限を与える。接続プールに DBA アカウントを使用してはならない。
- **すべての AutoREST スキーマで `p_auto_rest_auth => TRUE` を有効にする**: デフォルトですべての AutoREST エンドポイントに認証を強制する。デフォルトを公開にするのではなく、公開オブジェクトを明示的にホワイトリスト化する。
- **資格証明を定期的にローテーションする**: クライアント・シークレット、DB パスワード、および SSL 証明書には、すべて定義されたローテーション・スケジュールを持たせる。OCI Vault や HashiCorp Vault を使用してローテーションを自動化する。
- **WAF (Web Application Firewall) の背後で ORDS を実行する**: ORDS の前に OCI WAF や AWS WAF を配置することで、ORDS 自体では軽減できない一般的な Web 攻撃 (OWASP Top 10) に対する保護を追加する。
- **すべての ORDS リクエストを監査する**: リクエスト・ロギングを有効にし、ログを SIEM に集約する。401 エラーの急増（資格証明の詰め込み攻撃）、500 エラーの急増（悪用試行）、異常なペイロード・サイズにアラートを出す。
- **ORDS を最新の状態に保つ**: Oracle は定期的に ORDS のセキュリティ・パッチをリリースしている。Oracle Security Alerts を購読し、速やかに更新すること。

## よくある間違い

- **認証済みエンドポイントでのワイルドカード CORS**: OAuth2 Bearer トークンを使用している環境での `*` CORS ポリシーは、トークンのセキュリティを無効にする。あらゆるサイトがクロスオリジン・リクエストを行えるようになってしまうため、明示的なオリジン許可リストを使用すること。
- **構成ファイルにパスワードを書き込もうとする**: ORDS はすべての資格証明を Oracle ウォレット (`credentials/` ディレクトリ) に保存する。構成ファイルにパスワードを記載すべきではない。設定やローテーションには `ords config secret set db.password` を使用すること。構成ファイルに直接パスワードを埋め込もうとしても機能せず、ORDS の起動を妨げる可能性がある。
- **本番環境で不要な Database Actions を無効にしていない**: Database Actions (`feature.sdw=true`) は強力な SQL 実行インターフェースを提供する。API 専用の ORDS インスタンスでは無効にすること: `feature.sdw=false`。
- **すべてのサービスで同じ OAuth クライアントを使用している**: 共有クライアントが侵害されると、すべてのサービスがアクセスを失う。サービス/アプリケーションごとに 1 つの OAuth クライアントを使用すること。
- **スキーマ変更後にすべてのエンドポイントの認証をテストしていない**: 新しいモジュールを追加しても、自動的に保護されるわけではない。各新規エンドポイントが期待される認証を要求することを確認すること。
- **ORDS ログの ORA-00942 や ORA-04043 エラーを無視している**: これらはオブジェクトや権限の欠落を示しており、構成ミスや権限昇格の試みの可能性を示唆している。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替手段を維持すること。
- 19c と 26ai の両方をサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [ORDS Developer's Guide — Securing Oracle REST Data Services](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/about-oracle-rest-data-services.html)
- [ORDS Configuration Settings Reference — Security Settings](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/configuration-settings.html)
- [Oracle Database Vault Administrator's Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/dvadm/index.html)
