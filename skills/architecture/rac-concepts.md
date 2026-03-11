# Oracle Real Application Clusters (RAC) アーキテクチャ

## 概要

Oracle Real Application Clusters (RAC) は、複数の Oracle インスタンスが同一のデータベースを同時にマウントおよびオープンできるようにする、共有ディスク・クラスタ・データベース・テクノロジーである。各ノードは独自の Oracle インスタンス（メモリー構造 + バックグラウンド・プロセス）を実行するが、すべてのノードは共有ストレージ（ASM またはクラスタ・ファイル・システム）に格納された同一のデータファイルを共有する。RAC は、インスタンスの生存性による高可用性と、ノード間でのワークロード分散による水平方向のスケーラビリティを提供する。

RAC は、Oracle の Maximum Availability Architecture (MAA) の要であり、通常はサイト内の HA（高可用性）のための RAC と、サイト間の DR（災害復旧）のための Data Guard と組み合わせて運用される。

---

## 1. コア・アーキテクチャ・コンポーネント

### 共有ストレージ

すべての RAC ノードは、同一のデータファイル、制御ファイル、REDO ログ・グループ、およびアーカイブ・ログにアクセスする。Oracle Automatic Storage Management (ASM) が推奨されるストレージ・レイヤーである。ASM は以下を提供する：

- I/O スループット向上のためのディスク間のストライピング
- ローカルな冗長性のためのミラーリング
- データベース以外のファイルのための ASM クラスタ・ファイル・システム (ACFS)

各ノードは、独自の ASM インスタンス (`+ASM1`, `+ASM2` など) を独立した Oracle インスタンス・タイプとして実行する。

### Oracle Clusterware (Grid Infrastructure)

Grid Infrastructure (GI) は、RAC の基盤となるクラスタ・ソフトウェア・スタックである。RAC データベースよりも前にインストールする必要がある。主要な GI コンポーネントは以下の通り：

| コンポーネント | 用途 |
|---|---|
| Oracle Cluster Registry (OCR) | クラスタ構成、リソース定義、および構成情報を格納する |
| 投票ディスク (Voting Disk) | スプリット・ブレイン発生時のノード・エビクション（追い出し）の判断に使用される |
| CSS (Cluster Synchronization Services) | ハートビートとメンバーシップ管理を行う |
| CRS (Cluster Ready Services) | リソース管理。DB、VIP、SCAN の起動/停止/監視を行う |
| CTSS (Cluster Time Synchronization Service) | NTP が使用されていない場合にノード間の時刻を同期させる |
| SCAN (Single Client Access Name) | 1～3 個の IP に解決される単一のホスト名。接続の負荷分散に使用される (Oracle は 3 個を推奨) |

### RAC 固有のインスタンス・コンポーネント

各 RAC インスタンスには、シングル・インスタンスにはない独自のバックグラウンド・プロセスがある：

| プロセス | 説明 |
|---|---|
| LMS (Lock Manager Server) | 他のノードからのブロック転送要求を処理する。インスタンスごとに複数の LMS プロセスが存在する |
| LMD (Lock Manager Daemon) | エンキュー要求を管理する。リモートの LMD と通信する |
| LCK (Lock Process) | インスタンス・ロック (非 PCM ロック) を処理する |
| LMON (Global Enqueue Service Monitor) | クラスタの再構成イベントを監視し、グローバル・エンキューとインスタンスのリカバリを処理する |
| DIAG (Diagnosability Process) | グローバル・リソースの問題に関する診断データをキャプチャする |
| RMSn (RAC Management Server) | クラスタ内の Oracle リソースを管理する |

---

## 2. キャッシュ・フュージョンとインターコネクト

キャッシュ・フュージョン（Cache Fusion）は、ディスクへの書き込みや再読み込み介さず、プライベートなインターコネクト経由でインスタンス間でデータ・ブロックを移動させる Oracle 独自のプロトコルである。共有ディスク型 RAC を実用的なものにする決定的なテクノロジーである。

### キャッシュ・フュージョンの仕組み

1. インスタンス 1 がディスクからブロック 42 を読み取り、自身のバッファ・キャッシュに格納する。
2. インスタンス 2 がブロック 42 を必要とする。ディスク I/O を行う代わりに、グローバル・キャッシュ・サービス (GCS) が、インスタンス 1 がブロック 42 を保持していることを特定する。
3. GCS はインスタンス 1 に対し、ブロック 42 をプライベート・インターコネクト経由で直接インスタンス 2 に送信するよう指示する。
4. インスタンス 2 は、ディスクにアクセスすることなく最新バージョンのブロックを受け取る。

このブロック転送は、受信側が読取り一貫性のために古いイメージを必要とする場合は **CR (Consistent Read) 転送**と呼ばれ、変更のために最新のコミット済みバージョンを必要とする場合は**現行 (Current) ブロック転送**と呼ばれる。

### プライベート・インターコネクト

インターコネクトは、キャッシュ・フュージョンのトラフィックとクラスタのハートビート専用に使用される、専用の低レイテンシ・広帯域ネットワークである。要件は以下の通り：

- **プライベート**ネットワークであること（インターコネクトのトラフィックをパブリック・インタフェース経由でルーティングしてはいけない）
- 目標レイテンシ: 往復 1ms 未満
- 帯域幅: 最低 10 GbE。高スループットのワークロードでは 25 GbE または InfiniBand を推奨
- 冗長性: ボンディングされた NIC（アクティブ/パッシブまたはアクティブ/アクティブ）を強く推奨

```sql
-- GV$ ビューからインターコネクト構成を確認
SELECT inst_id, name, ip_address, is_public
FROM   gv$cluster_interconnects
ORDER  BY inst_id, name;

-- インターコネクトの統計を確認
SELECT inst_id,
       gc_cr_blocks_received,
       gc_current_blocks_received,
       gc_cr_blocks_served,
       gc_current_blocks_served
FROM   gv$sysstat
WHERE  name IN ('gc cr blocks received', 'gc current blocks received',
                'gc cr blocks served',  'gc current blocks served')
ORDER  BY inst_id;
```

---

## 3. グローバル・キャッシュ・サービス (GCS) とグローバル・エンキュー・サービス (GES)

### グローバル・キャッシュ・サービス (GCS)

GCS は、すべてのインスタンスにおけるすべてのデータ・ブロックの状態を管理する。各ブロックには、その状態を追跡するマスター・インスタンスがある。GCS は、すべてのブロックに対して **PCM (Parallel Cache Management) ロック**と呼ばれる分散ロックを維持する。

GCS が追跡するブロックの状態：
- **LOCAL** — ローカル・インスタンスのみが所有し、変更されている可能性がある
- **SHARED** — 複数のインスタンスが読取り一貫性のあるコピーを保持している
- **NULL** — インスタンスがそのブロックを現行モードで保持する必要がなくなった

### グローバル・エンキュー・サービス (GES)

GES は、ディクショナリ・ロック、DML ロック、シーケンス、DDL ロックなど、インスタンスをまたぐキャッシュ・フュージョン以外のリソースを管理する。GES は、インスタンス 1 がある行のロックを取得した場合、インスタンス 2 がその行に対して競合するロックを取得できないことを保証する。

### GCS/GES アクティビティの監視

```sql
-- グローバル・キャッシュ効率: キャッシュ・フュージョンによって回避されたディスク読み取りの割合
SELECT inst_id,
       ROUND(
           (gc_cr_blocks_received + gc_current_blocks_received) /
           NULLIF(physical_reads + gc_cr_blocks_received + gc_current_blocks_received, 0) * 100,
           2
       ) AS cache_fusion_pct
FROM (
    SELECT inst_id,
           SUM(CASE WHEN name = 'gc cr blocks received'      THEN value ELSE 0 END) AS gc_cr_blocks_received,
           SUM(CASE WHEN name = 'gc current blocks received' THEN value ELSE 0 END) AS gc_current_blocks_received,
           SUM(CASE WHEN name = 'physical reads'             THEN value ELSE 0 END) AS physical_reads
    FROM   gv$sysstat
    WHERE  name IN ('gc cr blocks received', 'gc current blocks received', 'physical reads')
    GROUP  BY inst_id
)
ORDER  BY inst_id;

-- インスタンス間のブロック転送を引き起こしている上位セグメント
SELECT segment_name, segment_type,
       gc_buffer_busy_waits, gc_cr_blocks_received, gc_current_blocks_received
FROM   gv$segment_statistics
WHERE  statistic_name IN ('gc buffer busy waits')
  AND  value > 0
ORDER  BY value DESC
FETCH  FIRST 20 ROWS ONLY;
```

---

## 4. RAC サービスの構成

RAC サービスは、特定のノードにワークロードを振り向け、透過的なフェイルオーバーを可能にするための主要なメカニズムである。サービスは、クライアントが特定のインスタンスに直接接続するのではなく、接続対象として指定する名前付きのエンティティである。

### サービスの種類

| サービスのタイプ | 説明 |
|---|---|
| 優先/使用可能 (Preferred/Available) | サービスは優先ノードで実行される。障害時には使用可能ノードにフェイルオーバーする |
| 均一 (Uniform) | すべてのノードで同時にサービスを実行する |
| 管理者管理 (Administrator-managed) | 優先インスタンスと使用可能インスタンスを手動で定義する |
| ポリシー管理 (Policy-managed) | サーバー・プールのポリシーに基づいて Oracle GI がインスタンス数を動的に管理する |

### サービスの作成と構成

```sql
-- DBCA または SRVCTL (推奨) を使用
-- クラスタ・ノードの OS コマンドラインから:
--
-- 優先インスタンスを指定してサービスを作成
-- srvctl add service -db MYDB -service OLTP_SVC \
--     -preferred MYDB1,MYDB2 -available MYDB3

-- サービスの構成確認
-- srvctl config service -db MYDB -service OLTP_SVC

-- サービスの起動 / 停止
-- srvctl start  service -db MYDB -service OLTP_SVC
-- srvctl stop   service -db MYDB -service OLTP_SVC

-- SQL*Plus 内からプログラムでサービスを作成する場合
BEGIN
    DBMS_SERVICE.CREATE_SERVICE(
        service_name  => 'OLTP_SVC',
        network_name  => 'OLTP_SVC',
        goal          => DBMS_SERVICE.GOAL_THROUGHPUT,
        clb_goal      => DBMS_SERVICE.CLB_GOAL_LONG
    );
END;
/

-- 各インスタンスでアクティブなサービスの確認
SELECT inst_id, name, network_name, goal, clb_goal
FROM   gv$active_services
WHERE  name NOT IN ('SYS$BACKGROUND', 'SYS$USERS')
ORDER  BY name, inst_id;
```

### パフォーマンスのためのサービス属性

```sql
-- サービス・レベルの閾値を超えた場合にアラートをトリガーする
BEGIN
    DBMS_SERVICE.MODIFY_SERVICE(
        service_name            => 'OLTP_SVC',
        goal                    => DBMS_SERVICE.GOAL_SERVICE_TIME,
        clb_goal                => DBMS_SERVICE.CLB_GOAL_SHORT,
        -- コールあたりの経過時間が 5 秒を超えた場合にアラートを出す
        aq_ha_notifications     => TRUE,
        -- コミット結果の追跡 (最大 1 回の実行保証のため)
        commit_outcome          => TRUE,
        retention_timeout       => 604800  -- 7 日間 (秒単位)
    );
END;
/
```

---

## 5. ノード・アフィニティ

ノード・アフィニティ（Node Affinity）とは、特定のワークロードやスキーマをクラスタ内の特定のノードにバインド（固定）する手法である。これによりキャッシュ・フュージョンのトラフィックを削減できる。同じデータが常に同じノードからアクセスされれば、インスタンスをまたぐブロック転送は発生しないためである。

### アプリケーション・レベルのアフィニティ

最も信頼性の高いアフィニティは、特定のインスタンスにバインドされた専用のサービスを経由してアプリケーションの接続をルーティングすることで実現される：

```
OLTP アプリケーション  -> OLTP_SVC  -> ノード 1 & 2 (優先)
レポート・アプリケーション -> RPT_SVC   -> ノード 3 & 4 (優先)
バッチ・ジョブ        -> BATCH_SVC -> ノード 4 (優先)
```

この構成では、OLTP データ・ブロックは主にノード 1/2 のバッファ・キャッシュに存在し、レポート・データは主にノード 3/4 のキャッシュに存在するため、インスタンスをまたぐ転送が最小限に抑えられる。

### 表のパーティショニングとアフィニティ

複数のサーバー・プールを持つポリシー管理データベースでは、パーティション化された表を使用して、異なるパーティションが主に異なるノード・セットからアクセスされるように構成できる。これを**パーティション・アフィニティ**と呼ぶ。

```sql
-- 例: 地域ごとのアプリが特定のノードにマップされた地域固有のサービスを通じて接続する、範囲パーティション化された売上表
CREATE TABLE SALES (
    sale_id     NUMBER        NOT NULL,
    region_id   NUMBER(2)     NOT NULL,
    sale_date   DATE          NOT NULL,
    amount      NUMBER(12,2)  NOT NULL
)
PARTITION BY RANGE (region_id) (
    PARTITION sales_region_1  VALUES LESS THAN (6),   -- ノード 1-2
    PARTITION sales_region_2  VALUES LESS THAN (11),  -- ノード 3-4
    PARTITION sales_region_3  VALUES LESS THAN (16),  -- ノード 5-6
    PARTITION sales_other     VALUES LESS THAN (MAXVALUE)
);
```

---

## 6. RAC 固有の待機イベント

RAC には、シングル・インスタンスには存在しない特有の待機イベントが導入されている。これらのイベントの待機時間が長い場合は、インターコネクトまたはキャッシュの競合の問題を示している。

### 重要な待機イベント

| 待機イベント | 原因 | 閾値 |
|---|---|---|
| `gc buffer busy acquire` | リモートから送信されているブロックを取得するためにローカル・セッションが待機中 | 平均 1ms 未満 |
| `gc buffer busy release` | 他のローカル・セッションがリモートから要求されているブロックを保持している間に、ローカル・セッションが待機中 | 平均 1ms 未満 |
| `gc cr block busy` | マスターが転送処理中に、ブロックの CR コピーを要求した | 平均 2ms 未満 |
| `gc current block busy` | 現行ブロックを要求中。リモート・インスタンスがまだ送信していない | 平均 2ms 未満 |
| `gc cr block 2-way` | 通常の 2 者間 CR ブロック転送 (要求側 + マスター)。許容範囲のベースライン | 1ms 未満 |
| `gc current block 2-way` | 通常の 2 者間現行ブロック転送 | 1ms 未満 |
| `gc cr block 3-way` | 3 者間転送 (要求側 + マスター + 保持側)。レイテンシが高くなる | 2ms 未満 |
| `gc current block 3-way` | 3 者間現行ブロック転送 | 2ms 未満 |
| `gcs log flush sync` | ブロック転送前に、リモート・インスタンスが REDO ログをフラッシュするのを待機中 | REDO を確認 |
| `enq: TX - row lock contention` | 他のインスタンスで保持されている行レベルのロック | アプリケーションの問題 |

```sql
-- 待機時間合計による上位 RAC 待機イベント
SELECT inst_id,
       event,
       total_waits,
       time_waited_micro / 1e6          AS total_sec,
       ROUND(time_waited_micro / NULLIF(total_waits, 0) / 1000, 3) AS avg_wait_ms
FROM   gv$system_event
WHERE  event LIKE 'gc %'
   OR  event LIKE 'gcs %'
   OR  event LIKE 'ges %'
ORDER  BY time_waited_micro DESC
FETCH  FIRST 20 ROWS ONLY;

-- セッション・レベルの RAC 待機 (特定の接続の診断用)
SELECT sid, event, state, wait_class,
       seconds_in_wait, p1, p2, p3
FROM   gv$session
WHERE  wait_class != 'Idle'
  AND  (event LIKE 'gc %' OR event LIKE 'gcs %')
ORDER  BY seconds_in_wait DESC;
```

---

## 7. 透過的アプリケーション・フェイルオーバー (TAF) と 高速接続フェイルオーバー (FCF)

### 透過的アプリケーション・フェイルオーバー (TAF)

TAF は、TNS 記述子や Oracle 接続プールで構成される、クライアント側のフェイルオーバー・メカニズムである。クライアントが接続しているインスタンス障害時に、TAF は自動的に生存しているインスタンスに再接続し、オプションで障害発生時点の SELECT 文を再実行する。

```
# TAF を含む TNS エントリ (tnsnames.ora)
MYDB_TAF =
  (DESCRIPTION =
    (FAILOVER = ON)
    (LOAD_BALANCE = OFF)
    (ADDRESS = (PROTOCOL = TCP)(HOST = node1-vip)(PORT = 1521))
    (ADDRESS = (PROTOCOL = TCP)(HOST = node2-vip)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = OLTP_SVC)
      (FAILOVER_MODE =
        (TYPE = SELECT)    -- またはクエリ以外のフェイルオーバー用の SESSION
        (METHOD = BASIC)   -- または事前にシャドウ接続を確立しておく PRECONNECT
        (RETRIES = 30)
        (DELAY = 5)
      )
    )
  )
```

TAF の制限事項：
- 実行中の DML は再現**されない**。アプリケーションは未コミットのトランザクションに対してエラーを受け取る。
- `TYPE=SELECT` は問合せを最初から再実行する（すでにフェッチされた行はスキップされる）。
- TAF はトランザクション途中のネットワーク分断を保護しない。

### 高速接続フェイルオーバー (FCF)

FCF は Oracle Notification Service (ONS) を使用し、JDBC Thin ドライバや Universal Connection Pool (UCP) で構成される。Java アプリケーションにとって、FCF は TAF より優れている：

- サービスが停止すると即座に、接続プールが ONS イベントを（GI イベント通知システム経由で）受信する。
- 古くなった（無効な）接続は、アプリケーションがそれを使おうとする前に予防的にプールから除去される。
- 実行中トランザクションの透過的な再現のためのアプリケーション・コンティニュイティ (AC) と連携する。

```java
// FCF 用の UCP 構成 (イメージ)
// dataSource.setFastConnectionFailoverEnabled(true);
// dataSource.setONSConfiguration("nodes=node1:6200,node2:6200");
```

### アプリケーション・コンティニュイティ (AC) と 透過的アプリケーション・コンティニュイティ (TAC)

アプリケーション・コンティニュイティ (AC: Oracle 12c R1 導入) は、TAF の概念を拡張し、リカバリ可能なエラー後、DML を含む実行中トランザクションを透過的に再現する。透過的アプリケーション・コンティニュイティ (TAC: 18c 導入、19c 拡張) は、サービスの `failover_type => 'AUTO'` を使用することで、アプリケーションの構成を変更することなくこれを実現する。

```sql
-- サービスに対してアプリケーション・コンティニュイティが有効かどうかの確認
SELECT name, failover_type, failover_method, goal, commit_outcome, retention_timeout
FROM   dba_services
WHERE  name = 'OLTP_SVC';

-- サービスでアプリケーション・コンティニュイティを有効化 (failover_type => 'TRANSACTION' = AC)
-- 透過的アプリケーション・コンティニュイティ (TAC, 18c+) の場合は、failover_type => 'AUTO' を使用
BEGIN
    DBMS_SERVICE.MODIFY_SERVICE(
        service_name    => 'OLTP_SVC',
        failover_type   => 'TRANSACTION',  -- AC を有効化。18c 以降で TAC を使う場合は 'AUTO'
        commit_outcome  => TRUE,
        retention_timeout => 86400
    );
END;
/
```

---

## 8. クラスタ検証ユーティリティ (CVU)

CVU (`cluvfy`) は、クラスタ環境における Oracle のインストール前およびインストール後の診断ツールである。ネットワーク構成、OS パラメータ、共有ストレージ、およびクラスタ・ソフトウェアの健全性を確認する。

```bash
# インストール前チェック (Grid Infrastructure インストール前に実行)
# cluvfy stage -pre crsinst -n node1,node2 -verbose

# インストール後チェック
# cluvfy stage -post crsinst -n node1,node2

# 任意のタイミングでのクラスタ検証
# cluvfy comp sys     -n node1,node2        -- OS パラメータ
# cluvfy comp nodecon -n node1,node2        -- ノードの接続性
# cluvfy comp ocr     -n node1,node2        -- OCR の整合性
# cluvfy comp ssa     -n node1,node2 -s disk_group_name  -- 共有ストレージ

-- SQL*Plus からの RAC データベース健全性の確認
-- すべてのインスタンスが開いているかの確認
SELECT inst_id, instance_name, host_name, status, database_status
FROM   gv$instance
ORDER  BY inst_id;

-- クラスタ・インターコネクトの確認
SELECT inst_id, name, ip_address, is_public, source
FROM   gv$cluster_interconnects;

-- すべてのインスタンスから全データファイルにアクセスできるか確認
SELECT inst_id, file#, status, name
FROM   gv$datafile_header
WHERE  status != 'ONLINE'
ORDER  BY inst_id, file#;

-- 投票ディスクと OCR のステータス確認 (OS から root で実行)
-- crsctl query css votedisk
-- ocrcheck
```

---

## 9. ベスト・プラクティス

- **すべてのクライアント接続に SCAN (Single Client Access Name) を使用する。** SCAN は、ノード数に関係なくクライアントに単一のアドレスを提供する。アプリケーションの接続文字列に VIP アドレスをハードコードしてはいけない。
- **Grid Infrastructure と データベース・ホームを別々のファイル・システムで運用する。** GI のパッチ適用と DB のパッチ適用は独立している。ホームを分けることで意図しない停止を防止できる。
- **ブロック転送のピーク時の負荷に合わせてインターコネクトのサイズを決定する。** `gc cr blocks received` と `gc current blocks received` のレートを監視して、必要な帯域幅を予測すること。インターコネクトの飽和は、RAC のパフォーマンス低下の主要な原因である。
- **デプロイ前にサービスを設計する。** サービスは、アプリケーションのワークロード・タイプ（OLTP、レポート、バッチ）と 1 対 1 で対応させるべきである。1 つのサービスにワークロードを混在させると、診断と隔離が不可能になる。
- **RAC では、`SID=*` または `SID=<sid>` を指定せずに初期化パラメータを設定するために `ALTER SYSTEM` を使用してはいけない。** 1 つのインスタンスで誤った構成を設定すると、そのインスタンスがクラッシュする原因になる。
- **新規デプロイではポリシー管理データベースを優先する。** サーバー・プールを使用すると、ノード障害時に Oracle が手動操作なしで自動的にノード間でインスタンスをリバランスできる。
- **Cluster Health Monitor (CHM/OS Watcher) を有効にする。** CHM は、ノードごとに 1 秒単位で OS レベルのメトリック（CPU、メモリー、ネットワーク、ディスク I/O）をキャプチャし、ノード・エビクション後の事後分析において非常に貴重なデータとなる。

---

## 10. よくある間違いとその回避方法

### 間違い 1: インターコネクトのトラフィックをパブリック・ネットワーク経由でルーティングする

`/etc/hosts` や DNS に適切なプライベート・ホスト名のエントリがない場合、Oracle はデフォルトでパブリック・ネットワークをインターコネクトのトラフィックに使用してしまうことがある。これによりパブリック NIC が飽和し、深刻な GCS レイテンシが発生する。

**対策:** インストール前後に、インターコネクトの IP がプライベート・ネットワーク上にあることを必ず確認すること。

```sql
-- インターコネクトがパブリック IP でないことを確認
SELECT name, ip_address, is_public FROM gv$cluster_interconnects;
-- アクティブなインターコネクトの is_public は 'NO' である必要がある
```

### 間違い 2: インスタンス間で共有されるデフォルトの UNDO および TEMP 表領域を使用する

RAC では、**1 インスタンスにつき 1 つの UNDO 表領域**が必要である。RAC で単一の UNDO 表領域を共有することはサポートされていない。

```sql
-- インスタンスごとの UNDO 表領域の割り当てを確認
SELECT inst_id, value AS undo_tablespace
FROM   gv$parameter
WHERE  name = 'undo_tablespace'
ORDER  BY inst_id;

-- 各 inst_id に固有の UNDO 表領域名がある必要がある
-- UNDOTBS1 -> インスタンス 1, UNDOTBS2 -> インスタンス 2 など
```

### 間違い 3: キャッシュ・フュージョンのためにシーケンス駆動の主キーに索引を付けない

すべてのインスタンスで単一のシーケンスを共有すると、索引の「右側（最大値側）」のリーフ・ブロックがホット・ブロックとなり、すべてのインスタンスが更新を争うことになる。これにより、深刻な `gc buffer busy` 待機が発生する。

**対策:** 高い同時実行性で INSERT が発生する RAC 環境では、シーケンス生成されたキーに対して**リバース・キー索引**（逆キー索引）またはハッシュ・パーティション索引を使用する。

```sql
-- リバース・キー索引により右側の競合を削減
CREATE INDEX IX_ORDERS_ORDER_ID ON ORDERS (order_id) REVERSE;

-- または、SGA 負荷を軽減するために大きな CACHE 値を持つシーケンスを使用する
CREATE SEQUENCE SEQ_ORDERS
    START WITH 1
    INCREMENT BY 1
    CACHE 1000       -- 各インスタンスが 1000 個の値を事前割り当てし、GES トラフィックを削減
    NOORDER;         -- NOORDER は RAC での順序保証のオーバーヘッドを回避する (サロゲートキーであれば問題ない)
```

### 間違い 4: ノード・エビクションの根本原因を無視する

ノード・エビクション（ハングしたノードや遅いノードを強制的にクラスタから除去すること）は、ハードウェア障害として扱われることが多いが、実際には以下のような原因で発生する：
- クラスタ・ロックを保持している OS プロセスのフリーズ
- インターコネクトの遅延によるハートビートの失敗
- 負荷の高いノードが `misscount` タイムアウト内に CSS に応答できなかった場合

ハードウェアの故障と結論付ける前に、必ず `cssd.log`、`alert_<sid>.log`、および CHM データを総合的に分析すること。

### 間違い 5: `opatchauto` を実行せずにパッチを適用する

RAC 環境では、GI と RAC のパッチは `opatchauto` を使用してローリング形式で適用する必要がある。`opatchauto` を使わずに手動でパッチを適用すると、ノード間で GI と DB のホームが矛盾した状態になり、スプリット・ブレインのシナリオにつながる可能性がある。

---


## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Real Application Clusters Administration and Deployment Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/racad/) — RAC アーキテクチャ, バックグラウンド・プロセス, GCS/GES, サービス, TAF, CVU
- [Oracle RAC 19c Glossary](https://docs.oracle.com/en/database/oracle/oracle-database/19/racad/glossary.html) — LMON, LMSn, LMD, Cache Fusion, GCS, GES, SCAN, OCR, CSS, CTSS 定義
- [About RAC Background Processes (12c doc, applies to 19c)](https://docs.oracle.com/database/121/RACAD/GUID-AEBD3F49-4F10-4BDE-9008-DC1AF8E7DB42.htm) — LMS, LMD, LCK, LMON, DIAG, RMSn の説明
- [About Connecting to an Oracle RAC Database Using SCANs](https://docs.oracle.com/en/database/oracle/oracle-database/19/rilin/about-connecting-to-an-oracle-rac-database-using-scans.html) — SCAN は 1〜3 個の IP に解決。Oracle は 3 個を推奨
- [Ensuring Application Continuity (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/racad/ensuring-application-continuity.html) — AC は 12c R1 導入。TAC は 18c 導入。FAILOVER_TYPE 値について
- [DBMS_SERVICE (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SERVICE.html) — failover_type の有効な値 (TRANSACTION, SELECT, SESSION, NONE)
