# SQLclによるDDL生成

## 概要

SQLclは、既存のデータベース・オブジェクトの `CREATE` ステートメントを迅速に抽出するための強力な `DDL` コマンドを提供します。このコマンドは、裏側で `DBMS_METADATA` パッケージを使用していますが、複雑なPL/SQLブロックを記述することなく、コマンドラインから直接、きめ細かな制御と読みやすいフォーマットでDDLを生成するための簡素化されたインターフェースを提供します。これは、ベースラインのキャプチャ、異なる環境（開発からテストなど）間でのオブジェクトの複製、およびスキーマ定義のバージョン管理に不可欠です。

このガイドでは以下を扱います：
- `DDL` コマンドの基本構文
- `SET DDL` による出力フォーマットの制御
- 特定のオブジェクト・タイプ（表、索引、パッケージなど）の生成
- ストレージ、表領域、セグメント属性の除外
- バージョン管理のためのデプロイメント・スクリプトの作成

---

## DDLコマンド

`DDL` コマンドは、現在のスキーマまたは指定されたスキーマ内の任意のオブジェクトに対して機能します。

### 基本構文

```sql
DDL [object_name] [object_type]
```

例：

```sql
DDL employees
DDL hr.employees
DDL emp_details_view VIEW
DDL add_job_history PROCEDURE
```

オブジェクト名のみを指定すると（タイプを省略すると）、SQLclは名前の一致を検索し、存在する場合はDDLを返します。複数のタイプ（同じ名前の表と索引など）が存在する場合は、タイプを明示的に指定する必要があります。

### 出力のキャプチャ

DDL出力をファイルに保存するには、`SPOOL` を使用します。

```sql
SPOOL employees_ddl.sql
DDL employees
SPOOL OFF
```

---

## DDL出力の構成 (SET DDL)

`DDL` コマンドを実行する前に、`SET DDL` オプションを使用して、生成されるSQLから環境固有の属性（表領域、ストレージ、共有など）をストリップ（除去）できます。これは、スクリプトを異なるデータベース（開発用から本番用など）間でポータブルにするために重要です。

### 一般的な設定

| コマンド | 説明 |
|---|---|
| `SET DDL STORAGE OFF` | `INITIAL`, `NEXT`, `PCTINCREASE` などのストレージ句を除外する |
| `SET DDL TABLESPACE OFF` | `TABLESPACE "DATA"` 句を除外する（デフォルトの表領域を使用する場合に有用） |
| `SET DDL SEGMENT_ATTRIBUTES OFF` | セグメント関連の全属性を除外する |
| `SET DDL CONSTRAINTS_AS_ALTER ON` | インラインではなく、`ALTER TABLE` として制約を生成する |
| `SET DDL PRETTY ON` | インデントと改行を使用してフォーマットを整える |
| `SET DDL SQLTERMINATOR ON` | 各ステートメントの末尾にセミコロン（;）を追加する |

### 推奨されるバージョン管理用設定

きれいなポータブルな定義を取得するには、これらを組み合わせます：

```sql
SET DDL PRETTY ON
SET DDL SQLTERMINATOR ON
SET DDL STORAGE OFF
SET DDL TABLESPACE OFF
SET DDL SEGMENT_ATTRIBUTES OFF
SET DDL SPECIFICATION ON
SET DDL BODY ON
```

---

## さまざまなオブジェクト・タイプのDDL生成

### 表と索引

表を生成すると、通常はその索引と制約も含まれます。

```sql
DDL employees TABLE
```

索引のみを個別に抽出する場合：

```sql
DDL emp_name_ix INDEX
```

### ビュー、マテリアライズド・ビュー

```sql
DDL emp_details_view VIEW
DDL sales_summary_mv MATERIALIZED_VIEW
```

### PL/SQL（パッケージ、プロシージャ、ファンクション）

```sql
-- パッケージ全体（仕様部と本体の両方）
DDL my_api_pkg PACKAGE

-- 仕様部のみ
DDL my_api_pkg PACKAGE_SPEC

-- 本体のみ
DDL my_api_pkg PACKAGE_BODY
```

### シーケンス、シノニム、トリガー

```sql
DDL emp_seq SEQUENCE
DDL employees SYNONYM
DDL secure_emp_trg TRIGGER
```

---

## バージョン管理のベスト・プラクティス

データベース・スキーマをGitリポジトリなどで管理する場合、以下のワークフローが推奨されます。

1. **環境固有の情報を除去する**：`SET DDL STORAGE OFF` および `SET DDL TABLESPACE OFF` を使用して、開発環境の物理構成が本番環境の構成スクリプトに「ハードコード」されないようにします。
2. **オブジェクトごとにファイルを分ける**：1つの巨大なファイルにすべてを抽出するのではなく、オブジェクトごとに個別の `.sql` ファイルを作成してください。
   - `tables/employees.sql`
   - `views/emp_v.sql`
   - `packages/hr_api.pks` (仕様部)
   - `packages/hr_api.pkb` (本体)
3. **きれいな命名規則を使用する**：ファイル名にはオブジェクト名を含めます。
4. **生成の自動化**：多数のオブジェクトがある場合は、JavaScriptスクリプトを使用して各オブジェクトをループし、`SPOOL` を介して個別のファイルに保存します。

---

## DBMS_METADATAとの違い

`DDL` コマンドは `DBMS_METADATA` のラッパーですが、以下のような主な違いがあります。

| 機能 | SQLcl `DDL` | `DBMS_METADATA.GET_DDL` |
|---|---|---|
| 使いやすさ | 非常に高い（1ワード） | 低い（複雑なPL/SQLクエリが必要） |
| フォーマット | 常にきれい (PRETTY ON) | デフォルトではXMLまたは読みにくいSQL |
| 構成 | シンプルな `SET` コマンド | `SET_TRANSFORM_PARAM` の呼び出しが必要 |
| セッションの状態 | SQLclセッションに保持される | 関数呼び出しごとに設定が必要 |
| 依存関係 | 通常、関連オブジェクトを自動取得 | 各タイプを個別にフェッチする必要がある |

---

## 自動化とスクリプト

SQLclのJavaScriptエンジンを使用して、DDLの抽出を自動化できます。

```javascript
// extract_all_tables.js
var tables = util.executeReturnListofList("SELECT table_name FROM user_tables");

for (var i = 1; i < tables.length; i++) {
    var tname = tables[i][0];
    print("Extracting " + tname + "...");
    sqlcl.setStmt("SPOOL tables/" + tname.toLowerCase() + ".sql\n" +
                  "DDL " + tname + "\n" +
                  "SPOOL OFF");
    sqlcl.run();
}
```

実行方法：
```sql
sql hr/hr@db @extract_all_tables.js
```

---

## ベスト・プラクティス

- Gitなどにコミットする前に、`SET DDL` 設定を使用して、`STORAGE`, `TABLESPACE`, `SEGMENT_ATTRIBUTES` を必ず `OFF` にしてください。これにより、環境固有の物理構成によって差分が発生するのを防げます。
- 複数のスキーマを扱う場合は、曖昧さを避けるために常に `DDL [schema].[object]` 形式か、`DDL [object] [type]` を使用してください。
- 異なる環境（Oracle 19cから23cなど）間でDDLを移行する場合、新しい予約語や非推奨の属性（古い `PARALLEL` 句など）が含まれていないか確認してください。
- SQLclでDDLを生成する際は、`SET DDL SQLTERMINATOR ON` を忘れないでください。これを忘れると、生成されたスクリプトを実行する際に各ステートメントが正しく区切られず、エラーになります。
- PL/SQL（パッケージやトリガーなど）の場合、出力にプロンプトを正常に終了させるための `/`（スラッシュ）が含まれていることを確認してください。

---

## よくある間違いと回避策

**間違い：生成されたDDLにターゲット環境には存在しない表領域が含まれている**
`SET DDL TABLESPACE OFF` を設定せずにDDLを生成すると、ソース環境の `TABLESPACE "DEVELOPMENT_DATA"` 句が含まれてしまいます。これがターゲット環境（本番など）に存在しない場合、スクリプトの実行は失敗します。

**間違い：索引やトリガーが2回カウントされてしまう**
`DDL table_name` を実行すると、通常はその表に関連する索引も含まれます。索引に対しても個別に `DDL index_name` を実行して個別のファイルに保存すると、デプロイ時にオブジェクトが重複する可能性があります。表の定義にインラインで含めるか、個別に抽出し、表の出力からは除外（`SET DDL INDEXES OFF`）するか、どちらか一方に統一してください。

**間違い：DBMS_OUTPUTの有効化を忘れてDDLが空になる**
`DDL` コマンドの結果は標準出力に直接表示されますが、それを呼び出している内部のPL/SQL（`DBMS_METADATA` など）が失敗した場合、`SERVEROUTPUT` がオフだと詳細なエラー・メッセージが見えなくなることがあります。問題が発生した場合は `SET SERVEROUTPUT ON` を試してください。

**間違い：大規模なスキーマで 1 つのファイルにスプールし、マージ競合が発生する**
すべてのテーブルとビューを1つの `schema.sql` ファイルにスプールすると、複数人で開発している場合にバージョン管理上のマージ競合が多発します。必ずオブジェクトごとにファイルを分ける自動化スクリプトを使用してください。

**間違い：SQLTERMINATORをオフにして、最後に（;）がない定義を生成してしまう**
デフォルトではSQL終端文字が含まれない場合があります。`SET DDL SQLTERMINATOR ON` を忘れずに設定してください。

---

## 参考資料

- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLclによるきれいなDDLの生成 — そのJeffSmith](https://www.thatjeffsmith.com/archive/2016/11/generating-ddl-extracting-your-database-objects-with-sqlcl/)
- [SQLclのSET DDLコマンド・リファレンス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/set-ddl.html)
- [SQLcl リリース・ノート 25.2](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.html)
- [Oracle SQLcl リリース・インデックス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/index.html)
