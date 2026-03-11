# Oracle 移行ツール・リファレンス

## 概要

データベースの Oracle への移行を、完全に手動で行うことはほとんどない。スキーマ変換、データ移行、アセスメント、および継続的なレプリケーションを自動化するためのツール・スイートが用意されている。本ガイドでは、Oracle 移行エコシステムにおける最も重要なツールを網羅する。具体的には、Oracle SQL Developer Migration Workbench、AWS Schema Conversion Tool (SCT)、ora2pg、SSMA for Oracle、Oracle Zero Downtime Migration (ZDM)、および機能比較表について解説する。

どのフェーズでどのツールを使用すべきかを理解することは、ツール自体の使い方を知ることと同じくらい重要である。ほとんどの移行プロジェクトでは、少なくとも2つのツールが必要になる。1つはスキーマ変換用、もう1つはデータ移行用である。

---

## Oracle SQL Developer Migration Workbench

Oracle SQL Developer には、Migration Workbench（移行ワークベンチ）が組み込まれている。これは、MySQL、SQL Server、Sybase ASE、DB2、Microsoft Access、PostgreSQL、および Teradata からの移行をサポートしている（それぞれの JDBC ドライバを使用）。

### ステップ・バイ・ステップ：SQL Server から Oracle への移行

#### ステップ 1 — 移行リポジトリのセットアップ

Migration Workbench は、移行メタデータを保存するために専用の Oracle スキーマを必要とする。

```sql
-- 専用の移行リポジトリ・スキーマを作成
CREATE USER migration_repo IDENTIFIED BY "repo_password"
    DEFAULT TABLESPACE users QUOTA UNLIMITED ON users;
GRANT CONNECT, RESOURCE TO migration_repo;
GRANT CREATE VIEW TO migration_repo;
GRANT CREATE MATERIALIZED VIEW TO migration_repo;
```

SQL Developer で、**「ツール」→「移行」→「移行リポジトリの作成」**を選択し、作成した `migration_repo` スキーマを指定する。

#### ステップ 2 — 移行元データベース接続の作成

- 「ファイル」→「新規接続」→ 接続タイプとして「SQL Server」を選択。
- JDBC URL、資格証明を入力し、接続をテストする。
- ヒント：SQL Developer のクラスパスに SQL Server JDBC ドライバ (jTDS または Microsoft JDBC) を追加しておく必要がある。

#### ステップ 3 — 移行元データベースのキャプチャ

1. 移行ワークベンチ（「ツール」→「移行」）で、新しい移行プロジェクトを開始する。
2. 移行元接続を右クリック → **「SQL Serverデータベースのキャプチャ」**を選択。
3. キャプチャ対象のデータベース/スキーマを選択する。
4. SQL Developer がすべての DDL、制約、ビュー、ストアド・プロシージャ、およびデータを読み取る。

#### ステップ 4 — 変換

1. キャプチャされた移行元オブジェクトを右クリック → **「Oracleへの変換」**を選択。
2. SQL Developer が、キャプチャされたすべてのオブジェクトに対して Oracle 形式の DDL を生成する。
3. **「移行ログ」**でエラーや警告を確認する。
   - 緑：自動的に変換された。
   - 黄：注意が必要だが変換された。
   - 赤：変換できなかった。手動での対応が必要。

#### ステップ 5 — 生成された DDL の検査と編集

「移行プロジェクト」ツリーで変換後のオブジェクトを確認する。各オブジェクトをクリックすると、生成された Oracle DDL が表示される。警告が表示されたオブジェクトについては、SQL Developer 上で直接編集する。

一般的な手動修正の例：
- 動的 SQL を含むストアド・プロシージャ。
- T-SQL 固有のシステム関数。
- アイデンティティ列のシード/増分値。
- 照合順序（Collation）に依存する比較。

#### ステップ 6 — 移行先 Oracle 接続の作成

- 移行先となる Oracle データベースへの接続を追加する。
- 移行先ユーザー/スキーマに、`CREATE TABLE`, `CREATE PROCEDURE`, `CREATE INDEX` などの権限があることを確認する。

#### ステップ 7 — スキーマの移行

1. 変換後のスキーマを右クリック → **「Oracleへのスキーマの移行」**を選択。
2. SQL Developer が、移行先 Oracle に対してすべての DDL を実行する。
3. 出力ウィンドウでエラーを確認する。

#### ステップ 8 — データ移行

1. 移行プロジェクトを右クリック → **「データの移行」**を選択。
2. SQL Developer が、JDBC 経由で SQL Server から Oracle へ行をストリーム転送する。
3. 「移行データ」パネルで進捗を監視する。

#### ステップ 9 — 検証

SQL Developer は、移行後に基本的な行数比較機能を提供する。詳細な検証については、`migration-data-validation.md` を参照。

### SQL Developer Migration Workbench が得意なこと

- ほとんどのデータ型を含む表定義。
- 索引と制約 (主キー, ユニーク, 外部キー, チェック)。
- ビュー（構文変換を含む）。
- シーケンスとアイデンティティ列。
- シンプルなストアド・プロシージャと関数。
- トリガー（部分的な変換）。

### 制限事項

- CLR オブジェクト (SQL Server) は処理できない。
- プロシージャ内の動的 SQL へのサポートは限定的。
- 全文検索索引はサポートしていない。
- 大規模なデータ移行（数百万行以上）のパフォーマンスは、SQL*Loader ダイレクト・パスより遅い。
- 低ダウンタイム移行のための増分移行/CDC 能力はない。

---

## AWS Schema Conversion Tool (SCT) — Oracle への移行

AWS SCT は主に AWS への移行（RDS, Aurora, Redshift）を目的として設計されているが、Oracle を移行元および構成先の両方としてサポートしている。プロシージャ変換のための洗練されたルール・エンジンを備えている。

### Oracle を移行先とするサポート対象の移行元

- SQL Server → Oracle
- MySQL → Oracle
- PostgreSQL → Oracle
- Teradata → Oracle
- SAP ASE (Sybase) → Oracle

### Oracle への移行での AWS SCT の使用方法

1. **SCT をインストール**する（AWS のダウンロード・ページから）。SCT はデスクトップ・アプリケーションとして動作する。

2. **新しいプロジェクトを作成する:**
   - 「File」→「New Project」。
   - 移行元データベースのタイプを選択し、ターゲットに「Oracle」を選択する。

3. **移行元および移行先データベースに接続する。**

4. **アセスメント・レポートを実行する:**
   - 「View」→「Assessment Report」。
   - SCT が各オブジェクトを分類する：自動変換可能、警告付きで変換可能、アクションが必要。
   - 各カテゴリについて、変換率の推定値が表示される。

5. **変換ダッシュボードの確認:**
   - ダッシュボードには「変換複雑度」スコアが表示される。
   - 手動アクションが必要なオブジェクトは、解説とともにハイライトされる。

6. **スキーマの変換:**
   - 移行元オブジェクトを右クリック → 「Convert Schema」。
   - 右側のパネルで、変換後の Oracle DDL を確認する。

7. **アクション項目の処理:**
   - SCT は、Oracle ドキュメントへのリンクが含まれた具体的な「To-Do」項目を提供する。
   - 各アクション項目について、生成されたコードを編集するか、SCT の提案を受け入れる。

8. **Oracle への適用:**
   - 変換されたスキーマを右クリック → 「Apply to Database」。

### SCT Extension Pack (拡張パック)

SQL Server からの移行において、SCT は SQL Server のシステム関数に対応する Oracle 実装を作成する拡張パックを提供する。

```sql
-- SCT は、以下のような関数と同等の Oracle 関数を含む拡張スキーマをインストールする：
-- CHARINDEX → aws_sqlserver_ext.charindex(...)
-- LEFT       → aws_sqlserver_ext.left(...)
-- FORMAT     → aws_sqlserver_ext.format(...)
```

これにより、変換後のコードでこれらすべての箇所を書き換えることなく呼び出しが可能になる。ただし、長期的には拡張パックの呼び出しをネイティブな Oracle の同等機能に置き換えるべきである。

### AWS SCT の強み

- SQL Server から Oracle への変換品質が非常に高い。
- 複雑な T-SQL パターンを処理できる。
- マッピングできない関数のための拡張パック。
- 作業量見積もりのための優れたアセスメント指標。
- 無料で使用できる。

### AWS SCT の制限事項

- AWS 以外の移行であっても、AWS アカウントのセットアップが必要。
- 一部の複雑なストアド・プロシージャは、依然として手動での作業が必要。
- 非常に大規模なスキーマの場合、UI が重くなることがある。
- データ移行自体は行わない（別途 AWS DMS や SQL*Loader を使用する）。

---

## SSMA (SQL Server Migration Assistant) for Oracle

> ⚠️ **方向の注意：** SSMA for Oracle は、Oracle **から** SQL Server **へ**の移行ツールである（逆ではない）。セクション名が誤解を招く可能性があるが、Oracle が移行先となる場合はこのツールは使用しない。

注：Oracle を移行先とする場合は、代わりに AWS SCT または SQL Developer Migration Workbench を使用すること。他のガイド（`migrate-sqlserver-to-oracle.md` など）で「SSMA for Oracle」に言及している場合は、SQL Server → Oracle の移行を指しているのではなく、AWS SCT や SQL Developer の活用を意図していると理解すべきである。

SSMA ファミリには以下が含まれる：
- SSMA for Oracle (Oracle → SQL Server)
- SSMA for MySQL (MySQL → SQL Server/Azure SQL)
- SSMA for Sybase (Sybase → SQL Server)
- SSMA for Access (Access → SQL Server)

SQL Server → Oracle への移行には、AWS SCT または SQL Developer Migration Workbench が適している。

### SSMA が変換するもの (Oracle から SQL Server への方向)

- 表、ビュー、シーケンス、シノニム。
- ストアド・プロシージャ、関数、パッケージ、トリガー。
- PL/SQL から T-SQL への翻訳。
- `ROWNUM` を `TOP` / `ROW_NUMBER()` へ変換。
- `DECODE` を `CASE` へ変換。
- Oracle の日付/時刻関数を SQL Server の同等機能へ変換。

---

## ora2pg — Oracle/MySQL から PostgreSQL への移行 (Oracle 移行用ではない)

> ⚠️ **方向の注意：** ora2pg は、**Oracle** (または MySQL) データベースを **PostgreSQL** へ移行するためのオープンソースの Perl ツールである。これは Oracle へ移行するためのツールでは**ない**。移行元が Oracle または MySQL であり、移行先は常に PostgreSQL である。Oracle が移行先となる場合は ora2pg を使用しないこと。PostgreSQL から Oracle への移行には、SQL Developer Migration Workbench や手動エクスポート / SQL*Loader のワークフローを使用する。

ora2pg は、組織が Oracle から PostgreSQL へ移行する場合（本ガイドの焦点の逆向き）に必要となる可能性があるため、完全性のためにここにリストされている。そのアセスメント・レポート形式は、複雑度見積もり手法の有用なリファレンスにもなる。

### アセスメント・レポート形式（参考用）

```
-------------------------------------------------------------------------------
Ora2Pg migration level : B-5
-------------------------------------------------------------------------------
Total estimated cost: 226 workday(s)
Migration levels:
  A - Migration that might be run automatically
  B - Migration with code rewrite and a human action is required
  C - Migration that has no equivalent in Oracle
-------------------------------------------------------------------------------
Object type | Number | Invalid | Estimated cost | Comments
-------------------------------------------------------------------------------
TABLE       |    145 |       0 |             29 |
VIEW        |     42 |       3 |             21 |
PROCEDURE   |     67 |      12 |            134 |
FUNCTION    |     23 |       2 |             34 |
TRIGGER     |     18 |       0 |              8 |
-------------------------------------------------------------------------------
```

### ora2pg に関する注意点

- Oracle → PostgreSQL (および MySQL → PostgreSQL) の移行用。Oracle は**移行元**であり、移行先ではない。
- 無料でオープンソース。活発にメンテナンスされている。
- 移行の複雑さに関するアセスメント・レポートを提供。
- Oracle が移行先となる場合は、どのような移行にも適用できない。

---

## Oracle Zero Downtime Migration (ZDM)

Oracle Zero Downtime Migration は、Oracle データベースを Oracle がホストするターゲット（Oracle Cloud Infrastructure (OCI)、Oracle Database@Azure、Oracle Database@Google Cloud、Oracle Database@AWS、Exadata Cloud at Customer、およびオンプレミスの Exadata）へ、最小限またはゼロのダウンタイムで移行するためのエンタープライズ・ツールである。オンライン移行の切り替え期間中、継続的なレプリケーションを行うために Oracle GoldenGate を使用する。移行元データベースは、オンプレミス、OCI、またはサードパーティのクラウドを問わない。

### ZDM アーキテクチャ

```
移行元 Oracle DB
     |
     v (Data Pump または RMAN による初期の一括コピー)
移行先 Oracle DB (OCI, Exadata, Autonomous)
     |
     ^ (GoldenGate による継続的な REDO ログのレプリケーション)
     |
[切り替えポイント：アプリケーションの接続先をリダイレクト]
```

### ZDM 移行フェーズ

1. **VALIDATE (検証)** — 前提条件のチェック：接続性、バージョンの互換性、容量、権限。
2. **SETUP (セットアップ)** — GoldenGate、ネットワーク、移行先データベースの構成。
3. **INITIALIZE (初期化)** — Data Pump または RMAN による初期データの一括転送。
4. **REPLICATE (レプリケーション)** — GoldenGate が移行元から移行先へ進行中の変更をレプリケート。
5. **MONITOR (監視)** — レプリケーション遅延がゼロに近づくまで監視。
6. **CUTOVER (切り替え)** — アプリケーションの接続先を移行先へ切り替え、レプリケーションを停止。

### ZDM の構成例

```bash
# ZDM は、独立した ZDM ホストにインストールされる
# 構成ファイル: zdmconfig.rsp

MIGRATION_METHOD=ONLINE_PHYSICAL   # または ONLINE_LOGICAL, OFFLINE_PHYSICAL
PLATFORM_TYPE=EXACS                # ターゲット: EXACS, DBCS, AUTONOMOUS_DATABASE
TARGETDATABASEADMINUSERNAME=admin
TARGETDATABASEADMINPASSWORD=<password>
SOURCEDATABASEADMINUSERNAME=sys
SOURCEDATABASEADMINPASSWORD=<password>

# GoldenGate 設定 (オンライン移行用)
GOLDENGATESOURCEHOME=/u01/app/goldengate
GOLDENGATESOURCEHOSTUSERNAME=oracle
GOLDENGATETARGETHOME=/u01/app/goldengate_tgt

# Data Pump 設定 (初期の一括ロード用)
DATAPUMPSETTINGS_JOBMODE=SCHEMA
DATAPUMPSETTINGS_DATAPUMPPARAMETERS_PARALLELISM=4
```

```bash
# 移行の検証
zdmcli migrate database -sourcedb ORCL \
    -sourcenode source_host \
    -srcauth zdmauth -srcarg1 user:oracle \
    -targetdatabase cdb_name \
    -targethostid abc123 \
    -tdbtokenarr token_string \
    -rsp /etc/zdm/zdmconfig.rsp \
    -eval   # -eval フラグでドライ・ランによる検証のみを実行
```

### ZDM の強み

- Oracle から Oracle への移行において、ニアゼロ・ダウンタイムを実現。
- GoldenGate が組み込まれている。
- 物理的 (RMAN) および論理的 (Data Pump) な移行方法の両方をサポート。
- 移行先として OCI Autonomous Database をサポート。
- Oracle によって公式にサポートおよびドキュメント化されている。

### ZDM の制限事項

- Oracle から Oracle への移行のみを対象とする（クロス RDBMS 移行は不可）。
- サポート対象の移行先は Oracle ブランドのクラウドおよびオンプレミスの Exadata プラットフォームに限定される。汎用的なハードウェア上の任意のオンプレミス Oracle 環境は対象外。
- オンライン（ニアゼロ・ダウンタイム）移行には GoldenGate のライセンスが必要（オフライン移行では不要）。
- 初めて使用する場合、セットアップが複雑。

---

## 各ツールの機能比較

| 機能 | SQL Dev Workbench | AWS SCT | ora2pg | ZDM | GoldenGate |
|---|---|---|---|---|---|
| PostgreSQL → Oracle | ○ | ○ | × (方向違い) | × | × |
| MySQL → Oracle | ○ | ○ | × (方向違い) | × | ○ |
| SQL Server → Oracle | ○ | ○ | × | × | ○ |
| Sybase → Oracle | ○ | ○ | × | × | × |
| DB2 → Oracle | ○ | × | × | × | × |
| Teradata → Oracle | ○ | × | × | × | × |
| Oracle → PostgreSQL | × | × | ○ (主要ケース) | × | × |
| Oracle → Oracle | × | × | × | ○ | ○ |
| スキーマ変換 | ○ | ○ | ○ (Ora→PGのみ) | × | × |
| データ移行 | ○ (低速) | × | ○ (Ora→PGのみ) | ○ | ○ |
| アセスメント・レポート | 基本的 | 詳細 | 詳細 (Ora→PG) | 検証のみ | × |
| ニアゼロ・ダウンタイム | × | × | × | ○ | ○ |
| コスト | 無料 | 無料 | 無料 | OCIに含まれる | ライセンス制 |
| GUI | ○ | ○ | × (CLI) | CLI | ○ |
| ストアド・プロシージャ変換 | 良 | 最良 | Oracle 向けは N/A | N/A | N/A |

---

## 適切なツール構成の選択

### SQL Server → Oracle の場合

1. スキーマ変換（表、索引、プロシージャ）：**AWS SCT**。
2. 大規模データの一括移行：**SQL*Loader** または **Oracle Data Pump**。
3. ニアゼロ・ダウンタイムが必要な場合：**Oracle GoldenGate**。

### PostgreSQL → Oracle の場合

1. スキーマ変換（表、索引、ビュー、プロシージャ）：**SQL Developer Migration Workbench**。
2. 大規模データのロード：**SQL*Loader**。`\COPY` コマンドで CSV 出力して利用。
3. 詳細なアセスメント・レポートを伴うスキーマ変換の代替手段：**AWS SCT**。

### MySQL → Oracle の場合

1. スキーマ変換：**SQL Developer Migration Workbench**。
2. データ移行：CSV 出力した上で **SQL*Loader**。

### Oracle → Oracle (クラウド移行) の場合

1. 移行ライフサイクル全体の管理：**Oracle ZDM**。
2. オフラインでのスキーマ / データ転送：**Data Pump**。
3. オンラインでの継続的なレプリケーション：**GoldenGate**。

### MongoDB → Oracle の場合

専用ツールはない。以下を組み合わせて使用する：
1. データ抽出：**mongoexport**。
2. 変換処理：カスタムの Python ETL スクリプト。
3. Oracle へのロード：**SQL*Loader**。
4. ドキュメント型アクセス・レイヤーの作成：**Oracle JSON Duality Views** (23c)。

---

## ツールを使用する際のベスト・プラクティス

1. **常に最初にアセスメントを実行する。** 前述のすべてのツールには、アセスメント・モードやレポート・モードがある。実際の移行を開始する前に必ず実行すること。アセスメントによって未知の複雑さが明らかになり、作業の見積もりが可能になる。

2. **段階的に移行する。** まず一部の表のサブセットをツールで移行し、入念に検証してから次のグループに進む。大規模なスキーマの場合、段階的な検証なしに一回でフル移行を試みてはいけない。

3. **生成された DDL をバージョン管理に含める。** すべてのスキーマ変換ツールは SQL ファイルを生成する。これらを移行先データベースに適用する前に Git にコミットすること。これにより監査証跡が作成され、ロールバックが可能になる。

4. **ストアド・プロシージャの変換は手動でテストする。** 自動的に 100% 完璧にプロシージャを変換できるツールはない。本番環境にデプロイする前に、変換されたすべてのプロシージャを人間がレビューするように計画を立てること。

5. **データ・ロードには PARALLEL および DIRECT パス・オプションを使用する。** SQL*Loader で大規模なデータセットをロードする場合、スループットを最大化するために常に `DIRECT=TRUE` と `PARALLEL=TRUE` を使用すること。

6. **ロードのたびに検証を行う。** 各テーブルの移行後に行カウントとハッシュ・ベースの検証クエリを実行する。詳細な検証パターンについては、`migration-data-validation.md` を参照。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- 本ファイル内の基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効。
- 21c, 23c, または 23ai としてマークされた機能は、Oracle Database 26ai 対応機能として扱う。
- 複数バージョンをサポートする環境では、リリース更新（RU）によってデフォルト値や非推奨機能が異なる場合があるため、19c と 26ai の両方で動作をテストすること。

## ソース

- [Oracle SQL Developer Migration Workbench documentation](https://docs.oracle.com/en/database/oracle/sql-developer/23.1/rptug/migration-workbench.html)
- [Oracle Zero Downtime Migration (ZDM) documentation](https://docs.oracle.com/en/database/oracle/zero-downtime-migration/index.html)
- [AWS Schema Conversion Tool User Guide](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/CHAP_Welcome.html)
- [Oracle Database 19c Utilities — SQL*Loader](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-sql-loader.html)
- [Oracle Database 19c Utilities — Data Pump Export/Import](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/datapump-overview.html)
