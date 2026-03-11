# utPLSQL による PL/SQL ユニット・テスト (Unit Testing)

## 概要

utPLSQL は、Oracle PL/SQL におけるデファクト・スタンダードのユニット・テスト・フレームワークです。Java の JUnit のような xUnit の規約に従っており、CI/CD パイプラインとの統合、JUnit XML 形式の出力、およびコード・カバレッジ・レポートの作成が可能です。このガイドでは、インストール、テストの構造、フル・アサーション API、および統合パターンについて説明します。

---

## インストールと構成

### インストール

```bash
# GitHub から最新のリリースをダウンロード
# https://github.com/utPLSQL/utPLSQL/releases

# SYS または DBA 権限を持つユーザーで接続
sqlplus sys/password@database as sysdba

# インストール・スクリプトを実行 (デフォルトでは UT3 スキーマにインストール)
@install.sql UT3 UT3_TABLESPACE

# 開発者およびアプリケーション・スキーマにアクセス権を付与
GRANT EXECUTE ON ut3.ut TO my_test_user;
GRANT SELECT ON ut3.ut_annotation_manager TO my_test_user;

-- または便利なパッケージ・エイリアスを使用
-- デフォルトでパブリック・シノニムが作成されます
```

### インストール後の確認

```sql
-- インストールの確認
SELECT object_name, object_type, status
FROM   all_objects
WHERE  owner = 'UT3'
  AND  object_type IN ('PACKAGE', 'TYPE')
  AND  status = 'INVALID';

-- 行が返されないことを確認
-- セルフテストの実行:
BEGIN
  ut.run();
END;
/
```

---

## テスト・パッケージの構造

utPLSQL のテスト・スイートは、特別なアノテーション・コメント（`-- %suite`, `-- %test` など）を含む PL/SQL パッケージです。

```sql
-- テスト・パッケージ仕様部
CREATE OR REPLACE PACKAGE test_order_mgmt_pkg AS
  -- %suite(注文管理テスト)
  -- %suitepath(myapp.orders)

  -- %beforeall
  PROCEDURE setup_test_data;

  -- %afterall
  PROCEDURE cleanup_test_data;

  -- %beforeeach
  PROCEDURE reset_state;

  -- %aftereach
  PROCEDURE verify_no_side_effects;

  -- %test(有効な顧客による注文作成)
  PROCEDURE test_create_order_valid;

  -- %test(顧客が NULL の場合の注文作成で例外が発生すること)
  -- %throws(-20001)
  PROCEDURE test_create_order_null_customer;

  -- %test(出荷済み注文のキャンセルで例外が発生すること)
  -- %throws(-20010)
  PROCEDURE test_cancel_shipped_order;

  -- %test(既知の注文に対する注文ステータスの取得)
  PROCEDURE test_get_order_status;

  -- %test(未知の注文に対して NULL が返されること)
  PROCEDURE test_get_status_unknown_order;

  -- %disabled
  -- %test(統合テスト - ユニット・テスト実行時はスキップ)
  PROCEDURE test_full_order_workflow;

END test_order_mgmt_pkg;
/
```

```sql
-- テスト・パッケージ本体
CREATE OR REPLACE PACKAGE BODY test_order_mgmt_pkg AS

  -- テスト・データ用のパッケージ変数
  g_test_customer_id  NUMBER;
  g_test_order_id     NUMBER;

  -- %beforeall: スイート内のすべてのテストの前に一度だけ実行
  PROCEDURE setup_test_data IS
  BEGIN
    -- テスト用顧客の挿入 (テスト終了後にロールバック可能)
    INSERT INTO customers (customer_id, customer_name, status)
    VALUES (99999, 'TEST CUSTOMER', 'ACTIVE')
    RETURNING customer_id INTO g_test_customer_id;
  END setup_test_data;

  -- %afterall: すべてのテストの後に一度だけ実行
  PROCEDURE cleanup_test_data IS
  BEGIN
    DELETE FROM customers WHERE customer_id = 99999;
  END cleanup_test_data;

  -- %beforeeach: 各テストの前に実行
  PROCEDURE reset_state IS
  BEGIN
    -- 各テストをクリーンな注文状態で開始
    DELETE FROM orders WHERE customer_id = 99999;
    g_test_order_id := NULL;
  END reset_state;

  -- %aftereach: 各テストの後に実行
  PROCEDURE verify_no_side_effects IS
  BEGIN
    -- テストが予期しないデータをコミットしていないか確認
    ut.expect(
      (SELECT COUNT(*) FROM orders WHERE customer_id = 99999 AND status = 'SHIPPED')
    ).to_equal(0);
  END verify_no_side_effects;

  -- %test(有効な顧客による注文作成)
  PROCEDURE test_create_order_valid IS
    l_new_order_id NUMBER;
  BEGIN
    -- Act (実行)
    order_mgmt_pkg.create_order(
      p_customer_id => g_test_customer_id,
      p_order_id    => l_new_order_id
    );

    -- Assert (検証): 注文が作成されたこと
    ut.expect(l_new_order_id).not_to_be_null();

    -- Assert: 正しいステータスであること
    ut.expect(order_mgmt_pkg.get_order_status(l_new_order_id))
      .to_equal('PENDING');

    -- Assert: 表内に存在すること
    ut.expect(
      (SELECT COUNT(*) FROM orders WHERE order_id = l_new_order_id)
    ).to_equal(1);
  END test_create_order_valid;

  -- %test(顧客が NULL の場合の注文作成で例外が発生すること)
  -- %throws(-20001)
  PROCEDURE test_create_order_null_customer IS
    l_order_id NUMBER;
  BEGIN
    -- NULL を渡した際、ORA-20001 が発生すべき
    order_mgmt_pkg.create_order(
      p_customer_id => NULL,
      p_order_id    => l_order_id
    );
    -- utPLSQL が例外を捕捉し、%throws と一致するか検証する
  END test_create_order_null_customer;

  -- ... 他のテストの実装 ...

END test_order_mgmt_pkg;
/
```

---

## アノテーション・リファレンス

| アノテーション | 記述場所 | 説明 |
|---|---|---|
| `-- %suite` | パッケージ仕様部 | テスト・スイートとしてマーク。括弧内に表示名を指定可能 |
| `-- %suitepath(path)` | パッケージ仕様部 | スイートを整理するための階層パス (例: `myapp.orders`) |
| `-- %test` | プロシージャ仕様部 | テスト・メソッドとしてマーク。括弧内に表示名を指定可能 |
| `-- %beforeall` | プロシージャ仕様部 | スイート内の全テストの前に 1 回実行 |
| `-- %afterall` | プロシージャ仕様部 | スイート内の全テストの後に 1 回実行 |
| `-- %beforeeach` | プロシージャ仕様部 | 各テストの前に実行 |
| `-- %aftereach` | プロシージャ仕様部 | 各テストの後に実行 |
| `-- %throws(error_code)` | テスト・プロシージャ | 期待される例外。このエラーが発生すればテスト成功 |
| `-- %disabled` | プロシージャ仕様部 | テスト実行時にこのテストをスキップ |
| `-- %context(name)` | インライン | スイート内のテストをサブコンテキストにグループ化 |
| `-- %endcontext` | インライン | コンテキスト・グループの終了 |
| `-- %tags(tag1,tag2)` | テスト/スイート | テストを選択的に実行するためのタグ |

---

## フル・アサーション API

### 基本的な期待値 (Expectations)

```sql
-- 等価性
ut.expect(actual_value).to_equal(expected_value);
ut.expect(actual_value).not_to_equal(expected_value);

-- NULL チェック
ut.expect(l_var).to_be_null();
ut.expect(l_var).not_to_be_null();

-- Boolean
ut.expect(l_flag).to_be_true();
ut.expect(l_flag).to_be_false();

-- 数値比較
ut.expect(l_count).to_be_greater_than(0);
ut.expect(l_count).to_be_greater_or_equal(1);
ut.expect(l_count).to_be_less_than(100);
ut.expect(l_count).to_be_less_or_equal(99);
ut.expect(l_count).to_be_between(1, 99);

-- LIKE パターン
ut.expect(l_message).to_be_like('%ERROR%');
ut.expect(l_message).not_to_be_like('%SUCCESS%');

-- 大文字小文字を区別しない一致
ut.expect(l_name).to_be_like_ignoring_case('%smith%');
```

### コレクションとカーソルのアサーション

```sql
-- 結果セットの比較 (最も強力なアサーション)
DECLARE
  l_actual   SYS_REFCURSOR;
  l_expected SYS_REFCURSOR;
BEGIN
  -- 実際の値を取得するカーソル
  OPEN l_actual FOR
    SELECT * FROM orders WHERE customer_id = 99999 ORDER BY order_id;

  -- 期待される値のカーソル
  OPEN l_expected FOR
    SELECT * FROM (
      VALUES ROW(1001, 99999, 'PENDING', SYSDATE),
             ROW(1002, 99999, 'SHIPPED', SYSDATE - 1)
    ) t(order_id, customer_id, status, created_at)
    ORDER BY order_id;

  ut.expect(l_actual).to_equal(l_expected);
END;

-- 一部の列を除外した比較 (タイムスタンプなどを無視する場合に便利)
ut.expect(l_actual).to_equal(l_expected)
  .exclude_columns(ut_varchar2_list('created_at', 'updated_at'));

-- 順序を問わない比較
ut.expect(l_actual).to_equal(l_expected).unordered;

-- コレクションのアサーション
DECLARE
  l_actual   ut_varchar2_list;
  l_expected ut_varchar2_list;
BEGIN
  -- ... l_actual にテスト対象の出力を投入 ...
  l_expected := ut_varchar2_list('Alice', 'Bob', 'Carol');

  ut.expect(l_actual).to_equal(l_expected);
END;
```

### 例外のテスト

```sql
-- 方法 1: %throws アノテーション (単一の例外を期待する場合に最もシンプル)
-- %throws(-20001)
PROCEDURE test_invalid_input IS
BEGIN
  validate_customer(NULL);  -- ORA-20001 が発生するはず
END;

-- 方法 2: ut.expect().to_throw() によるインライン・テスト
PROCEDURE test_multiple_exception_cases IS
BEGIN
  -- 無効なデータで呼び出した際に例外が発生することをテスト
  ut.expect(
    PROCEDURE() IS BEGIN validate_customer(NULL); END;
  ).to_throw(-20001);

  ut.expect(
    PROCEDURE() IS BEGIN validate_customer(-1); END;
  ).to_throw(-20002, 'negative customer id');
END;
```

---

## テスト・データの分離戦略

### 戦略 1: セーブポイントによるロールバック (utPLSQL のデフォルト動作)

utPLSQL は各テストをトランザクションで囲みます。デフォルトでは**ロールバックされません**。テスト内でトランザクションを管理する必要があります。

```sql
-- ベスト・プラクティス: テスト内ではコミットしない
-- ROLLBACK を使用するか、テスト・クリーンアップ・プロシージャに任せる

-- %beforeeach
PROCEDURE reset_state IS
BEGIN
  ROLLBACK;  -- 前のテストの影響を元に戻す
  DELETE FROM orders WHERE customer_id = 99999;
END reset_state;
```

### 戦略 2: テスト・データ用の自律型トランザクション

```sql
-- テストのトランザクション・コンテキストに依存しないテスト・データの挿入
-- %beforeall
PROCEDURE setup_test_data IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO test_customers VALUES (99999, 'TEST CORP', 'ACTIVE');
  COMMIT;
END setup_test_data;

-- %afterall
PROCEDURE cleanup_test_data IS
  PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  DELETE FROM test_customers WHERE customer_id = 99999;
  COMMIT;
END cleanup_test_data;
```

### 戦略 3: 専用テスト・スキーマ

本番環境のコピーであるテスト専用スキーマを使用します。テスト実行の合間に、リフレッシュ・スクリプト等を使用してスキーマをリセットします。

---

## テストの実行

### SQL*Plus または SQLcl から

```sql
-- 現在のユーザーに見えるすべてのテストを実行
BEGIN ut.run(); END;
/

-- 特定のスイートを実行
BEGIN ut.run('test_order_mgmt_pkg'); END;
/

-- パス指定で実行
BEGIN ut.run('myapp.orders'); END;
/

-- 特定のテスト・メソッドを実行
BEGIN ut.run('test_order_mgmt_pkg.test_create_order_valid'); END;
/

-- 特定のリポーターを指定して実行
BEGIN
  ut.run(
    a_paths    => ut_varchar2_list(':myapp'),
    a_reporter => ut_documentation_reporter()
  );
END;
/
```

### JUnit XML 出力 (CI/CD 用)

```bash
# utPLSQL CLI を使用 (CI/CD に推奨)
# https://github.com/utPLSQL/utPLSQL-cli

utplsql run user/password@//host:1521/service \
  -p=':myapp' \
  -f=ut_junit_reporter \
  -o=test-results.xml \
  -c  # カラー出力を有効化
```

### 利用可能なリポーター

| リポーター | 出力形式 | 主な用途 |
|---|---|---|
| `ut_documentation_reporter` | 人間が読みやすいテキスト | ローカル開発 |
| `ut_junit_reporter` | JUnit XML | CI/CD (Jenkins, GitLab CI, GitHub Actions) |
| `ut_sonar_test_reporter` | SonarQube 形式 | SonarQube 連携 |
| `ut_teamcity_reporter` | TeamCity 形式 | JetBrains TeamCity |
| `ut_tap_reporter` | TAP (Test Anything Protocol) | 汎用 CI ツール |
| `ut_coveralls_reporter` | Coveralls JSON | Coveralls コード・カバレッジ・サービス |

---

## コード・カバレッジ・レポート

```sql
-- テスト実行中にカバレッジ収集を有効化
BEGIN
  ut.run(
    a_paths              => ut_varchar2_list(':myapp'),
    a_reporter           => ut_documentation_reporter(),
    a_coverage_schemes   => ut_varchar2_list('MY_APP_SCHEMA'),
    a_coverage_reporter  => ut_coverage_sonarqube_reporter()
  );
END;
/

-- カバレッジ結果の確認
SELECT object_name, object_type,
       covered_lines,
       total_lines,
       ROUND(covered_lines / NULLIF(total_lines, 0) * 100, 1) AS pct_covered
FROM   ut3.ut_coverage_results
ORDER BY pct_covered;
```

---

## モック (Mocking)

utPLSQL は `ut_mock` API を通じてプロシージャ/ファンクションのモックをサポートしています（utPLSQL v3.1+ ）。

```sql
-- 指定した依存関係をスタックする（モック化する）
-- テスト対象コードが external_service_pkg.send_email() を呼び出すとする
-- 実際にメールを送信しないようにモック化する

-- %test(注文完了メールが送信されること)
PROCEDURE test_order_email_sent IS
BEGIN
  -- モック設定: send_email の呼び出しを記録し、常に TRUE を返すようにする
  ut_mock.set_mock(
    a_object_name  => 'external_service_pkg',
    a_method_name  => 'send_email',
    a_return_value => TRUE
  );

  -- Act: 注文処理 (内部で send_email が呼ばれる)
  order_mgmt_pkg.create_order(p_customer_id => 99999, p_order_id => l_id);

  -- Assert: send_email が正確に 1 回呼ばれたことを検証
  ut_mock.verify_call_count(
    a_object_name => 'external_service_pkg',
    a_method_name => 'send_email',
    a_times       => 1
  );
END test_order_email_sent;
```

---

## CI/CD 統合

### GitHub Actions の例

```yaml
name: PL/SQL Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      oracle:
        image: gvenzl/oracle-xe:21-slim
        env:
          ORACLE_PASSWORD: test_password
        ports:
          - 1521:1521

    steps:
      - uses: actions/checkout@v3

      - name: Run utPLSQL tests
        run: |
          utplsql run system/test_password@//localhost:1521/XEPDB1 \
            -p=':myapp' \
            -f=ut_junit_reporter \
            -o=test-results/results.xml

      - name: Publish Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: test-results/results.xml
```

---

## ベスト・プラクティス

- コードを書く前、または同時にテストを書く（TDD または並行開発）。
- 各テスト・プロシージャは、正確に** 1 つの** 振る舞いやシナリオをテストすべきである。
- 各テストで変わる状態には `%beforeeach` を使用し、テスト間で共有できる高コストなセットアップには `%beforeall` を使用する。
- 既存データと衝突する可能性のある ID をハードコードしない。シーケンスや既知の安全な範囲を使用する。
- 正常系（ハッピー・パス）だけでなく、異常系（`%throws`）もテストする。
- カーソル・アサーションはデータを返す機能をテストする最も強力な方法であり、単純な `COUNT(*)` よりも推奨される。
- `.exclude_columns()` を使用して、比較時に `created_at` などの非決定的な列を無視する。
- 最低 80% 以上のカバレッジを目指し、単なる行カバレッジではなく分岐カバレッジを重視する。
- テストを CI/CD パイプラインに統合し、コミットごとに実行する。
- `-- %tags` を使用して、高速なユニット・テストと低速な統合テストを分ける。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| テスト内でコミットする | データベースにテスト・データが永続的に残る | `%aftereach` で `ROLLBACK` するかコミットを避ける |
| 実装の詳細をテストする | リファクタリング時にテストが壊れる | 公開 API の振る舞いのみをテストする |
| 後片付け (Teardown) を行わない | テスト・データが蓄移し、将来のテスト失敗の原因になる | 常に `%afterall` でクリーンアップを行う |
| テスト・データに ID をハードコードする | 他のデータと衝突する | シーケンスを使用するか、既知の安全な ID 範囲を使用する |
| テスト内で `WHEN OTHERS THEN NULL` を使う | エラーが発生してもテストが成功してしまう | テスト・メソッド内で例外を握りつぶさない |
| `COUNT(*)` だけを検証する | 実際のデータ内容が正しいか検証できない | フル・コンテンツ検証にはカーソル・アサーションを使用する |
| 境界値テストを行わない | 正常系しかカバーされない | NULL 入力、空セット、境界値のテストを追加する |

---

## ソース

- [utPLSQL ドキュメント](https://utplsql.org/utPLSQL/latest/) — アノテーション、アサーション API、リポーター、カバレッジ、モック
- [utPLSQL GitHub ワークスペース](https://github.com/utPLSQL/utPLSQL) — インストール、v3.x リリース・ノート
- [utPLSQL-cli GitHub ワークスペース](https://github.com/utPLSQL/utPLSQL-cli) — CLI ランナー、JUnit XML 出力
