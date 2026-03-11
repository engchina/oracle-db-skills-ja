# Oracle DatabaseにおけるJSON

## 概要

OracleにおけるJSONの扱いは、VARCHAR2やCLOB列に文字列として格納する方式（12c）から、深いクエリの統合、索引付け、およびスキーマの適用が可能な専用のネイティブJSONデータ型（21c以降）へと進化してきた。Oracle 23cにおける「JSON関係ディアルティ・ビュー（JSON Relational Duality Views）」は、パラダイムシフトを意味しており、同じデータをJSONドキュメントとしてもリレーショナルな行としても同時にアクセス・変更することを可能にする。

本ガイドでは、ストレージ・オプション、SQL/JSON関数の全セット、索引戦略、および最新のJSONディアルティ・ビューのアーキテクチャまで、包括的に解説する。

---

## JSONのストレージ・オプション

### 21cより前: IS JSON制約を伴う VARCHAR2 / CLOB

```sql
-- 小規模なJSONドキュメント用の VARCHAR2 (≤32767バイト)
CREATE TABLE product_catalog (
    product_id   NUMBER PRIMARY KEY,
    product_name VARCHAR2(200) NOT NULL,
    attributes   VARCHAR2(4000)
        CONSTRAINT chk_attributes_json CHECK (attributes IS JSON)
);

-- 大規模ドキュメント用の CLOB
CREATE TABLE event_log (
    event_id     NUMBER PRIMARY KEY,
    event_data   CLOB
        CONSTRAINT chk_event_json CHECK (event_data IS JSON)
);

-- LAX vs STRICT によるJSON検証
-- LAX (デフォルト): 重複キー、末尾のカンマ、引用符なしのキーを許可
-- STRICT: 厳密なJSON構文を強制
CREATE TABLE strict_json_table (
    id   NUMBER PRIMARY KEY,
    data CLOB CONSTRAINT chk_strict CHECK (data IS JSON STRICT)
);
```

### 21c以降: ネイティブJSONデータ型

ネイティブの `JSON` 型は、JSONをコンパクトなバイナリ形式である OSON (Oracle Serialized Object Notation) で格納する。CLOB/VARCHAR2と比較した利点は以下の通り：
- アクセスごとのJSONパースが不要
- ストレージ・サイズが小さい
- クエリ実行が高速
- 自動的な構造検証

```sql
-- ネイティブJSON型 (21c以降)
CREATE TABLE orders (
    order_id     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  NUMBER NOT NULL,
    order_data   JSON NOT NULL,
    created_at   TIMESTAMP DEFAULT SYSTIMESTAMP
);

-- JSONドキュメントの挿入
INSERT INTO orders (customer_id, order_data)
VALUES (42, JSON('{"status": "pending",
                   "items": [
                     {"sku": "WGT-001", "qty": 2, "price": 29.99},
                     {"sku": "GAD-007", "qty": 1, "price": 149.99}
                   ],
                   "shipping": {"method": "express", "address": "123 Main St"}}'));

-- 文字列として挿入することも可能。OracleはパースしてバイナリJSONとして格納する
INSERT INTO orders (customer_id, order_data)
VALUES (43, '{"status": "shipped", "items": [{"sku": "WGT-002", "qty": 3, "price": 19.99}]}');
```

---

## ドット表記法 (簡易SQL/JSON)

Oracleのドット表記法は、JSONパスをシンプルかつ読みやすくナビゲートする方法を提供する。これは IS JSON制約を持つ VARCHAR2/CLOB 列およびネイティブJSON列の両方で動作する。

```sql
-- シンプルなドット表記
SELECT o.order_data.status,
       o.order_data.shipping.method,
       o.order_data.shipping.address
FROM   orders o
WHERE  o.order_data.status = 'pending';

-- 配列要素へのアクセス (0開始のインデックス)
SELECT o.order_data.items[0].sku    AS first_item_sku,
       o.order_data.items[0].price  AS first_item_price
FROM   orders o;

-- ドット表記はデフォルトで VARCHAR2 を返す。数値が必要な場合はサフィックスを使用
SELECT o.order_data.items[0].price.numberOnly() AS price_number
FROM   orders o;
```

---

## JSON_VALUE: スカラー値の抽出

`JSON_VALUE` は、JSONドキュメントから単一のスカラー値を抽出する。パスが存在しない場合や値がスカラーでない場合、デフォルトで `NULL` を返す。

```sql
-- 基本的な JSON_VALUE
SELECT JSON_VALUE(order_data, '$.status')             AS status,
       JSON_VALUE(order_data, '$.shipping.method')    AS ship_method,
       JSON_VALUE(order_data, '$.items[0].sku')       AS first_sku
FROM   orders;

-- RETURNING 句による型変換
SELECT JSON_VALUE(order_data, '$.items[0].price' RETURNING NUMBER(10,2)) AS price,
       JSON_VALUE(order_data, '$.created_ts'     RETURNING TIMESTAMP)    AS ts
FROM   orders;

-- エラー処理句
SELECT JSON_VALUE(order_data, '$.missing_field'
           DEFAULT 'N/A' ON EMPTY      -- パスが見つからない場合
           NULL ON ERROR)              -- JSONの形式不正や型の不一致の場合
AS   safe_value
FROM orders;

-- NULL ON EMPTY | ERROR ON EMPTY | DEFAULT 値 ON EMPTY
-- NULL ON ERROR | ERROR ON ERROR | DEFAULT 値 ON ERROR

-- WHERE 句での使用
SELECT order_id, customer_id
FROM   orders
WHERE  JSON_VALUE(order_data, '$.status') = 'pending'
  AND  JSON_VALUE(order_data, '$.shipping.method' RETURNING VARCHAR2) = 'express';
```

---

## JSON_QUERY: JSONフラグメントの抽出

`JSON_QUERY` は、JSONオブジェクトまたは配列を返す（スカラー値ではない）。ターゲットの値自体がJSON構造である場合に使用する。

```sql
-- shipping オブジェクト全体を抽出
SELECT JSON_QUERY(order_data, '$.shipping')       AS shipping_json,
       JSON_QUERY(order_data, '$.items')          AS items_array,
       JSON_QUERY(order_data, '$.items[0]')       AS first_item
FROM   orders;

-- WITH WRAPPER: 結果を配列のブラケット [] で囲む
-- パスが複数のアイテムを返す場合に必要
SELECT JSON_QUERY(order_data, '$.items[*].sku' WITH ARRAY WRAPPER) AS all_skus
FROM   orders;

-- WITH CONDITIONAL WRAPPER: 結果が既に配列でない場合のみ囲む
SELECT JSON_QUERY(order_data, '$.shipping' WITH CONDITIONAL WRAPPER) AS shipping
FROM   orders;

-- 整形表示 (Pretty printing)
SELECT JSON_QUERY(order_data, '$' RETURNING VARCHAR2(4000) PRETTY) AS pretty_json
FROM   orders WHERE order_id = 1;
```

---

## JSON_EXISTS: パスの存在確認

`JSON_EXISTS` は、パスが存在するか、または条件に一致するかをテストし、TRUE/FALSE を返す（WHERE句で使用される）。

```sql
-- パスの存在を確認
SELECT order_id FROM orders
WHERE  JSON_EXISTS(order_data, '$.shipping.tracking_number');

-- フィルタ条件を伴うテスト (JSON_EXISTS 述語)
SELECT order_id FROM orders
WHERE  JSON_EXISTS(order_data, '$.items[*]?(@.price > 100)');

-- 複数条件
SELECT order_id FROM orders
WHERE  JSON_EXISTS(order_data, '$.items[*]?(@.sku == "WGT-001" && @.qty >= 2)');

-- null値の確認
SELECT order_id FROM orders
WHERE  JSON_EXISTS(order_data, '$.status?(@ != null)');
```

---

## JSON_TABLE: JSONをリレーショナルな行に変換 (シュレッド)

`JSON_TABLE` は最も強力なJSON関数であり、JSONドキュメント（または配列）を仮想的なリレーショナル表に変換し、結合、フィルタリング、集計を可能にする。

```sql
-- 注文アイテムを個別の行に展開
SELECT o.order_id, o.customer_id, jt.sku, jt.qty, jt.price,
       jt.qty * jt.price AS line_total
FROM   orders o,
       JSON_TABLE(o.order_data, '$.items[*]'
           COLUMNS (
               sku    VARCHAR2(20)    PATH '$.sku',
               qty    NUMBER          PATH '$.qty',
               price  NUMBER(10,2)    PATH '$.price'
           )
       ) jt;

-- 階層データのためのネストされた JSON_TABLE
SELECT o.order_id, jt.method, jt.street, jt.city
FROM   orders o,
       JSON_TABLE(o.order_data, '$'
           COLUMNS (
               method  VARCHAR2(20)  PATH '$.shipping.method',
               NESTED PATH '$.shipping.address[*]' COLUMNS (
                   street  VARCHAR2(100)  PATH '$.street',
                   city    VARCHAR2(50)   PATH '$.city',
                   zip     VARCHAR2(10)   PATH '$.zip'
               )
           )
       ) jt;

-- エラー処理を伴う JSON_TABLE
SELECT customer_id, jt.item_sku, jt.item_price
FROM   orders,
       JSON_TABLE(order_data, '$.items[*]'
           ERROR ON ERROR  -- JSONの形式不正でエラーを発生させる
           COLUMNS (
               item_sku    VARCHAR2(20)   PATH '$.sku'   NULL ON ERROR,
               item_price  NUMBER(10,2)   PATH '$.price' DEFAULT 0 ON ERROR
           )
       ) jt;

-- 展開されたアイテムの集計
SELECT o.order_id,
       COUNT(jt.sku)       AS item_count,
       SUM(jt.qty * jt.price) AS order_total
FROM   orders o,
       JSON_TABLE(o.order_data, '$.items[*]'
           COLUMNS (
               sku    VARCHAR2(20)  PATH '$.sku',
               qty    NUMBER        PATH '$.qty',
               price  NUMBER(10,2)  PATH '$.price'
           )
       ) jt
GROUP  BY o.order_id;
```

---

## JSONの変更関数

```sql
-- JSON_MERGEPATCH: JSONドキュメントのパラレルなマージ/更新 (RFC 7396)
UPDATE orders
SET    order_data = JSON_MERGEPATCH(order_data,
           '{"status": "shipped", "shipped_at": "2025-01-15T10:30:00Z"}')
WHERE  order_id = 1;

-- JSON_TRANSFORM (21c以降): より強力で局所的な更新
UPDATE orders
SET    order_data = JSON_TRANSFORM(order_data,
           SET    '$.status'      = 'delivered',
           SET    '$.delivered_at' = SYSTIMESTAMP FORMAT JSON,
           APPEND '$.tags'        = 'completed'
       )
WHERE  order_id = 1;

-- キーの削除
UPDATE orders
SET    order_data = JSON_TRANSFORM(order_data,
           REMOVE '$.temp_processing_notes')
WHERE  order_id = 1;
```

---

## JSONの索引

JSONデータの適切な索引付けはパフォーマンスにおいて不可欠である。Oracleはいくつかのオプションを提供している。

### JSON_VALUE に対するファンクション索引

```sql
-- 特定のJSONスカラーパスに対する索引 (最も選択的で効率的)
CREATE INDEX idx_order_status
    ON orders (JSON_VALUE(order_data, '$.status' RETURNING VARCHAR2(20)));

CREATE INDEX idx_order_ship_method
    ON orders (JSON_VALUE(order_data, '$.shipping.method' RETURNING VARCHAR2(20)));

-- これらの索引はオプティマイザによって自動的に使用される
SELECT order_id FROM orders
WHERE  JSON_VALUE(order_data, '$.status' RETURNING VARCHAR2(20)) = 'pending';
-- またはドット表記を使用した場合
SELECT order_id FROM orders WHERE order_data.status = 'pending';
```

### JSON検索索引 (Oracle Text 全文検索)

JSONドキュメント全体に対する柔軟なマルチパス検索用：

```sql
-- 全JSONコンテンツに対する全文検索索引を作成
CREATE SEARCH INDEX idx_order_json_search ON orders (order_data)
    FOR JSON;

-- JSON_EXISTS と組み合わせて使用
SELECT order_id FROM orders
WHERE  JSON_EXISTS(order_data, '$.items[*]?(@.sku == "WGT-001")');

-- 検索索引の同期 (SYNC ON COMMIT でない場合)
EXEC CTX_DDL.SYNC_INDEX('idx_order_json_search');
```

### JSON + リレーショナル のコンポジット索引

```sql
-- 一般的なクエリパターンのための複合索引
CREATE INDEX idx_cust_status ON orders (
    customer_id,
    JSON_VALUE(order_data, '$.status' RETURNING VARCHAR2(20))
);

-- WHERE customer_id = ? AND order_data.status = ? をカバー
```

---

## JSON関係ディアルティ・ビュー (23c)

JSON関係ディアルティ・ビュー（JSON Relational Duality Views）は、Oracle 23cの主要機能の1つである。リレーショナル表のデータをJSONドキュメントとして公開し、JSONまたはSQLインターフェースのいずれからでも完全に照会・変更できるようにする。これにより、アプリケーション・オブジェクトとデータベースの行の間のインピーダンス・ミスマッチが解消される。

### ディアルティ・ビューの作成

```sql
-- 基底のリレーショナル表
CREATE TABLE customers_23 (
    customer_id  NUMBER PRIMARY KEY,
    name         VARCHAR2(100),
    email        VARCHAR2(200)
);

CREATE TABLE orders_23 (
    order_id     NUMBER PRIMARY KEY,
    customer_id  NUMBER REFERENCES customers_23(customer_id),
    status       VARCHAR2(20),
    total_amount NUMBER(12,2)
);

CREATE TABLE order_items_23 (
    item_id     NUMBER PRIMARY KEY,
    order_id    NUMBER REFERENCES orders_23(order_id),
    sku         VARCHAR2(50),
    quantity    NUMBER,
    unit_price  NUMBER(10,2)
);

-- JSONディアルティ・ビュー: 顧客ごとに1つのJSONドキュメント（ネストされた注文を含む）
CREATE OR REPLACE JSON RELATIONAL DUALITY VIEW customer_orders_dv AS
    SELECT JSON {
        'customerId'  : c.customer_id,
        'name'        : c.name,
        'email'       : c.email,
        'orders'      : [
            SELECT JSON {
                'orderId'     : o.order_id,
                'status'      : o.status,
                'totalAmount' : o.total_amount,
                'items'       : [
                    SELECT JSON {
                        'sku'       : i.sku,
                        'quantity'  : i.quantity,
                        'unitPrice' : i.unit_price
                    }
                    FROM order_items_23 i WITH (INSERT UPDATE DELETE)
                    WHERE i.order_id = o.order_id
                ]
            }
            FROM orders_23 o WITH (INSERT UPDATE DELETE)
            WHERE o.customer_id = c.customer_id
        ]
    }
    FROM customers_23 c WITH (INSERT UPDATE DELETE);

-- ディアルティ・ビューを JSON として照会
SELECT * FROM customer_orders_dv WHERE json_value(data, '$.customerId') = 42;

-- ディアルティ・ビューを介した挿入 (全テーブルへ自動的に挿入)
INSERT INTO customer_orders_dv VALUES (
    '{"customerId": 100,
      "name": "Acme Corp",
      "email": "acme@example.com",
      "orders": [
        {"orderId": 5001, "status": "pending", "totalAmount": 599.98,
         "items": [{"sku": "WGT-001", "quantity": 2, "unitPrice": 299.99}]}
      ]}'
);
-- これにより customers_23, orders_23, および order_items_23 へ原子的に挿入される
```

---

## JSONの格納 vs. 照会: 設計上の考慮事項

### JSONとして格納すべき場合

- **可変構造**: 属性が製品カテゴリ、イベント、または顧客セグメントごとに異なる
- **スキーマレスな拡張**: スキーマ移行なしでフィールドを追加可能にしたい
- **ドキュメント指向データ**: 設定オブジェクト、APIペイロード、シリアライズされたオブジェクト
- **ネスト/配列データ**: 自然なJSON構造を持つ明細項目、タグ、監査証跡

### 代わりに正規化すべき場合

- **頻繁に照会されるスカラーフィールド**: 常に `$.status` で検索する場合、それは列として格納すべき
- **参照整合性が必要な場合**: 外部キーにはリレーショナルな列が必要
- **集計とレポート**: リレーショナル列に対する GROUP BY, SUM, AVG の方が高速かつ明確
- **索引の選択性**: `NUMBER` 列に対するBツリー索引は、JSONパス索引よりも圧倒的に優れている

### ハイブリッド・アプローチ (最も一般的)

```sql
-- 頻繁に照会されるフィールドはリレーショナル列として格納
-- 可変/拡張的な属性は JSON として格納
CREATE TABLE products (
    product_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku           VARCHAR2(50)  NOT NULL UNIQUE,
    product_name  VARCHAR2(200) NOT NULL,
    category_id   NUMBER        NOT NULL,  -- リレーショナルな外部キー
    price         NUMBER(10,2)  NOT NULL,  -- 索引、集計対象
    status        VARCHAR2(20)  DEFAULT 'ACTIVE',  -- 頻繁なフィルタ対象
    attributes    JSON,          -- 可変項目: 色、サイズ、素材など
    created_at    TIMESTAMP DEFAULT SYSTIMESTAMP
);

-- 頻繁に使用される列に対するリレーショナル索引
CREATE INDEX idx_products_category ON products(category_id, status, price);

-- 一般的なJSON属性に対するファンクション索引
CREATE INDEX idx_products_color
    ON products (JSON_VALUE(attributes, '$.color' RETURNING VARCHAR2(50)));
```

---

## ベスト・プラクティス

- **新しいスキーマにはネイティブの JSON 型 (21c以降) を使用する。** バイナリの OSON 形式は、CLOB ベースのストレージよりも大幅に高速である。
- **21cより前のデータベースでは、VARCHAR2/CLOB 列に IS JSON 制約を追加**し、挿入時に検証を行う。
- **頻繁に照会されるJSONパスには、全文検索索引ではなくファンクション索引を作成する。** 単一パスのクエリにはこちらの方が効率的である。
- **配列の展開には SELECT 句の JSON_VALUE ではなく FROM 句の JSON_TABLE を使用する。** セットベースでありオプティマイザとの相性が良い。
- **WHERE 句に出現するスカラー値は、リレーショナル列として格納**し、標準的な索引を付ける。JSONパス・クエリは、たとえ索引があっても、型指定された列に対するBツリーの効率には及ばない。
- **ドキュメントの更新には JSON_MERGEPATCH を使用する。** アプリケーション側で取得・パース・変更・再挿入する手間を省くことができる。
- **JSON側でスキーマを強制するために、JSONディアルティ・ビューで `VALIDATE` を有効にする。**

---

## よくある間違い

### 間違い 1: 大きな JSON に VARCHAR2 を使用する

VARCHAR2 は PL/SQL で 32,767 バイト、SQL で 4,000 バイト（`MAX_STRING_SIZE=EXTENDED` でない場合）に制限されている。これを超える可能性があるドキュメントには、CLOB またはネイティブ JSON を使用すること。

### 間違い 2: 照会される JSON パスに索引がない

```sql
-- これは全行のJSONに対するフル・テーブル・スキャンになる
SELECT * FROM orders WHERE JSON_VALUE(order_data, '$.status') = 'pending';

-- 修正案: ファンクション索引を追加
CREATE INDEX idx_order_status ON orders(JSON_VALUE(order_data, '$.status' RETURNING VARCHAR2(20)));
```

### 間違い 3: SQL/JSON 関数で済む場合にアプリケーション・コードで JSON をパースする

JSONドキュメント全体をアプリケーションに取得してパースし、1つのフィールドを取り出して戻すようなことはしない。SQLクエリ内で `JSON_VALUE` を使用すること。

### 間違い 4: JSON_VALUE が適切な場面で JSON_QUERY を使用する

`JSON_QUERY` はスカラー値に対しても JSON 文字列を返す。スカラー値の抽出には、常に `JSON_VALUE` を使用して、型指定された Oracle の値を取得すること。

### 間違い 5: JSON_VALUE における型変換の失念

`JSON_VALUE` はデフォルトで VARCHAR2 を返す。`RETURNING NUMBER` がないと、数値の比較や算術演算が誤った結果になったり、暗黙的な型変換エラーが発生したりする。

```sql
-- 誤り: 文字列と数値の比較
WHERE JSON_VALUE(order_data, '$.total') > 100  -- 文字列比較になる！

-- 正解
WHERE JSON_VALUE(order_data, '$.total' RETURNING NUMBER) > 100
```

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c JSON開発者ガイド (ADJSN)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adjsn/)
- [Oracle Database 19c SQL言語リファレンス — JSON関数](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [Oracle Database 21c — JSONデータ型](https://docs.oracle.com/en/database/oracle/oracle-database/21/adjsn/)
- [Oracle Database 23c — JSON関係ディアルティ・ビュー](https://docs.oracle.com/en/database/oracle/oracle-database/23/jsnvu/)
