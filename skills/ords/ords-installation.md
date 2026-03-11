# ORDS のインストールと構成

## 概要

Oracle REST Data Services (ORDS) のインストールには、ソフトウェアのダウンロード、Oracle Database の準備、ORDS メタデータ・スキーマを作成または更新するためのインストール・コマンドの実行、データベース接続プールの構成、およびオプションでの HTTPS のセットアップが含まれる。このガイドでは、オンプレミス、OCI Compute、またはコンテナ環境における、セルフマネージド（自己管理型）の ORDS インストールについて説明する。Autonomous Database の場合は ORDS がプリインストールされているため、このガイドの内容は適用されない。

---

## 前提条件

### ソフトウェア要件

| コンポーネント | 最小プログラム。 |
|---|---|
| Java (JDK/JRE) | 11 (ORDS 23+ では 17 以上を推奨) |
| Oracle Database | 11.2.0.4 (12c 以上を強く推奨) |
| ORDS | 22.x 以降 (最新版の使用を推奨) |
| OS | Linux x86-64, Windows, macOS (開発用のみ) |

続行する前に Java のバージョンを確認すること。

```shell
java -version
# 出力例:
# java version "17.0.8" 2023-07-18 LTS
# Java(TM) SE Runtime Environment (Oracle) ...
```

### データベースの前提条件

ORDS のインストールには、ORDS_METADATA スキーマと ORDS_PUBLIC_USER アカウントを作成するために、DBA または同等の権限を持つデータベース・ユーザーが必要である。以下の準備を行うこと。

1. ORDS をインストールするための PDB (プラグ可能データベース) または非 CDB。
2. 初回インストール用の SYS または DBA 権限を持つアカウントの資格証明。
3. 十分な表領域: ORDS_METADATA スキーマには約 50MB が必要。

```sql
-- 正しい PDB に接続していることを確認
SHOW CON_NAME;

-- 利用可能な表領域を確認
SELECT tablespace_name, bytes/1024/1024 AS mb_free
FROM dba_free_space
WHERE tablespace_name = 'SYSAUX'
ORDER BY 1;
```

---

## ORDS のダウンロード

ORDS は以下の場所から入手可能である。
- [Oracle Technology Network (OTN)](https://www.oracle.com/database/sqldeveloper/technologies/db-actions/download/)
- Oracle Maven リポジトリ（CI/CD パイプライン用）
- Oracle Container Registry（Docker イメージ）

```shell
# ords-latest.zip をダウンロード後、解凍する
unzip ords-latest.zip -d /opt/oracle/ords
ls /opt/oracle/ords
# ords.war   ords   docs/   ...

# ORDS の bin ディレクトリを PATH に追加
export PATH=$PATH:/opt/oracle/ords/bin
echo 'export PATH=$PATH:/opt/oracle/ords/bin' >> ~/.bashrc

# 確認
ords --version
# Oracle REST Data Services 24.x.x ...
```

---

## 構成ディレクトリの構造

ORDS は、すべての設定を保存するために（ソフトウェアのインストール先とは別の）構成ディレクトリを使用する。このディレクトリはアップグレード後も維持する必要がある。

```shell
# 構成ディレクトリの作成
mkdir -p /opt/oracle/ords/config
export ORDS_CONFIG=/opt/oracle/ords/config
```

インストール後の構成ディレクトリは以下のようになる。

```
/opt/oracle/ords/config/
├── databases/
│   └── default/
│       └── pool.json         # 接続プールの設定 (CLI で管理。直接編集しないこと)
├── global/
│   └── settings.json         # グローバルな ORDS 設定 (CLI で管理)
└── credentials              # Oracle ウォレット・ディレクトリ。パスワードはここに保存される (JSON には保存されない)
```

> すべての設定ファイルは、`ords config set` CLI によって管理される。JSON ファイルを手動で編集しないこと。パスワードは Oracle ウォレット (`credentials/`) に保存され、構成ファイルには一切表示されない。

---

## `ords install` コマンドの手順

### 対話型インストール

初回インストール時は、対話形式でインストーラーを実行する。

```shell
ords --config /opt/oracle/ords/config install
```

対話型インストーラーでは、以下の入力を求められる。

1. **接続タイプ**: 基本 (ホスト名/ポート/サービス名)、TNS 名、またはカスタム JDBC URL
2. **データベース・ホスト**: 例: `mydb.example.com`
3. **データベース・ポート**: 例: `1521`
4. **データベース・サービス名**: 例: `mypdb.example.com`
5. **ORDS 管理ユーザー**: 通常、初回インストール時は `SYS AS SYSDBA`
6. **SYS パスワード**
7. **ORDS ランタイム・ユーザー** (ORDS_PUBLIC_USER のパスワード): 強力なパスワードを設定する
8. **ORDS メタデータ用の表領域**: デフォルトは SYSAUX
9. **有効にする機能**: Database Actions、REST Enabled SQL など

インストール後、ORDS は以下を作成する。
- データベース内の `ORDS_METADATA` スキーマ
- `ORDS_PUBLIC_USER` データベース・アカウント
- 構成ディレクトリ内のプール構成ファイル

### 自動化のためのサイレント/非対話型インストール

CI/CD パイプライン、自動プロビジョニング、または Ansible/Terraform ワークフローの場合、レスポンス・ファイルや環境変数を使用したサイレント・インストールを使用する。

**方法 1: 標準入力 (stdin) 経由でパスワードを渡す**

```shell
ords --config /opt/oracle/ords/config install \
  --admin-user SYS \
  --db-hostname mydb.example.com \
  --db-port 1521 \
  --db-servicename mypdb.example.com \
  --feature-db-api true \
  --feature-rest-enabled-sql true \
  --feature-sdw true \
  --log-folder /var/log/ords \
  --password-stdin <<EOF
SysPassword123!
OrdsPublicUserPwd456!
EOF
```

**方法 2: `ords install --interactive false` を使用する**

```shell
ords --config /opt/oracle/ords/config install \
  --interactive false \
  --db-hostname mydb.example.com \
  --db-port 1521 \
  --db-servicename mypdb.example.com \
  --db-username ORDS_PUBLIC_USER \
  --admin-user SYS \
  --feature-sdw true
# パスワードは別途、または環境変数経由で入力を求められる
```

**方法 3: 事前にプール構成を作成し、`--db-only` でインストールする**

最初にプール構成を書き込み、次にデータベースのインストール・フェーズのみを実行する。

```shell
ords --config /opt/oracle/ords/config config set db.hostname mydb.example.com
ords --config /opt/oracle/ords/config config set db.port 1521
ords --config /opt/oracle/ords/config config set db.servicename mypdb.example.com

# 標準入力経由でパスワードを設定
echo "SysPassword123!" | ords --config /opt/oracle/ords/config install \
  --admin-user "SYS AS SYSDBA" \
  --password-stdin
```

---

## プール構成リファレンス

すべてのプール設定は、`ords config set` CLI を介してのみ管理される。**パスワードは `credentials/` ディレクトリ内の Oracle ウォレットに保存される**ため、いかなる構成ファイルにも表示されない。ORDS が生成する JSON 構成ファイルを手動で編集してはならない。

```shell
# 接続パラメータの設定
ords --config /opt/oracle/ords/config config set db.hostname mydb.example.com
ords --config /opt/oracle/ords/config config set db.port 1521
ords --config /opt/oracle/ords/config config set db.servicename mypdb.example.com
ords --config /opt/oracle/ords/config config set db.username ORDS_PUBLIC_USER

# パスワードの設定 — Oracle ウォレットに保存され、設定ファイルには非表示
ords --config /opt/oracle/ords/config config secret set db.password \
  --password-stdin <<< "MySecurePassword123!"

# UCP プール・サイズの設定
ords --config /opt/oracle/ords/config config set jdbc.InitialLimit 5
ords --config /opt/oracle/ords/config config set jdbc.MinLimit 5
ords --config /opt/oracle/ords/config config set jdbc.MaxLimit 30

# 機能フラグの設定
ords --config /opt/oracle/ords/config config set feature.sdw true
```

主要なパラメータ:

| パラメータ | 説明 | 推奨値 |
|---|---|---|
| `jdbc.InitialLimit` | 起動時に作成される接続数 | 5-10 |
| `jdbc.MinLimit` | 維持される最小プール・サイズ | 5-10 |
| `jdbc.MaxLimit` | 最大接続数 (ハード制限) | 20-50 (DB の上限に合わせる) |
| `jdbc.statementTimeout` | アイドルなステートメントが閉じられるまでの秒数 | 900 |
| `jdbc.InactivityTimeout` | アイドルな接続が閉じられるまでの秒数 | 1800 |
| `db.connectionType` | basic / tns / customurl | basic |

---

## mTLS 用の Oracle ウォレット・セットアップ (ATP/ADW)

ORDS を Autonomous Database (ATP または ADW) に接続する場合、mTLS にはウォレットが必要である。これはセルフパネージドな ORDS を ADB に対して実行する場合に適用される。

### ステップ 1: ウォレットのダウンロード

OCI コンソール → Autonomous Database → DB 接続 から `Wallet_<DBName>.zip` をダウンロードする。

```shell
mkdir -p /opt/oracle/ords/wallet
unzip Wallet_MYATP.zip -d /opt/oracle/ords/wallet
ls /opt/oracle/ords/wallet
# cwallet.sso  ewallet.p12  keystore.jks  ojdbc.properties
# sqlnet.ora   tnsnames.ora  truststore.jks
```

### ステップ 2: mTLS 用の ORDS プール構成

```shell
# 接続タイプを TNS に設定
ords --config /opt/oracle/ords/config config set db.connectionType tns

# ウォレット・ディレクトリ (tnsnames.ora と sqlnet.ora を含む) を指定
ords --config /opt/oracle/ords/config config set db.tnsAliasName myatp_high
ords --config /opt/oracle/ords/config config set \
  db.wallet.zip.path /opt/oracle/ords/wallet/Wallet_MYATP.zip

# または TNS_ADMIN 環境変数を設定
export TNS_ADMIN=/opt/oracle/ords/wallet
```

CLI を使用してウォレットベースの接続プールを構成する:

```shell
ords --config /opt/oracle/ords/config config set db.connectionType tns
ords --config /opt/oracle/ords/config config set db.tnsAliasName myatp_high
ords --config /opt/oracle/ords/config config set \
  db.wallet.zip.path /opt/oracle/ords/wallet/Wallet_MYATP.zip
ords --config /opt/oracle/ords/config config set db.username ORDS_PUBLIC_USER
ords --config /opt/oracle/ords/config config secret set db.password \
  --password-stdin <<< "MySecurePassword123!"
```

---

## HTTPS/TLS の構成

### オプション A: 自己署名証明書を使用した ORDS スタンドアロン (開発用)

```shell
# 自己署名証明書の生成
keytool -genkeypair \
  -alias ords-ssl \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365 \
  -keystore /opt/oracle/ords/config/ords/standalone/ords.jks \
  -storepass changeit \
  -dname "CN=myserver.example.com, OU=ORDS, O=MyOrg, C=US"
```

```shell
# スタンドアロン・モードで使用するように構成
ords --config /opt/oracle/ords/config config set \
  standalone.https.port 8443
ords --config /opt/oracle/ords/config config set \
  standalone.https.cert /opt/oracle/ords/config/ords/standalone/ords.jks
ords --config /opt/oracle/ords/config config set \
  standalone.https.cert.secret changeit
```

### オプション B: リバース・プロキシの背後にある ORDS (本番環境推奨)

TLS を終端する Nginx や Apache HTTPD の背後で、ORDS を HTTP (ポート 8080) で実行する。ORDS は内部でプレーンな HTTP を受信する。

```nginx
# /etc/nginx/conf.d/ords.conf
server {
    listen 443 ssl;
    server_name api.mycompany.com;

    ssl_certificate     /etc/ssl/certs/mycompany.crt;
    ssl_certificate_key /etc/ssl/private/mycompany.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location /ords/ {
        proxy_pass         http://localhost:8080/ords/;
        proxy_set_header   Host $host;
        proxy_set_header   X-Forwarded-For $remote_addr;
        proxy_set_header   X-Forwarded-Proto https;
        proxy_read_timeout 300;
        proxy_send_timeout 300;
    }
}
```

転送されたプロトコル・ヘッダーを信頼するように ORDS を設定する:

```shell
ords --config /opt/oracle/ords/config config set \
  security.forceHTTPS true
```

---

## ORDS の開始と停止

```shell
# フォアグラウンドで開始 (開発用)
ords --config /opt/oracle/ords/config serve

# 特定のポートを指定して開始
ords --config /opt/oracle/ords/config serve --port 8080

# バックグラウンド・サービスとして実行 (Linux systemd)
```

```ini
# /etc/systemd/system/ords.service
[Unit]
Description=Oracle REST Data Services
After=network.target

[Service]
Type=simple
User=oracle
Environment="ORDS_CONFIG=/opt/oracle/ords/config"
ExecStart=/opt/oracle/ords/bin/ords --config /opt/oracle/ords/config serve
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```shell
systemctl daemon-reload
systemctl enable ords
systemctl start ords
systemctl status ords
```

---

## ORDS のアップグレード

ORDS のアップグレードは 2 つのフェーズで行われる。ソフトウェアの更新と、それに続くデータベース・メタデータ・スキーマのアップグレードである。

```shell
# 1. ORDS の停止
systemctl stop ords

# 2. 新しい ORDS バージョンのダウンロードと解凍
unzip ords-24.x.x.zip -d /opt/oracle/ords_new

# 3. 新しいバージョンを指すように PATH を更新
export PATH=/opt/oracle/ords_new/bin:$PATH

# 4. データベース内の ORDS スキーマをアップグレード
# (既存のプール構成を使用するため、再インストールは不要)
ords --config /opt/oracle/ords/config install \
  --admin-user "SYS AS SYSDBA" \
  --db-hostname mydb.example.com \
  --db-port 1521 \
  --db-servicename mypdb.example.com

# 5. 新しいバージョンの ORDS を開始
systemctl start ords

# 6. 確認
ords --version
curl http://localhost:8080/ords/_/db-api/stable/metadata-catalog/
```

`ords install` コマンドはべき等（同じ結果を保証）である。すでにスキーマが存在する場合はアップグレードが行われ、存在しない場合は新規に作成される。

---

## ベスト・プラクティス

- **構成ディレクトリをソフトウェア・ディレクトリから分離する**: 構成ディレクトリは、ソフトウェアのアップグレード後も保持される必要がある。`/opt/oracle/ords/config` などに保存し、ソフトウェア・インストール・ディレクトリ内には置かないこと。
- **すべての構成変更には ORDS CLI を使用する**: XML ファイルの手動編集は避ける。CLI がウォレット管理、スキーマ検証、および構成のリフレッシュを処理する。
- **パスワードは Oracle ウォレットに保存する**: ORDS は、構成ディレクトリ内の Oracle ウォレット (`credentials/`) にすべてのパスワードを保存する。パスワードが構成ファイルに表示されることは一切ない。資格証明の設定やローテーションには、常に `ords config secret set db.password` を使用すること。構成ファイルにパスワードを直接書き込もうとしてはならない。
- **ADB には TNS 別名を使用する**: TNS 別名を使用したウォレットベースの接続は、カスタム JDBC URL よりも保守性が高い。
- **構成変更後は `ords validate` でテストする**: `ords --config /path/config validate` を実行すると、プールの接続性がチェックされ、問題が報告される。
- **DB の最大セッション数に基づいて `jdbc.MaxLimit` を設定する**: 制限が高すぎると DB 接続を使い果たす可能性がある。`SELECT * FROM v$resource_limit WHERE resource_name = 'sessions'` で制限を確認すること。
- **ORDS ログ・ディレクトリを高速なストレージに配置する**: 高スループットの ORDS サーバーは大量のログを出力する。ログ・ディレクトリには SSD ベースのボリュームを使用すること。

## よくある間違い

- **ORDS ソフトウェアのアップグレード後に DB スキーマをアップグレードしていない**: 新しい ORDS を古いスキーマで実行するとエラーが発生する。バイナリをアップグレードした後は、必ず `ords install` を実行すること。
- **`jdbc.MaxLimit` の設定が低すぎる (デフォルトは 10)**: デフォルト値は開発用には適切だが、本番環境では同時実行数に応じて 20 以上、場合によっては 100 以上が必要になる。
- **`TNS_ADMIN` を設定せずに TNS を使用している**: ウォレット・ディレクトリのパスが設定されていても `TNS_ADMIN` が設定されていない場合、Oracle JDBC は `tnsnames.ora` を見つけることができない。
- **root ユーザーで ORDS を実行している**: 不要な権限を持たない専用の OS ユーザー (`oracle` や `ords`) で常に実行すること。
- **ファイアウォール・ポートの開放を忘れている**: ORDS スタンドアロンのデフォルト・ポートは 8080 (HTTP) と 8443 (HTTPS) である。OS のファイアウォールやクラウドのセキュリティ・グループでこれらの通過が許可されていることを確認すること。
- **CDB ルートにインストールしている**: ORDS は常に PDB にインストールし、CDB ルートへはインストールしないこと。CDB ルートの ORDS_METADATA は、スキーマ解決の問題を引き起こす。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替手段を維持すること。
- 19c と 26ai の両方をサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle REST Data Services Installation and Configuration Guide](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/index.html)
- [ORDS CLI Reference — ords install](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/installing-oracle-rest-data-services.html)
- [ORDS Configuration Settings Reference](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/configuration-settings.html)
