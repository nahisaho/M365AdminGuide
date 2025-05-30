# OneDrive for Business 管理

OneDrive for Business のストレ        try {
            Set-SPOSite -Identity $oneDriveUrl -StorageQuota $StudentQuotaMB
            Write-Host "✅ 児童生徒ストレージ設定完了: $($student.UserPrincipalName)" -ForegroundColor Green
        }
        catch {
            Write-Host "⚠️ 設定スキップ (OneDrive未作成): $($student.UserPrincipalName)" -ForegroundColor Yellow
        }
    }
    
    # 教職員用クォータ
    $faculty = Get-MgUser -Filter "jobTitle eq '教職員'" -All部共有制御に関するガイドです。

## 概要

OneDrive for Business は、個人用クラウドストレージとファイル同期サービスです。適切な設定により、セキュリティを維持しながら効率的な個人ファイル管理を実現できます。

## 前提条件

- Microsoft 365 管理者権限
- SharePoint 管理者権限
- PowerShell モジュール：Microsoft.Online.SharePoint.PowerShell

## PowerShell 接続

**以下のPowerShellコードの処理内容:**

1. `Install-Module -Name Microsoft.Online.SharePoint.PowerShell` - SharePoint Online管理用PowerShellモジュールをインストール
2. `Connect-SPOService -Url` - SharePoint Online管理センターに接続、テナント固有の管理URLを指定
3. `Connect-MgGraph -Scopes` - Microsoft Graph PowerShell接続（推奨方法）、サイトとファイルの完全制御権限を要求

```powershell
# SharePoint Online PowerShell モジュールのインストール
Install-Module -Name Microsoft.Online.SharePoint.PowerShell -Scope CurrentUser

# SharePoint Online 管理センターへの接続
Connect-SPOService -Url "https://yourtenant-admin.sharepoint.com"

# Microsoft Graph PowerShell（推奨）
Connect-MgGraph -Scopes "Sites.FullControl.All", "Files.ReadWrite.All"
```

## ストレージ容量管理

### テナントレベルのストレージ設定

**以下のPowerShellコードの処理内容:**

1. `Set-SPOTenant -OneDriveStorageQuota 5120` - テナント全体のOneDriveストレージ既定容量を5GB（5120MB）に設定
2. `Set-SPOSite -Identity` - 個別ユーザーのOneDriveサイトを特定のURLで指定
3. `-StorageQuota 10240` - 指定ユーザーのストレージ容量を10GB（10240MB）に個別設定

```powershell
# OneDrive ストレージクォータの設定
Set-SPOTenant -OneDriveStorageQuota 5120  # 5GB

# 個別ユーザーのストレージ容量設定
Set-SPOSite -Identity "https://contoso-my.sharepoint.com/personal/user_contoso_com" -StorageQuota 10240  # 10GB
```

### 🎓 教育機関向けストレージ容量設定

```powershell
# 教育機関向けのユーザー種別ごとの容量設定
function Set-EducationOneDriveQuotas {
    param(
        [int]$StudentQuotaMB = 2048,    # 児童生徒: 2GB
        [int]$FacultyQuotaMB = 10240,   # 教職員: 10GB
        [int]$AdminQuotaMB = 51200      # 管理者: 50GB
    )
    
    # 児童生徒用クォータ（制限的）
    $students = Get-MgUser -Filter "jobTitle eq '児童' or jobTitle eq '生徒'" -All
    foreach ($student in $students) {
        $oneDriveUrl = "https://contoso-my.sharepoint.com/personal/$($student.UserPrincipalName.Replace('@', '_').Replace('.', '_'))"
        try {
            Set-SPOSite -Identity $oneDriveUrl -StorageQuota $StudentQuotaMB
            Write-Host "✅ 児童生徒ストレージ設定完了: $($student.UserPrincipalName)" -ForegroundColor Green
        }
        catch {
            Write-Host "⚠️ 設定スキップ (OneDrive未作成): $($student.UserPrincipalName)" -ForegroundColor Yellow
        }
    }
    
    # 教職員用クォータ
    $faculty = Get-MgUser -Filter "jobTitle eq '教職員'" -All
    foreach ($staff in $faculty) {
        $oneDriveUrl = "https://contoso-my.sharepoint.com/personal/$($staff.UserPrincipalName.Replace('@', '_').Replace('.', '_'))"
        try {
            Set-SPOSite -Identity $oneDriveUrl -StorageQuota $FacultyQuotaMB
            Write-Host "✅ 教職員ストレージ設定完了: $($staff.UserPrincipalName)" -ForegroundColor Green
        }
        catch {
            Write-Host "⚠️ 設定スキップ (OneDrive未作成): $($staff.UserPrincipalName)" -ForegroundColor Yellow
        }
    }
}

# 実行例
Set-EducationOneDriveQuotas
```

## 外部共有ポリシー

### 🔒 外部共有のデフォルト設定（セキュリティ重視）

**重要**: OneDriveのデフォルト共有設定で、テナント内の全員に許可しない設定を実装します。

```powershell
# OneDrive の外部共有を無効化
Set-SPOTenant -OneDriveForGuestsEnabled $false

# デフォルト共有リンクを「特定のユーザー」に設定
Set-SPOTenant -DefaultSharingLinkType Direct

# 匿名リンクの無効化
Set-SPOTenant -FileAnonymousLinkType Disabled
Set-SPOTenant -FolderAnonymousLinkType Disabled

# デフォルト権限を「表示のみ」に制限
Set-SPOTenant -DefaultLinkPermission View
```

### 🚫 危険な共有設定の防止

```powershell
# 「組織内のすべてのユーザー」リンクの制御
function Set-SecureOneDriveDefaults {
    Write-Host "🔐 OneDrive セキュリティ設定を適用中..." -ForegroundColor Cyan
    
    # 最も制限的な設定
    Set-SPOTenant -DefaultSharingLinkType Direct        # 特定のユーザーのみ
    Set-SPOTenant -DefaultLinkPermission View           # 表示権限のみ
    Set-SPOTenant -OneDriveForGuestsEnabled $false      # ゲスト無効
    Set-SPOTenant -RequireAnonymousLinksExpireInDays 1  # 匿名リンク1日で期限切れ
    Set-SPOTenant -EmailAttestationRequired $true       # メール認証必須
    
    # ダウンロード制限
    Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
    Set-SPOTenant -LimitedAccessFileType WebPreviewableFiles
    
    Write-Host "✅ セキュリティ設定完了" -ForegroundColor Green
    Write-Host "📋 適用された設定:" -ForegroundColor Yellow
    Write-Host "   - デフォルトリンク: 特定のユーザー" -ForegroundColor White
    Write-Host "   - デフォルト権限: 表示のみ" -ForegroundColor White
    Write-Host "   - 外部ゲスト: 無効" -ForegroundColor White
    Write-Host "   - ダウンロード: 制限あり" -ForegroundColor White
}

# セキュリティ設定の適用
Set-SecureOneDriveDefaults
```

### 🎓 教育機関向け共有制限

```powershell
# 教育機関向けの段階的な共有制限
function Set-EducationOneDriveSharingPolicy {
    param(
        [string]$StudentPolicy = "Disabled",        # 児童生徒: 共有無効
        [string]$FacultyPolicy = "ExternalUserSharingOnly", # 教職員: 認証ユーザーのみ
        [string]$AdminPolicy = "ExternalUserAndGuestSharing" # 管理者: 制限付き外部共有
    )
    
    Write-Host "🎓 教育機関向け OneDrive 共有ポリシーを設定中..." -ForegroundColor Cyan
    
    # 児童生徒用の OneDrive 設定（最も制限的）
    $students = Get-MgUser -Filter "jobTitle eq '児童' or jobTitle eq '生徒'" -All
    foreach ($student in $students) {
        $oneDriveUrl = "https://contoso-my.sharepoint.com/personal/$($student.UserPrincipalName.Replace('@', '_').Replace('.', '_'))"
        try {
            Set-SPOSite -Identity $oneDriveUrl -SharingCapability $StudentPolicy
            Write-Host "✅ 児童生徒OneDrive制限: $($student.UserPrincipalName)" -ForegroundColor Green
        }
        catch {
            Write-Host "⚠️ 設定スキップ: $($student.UserPrincipalName)" -ForegroundColor Yellow
        }
    }
    
    # 全体的な共有リンクの安全な設定
    Set-SPOTenant -DefaultSharingLinkType Direct
    Set-SPOTenant -DefaultLinkPermission View
    Set-SPOTenant -RequireAnonymousLinksExpireInDays 7
    
    Write-Host "✅ 教育機関向け設定完了" -ForegroundColor Green
}

# 実行例
Set-EducationOneDriveSharingPolicy
```

## 同期設定

### OneDrive 同期クライアントの制御

```powershell
# 同期クライアントの制限設定
Set-SPOTenant -ExcludedFileExtensionsForSyncClient "exe,bat,cmd,scr,msi"

# ドメイン制限（管理されたデバイスのみ同期許可）
Set-SPOTenant -AllowedDomainGuids @("your-azure-ad-tenant-guid")

# 同期機能の詳細制御
Set-SPOTenant -DisableReactiveAuth $true  # レガシー認証無効
Set-SPOTenant -BlockMacSync $false        # Mac同期の制御
```

### 🔐 デバイス管理との連携

```powershell
# Intune 管理デバイスのみ同期許可
function Set-ManagedDeviceOnlySync {
    param(
        [string]$TenantId = "your-tenant-id"
    )
    
    # 管理されたデバイスのみ同期許可
    Set-SPOTenant -AllowedDomainGuids @($TenantId)
    Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
    
    # ファイル種別制限
    Set-SPOTenant -ExcludedFileExtensionsForSyncClient "exe,bat,cmd,scr,msi,app,deb,dmg,jar,ps1"
    
    Write-Host "✅ 管理デバイス限定同期設定完了" -ForegroundColor Green
    Write-Host "📱 Intune で以下の設定も確認してください:" -ForegroundColor Yellow
    Write-Host "   - デバイスコンプライアンスポリシー" -ForegroundColor White
    Write-Host "   - 条件付きアクセスポリシー" -ForegroundColor White
}

# 実行例
Set-ManagedDeviceOnlySync -TenantId "12345678-1234-1234-1234-123456789012"
```

## バックアップ・復旧

### 削除されたファイルの復旧

```powershell
# OneDrive のごみ箱確認（PowerShell では制限あり）
# Microsoft Graph を使用した削除アイテムの確認
Connect-MgGraph -Scopes "Files.ReadWrite.All"

# 特定ユーザーの削除アイテム取得
$userId = "user@contoso.com"
$deletedItems = Get-MgUserDriveRootDeletedItem -UserId $userId

foreach ($item in $deletedItems) {
    Write-Host "削除ファイル: $($item.Name) - 削除日: $($item.DeletedDateTime)" -ForegroundColor Yellow
}
```

### バージョン履歴の管理

```powershell
# バージョン履歴の設定
Set-SPOTenant -EnableVersionExpirationSetting $true
Set-SPOTenant -MajorVersionLimit 50
Set-SPOTenant -MinorVersionLimit 10
```

## デバイス制限

### アクセス制御の詳細設定

```powershell
# IP アドレス制限
Set-SPOTenant -IPAddressEnforcement $true
Set-SPOTenant -IPAddressAllowList "192.168.1.0/24,10.0.0.0/16"

# 管理されていないデバイスからのアクセス制限
Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
Set-SPOTenant -LimitedAccessFileType OfficeOnlineFilesOnly
```

### 🎓 教育機関向けアクセス制御

```powershell
# 学校環境向けの厳格なアクセス制御
function Set-EducationDeviceRestrictions {
    param(
        [string[]]$SchoolIPRanges = @("192.168.0.0/16", "10.0.0.0/8"),
        [bool]$AllowPersonalDevices = $false
    )
    
    Write-Host "🏫 教育機関向けデバイス制限を設定中..." -ForegroundColor Cyan
    
    if ($AllowPersonalDevices -eq $false) {
        # 個人デバイスからのアクセスを制限
        Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
        Set-SPOTenant -LimitedAccessFileType WebPreviewableFiles
        Write-Host "✅ 個人デバイス制限を適用" -ForegroundColor Green
    }
    
    # 学校のIPアドレス範囲のみ許可
    if ($SchoolIPRanges.Count -gt 0) {
        Set-SPOTenant -IPAddressEnforcement $true
        Set-SPOTenant -IPAddressAllowList ($SchoolIPRanges -join ",")
        Write-Host "✅ IP制限を適用: $($SchoolIPRanges -join ', ')" -ForegroundColor Green
    }
    
    # 同期制限
    Set-SPOTenant -ExcludedFileExtensionsForSyncClient "exe,bat,cmd,scr,msi,app,dmg,pkg"
    
    Write-Host "🔐 教育機関向けセキュリティ設定完了" -ForegroundColor Green
}

# 実行例
Set-EducationDeviceRestrictions -SchoolIPRanges @("192.168.1.0/24") -AllowPersonalDevices $false
```

## 監査とコンプライアンス

### OneDrive 使用状況の監視

```powershell
# OneDrive 使用状況レポート
function Get-OneDriveUsageReport {
    # すべてのOneDriveサイトの使用状況
    $oneDriveSites = Get-SPOSite -IncludePersonalSite $true -Limit All -Filter "Url -like '-my.sharepoint.com/personal/'"
    
    $report = @()
    foreach ($site in $oneDriveSites) {
        $userReport = [PSCustomObject]@{
            UserPrincipalName = $site.Owner
            SiteUrl = $site.Url
            StorageUsedMB = $site.StorageUsageCurrent
            StorageQuotaMB = $site.StorageQuota
            StorageUsagePercent = [math]::Round(($site.StorageUsageCurrent / $site.StorageQuota) * 100, 2)
            LastModified = $site.LastContentModifiedDate
            SharingCapability = $site.SharingCapability
        }
        $report += $userReport
    }
    
    # 使用率の高いユーザー表示
    $report | Where-Object { $_.StorageUsagePercent -gt 80 } | 
        Sort-Object StorageUsagePercent -Descending |
        Format-Table UserPrincipalName, StorageUsagePercent, StorageUsedMB, LastModified
        
    return $report
}

# レポートの実行
$usageReport = Get-OneDriveUsageReport
```

### 外部共有の監査

```powershell
# 外部共有されているファイルの確認
function Get-OneDriveExternalSharingReport {
    Write-Host "🔍 外部共有監査を実行中..." -ForegroundColor Cyan
    
    # 外部共有が有効なOneDriveサイト
    $externalSharingSites = Get-SPOSite -IncludePersonalSite $true -Limit All -Filter "Url -like '-my.sharepoint.com/personal/'" |
        Where-Object { $_.SharingCapability -ne "Disabled" }
    
    Write-Host "⚠️ 外部共有が有効なOneDriveサイト数: $($externalSharingSites.Count)" -ForegroundColor Yellow
    
    foreach ($site in $externalSharingSites) {
        Write-Host "📁 $($site.Owner): $($site.SharingCapability)" -ForegroundColor White
    }
    
    # 統合監査ログでの詳細確認を推奨
    Write-Host "💡 詳細な共有アクティビティはMicrosoft Purviewで確認してください" -ForegroundColor Cyan
}

# 監査実行
Get-OneDriveExternalSharingReport
```

## セキュリティ強化

### Data Loss Prevention (DLP) 連携

```powershell
# DLP と連携したOneDrive設定
Set-SPOTenant -OneDriveForGuestsEnabled $false
Set-SPOTenant -NotifyOwnersWhenItemsReshared $true
Set-SPOTenant -NotifyOwnersWhenInvitationsAccepted $true
```

### 🔐 高度なセキュリティ設定

```powershell
# 包括的なセキュリティ設定
function Set-OneDriveMaxSecurity {
    Write-Host "🛡️ OneDrive 最高レベルセキュリティ設定を適用中..." -ForegroundColor Cyan
    
    # 外部共有の完全無効化
    Set-SPOTenant -DefaultSharingLinkType Direct
    Set-SPOTenant -DefaultLinkPermission View
    Set-SPOTenant -OneDriveForGuestsEnabled $false
    Set-SPOTenant -FileAnonymousLinkType Disabled
    Set-SPOTenant -FolderAnonymousLinkType Disabled
    
    # ダウンロード制御
    Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
    Set-SPOTenant -LimitedAccessFileType WebPreviewableFiles
    
    # 認証強化
    Set-SPOTenant -EmailAttestationRequired $true
    Set-SPOTenant -EmailAttestationReAuthDays 15
    
    # 通知強化
    Set-SPOTenant -NotifyOwnersWhenItemsReshared $true
    Set-SPOTenant -NotifyOwnersWhenInvitationsAccepted $true
    
    # 同期制限
    Set-SPOTenant -ExcludedFileExtensionsForSyncClient "exe,bat,cmd,scr,msi,app,dmg,pkg,deb,rpm"
    
    Write-Host "✅ 最高レベルセキュリティ設定完了" -ForegroundColor Green
    Write-Host "📋 適用された制限:" -ForegroundColor Yellow
    Write-Host "   - 外部共有: 完全無効" -ForegroundColor White
    Write-Host "   - ダウンロード: Web表示のみ" -ForegroundColor White
    Write-Host "   - 認証: メール認証15日" -ForegroundColor White
    Write-Host "   - 同期: 実行ファイル除外" -ForegroundColor White
}

# 最高セキュリティ設定の適用
Set-OneDriveMaxSecurity
```

## レポート作成

### 容量使用状況レポート

```powershell
# OneDrive 容量使用状況の詳細レポート
function New-OneDriveCapacityReport {
    param([string]$OutputPath = ".\OneDriveCapacityReport.csv")
    
    Write-Host "📊 OneDrive 容量レポートを生成中..." -ForegroundColor Cyan
    
    $report = Get-SPOSite -IncludePersonalSite $true -Limit All -Filter "Url -like '-my.sharepoint.com/personal/'" |
        Select-Object @{Name='UserEmail';Expression={$_.Owner}},
                     @{Name='StorageUsedGB';Expression={[math]::Round($_.StorageUsageCurrent/1024, 2)}},
                     @{Name='StorageQuotaGB';Expression={[math]::Round($_.StorageQuota/1024, 2)}},
                     @{Name='UsagePercent';Expression={[math]::Round(($_.StorageUsageCurrent/$_.StorageQuota)*100, 2)}},
                     @{Name='LastModified';Expression={$_.LastContentModifiedDate}},
                     @{Name='SharingCapability';Expression={$_.SharingCapability}}
    
    $report | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
    Write-Host "✅ レポート出力完了: $OutputPath" -ForegroundColor Green
    
    # サマリー表示
    $totalUsers = $report.Count
    $averageUsage = [math]::Round(($report | Measure-Object StorageUsedGB -Average).Average, 2)
    $highUsageUsers = ($report | Where-Object { $_.UsagePercent -gt 80 }).Count
    
    Write-Host "📈 レポートサマリー:" -ForegroundColor Yellow
    Write-Host "   総ユーザー数: $totalUsers" -ForegroundColor White
    Write-Host "   平均使用量: $averageUsage GB" -ForegroundColor White
    Write-Host "   使用率80%超: $highUsageUsers ユーザー" -ForegroundColor White
}

# レポート生成
New-OneDriveCapacityReport
```

## ベストプラクティス

### 🎓 教育機関向けベストプラクティス

1. **児童生徒データ保護**
   - 児童生徒のOneDriveは外部共有を完全に無効化
   - 容量制限を設定してコスト管理
   - 定期的な使用状況監査

2. **教職員の適切な利用**
   - デフォルト共有を「特定のユーザー」に限定
   - 個人情報を含むファイルには追加保護
   - 外部共有時の承認プロセス

3. **デバイス管理**
   - 学校支給デバイスのみ同期許可
   - 実行ファイルの同期を禁止
   - IP制限で学校内からのアクセスのみ許可

### 一般的なベストプラクティス

1. **セキュリティ**
   - 最小権限の原則
   - 定期的なアクセス監査
   - DLP ポリシーとの連携

2. **容量管理**
   - 適切なストレージクォータ設定
   - 使用状況の定期監視
   - 不要ファイルの削除推奨

3. **ガバナンス**
   - 共有ポリシーの明確化
   - ユーザー教育の実施
   - インシデント対応手順の策定

## トラブルシューティング

### よくある問題と解決方法

1. **同期エラー**
   ```powershell
   # 同期制限の確認
   Get-SPOTenant | Select-Object ExcludedFileExtensionsForSyncClient, AllowedDomainGuids
   ```

2. **共有ができない**
   ```powershell
   # 共有設定の確認
   Get-SPOSite -Identity "onedrive-url" | Select-Object SharingCapability, ConditionalAccessPolicy
   ```

3. **容量不足**
   ```powershell
   # ストレージ使用量の確認
   Get-SPOSite -Identity "onedrive-url" | Select-Object StorageUsageCurrent, StorageQuota
   ```

## 注意事項

- OneDrive の設定変更は即座に反映されない場合があります
- 個人用OneDriveの一括設定には制限があります
- PowerShell での個別ファイル操作は制限されています
- ユーザビリティとセキュリティのバランスを考慮してください

## 関連情報

- [SharePoint Online 管理](sharepoint-online.md)
- [セキュリティ管理](security-management.md)
- [Intune 端末管理](intune-device-management.md)
- [Microsoft Graph PowerShell](microsoft-graph-powershell.md)
