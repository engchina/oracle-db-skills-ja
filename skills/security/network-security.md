# Oracle ネットワーク・セキュリティ (Network Security)

## 概要

ネットワークは、Oracle データベースと、それに接続するすべてのアプリケーション、ユーザー、およびサービスとの間の攻撃面（アタック・サーフェス）である。Oracle のネットワーク・レイヤーを保護することは、以下の事項を保証することを意味する。

1.  接続が**認証**されていること — 正当なクライアントのみが接続できる。
2.  転送中のデータが**暗号化**されていること — トラフィックが傍受されたり読み取られたりしない。
3.  アクセスが**制限**されていること — 許可されたホストのみがリスナーに到達できる。
4.  リスナー自体が**強化（ハーデニング）**されていること — データベースの認証をバイパスするためにリスナーが悪用されない。

Oracle のネットワーク・セキュリティ・スタックには、Oracle Listener、Oracle Net Services（以前の SQL*Net）、`sqlnet.ora` 構成、Oracle ウォレット、およびデータベース内から行われるアウトバウンド・ネットワーク・コールのためのアクセス制御リスト (ACL) が含まれる。

---

## Oracle Listener

リスナーはデータベースへのゲートウェイである。入ってくる接続要求を受け取り、それらを Oracle サーバー・プロセスに引き渡す。設定ミスのリスナーは、Oracle における最も一般的なセキュリティ脆弱性の 1 つである。

### listener.ora — 安全な構成

```ini
# /opt/oracle/network/admin/listener.ora

# 特定のインターフェースのみにバインドする — 本番環境で 0.0.0.0 をリスニングしないこと
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCPS)
                 (HOST = db-server-hostname)
                 (PORT = 2484))       # TCPS (TLS) ポート
      (ADDRESS = (PROTOCOL = TCP)
                 (HOST = 10.0.1.50)   # 内部 IP のみ
                 (PORT = 1521))
    )
  )

# 動的なサービス登録を無効化する（外部からの操作を防止）
# すべてのサービスは以下のように静的に登録する必要がある
SECURE_REGISTER_LISTENER = (TCPS, IPC)

# リスナー経由の外部プロシージャ実行を無効化する（セキュリティ強化）
# 使用しない場合は EXTPROC を削除またはコメントアウトする
# (PROGRAM = extproc) の行は本番環境には存在させないこと

# リスナーの管理操作をローカル接続のみに制限する
ADMIN_RESTRICTIONS_LISTENER = ON
# これにより、リモートの LSNRCTL コマンドによるログ・ファイルの設定や、
# サーバー上のファイルを上書きするために悪用される可能性のあるトレースの設定を防止できる

# 静的なサービス登録（動的登録よりも安全）
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = ORCL)
      (ORACLE_HOME = /opt/oracle/product/19c/dbhome_1)
      (SID_NAME = ORCL)
    )
  )
```

### リスナーのパスワード (レガシー — 代わりに ADMIN_RESTRICTIONS を使用)

```bash
# Oracle 10g 以降では、リスナー・パスワードよりも ADMIN_RESTRICTIONS_LISTENER = ON が推奨される。
# もしリスナー・パスワードを設定する必要がある場合（古い環境など）:
lsnrctl
LSNRCTL> change_password
Old password: <空白のままエンター>
New password: <新しいパスワードを入力>
Confirm password: <新しいパスワードを再入力>
LSNRCTL> save_config
LSNRCTL> exit
```

### リスナーのステータスとセキュリティの確認

```bash
# サービスの一覧表示 (過剰な情報を露出させないこと)
lsnrctl status

# リスナー・バージョンの確認 (レスポンスでのバージョン露出を最小限にする)
# listener.ora で以下のように設定する:
# SECURE_CONTROL_LISTENER = (TCPS, IPC)

# リスナーに接続しているユーザーを確認
lsnrctl services

# 不正な動的サービス登録がないか確認
lsnrctl show dynamic_registration
# 出力は OFF であるか、既知のサービスのみが表示される必要がある
```

---

## Oracle 向けの SSL/TLS 構成

Oracle 接続に SSL/TLS を構成するには、サーバー証明書を保持するための Oracle ウォレットが必要であり、サーバー側とクライアント側の両方で `sqlnet.ora` を更新する必要がある。

### サーバー側の TLS 構成

#### 証明書を使用した Oracle ウォレットのセットアップ

```bash
# ステップ 1: ウォレットの作成
orapki wallet create -wallet /opt/oracle/wallet/tls -auto_login
# またはパスワード保護する場合:
orapki wallet create -wallet /opt/oracle/wallet/tls -pwd WalletP@ss!

# ステップ 2: 証明書署名要求 (CSR) の生成
orapki wallet add -wallet /opt/oracle/wallet/tls \
  -dn "CN=db-server.corp.example.com,OU=Database,O=Example Corp,C=US" \
  -keysize 2048 \
  -sign_alg sha256 \
  -pwd WalletP@ss!

# ステップ 3: 認証局 (CA) による署名のために CSR をエクスポート
orapki wallet export -wallet /opt/oracle/wallet/tls \
  -dn "CN=db-server.corp.example.com,OU=Database,O=Example Corp,C=US" \
  -request /tmp/db-server.csr \
  -pwd WalletP@ss!

# ステップ 4: CA から署名済み証明書をインポート
orapki wallet add -wallet /opt/oracle/wallet/tls \
  -trusted_cert -cert /tmp/ca-cert.pem -pwd WalletP@ss!

orapki wallet add -wallet /opt/oracle/wallet/tls \
  -user_cert -cert /tmp/db-server-signed.pem -pwd WalletP@ss!

# ステップ 5: ウォレットの内容を確認
orapki wallet display -wallet /opt/oracle/wallet/tls -pwd WalletP@ss!
```

#### sqlnet.ora — サーバー側

```ini
# /opt/oracle/network/admin/sqlnet.ora (サーバー側)

# SSL/TLS を有効化
SSL_VERSION = 1.2          # 最小 TLS 1.2。すべてのクライアントがサポートしている場合は 1.3 を設定
                            # SSLv3, TLSv1.0, TLSv1.1 は絶対に使用しないこと
SSL_CLIENT_AUTHENTICATION = FALSE  # 相互 TLS (クライアント証明書) を使用する場合は TRUE に設定

# 暗号化スイートを強力なもののみに制限
SSL_CIPHER_SUITES = (TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
                     TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
                     TLS_RSA_WITH_AES_256_CBC_SHA256)

# TLS 証明書用のウォレットの場所
WALLET_LOCATION =
  (SOURCE =
    (METHOD = FILE)
    (METHOD_DATA =
      (DIRECTORY = /opt/oracle/wallet/tls)))

# TDE ウォレット (TLS ウォレットとは別にする — 異なる場所でも可)
ENCRYPTION_WALLET_LOCATION =
  (SOURCE =
    (METHOD = FILE)
    (METHOD_DATA =
      (DIRECTORY = /opt/oracle/wallet/tde)))
```

#### listener.ora — TLS ポート

```ini
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCPS)(HOST = db-server.corp.example.com)(PORT = 2484))
    )
  )

SSL_CLIENT_AUTHENTICATION = FALSE

WALLET_LOCATION =
  (SOURCE =
    (METHOD = FILE)
    (METHOD_DATA =
      (DIRECTORY = /opt/oracle/wallet/tls)))
```

---

## Oracle Net 暗号化 (ネイティブ暗号化)

Oracle Advanced Security は、証明書やウォレットを必要とせずに、`sqlnet.ora` を介したネイティブな**ネットワーク・データ暗号化**も提供している。これは TLS よりも導入が簡単だが、保証レベルは低くなる（サーバーの身元確認が行われないため）。

```ini
# /opt/oracle/network/admin/sqlnet.ora

# ネイティブ・ネットワーク暗号化 — Oracle 独自のメカニズムを使用して転送中のデータを暗号化
# SSL 証明書は使用せず、キー交換にチャレンジ・レスポンス認証を使用する

# REQUIRED: 両側で暗号化が必須
# REQUESTED: 暗号化を優先する。クライアントがサポートしていない場合は非暗号化も受け入れる
# ACCEPTED: 暗号化を許可するが、必須ではない
# REJECTED: 暗号化された接続を拒否する

SQLNET.ENCRYPTION_SERVER = REQUIRED      # すべてのサーバー接続に暗号化を強制
SQLNET.ENCRYPTION_CLIENT = REQUIRED      # すべてのクライアント接続に暗号化を強制

# 優先するアルゴリズムを指定 (強力なものから順に)
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, AES192, AES128)
SQLNET.ENCRYPTION_TYPES_CLIENT = (AES256, AES192, AES128)

# 暗号化チェックサム（データの整合性 — 改ざん検知）
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_CLIENT = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA256)
SQLNET.CRYPTO_CHECKSUM_TYPES_CLIENT = (SHA256)

# 非推奨 — 3DES, DES, RC4 は含めないこと
# Bad 例: SQLNET.ENCRYPTION_TYPES_SERVER = (3DES168, DES, RC4_256)
```

---

## Oracle 接続記述子 (tnsnames.ora) — クライアント側 TLS

```ini
# /opt/oracle/network/admin/tnsnames.ora (クライアント側)

ORCL_TLS =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCPS)(HOST = db-server.corp.example.com)(PORT = 2484))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = ORCL)
    )
    (SECURITY =
      (MY_WALLET_DIRECTORY = /opt/oracle/wallet/client)
      (SSL_SERVER_CERT_DN = "CN=db-server.corp.example.com,OU=Database,O=Example Corp,C=US")
      # SSL_SERVER_CERT_DN はサーバー証明書の DN を検証する — 中間者攻撃 (MITM) を防止
    )
  )
```

---

## アウトバウンド・ネットワーク・コールのためのアクセス制御リスト (ACL)

`UTL_HTTP`, `UTL_TCP`, `UTL_SMTP`, `UTL_FILE` などの Oracle PL/SQL パッケージを使用すると、データベースからアウトバウンド（外部）接続を行うことができる。ACL がないと、これらのパッケージが悪用されてデータが漏洩したり、データベース・サーバーから内部ネットワーク攻撃が仕掛けられたり、内部サービスにアクセスされたりする可能性がある。

Oracle 11g 以降、これらのパッケージからのアウトバウンド接続には、明示的な ACL の付与が必要である。

### ネットワーク ACL の作成

```sql
-- 特定のホストとポートへの UTL_HTTP アクセスを許可
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host    => 'api.payment-gateway.com',
    lower_port => 443,
    upper_port => 443,
    ace     => xs$ace_type(
      privilege_list => xs$name_list('http'),   -- HTTP/HTTPS の場合は 'http'
      principal_name => 'WEBAPP_SVC',           -- Oracle ユーザーまたはロール
      principal_type => xs_acl.ptype_db         -- データベース・ユーザー/ロール
    )
  );
END;
/

-- UTL_TCP/UTL_SMTP 用の TCP アクセスを許可
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host    => 'smtp.corp.example.com',
    lower_port => 587,
    upper_port => 587,
    ace     => xs$ace_type(
      privilege_list => xs$name_list('connect', 'resolve'),
      principal_name => 'NOTIFICATION_APP',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/

-- DNS 解決のみを許可 (フル・アクセスなしでホスト名の解決のみを行う場合)
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host    => '*',           -- DNS 解決のみであればすべてのホストを許可
    lower_port => NULL,
    upper_port => NULL,
    ace     => xs$ace_type(
      privilege_list => xs$name_list('resolve'),
      principal_name => 'APP_USER',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/

-- ワイルドカード・ホスト (注意して使用する — サブドメインに制限)
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host    => '*.corp.example.com',  -- corp.example.com のサブドメインのみ
    lower_port => 443,
    upper_port => 443,
    ace     => xs$ace_type(
      privilege_list => xs$name_list('http', 'connect', 'resolve'),
      principal_name => 'INTERNAL_APP',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/
```

### ACL の管理と照会

```sql
-- すべてのホスト ACE (ホスト・ベースの ACL エントリ) を確認
SELECT host, lower_port, upper_port, ace_order,
       start_date, end_date, grant_option, inverted_principal,
       principal, privilege
FROM dba_host_aces
ORDER BY host, lower_port;

-- どのユーザーがどのホストに接続できるかを確認
SELECT host, lower_port, upper_port, principal, privilege
FROM dba_host_aces
WHERE principal = 'WEBAPP_SVC'
ORDER BY host, lower_port;

-- 特定のユーザーがホストにアクセスできるか確認
SELECT DBMS_NETWORK_ACL_ADMIN.CHECK_PRIVILEGE_ACLID(
  acl         => (SELECT aclid FROM dba_host_acls WHERE host = 'api.payment-gateway.com'),
  user        => 'WEBAPP_SVC',
  privilege   => 'http'
) AS privilege_granted
FROM dual;

-- より簡単な確認 (許可されていれば 1、そうでなければ NULL を返す)
SELECT DBMS_NETWORK_ACL_ADMIN.CHECK_PRIVILEGE(
  acl       => 'host=api.payment-gateway.com',
  user      => 'WEBAPP_SVC',
  privilege => 'http'
) AS granted
FROM dual;

-- ACE の削除
BEGIN
  DBMS_NETWORK_ACL_ADMIN.REMOVE_HOST_ACE(
    host    => 'api.payment-gateway.com',
    lower_port => 443,
    upper_port => 443,
    ace     => xs$ace_type(
      privilege_list => xs$name_list('http'),
      principal_name => 'WEBAPP_SVC',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/
```

### UTL_HTTP からのウォレット・ベースの HTTPS 呼び出し

UTL_HTTP から HTTPS 呼び出しを行い、リモート証明書を検証するには、リモート・サーバーの CA 証明書が Oracle ウォレットに含まれている必要がある。

```sql
-- SSL 検証にウォレットを使用するように UTL_HTTP を構成
BEGIN
  UTL_HTTP.SET_WALLET('file:/opt/oracle/wallet/outbound', 'WalletP@ss!');
END;
/

-- 証明書検証を伴う HTTPS 呼び出しの作成
DECLARE
  v_req  UTL_HTTP.REQ;
  v_resp UTL_HTTP.RESP;
  v_text VARCHAR2(4000);
BEGIN
  v_req  := UTL_HTTP.BEGIN_REQUEST('https://api.payment-gateway.com/v1/charge',
                                     'POST', 'HTTP/1.1');
  UTL_HTTP.SET_HEADER(v_req, 'Content-Type', 'application/json');
  UTL_HTTP.SET_HEADER(v_req, 'Authorization', 'Bearer ' || v_api_token);
  UTL_HTTP.WRITE_TEXT(v_req, '{"amount": 100}');
  v_resp := UTL_HTTP.GET_RESPONSE(v_req);
  UTL_HTTP.READ_TEXT(v_resp, v_text);
  UTL_HTTP.END_RESPONSE(v_resp);
EXCEPTION
  WHEN OTHERS THEN
    IF v_resp.status_code IS NOT NULL THEN
      UTL_HTTP.END_RESPONSE(v_resp);
    END IF;
    RAISE;
END;
/
```

---

## ファイアウォールとネットワーク・アーキテクチャ

### 推奨されるネットワーク・セグメンテーション

```
インターネット
    │
   DMZ
    │ (アプリケーション・ティア)
    ├── Web サーバー (TCP 80/443)
    │
ファイアウォール (アプリ・ティアから DB ティアへは 1521/2484 のみ許可)
    │
   DB ティア
    ├── Oracle プライマリ DB (TCP 1521 / TCPS 2484)
    └── Oracle スタンバイ DB (Data Guard 用の TCP 1521)
    │
   DBA アクセス・ネットワーク (個別 VLAN)
    └── 踏み台（ジャンプ）サーバー → Oracle DB (踏み台からのみ 1521/2484 許可)
```

### Oracle 用のファイアウォール規則

```
# データベース・サーバーへのインバウンド
ALLOW TCP 10.0.2.0/24 → 10.0.3.50:2484   # アプリ・ティアから DB へ (TCPS/TLS)
ALLOW TCP 10.0.4.0/24 → 10.0.3.50:2484   # DBA 踏み台から DB へ
DENY  TCP * → 10.0.3.50:1521             # 外部からの非暗号化ポートをブロック
DENY  TCP * → 10.0.3.50:2484             # その他すべてのインバウンドを拒否

# データベース・サーバーからのアウトバウンド (UTL_HTTP/UTL_SMTP 制限)
ALLOW TCP 10.0.3.50 → 10.0.5.20:25      # 内部の SMTP リレーへのみ許可
ALLOW TCP 10.0.3.50 → 10.0.5.30:443     # 承認された API ゲートウェイへのみ許可
DENY  TCP 10.0.3.50 → *                 # その他すべてのアウトバウンドをブロック
```

---

## Oracle 有効ノード・チェック (IP アロリスト登録)

Oracle Net は、`sqlnet.ora` の `valid_node_checking` パラメータを介して、独自の IP アロリスト登録機能を提供している。

```ini
# /opt/oracle/network/admin/sqlnet.ora

# 有効ノード・チェックを有効化
TCP.VALIDNODE_CHECKING = YES

# 接続を許可する IP アドレスを定義
TCP.INVITED_NODES = (10.0.2.10, 10.0.2.11, 10.0.4.50, 10.0.4.51, 127.0.0.1)

# 明示的にブロックする IP アドレスを定義（INVITED_NODES の代替案）
# TCP.EXCLUDED_NODES = (192.168.100.0/24, 10.255.0.0/16)

# 注: INVITED_NODES と EXCLUDED_NODES は排他的である。
# ホワイトリスト形式のアプローチである INVITED_NODES の方が安全である。
```

有効ノード・チェックは、接続がデータベースに届く前にリスナーによって処理されるため、効率的な最初の防御線となる。

---

## ネットワーク・セキュリティ構成の照会とテスト

```sql
-- データベースから見た現在の sqlnet.ora 暗号化設定を確認
SELECT network_service_banner
FROM v$session_connect_info
WHERE sid = SYS_CONTEXT('USERENV', 'SID');

-- 現在のすべてのセッションの暗号化ステータスを確認
SELECT s.sid, s.serial#, s.username, s.status,
       c.network_service_banner
FROM v$session s
JOIN v$session_connect_info c ON c.sid = s.sid
WHERE s.username IS NOT NULL
  AND c.network_service_banner LIKE '%Encryption%'
ORDER BY s.username;

-- 暗号化されていないセッションを特定
SELECT s.sid, s.serial#, s.username, s.osuser, s.machine, s.program
FROM v$session s
WHERE s.username IS NOT NULL
  AND s.sid NOT IN (
    SELECT c.sid
    FROM v$session_connect_info c
    WHERE c.network_service_banner LIKE '%Encryption service%'
       OR c.network_service_banner LIKE '%AES%'
  )
ORDER BY s.username;

-- リスナーのエンドポイントを確認
SELECT end_point
FROM v$listener_network
ORDER BY end_point;
```

---

## セキュリティ強化（ハーデニング）チェックリスト

```sql
-- 1. 危険なネットワーク・パッケージへの PUBLIC 権限の付与を確認
SELECT grantee, owner, table_name, privilege
FROM dba_tab_privs
WHERE grantee = 'PUBLIC'
  AND table_name IN ('UTL_HTTP', 'UTL_TCP', 'UTL_SMTP', 'UTL_FILE',
                      'UTL_MAIL', 'HTTPURITYPE', 'DBMS_LDAP')
ORDER BY table_name;

-- これらは PUBLIC から取り消し、特定のユーザーにのみ付与すべきである
REVOKE EXECUTE ON utl_http FROM PUBLIC;
REVOKE EXECUTE ON utl_tcp FROM PUBLIC;
REVOKE EXECUTE ON utl_smtp FROM PUBLIC;

-- 2. EXTPROC (外部プロシージャ) の登録を確認
SELECT name, network_name FROM v$service
WHERE name LIKE 'EXTPROC%';
-- 必要なければリスナーから EXTPROC を削除する

-- 3. リモート OS 認証が無効であることを確認 (極めて重要)
SHOW PARAMETER remote_os_authent;
-- FALSE であるべき。TRUE の場合は変更する: ALTER SYSTEM SET remote_os_authent = FALSE;

-- 4. 外部認証アカウント (OS 認証) を確認
SELECT username, external_name, authentication_type
FROM dba_users
WHERE authentication_type = 'EXTERNAL'
ORDER BY username;

-- 5. REMOTE_LOGIN_PASSWORDFILE が EXCLUSIVE または NONE であることを確認
SHOW PARAMETER remote_login_passwordfile;
-- EXCLUSIVE = 1 つの DB につき 1 つの SYSDBA（許容範囲）
-- SHARED = 複数のデータベースでパスワード・ファイルを共有（リスクあり）
-- NONE = パスワード・ファイルなし（OS 認証のみを使用。構成によっては許容範囲）

-- 6. O7_DICTIONARY_ACCESSIBILITY が FALSE であるか確認 (非特権ユーザーによる SYS オブジェクト参照を防止)
SHOW PARAMETER o7_dictionary_accessibility;
-- FALSE であるべき
ALTER SYSTEM SET o7_dictionary_accessibility = FALSE;
```

---

## ベスト・プラクティス

1.  **データベース接続には常に TCPS (TLS) を使用する**: 非暗号化の接続では、資格情報（Oracle Net ではパスワードは独自形式だが、リバース・エンジニアリング可能である）、クエリ・データ、セッション情報がネットワーク盗聴の危険にさらされる。

2.  **TLS 1.2 または 1.3 以上を使用する**: SSL 3.0, TLS 1.0, TLS 1.1 は無効にする。これらには既知の脆弱性（POODLE, BEAST など）がある。

3.  **`TCP.VALIDNODE_CHECKING` を介して接続可能なホストを制限する**: リスナー・レベルでアプリケーション・サーバーの IP をアロリスト登録することで、Oracle の認証レイヤーに届く前に不正なスキャンや接続の試行を遮断できる。

4.  **リスナーをパブリック・インターネットに公開しない**: データベース・リスナーへのアクセスは、アプリケーション・ティア・サーバーからのみ可能にすべきである。外部からのアクセスが必要な場合は、接続プロキシを使用することを検討する。

5.  **明示的に必要でない限り extproc リスナー・エントリを無効にする**: 外部プロシージャ・リスナーは、データベースから OS レベルのコード実行を可能にする一般的な攻撃ベクトルである。

6.  **`UTL_HTTP`, `UTL_TCP`, `UTL_SMTP` へのアクセスは特定のユーザーのみに許可する**: これらのパッケージが悪用されると、機密データを外部へ送信される可能性がある。使用可能なユーザーと宛先を厳格に制御すること。

7.  **`ADMIN_RESTRICTIONS_LISTENER = ON` を設定する**: リモートの `LSNRCTL` コマンドによるリスナーの再構成（例: ログ・パスを変更してシステム・ファイルを上書きするなどの攻撃）を防止する。

8.  **SSL 証明書が期限切れになる前に更新する**: 証明書の期限切れはサービス停止を招く。すべての Oracle ウォレット証明書について、期限の 90 日前にカレンダーのリマインダーを設定しておく。

---

## よくある間違いとその回避方法

### 間違い 1: `SQLNET.ENCRYPTION_SERVER = REQUESTED` を `REQUIRED` の代わりに使用する

```ini
# 不適切: 暗号化をサポートしていないクライアントでも非暗号化接続が許可されてしまう
SQLNET.ENCRYPTION_SERVER = REQUESTED

# 適切: すべての接続に暗号化を必須とし、準拠していないクライアントは拒否する
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_CLIENT = REQUIRED
```

### 間違い 2: tnsnames.ora で SSL_SERVER_CERT_DN を設定していない

`SSL_SERVER_CERT_DN` がないと、クライアントは任意の有効な証明書を受け入れてしまい、中間者攻撃が可能になる。

```ini
# 不適切: サーバーの身元検証が行われない
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCPS)(HOST = db-server)(PORT = 2484))
    (CONNECT_DATA = (SERVICE_NAME = ORCL)))

# 適切: サーバーの証明書 DN を検証する
ORCL =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCPS)(HOST = db-server)(PORT = 2484))
    (CONNECT_DATA = (SERVICE_NAME = ORCL))
    (SECURITY =
      (SSL_SERVER_CERT_DN = "CN=db-server.corp.com,OU=DB,O=Corp,C=US")))
```

### 間違い 3: ホスト ACL アクセスにワイルドカードを許可してしまっている

```sql
-- 不適切: UTL_HTTP が任意のホストを呼び出せるようになってしまう
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host => '*',  -- すべてのホストへのワイルドカード
    ace  => xs$ace_type(
      privilege_list => xs$name_list('connect', 'http'),
      principal_name => 'APP_USER',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/

-- 適切: 既知の特定のホストとポートに制限する
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host       => 'api.known-vendor.com',
    lower_port => 443,
    upper_port => 443,
    ace        => xs$ace_type(
      privilege_list => xs$name_list('http'),
      principal_name => 'APP_USER',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/
```

---

## コンプライアンスの考慮事項

### PCI-DSS
-   要件 2.2: すべてのシステム・コンポーネントに対する構成標準を作成する。
-   要件 4.1: オープンなパブリック・ネットワーク経由で会員データを送信する際は、強力な暗号化とセキュリティ・プロトコルを使用する (TLS 1.2 以上が必須)。
-   要件 6.4: 不要なサービスやポートを無効にする。
-   PCI DSS では、SSL および初期の TLS (1.0, 1.1) は明示的に禁止されている。

### HIPAA
-   45 CFR 164.312(e)(1): ネットワーク越しに送信される ePHI への不正アクセスを防ぐための技術的なセキュリティ手段を実装する。
-   45 CFR 164.312(e)(2)(ii): 転送中の ePHI を保護するための暗号化メカニズムを実装する。
-   HIPAA 準拠には TLS 1.2 以上が必要である。

### SOX
-   送信中にデータの整合性が維持されることを要求。
-   ネットワーク暗号化とチェックサム (`SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED`) の併用によりこれを満たす。

```sql
-- コンプライアンス検証: 本番データベースへのすべてのセッションが暗号化されていることを確認
SELECT COUNT(*) AS unencrypted_sessions
FROM v$session s
WHERE s.username IS NOT NULL
  AND s.sid NOT IN (
    SELECT DISTINCT c.sid
    FROM v$session_connect_info c
    WHERE LOWER(c.network_service_banner) LIKE '%aes%'
       OR LOWER(c.network_service_banner) LIKE '%encryption%'
  );
-- 準拠環境では、これは 0 を返すべきである。
```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

-   このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効である。
-   21c、23c、または 23ai と記された機能は、Oracle Database 26ai 対応機能として扱う。バージョンが混在する環境では、19c 互換の代替案を保持すること。
-   リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンをサポートする環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

## ソース

-   [Oracle Database 19c Security Guide (DBSEG)](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/)
-   [Oracle Database 19c Net Services Administrator's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/)
-   [Oracle Database 19c Net Services Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/)
-   [DBMS_NETWORK_ACL_ADMIN — Oracle Database 19c PL/SQL Packages and Types Reference](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_NETWORK_ACL_ADMIN.html)
