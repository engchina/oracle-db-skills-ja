# PL/SQL エラー処理 (Error Handling)

## 概要

堅牢なエラー処理は、PL/SQL 開発において最も重要な側面の一つです。Oracle は構造化された例外モデルを提供しており、正しく使用することで、信頼性が高く、診断可能で、メンテナンスしやすいコードを作成できます。このガイドでは、例外の階層、呼び出し手法、診断機能、ログ・パターン、および伝播ルールについて説明します。

---

## Oracle 例外の階層

すべての Oracle 例外は、共通の内部構造から派生しています。例外は以下の 3 つのカテゴリに分けられます。

### 1. 定義済み例外 (Predefined Exceptions)

一般的な Oracle エラーに対して名前が付けられた例外です。宣言は不要で、直接使用できます。

```sql
BEGIN
  SELECT salary INTO l_salary FROM employees WHERE employee_id = p_id;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    -- ORA-01403: SELECT INTO で行が返されなかった
    DBMS_OUTPUT.PUT_LINE('従業員が見つかりません');
  WHEN TOO_MANY_ROWS THEN
    -- ORA-01422: SELECT INTO で複数の行が返された
    DBMS_OUTPUT.PUT_LINE('複数の従業員が一致しました');
  WHEN VALUE_ERROR THEN
    -- ORA-06502: 変換エラーまたはサイズ制約エラー
    DBMS_OUTPUT.PUT_LINE('データ型またはサイズが一致しません');
  WHEN ZERO_DIVIDE THEN
    DBMS_OUTPUT.PUT_LINE('ゼロ除算が発生しました');
  WHEN DUP_VAL_ON_INDEX THEN
    -- ORA-00001: 一意制約違反
    DBMS_OUTPUT.PUT_LINE('値が重複しています');
  WHEN CURSOR_ALREADY_OPEN THEN
    DBMS_OUTPUT.PUT_LINE('カーソルはすでに開いています');
  WHEN INVALID_CURSOR THEN
    DBMS_OUTPUT.PUT_LINE('無効なカーソル操作です');
  WHEN LOGIN_DENIED THEN
    DBMS_OUTPUT.PUT_LINE('ユーザー名またはパスワードが無効です');
  WHEN PROGRAM_ERROR THEN
    DBMS_OUTPUT.PUT_LINE('内部 PL/SQL エラーが発生しました');
  WHEN TIMEOUT_ON_RESOURCE THEN
    DBMS_OUTPUT.PUT_LINE('リソースのタイムアウトが発生しました');
END;
/
```

**主な定義済み例外のリファレンス:**

| 例外名 | Oracle エラー | 説明 |
|---|---|---|
| `NO_DATA_FOUND` | ORA-01403 | SELECT INTO で行が返されなかった |
| `TOO_MANY_ROWS` | ORA-01422 | SELECT INTO で複数の行が返された |
| `DUP_VAL_ON_INDEX` | ORA-00001 | 一意制約違反 |
| `VALUE_ERROR` | ORA-06502 | 変換またはサイズのエラー |
| `ZERO_DIVIDE` | ORA-01476 | ゼロ除算 |
| `INVALID_NUMBER` | ORA-01722 | 暗黙的な数値変換の失敗 |
| `CURSOR_ALREADY_OPEN` | ORA-06511 | すでに開いているカーソルを再度開こうとした |
| `INVALID_CURSOR` | ORA-01001 | 無効なカーソルの参照 |
| `ROWTYPE_MISMATCH` | ORA-06504 | カーソル変数の型不一致 |
| `COLLECTION_IS_NULL` | ORA-06531 | 未初期化のコレクションに対する操作 |
| `SUBSCRIPT_BEYOND_COUNT` | ORA-06533 | コレクションのサイズを超えた索引指定 |
| `SUBSCRIPT_OUTSIDE_LIMIT` | ORA-06532 | 許容範囲外の索引指定 |
| `STORAGE_ERROR` | ORA-06500 | メモリ不足 |
| `TIMEOUT_ON_RESOURCE` | ORA-00051 | リソース待機中のタイムアウト |
| `LOGIN_DENIED` | ORA-01017 | 認証失敗 |

### 2. ユーザー定義例外 (User-Defined Exceptions)

ビジネス・ロジック・エラーのために、パッケージ仕様部またはプロシージャ/ファンクション内で宣言されます。

```sql
CREATE OR REPLACE PACKAGE order_exceptions_pkg AS
  e_order_not_found    EXCEPTION;
  e_insufficient_stock EXCEPTION;
  e_order_already_shipped EXCEPTION;
END order_exceptions_pkg;
/

-- ビジネス・ロジック内での使用例
PROCEDURE ship_order(p_order_id IN NUMBER) IS
  l_status VARCHAR2(20);
  l_stock  NUMBER;
BEGIN
  SELECT status INTO l_status
  FROM   orders WHERE order_id = p_order_id;

  IF l_status = 'SHIPPED' THEN
    RAISE order_exceptions_pkg.e_order_already_shipped;
  END IF;

  -- ... 出荷処理 ...

EXCEPTION
  WHEN order_exceptions_pkg.e_order_already_shipped THEN
    DBMS_OUTPUT.PUT_LINE('注文はすでに出荷済みです');
  WHEN NO_DATA_FOUND THEN
    RAISE order_exceptions_pkg.e_order_not_found;  -- ビジネス例外として再送出
END ship_order;
/
```

### 3. 名前なし (匿名) 例外と PRAGMA EXCEPTION_INIT

名前が付いていない Oracle エラー番号に、任意の名前を関連付けます。

```sql
DECLARE
  -- ORA-02292 (整合性制約違反 - 子レコードが存在する) に名前を割り当てる
  e_child_records_exist EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_child_records_exist, -2292);

  -- ORA-00054: リソース・ビジー (NOWAIT 指定時のロック)
  e_resource_busy EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_resource_busy, -54);

BEGIN
  DELETE FROM departments WHERE department_id = 10;

EXCEPTION
  WHEN e_child_records_exist THEN
    DBMS_OUTPUT.PUT_LINE('削除できません — 子レコードが存在します');
  WHEN e_resource_busy THEN
    DBMS_OUTPUT.PUT_LINE('対象の行は別のセッションによってロックされています');
END;
/
```

**ベスト・プラクティス**: よく使用する PRAGMA 例外は共通の例外パッケージで宣言し、各所で再宣言しないようにします。

```sql
CREATE OR REPLACE PACKAGE app_exceptions_pkg AS
  e_child_records_exist EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_child_records_exist, -2292);

  e_resource_busy EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_resource_busy, -54);

  e_deadlock EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_deadlock, -60);

  e_snapshot_too_old EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_snapshot_too_old, -1555);
END app_exceptions_pkg;
/
```

---

## RAISE と RAISE_APPLICATION_ERROR

### RAISE

現在の例外を再送出するか、宣言された例外を発生させます。

```sql
PROCEDURE process_data(p_id IN NUMBER) IS
BEGIN
  validate_data(p_id);
EXCEPTION
  WHEN VALUE_ERROR THEN
    -- ログを記録し、元の例外 (フル・スタックを保持) を再送出
    log_error(SQLERRM);
    RAISE;  -- VALUE_ERROR を元のスタック情報のまま再送出
END;
```

### RAISE_APPLICATION_ERROR

カスタム・メッセージとともにアプリケーション定義のエラーを発生させます。エラー番号は `-20000` から `-20999` の範囲である必要があります。

```sql
PROCEDURE validate_age(p_age IN NUMBER) IS
BEGIN
  IF p_age IS NULL THEN
    RAISE_APPLICATION_ERROR(-20001, '年齢を NULL にすることはできません');
  END IF;
  IF p_age < 0 OR p_age > 150 THEN
    RAISE_APPLICATION_ERROR(-20002, '年齢が有効範囲外です: ' || p_age);
  END IF;
END validate_age;
```

**第 3 引数 — 既存のエラー・スタックの保持:**

```sql
EXCEPTION
  WHEN OTHERS THEN
    -- TRUE = 既存のエラー・スタックに追加、FALSE = 置換 (デフォルト)
    RAISE_APPLICATION_ERROR(-20500, '注文処理中に予期しないエラーが発生しました', TRUE);
END;
```

`TRUE` を渡すと、元の Oracle エラーがスタックに保持され、診断が容易になります。

---

## エラー診断機能

### SQLERRM

現在または特定のエラー・コードに対するエラー・メッセージを返します。

```sql
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('エラー: ' || SQLERRM);
    -- 出力例: "ORA-01403: データが見つかりません"
    -- 引数なしの SQLERRM = 現在の例外
    -- SQLERRM(-1403) = そのエラー・コードのメッセージ
END;
```

**制限**: `SQLERRM` は 512 バイトで切り捨てられます。完全なメッセージを取得するには `DBMS_UTILITY.FORMAT_ERROR_STACK` を使用してください。

### DBMS_UTILITY.FORMAT_ERROR_STACK

連鎖したエラーを含む、完全なエラー・メッセージ (最大 2000 バイト) を返します。

```sql
EXCEPTION
  WHEN OTHERS THEN
    l_error_stack := DBMS_UTILITY.FORMAT_ERROR_STACK;
    -- SQLERRM のように 512 バイトで切り捨てられることなく完全なエラーを返す
END;
```

### DBMS_UTILITY.FORMAT_ERROR_BACKTRACE

**デバッグに不可欠**: 例外が捕捉された場所ではなく、**最初に関数が発生した行番号**を返します。

```sql
PROCEDURE outer_proc IS
BEGIN
  inner_proc;
EXCEPTION
  WHEN OTHERS THEN
    -- FORMAT_ERROR_BACKTRACE がないと、ここでエラーを捕捉したことしか分からない
    -- これを使用すれば inner_proc のどの行でエラーが発生したかが特定できる
    DBMS_OUTPUT.PUT_LINE('エラー・スタック: ' || DBMS_UTILITY.FORMAT_ERROR_STACK);
    DBMS_OUTPUT.PUT_LINE('バックトレース: '   || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
    -- 出力例:
    -- バックトレース: ORA-06512: "MYSCHEMA.INNER_PROC", 15行目
    --                ORA-06512: "MYSCHEMA.OUTER_PROC", 3行目
    RAISE;
END outer_proc;
```

### 比較表

| 機能 | 最大長 | 行番号の表示 | 呼び出しチェーンの表示 | 推奨用途 |
|---|---|---|---|---|
| `SQLERRM` | 512 バイト | なし | なし | 簡易的なログ記録 |
| `DBMS_UTILITY.FORMAT_ERROR_STACK` | 2000 バイト | なし | あり (連鎖エラー) | 完全なエラー・メッセージ |
| `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` | 可変 | あり | あり (フル・スタック) | 発生源の特定 |

**ベスト・プラクティス**: エラー・ログには常に `FORMAT_ERROR_STACK` と `FORMAT_ERROR_BACKTRACE` の両方を記録してください。

---

## 自律型トランザクションによる堅牢なエラー・ログ

課題: メイン・トランザクションがエラーによりロールバックされた場合、通常の `INSERT INTO error_log` もロールバックされてしまいます。ログを確実に残すために、`PRAGMA AUTONOMOUS_TRANSACTION` を使用します。

```sql
CREATE OR REPLACE PACKAGE BODY error_logger_pkg AS

  PROCEDURE log_error(
    p_module    IN VARCHAR2,
    p_procedure IN VARCHAR2,
    p_message   IN VARCHAR2 DEFAULT NULL
  ) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
    l_error_stack    VARCHAR2(4000);
    l_error_backtrace VARCHAR2(4000);
  BEGIN
    l_error_stack    := SUBSTR(DBMS_UTILITY.FORMAT_ERROR_STACK,    1, 4000);
    l_error_backtrace := SUBSTR(DBMS_UTILITY.FORMAT_ERROR_BACKTRACE, 1, 4000);

    INSERT INTO error_log (
      log_id,
      log_timestamp,
      db_user,
      os_user,
      module_name,
      procedure_name,
      error_stack,
      error_backtrace,
      custom_message,
      session_id
    ) VALUES (
      error_log_seq.NEXTVAL,
      SYSTIMESTAMP,
      SYS_CONTEXT('USERENV', 'SESSION_USER'),
      SYS_CONTEXT('USERENV', 'OS_USER'),
      p_module,
      p_procedure,
      l_error_stack,
      l_error_backtrace,
      p_message,
      SYS_CONTEXT('USERENV', 'SESSIONID')
    );
    COMMIT;  -- 自律型トランザクションのコミット — メイン・トランザクションには影響しない
  EXCEPTION
    WHEN OTHERS THEN
      -- ロギング自体が失敗した場合は伝播させない (元のエラーを隠さないため)
      ROLLBACK;
  END log_error;

END error_logger_pkg;
/

-- 使用パターン
PROCEDURE process_order(p_order_id IN NUMBER) IS
BEGIN
  -- ... ビジネス・ロジック ...
EXCEPTION
  WHEN OTHERS THEN
    error_logger_pkg.log_error(
      p_module    => 'ORDER_MGMT',
      p_procedure => 'PROCESS_ORDER',
      p_message   => 'order_id=' || p_order_id
    );
    RAISE;  -- ログ記録後は常に再送出する
END process_order;
```

---

## 例外の再送出 (Re-Raising)

### 単純な再送出 (引数なしの RAISE)

元の例外タイプとエラー・コードをすべて保持します。

```sql
EXCEPTION
  WHEN OTHERS THEN
    log_error(...);
    RAISE;  -- 元の例外がそのまま上位に伝播
```

### 別の例外としての再送出

スタックを保持したまま、元の例外を新しいアプリケーション・エラーでラップします。

```sql
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RAISE_APPLICATION_ERROR(
      -20404,
      '顧客が見つかりません: id=' || p_customer_id,
      TRUE  -- 元の ORA-01403 をスタックに含める
    );
END;
```

### ネストしたブロック内での再送出

```sql
BEGIN
  BEGIN
    -- 内側ブロック
    risky_operation;
  EXCEPTION
    WHEN DUP_VAL_ON_INDEX THEN
      -- ローカルで処理し、再送出しない — 例外はここで吸収される
      handle_duplicate;
  END;
  -- 例外が処理された場合、ここから続行される
  next_operation;
EXCEPTION
  WHEN OTHERS THEN
    -- 内側ブロックで処理されなかった例外のみを捕捉
    log_and_raise;
END;
```

---

## 例外の伝播ルール

1. 例外が発生し、現在のブロックに **ハンドラがない** 場合、**外側のブロック** に伝播します。
2. ハンドラのないまま **最も外側のブロック** に達した場合は、**呼び出し側** に伝播します。
3. どの呼び出し側でも処理されない場合、エラー・メッセージとともにクライアントに返されます。
4. 例外ハンドラは記述順に検索され、**最初に一致した** ハンドラが実行されます。
5. `WHEN OTHERS` はすべてに一致し、ブロックの **最後** のハンドラである必要があります。
6. **ハンドラ内で発生した** 例外は、即座に外側のブロックに伝播します（現在のブロックのハンドラはすでに使い果たされているため）。

```sql
-- 伝播の例
PROCEDURE a IS
BEGIN
  b;  -- b で未処理の例外が発生すると、ここに伝播する
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    handle_in_a;  -- b から伝播した NO_DATA_FOUND を処理
END a;

PROCEDURE b IS
BEGIN
  c;  -- c で未処理の例外が発生すると、ここに伝播する
  -- NO_DATA_FOUND のハンドラがないため、a へ伝播する
END b;

PROCEDURE c IS
BEGIN
  SELECT ... INTO ... FROM ...;  -- NO_DATA_FOUND が発生
  -- ハンドラなし — b へ伝播する
END c;
```

---

## WHEN OTHERS THEN NULL の危険性

これは PL/SQL 開発において最も危険なアンチパターンの一つです。すべての例外を黙って飲み込んでしまうため、バグの診断が不可能になります。

```sql
-- 危険: 絶対に行わないこと
EXCEPTION
  WHEN OTHERS THEN
    NULL;  -- 例外が黙って消失する
           -- 呼び出し側は失敗したことを知る由がない
           -- データが不整合になる可能性がある
           -- 本番環境での診断が不可能になる

-- これも危険: ログは記録するが再送出しない
EXCEPTION
  WHEN OTHERS THEN
    log_error(...);  -- ログは残るがそのまま通過する
    -- 呼び出し側は操作が成功したと思い込んでしまう！

-- 正解: ログを記録して再送出する (または新しい例外を発生させる)
EXCEPTION
  WHEN OTHERS THEN
    error_logger_pkg.log_error(
      p_module    => 'ORDER_PKG',
      p_procedure => 'PROCESS_ORDER'
    );
    RAISE;  -- 呼び出し側に判断を委ねる
```

**唯一の正当な用途**は、失敗が真に許容され、ドキュメント化されている「ベスト・エフォート」な操作（ロギング・プロシージャ自体の中など）だけです。

---

## 例外処理のベスト・プラクティス

1. **常に `FORMAT_ERROR_BACKTRACE` を記録する**: 行番号の情報は非常に貴重です。
2. **ログ記録には自律型トランザクションを使用する**: ロールバックに影響されずにログを残せます。
3. **`WHEN OTHERS THEN NULL` は絶対に使用しない**: 常にログを記録し、再送出します。
4. **エラー・ログを集中管理する**: 単一のパッケージでロギングを行い、一貫性を保ちます。
5. **共通パッケージで例外に名前を付ける**: 各パッケージで PRAGMA 例外を再定義しないようにします。
6. **具体的な例外から先に記述する**: `WHEN OTHERS` の前に `WHEN NO_DATA_FOUND` などを記述します。
7. **空の例外セクションは避ける**: 捕捉したからには、意味のある処理を行ってください。
8. **元のエラーを保持する**: `RAISE_APPLICATION_ERROR` を使用する際は、第 3 引数に `TRUE` を渡します。
9. **制御フローに例外を使用しない**: 例外は予期しない事態のためのものであり、通常の条件分岐用ではありません。

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| `WHEN OTHERS THEN NULL` | エラーを黙って飲み込む | 常にログを記録して再送出する |
| 再送出せずに例外を捕捉 | 呼び出し側が成功と誤認する | 再送出するか新しい例外を発生させる |
| `SQLERRM` のみの使用 | 512 バイトで切れる、行番号がない | `FORMAT_ERROR_STACK` + `FORMAT_ERROR_BACKTRACE` を使用 |
| 自律型トランザクションなしのログ記録 | メイン処理の失敗とともにログも消える | `PRAGMA AUTONOMOUS_TRANSACTION` を使用 |
| ユーザー・エラー番号の重複 | Oracle の内部エラーと衝突する可能性がある | -20000 〜 -20999 の範囲のみを使用 |
| ラップ時に `TRUE` を渡さない | 元のエラー・コンテキストが失われる | 第 3 引数に `TRUE` を指定 |
| ハンドラ内で例外が発生 | 外側ブロックへ予期せず伝播する | ハンドラ内のコードをネストした BEGIN/END で囲う |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。デフォルトや非推奨がリリース・アップデートによって異なる場合があるため。

- **Oracle 12c以降**: `UTL_CALL_STACK` パッケージにより、コール・スタック、エラー・スタック、バックトレースをプログラムで検査する API が提供されます（`DBMS_UTILITY` yよりも高機能）。
- **全バージョン**: `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` を機能させるには、例外ハンドラ内から呼び出す必要があります。
- **Oracle 10g以降**: `DBMS_UTILITY.FORMAT_ERROR_BACKTRACE` が導入されました。これ以前はデバッガなしで行番号の特定は困難でした。

```sql
-- Oracle 12c以降: UTL_CALL_STACK による構造化されたスタック検査
EXCEPTION
  WHEN OTHERS THEN
    FOR i IN 1..UTL_CALL_STACK.ERROR_DEPTH LOOP
      DBMS_OUTPUT.PUT_LINE(
        'エラー ' || i || ': ' ||
        UTL_CALL_STACK.ERROR_NUMBER(i) || ' - ' ||
        UTL_CALL_STACK.ERROR_MSG(i)
      );
    END LOOP;
    FOR i IN 1..UTL_CALL_STACK.BACKTRACE_DEPTH LOOP
      DBMS_OUTPUT.PUT_LINE(
        'バックトレース ' || i || ': ' ||
        UTL_CALL_STACK.BACKTRACE_UNIT(i) || ' ' ||
        UTL_CALL_STACK.BACKTRACE_LINE(i) || '行目'
      );
    END LOOP;
END;
```

---

## ソース

- Oracle Database 19c PL/SQL Language Reference — Exception Handling: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-error-handling.html
- Oracle Database 19c PL/SQL Language Reference — Predefined Exceptions: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/predefined-exceptions.html
- Oracle Database 19c PL/SQL Packages Reference — DBMS_UTILITY: https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_UTILITY.html
- Oracle Database 19c PL/SQL Packages Reference — UTL_CALL_STACK: https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/UTL_CALL_STACK.html
