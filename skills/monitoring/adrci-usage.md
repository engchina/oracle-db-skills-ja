# ADRCI の使用方法

## 概要

`adrci` (Automatic Diagnostic Repository Command Interpreter) は、自動診断リポジトリ (ADR) を管理するための Oracle のコマンドライン・インターフェースである。Oracle 11g で導入され、診断データの検索、インシデントの管理、Oracle Support 向けの診断情報のパッケージ化、および古い診断コンテンツのパージを行うための標準ツールとなっている。

すべての DBA は `adrci` に精通している必要がある。なぜなら、`adrci` はアラート・ログ、トレース・ファイル、インシデント、および問題に対して、構造化されフィルタリング可能なアクセスを提供するからである。これは、`grep` や `tail` のような単純なファイル・システム・ツールでは不可能、あるいは実用的ではない機能である。特に、Oracle Support 向けの診断パッケージ (IPS パッケージ) の作成や、複数の関連するインシデントの相関分析を行う際に極めて重要となる。

---

## ADR リポジトリの構造

ADR は、予測可能なディレクトリ階層で構成されたファイル・ベースのリポジトリである。この構造を理解しておくことは、ファイル・システムを直接ナビゲートする場合や、`adrci` の出力を解釈する場合に役立つ。

```
$ADR_BASE/
└── diag/
    └── rdbms/                          ← 製品タイプ
        └── <db_name>/                  ← DB一意名（小文字）
            └── <instance_name>/        ← インスタンス名 (ADRホーム)
                ├── alert/              ← XMLアラート・ログ (log.xml)
                ├── trace/              ← トレース・ファイル + テキスト形式アラート・ログ
                ├── incident/           ← インシデントごとのディレクトリ
                │   └── incdir_<id>/    ← 特定のインシデントに関するトレース・ファイル
                ├── incpkg/             ← IPSパッケージのステージング領域
                ├── cdump/              ← コア・ドンプ・ファイル
                ├── hm/                 ← ヘルス・モニターの結果
                ├── sweep/              ← スイープ（自動アクション）の結果
                └── metadata/           ← ADR内部メタデータ (SQLite DB)
```

### ADR ホームの確認

```sql
-- SQL*Plus または任意の SQL クライアントから実行
SELECT name, value
FROM   v$diag_info
ORDER BY name;
```

主な項目:

| 項目名 | 説明 |
|---|---|
| `ADR Base` | ADR のルート (`$ORACLE_BASE` または `DIAGNOSTIC_DEST`) |
| `ADR Home` | このインスタンスのフルパス |
| `Diag Alert` | XML アラート・ログのディレクトリ |
| `Diag Trace` | トレース・ファイルのディレクトリ（テキスト形式アラート・ログも含まれる） |
| `Diag Incident` | インシデント・ディレクトリ |

### 複数の ADR ホーム

単一のサーバーに複数の ADR ホームが存在する場合がある（データベース・インスタンス、リスナー、ASM など、Oracle 製品のインスタンスごとに 1 つ）。`adrci` は、それらすべてを同時に操作することができる。

```bash
# このサーバー上のすべての ADR ホームを表示
adrci> SHOW HOMES

# 出力例:
# ADR Homes:
# diag/rdbms/orcl/orcl
# diag/rdbms/testdb/testdb
# diag/tnslsnr/myserver/listener
# diag/clients/user_oracle/host_1234567890_11
```

---

## adrci の起動

```bash
# 対話モードで起動（環境変数 ORACLE_BASE を使用して ADR ベースを特定）
adrci

# 非対話モードで単一のコマンドを実行
adrci exec="SHOW ALERT -TAIL 50"

# 非対話モードで複数のコマンドを実行（セミコロンで区切る）
adrci exec="SET HOMEPATH diag/rdbms/orcl/orcl; SHOW ALERT -TAIL 50"

# スクリプト・ファイルから adrci コマンドを実行
adrci script=/path/to/adrci_commands.sql
```

---

## adrci コマンド・リファレンス

### ナビゲーションと構成

```bash
# 利用可能なすべての ADR ホームを表示
SHOW HOMES

# 作業対象の ADR ホームを設定（単一）
SET HOMEPATH diag/rdbms/orcl/orcl

# 複数のホームを設定（インスタンスを跨いだクエリ用）
SET HOMEPATH diag/rdbms/orcl/orcl diag/rdbms/testdb/testdb

# 現在のホーム設定を表示
SHOW HOMEPATH

# ADR ベースのパスを表示
SHOW BASE

# コマンドのヘルプを表示
HELP
HELP SHOW INCIDENT
HELP IPS
```

### アラート・ログの表示

```bash
# アラート・ログ全体を表示（ログが大きい場合は注意）
SHOW ALERT

# 最後の N 行を表示（tail -n に相当）
SHOW ALERT -TAIL 100

# 最後の N 行を表示し、監視を継続する（tail -f に相当）
SHOW ALERT -TAIL 50 -F

# 条件を指定してアラート・ログをフィルタリング（WHERE 句の構文）
SHOW ALERT -P "MESSAGE_TEXT LIKE '%ORA-%'"

# 時間範囲でフィルタリング
SHOW ALERT -P "ORIGINATING_TIMESTAMP > TIMESTAMP '2026-03-06 08:00:00'"

# 条件の組み合わせ
SHOW ALERT -P "ORIGINATING_TIMESTAMP > TIMESTAMP '2026-03-06 00:00:00' AND MESSAGE_TEXT LIKE '%ORA-00600%'"

# ターミナルに出力（ページャーが設定されていない場合のデフォルトの動作）
SHOW ALERT -TERM

# ADR 以外の場所にあるアラート・ファイルを指定して表示
SHOW ALERT -FILE /path/to/alert_file
```

`adrci` のフィルタリング（述語）では、基盤となる XML スキーマの列名を使用する。`SHOW ALERT` のフィルタリングに使用される主な列は以下の通り：

| 列名 | 説明 |
|--------|-------------|
| `ORIGINATING_TIMESTAMP` | メッセージが生成された時刻 |
| `MESSAGE_TEXT` | 実際のログ・メッセージ |
| `MESSAGE_TYPE` | メッセージ・タイプの数値 (1=Unknown, 2=Incident, 3=Error, 4=Warning, 5=Notification, 6=Trace) |
| `COMPONENT_ID` | メッセージを生成した Oracle コンポーネント |

### インシデントの管理

**インシデント (Incident)** とは、重大なエラーの 1 回の発生を指す。各インシデントには一意の ID が割り当てられ、`$ADR_HOME/incident/incdir_<id>/` の下にそのエラーに関連するすべての診断データを格納する専用のディレクトリが作成される。

**問題 (Problem)** とは、同じ根本原因を共有するインシデントのグループであり、**問題キー (Problem Key)**（例：`ORA 600 [kcbz_check_objd_typ_3]`）によって識別される。

```bash
# すべてのインシデントを表示（新しい順）
SHOW INCIDENT

# 詳細モードでインシデントを表示
SHOW INCIDENT -MODE DETAIL

# 特定のインシデントを表示
SHOW INCIDENT -P "INCIDENT_ID=12345"

# 時間範囲でインシデントを表示
SHOW INCIDENT -P "CREATE_TIME > TIMESTAMP '2026-03-06 00:00:00'"

# 問題キーでインシデントを表示
SHOW INCIDENT -P "PROBLEM_KEY LIKE '%ORA-00600%'"

# すべての問題（グループ化されたインシデント）を表示
SHOW PROBLEM

# 詳細モードで問題を表示
SHOW PROBLEM -MODE DETAIL

# 特定の問題を表示
SHOW PROBLEM -P "PROBLEM_ID=3"
```

`SHOW INCIDENT` の出力例：
```
ADR Home = /u01/app/oracle/diag/rdbms/orcl/orcl:
*************************************************************************
                                                     INCIDENT_ID PROBLEM_KEY                 CREATE_TIME
                                                     ----------- --------------------------- ----------------------------------------
                                                           12345 ORA 600 [kcbz_check_objd_t] 2026-03-06 14:23:15.000000 +00:00
                                                           12344 ORA 7445 [sigsegv]          2026-03-05 09:11:02.000000 +00:00
2 rows fetched
```

### トレース・ファイルの表示

```bash
# インシデントに関連付けられたトレース・ファイルを表示
SHOW TRACEFILE -I 12345

# ADR ホーム内のすべてのトレース・ファイルを表示
SHOW TRACEFILE

# パターンに一致するトレース・ファイルを表示
SHOW TRACEFILE "%ora_12345%"

# 特定のトレース・ファイルを表示
SHOW TRACE /u01/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_12345/orcl_ora_12345.trc

# トレース・ファイルの最後の N 行を表示
SHOW TRACE /path/to/file.trc -TAIL 100
```

---

## IPS: インシデント・パッケージング・システム

**インシデント・パッケージング・システム (IPS)** は、1 つ以上のインシデントに関連するすべての診断データを自動的に収集し、Oracle Support へのアップロードに適した ZIP ファイルにパッケージ化する機能である。これは診断データを収集するための正しい方法であり、関連するすべてのファイル（トレース・ファイル、アラート・ログの抜粋、システム状態のダンプ、SQL 計画ベースラインなど）が確実に含まれるようになる。

### IPS パッケージの作成

```bash
# 特定のインシデントに対するパッケージを作成
IPS CREATE PACKAGE INCIDENT 12345

# 時間範囲を指定してパッケージを作成
IPS CREATE PACKAGE TIMEWINDOW '2026-03-06 14:00:00' '2026-03-06 15:00:00'

# 特定の問題に対するパッケージを作成（同じ根本原因を持つすべてのインシデント）
IPS CREATE PACKAGE PROBLEM 3

# パッケージを作成し、即座に ZIP ファイルを生成
IPS CREATE PACKAGE INCIDENT 12345 CORRELATE BASIC

# 完全な関連付け（同じ問題に起因する関連インシデントを含む）を行って作成
IPS CREATE PACKAGE INCIDENT 12345 CORRELATE ALL
```

### パッケージへのファイルの追加と削除

```bash
# 現在のパッケージを表示
IPS SHOW PACKAGE

# 特定のパッケージの内容を表示
IPS SHOW PACKAGE PACKAGE 1

# ファイルをパッケージに追加
IPS ADD FILE /path/to/additional_trace.trc PACKAGE 1

# インシデントを既存のパッケージに追加
IPS ADD INCIDENT 12346 PACKAGE 1

# ファイルをパッケージから削除
IPS REMOVE FILE /path/to/file.trc PACKAGE 1
```

### ZIP ファイルの生成

```bash
# 現在のディレクトリにパッケージ 1 の ZIP ファイルを生成
IPS GENERATE PACKAGE 1 IN /tmp

# 指定したディレクトリに ZIP ファイルが作成される
# 出力例: /tmp/ORA600_20260306_142315_COM_1.zip
```

### Oracle Support へのアップロード

ZIP ファイルを生成した後の手順：
1. My Oracle Support (MOS) (`support.oracle.com`) にログインする。
2. サービス・リクエスト (SR) を作成または開く。
3. ZIP ファイルを SR にアップロードする。
4. SR の説明欄にインシデント ID と問題キーを貼り付ける。

---

## インシデントの関連付け

関連付け (Correlation) とは、同じ根本原因を共有、または同じ時間帯に発生した、関連するインシデントを特定するプロセスである。`adrci` は、IPS 使用時に自動的に、または `SHOW INCIDENT` のフィルタリングによって手動でこれを行うことができる。

```bash
# インシデント 12345 に関連付けられたインシデントを表示
IPS GET METADATA INCIDENT 12345

# 特定の問題（同じ問題キー）に関するすべてのインシデントを表示
SHOW INCIDENT -P "PROBLEM_ID=3"

# 手動での関連付け：特定のインシデントの前後 30 分以内のインシデント
SHOW INCIDENT -P "CREATE_TIME > TIMESTAMP '2026-03-06 14:00:00' AND CREATE_TIME < TIMESTAMP '2026-03-06 14:30:00'"
```

SQL を使用した、より広範な関連付けビュー：

```sql
-- インシデントとその問題キー、発生時刻
SELECT i.incident_id,
       i.create_time,
       p.problem_key,
       i.status
FROM   v$diag_incident i
JOIN   v$diag_problem  p ON i.problem_id = p.problem_id
ORDER BY i.create_time DESC
FETCH FIRST 50 ROWS ONLY;
```

```sql
-- 問題ごとのインシデント数（再発している問題の特定）
SELECT problem_id,
       problem_key,
       last_incident_time,
       COUNT(*) OVER (PARTITION BY problem_id) AS incident_count
FROM   v$diag_problem
ORDER BY last_incident_time DESC;
```

---

## 古い診断データのパージ

ADR データは時間の経過とともに蓄積される。Oracle にはデフォルトの保持ポリシーがあるが、特にインシデント発生率が高いシステムやディスク容量が限られているシステムでは、DBA が積極的にパージを管理する必要がある。

### デフォルトの保持ポリシー

これらのデフォルト値は、`SHOW CONTROL` で確認できる `LONGP_POLICY`（長期保持コンテンツ、デフォルトは 8760 時間 = 1 年）と `SHORTP_POLICY`（短期保持コンテンツ、デフォルトは 720 時間 = 30 日）の設定に対応している。

| データ型 | デフォルト保持期間 | ポリシー |
|-----------|------------------|--------|
| インシデント | 1 年 (8760 時間) | LONGP_POLICY |
| アラート・ログ (XML) | 1 年 (8760 時間) | LONGP_POLICY |
| トレース・ファイル | 30 日 (720 時間) | SHORTP_POLICY |
| コア・ドンプ | 30 日 (720 時間) | SHORTP_POLICY |

> **注意:** 自身の環境の現在の設定を確認するには、`adrci` 内で `SHOW CONTROL` を実行すること。デフォルト値は Oracle のバージョンやパッチ・レベルによって異なる場合がある。

### 保持ポリシーの表示と変更

```bash
# 現在の保持ポリシーを表示
SHOW CONTROL

# 短期保持ポリシーを 30 日に変更（時間単位：30 * 24 = 720）
# SHORTP_POLICY はトレース・ファイル、コア・ドンプ、パッケージ情報を対象とする
SET CONTROL (SHORTP_POLICY=720)     -- 30日（デフォルト）

# 長期保持ポリシーを 180 日に変更（時間単位：180 * 24 = 4320）
# LONGP_POLICY はインシデント・データ、インシデント・ドンプ、アラート・ログを対象とする
SET CONTROL (LONGP_POLICY=4320)     -- 180日

# 現在の ADR ホームのディスク使用量を表示
SHOW CONTROL
```

> **注意:** `SET CONTROL` に指定する値は、分ではなく **時間** 単位である。デフォルトの `SHORTP_POLICY` は 720 時間 (30 日)、`LONGP_POLICY` は 8760 時間 (365 日) である。

### 特定のデータのパージ

```bash
# 保持ポリシーより古いすべてのデータをパージ（自動パージ）
PURGE

# 指定した期間より古いデータをパージ（-AGE の値は 分 単位）
PURGE -AGE 10080 -TYPE INCIDENT     -- 10080分（7日）より古いインシデント
PURGE -AGE 43200 -TYPE TRACE        -- 43200分（30日）より古いトレース・ファイル

# 特定のインシデントをパージ（インシデント・ディレクトリとそのトレース・ファイルを削除）
PURGE -I 12344

# 範囲を指定してインシデントをパージ
PURGE -I 12340 12344
```

### スクリプトによるパージの自動化

```bash
#!/bin/bash
# purge_adr.sh — ADR をクリーンに保つために cron で週次実行する例
# 90日（129600分）より古いインシデント、30日（43200分）より古いトレースを削除

ORACLE_SID=orcl
export ORACLE_SID ORACLE_BASE=/u01/app/oracle

adrci exec="SET HOMEPATH diag/rdbms/orcl/orcl; PURGE -AGE 129600 -TYPE INCIDENT; PURGE -AGE 43200 -TYPE TRACE"
```

---

## 高度な adrci クエリ

### 複数インスタンスでの HOME 句の使用

```bash
# 複数の ADR ホームに対して同時にインシデントをクエリ
SET HOMEPATH diag/rdbms/orcl/orcl diag/rdbms/testdb/testdb

# 両方のホームを検索対象にする
SHOW INCIDENT

# 単一のホームに戻す
SET HOMEPATH diag/rdbms/orcl/orcl
```

### 非対話型スクリプト

```bash
# シェル・スクリプトから一連の adrci コマンドを実行
adrci << 'EOF'
SET HOMEPATH diag/rdbms/orcl/orcl
SHOW ALERT -P "ORIGINATING_TIMESTAMP > TIMESTAMP '2026-03-06 00:00:00' AND MESSAGE_TEXT LIKE '%ORA-%'" -OUT /tmp/todays_errors.txt
SHOW PROBLEM
SHOW INCIDENT -MODE BRIEF
EOF
```

```bash
# exec を使用して単一のコマンドを実行
adrci exec="SET HOMEPATH diag/rdbms/orcl/orcl; SHOW ALERT -TAIL 50" > /tmp/alert_tail.txt
```

### インシデント・トレース・ファイルのパス抽出

```sql
-- 特定のインシデントに関するすべてのトレース・ファイル・パスを SQL から取得
SELECT trace_filename
FROM   v$diag_trace_file
WHERE  con_id = 0  -- CDBルートまたは非CDB
ORDER BY change_time DESC;
```

```bash
# adrci 内：直近のインシデントの詳細モードを表示してパスを確認
SHOW INCIDENT -MODE DETAIL -P "CREATE_TIME > TIMESTAMP '2026-03-06 00:00:00'"
```

---

## ベスト・プラクティス

1. **Oracle Support 向けの診断パッケージ作成には必ず `adrci` を使用すること。** トレース・ファイルを個別に手動で ZIP 圧縮すると、Oracle Support が必要とするメタデータ、XML アラート・ログの抜粋、および関連データが欠落する。IPS ワークフローを使用することで、情報の完全性が保証される。

2. **自動パージを設定すること。** 定期的なパージを行わないと、稼働率の高いシステムでは ADR が数十から数百 GB を消費することがある。週次の `PURGE` コマンドをスケジュールし、ディスク容量が厳しい場合はトレース・ファイルの保持期間を短縮することを検討する。

3. **リアルタイムのトラブルシューティングには `SHOW ALERT -TAIL -F` を使用すること。** これは `tail -f` の `adrci` 版であり、ローテートされる可能性のあるファイル記述子ではなく、XML 形式から読み取るため、より信頼性が高い。

4. **ORA-00600 や ORA-07445 が発生した際は、必ずインシデント ID を控えること。** インシデント ID は、アラート・ログのエントリとトレース・ファイル・ディレクトリを紐付けるため、その後の分析や IPS パッケージ作成を容易にする。

5. **grep などの外部ツールではなく、述語 (Predicates) を使用してフィルタリングすること。** `adrci` の述語はインデックス化されたメタデータに対して動作するため、生のログ・ファイルをテキスト検索するよりも高速かつ正確である。

6. **パッケージ化する前にインシデントを関連付けること。** `IPS CREATE PACKAGE INCIDENT <id> CORRELATE ALL` を使用して、同じ根本原因に関連するインシデントをサポート・パッケージに含めるようにする。これにより、Oracle Support による解決が大幅に早まることが多い。

7. **ADR のディスク使用量を監視すること。** ディスク容量の監視対象に `$ADR_HOME` を含める。インシデントが大量発生（例：再発する ORA-00600 など）した場合、数時間でファイル・システムがいっぱいになる可能性がある。

---

## よくある間違いと回避方法

**間違い: 正しいホームを設定せずに `adrci` を実行する。**
複数のホームが存在し、`SET HOMEPATH` を実行しなかった場合、`adrci` は最初に見つかったホームを使用するか、すべてのホームをクエリする。すべての `adrci` セッションまたはスクリプトの最初で、明示的にホームを設定することを推奨する。

**間違い: OS から手動でトレース・ファイルを削除する。**
`$ADR_HOME/trace/` または `$ADR_HOME/incident/` から直接ファイルを削除すると、ADR のメタデータをバイパスすることになり、`adrci` でエラーを引き起こす孤立したエントリが残ってしまう。削除には必ず `adrci` 内の `PURGE` を使用すること。

**間違い: `CORRELATE ALL` を指定せずに IPS パッケージを作成する。**
最小限のパッケージ（`CORRELATE BASIC` または関連付けなし）では、関連するバックグラウンド・プロセスからの重要なトレース・ファイルが漏れることが多い。原因が不明確な場合は、常に `CORRELATE ALL` を使用すること。

**間違い: `IPS GENERATE` で `IN /path` を指定し忘れる。**
出力先パスを指定しない場合、ZIP ファイルは現在のディレクトリに作成される。現在のディレクトリに十分な空き容量がない場合や、一時的な場所で後に削除される可能性がある。常に十分な空き容量のある明示的な保存先パスを指定すること。

**間違い: `SHOW ALERT` がリアルタイムのデータを表示していると思い込む。**
`adrci` は XML アラート・ログから読み取る。XML ログはテキスト・ログと同期して更新されるが、非常に負荷の高いシステムでは、わずかな遅延が発生する場合がある。ライブ監視には `SHOW ALERT -TAIL -F` を使用するか、短期間の間隔で実行するスクリプトを使用すること。

**間違い: `PURGE -AGE` の単位を分ではなく日と勘違いする。**
`PURGE -AGE 30` は、30 日ではなく 30 分より古いデータをパージする。30 日を分に換算すると 43,200 分である。本番環境でパージ・コマンドを実行する前に、必ず単位を再確認すること。

**間違い: `PURGE -AGE` の単位と `SET CONTROL` の単位を混同する。**
`PURGE -AGE` は **分** 単位だが、`SET CONTROL (SHORTP_POLICY/LONGP_POLICY)` は **時間** 単位である。これらは異なるコマンドの異なる単位である。

---

## Oracle バージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c/23c 以降の機能は、Oracle Database 26ai 対応機能として扱う。
- 19c と 26ai では、リリース更新によってデフォルト設定や非推奨機能が異なる場合があるため、両方の環境をサポートする場合は構文とパッケージの動作をテストすること。

## ソース

- [Oracle Database 19c Administrator's Guide — Managing Diagnostic Data](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/diagnosing-and-resolving-problems.html)
- [Oracle Database 19c Utilities — ADRCI: ADR Command Interpreter](https://docs.oracle.com/en/database/oracle/oracle-database/19/sutil/oracle-adr-command-interpreter-adrci.html)
- [ADRCI SHOW ALERT syntax (12c reference, applies to 19c)](https://docs.oracle.com/database/121/SUTIL/GUID-8D62D6A0-99F4-465C-B088-5CCF259B7D80.htm)
- [ADRCI PURGE syntax](https://docs.oracle.com/database/121/SUTIL/GUID-92DD451B-C3A1-48D7-A147-3296E75572CB.htm)
- [ADRCI SET CONTROL syntax](https://docs.oracle.com/database/121/SUTIL/GUID-68ED8877-1132-45F1-8297-E1CCF8D34D98.htm)
- [Oracle Database 19c Reference — V$DIAG_INFO](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-DIAG_INFO.html)
