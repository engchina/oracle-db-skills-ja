# Oracle DatabaseにおけるXML

## 概要

Oracleは、Oracle 9iから `XMLType` データ型および Oracle XML DB (XMLDB) コンポーネントを通じて、ネイティブのXMLサポートを提供している。OracleのXML機能は、ストレージ、XQueryやXPathによる照会、生成、変換、および索引付けにまで及ぶ。新しいアプリケーション開発ではJSONがXMLに取って代わることが多くなったが、XMLは依然として以下の用途で不可欠である：

- EDI、HIPAA、および政府のデータ交換標準
- SOAPウェブサービスの統合
- レガシー・システムのインターフェース
- ドキュメント管理およびコンテンツ・リポジトリ
- 設定情報の格納

---

## XMLTypeのストレージ・オプション

Oracle XMLTypeは、3つの異なる内部形式で格納でき、それぞれパフォーマンス上のトレードオフがある。

### 1. オブジェクト・リレーショナル・ストレージ (スキーマ登録済みXML)

最適：高度に構造化され、頻繁に照会されるXMLで、安定した既知のスキーマを持つ場合。OracleはXML要素を内部的にリレーショナル列にマップし、高速なXPathナビゲーションを可能にする。

```sql
-- XMLスキーマの登録
BEGIN
    DBMS_XMLSCHEMA.REGISTER_SCHEMA(
        SCHEMAURL => 'http://myapp.com/schemas/order.xsd',
        SCHEMADOC => XMLTYPE(
            '<?xml version="1.0"?>
             <xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
                         targetNamespace="http://myapp.com/schemas/order">
               <xs:element name="Order">
                 <xs:complexType>
                   <xs:sequence>
                     <xs:element name="OrderId" type="xs:integer"/>
                     <xs:element name="CustomerName" type="xs:string"/>
                     <xs:element name="TotalAmount" type="xs:decimal"/>
                   </xs:sequence>
                 </xs:complexType>
               </xs:element>
             </xs:schema>'
        ),
        LOCAL => TRUE,
        GENTYPES => TRUE,
        GENBEAN => FALSE,
        GENTABLES => FALSE
    );
END;
/

-- スキーマベースの XMLType を持つ表の作成
CREATE TABLE orders_xml (
    order_id   NUMBER PRIMARY KEY,
    order_doc  XMLType
)
XMLTYPE COLUMN order_doc STORE AS OBJECT RELATIONAL
    XMLSCHEMA "http://myapp.com/schemas/order.xsd"
    ELEMENT "Order";
```

### 2. CLOB ストレージ (非構造化XML)

最適：可変構造のXML、主に全体を格納・取得するXML、レガシーデータ。セットアップが最も簡単。

```sql
-- CLOBとして格納される XMLType (非構造化)
CREATE TABLE contracts (
    contract_id   NUMBER PRIMARY KEY,
    contract_xml  XMLType
)
XMLTYPE COLUMN contract_xml STORE AS CLOB;

-- XMLの挿入
INSERT INTO contracts VALUES (
    1,
    XMLType('<Contract>
               <ContractId>1001</ContractId>
               <Parties>
                 <Party role="buyer">Acme Corp</Party>
                 <Party role="seller">Beta LLC</Party>
               </Parties>
               <EffectiveDate>2025-01-01</EffectiveDate>
               <Value currency="USD">50000</Value>
             </Contract>')
);

-- 文字列変数からの挿入
DECLARE
    v_xml CLOB := '<Contract><ContractId>1002</ContractId></Contract>';
BEGIN
    INSERT INTO contracts VALUES (1002, XMLType(v_xml));
    COMMIT;
END;
```

### 3. バイナリXMLストレージ (11g以降の推奨)

最適：スキーマ登録を行わない汎用的なXMLストレージ。パース後のバイナリ形式でコンパクトに格納される（JSONのOSONと概念的に似たアプローチ）。CLOBよりも高速で、オブジェクト・リレーショナルよりもシンプル。

```sql
CREATE TABLE product_specs (
    product_id  NUMBER PRIMARY KEY,
    spec_xml    XMLType
)
XMLTYPE COLUMN spec_xml STORE AS BINARY XML;

-- XMLの挿入
INSERT INTO product_specs VALUES (
    101,
    XMLType('<Specification>
               <ProductId>101</ProductId>
               <Name>Industrial Widget</Name>
               <Dimensions unit="mm">
                 <Width>150</Width>
                 <Height>75</Height>
                 <Depth>50</Depth>
               </Dimensions>
               <Materials>
                 <Material>Steel</Material>
                 <Material>Rubber</Material>
               </Materials>
             </Specification>')
);
```

---

## XMLTypeメソッドによるXPath抽出

```sql
-- XPath を使用して単一のノード値を抽出
SELECT EXTRACTVALUE(spec_xml, '/Specification/Name') AS product_name
FROM   product_specs;

-- XMLフラグメントを抽出 (XMLType を返す)
SELECT EXTRACT(spec_xml, '/Specification/Dimensions') AS dimensions_xml
FROM   product_specs;

-- existsNode: ノードの存在を確認
SELECT product_id
FROM   product_specs
WHERE  EXISTSNODE(spec_xml, '/Specification/Materials/Material[text()="Steel"]') = 1;

-- XMLQuery: XMLType を返す XQuery 評価
SELECT XMLQuery('$x/Specification/Name/text()'
                PASSING spec_xml AS "x"
                RETURNING CONTENT) AS name_value
FROM   product_specs;
```

注意: `EXTRACTVALUE`, `EXTRACT`, `EXISTSNODE` は 11g 以降で非推奨。`XMLQUERY`, `XMLEXISTS`, `XMLTABLE` の使用が推奨される。

---

## XMLTable: XMLをリレーショナルな行に変換 (シュレッド)

`XMLTable` は、XMLドキュメントをリレーショナルデータに変換するための最新かつ推奨される方法である。XQueryパス式を使用してドキュメントをナビゲートする。

```sql
-- 基本的な XMLTable の使用
SELECT x.product_id_val, x.product_name, x.width, x.height
FROM   product_specs p,
       XMLTable('/Specification'
           PASSING p.spec_xml
           COLUMNS
               product_id_val  NUMBER         PATH 'ProductId',
               product_name    VARCHAR2(200)  PATH 'Name',
               width           NUMBER         PATH 'Dimensions/Width',
               height          NUMBER         PATH 'Dimensions/Height'
       ) x;

-- 繰り返し要素の展開 (配列のような扱い)
SELECT p.product_id, m.material_name
FROM   product_specs p,
       XMLTable('/Specification/Materials/Material'
           PASSING p.spec_xml
           COLUMNS
               material_name  VARCHAR2(100)  PATH '.'
       ) m;

-- 属性値の抽出
SELECT x.currency, x.value
FROM   contracts c,
       XMLTable('/Contract/Value'
           PASSING c.contract_xml
           COLUMNS
               currency  VARCHAR2(3)     PATH '@currency',  -- 属性には @ を使用
               value     NUMBER(15,2)    PATH '.'
       ) x;

-- 階層データのためのネストされた XMLTable
SELECT p.product_id, outer_x.dim_unit, inner_x.dim_name, inner_x.dim_value
FROM   product_specs p,
       XMLTable('/Specification/Dimensions'
           PASSING p.spec_xml
           COLUMNS
               dim_unit  VARCHAR2(10)  PATH '@unit',
               dim_xml   XMLType       PATH '.'
       ) outer_x,
       XMLTable('/*'
           PASSING outer_x.dim_xml
           COLUMNS
               dim_name   VARCHAR2(50)  PATH 'fn:name(.)',
               dim_value  NUMBER        PATH '.'
       ) inner_x;
```

### 名前空間を伴う XMLTable

```sql
-- 名前空間宣言を含む XML
SELECT x.order_id, x.customer
FROM   XMLTable(
           XMLNAMESPACES('http://orders.example.com' AS "ord"),
           '/ord:OrderSet/ord:Order'
           PASSING XMLType(
               '<OrderSet xmlns="http://orders.example.com">
                  <Order><OrderId>1</OrderId><Customer>Acme</Customer></Order>
                  <Order><OrderId>2</OrderId><Customer>Beta</Customer></Order>
                </OrderSet>'
           )
           COLUMNS
               order_id  NUMBER        PATH 'ord:OrderId',
               customer  VARCHAR2(100) PATH 'ord:Customer'
       ) x;
```

---

## XMLの生成: XMLElement, XMLForest, XMLAgg

Oracleは、リレーショナルデータからXMLを生成する関数を提供している。

```sql
-- XMLElement: 値からXML要素を作成
SELECT XMLElement("Employee",
           XMLElement("Name", first_name || ' ' || last_name),
           XMLElement("Department", department_id),
           XMLElement("Salary", salary)
       ).getClobVal() AS employee_xml
FROM   employees
WHERE  department_id = 10;

-- XMLForest: 列から要素のシーケンスを作成
SELECT XMLElement("Employee",
           XMLForest(
               employee_id AS "Id",
               first_name  AS "FirstName",
               last_name   AS "LastName",
               hire_date   AS "HireDate"
           )
       ).getClobVal() AS emp_xml
FROM   employees;

-- XMLAttributes: 要素に属性を追加
SELECT XMLElement("Product",
           XMLAttributes(
               product_id AS "id",
               'active'   AS "status"
           ),
           XMLForest(
               product_name AS "Name",
               list_price   AS "Price"
           )
       ).getClobVal() AS product_xml
FROM   products;

-- XMLAgg: 複数のXML要素を1つの親要素に集約
SELECT XMLElement("Department",
           XMLAttributes(department_id AS "id"),
           XMLAgg(
               XMLElement("Employee",
                   XMLForest(
                       employee_id AS "Id",
                       first_name  AS "Name"
                   )
               )
               ORDER BY last_name
           )
       ).getClobVal() AS dept_xml
FROM   employees
GROUP  BY department_id;
```

### XMLRoot と XMLDocument

```sql
-- XML宣言とルート要素を追加
SELECT XMLRoot(
           XMLElement("Employees",
               XMLAgg(XMLElement("Employee", first_name || ' ' || last_name))
           ),
           VERSION '1.0',
           STANDALONE YES
       ).getClobVal() AS xml_doc
FROM   employees;
```

---

## XMLQuery と XMLExists による XQuery

```sql
-- XMLQuery: XQuery 式を実行し、XMLType を返す
SELECT XMLQuery(
           'for $o in /OrderSet/Order
            where $o/Status = "PENDING"
            return $o/OrderId'
           PASSING order_xml
           RETURNING CONTENT
       ).getStringVal() AS pending_ids
FROM   order_documents;

-- XMLExists: XQuery 述語によるテスト
SELECT order_id
FROM   order_documents
WHERE  XMLExists(
           'for $o in /Order/Items/Item
            where $o/Price > 100
            return $o'
           PASSING order_xml
       );

-- XQuery FLWOR 式
SELECT XMLQuery(
           'for $i in /Specification/Materials/Material
            order by $i
            return <mat>{$i/text()}</mat>'
           PASSING spec_xml
           RETURNING CONTENT
       ).getClobVal() AS sorted_materials
FROM   product_specs
WHERE  product_id = 101;
```

---

## XMLの索引

### XMLIndex (構造化および非構造化)

```sql
-- 非構造化 XMLIndex: すべてのテキストノードと属性値を索引付け
CREATE INDEX idx_contracts_xml
    ON contracts (contract_xml)
    INDEXTYPE IS XDB.XMLIndex;

-- 構造化 XMLIndex: ターゲットのパフォーマンス向上のために特定のパスを索引付け
CREATE INDEX idx_contracts_structured
    ON contracts (contract_xml)
    INDEXTYPE IS XDB.XMLIndex
    PARAMETERS ('PATHS (
        path (/Contract/ContractId)
        path (/Contract/EffectiveDate)
        path (/Contract/Value)
    )');

-- XML索引の削除
DROP INDEX idx_contracts_xml;
```

### 一般的な XPath 用のファンクション索引

```sql
-- 特定のスカラーパスを常に照会する場合
CREATE INDEX idx_spec_product_name
    ON product_specs (
        XMLCast(XMLQuery('/Specification/Name/text()'
                         PASSING spec_xml RETURNING CONTENT)
                AS VARCHAR2(200))
    );

-- 索引を活用するために同じ式を使用して照会
SELECT product_id
FROM   product_specs
WHERE  XMLCast(XMLQuery('/Specification/Name/text()'
                        PASSING spec_xml RETURNING CONTENT)
               AS VARCHAR2(200)) = 'Industrial Widget';
```

---

## Oracle XML DB (XMLDB) リポジトリ

XMLDBには、WebDAVやFTPプロトコル経由でアクセス可能な階層型リポジトリが含まれており、XMLドキュメントをデータベース内のフォルダのような構造で格納できる。

```sql
-- XMLDBリポジトリにフォルダを作成
CALL DBMS_XDB.CREATEFOLDER('/public/contracts');

-- XMLリソースを作成/格納
DECLARE
    v_result BOOLEAN;
BEGIN
    v_result := DBMS_XDB.CREATERESOURCE(
        ABSPATH => '/public/contracts/contract_1001.xml',
        DATA    => XMLTYPE(
            '<Contract>
               <ContractId>1001</ContractId>
               <Status>Active</Status>
             </Contract>'
        )
    );
    IF NOT v_result THEN
        RAISE_APPLICATION_ERROR(-20001, 'Resource already exists');
    END IF;
    COMMIT;
END;

-- RESOURCE_VIEW を使用してリポジトリ内の XML ファイルを照会
SELECT PATH(1) AS file_path, XMLType(RES) AS resource_xml
FROM   RESOURCE_VIEW
WHERE  UNDER_PATH(RES, '/public/contracts', 1) = 1;

-- リポジトリドキュメントの内容を照会
SELECT x.contract_id, x.status
FROM   RESOURCE_VIEW rv,
       XMLTable('/Contract'
           PASSING XMLType(rv.RES)
           COLUMNS
               contract_id  NUMBER        PATH 'ContractId',
               status       VARCHAR2(20)  PATH 'Status'
       ) x
WHERE  UNDER_PATH(rv.RES, '/public/contracts', 1) = 1;
```

---

## XML型の変換とユーティリティ

```sql
-- XMLType を CLOB に変換
SELECT spec_xml.getClobVal() AS xml_clob FROM product_specs;

-- XMLType を VARCHAR2 に変換 (十分に小さい場合)
SELECT spec_xml.getStringVal() AS xml_string FROM product_specs WHERE product_id = 101;

-- VARCHAR2/CLOB を XMLType に変換
SELECT XMLType('<root><value>42</value></root>') AS xml_val FROM DUAL;

-- 登録済みスキーマに対して XML を検証
SELECT spec_xml.isSchemaValid('http://myapp.com/schemas/product.xsd') AS is_valid
FROM   product_specs;

-- XSLT による変換
SELECT XMLType('<data><item>Hello</item></data>').transform(
    XMLType('<?xml version="1.0"?>
             <xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform" version="1.0">
               <xsl:template match="/">
                 <result><xsl:value-of select="/data/item"/></result>
               </xsl:template>
             </xsl:stylesheet>')
).getClobVal() AS transformed
FROM   DUAL;

-- XMLの整形表示
SELECT XMLSerialize(DOCUMENT spec_xml INDENT SIZE = 2) AS formatted_xml
FROM   product_specs;
```

---

## ベスト・プラクティス

- 特段の理由がない限り、新しい XMLType 列には **バイナリXMLストレージ**（CLOBやオブジェクト・リレーショナルではなく）を選択すること。バイナリXMLは、ストレージ効率、クエリ・パフォーマンス、および柔軟性のバランスが最も優れている。
- 新規開発では、すべての箇所で（非推奨の EXTRACTVALUE/EXTRACT ではなく）**XMLTable および XMLQuery** を使用すること。
- 頻繁に照会されるパスについては、構造化 XMLIndex またはファンクション索引を使用して、**特定の XPath 式に索引を付ける**こと。
- 1つのフィールドを抽出するために、XMLドキュメント全体をアプリケーション・コードに取得することを避ける。データベース・レベルでシュレッドするために XMLTable/XMLQuery を使用すること。
- XML出力を生成する際は、**XMLSerialize** を使用してシリアライズを制御すること。これにより、エンコーディング、インデント、および名前空間宣言が正しく処理される。
- **巨大なXMLドキュメント（1MB超）**の場合は、CLOBストレージを検討し、ドキュメント全体をメモリーにロードするのではなく `DBMS_XMLPARSER` ストリーミング API で処理することを検討する。
- 構造エラーを早期に発見するために、スキーマ登録または `IS JSON` に相当する機能 (`XMLTYPE ... VALIDATING`) を使用して、**挿入時に XML を検証**すること。

---

## よくある間違い

### 間違い 1: 非推奨の関数の使用

```sql
-- 非推奨: 新規コードでは避けること
SELECT EXTRACTVALUE(xml_col, '/Root/Value') FROM t;
SELECT EXTRACT(xml_col, '/Root/Child') FROM t;

-- 推奨
SELECT XMLCast(XMLQuery('/Root/Value/text()' PASSING xml_col RETURNING CONTENT) AS VARCHAR2(100)) FROM t;
SELECT XMLQuery('/Root/Child' PASSING xml_col RETURNING CONTENT) FROM t;
```

### 間違い 2: 頻繁に照会される XML への CLOB ストレージの使用

CLOBとして格納すると、XPathクエリが実行されるたびにドキュメントを最初からパースする必要がある。頻繁に照会されるXMLには、バイナリXMLまたは XMLIndex を伴うオブジェクト・リレーショナル・ストレージを使用すること。

### 間違い 3: 名前空間の処理漏れ

XMLTable/XMLQuery で名前空間の宣言を忘れると、結果セットが黙って空になる。Oracleは、名前空間で修飾されたパスが一致しない場合、エラーではなく「行なし」を返すためである。

```sql
-- 誤り: 名前空間を無視し、何も返さない
SELECT * FROM XMLTable('/Order/Id' PASSING namespace_xml_col COLUMNS id NUMBER PATH '.');

-- 正解: 名前空間を宣言する
SELECT * FROM XMLTable(
    XMLNAMESPACES('http://orders.com/v1' AS "o"),
    '/o:Order/o:Id'
    PASSING namespace_xml_col
    COLUMNS id NUMBER PATH '.'
);
```

### 間違い 4: XMLType を = で比較する

XMLType は `=` で比較できない。XMLExists や XMLQuery を使用するか、最初にスカラー値を抽出してから比較すること。

```sql
-- 誤り
WHERE spec_xml = XMLType('<Specification>...')

-- 正解: 抽出した値を比較する
WHERE XMLCast(XMLQuery('/Specification/ProductId/text()'
                        PASSING spec_xml RETURNING CONTENT) AS NUMBER) = 101
```

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c XML Developer's Kit プログラマーズ・ガイド (ADXDK)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adxdk/)
- [Oracle Database 19c SQL言語リファレンス — XML関数](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
- [Oracle XML DB デベロッパーズ・ガイド (ADXDB)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adxdb/)
