# Oracle Databaseにおけるシーケンスとアイデンティティ列

## 概要

一意で単調増加する識別子を生成することは、リレーショナル・データベース設計における基本的な要件である。Oracleは以下の2つの仕組みを提供している：

- **シーケンス (Sequences)** — どの表からも独立して数値を生成する、スタンドアロンのスキーマ・オブジェクト
- **アイデンティティ列 (Identity Columns)** — シーケンス生成を表のDDLと組み合わせた列レベルの構文 (Oracle 12cで導入)

どちらも最終的には同じ内部シーケンス・エンジンを使用するが、アイデンティティ列の方がサロゲートキー（代替キー）を定義するための、よりクリーンで宣言的な方法を提供している。

---

## CREATE SEQUENCE: 全構文とオプション

```sql
CREATE SEQUENCE schema.sequence_name
    START WITH      initial_value        -- 最初に生成される値 (デフォルト: 1)
    INCREMENT BY    step_value           -- 正 = 昇順, 負 = 降順 (デフォルト: 1)
    MINVALUE        min_value            -- 下限値
    MAXVALUE        max_value            -- 上限値 (昇順のデフォルト: 10^27)
    NOCYCLE | CYCLE                      -- NOCYCLEは上限に達すると ORA-08004 を発生させる (デフォルト: NOCYCLE)
    CACHE n | NOCACHE                    -- n個の値をメモリーに事前割り当てする (デフォルト: 20)
    ORDER | NOORDER                      -- ORDERは生成順序を保証する (RAC環境でのみ重要)
    KEEP | NOKEEP                        -- セッション復元後のシーケンス動作に影響
    SCALE | NOSCALE;                     -- 23c: 一意性のためにインスタンス/シャード情報を付加
```

### 基本構成例

```sql
-- シンプルな主キー用シーケンス (最も一般的なパターン)
CREATE SEQUENCE seq_customer_id
    START WITH 1000
    INCREMENT BY 1
    MAXVALUE 9999999999
    NOCYCLE
    CACHE 100;

-- 降順シーケンス
CREATE SEQUENCE seq_priority_desc
    START WITH 10000
    INCREMENT BY -1
    MINVALUE 1
    NOCYCLE
    CACHE 50;

-- 偶数を生成するステップ・シーケンス
CREATE SEQUENCE seq_even_numbers
    START WITH 2
    INCREMENT BY 2
    MAXVALUE 1000000
    NOCYCLE
    CACHE 20;
```

### シーケンスの使用

```sql
-- NEXTVAL: シーケンスを進め、新しい値を返す
-- CURRVAL: このセッションで直前に NEXTVAL で生成された値を返す (進めない)

-- INSERT 文での使用
INSERT INTO customers (customer_id, name, email)
VALUES (seq_customer_id.NEXTVAL, 'Acme Corp', 'admin@acme.com');

-- PL/SQL で生成された値を取得する
DECLARE
    v_new_id customers.customer_id%TYPE;
BEGIN
    INSERT INTO customers (customer_id, name, email)
    VALUES (seq_customer_id.NEXTVAL, 'Beta LLC', 'info@beta.com')
    RETURNING customer_id INTO v_new_id;

    -- 生成された v_new_id を子レコードの挿入に使用する
    INSERT INTO customer_contacts (contact_id, customer_id, contact_name)
    VALUES (seq_contacts.NEXTVAL, v_new_id, 'John Smith');

    COMMIT;
END;

-- SELECT 文での使用 (表を使わずに連番を生成するのに便利)
SELECT seq_customer_id.NEXTVAL FROM DUAL;

-- NEXTVAL 呼び出し後の同一セッション内での CURRVAL
SELECT seq_customer_id.CURRVAL FROM DUAL;
```

### シーケンスの変更と削除

```sql
-- キャッシュ・サイズの変更
ALTER SEQUENCE seq_customer_id CACHE 500;

-- 新しい値から再開する (データ移行後に便利)
ALTER SEQUENCE seq_customer_id RESTART START WITH 50000;  -- 18c以降

-- 18cより前のバージョン: INCREMENT を一時的に大きくして再開値を調整する
SELECT last_number FROM user_sequences WHERE sequence_name = 'SEQ_CUSTOMER_ID';
-- 現在が 1000 で 50000 から再開したい場合、increment を 49000 に設定
ALTER SEQUENCE seq_customer_id INCREMENT BY 49000;
SELECT seq_customer_id.NEXTVAL FROM DUAL;  -- これで 50000 に到達
ALTER SEQUENCE seq_customer_id INCREMENT BY 1;

-- シーケンスの削除
DROP SEQUENCE seq_customer_id;
```

---

## CACHE オプション: パフォーマンスへの影響

`CACHE` は、パフォーマンスにおいて最も影響力のあるシーケンス・パラメータである。Oracleは `CACHE` 分の値をSGAに事前割り当てする。キャッシュされたシーケンスのインクリメントは純粋なメモリー内操作であり、非常に高速である。キャッシュを使用しない場合 (`NOCACHE`)、`NEXTVAL` を呼び出すたびに `seq$` システム表への書き込みが発生する。

### パフォーマンス比較

| 構成 | NEXTVAL のコスト | 備考 |
|---|---|---|
| `NOCACHE` | 約1回のredoログ書き込み | 約100倍遅い。OLTPでは避けること |
| `CACHE 20` (デフォルト) | メモリー内インクリメント | 低ボリュームに適している |
| `CACHE 100` | メモリー内インクリメント | 中程度のOLTPに推奨 |
| `CACHE 1000+` | メモリー内インクリメント | 大量挿入、高ボリューム向け |

```sql
-- AWR でシーケンスのキャッシュ・パフォーマンスを確認
SELECT object_name, value AS total_waits
FROM   v$sesstat st
JOIN   v$statname sn ON st.statistic# = sn.statistic#
WHERE  sn.name = 'sequence misses'
  AND  st.sid = SYS_CONTEXT('USERENV', 'SID');
```

### インスタンス再起動時のキャッシュ喪失

データベースが再起動すると、キャッシュされていたが使用されていなかったシーケンス値は**破棄される**。例えば、1000個キャッシュし、50個使用した状態で再起動すると、次のセッションは値 1051 から開始され、950個の欠番が生じる。これは正常な動作であり、設計通りの挙動である。アプリケーションが欠番を許容できない場合は、CACHEを使用できず、パフォーマンス上の不利益を被ることになる。

```sql
-- 再起動時に失われる可能性のある値の数を確認
SELECT sequence_name, last_number, cache_size,
       last_number + cache_size - 1 AS max_cached_value,
       (cache_size - 1) AS max_possible_gap
FROM   user_sequences
WHERE  sequence_name = 'SEQ_CUSTOMER_ID';
```

---

## シーケンスにおける欠番 (Gaps)

シーケンス値は設計上、**欠番（ギャップ）が発生しないことを保証していない**。欠番は以下の理由で発生する：

1. **再起動時のキャッシュ喪失** — 使用されなかったキャッシュ値が破棄される
2. **ロールバック** — `NEXTVAL` はロールバックされない。INSERT がロールバックされても消費された値は戻らない
3. **CYCLE** — シーケンスが上限に達してラップアラウンドした場合
4. **DELETE** — 行（およびそのシーケンス値）が削除された場合

```sql
-- シーケンス値はロールバックされない
BEGIN
    INSERT INTO orders (order_id) VALUES (seq_orders.NEXTVAL);  -- 1001を使用
    ROLLBACK;  -- INSERT は取り消されるが、1001は失われたまま
END;

BEGIN
    INSERT INTO orders (order_id) VALUES (seq_orders.NEXTVAL);  -- 1002を使用
    COMMIT;
END;
-- 欠番: orders 表に 1001 は存在しない
```

**欠番のない番号付けが必要な場合:** 行ロックを伴う専用の表や `DBMS_LOCK` によるシリアライズ（直列化）など、別の仕組みを使用する必要がある。しかし、これはほとんどの場合において間違いである。欠番のないシーケンスは直列実行を必要とし、パフォーマンスのボトルネックになるためである。税務コンプライアンスのための請求書番号など、ビジネス上「欠番なし」が必要な場合は、コミット後の別の番号付けステップを用意すべきである。

---

## ORDER vs. NOORDER (RAC 環境)

Oracle RAC (Real Application Clusters) 環境では、複数のインスタンスがそれぞれのキャッシュから独立してシーケンス値を生成できる。`NOORDER`（デフォルト）の場合、ノード間で値の生成順序は保証されない。例えば、ノード1が 100–199、ノード2が 200–299 を生成し、リクエストが 150, 250, 160, 210 のように入り混じって取得される可能性がある。

`ORDER` を指定すると、Oracleはクラスター・インターコネクトを通じてシリアライズ（直列化）を行い、すべてのRACノードにわたって厳密な生成順序を保証する。これには重大なパフォーマンスへの影響がある。

```sql
-- ORDER シーケンス (RAC で厳密な順序がどうしても必要な場合のみ)
CREATE SEQUENCE seq_ordered_id
    ORDER
    CACHE 10;  -- ORDER の場合はキャッシュを小さくするのが有益。大きいと値を浪費する

-- RAC 上のほとんどのアプリケーションでは、NOORDER + CACHE が正しい選択である
CREATE SEQUENCE seq_fast_id
    NOORDER
    CACHE 500;
```

---

## アイデンティティ列 (Identity Columns) (12c以降)

アイデンティティ列は、自動インクリメントされる主キーを定義するための現代的な方法である。Oracleは内部で管理されるシーケンスを自動的に作成・管理する。

### GENERATED ALWAYS

列の値は**常に**Oracleによって生成される。その列に対して直接 INSERT や UPDATE を行うことはできない。

```sql
CREATE TABLE employees (
    employee_id   NUMBER GENERATED ALWAYS AS IDENTITY
                      (START WITH 1000 INCREMENT BY 1 CACHE 100),
    first_name    VARCHAR2(50) NOT NULL,
    last_name     VARCHAR2(50) NOT NULL,
    hire_date     DATE DEFAULT SYSDATE NOT NULL,
    CONSTRAINT pk_employees PRIMARY KEY (employee_id)
);

-- 正しい INSERT: アイデンティティ列を省略する
INSERT INTO employees (first_name, last_name) VALUES ('Jane', 'Doe');

-- 誤り: 明示的な値の指定は拒否される
INSERT INTO employees (employee_id, first_name, last_name)
VALUES (9999, 'Jane', 'Doe');
-- ORA-32795: alwaysとして生成されるアイデンティティ列に値を挿入できません
```

### GENERATED BY DEFAULT

列が省略された場合にOracleが値を生成する。明示的に値を指定して上書きすることも可能である。

```sql
CREATE TABLE products (
    product_id    NUMBER GENERATED BY DEFAULT AS IDENTITY,
    product_name  VARCHAR2(100) NOT NULL,
    sku           VARCHAR2(50)  NOT NULL UNIQUE
);

-- Oracle が ID を生成する
INSERT INTO products (product_name, sku) VALUES ('Widget Pro', 'WGT-001');

-- 明示的な値で上書きする (データ移行などに便利)
INSERT INTO products (product_id, product_name, sku) VALUES (9999, 'Legacy Widget', 'WGT-LEG');
```

### GENERATED BY DEFAULT ON NULL

`GENERATED BY DEFAULT` と同様だが、明示的に `NULL` が挿入された場合にも値を生成する：

```sql
CREATE TABLE sessions (
    session_id    NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY,
    session_token VARCHAR2(64) NOT NULL,
    created_at    TIMESTAMP DEFAULT SYSTIMESTAMP
);

-- NULL の挿入により値の生成がトリガーされる
INSERT INTO sessions (session_id, session_token) VALUES (NULL, 'abc123');
-- Oracle は NULL を挿入する代わりに session_id を生成する
```

### 生成されたアイデンティティ値の取得

```sql
-- PL/SQL で RETURNING を使用する
DECLARE
    v_new_id employees.employee_id%TYPE;
BEGIN
    INSERT INTO employees (first_name, last_name)
    VALUES ('Bob', 'Smith')
    RETURNING employee_id INTO v_new_id;

    DBMS_OUTPUT.PUT_LINE('New employee ID: ' || v_new_id);
    COMMIT;
END;
```

```java
// JDBC での例
String sql = "INSERT INTO employees (first_name, last_name) VALUES (?, ?)";
try (PreparedStatement ps = conn.prepareStatement(sql, new String[]{"EMPLOYEE_ID"})) {
    ps.setString(1, "Bob");
    ps.setString(2, "Smith");
    ps.executeUpdate();

    try (ResultSet generatedKeys = ps.getGeneratedKeys()) {
        if (generatedKeys.next()) {
            long newId = generatedKeys.getLong(1);
            System.out.println("Generated ID: " + newId);
        }
    }
}
```

### 内部シーケンスの確認

```sql
-- アイデンティティ列には Oracle によって管理される内部シーケンスがある
SELECT column_name, identity_column, data_default,
       generation_type, sequence_name, increment_by, start_with, cache_size
FROM   user_tab_identity_cols
WHERE  table_name = 'EMPLOYEES';
```

---

## SYS_GUID() vs. シーケンスによる UUID

一意の識別子を生成するもう1つの方法として、Oracleは `SYS_GUID()` (System Globally Unique Identifier) を提供している。SYS_GUID は 16バイトの RAW 値（32文字の16進数として表示）を返す。

### 比較

| 特徴 | シーケンス | SYS_GUID() |
|---|---|---|
| データ型 | NUMBER | RAW(16) |
| 値のサイズ | 約10バイト | 16バイト |
| 索引の効率 | 非常に優れている (単調) | 低い (ランダム。索引の断片化を引き起こす) |
| グローバルな一意識 | DB内のみ | はい |
| 人間への可読性 | あり | なし (16進数文字列) |
| パフォーマンス | 非常に優れている (キャッシュ時) | 良好 (ただしランダム) |
| DB間でのデータ統合 | 競合の可能性あり | 一意性が保たれる |

```sql
-- SYS_GUID() の使用例
CREATE TABLE distributed_events (
    event_id    RAW(16) DEFAULT SYS_GUID() PRIMARY KEY,
    event_type  VARCHAR2(50),
    payload     CLOB,
    created_at  TIMESTAMP DEFAULT SYSTIMESTAMP
);

INSERT INTO distributed_events (event_type, payload)
VALUES ('ORDER_PLACED', '{"order_id": 1001}');

-- 可読性の高い UUID として表示
SELECT RAWTOHEX(event_id) AS event_id_hex,
       REGEXP_REPLACE(RAWTOHEX(event_id),
           '([A-F0-9]{8})([A-F0-9]{4})([A-F0-9]{4})([A-F0-9]{4})([A-F0-9]{12})',
           '\1-\2-\3-\4-\5') AS uuid_formatted,
       event_type
FROM   distributed_events;
```

### 標準的な UUID (VARCHAR2) 方式

IDを外部に公開するREST APIなどで RFC 4122 UUID 文字列が必要な場合：

```sql
-- チェック制約付きの VARCHAR2 として格納
CREATE TABLE api_resources (
    resource_id  VARCHAR2(36) DEFAULT LOWER(
                     REGEXP_REPLACE(RAWTOHEX(SYS_GUID()),
                         '([A-F0-9]{8})([A-F0-9]{4})([A-F0-9]{4})([A-F0-9]{4})([A-F0-9]{12})',
                         '\1-\2-\3-\4-\5')) PRIMARY KEY,
    resource_type VARCHAR2(50),
    data          CLOB
);

-- あるいは、アプリケーションで生成して渡す
INSERT INTO api_resources (resource_id, resource_type, data)
VALUES (LOWER(REGEXP_REPLACE(RAWTOHEX(SYS_GUID()),
           '([A-F0-9]{8})([A-F0-9]{4})([A-F0-9]{4})([A-F0-9]{4})([A-F0-9]{12})',
           '\1-\2-\3-\4-\5')),
       'ORDER', '{}');
```

### 使い分けの基準

- **シーケンス / アイデンティティ列**: ほとんどのアプリケーションにおける主キーのデフォルト。高速でコンパクト、索引との相性が良い。
- **SYS_GUID() / UUID**: レコードが複数のデータベース間でマージされる場合、またはAPIでIDが公開される場合（値が連番でないため、列挙攻撃を回避できる）、あるいは挿入前にデータベース外でIDを生成する必要がある場合に使用する。

**重要**: ランダムな UUID を主キーにすると、**索引の断片化**が発生する。新しい UUID が Bツリー索引のランダムな位置に挿入されるため、頻繁なページ・スプリット（ページ分割）が起こる。挿入の多い表では、単調増加するシーケンス値と比較して、パフォーマンスが大幅に低下する可能性がある。データベース内部のキーにはシーケンスを使用し、外部向けのAPIキーにはUUIDを使用することを検討すべきである。

---

## ベスト・プラクティス

- **OLTPシーケンスには `CACHE 100` 以上を使用する。** デフォルトの 20 は、挿入の多いシステムでは低すぎる。
- **新しい表には、手動のシーケンス挿入よりもアイデンティティ列を優先する。** 構文がよりクリーンで、意図が明確になる。
- **シーケンスに欠番がないと仮定しない。** 欠番を許容できるようにアプリケーションを設計すること。ビジネス・ルールで連番が必要な場合は、コミット後の別のステップで生成する。
- **特段の理由がない限り `GENERATED ALWAYS` を使用する。** 誤った手動挿入を防ぐことができる。
- **大量データのロード時**は、ロード前に `CACHE` を非常に大きく（例：`CACHE 10000`）設定し、終了後に元に戻すことで、redoの競合を最小限に抑えることができる。
- **RAC環境**では、十分な CACHE を設定した `NOORDER` を使用すること。`ORDER` によるパフォーマンス・コストは、それに見合う価値があることは稀である。
- **`NOCYCLE` を避けるのではなく**、最大範囲を十分に確認すること。`CYCLE` による無言のラップアラウンドが発生すると、主キーの一意性が破壊される。

---

## よくある間違い

### 間違い 1: 「監査」目的で NOCACHE を使用する

監査用シーケンスで「欠番なし」を求めて `NOCACHE` を使用するケースがある。これは最悪の選択である。ロールバックによって欠番は結局発生するし、`NEXTVAL` ごとにデータ・ディクショナリへの書き込みが発生するため、大幅に遅くなる。

### 間違い 2: MAXVALUE なしで CYCLE を使用する

```sql
-- 危険: デフォルトの MAXVALUE は 10^27。CYCLE を指定すると、
-- いずれ 1 に戻り、既存の主キー値と衝突する。
CREATE SEQUENCE seq_bad CYCLE;  -- MAXVALUE が省略されている

-- 安全: 現実的な MAXVALUE を設定し、CYCLE を明示的に処理する。
-- あるいは単純に NOCYCLE (デフォルト) を使用し、枯渇を監視する。
```

### 間違い 3: LIMIT なしで大量に NEXTVAL を取得する

```sql
-- 100万行のシーケンス値を瞬時に生成してしまう。恐らく意図した動作ではない。
SELECT seq_orders.NEXTVAL FROM some_large_table;
```

### 間違い 4: セッションをまたいで CURRVAL を信頼する

`CURRVAL` は、**現在のセッション**で `NEXTVAL` によって生成された直近の値を返す。他のセッションや、再接続後には意味を持たない。

### 間違い 5: データのエクスポート/インポートでアイデンティティ列を忘れる

`GENERATED ALWAYS` を使用している場合、Data Pump のエクスポートにはシーケンスの定義も含まれる。インポート時にターゲット表に既にデータがある場合、シーケンスが重複した値を生成する可能性がある。インポート後は必ず `LAST_NUMBER` を確認し、シーケンスをリセットすること。

```sql
-- Data Pump インポート後、アイデンティティ・シーケンスを修正する
SELECT max(employee_id) FROM employees;  -- 例えば 5432 だった場合
ALTER TABLE employees MODIFY employee_id GENERATED ALWAYS AS IDENTITY (START WITH 5433);
```

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c SQL言語リファレンス — CREATE SEQUENCE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-SEQUENCE.html)
- [Oracle Database 19c SQL言語リファレンス — アイデンティティ列](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-TABLE.html)
- [Oracle Database 19c アプリケーション・開発者ガイド (ADFNS)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/)
