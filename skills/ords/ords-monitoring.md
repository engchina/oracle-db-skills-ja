# ORDS のモニタリング、パフォーマンス・チューニング、およびトラブルシューティング

## 概要

効率的な ORDS の運用には、リクエスト・パターン、接続プールの状態、スロー・クエリ、およびエラー率の可視化が必要である。ORDS は、モニタリング用にログ・ファイル、ステータス・エンドポイント、およびデータベース・ビューを提供している。このガイドでは、ログ・ファイルの解釈、接続プールの監視、パフォーマンス・チューニング・パラメータ、一般的なエラー・パターンの診断、およびロード・バランサーや稼働監視（アップタイム・モニター）用のヘルス・チェックのセットアップについて説明する。

---

## ORDS ログ・ファイルの場所と構成

### ログ・ディレクトリ

デフォルトでは、ORDS は構成ディレクトリ内の `logs/` ディレクトリにログを書き込む。

```
/opt/oracle/ords/config/
└── logs/
    ├── ords/
    │   ├── ords_log_2024-01-15.log     # メインのアプリケーション・ログ
    │   ├── ords_log_2024-01-15.log.1   # ロールオーバーされたログ
    │   └── ...
    └── ...
```

カスタム・ログ・パスを構成する:

```shell
ords --config /opt/oracle/ords/config config set log.dir /var/log/ords
```

または開始時に指定する:

```shell
ords --config /opt/oracle/ords/config serve \
  --log-folder /var/log/ords
```

### ログ・レベル

ORDS は Java Logging (JUL) を使用しており、以下のレベルを構成できる。

```shell
# グローバルなログ・レベルの設定
ords --config /opt/oracle/ords/config config set log.level INFO

# 利用可能なレベル: FINEST, FINER, FINE, CONFIG, INFO, WARNING, SEVERE
# FINE = デバッグ・レベル。個々の SQL 実行を表示
# INFO = 通常の運用状況
# WARNING = 回復可能なエラー
# SEVERE = 致命的なエラー
```

本番環境では `INFO` レベル、トラブルシューティング時には `FINE` または `FINER` を推奨する。

---

## リクエスト・ロギングの有効化

リクエスト・ロギングは、各 HTTP リクエスト의 URL、メソッド、ステータス・コード、レスポンス時間、およびクライアント情報を記録する。

```shell
# ORDS CLI を介して外部エラー・パスを構成
ords --config /opt/oracle/ords/config config set error.externalPath /var/log/ords/errors
```

ORDS CLI を介してリクエスト・ロギングを構成する:

```shell
# アクセス/リクエスト・ログの有効化
ords --config /opt/oracle/ords/config config set log.logging.requestlog true

# ログ形式 (combined = Apache combined ログ形式)
ords --config /opt/oracle/ords/config config set log.logging.requestlog.format combined

# 5000ミリ秒より遅いリクエストを「slow」としてログ出力する
ords --config /opt/oracle/ords/config config set log.logging.requestlog.slow 5000
```

### リクエスト・ログのエントリ例

```
2024-01-15 14:23:05.123 [INFO] [worker-5] oracle.dbtools.http.ords.RequestLog
  method=GET path=/ords/hr/v1/employees/ status=200
  elapsed=145ms rows=25 pool=default user=reporting-svc
  remote=10.0.1.42 userAgent=MyApp/1.0
```

フィールド名:
- `elapsed`: ハンドラー実行の合計時間（ミリ秒）
- `rows`: 返された結果の行数
- `pool`: 使用された接続プールの名前
- `user`: 認証されたユーザー名（公開リクエストの場合は `oracle`）

---

## ORDS ログの解釈

### 起動時のメッセージ

```
2024-01-15 09:00:01.001 INFO  Starting Oracle REST Data Services version 24.1.0
2024-01-15 09:00:01.145 INFO  Configuration directory: /opt/oracle/ords/config
2024-01-15 09:00:02.300 INFO  Pool: default - initialized, 5 connections
2024-01-15 09:00:02.410 INFO  Listening on port 8080 (HTTP)
2024-01-15 09:00:02.412 INFO  Listening on port 8443 (HTTPS)
```

### 接続プールに関するメッセージ

```
# プールの拡張 (負荷時の正常な動作)
2024-01-15 14:25:01.000 INFO  Pool: default - growing pool, connections=8/30

# プールの枯渇 (警告: すべての接続が使用中)
2024-01-15 14:25:30.000 WARNING Pool: default - pool exhausted, waiting for connection
  waitTime=2341ms maxWait=60000ms

# 接続検証の失敗 (DB に到達不能、または接続が切断された状態)
2024-01-15 14:30:00.000 WARNING Pool: default - connection validation failed
  ORA-03114: not connected to ORACLE
```

### 認証エラー

```
# 無効または期限切れのトークン
2024-01-15 15:00:01.000 INFO  Authentication failed: invalid_token
  path=/ords/hr/v1/employees/ realm=hr

# 保護されたエンドポイントでのトークン不足
2024-01-15 15:00:02.000 INFO  Authorization required: hr.employees.read
  path=/ords/hr/v1/employees/ client=10.0.1.42
```

### SQL 実行エラー

```
# ORA-00942: 表またはビューが存在しない
2024-01-15 16:00:01.000 SEVERE SQL execution failed
  path=/ords/hr/v1/employees/ method=GET pool=default
  ORA-00942: table or view does not exist

# ORA-01403: データが見つからない (collection_item の場合)
2024-01-15 16:00:05.000 INFO  Resource not found
  path=/ords/hr/v1/employees/999 → HTTP 404
```

---

## スロー・リクエスト・ログ

ORDS は、設定された閾値を超えるリクエストを、詳細情報とともにログ出力する。

```
2024-01-15 14:50:00.000 WARNING Slow request detected
  method=GET path=/ords/hr/v1/reports/annual-summary
  elapsed=12450ms threshold=5000ms pool=default
  sql="SELECT ... FROM orders o JOIN ..."
```

スロー・リクエストの調査方法:
1. ログから SQL をコピーする。
2. SQL Developer または SQLcl で `EXPLAIN PLAN` を使用して実行する。
3. 全表スキャン、索引の不足、またはデカルト結合がないか確認する。
4. URL からバインド変数の値を適用し、状況を正確に再現する。

---

## ORDS ステータス・エンドポイントによる接続プールの監視

### ORDS ステータスおよびヘルス・エンドポイント

```http
GET /ords/_/db-api/stable/database/ HTTP/1.1
Authorization: Bearer <admin-token>
```

レスポンス例:

```json
{
  "items": [
    {
      "poolName": "default",
      "status": "UP",
      "connections": {
        "opened": 12,
        "available": 8,
        "busy": 4,
        "max": 30
      },
      "statistics": {
        "requestsServed": 14523,
        "avgResponseTime": 87
      }
    }
  ]
}
```

### データベース・ビューによるプールの監視

```sql
-- V$ ビューを介した ORDS 接続アクティビティの確認 (DBA 権限が必要)
SELECT s.username, s.status, s.program, s.machine,
       s.sql_id, COUNT(*) AS connection_count
FROM   v$session s
WHERE  s.username = 'ORDS_PUBLIC_USER'
   OR  s.program LIKE '%ORDS%'
GROUP  BY s.username, s.status, s.program, s.machine, s.sql_id
ORDER  BY connection_count DESC;

-- ORDS 接続からのアクティブな SQL を表示
SELECT s.sid, s.username, s.status,
       sq.sql_text, sq.elapsed_time/1000000 AS elapsed_sec
FROM   v$session s
JOIN   v$sql sq ON sq.sql_id = s.sql_id
WHERE  s.username IN ('ORDS_PUBLIC_USER', 'HR')
AND    s.status = 'ACTIVE'
ORDER  BY elapsed_sec DESC;
```

---

## インストール済みモジュール用の DBA_ORDS_* ビュー

```sql
-- REST 有効化されたすべてのスキーマを表示
SELECT schema, url_mapping_pattern, auto_rest_auth, enabled
FROM   dba_ords_enabled_schemas
ORDER  BY schema;

-- スキーマごとのハンドラー数をカウント
SELECT s.schema,
       COUNT(DISTINCT m.id) AS module_count,
       COUNT(DISTINCT t.id) AS template_count,
       COUNT(h.id)          AS handler_count
FROM   dba_ords_enabled_schemas s
LEFT JOIN dba_ords_modules   m ON m.schema = s.schema
LEFT JOIN dba_ords_templates t ON t.module_id = m.id
LEFT JOIN dba_ords_handlers  h ON h.template_id = t.id
GROUP  BY s.schema
ORDER  BY s.schema;

-- すべての POST ハンドラーを検索 (監査すべき書き込み操作)
SELECT s.schema, m.name AS module, t.uri_template,
       h.method, h.source_type,
       SUBSTR(h.source, 1, 200) AS source_preview
FROM   dba_ords_enabled_schemas s
JOIN   dba_ords_modules         m ON m.schema = s.schema
JOIN   dba_ords_templates       t ON t.module_id = m.id
JOIN   dba_ords_handlers        h ON h.template_id = t.id
WHERE  h.method IN ('POST', 'PUT', 'DELETE')
ORDER  BY s.schema, m.name, t.uri_template;

-- ユーザー・スコープのビュー (現在のスキーマのみ)
SELECT m.name, t.uri_template, h.method, h.source_type
FROM   user_ords_modules   m
JOIN   user_ords_templates t ON t.module_id = m.id
JOIN   user_ords_handlers  h ON h.template_id = t.id
ORDER  BY m.name, t.uri_template, h.method;

-- AutoREST 有効化されたオブジェクトの確認
SELECT object_name, object_type, object_alias,
       auto_rest_auth, enabled
FROM   user_ords_enabled_objects
ORDER  BY object_type, object_name;

-- OAuth クライアントとその権限を表示
SELECT c.name, c.grant_type, c.status,
       cp.privilege_name
FROM   user_ords_clients   c
JOIN   user_ords_client_privileges cp ON cp.client_id = c.client_id
ORDER  BY c.name;
```

---

## パフォーマンス・チューニング

### 接続プール・サイズ

最も影響の大きいチューニング・パラメータ。以下に基づいて調整する。
- DB の最大セッション数: `SELECT value FROM v$parameter WHERE name = 'sessions'`
- 期待される ORDS の同時リクエスト数
- DB の CPU コア数 (プール・サイズは、DB の CPU 数 * 10 を超えないようにする)

```shell
# 現在のプール制限の確認
ords --config /opt/oracle/ords/config config list | grep jdbc

# 本番環境の推奨開始値
ords --config /opt/oracle/ords/config config set jdbc.InitialLimit 10
ords --config /opt/oracle/ords/config config set jdbc.MinLimit 10
ords --config /opt/oracle/ords/config config set jdbc.MaxLimit 50
```

### ステートメント・キャッシュ

接続ごとの解析済みカーソルをキャッシュする。高い値を設定すると、繰り返されるクエリのソフト解析オーバーヘッドが削減される。

```shell
ords --config /opt/oracle/ords/config config set jdbc.MaxStatementsLimit 10
# 繰り返されるクエリが多い API の場合は 20-50 に増やす
```

### ORDS JVM メモリのチューニング

```shell
# /etc/systemd/system/ords.service
[Service]
Environment="JAVA_OPTS=-Xms512m -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

- `-Xms512m`: 初期ヒープ (初期の GC 一時停止を避けるために設定)
- `-Xmx2g`: 最大ヒープ (高負荷な ORDS インスタンスでは 2-4GB が一般的)
- `-XX:+UseG1GC`: G1 ガベージ・コレクタ (低レイテンシ、サーバー・アプリに適している)

### リクエスト・スレッド・プール

```shell
# 高同時実行環境向けに HTTP リクエスト・スレッドを増やす (Jetty スタンドアロンの場合)
ords --config /opt/oracle/ords/config config set standalone.threadPoolMax 200
```

### ページネーションのデフォルト値

ページあたりのデフォルト行数を減らすことで、平均レスポンス時間と DB 負荷を軽減する。

```shell
# デフォルトは 25 行。モバイルや API クライアント向けには減らすことを検討
ords --config /opt/oracle/ords/config config set misc.pagination.defaultLimit 10
ords --config /opt/oracle/ords/config config set misc.pagination.maxRows 500
```

---

## ヘルス・チェック・エンドポイント

ロード・バランサーのヘルス・チェックには、ORDS 組み込みのエンドポイントを使用する。

```http
GET /ords/ HTTP/1.1
Host: myserver.example.com
```

ORDS が動作していれば `200 OK` を返す。以下のヘルス・チェック・パスとして使用する。

- AWS ALB: ターゲット・グループのヘルス・チェック・パス `/ords/`
- OCI Load Balancer: バックエンド・セットのヘルス・チェック・プログラム URL `/ords/`
- Nginx アップストリーム・ヘルス・チェック: `check interval=3000 rise=2 fall=3 timeout=2000 type=http; check_http_send "GET /ords/ HTTP/1.0\r\n\r\n"; check_http_expect_alive http_2xx;`
- Kubernetes liveness probe:

```yaml
livenessProbe:
  httpGet:
    path: /ords/
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 30
  failureThreshold: 3
```

### DB レベルのヘルス・チェック

DB 接続性を検証する、より深いヘルス・チェックを行う場合:

```http
GET /ords/_/db-api/stable/database/ HTTP/1.1
Authorization: Bearer <health-check-token>
```

接続プールの状態を返す。200 レスポンスが返れば、ORDS と DB 両方の接続性が確認されたことになる。

---

## 一般的なエラー・コードと診断

### HTTP 401 Unauthorized (未認証)

原因: Bearer トークンの欠落、無効、または期限切れ。

```
WWW-Authenticate: Bearer realm="hr",error="invalid_token",
  error_description="The access token expired"
```

診断:
- トークンの有効期限を確認 (デフォルトは 1 時間)
- トークン・エンドポイントの URL がスキーマと一致しているか確認: `/ords/{schema}/oauth/token`
- クライアントの権限割り当てを確認: `SELECT * FROM user_ords_client_privileges`

### HTTP 403 Forbidden (禁止)

原因: トークンは有効だが、必要な権限またはロールが不足している。

診断:
```sql
-- クライアントが保持している権限の確認
SELECT cp.privilege_name, cr.role_name
FROM   user_ords_client_privileges cp
JOIN   user_ords_clients c ON c.client_id = cp.client_id
LEFT JOIN user_ords_client_roles cr ON cr.client_id = c.client_id
WHERE  c.name = 'my-client';

-- 権限が必要とするロールの確認
SELECT privilege_name, role_name
FROM   user_ords_privilege_roles
WHERE  privilege_name = 'hr.employees.read';
```

### HTTP 404 Not Found (未検出)

原因:
1. スキーマが REST 有効化されていない
2. モジュールが公開されていない (`status = NOT_PUBLISHED`)
3. URL の入力ミス (スキーマのエイリアス、モジュールのパス、テンプレートのパターンを確認)
4. そのテンプレートに対して間違った HTTP メソッドを使用している

診断:
```sql
-- スキーマ・エイリアスの検証
SELECT schema, url_mapping_pattern, enabled
FROM   dba_ords_enabled_schemas WHERE schema = 'HR';

-- モジュールの検証
SELECT name, uri_prefix, status FROM user_ords_modules;

-- テンプレートの検証
SELECT uri_template FROM user_ords_templates t
JOIN user_ords_modules m ON m.id = t.module_id
WHERE m.name = 'hr.employees';
```

### HTTP 500 Internal Server Error (サーバー内部エラー)

原因: ハンドラーの SQL/PL/SQL 内での未処理の例外、または DB エラー。

診断: ORDS ログで詳細な ORA- エラーを確認する。

```
2024-01-15 16:30:00.000 SEVERE Handler error
  path=/ords/hr/v1/employees/ method=POST pool=default
  ORA-00001: unique constraint (HR.UK_EMP_EMAIL) violated
```

ORDS コンテキストにおける一般的な ORA- エラー:

| ORA エラー | 原因 | 対応策 |
|---|---|---|
| ORA-00942 | 表が見つからない (ORDS_PUBLIC_USER に権限がない) | ORDS_PUBLIC_USER または実行スキーマに SELECT 権限を付与 |
| ORA-01403 | `collection_item` ハンドラーでデータが見つからない | 正常な動作。自動的に HTTP 404 を返す |
| ORA-01400 | NOT NULL 制約違反 | ハンドラー内で必須フィールドの検証を行う |
| ORA-00001 | 一意制約違反 | 例外ハンドラー内で HTTP 409 Conflict を返す |
| ORA-03114 | Oracle に接続されていない | DB 停止または接続切断。ORDS が再接続を試みる |
| ORA-12514 | TNS リスナーがサービスを見つけられない | プール構成のサービス名が間違っている |
| ORA-28000 | アカウント・ロック (ORDS_PUBLIC_USER) | DB でアカウントをアンロック: `ALTER USER ORDS_PUBLIC_USER ACCOUNT UNLOCK` |

### HTTP 503 Service Unavailable (サービス利用不可)

原因: 接続プールの枯渇。すべての接続が使用中。

```
2024-01-15 17:00:00.000 SEVERE Pool exhausted
  pool=default busy=30 max=30 wait=60001ms
```

対応策:
1. `jdbc.MaxLimit` を増やす (DB が追加の接続を処理可能な場合)
2. 接続を保持し続けている遅いクエリを調査する
3. ロード・バランサーの背後に ORDS インスタンスを追加する
4. クエリ・タイムアウトを実装する: `ords --config ... config set jdbc.statementTimeout 30`

---

## ORDS 検証ツール

開始前に構成を確認するために、ORDS 自己診断を実行する。

```shell
ords --config /opt/oracle/ords/config validate
```

出力例:

```
INFO  ORDS validation starting
INFO  Validating pool 'default' connectivity...
INFO  Pool 'default': connected successfully (Oracle Database 19c)
INFO  ORDS schema version: 24.1.0 (matches binary version)
INFO  Validation complete: 0 errors, 1 warnings
```

一般的な検証失敗の例:
- `Pool 'default': connection failed` — ホスト名、ポート、または資格証明が正しくない
- `ORDS schema version 21.4 does not match binary 24.1` — `ords install` を実行してスキーマをアップグレードする
- `WARNING: feature.sdw enabled but APEX not detected` — Database Actions には APEX が必要

---

## ベスト・プラクティス

- **ログ・ローテーションのセットアップ**: ORDS ログ・ファイルは継続的に肥大化する。logrotate を構成するか、組み込みのローテーション設定 (`log.maxFileSize`, `log.maxBackupIndex`) を使用すること。
- **本番環境でのログ集約**: ORDS ログを集中管理システム (ELK Stack, OCI Logging, Splunk など) に転送する。複数の ORDS インスタンスにわたるエラー・パターンをクエリできるようにする。
- **プールの枯渇をアラート対象にする**: プールの枯渇 (`503 エラー`) はキャパシティの問題を示している。接続待ち時間が 1 秒を超えた場合にアラートを出すようにする。
- **ORDS と並行して DB 接続数を監視する**: `v$session` のカウント数を確認し、ORDS のプール使用率と DB セッション使用率を相関させる。
- **アプリケーション・エンドポイントではなく `/ords/` ヘルス・エンドポイントを使用する**: ヘルス・チェックは、実際に DB クエリを実行する REST API ではなく、軽量な ORDS ルート・エンドポイントを使用すべきである。
- **スロー・リクエストの傾向を経時的に追跡する**: スロー・リクエストの段階的な増加は、完全な障害の前兆であることが多い。P95/P99 レスポンス時間を追跡するダッシュボードを設定すること。
- **構成変更のたびに `ords validate` を実行する**: トラフィックが流入する前に、プールの接続性の問題を特定する。

## よくある間違い

- **404 エラーのデバッグ時に ORDS ログを確認していない**: 開発者は URL や DB 内のスキーマ構成を確認するだけで、ORDS ログにある決定的なエラー・メッセージを見落としがちである。
- **プールの枯渇警告を無視する**: `WARNING Pool: waiting for connection` は、しばしば「たまに起こる正常なこと」として扱われる。これは、即座の対応が必要なキャパシティ不足の先行指標である。
- **`jdbc.MaxLimit` を DB の `sessions` パラメータより高く設定している**: ORDS は DB の上限まで接続を開こうとするが、途中で失敗する。`jdbc.MaxLimit` は DB の `sessions` の 20-30% 程度低い値に設定すべきである。
- **本番環境を FINE ログ・レベルで実行している**: DEBUG/FINE レベルのロギングは、高負荷時に 1 時間あたり数百 MB のログを生成し、ディスクを圧迫し、パフォーマンスに影響を与える。
- **ORA-28000 (アカウント・ロック) を監視していない**: ORDS_PUBLIC_USER がログイン試行失敗回数を超えた場合（パスワード変更の不備など）、アカウントがロックされ、すべての ORDS リクエストが 503 で失敗する。
- **ORDS のヘルス・チェックが REST API ハンドラーを検証していると誤解している**: `/ords/` ヘルス・チェックは、ORDS が動作しており HTTP 通信が可能であることを確認するだけである。DB 接続性や REST モジュールの動作までは検証しない。完全な検証には `/ords/_/db-api/stable/database/` を使用すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。混在環境では 19c 互換の代替手段を維持すること。
- 19c と 26ai の両方をサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンで構文とパッケージの動作をテストすること。

## ソース

- [ORDS Developer's Guide — Monitoring and Diagnosing Oracle REST Data Services](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/orddg/about-oracle-rest-data-services.html)
- [ORDS Configuration Settings Reference — Logging and Performance](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/configuration-settings.html)
- [ORDS CLI Reference — ords validate](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/24.2/ordig/ords-command-line-interface.html)
