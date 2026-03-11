# 移行アセスメント・ガイド

## 概要

移行プロジェクトにおいて、最も重要なフェーズは事前の移行アセスメント（現状分析）である。アセスメントを省略したり、おろそかにしたりすることは、予算超過、納期遅延、あるいは本番移行時の障害を招く最大の要因となる。アセスメントでは、以下の4つの問いに対する答えを導き出す。「何を移行する必要があるか？」「どの程度の複雑さか？」「自動変換できないものは何か？」「どのようなリスクがあるか？」

本ガイドでは、包括的なアセスメント・チェックリスト、スキーマの複雑さのスコアリング手法、互換性のない機能の特定フレームワーク、作業量見積もりの指針、リスク・マトリックス、および Oracle の移行アセスメント・ツールの活用方法について解説する。

---

## 移行前アセスメント・チェックリスト

### 1. インベントリと対象範囲

移行コードを1行も書く前に、以下のインベントリ（目録）を確認する。

**スキーマ・オブジェクト:**
- [ ] すべての表をカウント（それぞれの概算行数を含む）
- [ ] すべての索引をカウント（クラスタ索引、カバリング索引、部分索引などを含む）
- [ ] すべてのビューをカウント（通常ビュー、マテリアライズド・ビュー/索引付きビュー、パーティション・ビュー）
- [ ] すべてのストアド・プロシージャをカウント
- [ ] すべての関数をカウント
- [ ] すべてのトリガーをカウント
- [ ] すべてのパッケージをカウント（DB2、Oracle-to-Oracle の場合）
- [ ] すべてのシーケンスをカウント
- [ ] すべてのシノニムをカウント
- [ ] すべてのデータベース・リンク / リンク・サーバーをカウント
- [ ] すべてのスケジュール・ジョブ / エージェントをカウント
- [ ] すべての外部表またはリンクされたファイル・パスをリストアップ
- [ ] 対象となるすべてのスキーマ/ユーザーをリストアップ

**データの特性:**
- [ ] データベースの総容量 (GB/TB)
- [ ] 最大の個別表（行数およびバイト数）
- [ ] LOB列を持つ表 (CLOB, BLOB, TEXT, IMAGE, BYTEA など)
- [ ] 50列以上ある表（複雑性の指標）
- [ ] JSON/XML/半構造化データ列を持つ表
- [ ] 空間データ/ジオメトリ列を持つ表
- [ ] バイナリ列または暗号化列の特定

**機能の使用状況:**
- [ ] パーティション表（パーティションのタイプと戦略）
- [ ] 全文検索索引
- [ ] 独自の索引タイプ (PostgreSQL の GiST, GIN、SQL Server の XML 索引など)
- [ ] 行レベル・セキュリティ / Virtual Private Database (VPD)
- [ ] 監査ログ・メカニズム
- [ ] レプリケーション構成（パブリケーション、サブスクリプション、ロジカル・レプリケーション）
- [ ] 他システムへのリンク・サーバーまたはデータベース・リンク
- [ ] CLR オブジェクト (SQL Server)、C 拡張 (PostgreSQL)、または Java ストアド・プロシージャ
- [ ] カスタム集計関数
- [ ] ユーザー定義型
- [ ] 独自の手続き型言語機能

**アプリケーションの接続性:**
- [ ] 移行元データベースに接続しているすべてのアプリケーションをリストアップ
- [ ] 接続文字列 / DSN 設定を特定
- [ ] 接続プール設定 (DRCP, pgBouncer, ProxySQL など) を特定
- [ ] 使用されている ORM (Hibernate, SQLAlchemy, Entity Framework など) を特定
- [ ] アプリケーション・コード内の生 SQL と ORM 生成 SQL を特定
- [ ] 直接 DB 接続を行っているレポート作成ツール (Tableau, Power BI, Crystal Reports など) を特定

---

## スキーマの複雑さのスコアリング

以下のスコアリング・モデルを使用して、オブジェクトごとの移行作業量を見積もる。単価にカウント数を掛けて、推定総日数を算出する。

### スコアリング表

| オブジェクト・タイプ | 低複雑度 | 中複雑度 | 高複雑度 |
|---|---|---|---|
| シンプルな表 (20列未満, LOBなし, パーティションなし) | 0.1 日 | — | — |
| 複雑な表 (50列以上, LOBあり, またはパーティションあり) | — | 0.5 日 | 1 日 |
| シンプルなビュー (単一の表, 基本的なフィルタ) | 0.1 日 | — | — |
| 複雑なビュー (複数表の結合, ウィンドウ関数) | — | 0.5 日 | 1 日 |
| シンプルなストアド・プロシージャ (50行未満, 基本的な CRUD) | 0.5 日 | — | — |
| 中程度のストアド・プロシージャ (50–200行, カーソル, ループ) | — | 1–2 日 | — |
| 複雑なストアド・プロシージャ (200行超, 動的 SQL, 一時表) | — | — | 3–5 日 |
| シンプルなトリガー (監査ログ, タイムスタンプ更新) | 0.25 日 | — | — |
| 複雑なトリガー (複数表, 条件分岐, プロシージャ呼び出し) | — | 1 日 | — |
| シンプルな関数 | 0.25 日 | — | — |
| 複雑な関数 (再帰, 多相入力) | — | 1–2 日 | — |
| パッケージ (DB2/Oracle) | — | 2–5 日 | — |
| スケジュール・ジョブ / エージェント | 各 0.5 日 | — | — |
| 全文検索機能 | — | — | 3–10 日 |
| 空間データ / ジオメトリ機能 | — | — | 5–15 日 |
| CLR オブジェクト | — | — | 5–20 日 |

### 複雑度分類 SQL クエリ

移行元データベースで以下のクエリを実行し、オブジェクトを複雑度別に分類する。

```sql
-- SQL Server: 行数によるストアド・プロシージャの分類
SELECT
    OBJECT_NAME(p.object_id) AS proc_name,
    (SELECT COUNT(*) FROM sys.sql_modules m
     WHERE m.object_id = p.object_id) AS has_definition,
    LEN(sm.definition) AS char_count,
    CASE
        WHEN LEN(sm.definition) < 1000 THEN 'Low'
        WHEN LEN(sm.definition) < 5000 THEN 'Medium'
        ELSE 'High'
    END AS complexity
FROM sys.procedures p
JOIN sys.sql_modules sm ON p.object_id = sm.object_id
ORDER BY LEN(sm.definition) DESC;
```

```sql
-- PostgreSQL: 行数による関数の分類
SELECT
    n.nspname AS schema,
    p.proname AS function_name,
    pg_get_functiondef(p.oid) AS definition,
    CASE
        WHEN LENGTH(pg_get_functiondef(p.oid)) < 500  THEN 'Low'
        WHEN LENGTH(pg_get_functiondef(p.oid)) < 2500 THEN 'Medium'
        ELSE 'High'
    END AS complexity
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY LENGTH(pg_get_functiondef(p.oid)) DESC;
```

```sql
-- Oracle 移行元 (Oracle-to-Oracle 移行の場合):
SELECT
    owner,
    object_type,
    object_name,
    status,
    CASE
        WHEN dbms_metadata.get_ddl(object_type, object_name, owner) IS NULL THEN 'Unknown'
        WHEN LENGTH(dbms_metadata.get_ddl(object_type, object_name, owner)) < 500 THEN 'Low'
        WHEN LENGTH(dbms_metadata.get_ddl(object_type, object_name, owner)) < 3000 THEN 'Medium'
        ELSE 'High'
    END AS complexity
FROM all_objects
WHERE owner = 'SOURCE_SCHEMA'
  AND object_type IN ('PROCEDURE', 'FUNCTION', 'PACKAGE', 'TRIGGER', 'VIEW')
ORDER BY object_type, object_name;
```

---

## 互換性のない機能の特定

### PostgreSQL の互換性のない機能

```sql
-- SERIAL 列の特定 (Oracle では IDENTITY または SEQUENCE が必要)
SELECT table_name, column_name, column_default
FROM information_schema.columns
WHERE column_default LIKE 'nextval(%'
  AND table_schema = 'public';

-- ARRAY 列の特定 (Oracle では正規化が必要)
SELECT table_name, column_name, data_type, udt_name
FROM information_schema.columns
WHERE data_type = 'ARRAY'
  AND table_schema = 'public';

-- BOOLEAN 列の特定 (23c 未満の Oracle では NUMBER(1) が必要)
SELECT table_name, column_name
FROM information_schema.columns
WHERE data_type = 'boolean'
  AND table_schema = 'public';

-- ENUM 型の特定 (Oracle では VARCHAR2 + CHECK 制約が必要)
SELECT typname, enumlabel
FROM pg_enum
JOIN pg_type ON pg_enum.enumtypid = pg_type.oid
ORDER BY typname, enumsortorder;

-- TEXT 列の特定 (CLOB にするかどうかの判断が必要)
SELECT table_name, column_name
FROM information_schema.columns
WHERE data_type = 'text'
  AND table_schema = 'public';

-- PostgreSQL 固有の機能を使用している関数の特定
SELECT routine_name, routine_definition
FROM information_schema.routines
WHERE routine_schema = 'public'
  AND (
    routine_definition ILIKE '%$1%'           -- PL/pgSQL 変数構文
    OR routine_definition ILIKE '%ILIKE%'     -- 大文字小文字を区別しない LIKE
    OR routine_definition ILIKE '%generate_series%'  -- PG 固有
    OR routine_definition ILIKE '%array_agg%'        -- PG 固有
  );
```

### SQL Server の互換性のない機能

```sql
-- IDENTITY 列の特定
SELECT
    t.name AS table_name,
    c.name AS column_name,
    c.seed_value,
    c.increment_value
FROM sys.tables t
JOIN sys.columns c ON t.object_id = c.object_id
JOIN sys.identity_columns ic ON c.object_id = ic.object_id AND c.column_id = ic.column_id;

-- UNIQUEIDENTIFIER 列の特定 (Oracle では RAW(16) が必要)
SELECT TABLE_NAME, COLUMN_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE DATA_TYPE = 'uniqueidentifier';

-- 動的 SQL を使用しているプロシージャの特定
SELECT OBJECT_NAME(object_id) AS proc_name
FROM sys.sql_modules
WHERE definition LIKE '%EXEC%(%'
   OR definition LIKE '%sp_executesql%'
   OR definition LIKE '%EXECUTE%(%';

-- CLR オブジェクトの特定
SELECT assembly_name, create_date
FROM sys.assemblies
WHERE is_user_defined = 1;

-- リンク・サーバーの参照を特定
SELECT DISTINCT OBJECT_NAME(referencing_id) AS obj_name
FROM sys.sql_expression_dependencies
WHERE referenced_server_name IS NOT NULL;

-- 全文検索索引の特定
SELECT t.name AS table_name, c.name AS column_name
FROM sys.fulltext_index_columns fic
JOIN sys.tables t ON fic.object_id = t.object_id
JOIN sys.columns c ON fic.object_id = c.object_id AND fic.column_id = c.column_id;
```

### MySQL の互換性のない機能

```sql
-- AUTO_INCREMENT 列の特定
SELECT table_name, column_name
FROM information_schema.columns
WHERE extra = 'auto_increment'
  AND table_schema = DATABASE();

-- ENUM および SET 列の特定
SELECT table_name, column_name, data_type, column_type
FROM information_schema.columns
WHERE data_type IN ('enum', 'set')
  AND table_schema = DATABASE();

-- ゼロ日付をデフォルトにしている列の特定
SELECT table_name, column_name, column_default
FROM information_schema.columns
WHERE column_default IN ('0000-00-00', '0000-00-00 00:00:00')
  AND table_schema = DATABASE();

-- MySQL 固有の関数を使用しているストアド・プロシージャ/関数の特定
SELECT routine_name, routine_definition
FROM information_schema.routines
WHERE routine_schema = DATABASE()
  AND (
    routine_definition LIKE '%GROUP_CONCAT%'
    OR routine_definition LIKE '%LIMIT %'
    OR routine_definition LIKE '%IF(%'   -- MySQL IF() 関数
    OR routine_definition LIKE '%IFNULL%'
  );
```

---

## 作業量の見積もり

### 作業量見積もりフレームワーク

以下の式を基準とし、プロジェクト固有の要因で調整する。

```
総作業量 = スキーマ作業量 + データ移行作業量 + テスト作業量 + バッファ

スキーマ作業量   = SUM(オブジェクト数 × 複雑度レベル別の単価)
データ移行作業量  = (合計 GB / 1時間あたりのスループット GB) × 検証係数
テスト作業量     = スキーマ作業量の 40% (最小)
バッファ        = 合計の 25% (未知の要因用)
```

### 一般的なスループットの目安

| 移行方法 | 一般的なスループット |
|---|---|
| SQL*Loader DIRECT パス (ローカル) | 1–5 GB/分 |
| SQL*Loader ネットワーク経由 | 100 MB–1 GB/分 |
| Data Pump (同一サーバー) | 2–10 GB/分 |
| JDBC 1行ずつの挿入 | 10–50 MB/分 |
| Oracle ZDM + GoldenGate | 変動（レプリケーション遅延は変更量に依存） |

### プロジェクト複雑度の乗数

| 要因 | 乗数 |
|---|---|
| チームにとって初めての Oracle 移行 | 1.5倍 |
| 複雑なストアド・プロシージャが作業の 50% を超える | 1.3倍 |
| 空間データ / 全文検索 / CLR 機能が存在する | 1.4倍 |
| 低ダウンタイム要件 (4時間未満) | 1.6倍 |
| 既知のデータ品質の問題がある | 1.4倍 |
| テスト・カバレッジのないレガシー・コード | 1.5倍 |

### 作業量見積もりシートの例

```
移行元: SQL Server 2016
移行先: Oracle 19c
スキーマ: 表 200, ビュー 45, ストアド・プロシージャ 120, 関数 30, トリガー 15

オブジェクト作業量:
  表 200 × 0.2 日 (混合複雑度の平均)                    =  40 日
  ビュー 45 × 0.3 日 (混合複雑度の平均)                  =  14 日
  シンプルなプロシージャ 60 × 0.5 日                      =  30 日
  中程度のプロシージャ 40 × 1.5 日                       =  60 日
  複雑なプロシージャ 20 × 4 日                          =  80 日
  関数 30 × 0.5 日                                    =  15 日
  トリガー 15 × 0.5 日                                  =   8 日
スキーマ小計:                                            247 日

データ移行:
  500 GB / 1 GB/分 = 約8時間のロード時間
  + 検証、再実行、エラー解決に 16 時間                    =  4 日

テスト:
  247 日の 40%                                         =  99 日

小計:                                                   350 日

バッファ (25%):                                          88 日

推定総計:                                               438 日
乗数 (初めての Oracle 移行):                             × 1.5
最終調整後合計:                                         657 人日
```

---

## リスク・マトリックス

各リスク要因を「発生確率 (1–5)」と「影響度 (1–5)」で評価する。掛け合わせた値がリスク・スコアとなる。

| リスク要因 | 具体的な兆候 | 発生確率 (1-5) | 影響度 (1-5) | 対策 |
|---|---|---|---|---|
| スキーマ複雑度が高い | 100以上のプロシージャ, CLR オブジェクト | ? | ? | 最初にツールによるアセスメントを実行する。段階的に移行する。 |
| データ品質の問題 | Null 制約違反, 制約の不一致 | ? | ? | 移行前にデータ品質のプロファイリングを実行する。 |
| 低ダウンタイム要件 | SLA < 4時間のメンテナンス・ウィンドウ | ? | ? | GoldenGate / ZDM を計画する。ステージング環境で切り替えをテストする。 |
| アプリケーション動作の差異 | NULL の意味論, 大文字小文字の区別 | ? | ? | 移行データに対してフル回帰テストを実行する。 |
| パフォーマンス低下 | オプティマイザの違い, 統計情報の欠如 | ? | ? | 本番切り替え前に主要なクエリのベンチマークを実行する。 |
| ロールバックの複雑さ | 大規模データ, バックアップなし | ? | ? | ロールバック手順を計画し、テストする。 |
| チームのスキル・ギャップ | チーム内に Oracle 経験者がいない | ? | ? | 教育を実施する。外部の Oracle DBA によるサポートを検討する。 |
| サードパーティ・ツールの互換性 | BI ツール, ETL パイプライン | ? | ? | すべての接続を監査する。各ツールを Oracle に対してテストする。 |

### リスク・スコアリング

| スコア | レベル | アクション |
|---|---|---|
| 1–4 | 低 | 監視を継続し、リスク一覧に記録する。 |
| 5–9 | 中 | 担当者を割り当て、対策計画を作成する。 |
| 10–15 | 高 | エスカレーションする。プロジェクト開始を遅らせる可能性がある。 |
| 16–25 | 致命的 | ブロッカー（障害）。プロジェクト開始前に解決する必要がある。 |

---

## Oracle データベース移行アセスメント関連リソース

> ⚠️ 未検証: 「Database Migration Assessment Framework (DMAF)」という用語は **AWS** のオープンソース・ツールを指し、Oracle の製品ではない。Oracle はその名称の製品を公開していない。Oracle の移行アセスメント・リソースは、Oracle SQL Developer Migration Workbench（組み込みアセスメント）、AWS SCT アセスメント・レポート（クロス RDBMS 用）、およびクラウド移行を対象とした **Oracle Cloud Migration Advisor** を通じて提供される。

### Oracle Cloud Migration Advisor

移行先が OCI (Oracle Autonomous Database または DBCS) の場合は、Oracle Cloud Migration Advisor を使用する。

> ⚠️ 未検証: Oracle Cloud Migration Advisor の正確なセットアップ手順は、OCI リージョンやサービス世代によって異なる場合がある。使用前に最新の OCI ドキュメントを確認すること。

1. **Oracle Cloud Database Migration** サービスをインストールするか、組み込みの OCI 移行ツールを使用する。
2. Advisor を移行元データベースに接続する。
3. ワークロード・キャプチャを実行して、SQL およびスキーマ・メタデータを収集する。
4. Advisor レポートを確認して、以下を把握する。
   - 互換性の問題。
   - パフォーマンス・リスク。
   - 移行元の機能を置き換えるための推奨される Oracle の機能。

### Oracle データ・ディクショナリ・クエリによる手動アセスメント

Oracle-to-Oracle の移行（例：オンプレミスからクラウド）を評価する場合：

```sql
-- 移行元スキーマ内のタイプ別オブジェクト数
SELECT object_type, COUNT(*) AS object_count, COUNT(CASE WHEN status != 'VALID' THEN 1 END) AS invalid_count
FROM all_objects
WHERE owner = 'SOURCE_SCHEMA'
GROUP BY object_type
ORDER BY object_count DESC;

-- パーティションが設定されている表
SELECT table_name, partitioning_type, partition_count
FROM all_part_tables
WHERE owner = 'SOURCE_SCHEMA';

-- LOB 列を持つ表
SELECT c.table_name, c.column_name, c.data_type
FROM all_tab_columns c
WHERE c.owner = 'SOURCE_SCHEMA'
  AND c.data_type IN ('CLOB', 'BLOB', 'NCLOB', 'BFILE')
ORDER BY c.table_name;

-- 使用されているデータベース・リンク
SELECT db_link, username, host FROM all_db_links WHERE owner = 'SOURCE_SCHEMA';

-- コンパイル・エラーのあるパッケージ
SELECT owner, name, type, line, text
FROM all_errors
WHERE owner = 'SOURCE_SCHEMA'
ORDER BY name, line;

-- 最大値に近づいているシーケンス
SELECT sequence_name, max_value, last_number,
       ROUND((last_number / NULLIF(max_value, 0)) * 100, 2) AS pct_used
FROM all_sequences
WHERE sequence_owner = 'SOURCE_SCHEMA'
ORDER BY pct_used DESC;
```

---

## 開始前に確認すべき質問事項

### 技術的な質問

1. **移行元データベースのバージョンは？** 古いバージョンには非推奨の機能が含まれている場合があり、移行の複雑さが増す。

2. **固有のデータ型や拡張機能は？** PostGIS, pg_trgm, hstore, SQL Server 空間データ, XML などは、それぞれに対応する Oracle の同等機能を確認する必要がある。

3. **使用されている文字セットは？** 移行元のエンコーディング (UTF-8, Latin-1, UTF-16) は、データ変換や Oracle の NLS 設定に影響する。

4. **最大の表は何か？** また、その表の移行時の許容ロード時間は？

5. **主キーのない表はあるか？** これらはレプリケーション・ベースの移行ツールを使用する際に問題となる。

6. **許容される最大ダウンタイムは？** これにより、オフライン移行 (Data Pump / SQL*Loader) かオンライン移行 (GoldenGate / ZDM) かを選択する。

7. **データベース・スキーマは安定しているか、現在進行形で変更されているか？** 移行中に変更されるスキーマの場合、DDL 変更をキャプチャするための変更管理手順が必要になる。

### 組織的な質問

8. **アプリケーション・コードの所有者は誰か？** ストアド・プロシージャの変更には、DBA だけでなくアプリケーション・チームの関与が必要になる場合がある。

9. **テスト・カバレッジはどの程度か？** 十分にテストされているアプリケーションであれば、デグレ（機能退行）を検出しやすい。テストがない場合は、手動による回帰テストが必要になる。

10. **コンプライアンスおよび監査の要件は？** GDPR, HIPAA, PCI-DSS, SOX などの要件は、Oracle のセキュリティ構成、暗号化設定、および監査構成に影響する可能性がある。

11. **ロールバック計画は？** 切り替え時に移行が失敗した場合、どれくらい迅速に元に戻せるか？ 特定時点へのリストアは可能か？ 並行稼働中に移行元システムは利用可能か？

12. **Go/No-Go 基準は何か？** 開始前に、具体的で測定可能な基準を定義しておくこと（`migration-cutover-strategy.md` を参照）。

13. **法規制や契約による変更禁止期間（フリーズ期間）はあるか？** 多くの組織には、年度末や四半期末など、大規模な移行を避けるべきフリーズ期間がある。

14. **パフォーマンス SLA は？** パフォーマンスが極めて重要な上位 10–20 個のクエリを特定し、切り替え前に Oracle 上でベンチマークを実施する必要がある。

---

## アセスメントの成果物

完全な移行アセスメントでは、以下の成果物を作成する必要がある。

1. **スキーマ・インベントリ・レポート:** スキーマおよびタイプ別の完全なオブジェクト数。表ごとのデータ・サイズと行数。

2. **複雑度評価レポート:** オブジェクトごとの複雑度スコア。評価の根拠と推奨されるアプローチ。

3. **互換性のない機能リスト:** 移行元データベースで使用されている機能のうち、Oracle に同等の機能がないもの、または大幅な再設計が必要なもの。

4. **作業量見積もり:** フェーズ別に分けられ、複雑度、乗数で調整された人日単位の作業量。

5. **リスク・レジスタ:** 特定されたリスク、その発生確率、影響度、および対策計。

6. **移行アプローチ・ドキュメント:** 推奨されるツール構成、移行フェーズ、およびタイムライン。

7. **Go/No-Go 基準:** 本番切り替え前に満たさなければならない、具体的で測定可能な基準。

8. **ロールバック計画:** 移行が失敗した場合に移行元データベースへ戻すための、ステップ・バイ・ステップの手順。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。異なるバージョンが混在する環境では、19c と互換性のある代替案を検討すること。
- 両方のサポート環境において、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合がある。19c と 26ai の両方で構文やパッケージの動作をテストすること。

## ソース

- [Oracle SQL Developer Migration documentation](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
- [AWS Schema Conversion Tool — Assessment Reports](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_AssessmentReport.html)
- [Oracle Database 19c — DBMS_METADATA package](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_METADATA.html)
