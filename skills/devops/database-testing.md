# utPLSQL によるデータベース・テスト

## 概要

PL/SQL コードのテストは、歴史的に「二の次」として扱われてきた。手動でのテストや、その場限りのスクリプト、あるいはテストを全く行わないといった状況も珍しくなかった。utPLSQL フレームワークは、構造化されたテスト・パッケージ、アサーション、セットアップ/ティーアダウンのライフサイクル、モック化、コード・カバレッジの測定、そして CI/CD パイプラインとの統合といった、ユニット・テストの規律を Oracle データベース開発にもたらす。

utPLSQL (バージョン 3.x) は、Steven Feuerstein 氏のオリジナル作品や v1 の現代的な後継である。これは、導入されたパッケージ・セットとして Oracle データベース内部で完全に動作し、外部のランライムを必要としない。テスト自体も PL/SQL で記述されるため、テスト対象の環境に対してネイティブである。

このガイドでは、テストの記述、テスト・データの管理、依存関係のモック化、パイプラインとの統合、およびカバレッジの測定といったライフサイクル全体をカバーする。

---

## utPLSQL のインストール

```shell
# リポジトリのクローン
git clone https://github.com/utPLSQL/utPLSQL.git
cd utPLSQL

# データベースへのインストール
# インストーラーが UT3 スキーマとすべてのフレームワーク・オブジェクトを作成する
sqlplus sys/password@//host:1521/service AS SYSDBA @install/install.sql

# 専用のテスト実行用アカウントを作成
sqlplus sys/password@//host:1521/service AS SYSDBA <<'EOF'
CREATE USER ut_runner IDENTIFIED BY "password";
GRANT CREATE SESSION TO ut_runner;
GRANT ut_runner TO ut_runner;   -- utPLSQL ロールを付与
EOF
```

---

## テスト・パッケージの構造

utPLSQL のテストは、アノテーション（`-- %` で始まるコメント）が付加された PL/SQL パッケージとして構成される。アノテーションがフレームワークの動作を制御する。

| アノテーション | 用途 |
|---|---|
| `%suite` | パッケージをテスト・スイートとしてマークする |
| `%test` | プロシージャをテスト・ケースとしてマークする |
| `%beforeall` | スイート全体の開始前に 1 回実行される |
| `%afterall` | スイート全体の終了後に 1 回実行される |
| `%beforeeach` | 各テストの実行前に毎回実行される |
| `%aftereach` | 各テストの実行後に毎回実行される |
| `%displayname` | 人間が読みやすいスイート/テスト名 |
| `%disabled` | テストをスキップする |
| `%throws` | 特定の例外コードの発生を期待する |

### 基本的なテスト・パッケージの例

```sql
-- パッケージ仕様部
CREATE OR REPLACE PACKAGE ut_pkg_orders AS
  -- %suite(注文処理のテスト)
  -- %suitepath(app.orders)

  -- %beforeall
  PROCEDURE setup_suite;

  -- %afterall
  PROCEDURE teardown_suite;

  -- %beforeeach
  PROCEDURE setup_test;

  -- %aftereach
  PROCEDURE teardown_test;

  -- %test(複数明細がある注文の合計金額算出)
  PROCEDURE test_order_total_multiple_lines;

  -- %test(存在しない注文ID指定時に例外が発生すること)
  -- %throws(-20001)
  PROCEDURE test_order_total_invalid_id;

END ut_pkg_orders;
/

-- パッケージ本体
CREATE OR REPLACE PACKAGE BODY ut_pkg_orders AS

  -- テスト用定数 (負の ID はテストデータとする慣習)
  c_test_customer_id CONSTANT NUMBER := -9001;
  c_test_order_id    CONSTANT NUMBER := -9001;

  -- -----------------------------------------------------------------------
  -- ライフサイクル・フック
  -- -----------------------------------------------------------------------

  PROCEDURE setup_suite IS
  BEGIN
    INSERT INTO CUSTOMERS_T (CUSTOMER_ID, FIRST_NAME, LAST_NAME, EMAIL)
    VALUES (c_test_customer_id, 'Test', 'User', 'test@example.com');
    COMMIT;
  END setup_suite;

  PROCEDURE teardown_suite IS
  BEGIN
    DELETE FROM ORDER_LINES WHERE ORDER_ID < 0;
    DELETE FROM ORDERS       WHERE ORDER_ID < 0;
    DELETE FROM CUSTOMERS_T  WHERE CUSTOMER_ID < 0;
    COMMIT;
  END teardown_suite;

  PROCEDURE setup_test IS
  BEGIN
    INSERT INTO ORDERS (ORDER_ID, CUSTOMER_ID, ORDER_DATE, STATUS_CODE)
    VALUES (c_test_order_id, c_test_customer_id, SYSDATE, 'PENDING');
    COMMIT;
  END setup_test;

  PROCEDURE teardown_test IS
  BEGIN
    DELETE FROM ORDER_LINES WHERE ORDER_ID = c_test_order_id;
    DELETE FROM ORDERS       WHERE ORDER_ID = c_test_order_id;
    COMMIT;
  END teardown_test;

  -- -----------------------------------------------------------------------
  -- テスト・ケース
  -- -----------------------------------------------------------------------

  PROCEDURE test_order_total_multiple_lines IS
    v_actual NUMBER;
  BEGIN
    -- 準備: 明細の挿入
    INSERT INTO ORDER_LINES (LINE_ID, ORDER_ID, PRODUCT_ID, QTY, UNIT_PRICE)
    VALUES (-1, c_test_order_id, 1001, 2, 19.99);

    INSERT INTO ORDER_LINES (LINE_ID, ORDER_ID, PRODUCT_ID, QTY, UNIT_PRICE)
    VALUES (-2, c_test_order_id, 1002, 1, 49.99);
    COMMIT;

    -- 実行
    v_actual := PKG_ORDERS.get_order_total(c_test_order_id);

    -- 検証 (Assertion)
    ut.expect(v_actual).to_equal(89.97);   -- 2 * 19.99 + 1 * 49.99
  END test_order_total_multiple_lines;

  PROCEDURE test_order_total_invalid_id IS
    v_dummy NUMBER;
  BEGIN
    -- ORA-20001 の発生を期待
    v_dummy := PKG_ORDERS.get_order_total(-99999);
  END test_order_total_invalid_id;

END ut_pkg_orders;
/
```

---

## アサーション (Assertions)

utPLSQL は、`ut.expect(実際値).to_*(期待値)` を中心とした流れるようなアサーション API を使用する。

```sql
-- 等価性の検証
ut.expect(v_count).to_equal(5);
ut.expect(v_name).to_equal('ACTIVE');

-- NULL チェック
ut.expect(v_result).to_be_null();
ut.expect(v_result).not_to_be_null();

-- 数値の比較
ut.expect(v_total).to_be_greater_than(0);
ut.expect(v_total).to_be_between(10, 999);

-- ブール値
ut.expect(v_flag).to_be_true();

-- 文字列のパターンマッチ
ut.expect(v_message).to_be_like('%error%');         -- SQL LIKE パターン
ut.expect(v_message).to_match('^ERROR:.*\d{4}$');   -- 正規表現
```

### カーソルの比較

クエリの結果を検証する際に非常に強力な機能である。

```sql
PROCEDURE test_active_customer_view IS
BEGIN
  ut.expect(
    CURSOR(
      SELECT CUSTOMER_ID, EMAIL FROM CUSTOMERS WHERE CUSTOMER_ID = c_test_customer_id
    )
  ).to_equal(
    CURSOR(
      SELECT c_test_customer_id AS CUSTOMER_ID, 'test@example.com' AS EMAIL FROM DUAL
    )
  );
END test_active_customer_view;
```

---

## テスト・データの管理

### テスト・データを分離する戦略

- **負の ID 慣習:** テスト・データ専用に負の ID 範囲（または非常に大きな ID 範囲）を予約する。事後処理でその範囲を一括削除する。本番データとの混ざりを防ぐシンプルかつ効果的な方法である。
- **セーブポイントによるロールバック:** テストごとに `SAVEPOINT` を作成し、最後に `ROLLBACK TO SAVEPOINT` を行う。最も高速だが、`COMMIT` に依存するロジック（バックグラウンド処理など）のテストには向かない。
- **専用テスト・スキーマ:** CI 実行のたびに使い捨てのテスト・ユーザーを作成し、移行（Migration）を適用してテストを実行する。分離レベルは最高だが、オーバーヘッドが大きい。

---

## モック化 (Mocking)

utPLSQL は、パッケージ内の関数やプロシージャの呼び出しをスタブに置き換えるモック・フレームワークを含んでいる。これにより、外部サービスや高負荷なクエリなどの依存関係を切り離してテストできる。

```sql
PROCEDURE test_price_in_eur IS
BEGIN
  -- 外部レート取得サービス (PKG_EXCHANGE_RATES.GET_RATE) が常に 0.92 を返すようにモック化
  ut3.ut_mock.package_function(
    a_owner       => 'APP_OWNER',
    a_package     => 'PKG_EXCHANGE_RATES',
    a_name        => 'GET_RATE',
    a_return_value => 0.92
  );

  -- テスト対象を呼び出し
  ut.expect(
    PKG_PRICING.convert_to_eur(p_amount_usd => 100.00)
  ).to_equal(92.00);

  -- モックが期待通り 1 回呼ばれたか検証
  ut3.ut_mock.expect_called(
    a_owner   => 'APP_OWNER',
    a_package => 'PKG_EXCHANGE_RATES',
    a_name    => 'GET_RATE',
    a_times   => 1
  );
END test_price_in_eur;
```

---

## テストの実行

```sql
-- データベース内のすべてのテストを実行
EXEC ut.run();

-- 特定のスイート（パッケージ）を実行
EXEC ut.run('ut_pkg_orders');

-- 特定のテスト・ケースをパスで指定して実行
EXEC ut.run('ut_pkg_orders.test_order_total_multiple_lines');

-- タグにマッチするテストを実行
EXEC ut.run(a_tags => ut_varchar2_list('critical'));
```

---

## CI/CD パイプラインとの統合

### リポーター (出力形式)

utPLSQL は、CI ツールと連携するための複数のフォーマットをサポートしている。

```sql
-- JUnit XML (Jenkins, GitHub Actions, GitLab CI などで消費可能)
BEGIN
  ut.run(
    a_paths    => ut_varchar2_list(':app'),
    a_reporters => ut_reporters(ut_junit_reporter()),
    a_output_to => ut_output_to_file('/tmp/test-results.xml')
  );
END;
/
```

- **utPLSQL-cli:** Java 製のコマンドライン・クライアント。SQL*Plus を直接使わずに、シェルからテストを実行し、結果（JUnit XML など）をファイルに保存できるため、CI との親和性が高い。

---

## コード・カバレッジ (Code Coverage)

utPLSQL は Oracle の `DBMS_PROFILER` や `DBMS_PLSQL_CODE_COVERAGE` と統合し、テストの実行中に PL/SQL のどの行が実行されたかを測定できる。

```shell
# utPLSQL-cli を使用したカバレッジ測定例
./utplsql run app_owner/password@//host:1521/service \
  -f=ut_documentation_reporter  -o=/dev/stdout \
  -f=ut_coverage_html_reporter  -o=coverage.html \
  -source_path=src/plsql \
  -test_path=tests \
  -coverage_schemes=APP_OWNER
```

---

## ベスト・プラクティス

- **実装ではなく「振る舞い」をテストする。** 内部の変数や状態ではなく、戻り値や表への挿入結果、発生した例外といった外部から観測可能な結果を検証する。そうすることで、コードのリファクタリング時にテストを書き直す必要がなくなる。
- **テストは高速に保つ。** 各テストはミリ秒単位で終わるべきである。テストが遅くなると開発者が実行をためらい、品質低下につながる。
- **例外パスを明示的にテストする。** `%throws` アノテーションを使用することで、エラー処理のロジックが正しく機能しているかを簡単に検証できる。PL/SQL の例外処理はバグの温床になりがちである。
- **プルリクエストごとにテストを実行する。** JUnit リポーターを使用して CI に組み込み、テストが失敗している場合はマージできないようにガード（ガバナンス）をかける。
- **カバレッジを測定し、目標値を設定する。** ビジネス上重要なパッケージについては、80% 以上のカバレッジを目指す。ただし、カバレッジの数値自体を追求するのではなく、テストの「質」が重要であることを忘れない。

---

## よくある間違い

**間違い: 実行順序に依存するテスト。**
各テストは独立して実行可能であるべきである。前のテストが残したデータに依存するようなテストは、壊れやすく、デバッグも困難になる。`%beforeeach` で毎回クリーンな状態を作るべきである。

**間違い: クリーンアップ忘れによるテスト・データの蓄積。**
事後処理 (`aftereach`) が失敗すると、テスト・データが溜まり続ける。負の ID 慣習を使用し、依存関係の順序（明細を消してからヘッダーを消す）を守って確実に削除すること。

**間違い: 本番データベースでのテスト実行。**
utPLSQL は実際にデータの挿入、更新、削除を行う。いくらクリーンアップを徹底するつもりでも、本番環境でテストを実行してはいけない。専用の開発・テスト環境や CI コンテナを使用すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [utPLSQL 公式ドキュメント](https://utplsql.org/utPLSQL/latest/)
- [utPLSQL GitHub リポジトリ](https://github.com/utPLSQL/utPLSQL)
- [utPLSQL-cli](https://github.com/utPLSQL/utPLSQL-cli)
- [DBMS_PLSQL_CODE_COVERAGE (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_PLSQL_CODE_COVERAGE.html)
