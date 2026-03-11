# PL/SQL セキュリティ (Security)

## 概要

PL/SQL セキュリティには、格納されたコードがデータベース権限をどのように行使するか、SQL インジェクションに対してどのように防御するか、および意図しない権限昇格をどのように回避するかという点が含まれます。Oracle は洗練された権限モデルとバリデーション・ユーティリティ（`DBMS_ASSERT`）を提供しており、これらを正しく使用することで、PL/SQL コードは一般的な攻撃ベクトルに対して耐性を持ちます。

---

## AUTHID: 定義者権限 vs 実行者権限

すべての PL/SQL ユニットは、**定義者権限**（デフォルト）または**実行者権限**（`AUTHID CURRENT_USER`）のいずれかで動作します。これにより、コードの実行時にどのスキーマの権限が使用されるかが決まります。

### 定義者権限 (デフォルト: AUTHID DEFINER)

誰がそのコードを呼び出すかに関わらず、そのコードを所有するスキーマの権限で実行されます。

```sql
-- 所有者スキーマ: APP_OWNER
CREATE OR REPLACE PROCEDURE definer_example
  AUTHID DEFINER  -- デフォルト。省略可能
AS
BEGIN
  -- APP_OWNER として実行される
  -- APP_OWNER は sensitive_data に対する SELECT 権限が必要
  -- 呼び出し側 (PUBLIC ユーザーなど) は、この権限を持っている必要はない
  INSERT INTO sensitive_data (col1) VALUES ('test');
END definer_example;
/
```

**定義者権限を使用すべきケース**:
- プロシージャが特定のスキーマの表にアクセスし、呼び出し側から権限の詳細を隠蔽（抽象化）したい場合
- 表への直接アクセスを許可せずに、プロシージャの実行（EXECUTE）権限のみを付与したい場合
- 呼び出し側が基礎となる表に直接アクセスできないように制限された API レイヤーを実装する場合

**セキュリティ上の影響**: 呼び出しユーザーは、自分自身が直接持っていない権限を行使できます。これは管理された API を提供するための設計通りの動作ですが、プロシージャに SQL インジェクションの脆弱性がある場合、攻撃者がその権限昇格を利用できるリスクとなります。

### 実行者権限 (AUTHID CURRENT_USER)

呼び出しユーザーの権限で実行されます。オブジェクトの解決は、呼び出し側のスキーマで行われます。

```sql
-- 呼び出し側のコンテキストで動作する汎用ユーティリティ
CREATE OR REPLACE PROCEDURE invoker_example
  AUTHID CURRENT_USER
AS
BEGIN
  -- 呼び出しユーザーとして実行される
  -- 'my_log' 表は、APP_OWNER ではなく呼び出し側のスキーマから解決される
  INSERT INTO my_log (log_time, message) VALUES (SYSDATE, 'Called');
  -- ^ 各呼び出し側は、自身のスキーマに my_log 表を持ち、INSERT 権限を持っている必要がある
END invoker_example;
/
```

**実行者権限を使用すべきケース**:
- 呼び出し側のオブジェクトに対して操作を行う必要がある汎用プロシージャ
- 汎用的なロギング、監査、またはバッチ・フレームワーク・コード
- すべてのテナントが同一のスキーマ構造を持つマルチテナント・システム
- 意図しない権限昇格を回避したい場合

### 比較表

| 特性 | 定義者権限 (Definer Rights) | 実行者権限 (Invoker Rights) |
|---|---|---|
| 使用される権限 | プロシージャの所有者 | プロシージャの呼び出し側 |
| オブジェクトの解決 | 所有者のスキーマ | 呼び出し側のスキーマ |
| 呼び出し側の表権限 | 不要 | 必要 |
| 共有 API での使用 | 適している | 適していない |
| 汎用ユーティリティでの使用 | リスクあり | 推奨される |
| 権限昇格のリスク | 高い | 低い |
| デフォルト動作 | はい | いいえ (明示的な指定が必要) |

```sql
-- パターン: データ・アクセス・レイヤーでの定義者権限
-- APP_OWNER がアプリ・ロールに EXECUTE 権限を付与し、表権限は直接付与しない
CREATE OR REPLACE PACKAGE customer_api_pkg
  AUTHID DEFINER
AS
  FUNCTION get_customer(p_id IN NUMBER) RETURN customers%ROWTYPE;
  PROCEDURE update_customer_email(p_id IN NUMBER, p_email IN VARCHAR2);
END customer_api_pkg;
/
-- GRANT EXECUTE ON customer_api_pkg TO app_role;
-- app_role から表への直接アクセスを剥奪し、パッケージ経由のみ許可する
```

---

## PL/SQL における SQL インジェクションの攻撃ベクトル

SQL インジェクションは、攻撃者が制御するデータが SQL または PL/SQL 文に直接連結された場合に発生します。PL/SQL では、主に動的 SQL で発生します。

### 脆弱なパターン

```sql
-- 脆弱な例: ユーザー入力を直接連結している
CREATE OR REPLACE PROCEDURE get_employee_bad(
  p_name IN VARCHAR2
) AS
  l_sql   VARCHAR2(1000);
  l_count NUMBER;
BEGIN
  -- p_name = "' OR '1'='1" の場合、すべての行が返される
  -- p_name = "' ; DROP TABLE employees;--" の場合、Oracle は無視する (EXECUTE IMMEDIATE は単一文)
  -- しかし： "' UNION SELECT password,null,null FROM dba_users--" はデータを漏洩させる可能性がある
  l_sql := 'SELECT COUNT(*) FROM employees WHERE last_name = ''' || p_name || '''';
  EXECUTE IMMEDIATE l_sql INTO l_count;
  DBMS_OUTPUT.PUT_LINE(l_count);
END get_employee_bad;
/
```

### 安全なパターン: バインド変数

```sql
-- 安全な例: バインド変数を使用 — p_name はあくまでデータとして扱われ、SQL 構文にはならない
CREATE OR REPLACE PROCEDURE get_employee_safe(
  p_name IN VARCHAR2
) AS
  l_count NUMBER;
BEGIN
  EXECUTE IMMEDIATE
    'SELECT COUNT(*) FROM employees WHERE last_name = :1'
    INTO l_count USING p_name;
  DBMS_OUTPUT.PUT_LINE(l_count);
END get_employee_safe;
/
```

### バインド変数が使用できないケース

表名、列名、およびスキーマ名は、構造的な SQL コンポーネントであるため、バインド変数にすることはできません。これらを動的に指定する場合は、連結する前に `DBMS_ASSERT` で入力を検証してください。

---

## DBMS_ASSERT パッケージ

`DBMS_ASSERT` は、動的 SQL で使用される前に、無効な、あるいは悪意のある可能性のある入力を検証して例外を発生させるルーチンを提供します。

```sql
-- 入力が単純な SQL 名（特殊文字なし）であることを検証
BEGIN
  DBMS_ASSERT.SIMPLE_SQL_NAME('employees');     -- OK: 'employees' を返す
  DBMS_ASSERT.SIMPLE_SQL_NAME('my table');      -- 例外 ORA-44003: 無効な SQL 名
  DBMS_ASSERT.SIMPLE_SQL_NAME('emp;DROP TABLE');-- 例外 ORA-44003
END;

-- 入力が存在するスキーマ・オブジェクトであることを検証
BEGIN
  DBMS_ASSERT.SQL_OBJECT_NAME('SCOTT.EMPLOYEES'); -- オブジェクトが存在すれば OK
  DBMS_ASSERT.SQL_OBJECT_NAME('nonexistent_obj'); -- 例外 ORA-44002: 見つからない
END;

-- 文字列リテラルを安全に引用符で囲む
DECLARE
  l_safe_val VARCHAR2(200);
BEGIN
  -- ENQUOTE_LITERAL は単一引用符で囲み、内部の引用符を 2 重にする
  l_safe_val := DBMS_ASSERT.ENQUOTE_LITERAL('O''Brien');
  -- 結果: 'O''Brien'  (SQL 文字列リテラルとして埋め込み可能)
  DBMS_OUTPUT.PUT_LINE(l_safe_val);
END;

-- 識別子 (オブジェクト名) を安全に引用符で囲む
DECLARE
  l_table_name VARCHAR2(100) := 'MY TABLE';  -- スペースを含む
  l_quoted     VARCHAR2(100);
BEGIN
  -- ENQUOTE_NAME は識別子として使用するために二重引用符で囲む
  l_quoted := DBMS_ASSERT.ENQUOTE_NAME('MY TABLE', FALSE);
  -- 結果: "MY TABLE"  (有効な引用付き識別子)
  DBMS_OUTPUT.PUT_LINE(l_quoted);
END;
```

### DBMS_ASSERT 関数リファレンス

| 関数 | 用途 | 発生する例外 |
|---|---|---|
| `SIMPLE_SQL_NAME(str)` | 特殊文字、スペース、SQL キーワードが含まれていないか検証 | ORA-44003 |
| `QUALIFIED_SQL_NAME(str)` | schema.object[.subobject] 形式であることを検証 | ORA-44004 |
| `SQL_OBJECT_NAME(str)` | 形式の検証に加え、オブジェクトの実在を確認 | ORA-44002 |
| `SCHEMA_NAME(str)` | 形式の検証に加え、スキーマの実在を確認 | ORA-44001 |
| `ENQUOTE_LITERAL(str)` | リテラル値として引用符で囲む（単一引用符、埋め込み引用符のエスケープ） | ドキュメント上の規定なし |
| `ENQUOTE_NAME(str, cap)` | 識別子として二重引用符で囲む | ドキュメント上の規定なし |
| `NOOP(str)` | 文字列をそのまま返す。検証が行われていないことを示す目的で使用 | 発生しない |

### 安全な動的 DDL パターン

```sql
CREATE OR REPLACE PROCEDURE create_partition(
  p_table_name     IN VARCHAR2,
  p_partition_name IN VARCHAR2,
  p_upper_bound    IN DATE
) AS
  l_table     VARCHAR2(128);
  l_partition VARCHAR2(128);
  l_sql       VARCHAR2(500);
BEGIN
  -- 動的 DDL に使用する前にインプットを検証
  l_table     := DBMS_ASSERT.SIMPLE_SQL_NAME(p_table_name);
  l_partition := DBMS_ASSERT.SIMPLE_SQL_NAME(p_partition_name);

  -- 日付値: TO_CHAR を使用して安全なリテラルを生成 (日付であればインジェクションの余地なし)
  l_sql := 'ALTER TABLE ' || l_table ||
           ' ADD PARTITION ' || l_partition ||
           ' VALUES LESS THAN (DATE ''' ||
           TO_CHAR(p_upper_bound, 'YYYY-MM-DD') || ''')';

  EXECUTE IMMEDIATE l_sql;
END create_partition;
/
```

---

## 定義者権限における権限昇格のリスク

SQL インジェクションの脆弱性を含む定義者権限のプロシージャでは、攻撃者は呼び出し側ではなく所有者の権限を行使できてしまいます。これは Oracle において最も危険な SQL インジェクションの形態です。

```sql
-- DBA_TOOLS (DBA権限を持つスキーマ) が所有者
-- SELECT ANY TABLE 権限を持っているとする
CREATE OR REPLACE PROCEDURE report_generator(
  p_table_name IN VARCHAR2
) AUTHID DEFINER AS
  l_sql    VARCHAR2(500);
  l_count  NUMBER;
BEGIN
  -- 脆弱：DBA権限 ＋ SQLインジェクション ＝ あらゆる表を読み取られる
  l_sql := 'SELECT COUNT(*) FROM ' || p_table_name;
  EXECUTE IMMEDIATE l_sql INTO l_count;
  DBMS_OUTPUT.PUT_LINE(l_count);
END;
-- 攻撃者が渡す値の例: 'SYS.USER$ WHERE ROWNUM=1 UNION SELECT username FROM dba_users--'
-- 結果：攻撃者は DBA_TOOLS の権限を利用して DBA_USERS などを読み取ることができる
```

**動的 SQL を含む定義者権限プロシージャの対策チェックリスト**:
1. すべてのデータ値にバインド変数を使用する
2. すべての構造的 SQL 要素（表名、列名等）を `DBMS_ASSERT` で検証する
3. 許可する表名/列名のホワイトリストを管理テーブル等で保持し、それと照らし合わせて検証する
4. 公開プロシージャを所有するスキーマに `SELECT ANY TABLE` や `DBA` ロールを付与しない

---

## セキュア・コーディング・チェックリスト

- [ ] すべての動的 SQL で、データ値にバインド変数を使用しているか
- [ ] 動的 SQL 内の表名/列名が `DBMS_ASSERT.SIMPLE_SQL_NAME` で検証されているか
- [ ] 定義者権限のプロシージャが、意図せず呼び出し側に高い権限を与えていないか
- [ ] エラー・メッセージが、内部のスキーマ名や SQL 構造をエンド・ユーザーに公開していないか
- [ ] 事前に存在を確認すべきオブジェクトに `DBMS_ASSERT.SQL_OBJECT_NAME` を使用しているか
- [ ] PL/SQL ソース内に認証情報（パスワード等）をハードコードしていないか
- [ ] `EXECUTE IMMEDIATE` で文字列連結ではなく `USING` 句（バインド変数）を使用しているか
- [ ] 呼び出し側の権限が適切な機密性の高いプロシージャで、`AUTHID CURRENT_USER` を検討したか
- [ ] パッケージ仕様部で、公開すべきでない内部型やカーソルを露出させていないか
- [ ] 監査ログに呼び出しユーザー (`SYS_CONTEXT('USERENV','SESSION_USER')`) を含めているか

---

## Oracle Database Vault におけるコンテキスト

Oracle Database Vault は、標準の Oracle 権限の上にレルムベースのアクセス制御を追加します。適切に構成されていれば、DBA ロールを持つアカウントであってもアプリケーション・データへのアクセスをブロックできます。

```sql
-- Database Vault が有効かどうかを確認
SELECT parameter, value FROM v$option WHERE parameter = 'Oracle Database Vault';

-- Vault 有効時、PL/SQL で現在のセッション・コンテキストを読み取ることができる
SELECT SYS_CONTEXT('DVSYS', 'ROLE') FROM DUAL;  -- 現在のレルム・ロール

-- 例：接続元 IP や認証方法に基づいて実行を制限
IF SYS_CONTEXT('DVSYS', 'NETWORK_IP_ADDRESS') NOT LIKE '10.0.%' THEN
  RAISE_APPLICATION_ERROR(-20403, '許可されていないネットワーク場所からのアクセスです');
END IF;
```

---

## 追加のセキュリティ・パターン

### パッケージへのアクセス制限 (12.2+)

```sql
-- sensitive_ops_pkg を、特定のパッケージからのみ呼び出し可能にする
CREATE OR REPLACE PACKAGE sensitive_ops_pkg
  ACCESSIBLE BY (PACKAGE order_processor_pkg, PACKAGE batch_runner_pkg)
AS
  PROCEDURE delete_order(p_id IN NUMBER);
END sensitive_ops_pkg;
/
-- これ以外のユニットから呼び出そうとすると、コンパイル・エラー PLS-00904 が発生する
```

### エラー・メッセージによるデータ漏洩の回避

```sql
-- 誤り：内部の SQL 構造が露出する
EXCEPTION
  WHEN OTHERS THEN
    RAISE_APPLICATION_ERROR(-20001,
      'Error in: SELECT * FROM ' || l_table_name || ' - ' || SQLERRM);

-- 正しい：クライアントには汎用メッセージを返し、内部ログに詳細を残す
EXCEPTION
  WHEN OTHERS THEN
    error_logger_pkg.log_error('ORDER_PKG', 'PROCESS_ORDER');
    RAISE_APPLICATION_ERROR(-20001, '内部エラーが発生しました。参照 ID: ' || l_log_id);
```

---

## よくある間違いとアンチパターン

| アンチパターン | リスク | 対策 |
|---|---|---|
| 動的 SQL での文字列連結 | SQL インジェクション | バインド変数 + `DBMS_ASSERT` |
| インジェクション脆弱性のあるコードへの定義者権限付与 | 権限昇格 | インジェクションを修正し、実行者権限を検討する |
| エラー・メッセージでのスキーマ構造露出 | 情報漏洩 | クライアントへのエラーを汎用化し、内部ログを使用する |
| `EXECUTE ANY PROCEDURE` の付与 | 呼び出し側があらゆる定義者権限コードを実行可能になる | 特定のプロシージャのみに権限を付与する |
| PL/SQL 内のパスワードのハードコード | 認証情報の漏洩 | Oracle Wallet や認証情報なしの DB リンクを使用する |
| DDL 前に `p_table_name` を検証しない | 表名インジェクション | `DBMS_ASSERT.SIMPLE_SQL_NAME` + ホワイトリスト |
| 検証の代わりに `NOOP` を使用する | 実際には何も検証されない | 適切な `DBMS_ASSERT` 関数を使用する |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **Oracle 12.2以降**: `ACCESSIBLE BY` 句により、PL/SQL ユニット間のアクセス制御がコンパイル時に強制されます。
- **Oracle 12c以降**: `DBMS_PRIVILEGE_CAPTURE` による権限分析が可能となり、実際に使用されている権限を特定して最小権限の原則を適用できます。
- **全バージョン**: `DBMS_ASSERT` は Oracle 10.2 以降で利用可能です。動的 SQL を構築する際には一貫して使用してください。

---

## ソース

- [Oracle Database PL/SQL Language Reference 19c — AUTHID Clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/AUTHID-clause.html)
- [DBMS_ASSERT (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_ASSERT.html)
- [Oracle Database Security Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/)
- [Oracle Database PL/SQL Language Reference 19c — ACCESSIBLE BY Clause](https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/ACCESSIBLE-BY-clause.html)
