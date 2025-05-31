# Microsoft 365 ライセンス管理とライセンス別機能

Microsoft 365 の各ライセンス（A1、A3、A5）で利用できる機能の詳細と管理方法について説明します。

## 概要

Microsoft 365 Education には主に3つのライセンスプランがあり、それぞれ異なる機能セットを提供しています。適切なライセンス管理により、コストを最適化しながら必要な機能を提供できます。

## ライセンス比較表

### 基本機能

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| **Exchange Online** | ✅ 2GB | ✅ 100GB | ✅ 100GB |
| **SharePoint Online** | ✅ 1TB | ✅ 1TB | ✅ 1TB |
| **OneDrive for Business** | ✅ 最大100GB | ✅ 最大100GB | ✅ 最大100GB |
| **Microsoft Teams** | ✅ 基本 | ✅ 拡張 | ✅ 拡張 |
| **Office アプリ（Web版）** | ✅ | ✅ | ✅ |
| **Office アプリ（デスクトップ）** | ❌ | ✅ | ✅ |

### セキュリティ・コンプライアンス機能

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| **Microsoft Entra ID Premium P1** | ❌ | ✅ | ✅ |
| **Microsoft Entra ID Premium P2** | ❌ | ❌ | ✅ |
| **Microsoft Intune** | ❌ | ✅ | ✅ |
| **条件付きアクセス** | ❌ | ✅ | ✅ |
| **多要素認証** | ✅ 基本 | ✅ 拡張 | ✅ 拡張 |
| **Identity Protection** | ❌ | ❌ | ✅ |
| **Privileged Identity Management** | ❌ | ❌ | ✅ |

### Microsoft Defender 機能

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| **Defender for Office 365 Plan 1** | ❌ | ✅ | ✅ |
| **Defender for Office 365 Plan 2** | ❌ | ❌ | ✅ |
| **Safe Attachments** | ❌ | ✅ | ✅ |
| **Safe Links** | ❌ | ✅ | ✅ |
| **Advanced Threat Protection** | ❌ | ❌ | ✅ |
| **Threat Intelligence** | ❌ | ❌ | ✅ |

### 分析・レポート機能

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| **Microsoft Purview** | ✅ 基本 | ✅ 標準 | ✅ プレミアム |
| **Power BI Pro** | ❌ | ❌ | ✅ |
| **MyAnalytics** | ❌ | ✅ | ✅ |
| **Workplace Analytics** | ❌ | ❌ | ✅ |
| **Advanced eDiscovery** | ❌ | ❌ | ✅ |

## PowerShell によるライセンス管理

### ライセンス情報の確認

**以下のPowerShellコードの処理内容:**

1. `Connect-MgGraph -Scopes` - 組織情報とユーザー管理権限でMicrosoft Graphに接続
2. `Get-MgSubscribedSku` - テナント内で利用可能な全ライセンスSKU情報を取得
3. `ConsumedUnits` - 現在使用中のライセンス数を表示
4. `$_.PrepaidUnits.Enabled - $_.ConsumedUnits` - 利用可能な残りライセンス数を計算
5. `Where-Object {$_.SkuPartNumber -like "*EDUCATION*"}` - 教育機関向けライセンスのみをフィルタリング

```powershell
# Microsoft Graph に接続
Connect-MgGraph -Scopes "Organization.Read.All", "User.ReadWrite.All"

# 利用可能なライセンスの確認
Get-MgSubscribedSku | Select-Object SkuPartNumber, ConsumedUnits, @{Name="Available";Expression={$_.PrepaidUnits.Enabled - $_.ConsumedUnits}}

# A1、A3、A5 ライセンスの詳細表示
$educationSkus = Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -like "*EDUCATION*"}
$educationSkus | Format-Table SkuPartNumber, ConsumedUnits, @{Name="Available";Expression={$_.PrepaidUnits.Enabled - $_.ConsumedUnits}}
```

### ユーザーライセンスの確認

**以下のPowerShellコードの処理内容:**

1. `Get-MgUser -UserId -Property AssignedLicenses` - 指定ユーザーの割り当てられたライセンス情報を取得
2. `$user.AssignedLicenses` - ユーザーに割り当てられたライセンスの配列をループ処理
3. `Get-MgSubscribedSku -SubscribedSkuId $_.SkuId` - ライセンスIDから具体的なSKU情報を取得
4. `$sku.SkuPartNumber` - ライセンスの識別名（A1_FACULTY、A3_FACULTY等）を表示

```powershell
# 特定ユーザーのライセンス確認
$user = Get-MgUser -UserId "user@contoso.edu" -Property AssignedLicenses
$user.AssignedLicenses | ForEach-Object {
    $sku = Get-MgSubscribedSku -SubscribedSkuId $_.SkuId
    Write-Host "ライセンス: $($sku.SkuPartNumber)"
}

# ライセンス別ユーザー数の確認
$allUsers = Get-MgUser -All -Property AssignedLicenses
$licenseStats = @{}

foreach ($user in $allUsers) {
    foreach ($license in $user.AssignedLicenses) {
        $sku = Get-MgSubscribedSku -SubscribedSkuId $license.SkuId
        if ($licenseStats.ContainsKey($sku.SkuPartNumber)) {
            $licenseStats[$sku.SkuPartNumber]++
        } else {
            $licenseStats[$sku.SkuPartNumber] = 1
        }
    }
}

$licenseStats
```

### ライセンスの割り当て

```powershell
# A1 ライセンスの割り当て
$a1SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "STANDARDWOFFPACK_FACULTY"}).SkuId
$license = @{
    AddLicenses = @(
        @{
            SkuId = $a1SkuId
        }
    )
    RemoveLicenses = @()
}
Set-MgUserLicense -UserId "faculty@contoso.edu" -BodyParameter $license

# A3 ライセンスの割り当て
$a3SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPACK_FACULTY"}).SkuId
$license = @{
    AddLicenses = @(
        @{
            SkuId = $a3SkuId
        }
    )
    RemoveLicenses = @()
}
Set-MgUserLicense -UserId "principal@contoso.edu" -BodyParameter $license

# A5 ライセンスの割り当て
$a5SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPREMIUM_FACULTY"}).SkuId
$license = @{
    AddLicenses = @(
        @{
            SkuId = $a5SkuId
        }
    )
    RemoveLicenses = @()
}
Set-MgUserLicense -UserId "manager@contoso.edu" -BodyParameter $license
```

## ライセンス別設定ガイド

### A1 ライセンス環境での制限事項と対処法

#### 制限事項
- デスクトップ版 Office アプリが利用不可
- Intune によるデバイス管理が利用不可
- 高度なセキュリティ機能が制限

#### 対処法
```powershell
# A1 ユーザーに Web 版 Office の利用を促すメッセージ設定
$messagePolicy = @{
    DisplayName = "A1ユーザー向けお知らせ"
    Description = "Web版Officeの利用を推奨"
    MessageText = "デスクトップ版OfficeはA3/A5ライセンスで利用可能です。現在はWeb版をご利用ください。"
}

# 条件付きアクセスでデスクトップアプリのアクセスを制御
$conditionalAccessPolicy = @{
    DisplayName = "A1ユーザー デスクトップアプリ制限"
    State = "enabled"
    Conditions = @{
        Users = @{
            IncludeUsers = @() # A1ライセンスユーザーのグループIDを指定
        }
        ClientAppTypes = @("mobileAppsAndDesktopClients")
    }
    GrantControls = @{
        Operator = "OR"
        BuiltInControls = @("block")
    }
}
```

### A3 ライセンス環境での活用

#### 主要機能の設定
```powershell
# Intune デバイス管理の有効化（A3以上）
Connect-MSGraph

# コンプライアンスポリシーの作成
$compliancePolicy = @{
    '@odata.type' = "#microsoft.graph.windowsCompliancePolicy"
    displayName = "A3 Windows コンプライアンス"
    passwordRequired = $true
    passwordMinimumLength = 8
    osMinimumVersion = "10.0.19041"
}

New-IntuneDeviceCompliancePolicy -bodyParameter $compliancePolicy

# 条件付きアクセスの設定（A3以上）
$conditionalAccess = @{
    DisplayName = "A3 多要素認証要求"
    State = "enabled"
    Conditions = @{
        Users = @{
            IncludeUsers = @() # A3ライセンスユーザーグループ
        }
        Applications = @{
            IncludeApplications = @("All")
        }
    }
    GrantControls = @{
        Operator = "AND"
        BuiltInControls = @("mfa", "compliantDevice")
    }
}
```

### A5 ライセンス環境での高度な機能

#### Identity Protection の設定
```powershell
# リスクベースの条件付きアクセス（A5のみ）
$riskPolicy = @{
    DisplayName = "A5 リスクベースアクセス制御"
    State = "enabled"
    Conditions = @{
        Users = @{
            IncludeUsers = @() # A5ライセンスユーザーグループ
        }
        UserRiskLevels = @("medium", "high")
        SignInRiskLevels = @("medium", "high")
    }
    GrantControls = @{
        Operator = "AND"
        BuiltInControls = @("mfa", "passwordChange")
    }
}

# Privileged Identity Management の設定
$pimRole = @{
    RoleDefinitionId = "role-id"
    PrincipalId = "user-id"
    RequestType = "AdminAdd"
    Justification = "A5ライセンスによる特権アクセス管理"
    ScheduleInfo = @{
        StartDateTime = (Get-Date)
        Duration = "PT8H"  # 8時間
    }
}
```

## ライセンス最適化戦略

### 自動ライセンス割り当て

```powershell
# 部署に基づくライセンス自動割り当て
function Set-DepartmentBasedLicense {
    param(
        [string]$UserId,
        [string]$Department
    )
    
    $user = Get-MgUser -UserId $UserId -Property Department
    
    switch ($user.Department) {
        "教職員" {
            # 教職員にはA3を割り当て
            $skuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPACK_FACULTY"}).SkuId
        }
        "管理職" {
            # 管理職にはA5を割り当て
            $skuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPREMIUM_FACULTY"}).SkuId
        }
        "児童生徒" {
            # 児童生徒にはA1を割り当て
            $skuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "STANDARDWOFFPACK_STUDENT"}).SkuId
        }
        default {
            # デフォルトはA1
            $skuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "STANDARDWOFFPACK_FACULTY"}).SkuId
        }
    }
    
    $license = @{
        AddLicenses = @(@{ SkuId = $skuId })
        RemoveLicenses = @()
    }
    
    Set-MgUserLicense -UserId $UserId -BodyParameter $license
}

# 新規ユーザー作成時の自動ライセンス割り当て
$newUser = @{
    DisplayName = "新規 教員"
    UserPrincipalName = "newfaculty@contoso.edu"
    Department = "教職員"
    # ... その他のプロパティ
}

$createdUser = New-MgUser @newUser
Set-DepartmentBasedLicense -UserId $createdUser.Id -Department $newUser.Department
```

### ライセンス使用状況レポート

```powershell
# ライセンス使用状況の詳細レポート
function Get-LicenseUsageReport {
    $report = @()
    
    $skus = Get-MgSubscribedSku
    foreach ($sku in $skus) {
        if ($sku.SkuPartNumber -like "*EDUCATION*") {
            $usageData = @{
                License = $sku.SkuPartNumber
                Total = $sku.PrepaidUnits.Enabled
                Consumed = $sku.ConsumedUnits
                Available = $sku.PrepaidUnits.Enabled - $sku.ConsumedUnits
                UsagePercentage = [math]::Round(($sku.ConsumedUnits / $sku.PrepaidUnits.Enabled) * 100, 2)
            }
            $report += New-Object PSObject -Property $usageData
        }
    }
    
    return $report | Sort-Object License
}

# レポートの実行と表示
$usageReport = Get-LicenseUsageReport
$usageReport | Format-Table -AutoSize

# CSV ファイルへの出力
$usageReport | Export-Csv -Path "C:\temp\license-usage-report.csv" -NoTypeInformation -Encoding UTF8
```

## ライセンス別機能制限の管理

### 機能制限の実装

```powershell
# A1 ユーザーに対する機能制限の設定
function Set-A1Restrictions {
    param([string]$GroupId)
    
    # デスクトップ Office アプリのブロック
    $appPolicy = @{
        DisplayName = "A1 デスクトップアプリ制限"
        State = "enabled"
        Conditions = @{
            Users = @{
                IncludeGroups = @($GroupId)
            }
            ClientAppTypes = @("mobileAppsAndDesktopClients")
            Applications = @{
                IncludeApplications = @("d3590ed6-52b3-4102-aeff-aad2292ab01c") # Office 365
            }
        }
        GrantControls = @{
            Operator = "OR"
            BuiltInControls = @("block")
        }
    }
    
    # OneDrive 同期制限
    $oneDrivePolicy = @{
        DisplayName = "A1 OneDrive 制限"
        AllowedDomainGuids = @("tenant-guid")
        BlockMacSync = $true
        DisableReactiveAuth = $true
    }
}

# A3 ユーザーの拡張機能有効化
function Enable-A3Features {
    param([string]$GroupId)
    
    # Intune 管理の有効化
    $intunePolicy = @{
        DisplayName = "A3 Intune 管理"
        DeviceEnrollmentLimit = 10
        AllowUserToJoinDevices = $true
    }
    
    # 条件付きアクセスの適用
    $conditionalAccess = @{
        DisplayName = "A3 条件付きアクセス"
        State = "enabled"
        Conditions = @{
            Users = @{
                IncludeGroups = @($GroupId)
            }
        }
        GrantControls = @{
            BuiltInControls = @("compliantDevice", "mfa")
        }
    }
}
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. ライセンス不足エラー

```powershell
# ライセンス不足の確認
$skus = Get-MgSubscribedSku
foreach ($sku in $skus) {
    $available = $sku.PrepaidUnits.Enabled - $sku.ConsumedUnits
    if ($available -lt 5) {  # 残り5ライセンス未満
        Write-Warning "ライセンス不足: $($sku.SkuPartNumber) - 残り$available"
    }
}

# アラートメールの送信
if ($available -lt 5) {
    Send-MailMessage -To "admin@contoso.edu" -Subject "ライセンス不足警告" -Body "ライセンス追加購入を検討してください"
}
```

#### 2. 機能利用不可エラー

```powershell
# ユーザーの機能利用可否チェック
function Test-FeatureAvailability {
    param(
        [string]$UserId,
        [string]$FeatureName
    )
    
    $user = Get-MgUser -UserId $UserId -Property AssignedLicenses
    $hasA3orA5 = $false
    
    foreach ($license in $user.AssignedLicenses) {
        $sku = Get-MgSubscribedSku -SubscribedSkuId $license.SkuId
        if ($sku.SkuPartNumber -like "*ENTERPRISEPACK*" -or $sku.SkuPartNumber -like "*ENTERPRISEPREMIUM*") {
            $hasA3orA5 = $true
            break
        }
    }
    
    switch ($FeatureName) {
        "Intune" { return $hasA3orA5 }
        "DesktopOffice" { return $hasA3orA5 }
        "ConditionalAccess" { return $hasA3orA5 }
        "IdentityProtection" { 
            return ($sku.SkuPartNumber -like "*ENTERPRISEPREMIUM*")
        }
        default { return $false }
    }
}

# 使用例
$canUseIntune = Test-FeatureAvailability -UserId "user@contoso.edu" -FeatureName "Intune"
if (-not $canUseIntune) {
    Write-Host "ユーザーはIntune機能を利用できません。A3またはA5ライセンスが必要です。"
}
```

## ベストプラクティス

### 1. ライセンス管理

- **定期的な利用状況確認**: 月次でライセンス使用率をレビュー
- **自動化**: 部署やロールに基づく自動ライセンス割り当て
- **コスト最適化**: 必要最小限のライセンスレベルを選択

### 2. 機能制限の適切な実装

- **段階的な制限**: A1→A3→A5の順に機能を段階的に拡張
- **ユーザー教育**: 各ライセンスで利用可能な機能を明確に案内
- **代替手段の提供**: 制限された機能の代替案を提示

### 3. セキュリティ設定

- **ライセンスレベルに応じたセキュリティ**: 各ライセンスで最大限のセキュリティを確保
- **段階的な保護**: A1でも基本的なセキュリティは確保
- **高度な保護**: A5では最新のセキュリティ機能を活用

## 注意事項

- ライセンス変更は即座に反映されますが、一部の機能は最大24時間かかる場合があります
- 下位ライセンスへの変更時は、利用中の機能が停止する可能性があります
- 児童生徒用ライセンス（STUDENT）と教職員用ライセンス（FACULTY）では利用条件が異なります

## 関連情報

- [Microsoft 365 Education ライセンス公式ガイド](https://docs.microsoft.com/ja-jp/microsoft-365/education/)
- [Microsoft Entra ID ライセンス管理](https://docs.microsoft.com/ja-jp/entra/fundamentals/license-users-groups)
- [ユーザー管理](user-management.md)
- [セキュリティ管理](security-management.md)
- [Intune 端末管理](intune-device-management.md)
