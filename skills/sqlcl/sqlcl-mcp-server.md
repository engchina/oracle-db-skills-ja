# SQLcl MCP サーバー

## 概要

SQLcl MCP サーバーは、Oracle Database の機能を **Model Context Protocol (MCP)** 経由で AI アシスタントに公開する、Oracle SQLcl（**25.2 以降**）の組み込み機能です。SQLcl が MCP サーバーとして動作し、データベース接続を保持して認証を処理する一方で、AI クライアント（Claude Desktop、Claude Code、VS Code Cline など）は定義済みの MCP ツール呼び出しを通じてインタラクションを駆動します。

通信には **`stdio` のみ** を使用します。AI クライアントは SQLcl を子プロセスとして起動し、stdin/stdout 経由で通信します。HTTP、SSE、またはネットワーク・ポートは使用されません。

---

## 前提条件

- **SQLcl 25.2 以降**（MCP は 24.3 以前には存在しませんでした）
- **JRE 17 または 21**

バージョンの確認：

```shell
sql -V
# SQLcl: Release 25.2.0 Production 以降が必要
```

macOS でのアップグレード：

```shell
brew upgrade sqlcl
```

---

## ステップ 1：データベース接続の保存

MCP サーバーはコマンドラインでの資格情報の入力を受け付けません。MCP サーバーを起動する前に、SQLcl の接続ストアを使用して接続を事前に保存しておく必要があります。

`-save` および `-savepwd` フラグを使用して接続し、保存します。

```shell
sql /nolog
```

```sql
conn -save my_connection -savepwd username/password@//hostname:1521/service_name
```

- `-save <名前>` — 接続に名前を付けて保存します。
- `-savepwd` — パスワードを `~/.dbtools` に安全に保存します。

MCP サーバーがパスワードを使用できるようにするには、必ず `-savepwd` を指定してパスワードを保存する **必要があります**。保存完了後、AI クライアントは `connect` MCP ツールを介してこの名前付き接続を参照します。

TNS ベースの接続の場合は、SQLcl が `tnsnames.ora` を見つけられるように `TNS_ADMIN` を設定してください。

```shell
sql /nolog
```

```sql
conn -save my_tns_connection -savepwd username/password@tns_alias
```

---

## ステップ 2：MCP サーバーの起動

`-mcp` フラグを指定して SQLcl を起動します。

```shell
sql -mcp
```

SQLcl は MCP サーバー・モードで起動し、stdin/stdout で待機します。`-mcp` 使用時のデフォルトの制限レベル（restrict level）は **4** です（最も厳しい制限。後述の制限レベルのセクションを参照）。

起動確認メッセージが表示されます：

```
---------- MCP SERVER STARTUP ----------
MCP Server started successfully on Fri Jun 13 13:52:13 WEST 2025
Press Ctrl+C to stop the server
----------------------------------------
```

異なる制限レベルを使用する場合：

```shell
sql -R 1 -mcp
```

---

## ステップ 3：AI クライアントの構成

### Claude Desktop

`~/Library/Application Support/Claude/claude_desktop_config.json`（macOS）または `%APPDATA%\Claude\claude_desktop_config.json`（Windows）を編集します：

```json
{
  "mcpServers": {
    "sqlcl": {
      "command": "/path/to/sql",
      "args": ["-mcp"]
    }
  }
}
```

TNS 接続の場合は、起動されたプロセスが `tnsnames.ora` を見つけられるように `TNS_ADMIN` を渡します（MCP クライアント・プロセスはシェルの環境変数を継承しないため）：

```json
{
  "mcpServers": {
    "sqlcl": {
      "command": "/path/to/sql",
      "args": ["-mcp"],
      "env": {
        "TNS_ADMIN": "/path/to/tns/directory"
      }
    }
  }
}
```

`sql` への絶対パスを使用してください。以下のコマンドで確認できます：

```shell
which sql
```

設定を編集した後、Claude Desktop を再起動します。

### Claude Code

`claude mcp add` コマンドを使用してサーバーを追加します：

```shell
claude mcp add sqlcl /path/to/sql -- -mcp
```

または、プロジェクト・ディレクトリ内の `.mcp.json` を手動で作成/編集します：

```json
{
  "mcpServers": {
    "sqlcl": {
      "command": "/path/to/sql",
      "args": ["-mcp"]
    }
  }
}
```

サーバーが登録されていることを確認します：

```shell
claude mcp list
```

### VS Code Cline

`cline_mcp_settings.json` を編集します：

```json
{
  "mcpServers": {
    "sqlcl": {
      "command": "/path/to/sql",
      "args": ["-mcp"],
      "disabled": false
    }
  }
}
```

---

## MCP ツール

SQLcl MCP サーバーからは 5 つのツールが公開されています。Oracle は SQLcl のリリースごとに新しいツールを追加しています。

| ツール名 | 説明 |
|------|-------------|
| `list-connections` | `~/.dbtools` に保存されているすべての Oracle Database 接続を検出し、リスト表示します。 |
| `connect` | 保存されている名前付き接続のいずれかに対して接続を確立します。 |
| `disconnect` | 現在アクティブな Oracle Database 接続を終了します。 |
| `run-sql` | 接続されているデータベースに対して、標準的な SQL クエリおよび PL/SQL コード・ブロックを実行します。 |
| `run-sqlcl` | SQLcl 固有のコマンドや拡張機能を実行します。 |

AI クライアントは、まず `list-connections` を呼び出して利用可能な接続を検出し、次に `connect` でセッションを確立し、最後に `run-sql` または `run-sqlcl` を呼び出してデータベースとやり取りします。

---

## 制限レベル (Restrict Levels)

`-R` フラグは、MCP サーバーで利用可能な SQLcl コマンドを制御します。`-mcp` が使用される場合、デフォルトは **レベル 4**（最も制限が厳しい）です。

| レベル | ブロックされるもの |
|-------|----------------|
| `0` | なし — すべてのコマンドが許可されます。 |
| `1` | ホスト/OS コマンド（`host`, `!`, `$`, `edit`） |
| `2` | レベル 1 + ファイル保存コマンド（`save`, `spool`, `store`） |
| `3` | レベル 2 + スクリプト実行（`@`, `@@`, `get`, `start`） |
| `4` | レベル 3 + 100 以上の追加コマンド — **`-mcp` のデフォルト** |

例 — デフォルトよりも少し緩和する場合：

```shell
sql -R 3 -mcp
```

制限レベルを指定した構成例：

```json
{
  "mcpServers": {
    "sqlcl": {
      "command": "/path/to/sql",
      "args": ["-R", "1", "-mcp"]
    }
  }
}
```

---

## モニタリング

### アクティビティ・ログ構成表

SQLcl は、接続先のスキーマに `DBTOOLS$MCP_LOG` 表を自動的に作成し、すべての MCP アクティビティを記録します。

```sql
SELECT id, mcp_client, model, end_point_type, end_point_name, log_message
FROM DBTOOLS$MCP_LOG;
```

これにより、各呼び出しを行った AI クライアントやモデルの情報を含め、AI主導の SQL 実行に関する完全な監査証跡が提供されます。

### V$SESSION との統合

SQLcl は MCP 接続のために Oracle セッション・メタデータを設定します：

- `V$SESSION.MODULE` — MCP クライアント名に設定されます（例：`Claude Desktop`）
- `V$SESSION.ACTION` — LLM モデル名に設定されます。

これにより、DBA は AI 主導のセッションをリアルタイムで特定および監視できます。

### クエリ・タギング

MCP サーバーを介して LLM によって生成および実行されるすべての SQL には、自動的に以下のコメントがタグ付けされます：

```sql
/* LLM in use is [model-name] */ SELECT ...
```

これにより、AWR、ASH、および `V$SQL` において、AI 生成の SQL を識別できるようになります。

---

## セキュリティに関する考慮事項

### 最小権限のデータベース・ユーザーの使用

DBA アカウントを使用するのではなく、MCP 接続用に制限された専用のデータベース・ユーザーを保存して使用してください：

```sql
CREATE USER mcp_reader IDENTIFIED BY "StrongPassword123!";
GRANT CREATE SESSION TO mcp_reader;
GRANT SELECT ON oe.orders TO mcp_reader;
GRANT SELECT ON oe.customers TO mcp_reader;
-- スキーマのイントロスペクション（構造把握）のために SELECT ANY DICTIONARY を付与：
GRANT SELECT ANY DICTIONARY TO mcp_reader;
```

MCP サーバーを起動する前に、この接続を保存します：

```sql
conn -save mcp_readonly -savepwd mcp_reader/StrongPassword123!@//host:1521/svc
```

### AI ができること、できないこと

AI は、接続したデータベース・ユーザーの権限の範囲内でのみ動作します。制限レベルの設定によって、セッション内で利用可能な SQLcl コマンドがさらに制限されます。

**DB 権限に関わらずできないこと：**
- OS ファイルシステムへのアクセス（デフォルトの制限レベルでブロックされます）
- ネットワーク接続のオープン
- データベース権限の昇格

### TNS_ADMIN は唯一サポートされている環境変数です

SQLcl MCP サーバー向けにドキュメント化されている唯一の環境変数は `TNS_ADMIN` です。環境変数を介してパスワードを渡そうとしないでください。そのためのサポートされているメカニズムはありません。すべての資格情報は `conn -save -savepwd` を使用して事前に保存しておく必要があります。

---

## よくある間違い

| 間違い | 回避策 |
|---------|-----|
| SQLcl 24.3 以前を使用している | 25.2 以降にアップグレードしてください。以前のバージョンでは MCP は利用できません。 |
| `sql -mcp` コマンドラインで資格情報を渡そうとしている | 代わりに `conn -save -savepwd` で接続を事前に保存してください。 |
| MCP 構成で `sql` への相対パスを使用している | 絶対パス（`which sql`）を使用してください。AI クライアントはシェルの PATH を継承しません。 |
| TNS 接続の構成で `env` ブロックに `TNS_ADMIN` を含め忘れている | MCP クライアント・プロセスはシェルの環境変数を継承しません。構成内で `TNS_ADMIN` を明示的に設定してください。 |
| `-savepwd` なしで接続を保存している | MCP サーバーは保存済みのパスワードなしでは接続できません。必ず `-savepwd` を含めてください。 |
| HTTP/SSE トランスポートを期待している | SQLcl MCP は stdio のみであり、ネットワーク・ポートは関与しません。 |

---

## 関連スキル

- `sqlcl-basics.md` — SQLcl のインストール、接続方法、およびコア・コマンド
- `sqlcl-cicd.md` — パイプライン内で非対話的に SQLcl を使用する方法
- `security/privilege-management.md` — Oracle ユーザーの作成と最小権限の設定
- `monitoring/top-sql-queries.md` — V$SQL タギングを介した AI 生成 SQL の特定

---

## 参考資料

- [Oracle SQLcl 25.2 ユーザーズ・ガイド](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/oracle-sqlcl-users-guide.pdf)
- [SQLcl リリース・ノート 25.2 — MCP サーバーの導入](https://www.oracle.com/tools/sqlcl/sqlcl-relnotes-25.2.html)
- [SQLcl の開始と終了 — -mcp を含む起動フラグ](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/startup-sqlcl-settings.html)

> ⚠️ 未検証：検証時点では、SQLcl 25.2 の公式 MCP ドキュメント・ページにアクセスできませんでした。このファイル内の重要な事実（5 つのツール、stdio トランスポート、`-mcp` フラグ、バージョン 25.2 以降、`DBTOOLS$MCP_LOG` ログ表、`TNS_ADMIN` 以外の環境変数は不可、`conn -save -savepwd`）は、本番環境で使用する前に最新の Oracle SQLcl ドキュメントで確認する必要があります。
