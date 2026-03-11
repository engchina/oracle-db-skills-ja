# SQLclの出力フォーマット

## 概要

SQLclは、データの表示とエクスポートを容易にするために、従来のSQL*Plusよりも大幅に強化された出力フォーマット機能を提供します。コマンドラインでのインタラクティブな使用に適した構文ハイライト付きのグリッド表示（`ANSICONSOLE`）から、ダウンストリーム（後続処理）のシステムで直接使用できる `CSV`, `JSON`, `XML`, `INSERT` ステートメントなどの構造化データ出力まで、幅広いフォーマットをサポートしています。

このガイドでは以下を扱います：
- `SET SQLFORMAT` によるグローバルな出力形式の切り替え
- 各フォーマット（CSV, JSON, INSERTなど）の詳細
- `COLUMN` コマンドによる列ごとの微調整
- データのエクスポートとファイルへのスプーリング
- レポート作成とAPIペイロードのベスト・プラクティス

---

## SET SQLFORMAT コマンド

`SET SQLFORMAT` は、SQLclの出力エンジンの動作を制御する最も重要なコマンドです。

### 基本構文

```sql
SET SQLFORMAT [format_name]
```

現在のフォーマットを確認する場合：

```sql
SHOW SQLFORMAT
```

デフォルト（SQL*Plus互換）に戻す場合：

```sql
SET SQLFORMAT DEFAULT
```

---

## 利用可能なフォーマット

### ANSICONSOLE（インタラクティブ用途に最適）

端末の幅に合わせて列幅を自動調整し、ヘッダーに太字や色（構文ハイライト）を適用した読みやすいグリッド形式で出力します。

```sql
SET SQLFORMAT ANSICONSOLE
SELECT * FROM user_tables;
```

### CSV（データのエクスポート用）

カンマ区切り値として出力します。ダブルクォーテーションの囲みやエスケープが自動的に処理されます。

```sql
SET SQLFORMAT CSV
SELECT employee_id, last_name, hire_date FROM employees;
-- 100,"King",17-JUN-03
```

### JSON（APIおよびWeb用途）

クエリの結果をJSONオブジェクトの配列として出力します。

```sql
SET SQLFORMAT JSON
SELECT employee_id, last_name FROM employees WHERE department_id = 90;
-- {"results":[{"employee_id":100,"last_name":"King"}]}
```

### INSERT（データの移行用）

各行に対する `INSERT` ステートメントを生成します。

```sql
SET SQLFORMAT INSERT
SELECT * FROM regions;
-- INSERT INTO "REGIONS" (REGION_ID, REGION_NAME) VALUES (1, 'Europe');
```

### XML

階層化されたXML形式で出力します。

```sql
SET SQLFORMAT XML
SELECT employee_id, last_name FROM employees WHERE ROWNUM = 1;
```

### LOADER（SQL*Loader互換）

SQL*Loaderのデリミタ付き形式（通常はパイプ `|` 区切り）で出力します。

```sql
SET SQLFORMAT LOADER
SELECT * FROM employees;
```

### FIXED

列幅を固定した形式で出力します。

```sql
SET SQLFORMAT FIXED
SELECT * FROM employees;
```

### DELIMITED（カスタム区切り）

任意の文字で区切られた形式を出力します。

```sql
SET SQLFORMAT DELIMITED |
SELECT * FROM employees;
```

---

## COLUMN コマンド

`COLUMN` コマンドを使用すると、個々の列の表示名、フォーマット、および可視性を制御できます。

### 基本的な書式設定

```sql
-- 列名の別名（見出し）を設定
COLUMN salary HEADING "Monthly Salary"

-- 数値のフォーマットを設定（カンマ、ドル記号、小数点）
COLUMN salary FORMAT $99,999.99

-- 列の幅を15文字に固定
COLUMN last_name FORMAT A15

-- 長いテキストを折り返す
COLUMN notes FORMAT A30 WORD_WRAPPED
```

### 表示の制御

```sql
-- 特定の列を非表示にする
COLUMN commission_pct NOPRINT

-- 設定をクリアする
COLUMN salary CLEAR
```

---

## データの出力とスプーリング

データをファイルにエクスポートする際は、不要なメッセージを抑制するために `SET` コマンドを組み合わせて使用します。

### CSVエクスポートの例

```sql
SET SQLFORMAT CSV
SET FEEDBACK OFF
SET MARKUP CSV ON QUOTE ON  -- SQLcl 21c以降の追加オプション
SPOOL /tmp/employees_export.csv

SELECT * FROM employees;

SPOOL OFF
SET SQLFORMAT DEFAULT
```

### JSONエクスポートの例

```sql
SET SQLFORMAT JSON
SET FEEDBACK OFF
SPOOL /tmp/employees.json

SELECT * FROM employees;

SPOOL OFF
SET SQLFORMAT DEFAULT
```

---

## レポートとAPIのベスト・プラクティス

### 読みやすいレポートの作成

ターミナルでデータを閲覧する場合は、常に `ANSICONSOLE` を最初に使用してください。

```sql
SET SQLFORMAT ANSICONSOLE
SET PAGESIZE 50
SET FEEDBACK ON
SELECT department_name, manager_id FROM departments ORDER BY 1;
```

### 定型レポートの自動化

一貫した出力を得るために、`COLUMN` 定義と `SET` オプションをスクリプトに含めます。

```sql
-- daily_report.sql
SET PAGESIZE 100
SET LINESIZE 200
COLUMN employee_id HEADING "ID" FORMAT 999
COLUMN full_name HEADING "Name" FORMAT A30

SELECT employee_id, first_name || ' ' || last_name AS full_name 
FROM employees;
```

---

## ベスト・プラクティス

- ターミナルで作業する際は、最初に `SET SQLFORMAT ANSICONSOLE` を実行してください。これにより、`SET LINESIZE` や `SET PAGESIZE` を手動で細かく調整することなく、データの読みやすさが劇的に向上します。
- プログラムで処理するデータをエクスポートする場合は（CSVやJSONなど）、行数メッセージ（例：`10 rows selected.`）がファイルに混入してパースエラーが発生するのを防ぐため、必ず `SET FEEDBACK OFF` を設定してください。
- データの移行（ある表から別の表へ）を行う場合は、`SET SQLFORMAT INSERT` を使用してください。これにより生成された `INSERT` ステートメントは、ターゲット・データベースで直接実行可能な有効なSQLになります。
- 数値データの精度を維持するために、`COLUMN ... FORMAT` を使用して、必要な小数点以下の桁数が正しく表示・出力されるようにしてください。
- 日付列の出力が意図した形式（例：`YYYY-MM-DD`）になるように、エクスポート前に `ALTER SESSION SET NLS_DATE_FORMAT = '...'` を実行するか、クエリ内で `TO_CHAR` を使用してください。

---

## よくある間違いと回避策

**間違い：エクスポート・ファイルの中に「10 rows selected.」というメッセージが混じってしまう**
`SPOOL` を使用する前に `SET FEEDBACK OFF` を実行し忘れています。エクスポート用のスクリプトの先頭には、必ずこれを追加してください。

**間違い：CSV出力で長い文字列が切り捨てられてしまう**
`SET SQLFORMAT CSV` は通常、切り捨てを行いませんが、`SET LINESIZE` が非常に小さく設定されている場合に影響を受けることがあります。十分な `LINESIZE` を設定するか、`ANSICONSOLE` を避けてデータの出力に徹してください。

**間違い：JSON出力のネストされた構造を期待してしまう**
SQLclの `SET SQLFORMAT JSON` は、フラットな結果セットの各行をオブジェクトとする単一の配列を返します。オブジェクト・リレーショナルなネストされた構造（例：Orderの中に複数のLineItems）が必要な場合は、標準のSQL/JSON関数（`JSON_OBJECT`, `JSON_ARRAYAGG`）をクエリ内で直接使用してください。

**間違い：ANSICONSOLEを使用中に列ヘッダーが表示されない**
`SET PAGESIZE 0` が設定されていると、`ANSICONSOLE` でもヘッダーが表示されません。ページ区切りをなくしつつヘッダーを表示したい場合は、`SET PAGESIZE 50000`（非常に大きな値）を設定してください。

**間違い：SPOOLコマンドの後に SPOOL OFF を忘れてファイルが空のままになる**
OSによっては、`SPOOL OFF` が実行されるまでバッファがフラッシュ（書き出し）されないため、ファイルの内容が空に見えたり不完全だったりすることがあります。スクリプトの最後には必ず `SPOOL OFF` を含めてください。

---

## 参考資料

- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLclの出力オプションのリファレンス — そのJeffSmith](https://www.thatjeffsmith.com/archive/2017/01/formatting-query-results-in-sqlcl/)
- [SQLclのSET SQLFORMATコマンド・リファレンス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/set-sqlformat.html)
- [SQLcl リリース・ノート 25.2](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.html)
- [Oracle SQLcl リリース・インデックス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/index.html)
