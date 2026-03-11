# Oracle Text: 全文検索

## 概要

Oracle Text（旧称 ConText および interMedia Text）は、データベース・カーネルに組み込まれた Oracle 独自の全文検索エンジンである。アプリケーション・レベルの検索ライブラリ（Lucene, Elasticsearchなど）とは異なり、Oracle Text の索引はデータベース内のデータとともに保持される。これにより、外部インフラを必要とすることなく、SQL の結合、トランザクション、およびアクセス制御に全文検索を参加させることができる。

Oracle Text は以下の用途に最適である：
- ドキュメント・リポジトリおよびコンテンツ管理システム
- 製品カタログ検索（あいまい検索、ステミング、テーマ検索など）
- 規制文書検索（契約書、申請書、通信記録など）
- ナレッジベースおよび FAQ システム
- VARCHAR2, CLOB, または XMLType 列を持つすべての表

---

## 索引タイプ

Oracle Text は主に4つの索引タイプを提供している。適切なタイプを選択することが最も重要な決定事項となる。

| 索引タイプ | 最適な用途 | 備考 |
|---|---|---|
| `CONTEXT` | 大規模ドキュメント、CLOB/XMLType の全文検索 | 最も強力。バッチまたはスケジュールによる同期が必要 |
| `CTXCAT` | 短いテキスト、カタログ/ECサイト、ランク付けされた結果 | 複雑なクエリ式をサポート。リアルタイム更新 |
| `CTXRULE` | 受信ドキュメントのルーティング/カテゴリ分け | `MATCHES` 演算子を使用。クエリ・ルールに対してドキュメントを分類 |
| `CTXXPATH` | XMLType に対する XPath クエリ | XMLType の XPath 述語を最適化 |

---

## CONTEXT 索引: 大規模ドキュメント検索

### CONTEXT 索引の作成

```sql
-- 最小限の CONTEXT 索引 (すべてデフォルト設定)
CREATE INDEX idx_article_text
    ON articles (content)
    INDEXTYPE IS CTXSYS.CONTEXT;

-- 明示的なプリファレンスを指定した CONTEXT 索引
CREATE INDEX idx_product_desc
    ON products (description)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('
        LEXER           my_lexer
        WORDLIST        my_wordlist
        STOPLIST        ctxsys.default_stoplist
        MEMORY          128M
        SYNC (ON COMMIT)
    ');

-- マルチ列索引: 複数のテキスト列を1つの索引として作成
-- まず、列を連結するユーザ・データストアを作成する
BEGIN
    CTX_DDL.CREATE_PREFERENCE('product_store', 'MULTI_COLUMN_DATASTORE');
    CTX_DDL.SET_ATTRIBUTE('product_store', 'COLUMNS', 'title, description, tags');
END;
/

CREATE INDEX idx_product_fulltext
    ON products (title)  -- 最初の列を指定。他はデータストアで定義済み
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('DATASTORE product_store SYNC (ON COMMIT)');
```

### 索引の同期モード

CONTEXT 索引は、デフォルトではリアルタイムに更新されない。新規または変更された行は索引に同期する必要がある。

```sql
-- SYNC (ON COMMIT): コミットごとに自動同期 (12c以降。オーバーヘッドあり)
PARAMETERS ('SYNC (ON COMMIT)')

-- SYNC (EVERY "interval"): スケジュールに基づいたバックグラウンド同期
PARAMETERS ('SYNC (EVERY "SYSDATE + 1/24")')  -- 1時間おきに同期

-- 手動同期 (バッチ・システムで最も一般的)
EXEC CTX_DDL.SYNC_INDEX('idx_article_text');
-- メモリー割り当てを指定する場合
EXEC CTX_DDL.SYNC_INDEX('idx_article_text', '128M');

-- 索引の最適化 (断片化したリストの結合、削除済みエントリの除去)
EXEC CTX_DDL.OPTIMIZE_INDEX('idx_article_text', 'FAST');     -- 簡易デフラグ
EXEC CTX_DDL.OPTIMIZE_INDEX('idx_article_text', 'FULL');     -- 完全マージ (低速だが徹底的)
EXEC CTX_DDL.OPTIMIZE_INDEX('idx_article_text', 'TOKEN', maxtime => 300);  -- 最大5分間実行

-- 索引付け待ちの保留ドキュメント数を確認
SELECT COUNT(*) FROM ctx_pending WHERE idx_name = 'IDX_ARTICLE_TEXT';
```

---

## CTXCAT 索引: カタログ検索

CTXCAT は、カタログ検索（短いテキスト、テキスト＋構造化フィルタの組み合わせ）用に設計されている。CONTEXT とは異なり、DML と同時に自動的に更新される（手動同期は不要）。

```sql
-- CTXCAT 索引の作成 (同期不要)
CREATE INDEX idx_product_cat
    ON products (product_name)
    INDEXTYPE IS CTXSYS.CTXCAT;

-- 構造化属性のためのサブ索引を含む CTXCAT
BEGIN
    CTX_DDL.CREATE_INDEX_SET('product_idx_set');
    CTX_DDL.ADD_INDEX('product_idx_set', 'price');        -- NUMBER型
    CTX_DDL.ADD_INDEX('product_idx_set', 'category_id');  -- NUMBER型
    CTX_DDL.ADD_INDEX('product_idx_set', 'brand');        -- VARCHAR2型
END;
/

CREATE INDEX idx_product_ctxcat
    ON products (product_name)
    INDEXTYPE IS CTXSYS.CTXCAT
    PARAMETERS ('INDEX SET product_idx_set');
```

---

## CONTAINS 演算子

`CONTAINS` は CONTEXT 索引に対する主要な検索演算子である。関連性スコア（0–100、100が最も高い関連性）を返す。

```sql
-- 基本的なキーワード検索 (単一単語)
SELECT product_id, product_name
FROM   products
WHERE  CONTAINS(description, 'widget') > 0;

-- 複数単語 (暗黙的な AND)
SELECT product_id, product_name
FROM   products
WHERE  CONTAINS(description, 'industrial widget') > 0;

-- SELECT 句での関連性スコアの取得
SELECT product_id, product_name,
       SCORE(1) AS relevance  -- SCORE() は CONTAINS のラベル(1)と一致させる必要がある
FROM   products
WHERE  CONTAINS(description, 'industrial widget', 1) > 0
ORDER  BY relevance DESC;

-- CTXCAT 索引に対する CATSEARCH
SELECT product_id, product_name
FROM   products
WHERE  CATSEARCH(product_name, 'widget', 'category_id = 5 AND price < 100') > 0;
```

---

## クエリ演算子

Oracle Text は CONTAINS 演算子の文字列内で、豊富なクエリ言語をサポートしている。

### ブール演算子

```sql
-- AND: 両方の用語が含まれること
WHERE CONTAINS(text_col, 'oracle AND database') > 0;

-- OR: いずれかの用語が含まれること
WHERE CONTAINS(text_col, 'oracle OR mysql') > 0;

-- NOT: 用語を除外 (NOT には少なくとも1つの肯定的な用語が必要)
WHERE CONTAINS(text_col, 'database NOT oracle') > 0;

-- 短縮形: & = AND, | = OR, ~ = NOT
WHERE CONTAINS(text_col, 'oracle & database ~ mysql') > 0;

-- 優先順位: NOT > AND > OR (明確にするために括弧を使用)
WHERE CONTAINS(text_col, '(oracle | postgres) & (performance ~ slow)') > 0;
```

### フレーズ検索

```sql
-- 正確なフレーズ (単語がこの順序で隣接していること)
WHERE CONTAINS(description, '{high performance widget}') > 0;

-- Near: 用語が互いに N 単語以内にあること
WHERE CONTAINS(description, 'oracle NEAR database') > 0;
WHERE CONTAINS(description, 'oracle NEAR((database,performance), 5)') > 0;
-- 5単語以内
```

### あいまい検索 (Fuzzy Search)

あいまい検索は、スペルミスに対応できるよう、類似した単語を（編集距離に基づいて）検索する。

```sql
-- "widget" に対するあいまい一致 ("wigdet", "widgit" などを見つける)
WHERE CONTAINS(description, 'fuzzy(widget)') > 0;

-- スコアの閾値と拡張制限を指定したあいまい検索
WHERE CONTAINS(description, 'fuzzy(widget, 60, 100, weight)') > 0;
-- 60 = 最小類似度スコア, 100 = 最大拡張数

-- あいまい検索と正確な一致の組み合わせ
WHERE CONTAINS(description, 'fuzzy(widgit) & premium') > 0;
```

### ステミング検索 (語幹検索)

ステミング検索は、単語の語形変化を検索する（例: "run" を検索すると "running", "ran", "runs" も見つかる）。

```sql
-- stem 演算子: すべての語形変化を検索
WHERE CONTAINS(description, 'stem(install)') > 0;
-- install, installed, installing, installation, installs が対象となる

WHERE CONTAINS(description, 'stem(connect)') > 0;
-- connect, connected, connecting, connection, connections が対象となる

-- 明示的: 正確な単語のみに一致 (ステミングなし)
WHERE CONTAINS(description, 'exact(install)') > 0;
```

### ワイルドカード検索

```sql
-- 右側方一致 (前方一致検索)
WHERE CONTAINS(description, 'manag%') > 0;
-- manage, manager, management, managing が対象となる

-- 左側方一致 (後方一致検索)
WHERE CONTAINS(description, '%tion') > 0;
-- action, connection, installation... が対象となる

-- 中間一致
WHERE CONTAINS(description, '%connect%') > 0;
-- reconnect, disconnect, interconnection... が対象となる
```

### テーマ検索

```sql
-- ABOUT: 概念的/テーマ検索 (ナレッジ・ベースが必要)
WHERE CONTAINS(description, 'about(database performance)') > 0;
-- 正確な単語が含まれていなくても、"database performance" に
-- 概念的に関連するドキュメントを検索する
```

---

## レクサーとワードリスト: 言語設定

### カスタム・レクサーの作成

```sql
-- 基本的な英語レクサー
BEGIN
    CTX_DDL.CREATE_PREFERENCE('my_english_lexer', 'BASIC_LEXER');
    CTX_DDL.SET_ATTRIBUTE('my_english_lexer', 'PRINTJOINS', '_-');   -- _ と - をトークン内に保持
    CTX_DDL.SET_ATTRIBUTE('my_english_lexer', 'MIXED_CASE', 'NO');   -- 大文字小文字を区別しない
    CTX_DDL.SET_ATTRIBUTE('my_english_lexer', 'BASE_LETTER', 'YES'); -- アクセント記号を除去
END;
/

-- 多言語レクサー
BEGIN
    CTX_DDL.CREATE_PREFERENCE('global_lexer', 'WORLD_LEXER');
END;
/
```

### あいまい/ステミング用のカスタム・ワードリスト

```sql
BEGIN
    CTX_DDL.CREATE_PREFERENCE('my_wordlist', 'BASIC_WORDLIST');
    CTX_DDL.SET_ATTRIBUTE('my_wordlist', 'FUZZY_MATCH',      'ENGLISH');
    CTX_DDL.SET_ATTRIBUTE('my_wordlist', 'FUZZY_SCORE',      '60');
    CTX_DDL.SET_ATTRIBUTE('my_wordlist', 'FUZZY_NUMRESULTS', '5000');
    CTX_DDL.SET_ATTRIBUTE('my_wordlist', 'STEMMER',          'ENGLISH');
    CTX_DDL.SET_ATTRIBUTE('my_wordlist', 'WILDCARD_MAXTERMS','5000');
END;
/

-- 索引作成時に適用
CREATE INDEX idx_articles ON articles(content)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('LEXER my_english_lexer WORDLIST my_wordlist');
```

---

## マルチ列索引

Oracle Text は、データストアを使用して複数の列を1つの検索単位として索引付けできる。

```sql
-- 複数の列を1つの索引に連結
BEGIN
    CTX_DDL.DROP_PREFERENCE('product_multistore');
    CTX_DDL.CREATE_PREFERENCE('product_multistore', 'MULTI_COLUMN_DATASTORE');
    CTX_DDL.SET_ATTRIBUTE('product_multistore', 'COLUMNS',
        'product_name, short_description, long_description, keywords, brand_name');
    CTX_DDL.SET_ATTRIBUTE('product_multistore', 'DELIMITER', 'NEWLINE');
END;
/

CREATE INDEX idx_product_search
    ON products (product_name)  -- アンカー列 (表に存在する必要がある)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('DATASTORE product_multistore SYNC (ON COMMIT)');

-- 検索はいずれかの索引付き列に一致するものを見つける
SELECT product_id, product_name
FROM   products
WHERE  CONTAINS(product_name, 'industrial grade widget') > 0;
```

### URL/ファイル・データストア

```sql
-- OS上のファイルの内容を索引付け (FILE_DATASTORE)
BEGIN
    CTX_DDL.CREATE_PREFERENCE('file_store', 'FILE_DATASTORE');
    CTX_DDL.SET_ATTRIBUTE('file_store', 'PATH', '/data/documents');
END;
/

CREATE TABLE document_index (
    doc_id    NUMBER PRIMARY KEY,
    filename  VARCHAR2(500)  -- この列にファイル・パスを格納
);

CREATE INDEX idx_documents
    ON document_index(filename)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('DATASTORE file_store');

-- URLの内容を索引付け (URL_DATASTORE)
BEGIN
    CTX_DDL.CREATE_PREFERENCE('url_store', 'URL_DATASTORE');
    CTX_DDL.SET_ATTRIBUTE('url_store', 'TIMEOUT', '30');
    CTX_DDL.SET_ATTRIBUTE('url_store', 'HTTP_PROXY', 'proxy.mycompany.com');
END;
/
```

---

## HIGHLIGHT および SNIPPET 関数

これらの関数は、Google の検索結果のように、文脈に応じた結果表示（抜粋のハイライト）を生成する。

### CTX_DOC.HIGHLIGHT

```sql
-- 格納されたドキュメント内の一致用語をハイライトする
DECLARE
    v_markup CLOB;
BEGIN
    CTX_DOC.MARKUP(
        index_name => 'IDX_ARTICLE_TEXT',
        textkey    => '42',           -- ドキュメントの主キー
        text_query => 'database performance',
        restab     => v_markup,
        starttag   => '<b>',          -- 開始タグ
        endtag     => '</b>'          -- 終了タグ
    );
    DBMS_OUTPUT.PUT_LINE(DBMS_LOB.SUBSTR(v_markup, 4000, 1));
END;
```

### CTX_DOC.SNIPPET (検索UIに最適)

`SNIPPET` は、ドキュメントの最も関連性の高い部分（コンテキスト内の一致）を抽出し、ハイライトされた用語を含む短い抜粋として返す。

```sql
DECLARE
    v_snippet VARCHAR2(4000);
BEGIN
    CTX_DOC.SNIPPET(
        index_name => 'IDX_ARTICLE_TEXT',
        textkey    => TO_CHAR(42),    -- VARCHAR2型である必要がある
        text_query => 'database performance',
        restab     => v_snippet,
        starttag   => '<em>',
        endtag     => '</em>',
        separator  => '...',          -- 抜粋間の区切り文字
        numsnippets=> 3,              -- 抜粋フラグメントの数
        snippetlen => 200             -- 抜粋ごとの文字数
    );
    DBMS_OUTPUT.PUT_LINE(v_snippet);
END;
```

### SQL クエリでの CTX_DOC の使用

```sql
-- 検索結果に対するスニペットのインライン生成
SELECT a.article_id,
       a.title,
       SCORE(1) AS relevance,
       CTX_DOC.SNIPPET_QUERY('IDX_ARTICLE_TEXT',
           ROWID,
           'database performance',
           starttag  => '<b>',
           endtag    => '</b>',
           numsnippets => 2) AS excerpt
FROM   articles a
WHERE  CONTAINS(a.content, 'database performance', 1) > 0
ORDER  BY relevance DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## 索引のメンテナンス: 同期と最適化

### 同期: 新規/更新ドキュメントの索引付け

```sql
-- 手動同期: 保留中の変更を索引に反映
EXEC CTX_DDL.SYNC_INDEX('IDX_ARTICLE_TEXT');

-- メモリー・チューニング付き (大きいほど大量バッチで高速)
EXEC CTX_DDL.SYNC_INDEX('IDX_ARTICLE_TEXT', '256M');

-- 保留中のドキュメント数を確認
SELECT COUNT(*) FROM ctx_pending WHERE idx_name = 'IDX_ARTICLE_TEXT';

-- DBMS_SCHEDULER を使用した同期のスケジュール設定
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'SYNC_ARTICLE_INDEX',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'CTX_DDL.SYNC_INDEX(''IDX_ARTICLE_TEXT'', ''64M'');',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=MINUTELY; INTERVAL=15',  -- 15分ごと
        enabled         => TRUE
    );
END;
```

### 最適化: 索引のデフラグ

ドキュメントの更新や削除が繰り返されると、CONTEXT 索引は断片化していく。最適化によって断片化したポスティング・リストがマージされる。

```sql
-- 高速最適化: 簡易的なクリーンアップ
EXEC CTX_DDL.OPTIMIZE_INDEX('IDX_ARTICLE_TEXT', 'FAST');

-- 完全最適化: すべてのマージ (大規模な索引では数時間かかる場合がある)
EXEC CTX_DDL.OPTIMIZE_INDEX('IDX_ARTICLE_TEXT', 'FULL');

-- トークン・ベースの最適化: 最大 N 秒間実行
EXEC CTX_DDL.OPTIMIZE_INDEX('IDX_ARTICLE_TEXT', 'TOKEN', maxtime => 1800);  -- 30分

-- 夜間の最適化スケジュール設定
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'OPTIMIZE_ARTICLE_INDEX',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'CTX_DDL.OPTIMIZE_INDEX(''IDX_ARTICLE_TEXT'', ''FAST'');',
        start_date      => TRUNC(SYSTIMESTAMP) + 1 + 2/24,  -- 翌日の午前2時
        repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0',
        enabled         => TRUE
    );
END;
```

---

## Oracle Text 索引の監視

```sql
-- 索引のステータスと統計
SELECT idx_name, idx_type, idx_status, idx_language,
       idx_option, idx_docid_count, idx_sync_interval
FROM   ctx_indexes;

-- 同期中の索引エラー (トラブルシューティングに重要)
SELECT * FROM ctx_index_errors
WHERE  err_index_name = 'IDX_ARTICLE_TEXT'
ORDER  BY err_timestamp DESC;

-- トークン統計 (クエリ・パフォーマンスの分析に便利)
SELECT token_text, token_count, token_doc_count
FROM   dr$idx_article_text$i  -- 索引表: dr$<索引名>$i
WHERE  token_text = 'database'
ORDER  BY token_doc_count DESC;

-- 同期待ちの保留行
SELECT * FROM ctx_pending
WHERE  idx_name = 'IDX_ARTICLE_TEXT';

-- ユーザ定義のプリファレンス
SELECT pre_name, pre_class, pre_object, pre_attribute, pre_value
FROM   ctx_user_preferences;
```

---

## セクション・グループ: 構造化テキスト検索

セクション・グループを使用すると、HTML または XML ドキュメントの特定のセクションに限定して検索できる。

```sql
-- HTML セクション・グループ
BEGIN
    CTX_DDL.CREATE_SECTION_GROUP('html_sections', 'HTML_SECTION_GROUP');
    CTX_DDL.ADD_ZONE_SECTION('html_sections', 'title',  'title');  -- HTML <title>
    CTX_DDL.ADD_ZONE_SECTION('html_sections', 'heading','h1');     -- HTML <h1>
    CTX_DDL.ADD_ZONE_SECTION('html_sections', 'para',   'p');      -- HTML <p>
END;
/

CREATE INDEX idx_html_content ON web_pages(html_content)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('SECTION GROUP html_sections FORMAT HTML');

-- 特定の HTML セクション内を検索
SELECT page_id, url
FROM   web_pages
WHERE  CONTAINS(html_content, 'oracle WITHIN title') > 0;
-- "oracle" が <title> タグ内に含まれるドキュメントのみ一致する

-- XML セクション・グループ
BEGIN
    CTX_DDL.CREATE_SECTION_GROUP('contract_sections', 'XML_SECTION_GROUP');
    CTX_DDL.ADD_ZONE_SECTION('contract_sections', 'terms',      'Terms');
    CTX_DDL.ADD_ZONE_SECTION('contract_sections', 'definitions','Definitions');
    CTX_DDL.ADD_ZONE_SECTION('contract_sections', 'liability',  'Liability');
END;
/

-- Liability セクション内で "damages" に言及している契約書を検索
SELECT contract_id, contract_number
FROM   contracts
WHERE  CONTAINS(contract_xml_text, 'damages WITHIN liability') > 0;
```

---

## ベスト・プラクティス

- **頻繁に更新される短いテキスト（製品名、タイトル、タグなど）には `CTXCAT` を選択する。** 自動的に更新が反映される。大規模ドキュメントには `CONTEXT` を使用する。
- **許容できるデータの鮮度に基づいて `SYNC_INDEX` をスケジュールする。** `SYNC (ON COMMIT)` はオーバーヘッドがあるため、スケジューラを介した `SYNC (EVERY n MINUTES)` の方が通常は適している。
- **定期的に `OPTIMIZE_INDEX` を実行する**（アクティブなシステムでは週次または夜間）。断片化した索引は関連性スコアの低下を招く。
- **関連性スコアをランク付けに利用し、** `CONTAINS > 閾値` でフィルタリングすることで、関連性の低い結果を除外する。
- **全文検索が本当に必要な列のみを索引付けする。** Oracle Text 索引は、元のデータの 20–40% 程度のストレージを消費することがある。
- ドキュメント全体ではなく特定の部分を検索対象にする場合は、構造化ドキュメント用の**セクション・グループ**を使用する。
- 本番環境に移行する前に、実際のデータを使用して**あいまい検索やステミングのパラメータをテストする。** 過剰にあいまいな一致は、無関係な結果を大量に返してしまう。
- 列ごとに個別の索引を作成するのではなく、**`MULTI_COLUMN_DATASTORE`** を使用する。すべてのテキスト列を1つの索引にまとめる方が、`CONTAINS(col1, q) > 0 OR CONTAINS(col2, q) > 0` のように照会するよりも高速である。

---

## よくある間違い

### 間違い 1: DML 直後の照会 (同期前)

```sql
INSERT INTO articles (article_id, content) VALUES (999, 'Oracleパフォーマンスに関する新しい記事');
COMMIT;

-- 誤り: 索引が同期されていないため、0行が返される可能性がある
SELECT * FROM articles WHERE CONTAINS(content, 'Oracleパフォーマンス') > 0;

-- 正解: リアルタイムの検索が必要な場合は、確実に同期させる
EXEC CTX_DDL.SYNC_INDEX('IDX_ARTICLES');
SELECT * FROM articles WHERE CONTAINS(content, 'Oracleパフォーマンス') > 0;
```

### 間違い 2: CONTAINS の代わりに LIKE を使用する

```sql
-- 全文検索としては誤り: LIKE はフル・テーブル・スキャンを実行し、Text 索引を無視する
WHERE description LIKE '%高速な製品%'

-- 正解: 索引付けされた全文検索には CONTAINS を使用する
WHERE CONTAINS(description, '{高速な製品}') > 0
```

### 間違い 3: 大規模な削除/更新の後に最適化を忘れる

多数のドキュメントを削除または更新した後に `OPTIMIZE_INDEX` を実行しないと、索引に古い「ゴミ」エントリが蓄積される。これは索引を肥大化させ、クエリ・パフォーマンスを低下させる。大量の DML の後は、`CTX_DDL.OPTIMIZE_INDEX` を `FAST` または `FULL` で実行すること。

### 間違い 4: バイナリ・ドキュメント・タイプに対するフィルタ設定の誤り

Word文書、PDF、HTML（BLOBとして格納）などを索引付けする場合、`FILTER` プリファレンスを `INSO_FILTER` または `AUTO_FILTER` に設定する必要がある。これがないと、Oracle Text は生のバイナリ・コンテンツ（ゴミ）を索引付けしてしまう。

```sql
BEGIN
    CTX_DDL.CREATE_PREFERENCE('auto_filter', 'AUTO_FILTER');
END;
/

CREATE INDEX idx_docs ON documents(content_blob)
    INDEXTYPE IS CTXSYS.CONTEXT
    PARAMETERS ('FILTER auto_filter FORMAT COLUMN format_col');
```

### 間違い 5: ランク付けに SCORE() を使用しない

```sql
-- 関連性順のソートなし
SELECT * FROM articles WHERE CONTAINS(content, 'database', 1) > 0;
-- 順序が不定の結果が返される

-- 常に SCORE() を使用して関連性の高い順にソートする
SELECT article_id, title, SCORE(1) AS rel
FROM   articles
WHERE  CONTAINS(content, 'database', 1) > 0
ORDER  BY rel DESC;
```

---

## Oracleバージョンに関する注意 (19c vs 26ai)

- このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19cに有効。
- 21c、23c、または23aiとしてマークされた機能は、Oracle Database 26ai対応機能として扱う。混在バージョン構成の場合は、19c互換の代替案を保持すること。
- 両方のバージョンをサポートする環境では、リリースの更新によってデフォルトや非推奨が異なる可能性があるため、19cと26aiの両方で構文とパッケージの動作をテストすること。

## ソース

- [Oracle Text リファレンス 19c (CCREF)](https://docs.oracle.com/en/database/oracle/oracle-database/19/ccref/)
- [Oracle Text アプリケーション・開発者ガイド 19c (CCAPP)](https://docs.oracle.com/en/database/oracle/oracle-database/19/ccapp/)
- [Oracle Database 19c SQL言語リファレンス — CONTAINS](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/)
