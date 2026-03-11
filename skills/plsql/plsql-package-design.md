# PL/SQLパッケージ設計

## 概要

パッケージは、PL/SQLにおけるモジュール化プログラミングの基本単位です。関連するプロシージャ、ファンクション、型、変数、定数を1つの名前付きオブジェクトにまとめます。適切に設計されたパッケージは、メンテナンス性、パフォーマンス（一度コンパイルされ、セッションごとに一度ロードされる）を向上させ、パブリック/プライベートAPIの分離による情報の隠蔽を可能にします。

---

## パッケージ・アーキテクチャの原則

パッケージは2つの部分で構成されます。

- **パッケージ仕様部 (Spec)**: パブリック・インターフェース。呼び出し側から参照および使用できるもの。
- **パッケージ本体 (Body)**: 実装。プライベート・メンバーと、仕様部の裏側にあるコード。

仕様部は独立してコンパイルされます。本体のみが変更された場合、依存するオブジェクトは有効なままとなり、再コンパイルの連鎖を減らすことができます。

```sql
-- パッケージ仕様部: パブリックAPI
CREATE OR REPLACE PACKAGE order_mgmt_pkg AS

  -- パブリックな型
  TYPE t_order_status IS TABLE OF VARCHAR2(30) INDEX BY PLS_INTEGER;

  -- パブリックな定数
  c_status_pending   CONSTANT VARCHAR2(10) := 'PENDING';
  c_status_shipped   CONSTANT VARCHAR2(10) := 'SHIPPED';
  c_status_cancelled CONSTANT VARCHAR2(10) := 'CANCELLED';

  -- パブリックなプロシージャ/ファンクション
  PROCEDURE create_order(
    p_customer_id IN  orders.customer_id%TYPE,
    p_order_id    OUT orders.order_id%TYPE
  );

  FUNCTION get_order_status(
    p_order_id IN orders.order_id%TYPE
  ) RETURN VARCHAR2;

  PROCEDURE cancel_order(
    p_order_id IN orders.order_id%TYPE,
    p_reason   IN VARCHAR2 DEFAULT NULL
  );

END order_mgmt_pkg;
/

-- パッケージ本体: 実装 + プライベート・メンバー
CREATE OR REPLACE PACKAGE BODY order_mgmt_pkg AS

  -- プライベートな定数 (呼び出し側からは見えない)
  c_max_retries CONSTANT PLS_INTEGER := 3;

  -- プライベートなプロシージャ (仕様部には記述しない)
  PROCEDURE log_order_event(
    p_order_id IN orders.order_id%TYPE,
    p_event    IN VARCHAR2
  ) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    INSERT INTO order_audit_log (order_id, event_time, event_desc)
    VALUES (p_order_id, SYSTIMESTAMP, p_event);
    COMMIT;
  END log_order_event;

  -- パブリックなプロシージャの実装
  PROCEDURE create_order(
    p_customer_id IN  orders.customer_id%TYPE,
    p_order_id    OUT orders.order_id%TYPE
  ) IS
  BEGIN
    INSERT INTO orders (customer_id, status, created_at)
    VALUES (p_customer_id, c_status_pending, SYSDATE)
    RETURNING order_id INTO p_order_id;

    log_order_event(p_order_id, 'ORDER_CREATED');
    COMMIT;
  END create_order;

  FUNCTION get_order_status(
    p_order_id IN orders.order_id%TYPE
  ) RETURN VARCHAR2 IS
    l_status orders.status%TYPE;
  BEGIN
    SELECT status INTO l_status
    FROM   orders
    WHERE  order_id = p_order_id;
    RETURN l_status;
  EXCEPTION
    WHEN NO_DATA_FOUND THEN
      RETURN NULL;
  END get_order_status;

  PROCEDURE cancel_order(
    p_order_id IN orders.order_id%TYPE,
    p_reason   IN VARCHAR2 DEFAULT NULL
  ) IS
  BEGIN
    UPDATE orders
    SET    status     = c_status_cancelled,
           cancelled_at = SYSDATE,
           cancel_reason = p_reason
    WHERE  order_id = p_order_id;

    IF SQL%ROWCOUNT = 0 THEN
      RAISE_APPLICATION_ERROR(-20001, 'Order not found: ' || p_order_id);
    END IF;

    log_order_event(p_order_id, 'ORDER_CANCELLED');
    COMMIT;
  END cancel_order;

END order_mgmt_pkg;
/
```

---

## 仕様部と本体の分離戦略

| 仕様部 (SPEC) に配置するもの | 本体 (BODY) に配置するもの |
|---|---|
| 呼び出し側が使用する型 | プライベートな型 |
| パブリックなプロシージャ/ファンクションの定義 | すべてのプロシージャ/ファンクションの本体 |
| パブリックな定数 | プライベートな定数 |
| パブリックな変数 (可能な限り避ける) | プライベートな変数 |
| 呼び出し側がキャッチすべき例外 | プライベートな例外 |
| 呼び出し側が反復処理するカーソルの宣言 | すべてのカーソルの実装 |

**経験則**: 呼び出し側が参照する必要がないものは、本体に記述します。仕様部を最小限に抑えることで、依存関係を減らし、再コンパイルの頻度を下げることができます。

---

## パブリックAPIとプライベートAPIの設計

### パブリックAPIの設計原則

1. **安定したシグネチャ**: 仕様部のパラメータを変更すると、すべての依存オブジェクトが破損します。既存の呼び出しを壊さずにパラメータを追加するには、デフォルト値を使用します。
2. **意味のある名前**: `do_stuff` よりも `process_order` の方が適切です。
3. **単一責任**: 各プロシージャは1つのことだけを行います。
4. **戻り値 vs OUTパラメータ**: 単一の値を返すファンクションの方が組み合わせが容易です。複数の出力やDML操作を行う場合は、OUTパラメータを持つプロシージャを使用します。

```sql
-- 良い例: デフォルト値により、既存のコードを壊さずに追加機能を提供可能
PROCEDURE process_payment(
  p_order_id      IN orders.order_id%TYPE,
  p_amount        IN NUMBER,
  p_currency      IN VARCHAR2 DEFAULT 'USD',  -- 後から追加、既存呼び出しに影響なし
  p_send_receipt  IN BOOLEAN  DEFAULT TRUE    -- 後から追加、既存呼び出しに影響なし
);
```

### プライベート実装ヘルパー

実装の詳細はプライベートに保ちます。他のパッケージが本当に必要とする場合にのみ、仕様部に昇格させます。

```sql
-- プライベート・ヘルパー: 呼び出し側が直接呼び出すことのないバリデーション・ロジック
PROCEDURE validate_order_amount(
  p_amount   IN NUMBER,
  p_currency IN VARCHAR2
) IS
BEGIN
  IF p_amount <= 0 THEN
    RAISE_APPLICATION_ERROR(-20010, 'Amount must be positive');
  END IF;
  IF p_currency NOT IN ('USD', 'EUR', 'GBP') THEN
    RAISE_APPLICATION_ERROR(-20011, 'Unsupported currency: ' || p_currency);
  END IF;
END validate_order_amount;
```

---

## パッケージ初期化ブロック

パッケージ本体には、セッションごとに一度だけ、そのパッケージが最初に参照されたときに実行される初期化ブロック（オプション）を持たせることができます。

```sql
CREATE OR REPLACE PACKAGE BODY config_pkg AS

  g_env_name     VARCHAR2(50);
  g_debug_enabled BOOLEAN;

  -- 初期化ブロック: パッケージへの初回参照時にセッションごとに一度実行
  BEGIN
    -- 設定テーブルから設定をロード
    BEGIN
      SELECT setting_value
      INTO   g_env_name
      FROM   app_settings
      WHERE  setting_name = 'ENVIRONMENT';
    EXCEPTION
      WHEN NO_DATA_FOUND THEN
        g_env_name := 'UNKNOWN';
    END;

    g_debug_enabled := (g_env_name IN ('DEV', 'TEST'));
  END config_pkg;
/
```

### 接続プール使用時のパッケージ状態の落とし穴

**これは本番環境における重要な懸念事項です。** パッケージ・レベルの変数（グローバル状態）はセッション・スコープです。接続プール（JDBC、OCI、DRCPなど）を使用する場合、セッションは異なる論理的なユーザーやリクエスト間で再利用される可能性があります。前のユーザーのリクエストによるパッケージ状態が残ってしまう場合があります。

```sql
-- 危険: パッケージ変数がユーザー固有の状態を保持している
CREATE OR REPLACE PACKAGE session_context_pkg AS
  g_current_user_id NUMBER;  -- プールの再利用を跨いで永続する！
  PROCEDURE set_user(p_user_id IN NUMBER);
  FUNCTION  get_user RETURN NUMBER;
END session_context_pkg;
/

-- 安全な代替案: アプリケーション・コンテキスト (SYS_CONTEXT) を使用する
-- ログオン・トリガーやアプリケーション初期化コールで設定
BEGIN
  DBMS_SESSION.SET_CONTEXT(
    namespace => 'APP_CTX',
    attribute => 'USER_ID',
    value     => TO_CHAR(p_user_id)
  );
END;

-- どこからでも読み取れ、セッション固有であり、プールの再利用の影響を受けない
SELECT SYS_CONTEXT('APP_CTX', 'USER_ID') FROM DUAL;
```

### パッケージ状態の安全な使用法

パッケージ状態は以下のような場合に適しています。

- 表から一度だけロードされる**読み取り専用の設定**（環境フラグ、検索マップなど）
- 鮮度が許容され、セッションが1人のユーザーを表す場合の**セッション・スコープのキャッシュ**

```sql
CREATE OR REPLACE PACKAGE BODY lookup_cache_pkg AS

  TYPE t_code_map IS TABLE OF VARCHAR2(200) INDEX BY VARCHAR2(30);
  g_status_map t_code_map;
  g_map_loaded BOOLEAN := FALSE;

  PROCEDURE ensure_loaded IS
  BEGIN
    IF NOT g_map_loaded THEN
      FOR rec IN (SELECT code, description FROM status_codes) LOOP
        g_status_map(rec.code) := rec.description;
      END LOOP;
      g_map_loaded := TRUE;
    END IF;
  END ensure_loaded;

  FUNCTION get_status_desc(p_code IN VARCHAR2) RETURN VARCHAR2 IS
  BEGIN
    ensure_loaded;
    IF g_status_map.EXISTS(p_code) THEN
      RETURN g_status_map(p_code);
    END IF;
    RETURN 'UNKNOWN';
  END get_status_desc;

END lookup_cache_pkg;
/
```

---

## 凝集度と結合度

- **高い凝集度**: 同じデータを操作する、または同じ機能ドメインに属するプロシージャをグループ化します。`customer_pkg` は顧客操作を処理し、注文操作は処理しません。
- **低い結合度**: パッケージ間で循環参照が発生しないようにします。`pkg_a` が `pkg_b` を呼び出し、その逆も発生する場合は、共通のロジックを第3のパッケージに抽出します。

### 循環依存の検出

```sql
-- USER_DEPENDENCIESを使用して循環依存を確認
SELECT referenced_name, name
FROM   user_dependencies
WHERE  type = 'PACKAGE BODY'
  AND  referenced_type IN ('PACKAGE', 'PACKAGE BODY')
ORDER BY referenced_name;
```

---

## 前方宣言

パッケージ本体の中では、自身の後に定義されたプロシージャを呼び出すことはできません。呼び出すには前方宣言が必要です。

```sql
CREATE OR REPLACE PACKAGE BODY mutual_pkg AS

  -- 前方宣言により、process_aからprocess_bを呼び出せるようにする
  -- (process_bは後で定義されているため)
  PROCEDURE process_b(p_id IN NUMBER);

  PROCEDURE process_a(p_id IN NUMBER) IS
  BEGIN
    IF p_id > 0 THEN
      process_b(p_id - 1);  -- 前方宣言があるため有効
    END IF;
  END process_a;

  PROCEDURE process_b(p_id IN NUMBER) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('Processing: ' || p_id);
    IF p_id > 0 THEN
      process_a(p_id - 1);
    END IF;
  END process_b;

END mutual_pkg;
/
```

---

## オーバーロード

同じプロシージャ/ファンクション名を、異なるパラメータ・シグネチャで仕様部に複数記述できます。Oracleはコンパイル時に呼び出しを解決します。

```sql
CREATE OR REPLACE PACKAGE format_pkg AS

  -- オーバーロード: 名前は同じだがパラメータ・タイプが異なる
  FUNCTION format_date(p_date IN DATE)      RETURN VARCHAR2;
  FUNCTION format_date(p_date IN TIMESTAMP) RETURN VARCHAR2;
  FUNCTION format_date(p_date IN DATE, p_fmt IN VARCHAR2) RETURN VARCHAR2;

END format_pkg;
/

CREATE OR REPLACE PACKAGE BODY format_pkg AS

  FUNCTION format_date(p_date IN DATE) RETURN VARCHAR2 IS
  BEGIN
    RETURN TO_CHAR(p_date, 'YYYY-MM-DD');
  END format_date;

  FUNCTION format_date(p_date IN TIMESTAMP) RETURN VARCHAR2 IS
  BEGIN
    RETURN TO_CHAR(p_date, 'YYYY-MM-DD HH24:MI:SS.FF3');
  END format_date;

  FUNCTION format_date(p_date IN DATE, p_fmt IN VARCHAR2) RETURN VARCHAR2 IS
  BEGIN
    RETURN TO_CHAR(p_date, p_fmt);
  END format_date;

END format_pkg;
/
```

**オーバーロードの制限**: 戻り値の型のみに基づくオーバーロードはできません。INモードかOUTモードかだけの違いによるオーバーロードもできません。パラメータ・タイプが PLS_INTEGER と NUMBER のように同じファミリーのサブタイプであるだけの違いも不可です。

---

## パッケージ・サイズのガイドライン

巨大なパッケージはメンテナンスが難しく、コンパイル時間も長くなります。以下の基準で分割を検討してください。

| 兆候 | アクション |
|---|---|
| 本体が約1000〜1500行を超える | 機能領域ごとにサブパッケージに分割する |
| 仕様部に30以上のパブリック・メンバーがある | すべてが本当にパブリックであるべきか見直す |
| 複数のドメインが混在している | ドメインごとに分割する (顧客 vs 注文 vs 支払い) |
| 初期化ブロックが多くのスキーマの表にアクセスしている | 専用の設定パッケージに抽出する |

---

## ベスト・プラクティス

- 常に仕様部を先に作成し、その後に本体を作成します。これにより、本体が存在する前に、依存オブジェクトが仕様部に対してコンパイルできるようになります。
- 大規模な IN OUT コレクション・パラメータには `NOCOPY` を使用します（パフォーマンス・ガイドを参照）。
- パッケージ初期化ブロックにDMLを記述しないでください。暗黙的に実行され、予期しないコミットやロックの原因となる可能性があります。
- パッケージ・レベル（グローバル）の変数には `g_` プレフィックスを付け、ローカル変数と区別します。
- スキーマ変更に対応できるよう、変数の宣言は `%TYPE` を使用して列の型に固定します。
- APIドキュメントとしての役割を果たすよう、仕様部にコメントを記述します。

---

## よくある間違いとアンチパターン

| アンチパターン | 問題点 | 解決策 |
|---|---|---|
| パブリックなパッケージ変数 | 呼び出し側が内部状態に直接依存し、変更が困難になる | ゲッター/セッター・ファンクションを使用する |
| 接続プール使用時にユーザー識別情報をパッケージ・グローバルに保存する | リクエスト間で状態がリークする | アプリケーション・コンテキスト (`DBMS_SESSION.SET_CONTEXT`) を使用する |
| パッケージ間の循環依存 | コンパイル不可。メンテナンスの悪夢 | 共通の型/ユーティリティを別のベース・パッケージに抽出する |
| 1つの巨大な "utils" パッケージ | 凝集度がゼロになり、あらゆるものがそれに依存する | ドメインごとのパッケージに分割する |
| 初期化ブロックにビジネス・ロジックを記述する | 初回参照時に黙って実行されるため、デバッグが困難 | 明示的な初期化プロシージャを使用する |
| 本体のみの変更時に仕様部も再コンパイルする | すべての依存オブジェクトが無効化される | ロジック変更時は本体のみ、API変更時のみ仕様部を変更する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **Oracle 12c以降**: 表の不可視列（Invisible columns）は、その列が追加される前にコンパイルされたパッケージの `%ROWTYPE` には影響しません。再コンパイルが必要です。
- **Oracle 18c以降**: パッケージ仕様部内のプライベートなプロシージャ（12.2で導入された `ACCESSIBLE BY` を使用）により、パッケージ間のきめ細かなアクセス制御が可能。
- **Oracle 12.2以降**: `ACCESSIBLE BY` 句により、パッケージを呼び出せるユニットを制限可能。

```sql
-- 12.2以降: このパッケージへのアクセスを order_mgmt_pkg のみに制限する
CREATE OR REPLACE PACKAGE order_internals_pkg
  ACCESSIBLE BY (PACKAGE order_mgmt_pkg)
AS
  PROCEDURE internal_validate(p_order_id IN NUMBER);
END order_internals_pkg;
/
```

---

## ソース

- [Oracle Database PL/SQL Language Reference 19c — Packages](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-packages.html) — パッケージ構造、仕様部 vs 本体、オーバーロード、前方宣言、初期化
- [Oracle Database PL/SQL Language Reference 19c — ACCESSIBLE BY Clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/ACCESSIBLE-BY-clause.html) — 12.2以降のアクセス制御
- [DBMS_SESSION (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SESSION.html) — SET_CONTEXT によるアプリケーション・コンテキスト
