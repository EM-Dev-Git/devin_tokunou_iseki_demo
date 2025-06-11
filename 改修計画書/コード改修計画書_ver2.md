# コード改修計画書 ver2.0

## 概要

### 目的
単体テスト実施報告書で特定された失敗テストケース（TC007、TC008）の解決に向けた包括的なコード改修計画を策定する。特に、OutputSeikyuReportS.rdlcにおけるTANI（単位）フィールドの独立表示実装に焦点を当てる。

### 対象システム
- **システム名**: 井関物流 受払管理システム
- **対象モジュール**: 請求書出力機能（OutputSeikyuReportS.rdlc）
- **改修範囲**: RDLC レポート構造の TANI フィールド実装

### 改修要件
- TC007: RDLC TANI列参照テストの合格
- TC008: RDLC統合テストの合格
- TANI フィールドの独立表示機能実装

## 問題分析

### 単体テスト実施報告書からの問題特定

#### TC007: RDLC TANI列参照テスト
**現象**: OutputSeikyuReportS.rdlc内でTANI フィールドの独立参照が確認できない  
**テスト結果**: ❌ **FAIL**  
**根本原因**: RDLC構造内でTANI列の表示設定が未実装

#### TC008: RDLC統合テスト
**現象**: SURYO、TANI、KINGAKU フィールドの統合動作確認失敗  
**テスト結果**: ❌ **FAIL**  
**根本原因**: TANI フィールドの独立参照不備により統合テストも失敗

### 技術的根本原因分析

#### 現在のRDLC構造問題
1. **列定義**: 7列の構造で列幅は正しく設定済み
   - 列1: 0.69591cm
   - 列2: 6.12963cm  
   - 列3: 2.3175cm
   - 列4: 1.68974cm
   - 列5: 3.0cm (SURYO用)
   - 列6: 0.8cm (TANI用 - **テキストボックス未実装**)
   - 列7: 4.16001cm (KINGAKU用)

2. **テキストボックス実装状況**
   - ✅ SURYO: 実装済み（行536-570）
   - ❌ TANI: **未実装** - これが問題の根本原因
   - ✅ TANKA: 実装済み（行575-608）
   - ✅ KINGAKU: 実装済み（行613-647）

3. **データフィールド参照**
   - SURYO: `=Format(Fields!SURYO.Value,"#,###")` - 実装済み
   - TANI: `=Fields!TANI.Value` - **参照なし**
   - KINGAKU: `=Format(Fields!KINGAKU.Value,"#,###")` - 実装済み

## 技術的解決方案

### 1. RDLC構造修正

#### 1.1 TANIテキストボックスの挿入位置
**挿入場所**: OutputSeikyuReportS.rdlc 行572と行573の間  
**対象**: TablixCell構造内にTANIテキストボックスを追加

#### 1.2 実装するXML構造
```xml
<TablixCell>
  <CellContents>
    <Textbox Name="TANI">
      <KeepTogether>true</KeepTogether>
      <Paragraphs>
        <Paragraph>
          <TextRuns>
            <TextRun>
              <Value>=Fields!TANI.Value</Value>
              <Style>
                <FontFamily>ＭＳ ゴシック</FontFamily>
              </Style>
            </TextRun>
          </TextRuns>
          <Style>
            <TextAlign>Center</TextAlign>
          </Style>
        </Paragraph>
      </Paragraphs>
      <rd:DefaultName>TANI</rd:DefaultName>
      <Style>
        <Border>
          <Style>Solid</Style>
          <Width>0.5pt</Width>
        </Border>
        <TopBorder>
          <Style>None</Style>
        </TopBorder>
        <LeftBorder>
          <Style>None</Style>
        </LeftBorder>
        <VerticalAlign>Middle</VerticalAlign>
        <PaddingLeft>2pt</PaddingLeft>
        <PaddingRight>2pt</PaddingRight>
      </Style>
    </Textbox>
  </CellContents>
</TablixCell>
```

#### 1.3 ヘッダー行の修正
**挿入場所**: ヘッダー行（約386行目付近）  
**内容**: "単位" ヘッダーテキストボックスの追加

```xml
<TablixCell>
  <CellContents>
    <Textbox Name="Textbox_TANI_Header">
      <KeepTogether>true</KeepTogether>
      <Paragraphs>
        <Paragraph>
          <TextRuns>
            <TextRun>
              <Value>単位</Value>
              <Style>
                <FontFamily>ＭＳ ゴシック</FontFamily>
                <FontWeight>Bold</FontWeight>
              </Style>
            </TextRun>
          </TextRuns>
          <Style>
            <TextAlign>Center</TextAlign>
          </Style>
        </Paragraph>
      </Paragraphs>
      <Style>
        <Border>
          <Style>Solid</Style>
          <Width>0.5pt</Width>
        </Border>
        <VerticalAlign>Middle</VerticalAlign>
        <PaddingLeft>2pt</PaddingLeft>
        <PaddingRight>2pt</PaddingRight>
      </Style>
    </Textbox>
  </CellContents>
</TablixCell>
```

### 2. データソース検証

#### 2.1 T_SEIKYUテーブルのTANIフィールド確認
- TANIフィールドがデータセットに含まれていることを確認
- TANI_OPTIONS配列との整合性確認: ["個", "本", "台", "式", "kg", "m", "㎡", "その他"]

#### 2.2 データバインディング検証
- DataSet1におけるTANIフィールドのマッピング確認
- フィールドタイプ: String型での適切な設定

## 実装手順

### Phase 1: RDLC構造修正（即座実行）

#### ステップ1: TANIテキストボックス挿入
1. OutputSeikyuReportS.rdlcを開く
2. 行572（SURYO TablixCell終了）と行573（TANKA TablixCell開始）の間に挿入
3. 上記XML構造を正確に挿入
4. インデントと構文の整合性確認

#### ステップ2: ヘッダー行修正
1. ヘッダー行のTablixCell構造を特定
2. "単位"ヘッダーテキストボックスを適切な位置に挿入
3. 既存ヘッダーとの整合性確認

#### ステップ3: XML構文検証
1. RDLC XMLの構文エラーチェック
2. 名前空間とスキーマの整合性確認
3. Visual Studio等でのRDLCファイル検証

### Phase 2: データソース検証（必要に応じて）

#### ステップ4: データセット確認
1. DataSet1のフィールド定義確認
2. TANIフィールドの存在とタイプ確認
3. 必要に応じてデータソース設定更新

### Phase 3: 統合テスト

#### ステップ5: 単体テスト再実行
1. TC007: RDLC TANI列参照テスト実行
2. TC008: RDLC統合テスト実行
3. `Fields!TANI.Value`参照の検出確認

#### ステップ6: レポート生成テスト
1. サンプルデータでのレポート生成
2. TANI値の正しい表示確認
3. 列レイアウトと配置の検証

## 検証基準

### 成功基準

#### TC007合格基準
- OutputSeikyuReportS.rdlc内で`Fields!TANI.Value`参照が検出される
- TANIテキストボックスが適切に実装されている
- 単体テスト結果: PASS

#### TC008合格基準
- SURYO、TANI、KINGAKU フィールドの統合動作が正常
- レポート構造の整合性確認
- 単体テスト結果: PASS

#### 視覚的検証基準
- TANI列が0.8cm幅で正しく表示される
- 単位値（個、本、台等）が中央揃えで表示される
- 既存列（SURYO、TANKA、KINGAKU）との整合性維持

### 品質基準

#### コード品質
- XML構文の正確性
- 既存パターンとの一貫性
- 適切なスタイル設定

#### パフォーマンス
- レポート生成時間への影響最小化
- メモリ使用量の適正化

## リスク分析と対策

### 技術的リスク

#### リスク1: RDLC構文エラー
**影響度**: 高  
**対策**: 段階的実装とXML検証ツール使用

#### リスク2: データバインディング問題
**影響度**: 中  
**対策**: データソース事前検証とテストデータ準備

#### リスク3: 既存機能への影響
**影響度**: 低  
**対策**: 既存テストケースの再実行による回帰テスト

### 運用リスク

#### リスク4: レポート表示崩れ
**影響度**: 中  
**対策**: 複数環境での表示確認とユーザー受入テスト

## 実装タイムライン

### 即座実行（Phase 1）
- **所要時間**: 30分
- **作業内容**: RDLC構造修正
- **成果物**: 修正されたOutputSeikyuReportS.rdlc

### 検証フェーズ（Phase 2-3）
- **所要時間**: 60分
- **作業内容**: データソース検証と統合テスト
- **成果物**: テスト結果レポート

### 総所要時間
- **合計**: 90分
- **クリティカルパス**: RDLC構造修正

## 依存関係

### 技術的依存関係
- **なし**: 既存インフラストラクチャを使用
- **前提条件**: T_SEIKYUテーブルのTANIフィールド存在

### 外部依存関係
- **なし**: 外部システムとの連携不要

## 成果物

### 主要成果物
1. **修正されたOutputSeikyuReportS.rdlc**
   - TANIテキストボックス実装済み
   - ヘッダー行修正済み
   - XML構文検証済み

2. **テスト結果レポート**
   - TC007、TC008の合格確認
   - 統合テスト結果
   - 品質検証結果

### 関連文書
- 単体テスト実施報告書（更新版）
- 改修トレーサビリティマトリックス

## 品質保証

### テスト戦略
1. **単体テスト**: TC007、TC008の再実行
2. **統合テスト**: レポート生成機能全体の動作確認
3. **回帰テスト**: 既存機能への影響確認

### 検証手順
1. **構文検証**: XML構文とRDLCスキーマ適合性
2. **機能検証**: TANI値の正しい表示確認
3. **性能検証**: レポート生成時間の測定

## 承認とレビュー

### 技術レビュー
- **対象**: RDLC構造修正内容
- **基準**: 既存パターンとの一貫性、XML構文正確性

### 機能レビュー
- **対象**: TANI表示機能
- **基準**: 要件適合性、ユーザビリティ

## 実装後の監視

### 監視項目
1. **機能監視**: TANI値の正しい表示
2. **性能監視**: レポート生成時間
3. **エラー監視**: RDLC関連エラーの発生状況

### 対応手順
- 問題発生時の即座対応体制
- ロールバック手順の準備

---

**計画書作成日**: 2025年6月11日  
**作成者**: Devin AI  
**対象ブランチ**: main-tokunou-20250603-001  
**承認**: 要承認  
**実装予定**: 即座実行可能
