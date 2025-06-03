# 受払管理システム DB論理設計書 改修版

## 改修概要
本文書は、井関物流受払管理システムのデータベース設計に対する改修要件を反映した設計書です。

### 改修要件
1. **単位フィールド追加**: T_SEIKYUテーブルにTANI（単位）フィールドを追加
2. **制約追加**: 新規フィールドの制約設定
3. **インデックス最適化**: 検索性能向上のためのインデックス追加
4. **データ移行**: 既存データの互換性確保

## データベース基本情報

### データベース管理システム
- **DBMS**: Microsoft SQL Server Express
- **バージョン**: 2019以降
- **文字コード**: UTF-8
- **照合順序**: Japanese_CI_AS

### 接続情報
- **サーバー名**: localhost\SQLEXPRESS
- **データベース名**: IbUkeharaiDB
- **認証方式**: Windows認証

## テーブル設計

### T_SEIKYU（請求テーブル）改修版

#### テーブル概要
請求に関する情報を管理するテーブルです。改修により単位フィールドが追加されます。

#### テーブル定義
```sql
CREATE TABLE T_SEIKYU (
    SEIKYU_NO           VARCHAR(20)     NOT NULL,           -- 請求番号
    TORIHIKISAKI_CD     VARCHAR(10)     NOT NULL,           -- 取引先コード
    TORIHIKISAKI_NAME   VARCHAR(100)    NOT NULL,           -- 取引先名
    SEIKYU_DATE         DATE            NOT NULL,           -- 請求日
    HINMEI              VARCHAR(100)    NOT NULL,           -- 品名
    TANKA               DECIMAL(10,2)   NOT NULL,           -- 単価
    SURYO               DECIMAL(10,2)   NOT NULL,           -- 数量
    TANI                VARCHAR(10)     NOT NULL,           -- 単位（新規追加）
    KINGAKU             DECIMAL(12,2)   NOT NULL,           -- 金額
    GOUKEI              DECIMAL(12,2)   NULL,               -- 合計
    SAKUSEI_DATE        DATETIME        NOT NULL DEFAULT GETDATE(), -- 作成日時
    KOSHIN_DATE         DATETIME        NOT NULL DEFAULT GETDATE(), -- 更新日時
    SAKUSEI_USER        VARCHAR(50)     NOT NULL,           -- 作成者
    KOSHIN_USER         VARCHAR(50)     NOT NULL,           -- 更新者
    
    CONSTRAINT PK_T_SEIKYU PRIMARY KEY (SEIKYU_NO),
    CONSTRAINT FK_T_SEIKYU_TORIHIKISAKI FOREIGN KEY (TORIHIKISAKI_CD) 
        REFERENCES M_TORIHIKISAKI(TORIHIKISAKI_CD),
    CONSTRAINT CK_T_SEIKYU_TANKA CHECK (TANKA > 0),
    CONSTRAINT CK_T_SEIKYU_SURYO CHECK (SURYO > 0),
    CONSTRAINT CK_T_SEIKYU_KINGAKU CHECK (KINGAKU >= 0),
    CONSTRAINT CK_T_SEIKYU_TANI CHECK (TANI IN ('個', '本', '台', '式', 'kg', 'm', '㎡', 'その他'))
);
```

#### フィールド詳細

| フィールド名 | データ型 | NULL | 制約 | 説明 | 改修 |
|-------------|----------|------|------|------|------|
| SEIKYU_NO | VARCHAR(20) | NOT NULL | PK | 請求番号 | - |
| TORIHIKISAKI_CD | VARCHAR(10) | NOT NULL | FK | 取引先コード | - |
| TORIHIKISAKI_NAME | VARCHAR(100) | NOT NULL | - | 取引先名 | - |
| SEIKYU_DATE | DATE | NOT NULL | - | 請求日 | - |
| HINMEI | VARCHAR(100) | NOT NULL | - | 品名 | - |
| TANKA | DECIMAL(10,2) | NOT NULL | >0 | 単価 | - |
| SURYO | DECIMAL(10,2) | NOT NULL | >0 | 数量 | - |
| **TANI** | **VARCHAR(10)** | **NOT NULL** | **CHECK** | **単位** | **新規** |
| KINGAKU | DECIMAL(12,2) | NOT NULL | ≥0 | 金額 | - |
| GOUKEI | DECIMAL(12,2) | NULL | - | 合計 | - |
| SAKUSEI_DATE | DATETIME | NOT NULL | DEFAULT | 作成日時 | - |
| KOSHIN_DATE | DATETIME | NOT NULL | DEFAULT | 更新日時 | - |
| SAKUSEI_USER | VARCHAR(50) | NOT NULL | - | 作成者 | - |
| KOSHIN_USER | VARCHAR(50) | NOT NULL | - | 更新者 | - |

#### 新規追加フィールド詳細

**TANI（単位）フィールド**
- **データ型**: VARCHAR(10)
- **NULL制約**: NOT NULL
- **CHECK制約**: ('個', '本', '台', '式', 'kg', 'm', '㎡', 'その他')
- **デフォルト値**: なし（必須入力）
- **用途**: 請求明細の数量に対する単位を格納

## インデックス設計

### 既存インデックス
```sql
-- 主キーインデックス（自動作成）
CREATE UNIQUE CLUSTERED INDEX PK_T_SEIKYU 
ON T_SEIKYU (SEIKYU_NO);

-- 取引先コードインデックス
CREATE NONCLUSTERED INDEX IX_T_SEIKYU_TORIHIKISAKI_CD 
ON T_SEIKYU (TORIHIKISAKI_CD);

-- 請求日インデックス
CREATE NONCLUSTERED INDEX IX_T_SEIKYU_SEIKYU_DATE 
ON T_SEIKYU (SEIKYU_DATE);
```

### 新規追加インデックス
```sql
-- 単位別検索用インデックス
CREATE NONCLUSTERED INDEX IX_T_SEIKYU_TANI 
ON T_SEIKYU (TANI);

-- 複合インデックス（請求日+取引先+単位）
CREATE NONCLUSTERED INDEX IX_T_SEIKYU_COMPOSITE 
ON T_SEIKYU (SEIKYU_DATE, TORIHIKISAKI_CD, TANI);
```

## データ移行仕様

### 移行対象
- **テーブル**: T_SEIKYU
- **対象レコード**: 全既存データ

### 移行手順

#### 1. バックアップ作成
```sql
-- データベース全体のバックアップ
BACKUP DATABASE IbUkeharaiDB 
TO DISK = 'C:\Backup\IbUkeharaiDB_Before_Migration.bak';

-- テーブル単体のバックアップ
SELECT * INTO T_SEIKYU_BACKUP_20250603 FROM T_SEIKYU;
```

#### 2. テーブル構造変更
```sql
-- 単位フィールドを一時的にNULL許可で追加
ALTER TABLE T_SEIKYU 
ADD TANI VARCHAR(10) NULL;

-- 既存データに対するデフォルト値設定
UPDATE T_SEIKYU 
SET TANI = 'その他' 
WHERE TANI IS NULL;

-- NOT NULL制約の追加
ALTER TABLE T_SEIKYU 
ALTER COLUMN TANI VARCHAR(10) NOT NULL;

-- CHECK制約の追加
ALTER TABLE T_SEIKYU 
ADD CONSTRAINT CK_T_SEIKYU_TANI 
CHECK (TANI IN ('個', '本', '台', '式', 'kg', 'm', '㎡', 'その他'));
```

#### 3. インデックス作成
```sql
-- 新規インデックスの作成
CREATE NONCLUSTERED INDEX IX_T_SEIKYU_TANI 
ON T_SEIKYU (TANI);

CREATE NONCLUSTERED INDEX IX_T_SEIKYU_COMPOSITE 
ON T_SEIKYU (SEIKYU_DATE, TORIHIKISAKI_CD, TANI);
```

#### 4. データ検証
```sql
-- 移行後データ件数確認
SELECT COUNT(*) AS TOTAL_COUNT FROM T_SEIKYU;
SELECT COUNT(*) AS NULL_TANI_COUNT FROM T_SEIKYU WHERE TANI IS NULL;
SELECT TANI, COUNT(*) AS COUNT FROM T_SEIKYU GROUP BY TANI;

-- データ整合性確認
SELECT * FROM T_SEIKYU 
WHERE KINGAKU != (TANKA * SURYO) 
   OR TANKA <= 0 
   OR SURYO <= 0;
```

## 関連テーブル

### M_TORIHIKISAKI（取引先マスター）
```sql
CREATE TABLE M_TORIHIKISAKI (
    TORIHIKISAKI_CD     VARCHAR(10)     NOT NULL,
    TORIHIKISAKI_NAME   VARCHAR(100)    NOT NULL,
    YUBIN_NO            VARCHAR(8)      NULL,
    JUSHO               VARCHAR(200)    NULL,
    TEL_NO              VARCHAR(15)     NULL,
    FAX_NO              VARCHAR(15)     NULL,
    TANTO_NAME          VARCHAR(50)     NULL,
    SAKUSEI_DATE        DATETIME        NOT NULL DEFAULT GETDATE(),
    KOSHIN_DATE         DATETIME        NOT NULL DEFAULT GETDATE(),
    
    CONSTRAINT PK_M_TORIHIKISAKI PRIMARY KEY (TORIHIKISAKI_CD)
);
```

## SQL文例

### 基本的な検索
```sql
-- 請求データの基本検索（単位項目含む）
SELECT 
    SEIKYU_NO,
    TORIHIKISAKI_NAME,
    SEIKYU_DATE,
    HINMEI,
    TANKA,
    SURYO,
    TANI,           -- 新規追加項目
    KINGAKU,
    GOUKEI
FROM T_SEIKYU
WHERE SEIKYU_DATE BETWEEN '2025-06-01' AND '2025-06-30'
ORDER BY SEIKYU_NO;
```

### 単位別集計
```sql
-- 単位別の売上集計
SELECT 
    TANI,
    COUNT(*) AS 件数,
    SUM(KINGAKU) AS 売上合計,
    AVG(KINGAKU) AS 平均単価
FROM T_SEIKYU
WHERE SEIKYU_DATE >= '2025-01-01'
GROUP BY TANI
ORDER BY 売上合計 DESC;
```

### 請求書出力用クエリ
```sql
-- 請求書出力用の詳細データ取得
SELECT 
    s.SEIKYU_NO,
    s.TORIHIKISAKI_CD,
    t.TORIHIKISAKI_NAME,
    t.JUSHO,
    s.SEIKYU_DATE,
    s.HINMEI,
    s.TANKA,
    s.SURYO,
    s.TANI,
    s.KINGAKU,
    s.GOUKEI
FROM T_SEIKYU s
INNER JOIN M_TORIHIKISAKI t ON s.TORIHIKISAKI_CD = t.TORIHIKISAKI_CD
WHERE s.SEIKYU_NO = ?
ORDER BY s.SEIKYU_NO;
```

## パフォーマンス考慮事項

### 検索性能
- 単位フィールドでの検索頻度を考慮したインデックス設計
- 複合インデックスによる複数条件検索の最適化
- 統計情報の定期更新

### 容量設計
- 単位フィールド追加による容量増加: 約10バイト/レコード
- インデックス追加による容量増加: 約15%
- 年間データ増加量を考慮した容量計画

## セキュリティ設計

### アクセス制御
```sql
-- 読み取り専用ユーザーの作成
CREATE USER [IbUkeharai_ReadOnly] WITHOUT LOGIN;
GRANT SELECT ON T_SEIKYU TO [IbUkeharai_ReadOnly];

-- 更新権限ユーザーの作成
CREATE USER [IbUkeharai_Update] WITHOUT LOGIN;
GRANT SELECT, INSERT, UPDATE ON T_SEIKYU TO [IbUkeharai_Update];
```

### 監査ログ
```sql
-- 更新履歴テーブル
CREATE TABLE T_SEIKYU_AUDIT (
    AUDIT_ID            BIGINT IDENTITY(1,1) NOT NULL,
    SEIKYU_NO           VARCHAR(20)     NOT NULL,
    OPERATION_TYPE      VARCHAR(10)     NOT NULL, -- INSERT/UPDATE/DELETE
    OLD_TANI            VARCHAR(10)     NULL,
    NEW_TANI            VARCHAR(10)     NULL,
    AUDIT_DATE          DATETIME        NOT NULL DEFAULT GETDATE(),
    AUDIT_USER          VARCHAR(50)     NOT NULL,
    
    CONSTRAINT PK_T_SEIKYU_AUDIT PRIMARY KEY (AUDIT_ID)
);
```

## バックアップ・リストア戦略

### 定期バックアップ
```sql
-- 日次フルバックアップ
BACKUP DATABASE IbUkeharaiDB 
TO DISK = 'C:\Backup\IbUkeharaiDB_Full_' + 
          CONVERT(VARCHAR, GETDATE(), 112) + '.bak'
WITH COMPRESSION, CHECKSUM;

-- 時間別差分バックアップ
BACKUP DATABASE IbUkeharaiDB 
TO DISK = 'C:\Backup\IbUkeharaiDB_Diff_' + 
          CONVERT(VARCHAR, GETDATE(), 112) + '_' +
          CONVERT(VARCHAR, GETDATE(), 108) + '.bak'
WITH DIFFERENTIAL, COMPRESSION, CHECKSUM;
```

## 改修影響範囲

### データベース
- T_SEIKYUテーブルのスキーマ変更
- 新規インデックスの追加
- 既存クエリの修正

### アプリケーション
- 請求マスター登録画面
- 請求書出力画面
- 各種帳票出力機能

### 帳票
- 請求書テンプレート
- 売上集計レポート

## テスト計画

### 単体テスト
1. テーブル作成・変更の確認
2. 制約条件の動作確認
3. インデックスの効果確認

### 結合テスト
1. アプリケーションとの連携確認
2. 既存データとの互換性確認
3. パフォーマンステスト

### 運用テスト
1. バックアップ・リストアの確認
2. 監査ログの動作確認
3. セキュリティ設定の確認

---

**改修日**: 2025年6月3日  
**改修者**: データベース設計チーム  
**承認者**: システムアーキテクト
