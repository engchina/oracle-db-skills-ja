# Oracle APEX (Application Express)

## 概要

**Oracle APEX** (Application Express) は、Oracle Database に直接組み込まれたローコード開発プラットフォームである。開発者は、別のアプリケーション・サーバーを用意することなく、ブラウザベースのツールを使用して、Web アプリケーション、REST API、およびデータ駆動型のダッシュボードを構築できる。APEX アプリケーションはデータベース・エンジン内（または Oracle REST Data Services 経由）で動作し、Oracle の表を直接クエリする。アプリケーションはエクスポート可能なメタデータとしてデプロイおよびバージョン管理される。

APEX は、Oracle Database (Standard Edition 2 および Enterprise Edition) に追加費用なしで含まれている。最新バージョン（23.x, 24.x 以降）では、モダンな JavaScript フレームワークとの統合、ネイティブな JSON および REST サービス、ローコード・アプリケーション・ビルダー、および Oracle Cloud の Autonomous Database でのデプロイをサポートしている。

**主な強み:**
- データ中心の学内ツールやダッシュボードの超高速な開発サイクル
- Oracle Database ユーザー向けの追加ライセンス費用ゼロ
- 豊富な UI コンポーネントを備えた宣言的なページ設計
- SQL/PLSQL との深い統合 — 式、バリデーション、プロセスはすべてインラインで SQL/PLSQL を使用
- 認証（LDAP, SSO, OAuth2, カスタム）、認可、監査ログの組み込み機能

---

## APEX アーキテクチャ

### 実行パス

**Oracle REST Data Services (ORDS) を使用した APEX**
モダンで推奨されるデプロイ形式。ORDS は Java ベースのミドルウェア層（スタンドアロンまたは Tomcat/Kubernetes 等で動作）であり、HTTP/S 接続を処理し、APEX ページ・リクエストをデータベースにルーティングする。APEX 22 以降では ORDS が必須である。

```
ブラウザ
  |
  v
ORDS (Jetty / Tomcat / Container)
  |  -- APEX ページ・リクエスト --
  v
Oracle Database
  |-- APEX エンジン (APEX_xxxxxx スキーマ内の PL/SQL パッケージ)
  |-- アプリケーション・スキーマ (APPSCHEMA, HRSCHEMA 等)
  |-- APEX メタデータ (アプリケーション定義、ページ、アイテム)
```

**Autonomous Database (Oracle Cloud) 上の APEX**
ORDS と APEX がプリインストール・構成済みで提供される。

---

## ワークスペースとアプリケーション

### ワークスペース (Workspaces)

**ワークスペース**は、APEX における最上位の組織単位である。以下の役割を持つ名前空間として機能する：
- 1つ以上のアプリケーションを含む
- 1つ以上のデータベース・スキーマと関連付けられる（アプリケーションがクエリ可能なスキーマ）
- 独自の開発者と管理者を持つ
- 独自のストレージ・クォータ（割り当て制限）を持つ

### アプリケーション (Applications)

アプリケーションは、ページ、共有コンポーネント（ナビゲーション、選択リスト、認可スキーム等）、およびメタデータの集合である。アプリケーションは単一の SQL スクリプトとしてエクスポートされ、別の APEX インスタンスにインポートできる。これが標準のデプロイおよびバージョン管理の仕組みである。

---

## 解析スキーマ (Parsing Schema)

APEX アプリケーションは、**解析スキーマ**のコンテキストでクエリを実行する。つまり、アプリケーション内の SQL や PL/SQL で参照されるオブジェクトの解決には、このスキーマの権限が使用される。

---

## APEX のコンポーネント

### ページ構成要素

- **リージョン (Regions):** コンテンツ（レポート、チャート、フォームなど）のコンテナ。
- **ページ・アイテム (Page Items):** フォーム入力フィールドや隠し値。
- **ページ・プロセス (Page Processes):** データの保存や API 呼び出しを行う PL/SQL ロジック。
- **動的アクション (Dynamic Actions):** クライアント側の JavaScript イベントやサーバー側の AJAX 処理を宣言的に定義する。

### セッション・ステートの参照

APEX はユーザー入力や変数を**セッション・ステート**に保存する。SQL 内ではバインド変数として参照できる：

```sql
SELECT * FROM orders WHERE customer_id = :P2_CUSTOMER_ID;
```

---

## ORDS (Oracle REST Data Services)

ORDS は APEX の実行ホストであると同時に、フル機能の REST API 開発プラットフォームでもある。

### スキーマの REST 有効化

```sql
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled             => TRUE,
        p_schema              => 'APPSCHEMA',
        p_url_mapping_pattern => 'appschema'
    );
    COMMIT;
END;
/
```

### Table の自動 REST 有効化 (Auto-REST)

特定の表に対して、コードを書かずに CRUD 操作（GET, POST, PUT, DELETE）の API を公開できる。

```sql
BEGIN
    ORDS.ENABLE_OBJECT(
        p_enabled      => TRUE,
        p_schema       => 'APPSCHEMA',
        p_object       => 'ORDERS',
        p_object_alias => 'orders'
    );
END;
/
```

---

## ベスト・プラクティス

- **バインド変数を常に使用する:** SQL インジェクションを防ぎ、パフォーマンス（ソフト・パース）を向上させるために、SQL 内では必ず `:P1_ITEM` 形式で値を参照する。文字列連結で SQL を組み立ててはならない。
- **解析スキーマの権限を最小限にする:** 解析スキーマには必要最小限の権限（特定の表へのアクセス権など）のみを付与する。SYS や SYSTEM を解析スキーマにしてはならない。
- **認可スキーム (Authorization Schemes) を活用する:** コンポーネントの「非表示」設定だけに頼らず、サーバー側での認可チェック（認可スキーム）を必ず設定する。
- **変更後は必ずエクスポートして Git で管理する:** APEX アプリケーションの「正」のソースはエクスポートされた SQL ファイルである。定期的にエクスポートし、データベース・スキーマ・スクリプトと共にバージョン管理システムに保存すること。
- **APEX_DEBUG でボトルネックを特定する:** ページの応答が遅い場合は、デバッグ・モードを有効にして各コンポーネントの実行時間を確認する。

---

## よくある間違い

**間違い: フォームの入力値を直接 PL/SQL の文字列連結に含める。**
これは SQL インジェクション攻撃に対して非常に脆弱である。常に `:P_ITEM` 形式のバインド変数を使用すること。

**間違い: 開発環境と本番環境で同じ ORDS インスタンスを共有する。**
開発中の重い処理や不適切な構成が、本番環境の可用性に影響を与える可能性がある。環境ごとに ORDS インスタンスを分けるべきである。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 世代以降の機能は、Oracle Database 26ai 対応機能として扱う。

## ソース

- [Oracle APEX Documentation](https://docs.oracle.com/en/database/oracle/apex/)
- [Oracle REST Data Services Documentation](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/)
- [APEX_WEB_SERVICE Package Reference](https://docs.oracle.com/en/database/oracle/apex/24.1/aeapi/APEX_WEB_SERVICE.html)
