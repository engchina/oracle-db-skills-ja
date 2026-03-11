# MongoDB から Oracle への移行

## 概要

MongoDB から Oracle への移行は、データ・モデリングの哲学における根本的な転換を意味する。すなわち、ドキュメント指向でスキーマ・フレキシブルな NoSQL アプローチから、リレーショナルでスキーマ制約のあるモデルへの転換である。しかし、Oracle はこのギャップを埋めるために多大な投資を行ってきた。Oracle Database 21c ではバイナリ エンコードされた OSON 形式によるネイティブ JSON ストレージが導入され、Oracle 23c では **JSON リレーショナル・デュアリティ・ビュー (JSON Relational Duality Views)** が導入された。これにより、リレーショナル表に対してドキュメント・スタイルのアクセスが可能になり、両者の間の摩擦は大幅に軽減されている。

本ガイドでは、ドキュメント・モデルからリレーショナルへのマッピング戦略、Oracle の JSON 機能、集計パイプラインの変換、BSON 型のマッピング、および MongoDB からのデータ抽出について解説する。

---

## ドキュメント・モデルからリレーショナルへのマッピング戦略

### 戦略 1 — 完全な正規化（伝統的なリレーショナル）

各 MongoDB コレクションをテーブルに変換する。埋め込みドキュメントや配列を、外部キーを持つ子テーブルに分解する。このアプローチにより、クエリの柔軟性と参照整合性が最大化される。

**MongoDB ドキュメント：**
```json
{
  "_id": ObjectId("65a1234567890abcdef12345"),
  "customer_name": "Acme Corp",
  "email": "contact@acme.com",
  "addresses": [
    { "type": "billing", "street": "123 Main St", "city": "Springfield", "zip": "12345" },
    { "type": "shipping", "street": "456 Elm Ave", "city": "Shelbyville", "zip": "67890" }
  ],
  "tags": ["enterprise", "preferred"],
  "metadata": { "created_by": "admin", "source": "CRM" }
}
```

**Oracle 正規化スキーマ：**
```sql
CREATE TABLE customers (
    customer_id   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mongo_id      VARCHAR2(24) UNIQUE NOT NULL,  -- 元の _id を保持
    customer_name VARCHAR2(500) NOT NULL,
    email         VARCHAR2(255),
    created_by    VARCHAR2(100),
    source        VARCHAR2(100)
);

CREATE TABLE customer_addresses (
    address_id  NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id NUMBER NOT NULL,
    addr_type   VARCHAR2(20),
    street      VARCHAR2(500),
    city        VARCHAR2(200),
    zip         VARCHAR2(20),
    CONSTRAINT fk_addr_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE customer_tags (
    tag_id      NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id NUMBER NOT NULL,
    tag_value   VARCHAR2(100),
    CONSTRAINT fk_tag_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### 戦略 2 — Oracle 内に JSON として保存（ハイブリッド）

Oracle のネイティブ JSON 列にドキュメント構造をそのまま保持する。構造が非常に多様な場合や、ネストが深くリレーショナルな正規化が煩雑になるドキュメントに最適である。

```sql
-- Oracle 21c 以降のネイティブ JSON 型
CREATE TABLE customers (
    customer_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mongo_id    VARCHAR2(24) UNIQUE NOT NULL,
    doc         JSON NOT NULL
);

-- Oracle 12c ～ 20c での CLOB と IS JSON 制約の使用
CREATE TABLE customers (
    customer_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    mongo_id    VARCHAR2(24) UNIQUE NOT NULL,
    doc         CLOB,
    CONSTRAINT chk_customers_json CHECK (doc IS JSON)
);

-- ドキュメントのインサート
INSERT INTO customers (mongo_id, doc)
VALUES ('65a1234567890abcdef12345',
        '{"customer_name":"Acme Corp","email":"contact@acme.com"}');
```

### 戦略 3 — Oracle JSON デュアリティ・ビュー (23c) — 両方のメリットを享受

Oracle 23c の JSON リレーショナル・デュアリティ・ビューを使用すると、リレーショナル・スキーマをドキュメント・ストアとして扱うことができる。アプリケーションは JSON ドキュメントとして読み書きを行い、Oracle はデータをリレーショナルに保存する。これは、Oracle への新規移行において最も強力なアプローチである。

```sql
-- 基盤となるリレーショナル表
CREATE TABLE customers (
    customer_id   NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_name VARCHAR2(500) NOT NULL,
    email         VARCHAR2(255)
);

CREATE TABLE addresses (
    address_id  NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id NUMBER NOT NULL,
    addr_type   VARCHAR2(20),
    street      VARCHAR2(500),
    city        VARCHAR2(200),
    zip         VARCHAR2(20),
    CONSTRAINT fk_addr FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- JSON デュアリティ・ビュー — リレーショナル・データをドキュメントとして公開
CREATE OR REPLACE JSON RELATIONAL DUALITY VIEW customers_dv AS
SELECT JSON {
    '_id'           : c.customer_id,
    'customerName'  : c.customer_name,
    'email'         : c.email,
    'addresses'     : [
        SELECT JSON {
            'type'   : a.addr_type,
            'street' : a.street,
            'city'   : a.city,
            'zip'    : a.zip
        }
        FROM addresses a WITH INSERT UPDATE DELETE
        WHERE a.customer_id = c.customer_id
    ]
}
FROM customers c WITH INSERT UPDATE DELETE;

-- アプリケーションは、MongoDB であるかのようにデュアリティ・ビュー経由でクエリや更新が可能：
SELECT doc FROM customers_dv WHERE json_value(doc, '$._id') = '42';
```

---

## BSON 型から Oracle 型への変換

| MongoDB BSON 型 | Oracle 型 | 備考 |
|---|---|---|
| `ObjectId` | `VARCHAR2(24)` または `RAW(12)` | 24 文字の 16 進文字列。可読性のために VARCHAR2 を推奨 |
| `String` | `VARCHAR2(4000)` または `CLOB` | 長さに依存 |
| `Number (int32)` | `NUMBER(10)` | |
| `Number (int64)` | `NUMBER(19)` | |
| `Number (double)` | `BINARY_DOUBLE` | IEEE 754 64 ビット |
| `Number (Decimal128)` | `NUMBER(38,18)` | 高精度 10 進数 |
| `Boolean` | CHECK (0,1) 付き `NUMBER(1)` | または Oracle 23c BOOLEAN |
| `Date` | `TIMESTAMP WITH TIME ZONE` | BSON Date は UTC エポックからのミリ秒 |
| `Null` | `NULL` | |
| `Array` | 子テーブル（正規化）または JSON 列内の JSON 配列 | |
| `Embedded document` | 別テーブル（正規化）または JSON 列内の JSON オブジェクト | |
| `Binary data (BinData)` | `BLOB` または `RAW(n)` | |
| `ObjectId as _id` | `VARCHAR2(24)` + 主キー | |
| `UUID (subtype 3/4)` | `RAW(16)` または `VARCHAR2(36)` | |
| `Regular expression` | `VARCHAR2(500)` | プロパティとフラグを分けて保存 |
| `JavaScript code` | `CLOB` | テキストとして保存（Oracle 内での実行は不可） |
| `Timestamp (internal)` | `TIMESTAMP` | 内部的な BSON 型。標準の TIMESTAMP にマップ |

### 日付の変換

MongoDB BSON Date は、Unix エポックからの経過ミリ秒 (UTC) として保存される。これを Oracle の TIMESTAMP WITH TIME ZONE に変換する例：

```sql
-- mongo_ts が文字列としてのエポックミリ秒（NUMBER として格納）の場合
SELECT TIMESTAMP '1970-01-01 00:00:00 UTC' +
       NUMTODSINTERVAL(mongo_ts / 1000, 'SECOND') AS oracle_ts
FROM staging_raw;

-- 逆方向：Oracle TIMESTAMP から MongoDB のエポックミリ秒（比較用）
SELECT (oracle_ts - TIMESTAMP '1970-01-01 00:00:00 UTC') * 86400000 AS mongo_ms
FROM orders;
```

---

## 集計パイプラインから Oracle SQL への変換

MongoDB の集計パイプラインは、ドキュメントを変換する一連のステージである。各ステージには、対応する Oracle SQL が存在する。

### $match → WHERE

```javascript
// MongoDB
db.orders.aggregate([
  { $match: { status: "shipped", total: { $gte: 100 } } }
])
```

```sql
-- Oracle
SELECT * FROM orders WHERE status = 'shipped' AND total >= 100;
```

### $group → GROUP BY / 集計関数

```javascript
// MongoDB
db.orders.aggregate([
  { $group: {
      _id: "$customer_id",
      order_count: { $sum: 1 },
      total_spent: { $sum: "$amount" },
      avg_order:   { $avg: "$amount" },
      max_order:   { $max: "$amount" }
  }}
])
```

```sql
-- Oracle
SELECT customer_id,
       COUNT(*)          AS order_count,
       SUM(amount)       AS total_spent,
       AVG(amount)       AS avg_order,
       MAX(amount)       AS max_order
FROM orders
GROUP BY customer_id;
```

### $project → SELECT 列リスト

```javascript
// MongoDB
db.customers.aggregate([
  { $project: { customer_name: 1, email: 1, _id: 0,
                full_address: { $concat: ["$street", " ", "$city"] } } }
])
```

```sql
-- Oracle
SELECT customer_name, email, street || ' ' || city AS full_address
FROM customers;
```

### $sort → ORDER BY

```javascript
// MongoDB
db.orders.aggregate([{ $sort: { total: -1, order_date: 1 } }])
```

```sql
-- Oracle
SELECT * FROM orders ORDER BY total DESC, order_date ASC;
```

### $limit / $skip → FETCH FIRST / OFFSET

```javascript
// MongoDB
db.orders.aggregate([{ $skip: 20 }, { $limit: 10 }])
```

```sql
-- Oracle
SELECT * FROM orders OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### $lookup → JOIN

```javascript
// MongoDB $lookup (左外結合)
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customer_id",
      foreignField: "_id",
      as: "customer"
    }
  }
])
```

```sql
-- Oracle
SELECT o.*, c.customer_name, c.email
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id;
```

### $unwind → JSON_TABLE または Lateral 結合

```javascript
// MongoDB $unwind (配列を個別のドキュメントに展開)
db.orders.aggregate([{ $unwind: "$line_items" }])
```

```sql
-- Oracle: line_items が JSON 列内の JSON 配列である場合
SELECT o.order_id, o.order_date, li.*
FROM orders o,
     JSON_TABLE(o.line_items, '$[*]'
         COLUMNS (
             product_id NUMBER  PATH '$.product_id',
             qty        NUMBER  PATH '$.qty',
             price      NUMBER  PATH '$.price'
         )
     ) li;

-- Oracle: line_items が子テーブルである場合 (正規化アプローチ)
SELECT o.order_id, o.order_date, li.product_id, li.qty, li.price
FROM orders o
JOIN order_line_items li ON li.order_id = o.order_id;
```

---

## ドキュメント照会のための JSON_TABLE

`JSON_TABLE` は、JSON または CLOB 列に保存された JSON データを操作するための Oracle の最も強力なツールである。

```sql
-- サンプル・データ
CREATE TABLE mongo_import (
    id   NUMBER GENERATED ALWAYS AS IDENTITY,
    doc  JSON
);

-- ネストされたドキュメント・フィールドへのクエリ
SELECT jt.*
FROM mongo_import mi,
     JSON_TABLE(mi.doc, '$'
         COLUMNS (
             mongo_id      VARCHAR2(24)  PATH '$._id.$oid',
             customer_name VARCHAR2(500) PATH '$.customer_name',
             email         VARCHAR2(255) PATH '$.email',
             NESTED PATH '$.addresses[*]'
                 COLUMNS (
                     addr_type VARCHAR2(20)  PATH '$.type',
                     street    VARCHAR2(500) PATH '$.street',
                     city      VARCHAR2(200) PATH '$.city',
                     zip       VARCHAR2(20)  PATH '$.zip'
                 )
         )
     ) jt;
```

---

## MongoDB からのデータ抽出

### 方法 1 — mongoexport (JSON)

```bash
# コレクションを JSON にエクスポート (1 行 1 ドキュメント)
mongoexport \
  --host localhost:27017 \
  --db myapp \
  --collection orders \
  --out orders.json

# クエリ・フィルタを使用してエクスポート
mongoexport \
  --host localhost:27017 \
  --db myapp \
  --collection orders \
  --query '{"status": "shipped"}' \
  --out shipped_orders.json
```

### 方法 2 — Python ETL (pymongo と oracledb)

変換が必要な複雑なドキュメント構造に：

```python
from pymongo import MongoClient
import oracledb
import json

# ソース
mongo = MongoClient("mongodb://localhost:27017/")
db = mongo["myapp"]

# ターゲット (Thin モード)
ora = oracledb.connect(user="user", password="pass", dsn="localhost:1521/ORCL")
ora_cur = ora.cursor()

# ステージ 1: 生ドキュメントをステージング用 JSON 表にロード
ora_cur.execute("DELETE FROM mongo_staging")
insert_sql = "INSERT INTO mongo_staging (mongo_id, raw_doc) VALUES (:1, :2)"

batch = []
for doc in db.orders.find():
    mongo_id = str(doc['_id'])
    doc['_id'] = str(doc['_id'])
    # 日付などの変換を事前に行う
    batch.append((mongo_id, json.dumps(doc, default=str)))
    if len(batch) >= 1000:
        ora_cur.executemany(insert_sql, batch)
        batch = []

if batch:
    ora_cur.executemany(insert_sql, batch)
ora.commit()

# ステージ 2: 正規化テーブルへの変換とインサート
ora_cur.execute("""
    INSERT INTO orders (mongo_id, customer_id, status, total)
    SELECT jt.mongo_id, jt.customer_id, jt.status, TO_NUMBER(jt.total)
    FROM mongo_staging ms,
         JSON_TABLE(ms.raw_doc, '$' COLUMNS (
             mongo_id    VARCHAR2(24) PATH '$._id',
             customer_id VARCHAR2(50) PATH '$.customer_id',
             status      VARCHAR2(50) PATH '$.status',
             total       VARCHAR2(50) PATH '$.total'
         )) jt
""")
ora.commit()
```

---

## ベスト・プラクティス

1. **移行中に MongoDB の _id 値を保持する。** Oracle 側で `mongo_id` 列に元の MongoDB ObjectId を保存する。これによりデータの照合が可能になり、必要に応じたロールバックも容易になる。

2. **中間ステップとしてステージング JSON 表を使用する。** まず MongoDB の生 JSON ドキュメントを Oracle の JSON ステージング表にロードし、その後 SQL と `JSON_TABLE` を使用してリレーショナルなターゲット・スキーマに変換する。これにより、抽出フェーズと変換フェーズを分離できる。

3. **正規化前にコレクションをプロファイリングする。** MongoDB のスキーマ柔軟性により、同じコレクション内のドキュメントでも構造が大きく異なる場合がある。コレクションを照会し、実際のフィールド構成を把握すること。

4. **ドキュメント・スタイルのアクセスが必要なアプリには Duality ビューを検討する。** MongoDB ドキュメント API を前提としたアプリケーションの場合、Oracle 23c のデュアリティ・ビューを使用することで、JSON インターフェースを維持したままリレーショナルにデータを格納できる。

5. **欠落しているフィールドを適切に処理する。** MongoDB でフィールドが欠落している場合、Oracle では NULL となる。安全な抽出のために、`JSON_VALUE(doc, '$.field' DEFAULT NULL ON ERROR)` を使用すること。

6. **頻繁に照会される JSON フィールドに索引を作成する。** ハイブリッド JSON ストレージを使用する場合、頻繁に照会される JSON パスに対して JSON Search インデックスや関数ベースのインデックスを作成する。

---

## よくある移行の落とし穴

**落とし穴 1 — ポリモーフィックなコレクション：**
MongoDB のコレクションには、イベントの種類ごとに異なるフィールドを持つ「混合形状」のドキュメントが含まれることがよくある。Oracle では、共通フィールドをベース・テーブルに、可変フィールドを JSON 列に保持するか、判別列 (Discriminator Column) を持つテーブル継承パターンの使用を検討すること。

**落とし穴 2 — _id 型の前提：**
すべての MongoDB _id フィールドが ObjectId とは限らない。カスタム文字列や数値、複合値が使用されている場合がある。各コレクションの実際の _id 型を確認すること。

**落とし穴 3 — 埋め込み配列の更新：**
MongoDB は配列要素の効率的な更新 (`$push`, `$pull` 等) をサポートしている。Oracle リレーショナルでは、子テーブルに対する DELETE + INSERT が必要になる。アプリケーション・コード内の配列変更パターンをすべて確認すること。

**落とし穴 4 — MongoDB トランザクション vs Oracle トランザクション：**
MongoDB は v4.0 以降でマルチドキュメント ACID トランザクションをサポートしている。Oracle は常に完全な ACID トランザクションを提供してきた。MongoDB の過去のトランザクション制限を補完するために作成された複雑なロジックは、Oracle 移行時に簡素化できる場合がある。

**落とし穴 5 — フィールド名の大文字小文字の区別：**
MongoDB のフィールド名はケース・センシティブである。Oracle の列名は（二重引用符で囲まない限り）自動的に大文字に変換される。MongoDB の camelCase を Oracle の SNAKE_CASE にマッピングすることで、引用符の要件を回避することを推奨する。

**落とし穴 6 — 日付のタイムゾーン処理：**
BSON Date は常に UTC である。Oracle の TIMESTAMP WITH TIME ZONE にロードして UTC の意図を保持し、必要に応じてアプリケーション側でローカル時間に変換すること。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle Database 19c SQL Language Reference — JSON_TABLE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/JSON_TABLE.html)
- [Oracle Database 19c SQL Language Reference — JSON_VALUE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/JSON_VALUE.html)
- [Oracle Database 23c — JSON Relational Duality Views](https://www.oracle.com/database/23c/technologies/json-relational-duality-views/)
- [Oracle Database 19c JSON Developer's Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/adjsn/json-in-oracle-database.html)
- [python-oracledb documentation](https://python-oracledb.readthedocs.io/en/latest/)
