# ORDS アーキテクチャ: コンポーネント、リクエスト・フロー、およびデプロイメント・モデル

## 概要

Oracle REST Data Services (ORDS) は、HTTP クライアントと Oracle Database の間の架け橋として機能する Java EE アプリケーションである。REST API 呼び出しをデータベース操作に変換し、接続プールの管理、認証と認可の処理を行い、結果を JSON などの形式で返す。ORDS は、表やビューに対する AutoREST、カスタム PL/SQL ベースの REST API、Oracle APEX のホスティング、Database Actions (SQL Developer Web)、および Oracle Graph/Spatial REST インターフェースなど、幅広いユースケースをサポートしている。

ORDS のアーキテクチャを理解することは、Oracle Database 上で信頼性とスケーラビリティの高い REST API を設計するために不可欠である。これには、適切なデプロイメント・モデルの選択、接続プールのサイズの適切な設定、URL ルーティングの理解、および Oracle DB と ORDS バージョン間の互換性の維持が含まれる。

---

## 主要コンポーネント

### 1. ORDS アプリケーション層

ORDS は、Java サーブレット・コンテナにデプロイされる WAR (Web Application Archive) ファイルとしてパッケージ化されている。その内部コンポーネントには以下が含まれる。

- **リクエスト・ルーター (Request Router)**: 受信した HTTP リクエストを解析し、登録された URL パターンと照合して、適切なハンドラーに振り分ける。
- **モジュール/テンプレート/ハンドラー・レジストリ**: データベースのメタデータ・スキーマからロードされたすべての REST モジュールのメモリ内表現。設定された間隔でリフレッシュされる。
- **接続プール・マネージャー**: 設定されたデータベース・プールごとに 1 つ以上のデータベース接続プール (UCP) を管理する。
- **認証フィルター (Authentication Filter)**: 保護されたエンドポイントへのアクセスをインターセプトし、Bearer トークンや基本認証の資格証明を検証して、権限チェックを強制する。
- **結果シリアライザー (Result Serializer)**: データベースのクエリ結果（カーソル行、PL/SQL OUT パラメータ、BLOB）を JSON、XML、またはバイナリ出力に変換する。

### 2. ORDS メタデータ・スキーマ

ORDS は、すべての REST API 定義を Oracle Database 自体の中にある 2 つの主要スキーマに保存する。

**ORDS_METADATA** (ORDS によってインストールされる)
- REST モジュール、URL テンプレート、ハンドラー、権限、および OAuth クライアントを定義する表が含まれる。
- 主要な表: `ORDS_METADATA.ORDS_MODULES`, `ORDS_METADATA.ORDS_TEMPLATES`, `ORDS_METADATA.ORDS_HANDLERS`, `ORDS_METADATA.ORDS_PRIVILEGES`
- ORDS は起動時およびリフレッシュ間隔ごとにこのスキーマを読み取り、メモリ内にルーティング表を構築する。

**ORDS_PUBLIC_USER** (データベース・ユーザー)
- ORDS がルーティングの決定や AutoREST メタデータのルックアップに使用する、低権限のデータベース・ユーザー。
- 実際の REST ハンドラーの SQL は実行**しない**。ハンドラーの SQL は、スキーマ所有者または認証コンテキストによってマップされたユーザーとして実行される。
- `CREATE SESSION` 権限と ORDS_METADATA からの付与が必要である。

### 3. データベース接続プール

ORDS は、Oracle の JDBC 接続プールである **Universal Connection Pool (UCP)** を使用する。ORDS 構成内の各データベース・プールには、独自の UCP インスタンスが割り当てられる。プール・パラメータにより以下を制御する。

- 最小/最大プール・サイズ
- 接続のタイムアウトと検証
- ステートメント・キャッシュ・サイズ

```shell
# プールの設定は ORDS CLI を介して構成する（設定ファイルを直接編集しない）
ords config set db.hostname mydb.example.com
ords config set db.port 1521
ords config set db.servicename mypdb.example.com
ords config set db.username ORDS_PUBLIC_USER
# パスワードは Oracle ウォレットに保存され、設定ファイルには直接記述しない
ords config secret set db.password --password-stdin <<< "..."
ords config set feature.sdw true
```

### 4. REST 有効化されたスキーマごとの ORDS スキーマ・オブジェクト

スキーマが REST 有効化されると (`ORDS.ENABLE_SCHEMA`)、以下のメタデータが ORDS_METADATA に作成される。
- スキーマの別名 (Alias) と DB ユーザー名をリンクするスキーマ・レコード
- 明示的に定義された REST API のモジュール、テンプレート、およびハンドラー・レコード
- 有効化されたオブジェクト（表、ビュー、プロシージャ）の AutoREST メタデータ

---

## リクエスト・フロー

リクエストのライフサイクルを理解することは、トラブルシューティングやパフォーマンス・チューニングに役立つ。

```
HTTP クライアント
    │
    ▼
[ロード・バランサ / リバース・プロキシ] (オプション)
    │
    ▼
[ORDS サーブレット・コンテナ] ─── HTTPS 終端 (またはパススルー)
    │
    ├─ 1. URL 解析とルーティング
    │      メモリ内のモジュール/テンプレート・パターンと URL を照合
    │
    ├─ 2. 認証フィルター
    │      エンドポイントが保護されているかチェック
    │      OAuth Bearer トークンまたは基本認証を検証
    │      権限/ロールを解決
    │
    ├─ 3. 接続の取得
    │      ターゲット・プール用の JDBC 接続を UCP から取得
    │
    ├─ 4. ハンドラーの実行
    │      バインド・パラメータを使用して SQL クエリ / PL/SQL ブロックを実行
    │      暗黙的なパラメータ（:body, :body_text など）が注入される
    │
    ├─ 5. 結果のシリアライズ
    │      カーソル行 → JSON オブジェクトの配列
    │      BLOB → Content-Type ヘッダーを伴うバイナリ・ストリーム
    │      PL/SQL OUT パラメータ → JSON オブジェクト
    │
    └─ 6. レスポンス
           HTTP ステータス、ヘッダー、ボディをクライアントに返却
```

典型的な REST ハンドラーの実行例:
1. ORDS が `GET /ords/hr/employees/101` を受信
2. ルーターがこれをモジュール `hr`、テンプレート `employees/:id`、ハンドラー `GET` に照合
3. 認証チェック（権限が付与されている場合）により Bearer トークンを検証
4. ORDS が `default` UCP プールから DB 接続を取得
5. `SELECT * FROM employees WHERE employee_id = :id` を `:id = 101` で実行
6. 結果の行を JSON オブジェクトとしてシリアライズ
7. `200 OK` と JSON ボディを返却

---

## URL ルーティングとモジュール/テンプレート/ハンドラーの階層

ORDS REST API は、以下の 3 レベルの階層で整理されている。

```
モジュール (ベース・パス)
  └── テンプレート (相対 URL パターン)
        └── ハンドラー (HTTP メソッド + SQL/PL/SQL)
```

### モジュール (Module)

モジュールは最上位のコンテナである。ベース・パスを定義し、1 つのスキーマに関連付けられる。

```
ベース URL: /ords/{schema_alias}/{module_base_path}/
例:       /ords/hr/api/v1/
```

### テンプレート (Template)

テンプレートは、モジュールのベース・パスからの相対的な URL パターンを定義する。テンプレートは `:param` 構文を使用した **URI パラメータ** をサポートしている。

```
テンプレート: employees/
テンプレート: employees/:id
テンプレート: departments/:dept_id/employees
```

### ハンドラー (Handler)

ハンドラーは、特定のテンプレートの特定の HTTP メソッドに対して実行される実際の SQL または PL/SQL である。

```
GET  employees/        → コレクションを返す SELECT クエリ
GET  employees/:id     → 単一行を返す SELECT クエリ
POST employees/        → PL/SQL による INSERT
PUT  employees/:id     → PL/SQL による UPDATE
DELETE employees/:id   → PL/SQL による DELETE
```

### 完全な URL の構築

```
https://host:port/ords/{schema_alias}/{module_uri_prefix}/{template_uri}
                   │         │                │                  │
                   │    hr (schema)      api/v1/         employees/:id
                   │
               固定の ORDS プレフィックス
```

結果: `https://host:8443/ords/hr/api/v1/employees/101`

---

## デプロイメント・モデル

### 1. スタンドアロン (Jetty) — 開発および本番環境

ORDS には、組み込みの Eclipse Jetty サーバーが同梱されている。外部のアプリケーション・サーバーは不要である。

```shell
# ORDS スタンドアロンの起動
ords --config /opt/oracle/ords/config serve \
  --port 8080 \
  --secure-port 8443 \
  --certificate-hostname myserver.example.com
```

**長所**: シンプル、低オーバーヘッド、自動化が容易で、本番環境でも公式にサポートされている。
**短所**: 単一の JVM プロセス。非常に高い可用性が必要な場合は、ロード・バランサの背後で複数のインスタンスを実行する。

### 2. Apache Tomcat

ORDS WAR ファイルを Tomcat の `webapps/` ディレクトリにデプロイする。

```shell
cp /opt/oracle/ords/ords.war $CATALINA_HOME/webapps/ords.war
# 環境変数で ORDS 構成ディレクトリを設定
export ORDS_CONFIG=/opt/oracle/ords/config
```

Tomcat がスレッディング、SSL/TLS (コネクタ経由)、および JVM 管理を処理する。

**長所**: Java 運用チームになじみがある。Tomcat の監視ツールと統合できる。
**短所**: Tomcat の管理スキルの習得が必要。ORDS アップグレード時に WAR の再デプロイが必要。

### 3. Oracle WebLogic Server

ORDS WAR を WebLogic 管理サーバーまたはクラスターにデプロイする。WebLogic をすでに使用しているエンタープライズ環境で使用される。

**長所**: エンタープライズ・クラスター化、WebLogic 監視、Oracle サポートのバンドル。
**短所**: 多大な運用オーバーヘッド。ほとんどの ORDS デプロイメントには過剰。

### 4. Oracle Cloud Infrastructure (OCI)

#### Autonomous Database (ADB) 上の ORDS

ADB (ATP/ADW) には ORDS がプリインストールおよび事前構成されている。インストール作業は不要である。

- ORDS には、OCI コンソールに表示される ADB 専用の ORDS URL を介してアクセスする。
- 内部接続には mTLS ウォレットを使用する（手動のプール構成は不要）。
- ORDS メタデータは同じ ADB インスタンスに保存される。
- Oracle が ORDS のバージョン・アップグレードを管理する。

```
https://<unique-id>.adb.<region>.oraclecloudapps.com/ords/
```

#### OCI Compute / Container Instances 上の ORDS

ORDS は OCI Compute VM、または OCI Container Instances / OKE (Kubernetes) 内のコンテナとしてデプロイできる。Oracle Container Registry の公式 ORDS コンテナ・イメージを使用する。

```shell
docker pull container-registry.oracle.com/database/ords:latest

docker run -d \
  --name ords \
  -p 8080:8080 \
  -e ORACLE_HOST=mydb.example.com \
  -e ORACLE_PORT=1521 \
  -e ORACLE_SERVICE=mypdb.example.com \
  -v /opt/ords/config:/opt/oracle/ords/config \
  container-registry.oracle.com/database/ords:latest
```

---

## Autonomous Database (ADB) における ORDS

ADB はフルマネージドな ORDS デプロイメントを提供する。セルフマネージド ORDS との主な違いは以下の通り。

| 機能 | セルフマネージド ORDS | ADB ORDS |
|---|---|---|
| インストール | 手動 | プリインストール |
| 接続プール | 手動で構成 | 自動構成 |
| TLS | 手動の証明書管理 | Oracle 管理 |
| バージョン・アップグレード | 手動 | Oracle 管理 |
| スキーマ URL | `/ords/{alias}/` | `/ords/{alias}/` (同一) |
| Database Actions | オプション | 同梱 |
| カスタム・プール・パラメータ | 全制御可能 | DB パラメータにより制限あり |

ADB では、セルフマネージド ORDS と同様に `ORDS.ENABLE_SCHEMA`、`ORDS.DEFINE_MODULE` などを介してスキーマを REST 有効化し、REST API を定義する。SQL 構文は同一である。

---

## バージョン互換性

ORDS は Oracle Database とは別のリリース・サイクルに従っている。一般的なルールは以下の通り。

- ORDS は**前方互換性**がある: 新しいバージョンの ORDS は古いデータベース・バージョンでも動作する。
- ORDS の最小データベース要件: Oracle Database 11.2.0.4 (12c 推奨)。
- ADB 用の ORDS は常に最新の検証済みバージョンを実行する。
- ORDS 22.x 以降は `ords` CLI を使用する（従来の `java -jar ords.war` コマンドを置き換え）。
- ORDS 23.x では、強化された JWT/OAuth2 機能と OpenAPI 3.0 サポートの向上が導入された。

```
推奨される互換性マトリックス:
Oracle DB 19c / 21c / 23ai → ORDS 22.x, 23.x, 24.x (最新推奨)
Oracle DB 12.2              → ORDS 21.x 以降
Oracle DB 12.1              → ORDS 19.x 以降 (一部機能制限あり)
```

アップグレードの前に、必ず [ORDS リリース・ノート](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/)で特定のバージョンのマトリックスを確認すること。

---

## ベスト・プラクティス

- **ORDS をデータベースの近くにデプロイする**: ORDS と DB 間の高レイテンシはパフォーマンスを低下させる。同じデータ・センター / VCN を使用すること。
- **本番環境の高可用性のために、ロード・バランサの背後で複数の ORDS インスタンスを実行する**: ORDS はステートレスであり、どのインスタンスでも任意のリクエストを処理できる。
- **DB サーバーとは別の専用 ORDS サーバーを使用する**: 負荷のかかった ORDS は、JVM 内でかなりの CPU とメモリを消費する。
- **ORDS を最新の状態に保つ**: セキュリティ・パッチやパフォーマンス向上のためのアップデートが頻繁にリリースされている。ORDS 22 以降は、単一のコマンドでインプレース アップグレードをサポートしている。
- **UCP プール・サイズを調整する**: DB の接続制限と予想される同時実行数に基づいて設定する。デフォルトの最大プール・サイズ (10) は本番環境には小さすぎる。
- **最小権限の ORDS_PUBLIC_USER を使用する**: DBA アカウントを ORDS 接続ユーザーとして使用してはならない。
- **ORDS ログを監視する**: リクエスト・ログを有効にし、遅いリクエストのログを確認してパフォーマンスのボトルネックを特定する。

## よくある間違い

- **SYS または SYSTEM アカウントを ORDS 接続プール・ユーザーとして使用する**: これは重大なセキュリティ・リスクである。必ず専用の低権限ユーザーを使用すること。
- **メタデータ変更後に ORDS のリフレッシュを忘れる**: ORDS_METADATA 表で REST API 定義が直接変更された場合（SQL などを使用した場合）、ORDS はメモリ内のキャッシュをリフレッシュする必要がある。自動的にリフレッシュをトリガーする `ORDS.DEFINE_*` API を使用するか、ORDS を再起動すること。
- **スキーマの別名 (Alias) とスキーマ名を混同する**: URL 内のスキーマの別名（例: `/ords/hr/`）はスキーマの有効化時に設定され、データベース・ユーザー名と異なる場合がある。不一致は 404 エラーの原因となる。
- **ORDS と APEX が同じバージョンであることを前提にする**: ORDS と APEX には、それぞれ独立したバージョンの依存関係がある。APEX リリース・ノートで必要な最小 ORDS バージョンを確認すること。
- **HTTPS を構成しない**: スタンドアロン・モードの ORDS はデフォルトで HTTP になっている。開発環境以外では必ず HTTPS を構成すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替手段を維持すること。
- 19c と 26ai の両方をサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [Oracle REST Data Services Documentation](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/)
- [ORDS Developer's Guide — Architecture Overview](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/index.html)
- [ORDS Release Notes](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/)
