# PL/SQL コンパイラ・オプション (Compiler Options)

## 概要

Oracle は、PL/SQL コンパイラの構成パラメータを豊富に提供しており、最適化レベル、実行モデル、条件付きコンパイル、および環境検出機能を制御できます。これらのオプションを理解することで、開発者はパフォーマンスを微調整し、環境を認識するコードを作成し、エディションベースのリフレッシュ（リリース管理）を実装できるようになります。

---

## PLSQL_OPTIMIZE_LEVEL

PL/SQL オプティマイザの「積極性」を制御します。レベルを上げるとコードの実行速度は上がりますが、デバッグが困難になる場合があります。

| レベル | 名称 | 説明 |
|---|---|---|
| 0 | なし | Oracle9i の評価順序、副作用、例外、およびパッケージ初期化を維持します。非常に古いコードとの互換性デバッグにのみ使用します。 |
| 1 | 基本 | 不必要な計算や例外の排除などの幅広い最適化を適用しますが、通常は元のソース順序を維持します。迅速な開発に適したバランスです。 |
| 2 | 標準 | **デフォルト。** レベル 1 を超える最新の最適化技術（コード移動の最適化など）を適用します。ほとんどのコードにおいて最高の実行パフォーマンスを提供します。 |
| 3 | 積極的 | レベル 2 を超える高度な最適化（サブプログラムの自動インライン展開など）を適用します。このレベルでの `PRAGMA INLINE 'YES'` は、インライン展開の優先順位が高い呼び出しとしてマークされます。 |

```sql
-- セッション・レベルで設定 (セッション内の以降のすべてのコンパイルに影響)
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL = 2;  -- デフォルト

-- 特定のコンパイルに対して設定
ALTER PROCEDURE my_proc COMPILE PLSQL_OPTIMIZE_LEVEL = 3;

-- システム・レベルで設定 (オーバーライドされない限り、すべてのセッションに影響)
ALTER SYSTEM SET PLSQL_OPTIMIZE_LEVEL = 2 SCOPE = BOTH;

-- 個別オブジェクトの DDL 内で設定
CREATE OR REPLACE PROCEDURE fast_proc IS
  PRAGMA INLINE(helper_func, 'YES');  -- レベル 2 以上で、この呼び出しをインライン展開
BEGIN
  helper_func(42);
END fast_proc;
/
```

### PRAGMA INLINE (オプティマイザ・レベル 2 以上)

`PRAGMA INLINE` は、特定のファンクション呼び出しをインライン展開する（またはしない）ようにオプティマイザに指示できます。レベル 2 以上で有効です：

- レベル 2：`'YES'` はインライン展開を実行し、`'NO'` は回避します。
- レベル 3：`'YES'` はインライン展開の優先順位が高いことを示し（最終決定はコンパイラが行います）、`'NO'` は回避します。

```sql
CREATE OR REPLACE FUNCTION calculate_tax(p_amount NUMBER) RETURN NUMBER IS
BEGIN
  RETURN ROUND(p_amount * 0.0825, 2);
END calculate_tax;
/

CREATE OR REPLACE PROCEDURE process_invoices IS
BEGIN
  FOR inv IN (SELECT invoice_id, amount FROM invoices) LOOP
    PRAGMA INLINE(calculate_tax, 'YES');  -- この呼び出しをインライン展開
    UPDATE invoices
    SET    tax_amount = calculate_tax(inv.amount)
    WHERE  invoice_id = inv.invoice_id;
  END LOOP;
END process_invoices;
/
```

### 現在の最適化レベルの確認

```sql
-- コンパイル済みオブジェクトの最適化レベルを確認
SELECT object_name, object_type, plsql_optimize_level
FROM   user_plsql_object_settings
WHERE  object_type IN ('PROCEDURE', 'FUNCTION', 'PACKAGE BODY');

-- セッション・レベルの設定を確認
SELECT value FROM v$parameter WHERE name = 'plsql_optimize_level';
```

---

## PLSQL_CODE_TYPE: INTERPRETED vs NATIVE

PL/SQL をインタープリタ型のバイトコード（デフォルト）としてコンパイルするか、ネイティブの機械語コードとしてコンパイルするかを制御します。

| 設定値 | 説明 |
|---|---|
| `INTERPRETED` | デフォルト。PVM (PL/SQL Virtual Machine) バイトコードにコンパイルされ、実行時に解釈されます。 |
| `NATIVE` | ネイティブの C コード、さらにマシン命令にコンパイルされます。CPU 集約型のコードに適しています。 |

```sql
-- セッション・レベルで設定
ALTER SESSION SET PLSQL_CODE_TYPE = NATIVE;

-- 特定のオブジェクトをネイティブとしてコンパイル
ALTER PROCEDURE number_crunch COMPILE PLSQL_CODE_TYPE = NATIVE;

-- 既存オブジェクトの設定を確認
SELECT object_name, object_type, plsql_code_type
FROM   user_plsql_object_settings
ORDER BY object_type, object_name;
```

### ネイティブ・コンパイルが有効なケース

ネイティブ・コンパイルは、以下のケースで最も効果を発揮します：
- **CPU 集約型の数値計算アルゴリズム**（暗号化、圧縮、科学計算など）
- 複雑な PL/SQL ロジックを含む**タイトなループ**
- 起動コストよりも計算負荷が重視される**重い処理**

以下のようなケースでは**ほとんど、または全く効果がありません**：
- ほとんどの時間を SQL 操作に費やすプロシージャ（SQL エンジンで動作するため）
- I/O 境界の操作
- Java ストアド・プロシージャや外部プロシージャを頻繁に呼び出すコード

---

## 条件付きコンパイル (Conditional Compilation)

条件付きコンパイルを使用すると、boolean 条件や照会ディレクティブに基づいて、コンパイル時に異なるコードを含めたり除外したりできます。プリプロセッサは、通常の PL/SQL コンパイラの前に実行されます。

### 基本構文

```sql
-- $IF 条件 $THEN ... [$ELSIF 条件 $THEN ...] [$ELSE ...] $END
-- 注意: プリプロセッサ・ディレクティブにはセミコロンを付けません

CREATE OR REPLACE PROCEDURE conditional_example IS
  $IF $$debug_mode $THEN
    l_start_time TIMESTAMP := SYSTIMESTAMP;
  $END
BEGIN
  $IF $$debug_mode $THEN
    DBMS_OUTPUT.PUT_LINE('開始時刻: ' || TO_CHAR(l_start_time, 'HH24:MI:SS.FF3'));
  $END

  process_data;

  $IF $$debug_mode $THEN
    DBMS_OUTPUT.PUT_LINE('経過時間: ' ||
      EXTRACT(SECOND FROM (SYSTIMESTAMP - l_start_time)) || '秒');
  $END
END conditional_example;
/
```

---

## PLSQL_CCFLAGS: コンパイル時定数

`PLSQL_CCFLAGS` は、`$IF` 条件で使用されるカスタムの boolean または integer 型のコンパイル時定数を定義します。

```sql
-- コンパイル前にセッション・レベルでフラグを設定
ALTER SESSION SET PLSQL_CCFLAGS = 'debug_mode:TRUE, env_dev:TRUE, version:2';

-- あるいはシステム・レベルで設定 (将来のすべてのコンパイルに影響)
ALTER SYSTEM SET PLSQL_CCFLAGS = 'debug_mode:FALSE, env_dev:FALSE, version:2' SCOPE = BOTH;

-- コード内での使用例
CREATE OR REPLACE PROCEDURE env_aware_proc IS
BEGIN
  $IF $$debug_mode $THEN
    -- このブロックは debug_mode=TRUE の場合のみコンパイルされる
    DBMS_OUTPUT.PUT_LINE('[DEBUG] env_aware_proc を開始します');
  $END

  $IF $$version = 2 $THEN
    process_v2;
  $ELSIF $$version = 1 $THEN
    process_v1;
  $ELSE
    process_default;
  $END
END env_aware_proc;
/
```

### 環境検出パターン

```sql
-- 環境固有の動作をコンパイル
-- 本番: ALTER SESSION SET PLSQL_CCFLAGS = 'env_prod:TRUE, env_dev:FALSE';
-- 開発: ALTER SESSION SET PLSQL_CCFLAGS = 'env_prod:FALSE, env_dev:TRUE';

CREATE OR REPLACE FUNCTION get_service_url(p_service IN VARCHAR2) RETURN VARCHAR2 IS
BEGIN
  $IF $$env_prod $THEN
    RETURN 'https://api.production.com/' || p_service;
  $ELSIF $$env_dev $THEN
    RETURN 'https://api.dev.internal/' || p_service;
  $ELSE
    RETURN 'https://api.dev.internal/' || p_service;
  $END
END get_service_url;
/
```

---

## バージョン互換性のためのコンパイル時定数

条件付きコンパイルにより、単一のコード・ベースで複数の Oracle データベース・バージョンをサポートできます。

```sql
-- DBMS_DB_VERSION は、コンパイル時定数として Oracle バージョンを提供します
CREATE OR REPLACE PROCEDURE version_compatible_proc IS
BEGIN
  $IF DBMS_DB_VERSION.VER_LE_12_1 $THEN
    -- Oracle 12.1 以前: 回避策を使用
    legacy_12c_approach;
  $ELSIF DBMS_DB_VERSION.VER_LE_18 $THEN
    -- Oracle 12.2 から 18c
    intermediate_approach;
  $ELSE
    -- Oracle 19c 以上: 最新機能を使用
    modern_approach;
  $END
END version_compatible_proc;
/
```

---

## エディションベースのコンパイル (Edition-Based Redefinition)

Oracle 11gR2 で導入された Edition-Based Redefinition (EBR) を使用すると、エディションを使用して同じスキーマ内に複数のバージョンの PL/SQL オブジェクトを共存させることができます。これにより、ゼロ・ダウンタイムのアップグレードが可能になります。

```sql
-- 特定のセッションで新しいエディションに切り替える
ALTER SESSION SET EDITION = v2_edition;

-- プロシージャの新しいバージョンをコンパイル
-- このバージョンは v2_edition にのみ存在し、元のバージョンは ORA$BASE に残ります
CREATE OR REPLACE PROCEDURE get_order_status(p_id IN NUMBER) RETURN VARCHAR2 IS
BEGIN
  -- 新しい実装ロジック
  RETURN 'STATUS_ENHANCED';
END get_order_status;
/
```

---

## コンパイラ設定のベスト・プラクティス

| 設定項目 | 開発環境 | テスト環境 | 本番環境 |
|---|---|---|---|
| `PLSQL_OPTIMIZE_LEVEL` | 1 (デバッグ用) | 2 | 2 または 3 |
| `PLSQL_CODE_TYPE` | INTERPRETED | INTERPRETED | CPU 集約型のみ NATIVE |
| `PLSQL_WARNINGS` | ENABLE:ALL | ENABLE:ALL | ENABLE:SEVERE のみ |
| `PLSQL_CCFLAGS` | `debug_mode:TRUE` | `debug_mode:FALSE` | `debug_mode:FALSE` |

```sql
-- 推奨：一括コンパイル前にセッション設定を行うテンプレート
ALTER SESSION SET PLSQL_OPTIMIZE_LEVEL  = 2;
ALTER SESSION SET PLSQL_CODE_TYPE       = INTERPRETED;
ALTER SESSION SET PLSQL_WARNINGS        = 'ENABLE:ALL';
ALTER SESSION SET PLSQL_CCFLAGS         = 'debug_mode:FALSE, env_prod:TRUE';

-- 設定変更後に無効なオブジェクトを再コンパイル
BEGIN
  DBMS_UTILITY.COMPILE_SCHEMA(
    schema   => USER,
    compile_all => FALSE  -- FALSE = 無効なオブジェクトのみ
  );
END;
/
```

---

## よくある間違い

| 間違い | 問題点 | 解決策 |
|---|---|---|
| 本番環境で OPTIMIZE_LEVEL=0 を使用する | コード実行が遅くなる | 本番環境では常にレベル 2 以上を使用する |
| すべてに NATIVE コンパイルを適用する | メリットが少なく、コンパイル時間が長くなる | プロファイリングを事前に行い、CPU 負荷の高いコードに限定して適用する |
| オブジェクトごとに PLSQL_CCFLAGS を不整合に設定する | 同じフラグでコンパイルされたコードが異なる動作をする | すべての DDL の前に一貫してフラグを設定するスクリプトを使用する |
| 条件付きコンパイルと通常の IF を混同する | コンパイル時と実行時の混乱 | `$IF` はコンパイル時に処理され、`IF` は実行時に処理されます |
| $END を忘れる | コンパイル・エラーが発生する | `$IF ... $THEN` には必ず `$END` を対にする |

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- ハイブリッドな環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

- **Oracle 10gR2以降**: 条件付きコンパイルが導入されました。
- **Oracle 11gR2以降**: Edition-Based Redefinition (EBR) が導入されました。
- **Oracle 12.2以降**: `$$PLSQL_UNIT_OWNER` や `$$PLSQL_UNIT_TYPE` ディレクティブが追加されました。
- **Oracle 19c**: `DBMS_DB_VERSION.VER_LE_19` が 19c ドキュメントで確認できる最高の VER_LE 定数です。

---

## ソース

- Oracle Database 19c Reference — PLSQL_OPTIMIZE_LEVEL: https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/PLSQL_OPTIMIZE_LEVEL.html
- Oracle Database 19c PL/SQL Language Reference — INLINE Pragma: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/INLINE-pragma.html
- Oracle Database 19c PL/SQL Packages Reference — DBMS_DB_VERSION: https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_DB_VERSION.html
- Oracle Database 19c PL/SQL Language Reference — Conditional Compilation: https://docs.oracle.com/en/database/oracle/oracle-database/19/lnpls/plsql-language-fundamentals.html
