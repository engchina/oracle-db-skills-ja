# ORDS の認証と認可

## 概要

ORDS は、REST エンドポイントを保護するための OAuth2 ベースの完全なセキュリティ・モデルを提供している。アクセス制御は、**権限 (Privileges)**（どのエンドポイントを保護するか）と **OAuth2 クライアント**（どのアプリケーション/ユーザーがアクセスできるか）を通じて定義される。ORDS は、OAuth2 クライアント資格証明フロー（マシン間）、認可コード・フロー（ユーザー向け Web アプリケーション）、およびインプリシット・フローをサポートしている。さらに、ORDS は JWT プロファイル構成を介して外部アイデンティティ・プロバイダーをサポートしており、Oracle Identity Cloud Service (IDCS)、Azure AD、Okta、およびその他の OIDC 互換プロバイダーとの統合が可能である。

---

## 主要なセキュリティ概念

### 権限 (Privileges)

**権限**は、1 つ以上の URL パターン（REST モジュールまたは特定のテンプレート）に関連付けられた、名前付きのアクセス制御ゲートである。保護された URL へのリクエストには、必要な権限スコープを持つ有効な OAuth2 Bearer トークンを提示する必要がある。

### ロール (Roles)

**ロール**は、権限に割り当てることができるオプションのラベルである。クライアントまたはユーザーが権限を受け取るには、必要なロールを保持している必要がある。ロールは、OAuth2 スコープ・チェックに加えて、第 2 の認可レイヤーを提供する。

### OAuth クライアント (OAuth Clients)

**OAuth クライアント**は、保護された REST API を呼び出したいアプリケーション（またはユーザー・アカウント）を表す。クライアントにはクライアント ID とクライアント・シークレットが発行され、これらをトークン・エンドポイントでアクセス・トークンと交換する。

---

## 権限の定義

```sql
-- 従業員データ・エンドポイントを保護する権限を定義
BEGIN
  ORDS.DEFINE_PRIVILEGE(
    p_privilege_name => 'hr.employees.read',
    p_roles          => NULL,                  -- ロール制限なし（認証されたすべてのクライアント）
    p_label          => 'HR Employee Read',
    p_description    => 'HR 従業員データへの読み取りアクセス',
    p_comments       => NULL
  );
  COMMIT;
END;
/
```

### 必要なロールを伴う権限

```sql
DECLARE
  l_roles owa.vc_arr;
BEGIN
  -- 最初にロールを作成
  ORDS.CREATE_ROLE(p_role_name => 'HR_MANAGER');
  ORDS.CREATE_ROLE(p_role_name => 'HR_ADMIN');

  -- HR_MANAGER または HR_ADMIN ロールを必要とする権限を定義
  -- p_roles は owa.vc_arr 型（ORDS_TYPES.T_ORDS_STR_LIST ではない）
  l_roles(1) := 'HR_MANAGER';
  l_roles(2) := 'HR_ADMIN';

  ORDS.DEFINE_PRIVILEGE(
    p_privilege_name => 'hr.employees.write',
    p_roles          => l_roles,
    p_label          => 'HR Employee Write',
    p_description    => 'HR 従業員レコードの作成、更新、削除'
  );
  COMMIT;
END;
/
```

### モジュール・パターンへの権限の適用

```sql
BEGIN
  -- hr.employees モジュール内のすべてのエンドポイントを保護
  ORDS.PRIVILEGE_MAP_MODULE(
    p_privilege_name => 'hr.employees.read',
    p_module_name    => 'hr.employees'
  );
  COMMIT;
END;
/
```

### 特定の URL パターンへの権限の適用

```sql
BEGIN
  -- employees リソースへの書き込み操作のみを保護
  ORDS.CREATE_ROLE('hr.employees.write');

  -- 権限を URL パターンに関連付け
  ORDS.DEFINE_PRIVILEGE(
    p_privilege_name  => 'hr.employees.write',
    p_roles           => ORDS_TYPES.T_ORDS_STR_LIST('HR_ADMIN'),
    p_label           => 'Employee Write',
    p_description     => '従業員レコードの修正',
    p_module_name     => 'hr.employees',   -- モジュールにスコープ
    p_pattern         => 'employees/'
  );
  COMMIT;
END;
/
```

---

## OAuth2 クライアント資格証明フロー (マシン間)

このフローは、対話型ユーザーが存在しないサーバー間の API 呼び出しに使用される。バックエンド・サービスは、独自のクライアント ID とシークレットを使用してアクセス・トークンを取得する。

### ステップ 1: OAuth クライアントの作成

```sql
BEGIN
  OAUTH.CREATE_CLIENT(
    p_name            => 'reporting-service',
    p_grant_type      => 'client_credentials',
    p_owner           => 'Data Platform Team',
    p_description     => 'HR データ用自動レポート・サービス',
    p_support_email   => 'dataplatform@example.com',
    p_privilege_names => 'hr.employees.read'   -- カンマ区切りの権限名
  );
  COMMIT;
END;
/
```

### ステップ 2: クライアント ID とシークレットの取得

```sql
-- 生成された client_id と client_secret を確認
SELECT client_id, client_secret
FROM user_ords_clients
WHERE name = 'reporting-service';
```

`client_secret` は安全に保管すること。これは作成直後（またはリセットされるまで）のみ表示可能である。

### ステップ 3: クライアントへのロールの付与

```sql
BEGIN
  OAUTH.GRANT_CLIENT_ROLE(
    p_client_name => 'reporting-service',
    p_role_name   => 'HR_ADMIN'   -- 権限がこのロールを必要とする場合
  );
  COMMIT;
END;
/
```

### ステップ 4: アクセス・トークンの取得

```http
POST /ords/hr/oauth/token HTTP/1.1
Host: myserver.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=client_credentials
```

または、curl を使用した場合:

```shell
curl -s -X POST \
  https://myserver.example.com/ords/hr/oauth/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u "<client_id>:<client_secret>" \
  -d "grant_type=client_credentials"
```

レスポンス:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### ステップ 5: 保護されたエンドポイントの呼び出し

```http
GET /ords/hr/v1/employees/ HTTP/1.1
Host: myserver.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## OAuth2 認可コード・フロー (ユーザー向けアプリケーション)

人間であるユーザーが認証を行い、アプリケーションが自分の代わりにデータにアクセスすることを明示的に許可する必要がある場合に使用される。

### ステップ 1: 認可コード・クライアントの作成

```sql
BEGIN
  OAUTH.CREATE_CLIENT(
    p_name              => 'hr-portal',
    p_grant_type        => 'authorization_code',
    p_redirect_uri      => 'https://hrportal.example.com/callback',
    p_support_email     => 'portal-admin@example.com',
    p_privilege_names   => 'hr.employees.read,hr.employees.write',
    p_owner             => 'HR Portal Team',
    p_description       => 'HR セルフサービス・ポータル'
  );
  COMMIT;
END;
/
```

### ステップ 2: 認可コード・フローの実行

**ステップ 2a: ユーザーを ORDS 認可エンドポイントにリダイレクト**

```
https://myserver.example.com/ords/hr/oauth/auth
  ?response_type=code
  &client_id=<client_id>
  &redirect_uri=https://hrportal.example.com/callback
  &state=random_csrf_state_value
  &scope=hr.employees.read
```

**ステップ 2b: ユーザー認証**（ORDS ファースト・パーティ認証または外部 IdP 経由）

**ステップ 2c: ORDS が認可コードを伴って `redirect_uri` にリダイレクト**

```
https://hrportal.example.com/callback?code=AUTH_CODE_HERE&state=random_csrf_state_value
```

**ステップ 2d: 認可コードをアクセス・トークンに交換**

```http
POST /ords/hr/oauth/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=authorization_code
&code=AUTH_CODE_HERE
&redirect_uri=https://hrportal.example.com/callback
```

レスポンスには `access_token` と `refresh_token` が含まれる。

---

## OAuth クライアントとトークンの管理

```sql
-- すべての OAuth クライアントを一覧表示
SELECT client_id, name, grant_type, redirect_uri, status
FROM user_ords_clients;

-- 発行済みトークンの表示 (管理者向けビュー)
SELECT c.name, t.token_type, t.expires_in
FROM user_ords_tokens t
JOIN user_ords_clients c ON c.client_id = t.client_id;

-- クライアントの無効化 (そのクライアントのすべてのトークンを無効化)
BEGIN
  OAUTH.UPDATE_CLIENT(
    p_name   => 'reporting-service',
    p_status => 'REVOKED'
  );
  COMMIT;
END;
/

-- クライアントの削除
BEGIN
  OAUTH.DELETE_CLIENT(p_name => 'reporting-service');
  COMMIT;
END;
/

-- クライアント・シークレットのリセット
BEGIN
  OAUTH.RESET_CLIENT_SECRET(p_name => 'reporting-service');
  COMMIT;
END;
/

-- リセット後の新しいシークレットの取得
SELECT client_secret FROM user_ords_clients WHERE name = 'reporting-service';
```

---

## トークン・エンドポイントとディスカバリ

ORDS は、REST 有効化されたスキーマごとに標準の OAuth2 エンドポイントを公開している。

| エンドポイント | URL |
|---|---|
| トークン・エンドポイント | `/ords/{schema}/oauth/token` |
| 認可エンドポイント | `/ords/{schema}/oauth/auth` |
| ディスカバリ (OpenID Connect) | `/ords/{schema}/.well-known/openid-configuration` |
| JWKS (公開鍵) | `/ords/{schema}/oauth/keys` |

```shell
# OAuth2 構成の検出
curl https://myserver.example.com/ords/hr/.well-known/openid-configuration
```

---

## 外部アイデンティティ・プロバイダーのための JWT プロファイル構成

ORDS は、ORDS 自体が発行したものではない外部アイデンティティ・プロバイダー（Oracle IDCS、Azure AD、Okta、Keycloak など）によって発行された JWT を検証できる。

### ORDS での JWT 検証の構成

```shell
# 信頼された発行者と JWKS URI を設定
ords --config /opt/oracle/ords/config config set \
  security.jwt.allowedAge 3600

# JWT プロファイルの追加
ords --config /opt/oracle/ords/config config secret --global \
  jwt.verifier.1.issuer "https://login.microsoftonline.com/{tenant-id}/v2.0"

ords --config /opt/oracle/ords/config config secret --global \
  jwt.verifier.1.jwksUri "https://login.microsoftonline.com/{tenant-id}/discovery/v2.0/keys"

ords --config /opt/oracle/ords/config config secret --global \
  jwt.verifier.1.audience "api://my-ords-api"
```

構成後、クライアントは Azure AD からの JWT を Bearer トークンとして送信し、ORDS がそれを検証する。

```http
GET /ords/hr/v1/employees/ HTTP/1.1
Authorization: Bearer <Azure AD JWT>
```

### JWT クレームの ORDS ロールへのマッピング

JWT クレーム（`groups` や `roles` クレームなど）を ORDS ロールにマッピングするように ORDS を構成する。

```shell
ords --config /opt/oracle/ords/config config set --global jwt.claimRole roles
```

ORDS が `"roles": ["HR_ADMIN"]` を含む JWT を受け取ると、それを ORDS ロールの `HR_ADMIN` にマップし、そのロールを必要とする権限へのアクセスを許可する。

---

## ロールベースのアクセス制御 (RBAC) の概要

```
JWT/トークン ──► ORDS がトークンを検証 ──► ロール/スコープを抽出
                                             │
                                    ┌────────▼────────┐
                                    │   ORDS ロール    │
                                    │  HR_MANAGER      │
                                    │  HR_ADMIN        │
                                    └────────┬─────────┘
                                             │
                               ┌─────────────▼──────────────┐
                               │      ORDS 権限             │
                               │  hr.employees.read          │
                               │  hr.employees.write         │
                               └─────────────┬──────────────┘
                                             │
                               ┌─────────────▼──────────────┐
                               │   保護された URL パターン    │
                               │  /v1/employees/             │
                               │  /v1/employees/:id          │
                               └─────────────────────────────┘
```

---

## ファースト・パーティ認証 (ORDS ユーザー)

ORDS は、Database Actions や同様のツールのための直接的なユーザー認証をサポートしている。ORDS ユーザーはデータベース・アカウントにマップされる。

```sql
-- REST 専用の ORDS ユーザー（Database Actions ユーザー）を作成
-- Database Actions UI を使用するか、プログラムで DB ユーザー自体を介して行う

-- DB ユーザーに REST アクセスを付与（スキーマ所有者または DBA が実行する必要がある）
BEGIN
  ORDS.ENABLE_SCHEMA(
    p_enabled             => TRUE,
    p_schema              => 'ALICE',
    p_url_mapping_type    => 'BASE_PATH',
    p_url_mapping_pattern => 'alice',
    p_auto_rest_auth       => TRUE
  );
  COMMIT;
END;
/
```

---

## ベスト・プラクティス

- **すべてのサービス間呼び出しにクライアント資格証明を使用する**: クライアント・アプリケーション内に DB の資格証明を直接記述してはならない。OAuth2 クライアント資格証明を使用することで、DB のパスワードを変更せずにアクセスを取り消すことができる。
- **機密性の高い操作には短いトークン有効期限を設定する**: ORDS のデフォルトのトークン有効期限は 1 時間である。高いセキュリティが求められるコンテキストでは、より短い有効期限を構成し、リフレッシュ・トークンのロジックを実装すること。
- **権限を細かく設定する**: 読み取り、書き込み、管理者操作などの権限を個別に作成する。クライアントには、必要最小限の権限のみを付与すること。
- **ユーザー向けアプリケーションには外部 IdP を使用する**: ORDS 内でユーザーを管理するのではなく、JWT プロファイルを介して組織のアイデンティティ・プロバイダーと統合すること。これにより、SSO が可能になり、ユーザーのライフサイクル管理を一元化できる。
- **クライアント・シークレットを定期的にローテーションする**: クライアント・シークレットはパスワードと同じように扱う。定期的なスケジュールに基づき、または漏洩の可能性がある場合にローテーションを行う。`OAUTH.RESET_CLIENT_SECRET` を使用すること。
- **権限割り当てを監査する**: `user_ords_clients` および `user_ords_client_roles` を定期的に確認し、古くなったクライアントや過剰な権限を持つクライアントが存在しないか確認すること。

## よくある間違い

- **間違ったスキーマのトークン・エンドポイントを呼び出す**: ORDS のトークン・エンドポイントはスキーマごとにスコープされている (`/ords/{schema}/oauth/token`)。間違ったスキーマのパスを使用すると 404 エラーが返される。
- **クライアントへのロールの付与を忘れる**: 権限が特定のロールを必要とする場合、クライアントを作成して権限を付与するだけでは不十分である。`user_ords_client_roles` を確認すること。
- **Authorization ヘッダーではなくリクエスト・ボディでクライアント資格証明を送信する**: OAuth2 仕様では両方が許可されているが、ORDS の `client_credentials` グラントでは `Authorization: Basic <base64>` を必要とする。ボディベースの資格証明は ORDS では失敗する。
- **認可コードのリダイレクト URI を URL エンコードしない**: 認可コードの交換における `redirect_uri` は、末尾のスラッシュやエンコードを含め、登録された URI と正確に一致している必要がある。
- **クライアント側のコード内に `client_secret` を公開する**: ブラウザ・ベースのアプリ (SPA) では、PKCE を使用するか（使用しているバージョンで ORDS がサポートしている場合）、Backend-for-Frontend (BFF) パターンを使用すること。JavaScript 内にクライアント・シークレットを絶対に記述しないこと。
- **JWT 内のロールが ORDS の権限名と一致していると思い込む**: JWT のロールは ORDS ロールにマップされ、ORDS ロールが ORDS 権限に関連付けられる。これらは異なるレイヤーである。マッピングを慎重に構成すること。

---

## ソース

- [ORDS Developer's Guide — Securing Oracle REST Data Services](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/about-oracle-rest-data-services.html)
- [Oracle REST Data Services PL/SQL API Reference — OAUTH and ORDS Packages](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orrst/index.html)
- [ORDS OAuth2 Client Credentials Flow](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/rest-enabled-sql-service.html)
