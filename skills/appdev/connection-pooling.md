# Oracle Databaseにおける接続プール

## 概要

接続プール（コネクション・プーリング）は、Oracleアプリケーション開発における最も重要なパフォーマンス手法の1つである。データベース接続の確立はコストが高い。ネットワーク・ラウンドトリップ、認証、セッション状態の初期化、およびクライアントとサーバーの両方におけるメモリー割り当てが含まれるためである。接続プールは、あらかじめ確立された接続のセットを維持し、アプリケーションが借用、使用、および返却できるようにすることで、このコストを分散（償却）させる。

Oracleは主に2つのプール・アーキテクチャを提供している：

- **Universal Connection Pool (UCP)** — アプリケーション層によって管理されるクライアント側のプール。JDBC、Python (python-oracledb)、およびNode.js (node-oracledb)で利用可能。
- **Database Resident Connection Pooling (DRCP)** — データベース内部で管理されるサーバー側のプール。ステートレスなアプリケーション・サーバーからの数千の短寿命な接続に最適。

---

## Universal Connection Pool (UCP)

UCPはOracleのクライアント側接続プールである。物理的なデータベース接続のプールを維持する、純粋なJavaライブラリ（JDBC用）である。プールは単一のJVMプロセス内（または特定の構成ではプロセス間）で共有される。

### 主要なプール・パラメータ

| パラメータ | 説明 | 一般的な値 |
|---|---|---|
| `initialPoolSize` | プール起動時に作成される接続数 | 5–10 |
| `minPoolSize` | 維持される最小接続数 | 5 |
| `maxPoolSize` | 接続数のハード上限 | 20–100 |
| `connectionWaitTimeout` | 空き接続を待機する秒数 | 30 |
| `inactiveConnectionTimeout` | アイドル接続がクローズされるまでの秒数 | 300 |
| `validateConnectionOnBorrow` | 借用ごとに検証クエリを実行するか | true (開発), false (本番では `isValid` を使用) |
| `abandonedConnectionTimeout` | N秒以上保持されている接続を回収する | 120–600 |
| `timeToLiveConnectionTimeout` | 接続の最大有効期間 | 3600 |

### JDBC / UCP の例

```java
import oracle.ucp.jdbc.PoolDataSourceFactory;
import oracle.ucp.jdbc.PoolDataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class UCPExample {

    private static PoolDataSource pool;

    static {
        try {
            pool = PoolDataSourceFactory.getPoolDataSource();
            pool.setConnectionFactoryClassName("oracle.jdbc.pool.OracleDataSource");
            pool.setURL("jdbc:oracle:thin:@//db-host:1521/MYSERVICE");
            pool.setUser("app_user");
            pool.setPassword("secret");

            // プールのサイズ設定
            pool.setInitialPoolSize(10);
            pool.setMinPoolSize(5);
            pool.setMaxPoolSize(50);

            // タイムアウト設定 (秒)
            pool.setConnectionWaitTimeout(30);
            pool.setInactiveConnectionTimeout(300);
            pool.setAbandonedConnectionTimeout(120);
            pool.setTimeToLiveConnectionTimeout(3600);

            // 検証設定
            pool.setValidateConnectionOnBorrow(false); // 代わりに isValid() を使用

        } catch (Exception e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public static void queryExample(int customerId) throws Exception {
        // プールへの返却を確実にするため、常に try-with-resources を使用する
        try (Connection conn = pool.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT customer_name, email FROM customers WHERE customer_id = ?")) {

            ps.setInt(1, customerId);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    System.out.println(rs.getString("customer_name"));
                }
            }
        } // ここで接続は自動的にプールに返却される
    }
}
```

### UCP と Easy Connect Plus

Oracle 19c以降では、**Easy Connect Plus** 構文がサポートされており、接続文字列にプールやタイムアウトのパラメータを直接埋め込むことができる：

```
jdbc:oracle:thin:@//db-host:1521/MYSERVICE?oracle.jdbc.ReadTimeout=60000&oracle.net.CONNECT_TIMEOUT=10000
```

ウォレットを使用したTNSエイリアス解決（例：Autonomous Database）の場合：

```java
pool.setURL("jdbc:oracle:thin:@mydb_high?TNS_ADMIN=/path/to/wallet");
```

---

## Database Resident Connection Pooling (DRCP)

DRCPは、プールをデータベース・サーバー自体に移動させる。**接続ブローカ（Connection Broker）**プロセスが、サーバー・プロセス（**プール・サーバー**と呼ばれる）のプールを管理する。アプリケーションが接続すると、プール・サーバーを借用して処理を実行し、サーバー側のプロセスを破棄することなく返却する。

### DRCP を使用すべき場面

- 数千の短寿命でステートレスな接続（例：PHP, Pythonスクリプト）
- 長寿命のJDBCプールを維持できない中間層サーバー
- データベース・サーバーのメモリー削減が重要な状況
- クライアント側プールと組み合わせて、両方の利点を活用する場合

### DRCP の有効化

```sql
-- 接続プールを開始する (SYSDBAとして実行)
EXECUTE DBMS_CONNECTION_POOL.START_POOL();

-- プール・パラメータを構成する
EXECUTE DBMS_CONNECTION_POOL.CONFIGURE_POOL(
    pool_name         => 'SYS_DEFAULT_CONNECTION_POOL',
    minsize           => 4,
    maxsize           => 40,
    incrsize          => 2,
    session_cached_cursors => 20,
    inactivity_timeout => 300,
    max_think_time    => 120,
    max_use_session   => 500000,
    max_lifetime_session => 86400
);

-- プールのステータスを確認
SELECT connection_pool, status, minsize, maxsize, num_open_servers,
       num_busy_servers, num_waiting_requests
FROM   v$cpool_stats;
```

### DRCP 接続文字列

接続記述子のサービス名に `:POOLED` を追加する：

```
# tnsnames.ora
MYAPP_DRCP =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = db-host)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = POOLED)
      (SERVICE_NAME = MYSERVICE)
    )
  )
```

または Easy Connect を使用する場合：

```
db-host:1521/MYSERVICE:POOLED
```

### Python (python-oracledb) と DRCP

```python
import oracledb

# Thinモード (Oracle Client不要) - UCPスタイルのプール
pool = oracledb.create_pool(
    user="app_user",
    password="secret",
    dsn="db-host:1521/MYSERVICE",
    min=2,
    max=10,
    increment=1,
    ping_interval=60,       # 60秒ごとに接続を検証
    timeout=300,            # 5分後にアイドル接続をクローズ
    getmode=oracledb.POOL_GETMODE_WAIT,
    wait_timeout=5000       # 接続待機のミリ秒数
)

def fetch_customer(customer_id: int) -> dict:
    with pool.acquire() as conn:
        with conn.cursor() as cur:
            cur.execute(
                "SELECT customer_name, email FROM customers WHERE customer_id = :id",
                id=customer_id
            )
            row = cur.fetchone()
            return {"name": row[0], "email": row[1]} if row else {}

# DRCP: dsnを変更するだけ
pool_drcp = oracledb.create_pool(
    user="app_user",
    password="secret",
    dsn="db-host:1521/MYSERVICE:POOLED",  # :POOLED サフィックスでDRCPを有効化
    min=1,
    max=5,
    increment=1
)
```

### Node.js (node-oracledb) プール

```javascript
const oracledb = require('oracledb');

async function initPool() {
    await oracledb.createPool({
        user: 'app_user',
        password: 'secret',
        connectString: 'db-host:1521/MYSERVICE',
        poolMin: 4,
        poolMax: 20,
        poolIncrement: 2,
        poolTimeout: 60,        // 60秒後にアイドル接続を破棄
        poolPingInterval: 60,   // 60秒ごとに借用した接続を検証
        stmtCacheSize: 30       // 接続ごとのプリペアド文キャッシュ
    });
}

async function queryExample(customerId) {
    let conn;
    try {
        conn = await oracledb.getConnection();  // デフォルトのプールから借用
        const result = await conn.execute(
            `SELECT customer_name, email
             FROM   customers
             WHERE  customer_id = :id`,
            { id: customerId },
            { outFormat: oracledb.OUT_FORMAT_OBJECT }
        );
        return result.rows;
    } finally {
        if (conn) await conn.close();  // プールに返却（破棄されない）
    }
}
```

---

## 接続文字列の形式

### Easy Connect (最もシンプル)

```
host[:port][/service_name][:server_type][/instance_name]
```

例：
```
db-host/ORCL
db-host:1521/MYSERVICE
db-host:1521/MYSERVICE:POOLED
scan-host:1521/MYSERVICE
```

### Easy Connect Plus (19c以降, パラメータ付き)

```
(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=db-host)(PORT=1521))
  (CONNECT_DATA=(SERVICE_NAME=MYSERVICE))
  (CONNECT_TIMEOUT=10)(RETRY_COUNT=3)(RETRY_DELAY=3))
```

またはURL形式の文字列：

```
db-host:1521/MYSERVICE?connect_timeout=10&retry_count=3
```

### TNS Names エントリ

```
# $ORACLE_HOME/network/admin/tnsnames.ora
MYSERVICE =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = primary-host)(PORT = 1521))
      (ADDRESS = (PROTOCOL = TCP)(HOST = standby-host)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = MYSERVICE)
      (SERVER = DEDICATED)
    )
    (LOAD_BALANCE = YES)
    (FAILOVER = YES)
  )
```

### JDBC Thin URL

```
jdbc:oracle:thin:@//host:port/service
jdbc:oracle:thin:@host:port:SID
jdbc:oracle:thin:@(DESCRIPTION=...)
jdbc:oracle:thin:@tns_alias?TNS_ADMIN=/path/to/tns
```

---

## プール・サイズ設定のガイドライン

プールのサイズ設定を誤ることは、アプリケーションのスケーラビリティ低下の最も一般的な原因である。

### データベース・プールに適用した「リトルの法則」

```
最適なプール・サイズ ≈ (スループット × 平均クエリ実行時間)
```

例：200リクエスト/秒、平均クエリ実行時間 50ms の場合：
```
プール・サイズ = 200 * 0.050 = 10 接続
```

### 実践的な経験則

- **小さく始める。** 10–20接続の方が200接続よりもパフォーマンスが良いことが多い。接続数が多い = データベース・サーバー上でのコンテキスト・スイッチが増加するためである。
- **全アプリケーション・インスタンスの最大接続数** は、データベースの `PROCESSES` パラメータからシステム・オーバーヘッドを引いた値を超えてはならない。
- **`v$cpool_stats` と `v$session` を監視** して、プールの枯渇を検知する。
- Oracle RACの場合、ノードごとの接続を考慮する。`maxPoolSize` は合計ではなく、インスタンスごとに設定する。

```sql
-- プログラム/サービスごとの現在のセッション数を確認
SELECT program, service_name, status, COUNT(*) AS session_count
FROM   v$session
WHERE  type = 'USER'
GROUP  BY program, service_name, status
ORDER  BY session_count DESC;

-- DRCPプールが飽和状態か確認
SELECT num_busy_servers, num_open_servers,
       num_waiting_requests, num_requests
FROM   v$cpool_stats;
```

---

## 接続の検証

プール内の接続は、ネットワーク障害、データベースの再起動、ファイアウォールのアイドル・タイムアウトなどによって無効になる（スタレ状態になる）可能性がある。検証戦略は以下の通り：

| 戦略 | コスト | 信頼性 |
|---|---|---|
| `isValid()` / 借用時のping | 低 (1 ラウンドトリップ) | 高 |
| SQL検証クエリ (`SELECT 1 FROM DUAL`) | 低 | 高 |
| ハートビート・スレッド | バックグラウンド | 中 |
| 検証なし | ゼロ | 低 (無効な接続の可能性) |

```java
// Java: 使用前に接続をテストする
try (Connection conn = pool.getConnection()) {
    if (!conn.isValid(5)) {  // 5秒のタイムアウト
        // UCPは自動的に壊れた接続を入れ替える
        throw new SQLException("Connection validation failed");
    }
    // 処理を続行
}
```

```python
# python-oracledb: ping_interval でバックグラウンド検証を制御
pool = oracledb.create_pool(
    ...,
    ping_interval=60,  # 60秒以上アイドル状態の接続を貸し出す前にpingを実行
    ping_timeout=5     # pingに5秒以上かかれば失敗とみなす
)
```

---

## ベスト・プラクティス

- **常に finally ブロックまたは try-with-resources 内で接続をクローズ/解放する。** リークした接続は、`abandonedConnectionTimeout` が発生するまでプールから永久に失われる。
- **接続オブジェクトをリクエスト間でキャッシュしない。** 単一のリクエストのライフサイクル内で借用、使用、および返却を行う。
- **`maxPoolSize` を保守的に設定する。** データベースのCPUコア数が飽和した後は、接続数が増えるほどパフォーマンスは悪化する。
- **SIDではなくサービスを使用する。** 名前付きサービスに接続することで、TAF (Transparent Application Failover) やロード・バランシングを利用できる。
- **文キャッシュ（Statement Caching）を有効にする。** プール内の各接続にプリペアド文をキャッシュさせることで、繰り返しの解析（パース）呼び出しを排除できる。
- **プール・メトリクスを監視する。** `connectionWaitTimeout` 例外が発生した場合はアラートを出すようにする。これはプールの枯渇を意味している。
- **OLTPとバッチ・ジョブで別々のプールを使用する。** バッチ・ワークロードが対話的なクエリを枯渇させるのを防ぐため。
- **クラウド/コンテナ環境** では、`inactiveConnectionTimeout` をクラウド・ロード・バランサのアイドル・タイムアウト（多くの場合4分）よりも短く設定し、サイレントな接続切断を避ける。

---

## よくある間違い

### 間違い 1: 接続をクローズしない

```java
// 不正解 — 接続がプールに返却されない
public void badQuery() throws Exception {
    Connection conn = pool.getConnection();
    // ... 処理 ...
    // conn.close() を忘れている！
}

// 正解 — try-with-resources により確実な返却が保証される
public void goodQuery() throws Exception {
    try (Connection conn = pool.getConnection()) {
        // ... 処理 ...
    }
}
```

### 間違い 2: プール・サイズが大きすぎる

10台のアプリケーション・サーバーそれぞれに `maxPoolSize=500` を設定すると、データベースに対して最大5,000の接続が発生する可能性がある。Oracleはそれぞれに対してサーバー・プロセス（または共有サーバー・スロット）を割り当てる必要があり、OSプロセスのオーバーヘッドでデータベースが押し潰される。

**修正案:** リトルの法則を使用して計算する。まずは `maxPoolSize = DBサーバーのCPUコア数 * 2` から始める。

### 間違い 3: `connectionWaitTimeout` 例外を無視する

プールが枯渇すると、スレッドは待機し、最終的にタイムアウト例外をスローする。多くの開発者はこの例外をキャッチして握りつぶしてしまう。代わりに、メトリクスとして公開し、アラートの対象とすべきである。

### 間違い 4: 状態を持つセッションで DRCP を使用する

DRCPは使用後にセッション状態をリセットする。アプリケーションが以下のような場合は DRCP を使用してはならない：
- `DBMS_SESSION.SET_CONTEXT` を使用し、その持続を期待している場合
- 同一接続上の複数のリクエスト間で一時表に依存している場合
- 維持が必要な `ALTER SESSION` 設定を使用している場合

### 間違い 5: 間違ったサービス vs SID

常に **SERVICE_NAME** に接続し、**SID** は使用しない。サービスは TAF、ロード・バランシング、および接続時フェイルオーバーをサポートしている。クライアント接続において SID は非推奨である。

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c アプリケーション・開発者ガイド (ADFNS)](https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/)
- [DBMS_CONNECTION_POOL — Oracle Database 19c PL/SQLパッケージおよびタイプ・リファレンス](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_CONNECTION_POOL.html)
- [V$CPOOL_STATS — Oracle Database 19c リファレンス](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-CPOOL_STATS.html)
- [Universal Connection Pool 開発者ガイド](https://docs.oracle.com/en/database/oracle/oracle-database/19/jjucp/)
- [python-oracledb Documentation](https://python-oracledb.readthedocs.io/en/latest/)
