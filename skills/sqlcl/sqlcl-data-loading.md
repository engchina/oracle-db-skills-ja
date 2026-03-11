# SQLclによるデータ・ローディング

## 概要

SQLclは、外部ツール（SQL*Loaderなど）や複雑な外部表の構成を必要とせずに、CSVおよびJSONデータをOracle表に直接インポートするための強力な組み込み `LOAD` コマンドを提供します。`LOAD` コマンドはクライアント・ベースであり、SQLcl内で実行されます。つまり、ネットワーク経由でデータをストリーミングし、内部でJDBCバッチ・インサートを使用するため、アドホックなインポート、CI/CD環境でのデータ・シーディング、および開発者のワークフローに非常に便利です。

このガイドでは以下を扱います：
- `LOAD` コマンドの基本構文とオプション
- CSVデータのインポート
- JSONデータのインポート
- エラー処理、バッチ・サイズ、および日付フォーマット
- `load.xml` による高度な構成
- SQL*Loaderや外部表との比較

---

## LOAD コマンド

`LOAD` コマンドは、あらかじめ定義された表に対してファイルのデータをマップしようとします。

### 基本構文

```sql
LOAD [schema.]table_name data_file_name [options]
```

### 主要なオプション

| オプション | 説明 |
|---|---|
| `-skip <n>` | ファイルの先頭からスキップする行数 |
| `-batch <n>` | 1回のコミットあたりの行数 |
| `-date <fmt>` | 日付列に使用する日付フォーマット（例：`YYYY-MM-DD`） |
| `-del <char>` | CSVのデリミタ（デフォルトはカンマ） |
| `-enc <charset>` | ファイルのエンコーディング（例：`UTF-8`） |
| `-err <file>` | 個別のエラーを記録するファイル |
| `-head <on/off>` | ファイルの最初の行にヘッダー（列名）が含まれているかどうか（デフォルトはON） |
| `-json` | ファイルがJSONフォーマットであることを指定 |

---

## 基本的なCSVのロード

### 準備

```sql
CREATE TABLE staging_employees (
    employee_id NUMBER(10),
    first_name  VARCHAR2(50),
    last_name   VARCHAR2(50),
    hire_date   DATE,
    salary      NUMBER(10,2)
);
```

### デフォルト・インポート

ヘッダーがあり、日付がデータベースのデフォルトNLSフォーマットに従っているCSVの場合：

```sql
LOAD staging_employees employees.csv
```

### カスタマイズ・インポート

ヘッダーがなく、タブ区切りで、特定の日付フォーマットを持つファイルの場合：

```sql
LOAD staging_employees employees.tsv -head off -del \t -date YYYY-MM-DD
```

### パフォーマンス制御付きのインポート

```sql
LOAD staging_employees employees.csv -batch 5000 -skip 1
```

これにより、最初の行がスキップされ、5,000行ごとにコミットされます。

---

## JSONデータのロード

JSONをロードするには、ファイルがJSONの配列（各オブジェクトが1つの行を表す）である必要があります。

### 準備

```sql
CREATE TABLE customers_json (
    cust_id    NUMBER,
    cust_name  VARCHAR2(100),
    email      VARCHAR2(100)
);
```

### インポート

```sql
LOAD customers_json data.json -json
```

---

## エラー処理とロギング

インポート中にデータ型の不一致や制約違反が発生した場合、SQLclはエラーを表示し、オプションでログを記録できます。

### エラーの追跡

```sql
SET LOAD ERRORS_LOG ON
SET LOAD ERRORS_MAX 100
LOAD staging_employees employees.csv -err employees_errors.log
```

- `ERRORS_LOG` — 失敗した行を画面に表示します。
- `ERRORS_MAX` — 中断するまでに許容する最大エラー数。
- `-err` — ロードに失敗した実際の行を含むファイル。このファイルを修正して再度ロードすることができます（SQL*Loaderのバッド・ファイルと同様）。

### エラー時のロールバック

`WHENEVER SQLERROR` は `LOAD` コマンドのエラーもキャプチャするため、CI/CDにおいて非常に有効です。

```sql
WHENEVER SQLERROR EXIT 1 ROLLBACK;
LOAD my_table data.csv;
```

---

## 高度なオプション

### 列の明示的なマッピング

ファイル内の列の順序が表の順序と異なる場合は、`-column` オプションを使用します。

```sql
-- 表の列(ID, NAME)に対してファイルの第2列と第3列をマッピング
LOAD my_table my_data.csv -column 2,3
```

### NULLの処理

```sql
SET LOAD NULL_INDICATOR "[NULL]"
```

ファイル内で特定の値（`[NULL]` など）が出現した場合、それを実際のSQL NULLとして扱うように指定します。

### データのエンコーディング

UTF-16など、デフォルト以外のエンコーディングのファイルを扱う場合に指定します。

```sql
LOAD target_table data.csv -enc UTF-16
```

---

## 他のツールとの比較

| 機能 | SQLcl `LOAD` | SQL*Loader | 外部表（External Tables） |
|---|---|---|---|
| 必要条件 | SQLclのみ | Oracle Client/Instant Client | DBサーバーへのファイル・アクセス |
| 速度 | 高速（JDBCバッチ） | 最速（ダイレクト・パス対応） | 非常に高速（パラレル対応） |
| 複雑さ | 非常に低い（単一コマンド） | 中（コントロール・ファイルが必要） | 高（ディレクトリ・オブジェクトが必要） |
| JSON対応 | 組み込み | 複雑（SQL変換が必要） | 良好（Oracle 19c以降） |
| リモート・ロード | 良好（クライアントから実行） | 良好 | 不可（ファイルはサーバー側に必要） |
| 推奨ユースケース | 開発、小〜中規摸データ、CI/CD | 大規模データ、レガシー自動化 | ETL、DB内での大容量データ処理 |

---

## 構成ファイル (load.xml)

非常に複雑なロード（特定の列を無視したり、固定長のファイルを扱ったりする場合、SQL*Loaderのような細かい制御が必要になります）では、SQLclは `load.xml` 構成ファイルを生成・使用できます。

### 構成の生成

```sql
LOAD my_table data.csv -save
```

これにより、列のマッピング、日付フォーマット、およびデリミタを含む `load.xml`（または `my_table.xml`）が生成されます。

### 構成の使用

```sql
LOAD my_table data.csv -conf load.xml
```

---

## ベスト・プラクティス

- `LOAD` コマンドで本番用の表に直接インポートするのではなく、必ず一時的なステージング表を最初に使用してください。これにより、クレンジングやマージの前にデータを検証（重複、NULL、データ型など）できます。
- 中規摸から大規模なロード（10万行以上）の場合は、デフォルトよりも大きいバッチ・サイズ（例：`-batch 5000` または `10000`）を使用して、ネットワークの往復回数とコミット・オーバーヘッドを削減してください。
- インポートに失敗して不完全なデータが残るのを防ぐため、CI/CDパイプライン内でのロードには常に `WHENEVER SQLERROR EXIT ... ROLLBACK` を使用してください。
- UTF-8エンコーディングを使用し、インポート・コマンドで `-enc UTF-8` を明示的に指定することで、特に多言語データを扱う際の文字化けの問題を回避してください。
- 列の順序やフォーマットが時間の経過とともに変化する可能性がある自動化されたジョブには、コマンドライン・オプションに頼るのではなく、`-save` で生成された `load.xml` 構成ファイルを使用してください。

---

## よくある間違いと回避策

**間違い：日付フォーマットの不一致によるエラー**
CSVの日付が `MM/DD/YYYY` で、データベースのNLSフォーマットが `DD-MON-RR` の場合、ロードは失敗します。常に `-date YYYY-MM-DD` オプションを使用して、ファイルの内容に合わせてフォーマットを明示してください。

**間違い：大規模なロードでのバッチ・サイズが小さすぎる**
デフォルトのバッチ・サイズ（バージョンによって異なりますが、通常、50か100）で数百万行をロードすると、非常に時間がかかります。`-batch 5000` 以上を設定してパフォーマンスを向上させてください。

**間違い：ファイルのヘッダー行をデータとしてインポートしてしまう**
ファイルにヘッダー（列名）が含まれている場合、SQLclは自動的にその認識を試みますが、失敗して「Invalid Number」などのエラーが発生することがあります。手動で制御するには `-head on` または `-head off` を指定し、必要に応じて行をスキップするために `-skip 1` を組み合わせてください。

**間違い：Windows/Linux間でのエンコーディングや改行の問題**
Windowsで作成されたファイルにはBOM（Byte Order Mark）が含まれている場合があり、これがインポートを妨げることがあります。ファイルをUTF-8（BOMなし）で保存するか、SQLclの `-enc` オプションを使用してエンコーディングを正しく指定してください。

**間違い：ロードに失敗してもスクリプトが続行されてしまう**
`LOAD` はSQLコマンドのように動作しますが、失敗してもSQLclプロセスが停止しない場合があります。CI/CD環境では、必ずスクリプトの先頭でエラー時の終了設定を有効にしてください。

---

## 参考資料

- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLclのLOADコマンドによるデータ・インポート — ThatJeffSmith](https://www.thatjeffsmith.com/archive/2019/08/sqldeveloper-sqlcl-quick-tip-bulk-load-your-csvs-with-a-single-command/)
- [SQLcl リリース・ノート 25.2](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.html)
- [Oracle SQLcl リリース・インデックス](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/index.html)
