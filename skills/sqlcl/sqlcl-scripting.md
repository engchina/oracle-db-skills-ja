# JavaScriptによるSQLclスクリプティング

## 概要

SQLclにはJavaScriptエンジンが組み込まれており、JavaScriptのロジックとSQLの実行を組み合わせたスクリプトを記述できます。これにより、外部ツールや他のプログラミング言語を使用することなく、高度な自動化ワークフローをSQLcl内だけで構築することが可能になります。クエリ結果の反復処理、データの加工、ファイルの書き出し、Javaライブラリの呼び出し、さらには複雑な多段階のデータベース操作のオーケストレーションを、単一のスクリプトから実行できます。

SQLclで使用されるJavaScriptエンジンは、Javaランタイムに依存します。Nashornエンジンが同梱されているJava 11で実行する場合、ES5構文がサポートされます。Java 17以降で実行する場合（SQLcl 25.2+ の要件）、NashornはJDKに含まれなくなりました。JavaScriptサポートを利用するには、Oracle GraalVM JavaScript Runtimeプラグインをインストールするか、GraalVM上で実行する必要があります。これにより、ECMAScript 2021+ を完全にサポートするGraalJSが提供されます。SQLcl 22.1では、Java 17でJavaScriptを実行するためにGraalVM JavaScriptプラグインが必要でした。

> ⚠️ 未検証：すべてのユーザーにおいてデフォルトのエンジンがGraalJSに切り替わった正確なSQLclリリースのバージョン（22.x）。移行はSQLclのバージョンだけでなく、使用されているJDKにも依存します。`script` コマンドで確認し、お使いの環境でどのエンジンがアクティブかチェックしてください。

---

## scriptコマンド

`script` コマンドは、SQLclでJavaScriptを実行するためのエントリ・ポイントです。2つのモードがあります。

### インライン・スクリプト（対話型）

```sql
script
var result = util.executeQuery("SELECT SYSDATE FROM DUAL");
print(result);
/
```

ブロックは、`/` のみの行で終了します。

### スクリプト・ファイル

```sql
script /path/to/myscript.js
```

またはコマンドラインから：

```shell
sql ユーザー名/パスワード@サービス名 @script.sql
```

ここで、`script.sql` には以下が含まれます。

```sql
script /path/to/myscript.js
exit
```

---

## ctxオブジェクトと暗黙のグローバル変数

SQLclはJavaScriptエンジン内にいくつかのグローバル・オブジェクトを公開しています。

| オブジェクト | 説明 |
|---|---|
| `ctx` | SQLclコンテキスト・オブジェクト（プライマリAPI） |
| `util` | クエリの実行と出力のためのユーティリティ関数 |
| `args` | スクリプトに渡されたコマンドライン引数の配列 |
| `print()` | 標準出力に1行書き出す |
| `sqlcl` | 一部のバージョンにおける `ctx` のエイリアス |

`ctx` オブジェクトがメイン・インターフェースです。主なメソッドは以下の通りです。

```javascript
ctx.write("text\n");                    // 出力への書き出し（改行は自動追加されません）
ctx.getOutputStream();                  // 生の出力ストリームを取得
ctx.getProperty("sqlcl.version");       // SQLclの内部プロパティを読み取る
```

`util` オブジェクトは、最も頻繁に使用される関数を提供します。

```javascript
util.execute("SQL statement");           // DML/DDLを実行、戻り値はなし（void）
util.executeQuery("SELECT ...");         // クエリを実行、ResultSetに似たオブジェクトを返す
util.executeReturnListofList("SELECT ...");  // 配列の配列（2次元配列）を返す
util.executeReturnList("SELECT ...");    // 行オブジェクトの配列を返す
util.print("line\n");                    // 出力にプリント
util.getConnection();                    // 基盤となるJDBC接続を取得
```

---

## JavaScriptからのSQLの実行

### DMLまたはDDLの実行

```javascript
// 行を返さないステートメントを実行
util.execute("CREATE TABLE temp_log (id NUMBER, msg VARCHAR2(200))");
util.execute("INSERT INTO temp_log VALUES (1, 'Test message')");
util.execute("COMMIT");
util.execute("DROP TABLE temp_log PURGE");
```

### クエリの実行と結果の反復処理

最も一般的なパターンは `executeReturnListofList` を使用する方法で、JavaScriptの2次元配列が返されます。

```javascript
var rows = util.executeReturnListofList("SELECT table_name, num_rows FROM user_tables ORDER BY 1");

// 0行目は列ヘッダー
var headers = rows[0];
print(headers.join("\t"));

// 1行目以降がデータ
for (var i = 1; i < rows.length; i++) {
    print(rows[i].join("\t"));
}
```

### 名前付きオブジェクトとしてのクエリ実行

`executeReturnList` は、列名（プロパティ名）を持つオブジェクトの配列を返します。

```javascript
var rows = util.executeReturnList("SELECT employee_id, first_name, last_name, salary FROM employees WHERE department_id = 90");

for (var i = 0; i < rows.length; i++) {
    var row = rows[i];
    print("Employee: " + row.FIRST_NAME + " " + row.LAST_NAME + " | Salary: " + row.SALARY);
}
```

注：返されるオブジェクトのプロパティ名（列名）は常に大文字になります。

### クエリ内でのバインド変数

JDBCスタイルの `?` プレースホルダーと、バインド値の配列を使用します。

```javascript
var deptId = 50;
var rows = util.executeReturnListofList(
    "SELECT employee_id, first_name FROM employees WHERE department_id = ?",
    [deptId]
);
for (var i = 1; i < rows.length; i++) {
    print(rows[i][0] + " - " + rows[i][1]);
}
```

---

## JavaScriptからのJavaクラスへのアクセス

NashornとGraalJSはいずれも、Javaのクラスを直接インスタンス化できます。

```javascript
// Javaクラスのインポート
var File       = Java.type("java.io.File");
var FileWriter = Java.type("java.io.FileWriter");
var ArrayList  = Java.type("java.util.ArrayList");

// ファイルの存在確認
var f = new File("/tmp/myfile.txt");
print("Exists: " + f.exists());

// ファイルを1行ずつ読み込む
var BufferedReader = Java.type("java.io.BufferedReader");
var FileReader     = Java.type("java.io.FileReader");
var br = new BufferedReader(new FileReader("/tmp/input.csv"));
var line;
while ((line = br.readLine()) != null) {
    print(line);
}
br.close();
```

---

## ファイルの読み書き

### ファイルへの書き出し

```javascript
var FileWriter     = Java.type("java.io.FileWriter");
var BufferedWriter = Java.type("java.io.BufferedWriter");

var outputPath = "/tmp/schema_report.txt";
var bw = new BufferedWriter(new FileWriter(outputPath));

var rows = util.executeReturnListofList("SELECT object_type, object_name, status FROM user_objects ORDER BY 1, 2");

for (var i = 1; i < rows.length; i++) {
    bw.write(rows[i].join(",") + "\n");
}

bw.flush();
bw.close();
print("Report written to: " + outputPath);
```

### ファイルの読み込み

```javascript
var Files   = Java.type("java.nio.file.Files");
var Paths   = Java.type("java.nio.file.Paths");

var content = new java.lang.String(Files.readAllBytes(Paths.get("/tmp/input.sql")));
print(content);
```

### ファイルへの追記

```javascript
var FileWriter     = Java.type("java.io.FileWriter");
var BufferedWriter = Java.type("java.io.BufferedWriter");

// 第2引数の `true` は追記モードを有効にします
var bw = new BufferedWriter(new FileWriter("/tmp/audit.log", true));
bw.write(new java.util.Date() + " - Operation completed\n");
bw.close();
```

---

## 実用的な自動化の例

### スキーマ・オブジェクト目録レポート

```javascript
// schema_inventory.js
// すべてのユーザー・オブジェクトのステータスとサイズ情報を含むCSVレポートを生成

var FileWriter     = Java.type("java.io.FileWriter");
var BufferedWriter = Java.type("java.io.BufferedWriter");
var bw = new BufferedWriter(new FileWriter("/tmp/schema_inventory.csv"));

// ヘッダー
bw.write("OBJECT_TYPE,OBJECT_NAME,STATUS,LAST_DDL_TIME\n");

var sql = "SELECT object_type, object_name, status, " +
          "TO_CHAR(last_ddl_time,'YYYY-MM-DD HH24:MI:SS') AS last_ddl " +
          "FROM user_objects " +
          "ORDER BY object_type, object_name";

var rows = util.executeReturnListofList(sql);

for (var i = 1; i < rows.length; i++) {
    var r = rows[i];
    bw.write(r[0] + "," + r[1] + "," + r[2] + "," + (r[3] || "") + "\n");
}

bw.flush();
bw.close();
print("Schema inventory written to /tmp/schema_inventory.csv");
print("Total objects: " + (rows.length - 1));
```

### JSONへのデータ・エクスポート

```javascript
// export_to_json.js
// 表の内容をJSONファイルにエクスポート

var table  = "EMPLOYEES";
var output = "/tmp/employees.json";

var rows = util.executeReturnList("SELECT * FROM " + table);
var FileWriter     = Java.type("java.io.FileWriter");
var BufferedWriter = Java.type("java.io.BufferedWriter");
var bw = new BufferedWriter(new FileWriter(output));

bw.write("[\n");

for (var i = 0; i < rows.length; i++) {
    var row   = rows[i];
    var keys  = Object.keys(row);
    var parts = [];

    for (var j = 0; j < keys.length; j++) {
        var key = keys[j];
        var val = row[key];
        if (val === null || val === undefined) {
            parts.push('"' + key + '": null');
        } else if (typeof val === "number") {
            parts.push('"' + key + '": ' + val);
        } else {
            // 文字列値内のダブルクォーテーションをエスケープ
            var escaped = String(val).replace(/"/g, '\\"');
            parts.push('"' + key + '": "' + escaped + '"');
        }
    }

    var comma = (i < rows.length - 1) ? "," : "";
    bw.write("  {" + parts.join(", ") + "}" + comma + "\n");
}

bw.write("]\n");
bw.flush();
bw.close();
print("Exported " + rows.length + " rows to " + output);
```

### 進捗レポート付きの一括更新

```javascript
// batch_update.js
// 行をバッチ単位で処理し、進捗をレポート

var batchSize = 1000;
var processed = 0;
var errors    = 0;

// 処理対象のIDを取得
var ids = util.executeReturnListofList(
    "SELECT id FROM orders WHERE status = 'PENDING' AND ROWNUM <= 10000"
);

print("Processing " + (ids.length - 1) + " rows...");

for (var i = 1; i < ids.length; i++) {
    try {
        util.execute("UPDATE orders SET status = 'PROCESSING', updated_dt = SYSDATE WHERE id = " + ids[i][0]);
        processed++;

        if (processed % batchSize === 0) {
            util.execute("COMMIT");
            print("Committed batch: " + processed + " rows processed");
        }
    } catch (e) {
        errors++;
        print("ERROR on row " + ids[i][0] + ": " + e.message);
    }
}

// 最終コミット
util.execute("COMMIT");
print("Done. Processed: " + processed + " | Errors: " + errors);
```

### 動的なDDL生成

```javascript
// gen_ddl.js
// 特定の接頭辞を持つすべての表の CREATE TABLE DDL を生成

var prefix = args.length > 0 ? args[0] : "APP_";
var outFile = "/tmp/ddl_" + prefix + ".sql";

var FileWriter     = Java.type("java.io.FileWriter");
var BufferedWriter = Java.type("java.io.BufferedWriter");
var bw = new BufferedWriter(new FileWriter(outFile));

var tables = util.executeReturnListofList(
    "SELECT table_name FROM user_tables WHERE table_name LIKE '" + prefix + "%' ORDER BY 1"
);

for (var i = 1; i < tables.length; i++) {
    var tname = tables[i][0];
    // コンテキストを通じて SQLcl の DDL コマンドを実行
    var ddlRows = util.executeReturnListofList("SELECT DBMS_METADATA.GET_DDL('TABLE', '" + tname + "') AS ddl FROM DUAL");
    if (ddlRows.length > 1) {
        bw.write("-- Table: " + tname + "\n");
        bw.write(ddlRows[1][0] + "\n/\n\n");
    }
}

bw.flush();
bw.close();
print("DDL written to " + outFile);
```

### Eメール・スタイルのHTMLレポート

```javascript
// html_report.js
// データベースのヘルス・メトリクスのHTMLサマリーを生成

var outFile = "/tmp/db_health.html";
var FileWriter     = Java.type("java.io.FileWriter");
var BufferedWriter = Java.type("java.io.BufferedWriter");
var bw = new BufferedWriter(new FileWriter(outFile));

function writeRow(bw, cells) {
    bw.write("  <tr>");
    for (var i = 0; i < cells.length; i++) {
        bw.write("<td>" + (cells[i] !== null ? cells[i] : "NULL") + "</td>");
    }
    bw.write("</tr>\n");
}

bw.write("<html><body><h1>Database Health Report</h1>\n");
bw.write("<p>Generated: " + new java.util.Date() + "</p>\n");

// 無効なオブジェクト
bw.write("<h2>Invalid Objects</h2>\n");
bw.write("<table border='1'><tr><th>Type</th><th>Name</th><th>Status</th></tr>\n");
var invalid = util.executeReturnListofList(
    "SELECT object_type, object_name, status FROM user_objects WHERE status != 'VALID' ORDER BY 1, 2"
);
if (invalid.length <= 1) {
    bw.write("  <tr><td colspan='3'>No invalid objects</td></tr>\n");
} else {
    for (var i = 1; i < invalid.length; i++) {
        writeRow(bw, invalid[i]);
    }
}
bw.write("</table>\n");

bw.write("</body></html>\n");
bw.flush();
bw.close();
print("Report written to " + outFile);
```

---

## コマンドラインからのスクリプトの呼び出し

### SQLラッパー・ファイルを介したスクリプトの実行

`run_script.sql` を作成します：

```sql
script /path/to/myscript.js
exit
```

実行：

```shell
sql ユーザー名/パスワード@サービス名 @run_script.sql
```

### JavaScriptへの引数の渡し方

スクリプト・ファイル名の後に渡された引数は、`args` 配列で利用できます。

```sql
-- run_export.sql
script /path/to/export.js
exit
```

```shell
sql ユーザー名/パスワード@サービス名 @run_export.sql
```

JavaScriptスクリプト自体に引数を渡すには、それらをSQLの置換変数として定義し、JS内で `&1` などを参照します。

```javascript
// JS内で、SQLで設定された置換変数を読み込む
var tableName = "EMPLOYEES"; // ハードコードするか、引数から読み込む
```

よりクリーンな方法は、スクリプトを呼び出す前に `DEFINE` 変数を設定することです。

```sql
-- caller.sql
DEFINE TABLE_NAME = EMPLOYEES
DEFINE OUTPUT_DIR = /tmp
script /path/to/export.js
exit
```

JavaScript内：

```javascript
// SQLcl が JS に渡す前に置換を実行するため、これらの値が渡されます。
// または ctx を使用して定義済み変数を取得します。
```

---

## ベスト・プラクティス

- スクリプト内では必ずファイル・ハンドル（`bw.close()` / `br.close()`）を閉じてください。SQLclのJavaScriptにはJavaのtry-with-resourcesに相当する自動リソース管理がないため、閉じられていないハンドルはリソース・リークの原因になります。
- データとともに列ヘッダーも処理する必要がある場合は `util.executeReturnListofList` を選択してください。名前付きフィールド・アクセスが必要で、ヘッダー行が不要な場合は `util.executeReturnList` を使用してください。
- ビジネス・ロジックはJavaScriptで、データ・アクセスはSQLで記述するように役割を分けてください。文字列連結による複雑なSQL文の構築は避け、SQLインジェクションを防ぐために `?` バインド変数を使用したパラメータ化クエリを使用してください。
- 長時間実行されるスクリプトでは、大規模なアンドゥ・セグメントを回避し、ロック競合を減らすために、バッチ（500〜1000行ごとなど）でコミットしてください。
- バッチ・スクリプト内での個々の行操作はtry/catchブロックで囲み、1つの行の失敗によって全体の実行が中断されないようにしてください。
- 本番のデータセットに対してスクリプトを実行する前に、まずは少数の行数でインタラクティブにテストしてください。開発時は `WHERE ROWNUM <= 10` などの条件を付けてください。
- GraalJS環境（Java 17+ で GraalVM または GraalVM JavaScriptプラグインが必要）では、`let`, `const`, アロー関数, テンプレート・リテラル, `for...of` ループなどの最新のJS機能を使用できます。Nashorn（Java 11のみ）では、ES5構文を使用してください。

---

## よくある間違いと回避策

**間違い：小文字で列名にアクセスしようとする**
`executeReturnList` は、大文字の列名を持つオブジェクトを返します。`row.first_name` は `undefined` になります。必ず `row.FIRST_NAME` を使用してください。

**間違い：インライン・スクリプト・ブロックの後の `/` ターミネータを忘れる**
インラインの `script` ブロックは、独立した行の `/` で閉じる必要があります。これを忘れると、SQLclは無期限に入力を待ち続けます。

**間違い：`util.execute(...)` の戻り値を結果変数としてチェックしようとする**
`util.execute()` は、DML/DDLに対して戻り値なし（void）を返します。クエリ結果データが必要な場合は、`util.executeReturnListofList` または `util.executeReturnList` を使用してください。

**間違い：結果セット内の NULL 値を処理しない**
クエリ結果の NULL 列は、JavaScript では `null` として返されます。null と文字列を連結すると `"null"` という文字列になります。値を使用する前に、必ず `if (val !== null)` でチェックしてください。

**間違い：メモリー内に巨大な結果セットを読み込もうとする**
`executeReturnListofList` および `executeReturnList` は、結果セット全体を JavaScript 配列に読み込みます。数百万行の表の場合、メモリーが不足します。`WHERE ROWNUM <= N` を使用してバッチ処理するか、生の JDBC 接続を使用してカーソル・ベースのアプローチを検討してください。

**間違い：JSエンジンのバージョンに関する誤った想定**
GraalJS向けに `const`, アロー関数, `for...of` を使用して書かれたスクリプトは、Nashorn（Java 11）を使用する SQLcl では失敗します。最大限のポータビリティが必要な場合は ES5 構文を使用するか、Java ランタイムを確認して必要に応じて GraalVM JavaScript プラグインをインストールしてください（Java 17+ で必要）。

---

## 参考資料

- [Oracle oracle-db-tools SQLcl スクリプティング・ガイド (GitHub)](https://github.com/oracle/oracle-db-tools/blob/master/sqlcl/SCRIPTING.md)
- [Java 17 で Oracle SQLcl を使用して JavaScript を実行する方法 — ThatJeffSmith](https://www.thatjeffsmith.com/archive/2022/04/running-javascript-in-oracle-sqlcl-22-1/)
- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLcl リリース・ノート 25.2.1 — Java 17/21の要件](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.1.html)
