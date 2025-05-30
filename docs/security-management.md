# セキュリティ管理

Microsoft 365 環境のセキュリティ強化に関するガイドです。

## 概要

Microsoft 365 のセキュリティは多層防御の考え方に基づいて設計されています。適切な設定により、データ漏洩やサイバー攻撃から組織を保護できます。

## 前提条件

- Microsoft 365 管理者権限
- セキュリティ管理者権限（推奨）
- Microsoft Entra ID Premium P1 または P2 ライセンス（一部機能）

## 多要素認証（MFA）

### 📊 ライセンス別MFA機能

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| 基本MFA | ✅ | ✅ | ✅ |
| 条件付きアクセス | ❌ | ✅ | ✅ |
| リスクベース認証 | ❌ | ❌ | ✅ |
| カスタム認証方法 | ❌ | ✅ | ✅ |

### 条件付きアクセスによる MFA 設定

⚠️ **注意**: 条件付きアクセスはA3/A5ライセンスが必要です。

1. Microsoft Entra ID 管理センターにアクセス
2. **セキュリティ** > **条件付きアクセス** を選択
3. **新しいポリシー** を作成

**以下のPowerShellコードの処理内容:**

1. `Connect-MgGraph -Scopes "Policy.Read.All"` - Microsoft GraphにPolicyの読み取り権限で接続
2. `Get-MgPolicyConditionalAccessPolicy` - 現在設定されている条件付きアクセスポリシーを全て取得
3. `Select-Object DisplayName, State` - ポリシー名と有効/無効状態のみを表示して設定状況を確認

```powershell
# PowerShell で MFA 設定を確認
Connect-MgGraph -Scopes "Policy.Read.All"
Get-MgPolicyConditionalAccessPolicy | Select-Object DisplayName, State
```

### MFA 方法の設定

推奨する認証方法の優先順位：
1. Microsoft Authenticator アプリ
2. FIDO2 セキュリティキー
3. Windows Hello for Business
4. SMS（最後の手段）

## パスワードポリシー

### Microsoft Entra ID パスワード保護

**以下のPowerShellコードの処理内容:**

1. `Connect-MgGraph -Scopes "Policy.ReadWrite.Authentication"` - 認証ポリシーの読み書き権限でMicrosoft Graphに接続
2. `$passwordPolicy` - カスタム禁止パスワードリストの設定オブジェクトを作成
3. `CustomBannedPasswords` - 組織固有の禁止パスワード（会社名、一般的なパスワード等）を配列で定義
4. `EnableBannedPasswordCheckOnPremises` - オンプレミス環境でも禁止パスワードチェックを有効化
5. `BannedPasswordCheckOnPremisesMode` - 強制モードで禁止パスワードの使用を阻止

```powershell
# カスタム禁止パスワードリストの設定
Connect-MgGraph -Scopes "Policy.ReadWrite.Authentication"

$passwordPolicy = @{
    CustomBannedPasswords = @("Company123", "Password2024", "組織名123")
    EnableBannedPasswordCheckOnPremises = $true
    BannedPasswordCheckOnPremisesMode = "Enforced"
}

# 設定を更新（実際のコマンドは Microsoft Graph API を使用）
```

### パスワード無しサインイン

Windows Hello for Business や FIDO2 キーを使用したパスワードレス認証の設定：

1. **Microsoft Entra ID 管理センター** > **セキュリティ** > **認証方法**
2. 各認証方法を有効化
3. ユーザーグループを指定

## データ損失防止（DLP）

### DLP ポリシーの作成

```powershell
# Exchange Online DLP ポリシーの例
Connect-ExchangeOnline

$dlpPolicy = New-DlpPolicy -Name "個人情報保護ポリシー" -Template "Japan Personal Information Protection"

# ルールをカスタマイズ
Set-DlpPolicy -Identity "個人情報保護ポリシー" -Mode Enforce
```

### 機密ラベルの設定

Microsoft Purview で機密ラベルを設定：

1. Microsoft Purview コンプライアンス ポータルにアクセス
2. **情報保護** > **ラベル** を選択
3. 新しいラベルを作成

## デバイス管理

### Microsoft Intune によるデバイス管理

```powershell
# Intune デバイス情報の取得
Connect-MgGraph -Scopes "DeviceManagementManagedDevices.Read.All"
Get-MgDeviceManagementManagedDevice | Select-Object DeviceName, OperatingSystem, ComplianceState
```

### 条件付きアクセスでデバイス制御

マネージドデバイスからのアクセスのみを許可：

```json
{
  "conditions": {
    "deviceStates": {
      "includeStates": ["compliant", "domainJoined"]
    }
  }
}
```

## セキュリティベースライン

### Microsoft 365 セキュリティベースライン

推奨設定：

1. **レガシー認証の無効化**
   ```powershell
   # レガシー認証をブロック
   Connect-MgGraph -Scopes "Policy.ReadWrite.ConditionalAccess"
   
   $policy = @{
     DisplayName = "レガシー認証ブロック"
     State = "enabled"
     Conditions = @{
       ClientAppTypes = @("exchangeActiveSync", "other")
     }
     GrantControls = @{
       Operator = "OR"
       BuiltInControls = @("block")
     }
   }
   
   New-MgConditionalAccessPolicy -BodyParameter $policy
   ```

2. **管理者アカウントの保護**
   - 専用管理者アカウントの使用
   - 緊急アクセス用アカウントの設定
   - 管理者への MFA 強制

## セキュリティ監視

### Microsoft Defender for Cloud Apps

```powershell
# Cloud App Security ポリシーの確認
Connect-MgGraph -Scopes "Policy.Read.All"
# Cloud App Security の API を使用してポリシーを取得
```

### 監査ログの有効化

```powershell
# Exchange Online 監査ログの有効化
Connect-ExchangeOnline
Set-OrganizationConfig -AuditDisabled $false

# SharePoint Online 監査の有効化
Connect-SPOService -Url https://contoso-admin.sharepoint.com
Set-SPOTenant -EnableSharePointAuditLog $true
```

## インシデント対応

### セキュリティインシデント検出時の対応手順

1. **初期対応**
   - 影響範囲の特定
   - 関係者への連絡
   - 証拠の保全

2. **封じ込め**
   ```powershell
   # 侵害されたユーザーアカウントの無効化
   Update-MgUser -UserId "compromised@contoso.com" -AccountEnabled:$false
   
   # サインインセッションの無効化
   Revoke-MgUserSignInSession -UserId "compromised@contoso.com"
   ```

3. **調査**
   - 監査ログの分析
   - アクティビティの追跡

4. **復旧**
   - パスワードのリセット
   - MFA の再設定
   - 権限の見直し

## 定期的なセキュリティタスク

### 月次タスク

- [ ] アクセス権の棚卸し
- [ ] 異常なサインインアクティビティの確認
- [ ] DLP ポリシーの効果測定

### 四半期タスク

- [ ] 脆弱性評価の実施

## Microsoft Secure Score による安全性確認

### 📊 セキュリティスコアとは

Microsoft Secure Score は、組織のセキュリティ態勢を測定し、改善すべき領域を特定するツールです。0～1000 点で評価され、業界平均と比較することが可能です。

### 🎯 セキュリティスコアアクセス方法

#### 1. Web ブラウザでの確認

1. **Microsoft 365 Defender ポータル**（<https://security.microsoft.com>）にアクセス
2. **セキュリティ スコア** を選択
3. 現在のスコアと改善アクションを確認

#### 2. PowerShell での確認

```powershell
# Microsoft Graph モジュールの接続
Connect-MgGraph -Scopes "SecurityEvents.Read.All"

# 最新のセキュリティスコアを取得
$latestScore = Get-MgSecuritySecureScore -Top 1
Write-Host "現在のスコア: $($latestScore.CurrentScore) / $($latestScore.MaxScore)"
Write-Host "達成率: $([math]::Round(($latestScore.CurrentScore / $latestScore.MaxScore) * 100, 2))%"

# スコア履歴の取得（過去30日間）
$scoreHistory = Get-MgSecuritySecureScore -Top 30 | Sort-Object CreatedDateTime
$scoreHistory | Select-Object CreatedDateTime, CurrentScore, MaxScore | Format-Table
```

### 📈 詳細な確認方法とポイント

#### 現在のスコア分析

```powershell
# 詳細なスコア分析
$scoreDetails = Get-MgSecuritySecureScore -Top 1

# カテゴリ別スコアの表示
$scoreDetails.ControlScores | ForEach-Object {
    Write-Host "カテゴリ: $($_.ControlCategory)"
    Write-Host "スコア: $($_.Score) / $($_.Total)"
    Write-Host "達成率: $([math]::Round(($_.Score / $_.Total) * 100, 2))%"
    Write-Host "---"
}
```

#### 重要な確認項目

1. **全体スコア**
   - 現在のスコア / 最大可能スコア
   - 業界平均との比較（画面上で確認）
   - 同規模組織との比較
   - 過去30日間のトレンド

2. **カテゴリ別詳細分析**
   - **ID とアクセス管理** - 最優先領域
     - MFA の有効化状況
     - 条件付きアクセス ポリシー
     - 特権アカウントの保護
   - **デバイス セキュリティ** - Intune 関連
     - デバイス コンプライアンス
     - アプリ保護ポリシー
     - デバイス暗号化
   - **アプリケーション** - Office 365 設定
     - 安全な添付ファイル
     - 安全なリンク
     - フィッシング対策
   - **データ** - 情報保護設定
     - 機密ラベル
     - DLP ポリシー
     - 暗号化の設定

#### 3. 改善アクションの詳細確認

```powershell
# 改善アクションの取得
Connect-MgGraph -Scopes "SecurityActions.Read.All"

# 実装可能な改善アクション一覧
$actions = Get-MgSecuritySecureScoreControlProfile | 
    Where-Object {$_.ImplementationCost -eq "Low" -and $_.UserImpact -eq "Low"} |
    Select-Object Title, MaxScore, ImplementationCost, UserImpact, Threats

$actions | Sort-Object MaxScore -Descending | Format-Table
```

### 🔍 安全性確認の具体的手順

#### ダッシュボードでの日常確認（推奨：週1回）

1. **スコアトレンドの確認**
   - スコアが下がっていないか
   - 新しい改善アクションが追加されていないか
   - 実装済みアクションが正常に動作しているか

2. **改善アクションの優先順位付け**

   ```powershell
   # 高価値・低コストアクションの特定
   $highValueActions = Get-MgSecuritySecureScoreControlProfile | 
       Where-Object {
           $_.MaxScore -ge 5 -and 
           $_.ImplementationCost -eq "Low" -and 
           $_.UserImpact -in @("Low", "Moderate")
       } | Sort-Object MaxScore -Descending
   
   Write-Host "実装推奨アクション:"
   $highValueActions | Select-Object Title, MaxScore, ImplementationCost | Format-Table
   ```

3. **リスク評価の実施**
   - 組織の脅威レベルに応じた優先順位
   - コンプライアンス要件との整合性
   - ビジネス要件との兼ね合い

#### 月次の詳細分析

1. **スコア変動の要因分析**

   ```powershell
   # 過去30日間のスコア変動を分析
   $scoreHistory = Get-MgSecuritySecureScore -Top 30 | Sort-Object CreatedDateTime
   
   $scoreDiff = $scoreHistory[-1].CurrentScore - $scoreHistory[0].CurrentScore
   Write-Host "過去30日のスコア変動: $scoreDiff"
   
   # 改善された項目の確認
   if ($scoreDiff -gt 0) {
       Write-Host "改善が見られました。継続的な監視を続けてください。"
   } else {
       Write-Host "スコアが低下しています。新しい脅威や設定変更を確認してください。"
   }
   ```

2. **競合他社・業界比較**
   - 同業他社とのベンチマーク
   - 業界標準スコアとの比較
   - 目標値の設定・見直し

3. **コンプライアンス要件の確認**
   - GDPR、個人情報保護法の要件
   - 業界固有の規制要件
   - 監査準備の状況

### 📋 セキュリティスコア定期確認チェックリスト

#### 週次のスコア確認

- [ ] セキュリティスコアの確認（前週比較）
- [ ] 新しい改善アクションの確認
- [ ] 高優先度アクションの進捗確認
- [ ] セキュリティインシデントの有無確認

#### 月次のスコア分析

- [ ] カテゴリ別スコアの傾向分析
- [ ] 業界平均との比較レポート作成
- [ ] 完了したアクションの効果測定
- [ ] 新しいセキュリティ機能の評価
- [ ] ユーザー教育効果の確認

#### 四半期のスコア戦略見直し

- [ ] セキュリティ戦略の全体見直し
- [ ] 投資対効果（ROI）の分析
- [ ] コンプライアンス要件との整合性確認
- [ ] 緊急アクセス用アカウントの動作確認
- [ ] セキュリティポリシーの見直し

### 🚨 アラート設定と自動化

#### PowerShell による自動監視スクリプト

```powershell
# セキュリティスコア監視スクリプト
function Monitor-SecurityScore {
    param(
        [int]$ThresholdScore = 600,
        [string]$EmailTo = "security-team@company.com"
    )
    
    Connect-MgGraph -Scopes "SecurityEvents.Read.All"
    
    $currentScore = Get-MgSecuritySecureScore -Top 1
    $scorePercentage = ($currentScore.CurrentScore / $currentScore.MaxScore) * 100
    
    if ($currentScore.CurrentScore -lt $ThresholdScore) {
        # アラート送信ロジック
        $alertMessage = @"
セキュリティスコアが基準値を下回りました。
現在のスコア: $($currentScore.CurrentScore) / $($currentScore.MaxScore)
達成率: $([math]::Round($scorePercentage, 2))%
確認日時: $(Get-Date)

至急、セキュリティ設定の確認を行ってください。
"@
        
        Write-Warning $alertMessage
        # ここにメール送信やTeams通知のロジックを追加
    }
    
    return @{
        CurrentScore = $currentScore.CurrentScore
        MaxScore = $currentScore.MaxScore
        Percentage = $scorePercentage
        Status = if ($currentScore.CurrentScore -ge $ThresholdScore) { "正常" } else { "要注意" }
    }
}

# 実行例
$result = Monitor-SecurityScore -ThresholdScore 650
Write-Host "セキュリティスコア監視結果: $($result.Status)"
```

### 🎯 改善アクションの実装戦略

#### フェーズ1: 基本セキュリティ（目標スコア: 600点）

1. **管理者アカウントの保護**
   - 全管理者への MFA 強制
   - 専用管理者アカウントの作成
   - 緊急アクセス用アカウントの設定

2. **基本的なアクセス制御**
   - セキュリティの既定値の有効化
   - レガシー認証のブロック
   - 基本的な条件付きアクセス

#### フェーズ2: 強化セキュリティ（目標スコア: 750点）

1. **高度なアクセス制御**
   - 場所ベースの条件付きアクセス
   - デバイス準拠性の要求
   - リスクベース認証

2. **データ保護**
   - 基本的な DLP ポリシー
   - 機密ラベルの適用
   - 外部共有の制限

#### フェーズ3: 包括的セキュリティ（目標スコア: 850点以上）

1. **ゼロトラスト実装**
   - 包括的な条件付きアクセス
   - エンドポイント保護
   - 高度な脅威保護

2. **継続的監視**
   - 自動化された監視
   - インシデント対応の自動化
   - 予防的セキュリティ措置

### 🎯 改善アクションの優先順位

#### 高優先度（即座に実装推奨）

1. **管理者アカウントの MFA 有効化**
   ```powershell
   # 管理者の MFA 状況確認
   Connect-MgGraph -Scopes "UserAuthenticationMethod.Read.All"
   Get-MgUser -Filter "assignedLicenses/any(x:x/skuId eq guid'subscription-id')" | 
   Get-MgUserAuthenticationMethod
   ```

2. **セキュリティの規定値の有効化**
   - Microsoft Entra ID で「セキュリティの規定値」を有効化
   - すべてのユーザーに基本的な MFA を適用

3. **管理者ロールの確認**
   ```powershell
   # 特権ロール割り当ての確認
   Get-MgDirectoryRoleAssignment | Where-Object {$_.PrincipalType -eq "User"}
   ```

#### 中優先度（段階的実装）

1. **条件付きアクセス ポリシー**
   - 場所ベースのアクセス制御
   - デバイス準拠性の要求
   - アプリケーション制御

2. **データ損失防止 (DLP)**
   - 機密データの自動検出
   - 外部共有の制限
   - 暗号化の強制

#### 低優先度（長期計画）

1. **高度な脅威保護**
   - Microsoft Defender for Office 365
   - 高度な分析機能
   - 自動応答の設定

### 📋 定期確認チェックリスト

#### 週次確認
- [ ] 新しい改善アクションの確認
- [ ] スコアの変動要因分析
- [ ] 高優先度アクションの進捗確認

#### 月次確認
- [ ] カテゴリ別スコアの傾向分析
- [ ] 業界平均との比較
- [ ] 完了したアクションの効果測定
- [ ] 新しいセキュリティ機能の評価

#### 四半期確認
- [ ] セキュリティ戦略の見直し
- [ ] 投資対効果の分析
- [ ] コンプライアンス要件との整合性確認

### 🔧 改善アクションの実装例

#### 1. サインイン リスク ポリシー
```powershell
# リスクベース条件付きアクセス ポリシーの作成
$riskPolicy = @{
    displayName = "高リスク サインイン ブロック"
    state = "enabled"
    conditions = @{
        signInRiskLevels = @("high")
        applications = @{
            includeApplications = @("All")
        }
        users = @{
            includeUsers = @("All")
            excludeUsers = @("緊急アクセス用アカウントID")
        }
    }
    grantControls = @{
        operator = "OR"
        builtInControls = @("block")
    }
}
```

#### 2. デバイス コンプライアンス ポリシー
```powershell
# Intune デバイス コンプライアンス確認
Connect-MgGraph -Scopes "DeviceManagementConfiguration.Read.All"
Get-MgDeviceManagementDeviceCompliancePolicy | 
Select-Object DisplayName, CreatedDateTime, LastModifiedDateTime
```

### 📊 レポートとダッシュボード

#### PowerShell によるカスタム レポート
```powershell
# セキュリティスコア履歴の取得
$scoreHistory = Get-MgSecuritySecureScore -Top 30 | Sort-Object CreatedDateTime

# スコア推移のグラフデータ作成
$scoreHistory | Select-Object @{
    Name="Date"; Expression={$_.CreatedDateTime.ToString("yyyy-MM-dd")}
}, CurrentScore, MaxScore | 
Export-Csv -Path "SecurityScoreHistory.csv" -NoTypeInformation
```

### ⚠️ 実装時の注意事項

1. **段階的な実装**
   - 一度に多くの変更を行わない
   - ユーザーへの影響を事前評価
   - ロールバック計画の準備

2. **テスト環境での検証**
   - パイロット グループでの事前テスト
   - 業務への影響の確認
   - ユーザー フィードバックの収集

3. **ドキュメント化**
   - 変更内容の記録
   - 設定理由の明文化
   - 運用手順の更新

### 目標設定の例

| 期間 | 目標スコア | 重点領域 |
|------|------------|----------|
| 1ヶ月 | 600+ | 基本的な MFA とアクセス制御 |
| 3ヶ月 | 750+ | 条件付きアクセスと DLP |
| 6ヶ月 | 850+ | 高度な脅威保護 |
| 1年 | 900+ | ゼロトラスト アーキテクチャ完成 |

## ベストプラクティス

1. **ゼロトラストの原則**
   - 信頼せず、常に検証
   - 最小権限の原則
   - セキュリティ違反を前提とした設計

2. **定期的な見直し**
   - 設定の定期監査
   - ポリシーの効果測定
   - 新しい脅威への対応

3. **従業員教育**
   - フィッシング攻撃への対策
   - パスワード管理の重要性
   - インシデント報告の仕組み

## 注意事項

- セキュリティ設定変更前は必ずテスト環境で検証
- ユーザビリティとセキュリティのバランスを考慮
- 規制要件（GDPR、個人情報保護法等）への準拠

## 関連情報

- [Microsoft 365 セキュリティドキュメント](https://docs.microsoft.com/ja-jp/microsoft-365/security/)
- [Microsoft Entra ID セキュリティ](https://docs.microsoft.com/ja-jp/entra/fundamentals/concept-fundamentals-security-defaults)
- [Intune 端末管理](intune-device-management.md)
- [Microsoft Defender for Office 365](docs/defender-management.md)
