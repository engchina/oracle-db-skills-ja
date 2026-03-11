# Oracle アドバンスト・キューイング (AQ) とトランザクショナル・イベント・キュー (TQ)

## 概要

Oracle アドバンスト・キューイング (AQ) は、Oracle データベース・エンジンに直接組み込まれた、データベース統合型のメッセージ・キューイング機能である。外部のメッセージング・システムとは異なり、AQ はメッセージを通常のデータベース表に保存する。そのため、メッセージは Oracle のトランザクションに完全に完全参加し、データベースの信頼性保証を享受でき、さらに標準の SQL を使用してクエリを実行することも可能である。

Oracle Database 21c では、AQ が **トランザクショナル・イベント・キュー (Transactional Event Queue: TQ/TEQ)** として再ブランド化され、大幅に強化された。これにより、高スループットのパーティション化されたストレージ、Kafka 互換の API、およびスケーラビリティの向上が実現されつつ、従来の AQ API との完全な後方互換性も維持されている。

**Oracle AQ/TQ を使用すべきケース:**
- すでに Oracle 上で動作しており、確実なメッセージ配信を必要とするアプリケーション
- データベースへの書き込みとメッセージのパブリッシュ（発行）の間で、トランザクションの一貫性を必要とするシナリオ
- メッセージ・データをクエリしたり、レポートに使用したりする必要があるシステム
- 外部ブローカー（Kafka、RabbitMQ など）を追加することで運用が複雑になるのを避けたい環境

---

## コア・コンセプト

### キュー・タイプ (Queue Types)

**単一コンシューマ・キュー (Single-Consumer Queues)**
メッセージは、ちょうど 1 つのコンシューマによってデキュー（取り出し）される。これは最もシンプルなモデルであり、ポイント・ツー・ポイント (P2P) メッセージング・パターンに直接対応する。

**マルチコンシューマ・キュー (Multi-Consumer Queues)**
名前付きの複数のサブスクライバ（購読者）のそれぞれが、すべてのメッセージのコピーを受け取ることができる。これはパブリッシュ/サブスクライブ (pub/sub) パターンに対応する。各サブスクライバは、キュー内での自身の論理的な位置を保持する。

**例外キュー (Exception Queues)**
すべてのキューには例外キューが関連付けられている。配信不能なメッセージ（例：最大リトライ回数を超えたもの）は自動的にここへ移動され、キューの「毒（ポイズニング）」化を防ぐ。

### メッセージ・ペイロード・タイプ

| ペイロード・タイプ | 説明 |
|---|---|
| `RAW` | 非構造化バイナリ・データ |
| `VARCHAR2` | 文字列ペイロード |
| オブジェクト型 (Object type) | 任意の Oracle オブジェクト型 (最も一般的) |
| `DBMS_AQ.AQ$_JMS_TEXT_MESSAGE` | JMS 互換のテキスト・メッセージ |
| `JSON` | ネイティブ JSON ペイロード (21c 以降) |

### キュー表とキュー

**キュー表 (Queue Table)** は、メッセージを保存する基礎となるデータベース表である。**キュー (Queue)** は、キュー表の上に構築された論理オブジェクトである。1 つのキュー表で複数のキューをホストできる。キュー表のストレージ・オプションや索引構造は `DBMS_AQADM` によって管理される。

---

## DBMS_AQADM — 管理パッケージ

`DBMS_AQADM` は、キュー表の作成、キューの作成、キューの開始/停止、およびサブスクライバの追加/削除など、キュー・インフラストラクチャのライフサイクルを管理する。

### キュー表の作成

```sql
-- まずペイロード・タイプを定義
CREATE OR REPLACE TYPE order_payload_t AS OBJECT (
    order_id     NUMBER,
    customer_id  NUMBER,
    status       VARCHAR2(50),
    total_amount NUMBER(10,2),
    created_at   TIMESTAMP
);
/

-- オブジェクト型を使用したマルチコンシューマ・キュー表の作成
BEGIN
    DBMS_AQADM.CREATE_QUEUE_TABLE(
        queue_table        => 'order_queue_tab',
        queue_payload_type => 'order_payload_t',
        multiple_consumers => TRUE,       -- 単一コンシューマの場合は FALSE
        sort_list          => 'PRIORITY,ENQ_TIME',  -- デキュー順序の設定
        comment            => '注文処理イベント・キュー'
    );
END;
/
```

### キューの作成と開始

```sql
BEGIN
    -- キュー表の上にキューを作成
    DBMS_AQADM.CREATE_QUEUE(
        queue_name         => 'order_events_q',
        queue_table        => 'order_queue_tab',
        max_retries        => 5,
        retry_delay        => 60,    -- ロールバック後のリトライ待機時間(秒)
        retention_time     => 86400, -- デキュー済みメッセージの保持期間(秒)
        comment            => '注文ライフサイクル・イベント'
    );

    -- キューの開始（エンキューとデキューの両方を有効化）
    DBMS_AQADM.START_QUEUE(queue_name => 'order_events_q');
END;
/
```

### サブスクライバの追加 (マルチコンシューマの場合)

```sql
DECLARE
    subscriber_agent SYS.AQ$_AGENT;
BEGIN
    subscriber_agent := SYS.AQ$_AGENT('FULFILLMENT_SERVICE', NULL, 0);

    DBMS_AQADM.ADD_SUBSCRIBER(
        queue_name => 'order_events_q',
        subscriber => subscriber_agent,
        rule       => 'tab.user_data.status = ''NEW'''  -- コンテンツベースのフィルタリング
    );
END;
/
```

---

## DBMS_AQ — エンキューとデキュー

### エンキュー (Enqueue) の例

```sql
DECLARE
    enqueue_options    DBMS_AQ.ENQUEUE_OPTIONS_T;
    message_properties DBMS_AQ.MESSAGE_PROPERTIES_T;
    message_handle     RAW(16);
    payload            order_payload_t;
BEGIN
    payload := order_payload_t(10042, 5001, 'NEW', 249.99, SYSTIMESTAMP);

    -- 可視性の設定
    enqueue_options.visibility := DBMS_AQ.ON_COMMIT;  -- COMMIT 後にメッセージが可視になる

    -- メッセージ・プロパティの設定
    message_properties.priority    := 1;              -- 数値が小さいほど高優先順位
    message_properties.expiration  := 3600;           -- 1時間以内にデキューされなければ期限切れ

    DBMS_AQ.ENQUEUE(
        queue_name         => 'order_events_q',
        enqueue_options    => enqueue_options,
        message_properties => message_properties,
        payload            => payload,
        msgid              => message_handle
    );

    COMMIT;
END;
/
```

### デキュー (Dequeue) の例

```sql
DECLARE
    dequeue_options    DBMS_AQ.DEQUEUE_OPTIONS_T;
    message_properties DBMS_AQ.MESSAGE_PROPERTIES_T;
    message_handle     RAW(16);
    payload            order_payload_t;
BEGIN
    -- デキュー・オプションの設定
    dequeue_options.consumer_name := 'FULFILLMENT_SERVICE'; -- マルチコンシューマでは必須
    dequeue_options.dequeue_mode  := DBMS_AQ.REMOVE;         -- 取得後に削除
    dequeue_options.wait          := 30;                      -- 最大30秒待機

    DBMS_AQ.DEQUEUE(
        queue_name         => 'order_events_q',
        dequeue_options    => dequeue_options,
        message_properties => message_properties,
        payload            => payload,
        msgid              => message_handle
    );

    DBMS_OUTPUT.PUT_LINE('処理中の注文: ' || payload.order_id);

    COMMIT; -- コミット時にメッセージがキューから完全に削除される
EXCEPTION
    WHEN DBMS_AQ.TIME_OUT THEN
        DBMS_OUTPUT.PUT_LINE('待機時間内にメッセージが見つかりませんでした');
END;
/
```

---

## トランザクショナル・イベント・キュー (21c 以降)

TEQ は、イベント・ストリーミング・ワークロード向けに設計された**パーティション化された高スループット・ストレージ・エンジン**で AQ を強化する。

```sql
-- トランザクショナル・イベント・キューの作成 (21c+)
BEGIN
    DBMS_AQADM.CREATE_TRANSACTIONAL_EVENT_QUEUE(
        queue_name         => 'iot_sensor_teq',
        queue_payload_type => 'JSON',
        multiple_consumers => TRUE
    );

    DBMS_AQADM.START_QUEUE(queue_name => 'iot_sensor_teq');
END;
/
```

---

## ベスト・プラクティス

- **`ON_COMMIT` 可視性を使用する:** OLTP システムでのエンキュー・デキューでは、常に `ON_COMMIT` を使用する。`IMMEDIATE` を使用すると、トランザクションが完了する前にメッセージが公開され、不完全なデータが処理されるリスクがある。
- **リトライ設定を適切に行う:** すべてのキューで `max_retries` と `retry_delay` を設定する。これがないと、失敗し続けるメッセージが処理を永久にブロックする可能性がある。制限を超えたメッセージは例外キューに送られる。
- **例外キューを定期的に監視する:** メッセージが例外キューに入ったときにアラートを発するように監視機能を構築する。
- **デキュー・ループでは例外処理を忘れない:** `DBMS_AQ.TIME_OUT` や `DBMS_AQ.NO_MESSAGE_FOUND` を適切にハンドリングし、コンシューマが予期せず停止しないようにする。
- **キュー表の肥大化を防ぐ:** 処理済みの古いメッセージを削除するために、`DBMS_AQADM.PURGE_QUEUE_TABLE` を使用して定期的にクリーンアップを行う。

---

## よくある間違い

**間違い: デキュー後のコミットを忘れる。**
`ON_COMMIT` 可視性を使用している場合、コミットしなければメッセージはキューに残ったまま（あるいは元に戻る）となる。ビジネス・ロジックの完了と最終的なコミットをアトミック（不可分）に構成すること。

**間違い: キュー表を過剰に作成する。**
1 つのキュー表は複数の内部オブジェクト（索引など）を生成する。細かいバリエーションごとにキュー表を分けるのではなく、サブスクライバのルールや相関 ID を使用して、1 つのキュー内でストリームを区別することを検討する。

**間違い: `DBMS_AQ.FOREVER` 待機を多用する。**
`FOREVER` 指定による無期限待機は、データベース接続を永久に占有する。コネクション・プールを使用する環境では、有限の待機時間を設定し、ループ内でリトライするように構成すべきである。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [DBMS_AQADM — Oracle Database PL/SQL Packages and Types Reference 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_AQADM.html)
- [DBMS_AQ — Oracle Database PL/SQL Packages and Types Reference 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_AQ.html)
- [Oracle Database Advanced Queuing User's Guide 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/adque/index.html)
