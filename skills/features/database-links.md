# Oracle データベース・リンク

## 概要

**データベース・リンク** (dblink) とは、ローカルの Oracle データベースに保存される名前付きの接続記述子である。これにより、リモートのデータベースにあるオブジェクトを、あたかもローカルにあるかのように SQL 文から参照できるようになる。データベース・リンクには、接続情報（ホスト、ポート、サービス名）と、リンクのタイプに応じてリモート・セッションを確立するための資格情報（ユーザー名、パスワード）が格納される。

データベース・リンクは、分散データベース技術の多くが登場する前から存在する Oracle の標準機能であり、Oracle 環境内でのデータベース間のクエリ、分散 DML、およびレプリケーション・シナリオにおいて、現在も実用的なソリューションとして広く利用されている。

**データベース・リンクが適しているケース:**
- 同じ信頼されたネットワーク内にある 2 つの Oracle データベース間でのアドホックなクエリ
- 複数の Oracle ソースからデータを集約する定期的なバッチ・ジョブ
- Oracle データベース間のレプリケーションや同期（主にマテリアライズド・ビュー経由）
- 切り替え（カットオーバー）時に古いデータベースからデータを読み取る必要がある移行シナリオ

**データベース・リンクが適して「いない」ケース:**
- 高頻度な OLTP クエリ — ネットワークの往復オーバーヘッドが積み重なり、パフォーマンスが低下する
- 大規模な表同士のデータベースをまたいだジョイン — ネットワーク転送量が制御不能になる恐れがある
- 非 Oracle データベースへの直接接続（異機種間サービス / 汎用接続を使用すべき）
- 強力なセキュリティ分離が必要な状況 — dblink は暗号化されるとはいえ、暗黙的な信頼関係を構築するため

---

## データベース・リンクのタイプ

### 固定ユーザー・リンク (Fixed User Link)

特定の固定されたユーザーとしてリモート・データベースに接続する。誰がリンクを介してクエリを実行しても、リンク内に保存されたリモート資格情報が使用される。

```sql
CREATE DATABASE LINK sales_db_link
CONNECT TO remote_user IDENTIFIED BY "remote_password"
USING 'SALESDB';   -- TNS サービス名または接続文字列
```

### 接続ユーザー・リンク (Connected User Link)

ローカル・データベースに現在ログインしている**同じユーザー**としてリモートに接続する。リモート側にも同じ名前のアカウントが必要である。

```sql
CREATE DATABASE LINK hr_db_link
CONNECT TO CURRENT_USER
USING 'HRDB';
```

資格情報がデータベース内に保存されないため、固定ユーザー・リンクよりもセキュアである。

### 共有データベース・リンク (Shared Database Link)

複数のローカル・セッションで、単一のリモート・セッションを再利用する。リモート側の接続オーバーヘッドを軽減できる。

```sql
CREATE SHARED DATABASE LINK shared_dw_link
CONNECT TO dw_query_user IDENTIFIED BY "dw_password"
AUTHENTICATED BY local_user IDENTIFIED BY "local_password"
USING 'DWDB';
```

### パブリック・リンクとプライベート・リンク

デフォルトでは、作成したユーザーのみが使用できる「プライベート」リンクになる。すべてのユーザーが使用できる「パブリック」リンクを作成することも可能である。

```sql
-- プライベート・リンク (作成者のみ)
CREATE DATABASE LINK my_private_link ...;

-- パブリック・リンク (すべてのユーザーが利用可能)
CREATE PUBLIC DATABASE LINK corp_shared_link ...;
-- CREATE PUBLIC DATABASE LINK 権限が必要
```

---

## クエリでの使用方法

作成したリンクを使用してリモート・オブジェクトを参照するには、オブジェクト名の後ろに `@<リンク名>` を付与する。

### リモート・データベースへのクエリ

```sql
-- リモート表からの選択
SELECT employee_id, last_name
FROM   employees@hr_db_link
WHERE  department_id = 90;

-- ローカル表とリモート表のジョイン
SELECT l.order_id, r.customer_name
FROM   orders         l
JOIN   customers@crm_db_link r ON r.customer_id = l.customer_id;

-- シノニムを使用してリンク名を隠蔽
CREATE SYNONYM remote_customers FOR customers@crm_db_link;
SELECT * FROM remote_customers;
```

### リモート・データベースへの DML (分散 DML)

INSERT, UPDATE, DELETE, MERGE がサポートされている。

```sql
-- リモート表への挿入
INSERT INTO archive_orders@archive_db_link (order_id, amount)
SELECT order_id, amount FROM orders_local;

-- リモート・プロシージャの呼び出し
BEGIN
    archive_pkg.purge_old_records@archive_db_link(p_date => SYSDATE - 365);
END;
/
```

---

## 2フェーズ・コミット (分散トランザクション)

1 つのトランザクションが、データベース・リンクを介して**複数のデータベース**のデータを変更する場合、Oracle は自動的に **2フェーズ・コミット (2PC)** を使用して原子性（All or Nothing）を保証する。

1. **準備フェーズ (Prepare):** ローカル DB（調整役）が各リモート DB（参加役）に、コミットの準備ができているか問い合わせる。
2. **コミット・フェーズ (Commit):** 全員が「OK」であればコミットを指示。1つでも不可（またはタイムアウト）であれば全員にロールバックを指示する。

### 疑わしい (In-Doubt) トランザクションの監視

通信障害などで 2PC が途中で止まった場合、トランザクションは「疑わしい (In-Doubt)」状態として残る。

```sql
-- インダウト・トランザクションの確認
SELECT local_tran_id, state, host, db_user
FROM   dba_2pc_pending;

-- (調査後、必要であれば) 手動でコミットを強制
COMMIT FORCE '10.13.3.10.1';
```

---

## パフォーマンス上の注意点

### DRIVING_SITE ヒント

Oracle のオプティマイザは、ローカルでジョインを行うためにリモートから大量のデータを取得しようとすることがある。`DRIVING_SITE` ヒントを使用すると、リモート側でジョインを実行させ、データ転送量を大幅に削減できる場合がある。

```sql
-- リモート側 (c) でクエリを実行するよう指示
SELECT /*+ DRIVING_SITE(c) */
       o.order_id, c.customer_name
FROM   orders          o
JOIN   customers@remote_db c ON c.customer_id = o.customer_id
WHERE  c.country_code = 'JP';
```

---

## セキュリティ・ベスト・プラクティス

- **強力な権限を持つアカウントを共通リンクに使用しない:** リモート側の `DBA` ユーザーなどを指定した固定ユーザー・リンクは非常に危険である。リンク専用の最小権限ユーザー（特定の表への SELECT 権限のみなど）を作成して使用すること。
- **パブリック・リンクの作成には慎重になる:** パブリック・リンクは、そのデータベースのすべてのユーザーがリモートへの「抜け穴」として利用できる可能性がある。
- **パスワードの定期的な変更:** 固定ユーザー・リンクにはパスワードが保存されている。セキュリティ・ポリシーに従い、定期的にリンクを再作成してパスワードを更新すること。
- **不要なリンクの削除:** 移行完了後など、不要になったリンクは即座に削除する。放置されたリンクは潜在的なセキュリティ・リスクとなる。

---

## よくある間違い

**間違い: 大規模なジョインを頻繁に実行するコードに dblink を使用する。**
1 行ごとにネットワーク往復が発生し、システム全体が遅延する原因となる。パフォーマンスが重要な箇所では、マテリアライズド・ビューによるローカル・コピーの保持を検討すべきである。

**間違い: `DBA_2PC_PENDING` のエントリを無視する。**
インダウト・トランザクションは、リモート側でロックを保持し続けたり、リソースを消費し続けたりする。定期的に確認し、自動修復されない場合は手動での対応が必要になる。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [Oracle Database Administrator's Guide: Managing Distributed Databases](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-a-distributed-database.html)
- [Oracle Database SQL Language Reference: CREATE DATABASE LINK](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-DATABASE-LINK.html)
