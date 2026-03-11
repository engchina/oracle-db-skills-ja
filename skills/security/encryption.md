# Oracle Database の暗号化 (Encryption)

## 概要

Oracle の暗号化は、ストレージ・レイヤーで不正アクセスからデータを保護する。つまり、攻撃者がデータファイル、バックアップ・テープ、または表領域のエクスポートを盗んだとしても、暗号化キーがなければデータを読み取ることはできない。これは、アクセス制御（不正なクエリを防止）や監査（不正なアクセスを検出）とは別の保護レイヤーである。これらが合わさることで、多層防御戦略が形成される。

Oracle は主に 2 つの暗号化メカニズムを提供している。

-   **透過的データ暗号化 (TDE)**: ディスク上のデータファイル、表領域、および個々の列を暗号化する。暗号化と復号はアプリケーションに対して透過的であり、クエリ、DML、および DDL は以前とまったく同様に動作する。
-   **DBMS_CRYPTO**: アプリケーション・ロジックで特定の値を暗号化するための PL/SQL API。TDE が提供する範囲を超えて、アプリケーションが暗号化を制御する必要がある場合に有用である。

TDE には **Oracle Advanced Security Option** が必要である。これは、オンプレミスの Oracle Database Enterprise Edition（19c を含む）では別途ライセンスが必要な有償オプションである。Oracle Cloud の特定のデータベース・サービス・ティア（BaseDB EE-HP、BaseDB EE-EP、ExaDB）では、追加料金なしで含まれている。使用する特定のバージョンやデプロイメント・タイプについては、常に最新の Oracle 公式ライセンス情報ユーザー・マニュアルを確認すること。

---

## 透過的データ暗号化 (TDE) のアーキテクチャ

TDE は 2 階層のキー階層を使用する。

```
Oracle ウォレット / キー・ストア
  └── マスター暗号化キー (MEK)
        └── 表/表領域暗号化キー (DEK — データの暗号化キー)
              └── ディスク上の暗号化されたデータ・ブロック
```

**マスター暗号化キー (MEK)** は、Oracle ウォレット（パスワードで保護された PKCS#12 コンテナまたはハードウェア HSM）に保存される。**データ暗号化キー (DEK)** は、MEK によって暗号化された状態でデータベース内部に保存される。データを復号するには、ウォレットが開いている（MEK がメモリーにロードされている）必要がある。一度開くと、Oracle は暗号化と復号を透過的に処理する。

---

## Oracle ウォレットのセットアップと管理

### ウォレットの作成

```sql
-- ステップ 1: sqlnet.ora (または 12c 以降の init.ora) でウォレットの場所を設定
-- ファイル場所: $ORACLE_BASE/admin/<db_name>/wallet/

-- sqlnet.ora の場合:
-- ENCRYPTION_WALLET_LOCATION =
--   (SOURCE = (METHOD = FILE)
--     (METHOD_DATA = (DIRECTORY = /opt/oracle/admin/ORCL/wallet)))

-- 19c 以降では、WALLET_ROOT 初期化パラメータの使用を推奨:
-- sqlnet.ora の ENCRYPTION_WALLET_LOCATION は Oracle 19c で非推奨。
ALTER SYSTEM SET wallet_root = '/opt/oracle/admin/ORCL/wallet' SCOPE = SPFILE;
-- 再起動が必要。WALLET_ROOT が設定されている場合、sqlnet.ora の 
-- ENCRYPTION_WALLET_LOCATION よりも優先される。WALLET_ROOT と併せて TDE_CONFIGURATION も使用する。

-- TDE 構成 (キー・ストア・タイプ) を設定
ALTER SYSTEM SET tde_configuration = 'KEYSTORE_CONFIGURATION=FILE' SCOPE = BOTH;
```

```sql
-- ステップ 2: ウォレットの作成 (SQL*Plus から、SYSDBA または ADMINISTER KEY MANAGEMENT 権限を持つユーザーで接続)
ADMINISTER KEY MANAGEMENT CREATE KEYSTORE '/opt/oracle/admin/ORCL/wallet'
  IDENTIFIED BY "W@lletP@ssw0rd!";
-- ewallet.p12 ファイルが作成される

-- ステップ 3: ウォレットを開く
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN
  IDENTIFIED BY "W@lletP@ssw0rd!"
  CONTAINER = ALL;  -- すべての PDB（CDB の場合）で開く。特定の PDB の場合は CURRENT を使用

-- ステップ 4: マスター暗号化キー (MEK) の作成（およびアクティブ化）
ADMINISTER KEY MANAGEMENT SET KEY
  IDENTIFIED BY "W@lletP@ssw0rd!"
  WITH BACKUP USING 'pre_tde_backup'
  CONTAINER = ALL;
```

### 自動ログイン・ウォレット (無人再起動用)

パスワード保護されたウォレットは、データベースの再起動ごとに手動で開く必要がある。自動ログイン・ウォレットを使用すると、自動的に開くようになる。

```sql
-- 既存のパスワード保護されたウォレットから自動ログイン・ウォレットを作成
ADMINISTER KEY MANAGEMENT CREATE AUTO_LOGIN KEYSTORE
  FROM KEYSTORE '/opt/oracle/admin/ORCL/wallet'
  IDENTIFIED BY "W@lletP@ssw0rd!";
-- cwallet.sso（自動ログイン・ファイル）が作成される

-- ウォレットのステータス確認
SELECT wrl_type, wrl_parameter, status, wallet_type, keystore_mode, con_id
FROM v$encryption_wallet;
-- STATUS が 'OPEN'、WALLET_TYPE が 'AUTOLOGIN' になっていること

-- ローカル自動ログイン・ウォレットの場合（別のサーバーでは使用不可）:
ADMINISTER KEY MANAGEMENT CREATE LOCAL AUTO_LOGIN KEYSTORE
  FROM KEYSTORE '/opt/oracle/admin/ORCL/wallet'
  IDENTIFIED BY "W@lletP@ssw0rd!";
```

### ウォレット管理コマンド

```sql
-- ウォレットを開く（自動ログインでない場合、手動再起動後に必要）
ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN
  IDENTIFIED BY "W@lletP@ssw0rd!";

-- ウォレットを閉じる（メモリー内のデータが暗号化され、すべての TDE 操作が停止する）
ADMINISTER KEY MANAGEMENT SET KEYSTORE CLOSE
  IDENTIFIED BY "W@lletP@ssw0rd!";

-- ウォレットのバックアップ
ADMINISTER KEY MANAGEMENT BACKUP KEYSTORE
  USING 'backup_tag_name'
  IDENTIFIED BY "W@lletP@ssw0rd!";

-- ウォレットとキーのステータス確認
SELECT key_id, creation_time, activation_time, key_use, keystore_type,
       origin, backed_up, con_id
FROM v$encryption_keys;

-- すべてのキー・ストア詳細を確認
SELECT * FROM v$encryption_wallet;
```

---

## 表領域の暗号化

表領域全体の暗号化は、最も一般的で推奨される TDE の導入方法である。暗号化された表領域に作成されたすべてのオブジェクト（表、索引、LOB、UNDO データ）は自動的に暗号化される。

### 暗号化表領域の作成

```sql
-- 新しい暗号化表領域を作成 (AES256 を推奨アルゴリズムとする)
CREATE TABLESPACE sensitive_data
  DATAFILE '/opt/oracle/oradata/ORCL/sensitive_data01.dbf' SIZE 1G AUTOEXTEND ON
  ENCRYPTION USING AES256
  DEFAULT STORAGE (ENCRYPT);

-- AES128 も利用可能。3DES168 もサポートされているが、新規導入には非推奨。

-- 暗号化の確認
SELECT tablespace_name, encrypted
FROM dba_tablespaces
WHERE encrypted = 'YES';

-- 表を暗号化表領域に移動
ALTER TABLE hr.employees MOVE TABLESPACE sensitive_data;

-- 表の移動後に索引を再構築（索引は自動的には移動しない）
ALTER INDEX hr.emp_emp_id_pk REBUILD TABLESPACE sensitive_data;
ALTER INDEX hr.emp_department_ix REBUILD TABLESPACE sensitive_data;

-- 表のすべての索引を暗号化表領域に移動するためのコマンド生成
SELECT 'ALTER INDEX ' || owner || '.' || index_name ||
       ' REBUILD TABLESPACE sensitive_data;' AS rebuild_cmd
FROM dba_indexes
WHERE table_owner = 'HR' AND table_name = 'EMPLOYEES';
```

### 既存の表領域の暗号化 (オンライン、12c 以降)

```sql
-- オフライン暗号化 (高速だがダウンタイムが必要)
ALTER TABLESPACE users OFFLINE;
ALTER TABLESPACE users ENCRYPTION OFFLINE ENCRYPT;
ALTER TABLESPACE users ONLINE;

-- オンライン暗号化 (ダウンタイムなし、デフォルトで AES256 を使用)
ALTER TABLESPACE users ENCRYPTION ONLINE ENCRYPT;

-- 特定のアルゴリズムを指定したオンライン暗号化
ALTER TABLESPACE users ENCRYPTION ONLINE USING AES256 ENCRYPT;

-- オンライン暗号化の進捗確認
SELECT tablespace_name, encryption_status
FROM dba_tablespace_encryption_progress;

-- 暗号化進行中の表領域の場合
SELECT * FROM v$encrypted_tablespaces;
```

---

## 列レベルの暗号化

列レベル TDE は、表領域全体ではなく、個々の列を暗号化する。より詳細な制御が可能だが、オーバーヘッドや制限がある（暗号化された列には標準的な索引を作成できず、バッファ・キャッシュ内でも暗号化された状態で保存され、処理時に復号される）。

```sql
-- 既存の列に暗号化を追加
ALTER TABLE hr.employees
  MODIFY (ssn ENCRYPT USING 'AES256' NO SALT);
-- SALT は、頻度分析を防ぐためにランダム・データを追加する
-- 列が WHERE 句の等価結合で使用される場合は NO SALT が必要
-- (SALT を使用すると値が毎回異なるため、等価比較が失敗する)

-- SALT を使用した列の暗号化 (より安全だが、= でクエリできない)
ALTER TABLE patients.records
  MODIFY (credit_card_number ENCRYPT USING 'AES256');
-- SALT がデフォルト。ENCRYPT USING 'AES256' SALT と同等。

-- 暗号化列を持つ新しい表を作成
CREATE TABLE payroll.salary_data (
  employee_id   NUMBER(6)        NOT NULL,
  salary        NUMBER(8,2)      ENCRYPT USING 'AES256' NO SALT,
  bonus         NUMBER(8,2)      ENCRYPT USING 'AES256' NO SALT,
  bank_account  VARCHAR2(30)     ENCRYPT USING 'AES256',
  CONSTRAINT sal_emp_fk FOREIGN KEY (employee_id) REFERENCES hr.employees(employee_id)
);

-- 列の暗号化を解除
ALTER TABLE hr.employees MODIFY (ssn DECRYPT);

-- 暗号化されている列を確認
SELECT owner, table_name, column_name, encryption_alg, salt
FROM dba_encrypted_columns
ORDER BY owner, table_name, column_name;
```

### サポートされている暗号化アルゴリズム

| アルゴリズム | キー長 | 備考 |
|---|---|---|
| `AES128` | 128-bit | 許容範囲。NIST 承認済み。 |
| `AES192` | 192-bit | 良い選択肢。 |
| `AES256` | 256-bit | 推奨。FIPS 140-2 準拠。23ai 以降のデフォルト。 |
| `3DES168` | 168-bit | レガシー。新規導入には非推奨。 |
| `ARIA128` | 128-bit | 韓国標準。韓国の規制対応用。 |
| `ARIA192` | 192-bit | 韓国標準。 |
| `ARIA256` | 256-bit | 韓国標準。 |
| `GOST256` | 256-bit | ロシア標準 — **Oracle 23c で非推奨、Oracle 26ai でサポート廃止・削除**。新規導入には使用しないこと。 |

> **ARIA および GOST に関する注記:** ARIA は Oracle 19c でオフライン表領域暗号化用に追加された。GOST も 19c で追加されたが、23c で非推奨となり 26ai で完全に削除された。新規導入にはすべて AES256 を使用すること。

---

## キーのローテーション

定期的なキー・ローテーションはセキュリティのベスト・プラクティスであり、多くのフレームワークで遵守が求められる要件である。Oracle TDE は、データベースをオフラインにすることなくキーの更新をサポートしている。

### マスター暗号化キー (MEK) のローテーション

```sql
-- 新しいマスター暗号化キーを作成 (古いキーは、古いキーで暗号化された
-- データを復号するために保持される。Oracle は遷移を自動的に処理する)
ADMINISTER KEY MANAGEMENT SET KEY
  IDENTIFIED BY "W@lletP@ssw0rd!"
  WITH BACKUP USING 'pre_rotation_backup';

-- 新しいキーがアクティブであることを確認
SELECT key_id, creation_time, activation_time, key_use
FROM v$encryption_keys
ORDER BY creation_time DESC;
-- 最も新しくアクティブ化されたキーが現在の MEK である

-- ローテーション後、表領域の DEK を新しい MEK で再暗号化（オプションだが推奨）
-- これにより、稼働中のデータに対して古い MEK が不要になることが保証される
ALTER TABLESPACE sensitive_data ENCRYPTION REKEY;
-- これはオンライン操作である。表領域の DEK を新しい MEK で再暗号化する。
```

### CDB/PDB におけるキー・ローテーションのベスト・プラクティス

```sql
-- CDB 内の特定の PDB に対してキーをローテーション
ALTER SESSION SET CONTAINER = pdb_finance;

ADMINISTER KEY MANAGEMENT SET KEY
  IDENTIFIED BY "W@lletP@ssw0rd!"
  WITH BACKUP USING 'pdb_finance_rotation'
  CONTAINER = CURRENT;

-- すべての PDB を一度にローテーション
ADMINISTER KEY MANAGEMENT SET KEY
  IDENTIFIED BY "W@lletP@ssw0rd!"
  CONTAINER = ALL;
```

---

## アプリケーション層での暗号化のための DBMS_CRYPTO

アプリケーションが暗号化を直接制御する必要がある場合（例: 保存前に特定の値を暗号化する、外部システムに送信する必要があるデータを暗号化するなど）、`DBMS_CRYPTO` を使用する。

```sql
-- ランダムな暗号化キーを生成
DECLARE
  v_key RAW(32);  -- AES256 用の 256 ビット・キー
BEGIN
  v_key := DBMS_CRYPTO.RANDOMBYTES(32);
  -- このキーを安全に保管する（データベースではなく、キー管理システムなどに）
  DBMS_OUTPUT.PUT_LINE('Key: ' || RAWTOHEX(v_key));
END;
/

-- 値の暗号化
DECLARE
  v_data      RAW(2000);
  v_key       RAW(32)   := HEXTORAW('YOUR_KEY_HEX_HERE');  -- 32 バイト = 256 ビット
  v_iv        RAW(16)   := DBMS_CRYPTO.RANDOMBYTES(16);  -- 初期化ベクトル
  v_encrypted RAW(2000);
  v_plaintext VARCHAR2(200) := 'Sensitive data here';
BEGIN
  v_data      := UTL_RAW.CAST_TO_RAW(v_plaintext);
  v_encrypted := DBMS_CRYPTO.ENCRYPT(
    src => v_data,
    typ => DBMS_CRYPTO.ENCRYPT_AES256 + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
    key => v_key,
    iv  => v_iv
  );
  -- v_encrypted と v_iv を一緒に保存する（復号に両方が必要）
  DBMS_OUTPUT.PUT_LINE('Encrypted: ' || RAWTOHEX(v_encrypted));
END;
/

-- 値の復号
DECLARE
  v_key       RAW(32)   := HEXTORAW('YOUR_KEY_HEX_HERE');
  v_iv        RAW(16)   := HEXTORAW('YOUR_IV_HEX_HERE');
  v_encrypted RAW(2000) := HEXTORAW('YOUR_ENCRYPTED_HEX_HERE');
  v_decrypted RAW(2000);
  v_plaintext VARCHAR2(200);
BEGIN
  v_decrypted := DBMS_CRYPTO.DECRYPT(
    src => v_encrypted,
    typ => DBMS_CRYPTO.ENCRYPT_AES256 + DBMS_CRYPTO.CHAIN_CBC + DBMS_CRYPTO.PAD_PKCS5,
    key => v_key,
    iv  => v_iv
  );
  v_plaintext := UTL_RAW.CAST_TO_VARCHAR2(v_decrypted);
  DBMS_OUTPUT.PUT_LINE('Decrypted: ' || v_plaintext);
END;
/

-- 値のハッシュ化（一方向。パスワードや整合性チェック用）
DECLARE
  v_hash RAW(32);
BEGIN
  v_hash := DBMS_CRYPTO.HASH(
    src => UTL_RAW.CAST_TO_RAW('value_to_hash'),
    typ => DBMS_CRYPTO.HASH_SH256
  );
  DBMS_OUTPUT.PUT_LINE('SHA256: ' || RAWTOHEX(v_hash));
END;
/

-- 整合性検証のための MAC (Message Authentication Code)
DECLARE
  v_mac RAW(32);
  v_key RAW(32) := HEXTORAW('YOUR_KEY_HEX_HERE');
BEGIN
  v_mac := DBMS_CRYPTO.MAC(
    src => UTL_RAW.CAST_TO_RAW('data to authenticate'),
    typ => DBMS_CRYPTO.HMAC_SH256,
    key => v_key
  );
  DBMS_OUTPUT.PUT_LINE('HMAC-SHA256: ' || RAWTOHEX(v_mac));
END;
/
```

---

## 暗号化バックアップ

TDE で暗号化された表領域は、RMAN によって自動的に暗号化された状態でバックアップされる。TDE を使用していないデータベースや、追加のバックアップ暗号化が必要な場合:

```sql
-- バックアップ用の RMAN 暗号化を構成
RMAN> CONFIGURE ENCRYPTION FOR DATABASE ON;
RMAN> CONFIGURE ENCRYPTION ALGORITHM 'AES256';

-- TDE ウォレットを使用したバックアップ暗号化（透過的）
RMAN> BACKUP DATABASE;

-- パスフレーズを使用したバックアップ暗号化（ウォレットとは独立）
RMAN> SET ENCRYPTION ON IDENTIFIED BY "BackupP@ss!" ONLY;
RMAN> BACKUP DATABASE;

-- バックアップ暗号化の確認
RMAN> LIST BACKUP SUMMARY;
-- ENCRYPTED 列に 'YES' が表示される
```

---

## 監視と検証

```sql
-- すべての機密表領域が暗号化されているか検証
SELECT ts.tablespace_name,
       CASE WHEN ts.encrypted = 'YES' THEN 'ENCRYPTED' ELSE 'NOT ENCRYPTED' END AS status,
       ts.block_size,
       ROUND(SUM(df.bytes)/1024/1024/1024, 2) AS size_gb
FROM dba_tablespaces ts
JOIN dba_data_files df ON df.tablespace_name = ts.tablespace_name
WHERE ts.tablespace_name NOT IN ('SYSTEM', 'SYSAUX', 'TEMP', 'UNDOTBS1')
GROUP BY ts.tablespace_name, ts.encrypted, ts.block_size
ORDER BY ts.tablespace_name;

-- 暗号化されていない表領域にある表を確認
SELECT t.owner, t.table_name, t.tablespace_name
FROM dba_tables t
JOIN dba_tablespaces ts ON ts.tablespace_name = t.tablespace_name
WHERE ts.encrypted = 'NO'
  AND t.owner NOT IN ('SYS', 'SYSTEM', 'CTXSYS', 'MDSYS', 'ORDSYS', 'XDB')
ORDER BY t.owner, t.table_name;

-- TDE キー管理操作の監査
SELECT event_timestamp, dbusername, action_name, sql_text
FROM unified_audit_trail
WHERE action_name LIKE 'ADMINISTER KEY MANAGEMENT%'
ORDER BY event_timestamp DESC;
```

---

## ベスト・プラクティス

1.  **列だけでなく表領域を暗号化する**: 表領域全体の暗号化の方が管理が容易であり、列単位の暗号化よりもパフォーマンスのオーバーヘッドが少なく、索引、UNDO、一時データなども含むすべてのデータが保護される。

2.  **AES256 を使用する**: Oracle が TDE でサポートする最強のアルゴリズムであり、FIPS 140-2 にも準拠している。現代のハードウェアでは AES128 とのパフォーマンス差は無視できる。

3.  **サーバー自体が物理的に保護されている場合にのみ、本番環境でローカル自動ログイン・ウォレットを使用する**: 自動ログイン・ウォレット・ファイル (cwallet.sso) はパスワードなしで開く。このファイルがデータファイルと共に盗まれた場合、データが復号されてしまう可能性がある。

4.  **ウォレットはデータベースとは別にバックアップする**: ウォレットを失い、データファイルが暗号化されている場合、データは二度と取り出せない。ウォレットのバックアップは別の安全な場所に保管すること。

5.  **キー・ローテーションの前にウォレットのバックアップを有効にする**: キーをローテーションするときは、常に `WITH BACKUP` を使用すること。バックアップにより古いキーが保存されるため、必要に応じて古いデータを復号できる。

6.  **ウォレット操作を監査する**: すべての `ADMINISTER KEY MANAGEMENT` コマンドを監査対象とすべきである。権限のない DBA がキーをローテーションすると、データにアクセスできなくなる可能性がある。

7.  **暗号化後に復号をテストする**: 表領域で TDE を有効にした後、必ずテスト・クエリを実行してデータが読み取れることを確認する。ウォレットが期待通りのバージョンであることを確認する。

8.  **キー管理手順を文書化する**: 災害復旧シナリオでは、ウォレットをリストアして開く手順が時間的に非常に重要となる。文書化された手順書（ランブック）をテストし、安全に保管しておくこと。

---

## よくある間違いとその回避方法

### 間違い 1: ウォレットをデータファイルと同じ場所に保管する

```bash
# 不適切: データベース・ホーム内やデータファイルと同じ場所にウォレットがある
/opt/oracle/oradata/ORCL/wallet/

# 適切: データファイルとは別のファイルシステムにウォレットを置く
/secure/keystore/ORCL/wallet/
# または、最高レベルのセキュリティを確保するために HSM (ハードウェア・セキュリティ・モジュール) を使用する
```

### 間違い 2: キー・ローテーションの前にウォレットをバックアップしない

```sql
-- ローテーション時は必ず WITH BACKUP を使用する
ADMINISTER KEY MANAGEMENT SET KEY
  IDENTIFIED BY "W@lletP@ssw0rd!"
  WITH BACKUP USING 'before_rotation_2026_01';  -- バックアップに日付タグを付ける

-- バックアップが作成されたことを確認
SELECT key_id, backed_up, creation_time
FROM v$encryption_keys
ORDER BY creation_time DESC;
```

### 間違い 3: 索引が貼られた列に列レベルの暗号化を使用する

SALT を使用した列レベル TDE では、同じ値でも暗号化結果が毎回異なるため、標準的な索引を使用できなくなる。NO SALT を使用すると等価クエリは機能するが、頻度分析攻撃が可能になってしまう。

```sql
-- WHERE 句で使用される列の場合: NO SALT を使用するか、表領域の暗号化を優先する
ALTER TABLE employees MODIFY (emp_code ENCRYPT USING 'AES256' NO SALT);

-- 直接クエリされない列（例: 保存された銀行口座番号）の場合: SALT を使用しても問題ない
ALTER TABLE employees MODIFY (bank_account ENCRYPT USING 'AES256');  -- SALT がデフォルト
```

### 間違い 4: エクスポートが TDE をバイパスすることを忘れる

Data Pump のエクスポート (`expdp`) は、エクスポート前にデータを復号する。明示的に暗号化しない限り、エクスポートされたダンプ・ファイルは非暗号化状態になる。

```bash
# TDE で保護されたデータをエクスポートするときは常に ENCRYPTION を使用する
expdp system/password FULL=Y \
  DUMPFILE=full_backup.dmp \
  ENCRYPTION=ALL \
  ENCRYPTION_PASSWORD=DumpFileP@ss! \
  ENCRYPTION_ALGORITHM=AES256
```

---

## コンプライアンスの考慮事項

### PCI-DSS
-   要件 3.4: 強力な暗号技術を使用して、保存されている PAN（プライマリ・アカウント番号）を読み取り不能にする。
-   TDE は、保存された会員データに対するこの要件を直接満たす。
-   PCI-DSS では最低 AES-128 が必要。AES-256 が推奨される。
-   キー管理手順（誰がキーにアクセスできるか、どのくらいの頻度でローテーションするか）を文書化する必要がある。
-   要件 3.5: 会員データを保護するために使用されるキーを、開示や不正使用から保護する。

### HIPAA
-   45 CFR 164.312(a)(2)(iv): ePHI の暗号化および復号。
-   45 CFR 164.312(e)(2)(ii): 転送中の ePHI の暗号化。
-   TDE は、「保存時の暗号化」のアドレサブル仕様を満たす。
-   NIST ガイドラインでは、保存されているヘルスケア・データに対して AES-256 を推奨している。

### GDPR
-   第 32 条: 適切な技術的手段として、個人データの暗号化を実装する。
-   GDPR は特定のアルゴリズムを義務付けていないが、暗号化は適切な手段の代表例である。
-   暗号化されたデータが漏洩した場合でも、キーが侵害されていなければ、情報漏洩の通知が免除される場合がある。

### FIPS 140-2
-   米国連邦政府のシステムでは、FIPS 140-2 検証済みの暗号モジュールが必要である。
-   Oracle の AES256 実装は FIPS 140-2 検証済みである。
-   Oracle Network で FIPS モードを有効にする例:
    ```
    # sqlnet.ora の場合:
    SQLNET.FIPS_140 = TRUE
    ```

---

## Oracle バージョンに関する注意 (19c vs 26ai)

-   このファイルの基本的なガイダンスは、より新しい最小バージョンが明記されていない限り、Oracle Database 19c に有効である。
-   21c、23c、または 23ai と記された機能は、Oracle Database 26ai 対応機能として扱う。バージョンが混在する環境では、19c 互換の代替案を保持すること。
-   リリース・アップデートによってデフォルトや非推奨が異なる場合があるため、両方のバージョンをサポートする環境では、19c と 26ai の両方で構文とパッケージの動作をテストすること。

## ソース

-   [Oracle Database Advanced Security Guide 19c — Using Transparent Data Encryption](https://docs.oracle.com/en/database/oracle/oracle-database/19/asoag/using-transparent-data-encryption.html)
-   [Oracle Database Advanced Security Guide 19c — Changes in 19c (ARIA, GOST added)](https://docs.oracle.com/en/database/oracle/oracle-database/19/asoag/release-changes.html)
-   [Oracle Database Reference 19c — WALLET_ROOT Parameter](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/WALLET_ROOT.html)
-   [Oracle Database 19c Licensing Information User Manual](https://docs.oracle.com/en/database/oracle/oracle-database/19/dblic/Licensing-Information.html)
-   [Oracle PL/SQL Packages Reference 19c — DBMS_CRYPTO](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_CRYPTO.html)
-   [Oracle Database 19c New Features Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/newft/)
-   [Oracle Support Note: How to Convert From SQLNET.ENCRYPTION_WALLET_LOCATION to WALLET_ROOT (Doc ID 2642694.1)](https://support.oracle.com/knowledge/Oracle%20Database%20Products/2642694_1.html)
