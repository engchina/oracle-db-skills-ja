# SQLclの基本

## 概要

SQLcl（SQL Command Line）は、SQL*Plusの現代的な代替ツールとしてOracleが提供しています。Oracle Databaseのインストールに含まれるほか、スタンドアロンのダウンロードも可能な、無料のJavaベースのコマンドライン・インターフェースです。SQLclは、タブ補完、コマンド履歴、インライン編集、組み込みのLiquibaseサポート、JavaScriptスクリプト・エンジン、豊富な出力フォーマット・オプションなど、SQL*Plusと比較して大幅な機能強化を実現しています。

SQLclは単一のZIPファイルとして配布されており（インストーラーは不要）、JDK 17以降（JDK 21を推奨）がインストールされた任意のプラットフォームで動作します。また、macOSではHomebrew経由でも利用可能です。実行ファイル名は、SQL*Plusからの移行を容易にするために `sqlcl` ではなく `sql` となっています。

---

## インストール

### macOS（Homebrew経由 - 推奨）

```shell
brew install sqlcl
```

インストール後、コマンドは `sql` として使用可能になります。Homebrewは `brew upgrade sqlcl` を介してアップデートを管理します。

### 手動インストール（全プラットフォーム共通）

1. [Oracle Downloads](https://www.oracle.com/tools/downloads/sqlcl-downloads.html) から最新のSQLcl ZIPをダウンロードします。Oracleアカウントは不要です。
2. 任意のディレクトリに解凍します。
   ```shell
   unzip sqlcl-<version>.zip -d /opt/sqlcl
   ```
3. `bin` ディレクトリをPATHに追加します。
   ```shell
   export PATH=/opt/sqlcl/bin:$PATH
   ```
4. インストールを確認します。
   ```shell
   sql -V
   ```

### Javaの確認

SQLcl 25.2以降では、JDK 17またはJDK 21が必要です。Java 11のサポートはSQLcl 25.2で終了しました。複数のJDKがインストールされている場合は、正しいバージョンがPATHに含まれていることを確認してください。

```shell
java -version
```

> ⚠️ 未検証：現在のSQLclリリースの中にJVMを同梱しているものがあるかどうか。システムJDKが不要であると仮定する前に、公式のリリース・ノートを確認してください。

---

## Oracle Databaseへの接続

### 基本的なユーザー名/パスワード

```shell
sql ユーザー名/パスワード@ホスト名:ポート番号/サービス名
```

例：

```shell
sql hr/hr@localhost:1521/FREEPDB1
```

### Easy Connect構文（EZConnect）

Easy Connectは最もポータブルな接続方法であり、`tnsnames.ora` ファイルを必要としません。

```shell
sql ユーザー名/パスワード@//ホスト名:ポート番号/サービス名
```

追加のオプションが必要な場合は、Easy Connect Plus構文（Oracle 19c以降）も使用できます。

```shell
sql hr/hr@"(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=myhost)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=MYPDB)))"
```

### TNSエイリアス

`$TNS_ADMIN` または `$ORACLE_HOME/network/admin/tnsnames.ora` が構成されている場合：

```shell
sql ユーザー名/パスワード@MY_TNS_ALIAS
```

TNS管理ディレクトリを明示的に設定する場合：

```shell
export TNS_ADMIN=/path/to/wallet_or_tns
sql hr/hr@MY_SERVICE
```

### Oracle Cloud（Autonomous Database）ウォレット

Oracle CloudコンソールからウォレットZIPをダウンロードし、ローカル・ディレクトリに解凍します。

```shell
unzip Wallet_MyDB.zip -d /path/to/wallet
```

ウォレットを使用して接続します。

```shell
sql ユーザー名/パスワード@"(DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=adb.us-ashburn-1.oraclecloud.com)(PORT=1522))(CONNECT_DATA=(SERVICE_NAME=myadb_high.adb.oraclecloud.com))(SECURITY=(MY_WALLET_DIRECTORY=/path/to/wallet)))"
```

または、`TNS_ADMIN` をウォレット・ディレクトリに設定し、そこに含まれる定義済みのTNSエイリアスを使用します。

```shell
export TNS_ADMIN=/path/to/wallet
sql admin/MyPassword123@myadb_high
```

### パスワードの入力プロンプト（セキュア）

パスワードを省略すると入力が求められます（シェル履歴にパスワードが残るのを防ぎます）。

```shell
sql hr@localhost:1521/FREEPDB1
```

SQLclは `Password? (**********?)` とプロンプトを表示します。

### SYSDBA / SYSOPER 接続

```shell
sql sys/password@localhost:1521/ORCL as sysdba
sql / as sysdba
```

### SQLcl内からの接続

`CONNECT` コマンドを使用して、再起動せずに接続を切り替えます。

```sql
CONNECT hr/hr@localhost:1521/FREEPDB1
CONNECT admin@myadb_high
```

---

## SQL*Plusとの主な違い

| 機能 | SQL*Plus | SQLcl |
|---|---|---|
| タブ補完 | なし | あり（表、列、キーワード） |
| コマンド履歴 | なし | あり（上下矢印、HISTORYコマンド） |
| インライン編集 | なし | あり（カーソル・キー、readlineスタイル） |
| JavaScriptスクリプト | なし | あり（組み込みエンジン） |
| Liquibase | なし | 組み込み |
| 出力フォーマット | 限定的 | CSV, JSON, XML, INSERTなど |
| DDLコマンド | なし | あり（`DDL 表名`） |
| LOADコマンド | なし | あり（CSV/JSONの取り込み） |
| 構文ハイライト | なし | あり（ANSICONSOLEフォーマット） |
| ファイル・サイズ | 大きい（Oracle Clientが必要） | 小さい（スタンドアロンJAR） |
| 接続 | Oracle Clientライブラリが必要 | 純粋なJava、クライアント不要 |

SQLclは、同じSQL*Plusスクリプト（`@` および `@@` ディレクティブを含む `.sql` ファイル）を最小限の互換性の問題で受け入れます。ほとんどのSQL*Plusコマンド（`SET`, `COLUMN`, `SPOOL` など）は変更なしで動作します。

---

## 基本コマンド

### HELP

利用可能なコマンドとその構文を表示します。

```sql
HELP
HELP INDEX
HELP SET
HELP SPOOL
HELP CONNECT
```

### SET

SQLclの動作を制御します。一般的な設定：

```sql
-- 出力の詳細度を制御
SET ECHO ON             -- 実行前に各コマンドを表示
SET FEEDBACK ON         -- クエリ後に件数を表示
SET HEADING ON          -- 列ヘッダーを表示
SET TIMING ON           -- 実行時間を表示
SET SERVEROUTPUT ON     -- DBMS_OUTPUTを有効化
SET SERVEROUTPUT ON SIZE UNLIMITED

-- 出力フォーマット
SET LINESIZE 200        -- 出力行ごとの文字数
SET PAGESIZE 50         -- 1ページあたりの行数（0 = 改ページなし）
SET WRAP ON             -- 長い行を折り返す（デフォルト）
SET TRIM ON             -- 行末の空白を削除
SET NULL "[NULL]"       -- NULL値に表示するテキスト

-- 数値フォーマット
SET NUMFORMAT 999,999,999
SET NUMWIDTH 15

-- 出力フォーマット（SQLcl固有）
SET SQLFORMAT CSV
SET SQLFORMAT JSON
SET SQLFORMAT DEFAULT
```

### SHOW

現在の設定を表示します。

```sql
SHOW ALL                -- すべてのSET値を表示
SHOW SQLFORMAT          -- 現在の出力フォーマット
SHOW USER               -- 現在のユーザー名
SHOW CON_NAME           -- 現在のコンテナ（CDB/PDB）
SHOW PDBS               -- PDBのリストを表示（CDB$ROOT接続が必要）
SHOW ERRORS             -- 直前のオブジェクトのコンパイル・エラーを表示
SHOW PARAMETERS nls     -- NLSデータベース・パラメータを表示
SHOW SGA                -- SGAメモリーの内訳を表示
```

### HISTORY

SQLclはセッションをまたいで永続的なコマンド履歴を保持します（`~/.sqlcl/history.log` に保存）：

```sql
HISTORY                 -- 最近のコマンド履歴を表示
HISTORY 20              -- 直近の20個のコマンドを表示
HISTORY FULL            -- タイムスタンプを含む完全な履歴を表示
HISTORY USAGE           -- 最も頻繁に使用されるコマンドを表示
HISTORY SCRIPT          -- 履歴をスクリプトとして保存
HISTORY CLEAR           -- 履歴をクリア
```

履歴番号を指定してコマンドを再実行します：

```sql
HISTORY 5               -- 5番目のコマンドを再実行
```

### ALIAS

頻繁に使用するコマンドのショートカットを作成します：

```sql
-- エイリアスの作成
ALIAS tables=SELECT table_name, num_rows FROM user_tables ORDER BY 1;
ALIAS cols=SELECT column_name, data_type, nullable FROM user_tab_columns WHERE table_name = UPPER('&1');

-- エイリアスの使用
tables
cols employees

-- すべてのエイリアスをリスト表示
ALIAS LIST

-- エイリアスの削除
ALIAS DROP tables

-- エイリアスを永続的に保存（~/.sqlcl/aliases.xml に保存）
ALIAS SAVE
ALIAS LOAD
```

### スクリプトの実行

```sql
-- スクリプト・ファイルの実行
@/path/to/script.sql
@@relative/to/current.sql  -- 現在のスクリプトからの相対パス

-- URLからの実行
@https://example.com/script.sql

-- 引数の渡し方
@script.sql arg1 arg2
-- 引数は &1, &2 などとしてアクセス可能
```

---

## コマンドの呼び出しと編集

SQLclはreadlineスタイルの編集のためにJLineを使用します：

| キー | 動作 |
|---|---|
| 上/下矢印 | 履歴の移動 |
| Ctrl+R | 履歴の逆方向検索 |
| Ctrl+A | 行の先頭に移動 |
| Ctrl+E | 行の末尾に移動 |
| Ctrl+K | 行の末尾まで削除 |
| Ctrl+U | 行全体を削除 |
| Ctrl+W | 前の単語を削除 |
| Tab | 自動補完 |
| Alt+. | 直前のコマンドの最後の引数を挿入 |

### 複数行の編集

SQLclは複数行のSQL編集をサポートしています。SQL文を複数行にわたって記述するにはEnterキーを押します。プロンプトが `SQL>` から行番号に変わります。実行するには空行で `/` を入力するか、セミコロンを入力します：

```
SQL> SELECT employee_id,
  2         first_name,
  3         last_name
  4    FROM employees
  5   WHERE department_id = 90
  6  /
```

`/` コマンドはSQLバッファを再実行します。現在のバッファを表示するには `L`（LIST）を使用します：

```sql
LIST            -- SQLバッファを表示
L 3             -- 3行目を表示
C /old/new/     -- バッファ内のテキストを変更（SQL*Plusスタイル）
```

---

## タブ補完

タブ補完は以下の項目で機能します：

- SQLキーワード（`SELECT`, `FROM`, `WHERE` など）
- 表名およびビュー名
- 列名（`SELECT`, `WHERE` などの後のコンテキストを認識）
- SQLclコマンド（`HISTORY`, `ALIAS`, `DDL` など）
- ファイル・パス（`@` および `SPOOL` 用）
- バインド変数名

Tabキーを1回押すと補完され、一致するものが複数ある場合に2回押すとすべての候補が表示されます。

---

## 起動および構成ファイル

SQLclは、SQL*Plusと同様に起動時に `login.sql` を読み込みます：

- 最初にカレント・ディレクトリを検索
- 次に `$SQLPATH`
- 次に `$HOME/.sqlcl/login.sql`

`login.sql` の例：

```sql
-- ~/.sqlcl/login.sql
SET LINESIZE 200
SET PAGESIZE 100
SET TIMING ON
SET SERVEROUTPUT ON SIZE UNLIMITED
SET SQLFORMAT ANSICONSOLE

ALIAS tables=SELECT table_name, num_rows, last_analyzed FROM user_tables ORDER BY 1;
ALIAS inval=SELECT object_type, object_name, status FROM user_objects WHERE status != 'VALID' ORDER BY 1, 2;
```

---

## 便利なランタイム・コマンド

```sql
-- SQLclの終了
EXIT
QUIT
EXIT 1          -- 特定のリターン・コードで終了

-- 画面のクリア
CLEAR SCREEN
CL SCR

-- 現在の日時を表示
SELECT SYSDATE FROM DUAL;

-- クエリの時間を計測
SET TIMING ON
SELECT COUNT(*) FROM big_table;
-- Elapsed: 00:00:01.234

-- 現在のデータベース情報を表示
SELECT * FROM v$instance;
SELECT name, db_unique_name, open_mode FROM v$database;

-- オブジェクトの定義を表示
DESC employees
DESC hr.employees
DESC my_package
```

---

## ベスト・プラクティス

- 共有環境やログが記録される環境では、資格情報がシェルの履歴やプロセス・リストに表示されるのを防ぐため、パスワード・プロンプトが表示される形式で `CONNECT` コマンドを使用してください（コマンドラインからパスワードを省略します）。
- スクリプトのポータビリティを維持するため、接続文字列にウォレットのパスを埋め込むのではなく、シェル・プロファイルで `TNS_ADMIN` を設定してください。
- 一般的な `SET` コマンドや `ALIAS` 定義は `~/.sqlcl/login.sql` に保存し、すべてのセッションが一貫した構成で開始されるようにしてください。
- インタラクティブな操作中は `SET FEEDBACK ON` と `SET TIMING ON` を使用し、出力データのみが重要なスクリプトではこれらを `OFF` にしてください。
- `DBMS_OUTPUT` が切り捨てられないように、固定サイズではなく `SET SERVEROUTPUT ON SIZE UNLIMITED` を優先してください。
- バッチ処理の前後で `SPOOL /path/to/output.log` と `SPOOL OFF` を使用し、後で確認できるようにすべての出力をキャプチャしてください。
- Autonomous Databaseに接続する場合、ウォレット・パスを `TNS_ADMIN` に保存し、`tnsnames.ora` と `sqlnet.ora`（実際のウォレット・ファイル以外）をバージョン管理にコミットして、接続文字列を再現可能にしてください。

---

## よくある間違いと回避策

**間違い：プロセス・リストやシェル履歴でパスワードが見えてしまう**
接続文字列からパスワードを省略してください。SQLclが安全にプロンプトを表示します。あるいは、ウォレットを使用するか、スクリプトの開始時に `DEFINE` コマンドを使用して入力を促します。

**間違い：SQL*Plusとの互換性によるスクリプトの失敗**
ほとんどのSQL*Plusスクリプトは、SQLclでそのまま動作します。問題が発生した場合は、複雑な書式設定を伴う `COLUMN ... NOPRINT` や非常に古い `SET` オプションの使用を確認してください。SQLclは、SQL*Plusディレクティブの大部分をサポートしています。

**間違い：`SET PAGESIZE 0` によって列ヘッダーが表示されなくなる**
`DEFAULT` フォーマットでは、`PAGESIZE 0` は改ページを抑制しますが、列ヘッダーも抑制します。改ページなしでヘッダーを表示したい場合は、`SET HEADING ON` とともに `SET PAGESIZE 50000` を使用してください。

**間違い：データ・エクスポート・スクリプトで `SET FEEDBACK OFF` を忘れる**
`25 rows selected.` のような行数メッセージが出力ファイルに含まれてしまいます。後続の処理で使用するためにデータをスプールする場合は、必ず `SET FEEDBACK OFF` と `SET HEADING OFF` を設定してください。

**間違い：タブ補完が機能しない**
タブ補完を使用するには、ターミナルが制御文字を渡すモードである必要があります。パイプや非対話型のシェル置換ではなく、対話型のターミナルでSQLclを実行していることを確認してください。

**間違い：スクリプト内でSQL*Plusの実行パスを使用している**
SQLclに移行した後は、スクリプトやエイリアスを `sqlplus` ではなく `sql` を使用するように更新してください。標準的なSQLやPL/SQLの実行における動作はほぼ同じです。

---


## Oracleバージョンの注意点 (19c対26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明示的に指定されていない限り、Oracle Database 19cに有効です。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応の機能として扱う必要があります。複数のバージョンが混在する環境では、19c互換の代替案を保持してください。
- 両方のバージョンをサポートする環境では、リリース・アップデートによってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストしてください。

## 参考資料

- [Oracle SQLcl 製品ページ](https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/)
- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLclの開始と終了 — 起動フラグおよびバージョン・フラグ](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/startup-sqlcl-settings.html)
- [SQLcl リリース・ノート 25.2](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.html)
- [SQLcl リリース・ノート 25.2.1 — Java 17/21の要件確認済み](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.1.html)
