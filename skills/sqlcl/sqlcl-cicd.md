# CI/CDパイプラインにおけるSQLcl

## 概要

SQLclは、Oracle Clientのインストールが不要なスタンドアロンのJava実行ファイルであり、非対話型（ヘッドレス）実行をサポートし、ウォレットを介してOracle Cloud Autonomous Databaseに接続でき、CI/CDシステムが処理可能な意味のある終了コードを返すため、CI/CDパイプラインに非常に適しています。組み込みのLiquibaseサポートと組み合わせることで、SQLclは、自動デプロイメント・パイプラインにおけるスキーマ移行、データ・シーディング、DDL抽出、および検証チェックのための単一のツールとして機能します。

このガイドでは以下を扱います：
- 非対話モードでのSQLclの実行
- 終了コードの処理
- 対話型プロンプトなしでのクラウド・データベースへの接続
- GitHub ActionsおよびGitLab CIとの統合
- 環境変数の置換
- ロギングとエラー・キャプチャのパターン

---

## 非対話モードでのSQLclの実行

### 基本的なヘッドレス実行

`-S`（silent）フラグを使用すると、SQLclのバナーとすべての対話型プロンプトが抑制されます。

```shell
sql -S ユーザー名/パスワード@サービス名 @deploy.sql
```

`@deploy.sql` スクリプトが実行され、スクリプトが完了するか `EXIT` コマンドに達するとSQLclは終了します。

### stdinを介したコマンドの実行

```shell
echo "SELECT COUNT(*) FROM employees; EXIT;" | sql -S ユーザー名/パスワード@サービス名
```

または、ヒアドキュメントを使用します（複数行のスクリプトに推奨）：

```shell
sql -S ユーザー名/パスワード@サービス名 <<'EOF'
SET FEEDBACK ON
SELECT COUNT(*) FROM employees;
EXIT
EOF
```

### スクリプト・ファイルの実行

```shell
sql -S ユーザー名/パスワード@サービス名 @/path/to/script.sql
```

### コマンドライン -c フラグ（インライン・コマンド）

> ⚠️ 未検証：インラインSQLコマンドを渡すための `-c` フラグは、公式のSQLcl 25.2スタートアップ・フラグのドキュメント（`-H[ELP]`, `-V[ERSION]`, `-C[OMPATIBILITY]`, `-L[OGON]`, `-NOLOGINTIME`, `-R[ESTRICT]`, `-S[ILENT]`, `-mcp`）には記載されていません。代わりにstdinまたはスクリプト・ファイルを使用してください。

```shell
# 推奨：stdinまたはスクリプト・ファイルを使用
echo "SELECT SYSDATE FROM DUAL; EXIT;" | sql -S ユーザー名/パスワード@サービス名
sql -S ユーザー名/パスワード@サービス名 @script.sql
```

---

## 終了コードの処理

SQLclは、成功時には `0`、失敗時にはゼロ以外のコードで終了します。ただし、SQLスクリプト内の失敗によって確実にゼロ以外の終了コードを返すようにするには、スクリプトの先頭で `WHENEVER SQLERROR` を使用する必要があります。

### 基本的な終了コードのパターン

CI/CDの各SQLスクリプトは、以下のように開始する必要があります。

```sql
-- deploy.sql
WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK
WHENEVER OSERROR  EXIT 9 ROLLBACK

SET FEEDBACK ON
SET ECHO ON

-- ここにSQLステートメントを記述
ALTER TABLE employees ADD (middle_name VARCHAR2(30));

COMMIT;
EXIT 0
```

| ステートメント | 意味 |
|---|---|
| `WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK` | SQLエラーが発生した場合、Oracleエラー・コードを返して終了し、未コミットの変更をロールバックする |
| `WHENEVER OSERROR EXIT 9 ROLLBACK` | OSエラーが発生した場合、終了コード9を返して終了し、ロールバックする |
| `EXIT 0` | 最後に明示的に成功として終了する |
| `EXIT 1` | 明示的な失敗として終了する（検証に失敗した場合などに使用） |

### シェルでの終了コードの確認

```shell
sql -S ユーザー/パスワード@サービス名 @deploy.sql
EXIT_CODE=$?

if [ $EXIT_CODE -ne 0 ]; then
    echo "ERROR: SQL deployment failed with exit code $EXIT_CODE"
    exit $EXIT_CODE
fi
echo "Deployment successful"
```

### 検証スクリプトの終了コード

```sql
-- validate_schema.sql
WHENEVER SQLERROR EXIT SQL.SQLCODE

-- 必要な表が存在することを確認
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM user_tables
    WHERE table_name IN ('EMPLOYEES','DEPARTMENTS','JOBS');

    IF v_count < 3 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Required tables missing. Expected 3, found ' || v_count);
    END IF;
END;
/

-- 無効なオブジェクトがないことを確認
DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count FROM user_objects WHERE status = 'INVALID';
    IF v_count > 0 THEN
        RAISE_APPLICATION_ERROR(-20002, v_count || ' invalid objects found after deployment');
    END IF;
END;
/

EXIT 0
```

---

## ヘッドリス・モードでのOracle Cloudウォレットによる接続

### ウォレットのセットアップ

```shell
# CIランナーからアクセス可能なディレクトリにウォレットを解凍
mkdir -p /tmp/wallet
echo "$WALLET_ZIP_BASE64" | base64 -d > /tmp/wallet.zip
unzip -q /tmp/wallet.zip -d /tmp/wallet
chmod 600 /tmp/wallet/*
```

### TNS_ADMINの設定

```shell
export TNS_ADMIN=/tmp/wallet
```

ウォレット・ディレクトリ内の `sqlnet.ora` の内容：
```
WALLET_LOCATION=(SOURCE=(METHOD=file)(METHOD_DATA=(DIRECTORY=/tmp/wallet)))
SSL_SERVER_DN_MATCH=yes
```

### 接続

```shell
# ウォレットのtnsnames.oraで定義されているTNSエイリアスを使用
sql -S "${DB_USER}/${DB_PASSWORD}@${DB_SERVICE_NAME}" @deploy.sql
```

ここで `DB_SERVICE_NAME` は、ウォレットの `tnsnames.ora` で定義されているエイリアスのいずれか（例：`myatp_high`, `myatp_medium`, `myatp_low`）です。

### クラウド接続の完全な例

```shell
export TNS_ADMIN=/tmp/wallet
sql -S admin/MyPassword123@myatp_high <<'EOF'
WHENEVER SQLERROR EXIT SQL.SQLCODE
SELECT instance_name, status FROM v$instance;
EXIT 0
EOF
```

---

## 環境変数の置換

### SQLスクリプトでのシェル変数の使用

SQLclはSQL*Plusスタイルの置換変数（`&variable_name`）をサポートしています。スクリプトを実行する前に変数を定義することで、環境を通じて値を渡すことができます。

```sql
-- deploy_env.sql
DEFINE ENV     = &1
DEFINE APP_VER = &2

PROMPT Deploying version &APP_VER to environment &ENV
SELECT 'Deploying to: ' || '&ENV' AS info FROM DUAL;
```

コマンドラインから引数を渡します：

```shell
sql -S ユーザー/パスワード@サービス名 @deploy_env.sql PROD v2.5.1
```

### シェル変数の展開の使用

シェルの環境変数の場合は、ヒアドキュメント内でシェルの変数置換を使用します。

```shell
export APP_VERSION="2.5.1"
export DEPLOY_ENV="production"

sql -S "${DB_USER}/${DB_PASSWORD}@${DB_SERVICE}" <<EOF
WHENEVER SQLERROR EXIT SQL.SQLCODE

INSERT INTO deployment_log (version, environment, deployed_at)
VALUES ('${APP_VERSION}', '${DEPLOY_ENV}', SYSDATE);

COMMIT;
EXIT 0
EOF
```

注：ヒアドキュメント内でシェル変数を展開できるようにするには、`<<EOF`（`<<'EOF'` ではなく）を使用してください。

### スクリプト内部構成用のDEFINE変数

```sql
-- parameters.sql (パイプライン・スクリプトの開始時に読み込み)
DEFINE SCHEMA_NAME  = HR
DEFINE APP_VERSION  = 2.5.1
DEFINE ROLLBACK_TAG = v2.4.0

PROMPT Schema: &SCHEMA_NAME
PROMPT Version: &APP_VERSION
```

```sql
-- deploy.sql
@parameters.sql
WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK
lb tag -tag &APP_VERSION
lb update -changelog-file controller.xml
EXIT 0
```

---

## GitHub Actionsとの統合

### 基本的なワークフロー

```yaml
# .github/workflows/deploy-db.yml
name: Deploy Database Changes

on:
  push:
    branches: [main]
    paths:
      - 'db/**'
  pull_request:
    branches: [main]
    paths:
      - 'db/**'

env:
  TNS_ADMIN: /tmp/wallet

jobs:
  validate:
    name: Validate SQL Changes
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: Install SQLcl
        run: |
          curl -sL https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip -o sqlcl.zip
          unzip -q sqlcl.zip -d /opt
          echo "/opt/sqlcl/bin" >> $GITHUB_PATH

      - name: Set up Oracle wallet
        run: |
          mkdir -p /tmp/wallet
          echo "${{ secrets.WALLET_ZIP_B64 }}" | base64 -d | unzip -q -d /tmp/wallet -

      - name: Validate changelog status
        run: |
          cd db
          sql -S "${{ secrets.DB_USER }}/${{ secrets.DB_PASS }}@${{ secrets.DB_SERVICE }}" <<'EOF'
          WHENEVER SQLERROR EXIT SQL.SQLCODE
          lb status -changelog-file controller.xml
          EXIT 0
          EOF

  deploy:
    name: Deploy to Test
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main'
    environment: test
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: Install SQLcl
        run: |
          curl -sL https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip -o sqlcl.zip
          unzip -q sqlcl.zip -d /opt
          echo "/opt/sqlcl/bin" >> $GITHUB_PATH

      - name: Set up Oracle wallet
        run: |
          mkdir -p /tmp/wallet
          echo "${{ secrets.WALLET_ZIP_B64 }}" | base64 -d | unzip -q -d /tmp/wallet -

      - name: Tag pre-deployment state
        run: |
          cd db
          VERSION="${{ github.sha }}"
          sql -S "${{ secrets.DB_USER }}/${{ secrets.DB_PASS }}@${{ secrets.DB_SERVICE }}" <<EOF
          WHENEVER SQLERROR EXIT SQL.SQLCODE
          lb tag -tag pre-${VERSION}
          EXIT 0
          EOF

      - name: Apply Liquibase changes
        run: |
          cd db
          sql -S "${{ secrets.DB_USER }}/${{ secrets.DB_PASS }}@${{ secrets.DB_SERVICE }}" <<'EOF'
          WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK
          SET ECHO ON
          lb update -changelog-file controller.xml
          EXIT 0
          EOF

      - name: Run post-deployment validation
        run: |
          cd db
          sql -S "${{ secrets.DB_USER }}/${{ secrets.DB_PASS }}@${{ secrets.DB_SERVICE }}" @validate_schema.sql

      - name: Upload deployment log on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: deployment-log
          path: /tmp/sqlcl-deploy.log
```

---

## GitLab CIとの統合

```yaml
# .gitlab-ci.yml

variables:
  TNS_ADMIN: "/tmp/wallet"

stages:
  - validate
  - deploy
  - verify

.sqlcl_setup: &sqlcl_setup
  before_script:
    - apt-get update -qq && apt-get install -y -qq unzip curl
    - curl -sL https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip -o sqlcl.zip
    - unzip -q sqlcl.zip -d /opt
    - export PATH="/opt/sqlcl/bin:$PATH"
    - mkdir -p /tmp/wallet
    - echo "$WALLET_ZIP_B64" | base64 -d | unzip -q -d /tmp/wallet -

validate_changes:
  stage: validate
  <<: *sqlcl_setup
  script:
    - cd db
    - |
      sql -S "${DB_USER}/${DB_PASS}@${DB_SERVICE}" <<'SQLEOF'
      WHENEVER SQLERROR EXIT SQL.SQLCODE
      lb status -changelog-file controller.xml
      EXIT 0
      SQLEOF
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy_db:
  stage: deploy
  <<: *sqlcl_setup
  script:
    - cd db
    - |
      sql -S "${DB_USER}/${DB_PASS}@${DB_SERVICE}" <<'SQLEOF'
      WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK
      SET ECHO ON
      lb tag -tag pre-${CI_COMMIT_SHORT_SHA}
      lb update -changelog-file controller.xml
      EXIT 0
      SQLEOF
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  environment:
    name: production
    action: start

verify_deployment:
  stage: verify
  <<: *sqlcl_setup
  script:
    - cd db
    - sql -S "${DB_USER}/${DB_PASS}@${DB_SERVICE}" @validate_schema.sql
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## ロギングとエラーのキャプチャ

### すべてのSQLcl出力のキャプチャ

```shell
# stdoutとstderrの両方をログ・ファイルにリダイレクト
sql -S ユーザー/パスワード@サービス名 @deploy.sql 2>&1 | tee /tmp/deploy.log

# 終了コードの確認（teeがパイプラインを維持し、PIPESTATUSがそれをキャプチャします）
EXIT_CODE=${PIPESTATUS[0]}
if [ $EXIT_CODE -ne 0 ]; then
    echo "Deployment failed. Log:"
    cat /tmp/deploy.log
    exit $EXIT_CODE
fi
```

### 詳細なSQL側ロギングのためのSPOOL

行数やタイミングを含むデータベース側の出力をキャプチャするために、SQLスクリプトに `SPOOL` を追加します。

```sql
-- deploy.sql
SPOOL /tmp/deploy_output.log
WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK
SET ECHO ON
SET FEEDBACK ON
SET TIMING ON
SET SERVEROUTPUT ON SIZE UNLIMITED

-- デプロイメント・ステートメント
lb update -changelog-file controller.xml

SPOOL OFF
EXIT 0
```

### CI解析用の構造化されたログ出力

```sql
-- deploy_structured.sql
WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK
SET ECHO OFF
SET FEEDBACK OFF

PROMPT [INFO] Starting deployment at &_DATE
PROMPT [INFO] Connected as: &_USER to &_CONNECT_IDENTIFIER

lb update -changelog-file controller.xml

DECLARE
    v_count NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_count FROM databasechangelog
    WHERE dateexecuted > SYSDATE - 1/24;
    DBMS_OUTPUT.PUT_LINE('[INFO] Changesets applied in last hour: ' || v_count);
END;
/

PROMPT [INFO] Deployment completed successfully
EXIT 0
```

### 失敗時のロールバック・パターン

```sql
-- deploy_with_rollback.sql
WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK

-- ロールバック・ターゲット用にデプロイ前の状態をタグ付け
lb tag -tag pre-deploy-&1

-- 変更の適用
lb update -changelog-file controller.xml

-- デプロイの検証
DECLARE
    v_invalid NUMBER;
BEGIN
    SELECT COUNT(*) INTO v_invalid FROM user_objects WHERE status = 'INVALID';
    IF v_invalid > 0 THEN
        -- SQLERRORパスをトリガー → ROLLBACKし、エラーでEXIT
        RAISE_APPLICATION_ERROR(-20001,
            'Deployment produced ' || v_invalid || ' invalid objects. Rolling back.');
    END IF;
END;
/

EXIT 0
```

検証が失敗した場合、`WHENEVER SQLERROR EXIT ... ROLLBACK` がDMLロールバックをトリガーします。Liquibaseのスキーマ変更をロールバックするには、シェル・レベルのロールバック・ステップを追加します。

```shell
sql -S ユーザー/パスワード@サービス名 @deploy_with_rollback.sql "$CI_COMMIT_SHORT_SHA"
if [ $? -ne 0 ]; then
    echo "Deployment failed, rolling back Liquibase changes..."
    sql -S ユーザー/パスワード@サービス名 <<EOF
    lb rollback -tag pre-deploy-${CI_COMMIT_SHORT_SHA} -changelog-file controller.xml
    EXIT
EOF
    exit 1
fi
```

---

## CI/CDにおけるセキュリティのベスト・プラクティス

### 資格情報をハードコードしない

資格情報には必ずCI/CDのシークレット変数を使用してください。

```shell
# 推奨：環境変数から資格情報を取得
sql -S "${DB_USER}/${DB_PASS}@${DB_SERVICE}" @deploy.sql

# 非推奨：資格情報のハードコード
sql -S admin/MyPassword123@myatp_high @deploy.sql
```

### ウォレットをBase64シークレットとして保存

ウォレットのZIP全体をBase64エンコードされたCI/CDシークレットとして保存します。

```shell
# シークレットとして保存するためにウォレットをBase64に変換
base64 -i Wallet_MyATP.zip > wallet_b64.txt
# wallet_b64.txt の内容をCI/CDシークレット変数にコピー
```

パイプラインでデコードして使用します。

```shell
mkdir -p /tmp/wallet
echo "$WALLET_ZIP_B64" | base64 -d > /tmp/wallet.zip
unzip -q /tmp/wallet.zip -d /tmp/wallet
chmod 600 /tmp/wallet/*
export TNS_ADMIN=/tmp/wallet
```

### 最小権限のデプロイメント・アカウント

デプロイに必要な権限のみを持つ、専用のデプロイ・スキーマまたはユーザーを使用してください。

```sql
-- 必要な権限のみを持つデプロイ・ユーザーを作成
CREATE USER deploy_user IDENTIFIED BY "SecurePass123!";
GRANT CREATE SESSION TO deploy_user;
GRANT CREATE TABLE, CREATE VIEW, CREATE PROCEDURE, CREATE SEQUENCE TO deploy_user;
GRANT UNLIMITED TABLESPACE TO deploy_user;
-- 厳密に必要でない限り、DBA権限を付与しないでください
```

---

## ベスト・プラクティス

- CI/CD用のすべてのSQLスクリプトの先頭で、必ず `WHENEVER SQLERROR EXIT SQL.SQLCODE ROLLBACK` を使用してください。これがないと、SQLエラーが発生してもSQLclは実行を継続し、ステートメントが失敗しても終了コード0で終了してしまいます。
- すべてのCI/CD実行において、`-S`（silent）フラグを使用してください。これがないと、SQLclのバナーと接続メッセージがパイプラインのログに表示され、ログ・パーサーを混乱させる可能性があります。
- ウォレット・ディレクトリをリポジトリに入れないでください。Base64エンコードされたCI/CDシークレットとして保存し、パイプラインの実行時にデコードしてください。ウォレット・ファイル（`.sso`, `.jks`, `.p12`, `ewallet.p12`）をバージョン管理にコミットしないでください。
- すべてのデプロイの前に `lb tag` を使用して、ロールバック・ターゲットを作成してください。タグ名には必ずコミットSHAまたはパイプラインIDを含め、データベースがどのような状態であったかを正確に特定できるようにしてください。
- データベースの変更が、それに依存するコードとともにバージョン管理されるように、変更ログをアプリケーション・コードと同じリポジトリに保存してください。
- パイプライン・ステージで検証とデプロイを分離してください。検証ステージ（`lb status` のチェック、静的解析の実行）はプルリクエストで実行し、デプロイ・ステージはmain/masterブランチへのマージ時にのみ実行する必要があります。
- 失敗時に `SPOOL` の出力をキャプチャし、CIアーティファクトとしてアップロードしてください。SQLclの生の出力だけでは、何がうまくいかなかったのかを診断するのに不十分な場合があります。

---

## よくある間違いと回避策

**間違い：SQLエラーが黙って無視され、パイプラインが成功してしまう**
`WHENEVER SQLERROR EXIT SQL.SQLCODE` がないと、SQLclはデフォルトでSQLエラーを無視し、終了コード0で終了します。CIスクリプトの最初のステートメントとして、必ず `WHENEVER` ディレクティブを追加してください。

**間違い：接続前に TNS_ADMIN が設定されていない**
`TNS_ADMIN` が設定されていないと、SQLclはウォレットを見つけることができず、接続に失敗します。パイプラインのステップで環境変数として `TNS_ADMIN` を設定するか、CIステップの `before_script` でエクスポートしてください。SQLclを実行する前に `echo $TNS_ADMIN` で確認してください。

**間違い：ウォレット・ファイルの権限が正しくない**
Linux上のOracleでは、ウォレット・ファイルは所有ユーザーのみが読み取り可能（chmod 600）である必要があります。CIランナーがファイルを解凍した際に権限が緩すぎる場合があり、SSLハンドシェイクの失敗を引き起こします。解凍後は必ず `chmod 600 /tmp/wallet/*` を実行してください。

**間違い：変数展開を含むヒアドキュメントを引用符で囲んで展開を無効にしてしまう**
`<<'EOF'`（EOFを引用符で囲む）を使用すると、ヒアドキュメント内でのシェル変数の展開が防止されます。シェル変数を展開する必要がある場合は `<<EOF`（引用符なし）を使用し、`&variable` 置換をそのままSQLclに渡したい場合は `<<'EOF'` を使用してください。

**間違い：パイプラインですべての操作にDBAアカウントを使用している**
日常的なデプロイにDBAまたはADMINアカウントを使用することはセキュリティ上のリスクであり、手動による介入とパイプラインによる変更の監査を困難にします。最小限の権限を持つ専用のデプロイメント・アカウントを使用してください。

**間違い：パイプライン失敗時に Liquibase の変更がロールバックされない**
`WHENEVER SQLERROR EXIT ... ROLLBACK` は、未コミットのDMLトランザクションのみをロールバックします。LiquibaseのDDL変更（CREATE TABLE, ALTER TABLE）はOracleによって自動コミットされるため、トランザクションとしてロールバックすることはできません。デプロイ失敗後のスキーマのロールバックには、必ず `lb rollback -tag` を使用してください。

---

## 参考資料

- [SQLclの起動と終了 — 起動フラグのリファレンス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/startup-sqlcl-settings.html)
- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLcl リリース・ノート 25.2](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.html)
- [Oracle SQLcl リリース・インデックス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/index.html)
