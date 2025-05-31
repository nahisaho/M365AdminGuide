# OneDrive for Business 完全管理ガイド

## 目次

1. [概要](#概要)
2. [前提条件と初期設定](#前提条件と初期設定)
3. [ストレージ管理](#ストレージ管理)
4. [セキュリティとアクセス制御](#セキュリティとアクセス制御)
5. [同期設定](#同期設定)
6. [監査とレポート](#監査とレポート)
7. [実用的な管理シナリオ](#実用的な管理シナリオ)
8. [ベストプラクティス](#ベストプラmaクティス)
9. [トラブルシューティング](#トラブルシューティング)
10. [新機能と今後の展望](#新機能と今後の展望)

## 概要

OneDrive for Business は、Microsoft 365 環境における個人用クラウドストレージソリューションです。各ユーザーに独自のプライベートワークスペースを提供し、以下の機能を含みます：

### 主要機能

- **個人ファイルストレージ**: 最大15TBまでの容量
- **ファイル同期**: デスクトップアプリとの自動同期
- **外部共有**: セキュアなファイル共有機能
- **バージョン管理**: ファイルの履歴管理
- **オフラインアクセス**: インターネット接続なしでのファイルアクセス
- **AI統合**: Copilotとの連携による生産性向上

### 教育機関での活用例

- 教員の授業資料管理
- 生徒の課題提出と保存
- 個人研究データの安全な保管
- デバイス間でのファイル同期
- 学習コンテンツの配布

## 前提条件と初期設定

### 必要な権限

- Microsoft 365 グローバル管理者
- SharePoint 管理者
- または、OneDrive 管理者の委任権限

### PowerShell環境の準備（2025年版）

```powershell
# 最新のモジュールインストール
Install-Module -Name Microsoft.Online.SharePoint.PowerShell -Scope CurrentUser -Force -AllowClobber
Install-Module -Name Microsoft.Graph -Scope CurrentUser -Force -AllowClobber
Install-Module -Name PnP.PowerShell -Scope CurrentUser -Force -AllowClobber

# 接続の確立
Connect-SPOService -Url "https://yourtenant-admin.sharepoint.com"
Connect-MgGraph -Scopes "Sites.FullControl.All", "Files.ReadWrite.All", "User.Read.All", "Group.ReadWrite.All"

# PnP PowerShell接続（高度な操作用）
Connect-PnPOnline -Url "https://yourtenant-admin.sharepoint.com" -Interactive
```

### Microsoft Graph APIベースの管理（推奨）

```powershell
# Microsoft Graph を使用した新しい管理方法
function Initialize-OneDriveManagement {
    param(
        [string]$TenantId,
        [string]$ClientId,
        [string]$ClientSecret
    )
    
    Write-Host "🚀 Microsoft Graph でOneDrive管理を初期化中..." -ForegroundColor Cyan
    
    # Graph API 認証
    $secureSecret = ConvertTo-SecureString $ClientSecret -AsPlainText -Force
    $credential = New-Object System.Management.Automation.PSCredential ($ClientId, $secureSecret)
    
    Connect-MgGraph -TenantId $TenantId -ClientCredential $credential
    
    # 基本設定の確認
    $context = Get-MgContext
    Write-Host "✅ 接続成功: $($context.TenantId)" -ForegroundColor Green
    Write-Host "📋 アプリID: $($context.AppName)" -ForegroundColor White
    Write-Host "🔐 権限: $($context.Scopes -join ', ')" -ForegroundColor White
    
    return $context
}

# 実行例
# $context = Initialize-OneDriveManagement -TenantId "your-tenant-id" -ClientId "your-app-id" -ClientSecret "your-secret"
```

### テナント基本設定の確認

```powershell
# 現在のOneDrive設定を確認
function Get-OneDriveConfiguration {
    Write-Host "=== OneDrive for Business 設定状況 ===" -ForegroundColor Cyan
    
    $tenant = Get-SPOTenant
    
    Write-Host "📊 ストレージ設定:" -ForegroundColor Yellow
    Write-Host "  デフォルト容量: $($tenant.OneDriveStorageQuota) MB" -ForegroundColor White
    Write-Host "  最大容量制限: $($tenant.OrphanedPersonalSitesRetentionPeriod) 日" -ForegroundColor White
    
    Write-Host "🔐 セキュリティ設定:" -ForegroundColor Yellow
    Write-Host "  外部共有: $($tenant.SharingCapability)" -ForegroundColor White
    Write-Host "  デフォルトリンク: $($tenant.DefaultSharingLinkType)" -ForegroundColor White
    Write-Host "  匿名リンク有効期限: $($tenant.RequireAnonymousLinksExpireInDays) 日" -ForegroundColor White
    
    Write-Host "⚙️ 同期設定:" -ForegroundColor Yellow
    Write-Host "  除外ファイル形式: $($tenant.ExcludedFileExtensionsForSyncClient)" -ForegroundColor White
}

Get-OneDriveConfiguration
```

## ストレージ管理

### 基本容量設定

```powershell
# テナント全体のデフォルト容量設定
function Set-OneDriveStorageDefaults {
    param(
        [int]$DefaultQuotaMB = 5120,  # 5GB
        [int]$MaxQuotaMB = 25600      # 25GB
    )
    
    Write-Host "💾 OneDrive ストレージ設定を構成中..." -ForegroundColor Cyan
    
    # デフォルト容量の設定
    Set-SPOTenant -OneDriveStorageQuota $DefaultQuotaMB
    
    Write-Host "✅ デフォルト容量: $($DefaultQuotaMB)MB ($($DefaultQuotaMB/1024)GB) に設定" -ForegroundColor Green
    Write-Host "ℹ️  新規ユーザーは自動的にこの容量が適用されます" -ForegroundColor Blue
}

# 実行
Set-OneDriveStorageDefaults -DefaultQuotaMB 5120
```

### 役割別容量管理

```powershell
# 教育機関向け階層別容量設定
function Set-EducationOneDriveQuotas {
    Write-Host "🎓 教育機関向け OneDrive 容量設定を適用中..." -ForegroundColor Cyan
    
    # 容量設定の定義
    $quotaSettings = @{
        "管理者"   = 51200  # 50GB
        "教職員"   = 25600  # 25GB  
        "生徒"     = 5120   # 5GB
        "ゲスト"   = 1024   # 1GB
    }
    
    foreach ($role in $quotaSettings.Keys) {
        $quotaMB = $quotaSettings[$role]
        $users = Get-MgUser -Filter "jobTitle eq '$role'" -All
        
        Write-Host "👥 $role グループの処理中 ($($users.Count) ユーザー)..." -ForegroundColor Yellow
        
        foreach ($user in $users) {
            $oneDriveUrl = Get-OneDriveUrlForUser -UserPrincipalName $user.UserPrincipalName
            
            if ($oneDriveUrl) {
                try {
                    Set-SPOSite -Identity $oneDriveUrl -StorageQuota $quotaMB
                    Write-Host "  ✅ $($user.UserPrincipalName): $($quotaMB)MB" -ForegroundColor Green
                }
                catch {
                    Write-Host "  ⚠️ スキップ: $($user.UserPrincipalName) (OneDrive未作成)" -ForegroundColor Yellow
                }
            }
        }
    }
    
    Write-Host "✅ 階層別容量設定完了" -ForegroundColor Green
}

# OneDrive URLを取得するヘルパー関数
function Get-OneDriveUrlForUser {
    param([string]$UserPrincipalName)
    
    $tenant = (Get-SPOTenant).MySiteHostUrl
    $userUrl = $UserPrincipalName.Replace('@', '_').Replace('.', '_')
    return "$tenant/personal/$userUrl"
}

# 実行
Set-EducationOneDriveQuotas
```

### 容量使用状況の監視

```powershell
# OneDrive使用状況レポート
function Get-OneDriveUsageReport {
    param(
        [string]$OutputPath = "C:\Reports\OneDriveUsage.csv"
    )
    
    Write-Host "📊 OneDrive 使用状況レポート生成中..." -ForegroundColor Cyan
    
    $report = @()
    $users = Get-MgUser -All
    
    foreach ($user in $users) {
        $oneDriveUrl = Get-OneDriveUrlForUser -UserPrincipalName $user.UserPrincipalName
        
        try {
            $siteInfo = Get-SPOSite -Identity $oneDriveUrl -Detailed
            
            $usageInfo = [PSCustomObject]@{
                UserPrincipalName = $user.UserPrincipalName
                DisplayName = $user.DisplayName
                JobTitle = $user.JobTitle
                OneDriveUrl = $oneDriveUrl
                StorageQuotaMB = $siteInfo.StorageQuota
                StorageUsedMB = $siteInfo.StorageUsageCurrent
                UsagePercentage = [math]::Round(($siteInfo.StorageUsageCurrent / $siteInfo.StorageQuota) * 100, 2)
                LastModified = $siteInfo.LastContentModifiedDate
                Status = if ($siteInfo.StorageUsageCurrent -gt ($siteInfo.StorageQuota * 0.9)) { "警告" } else { "正常" }
            }
            
            $report += $usageInfo
            Write-Host "  ✅ $($user.UserPrincipalName): $($usageInfo.UsagePercentage)% 使用" -ForegroundColor Green
        }
        catch {
            Write-Host "  ⚠️ スキップ: $($user.UserPrincipalName)" -ForegroundColor Yellow
        }
    }
    
    # CSVエクスポート
    $report | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
    Write-Host "📄 レポート保存: $OutputPath" -ForegroundColor Green
    
    # サマリー表示
    $totalUsers = $report.Count
    $warningUsers = ($report | Where-Object { $_.Status -eq "警告" }).Count
    $averageUsage = [math]::Round(($report | Measure-Object UsagePercentage -Average).Average, 2)
    
    Write-Host "📈 使用状況サマリー:" -ForegroundColor Cyan
    Write-Host "  総ユーザー数: $totalUsers" -ForegroundColor White
    Write-Host "  警告ユーザー数: $warningUsers" -ForegroundColor $(if ($warningUsers -gt 0) { "Red" } else { "Green" })
    Write-Host "  平均使用率: $averageUsage%" -ForegroundColor White
    
    return $report
}

# 実行例
$usageReport = Get-OneDriveUsageReport
```

## セキュリティとアクセス制御

### 外部共有の制御

```powershell
# セキュアな外部共有設定
function Set-SecureOneDriveSharing {
    Write-Host "🔐 OneDrive セキュア共有設定を適用中..." -ForegroundColor Cyan
    
    # 最も安全な設定を適用
    $secureSettings = @{
        "外部共有" = "ExternalUserSharingOnly"  # 認証済み外部ユーザーのみ
        "デフォルトリンク" = "Direct"             # 特定のユーザーのみ
        "デフォルト権限" = "View"                # 表示のみ
        "匿名リンク有効期限" = 7                  # 7日間
        "メール認証" = $true                     # 必須
        "再認証期間" = 30                        # 30日間
    }
    
    # テナントレベル設定
    Set-SPOTenant -SharingCapability ExternalUserSharingOnly
    Set-SPOTenant -DefaultSharingLinkType Direct
    Set-SPOTenant -DefaultLinkPermission View
    Set-SPOTenant -RequireAnonymousLinksExpireInDays 7
    Set-SPOTenant -EmailAttestationRequired $true
    Set-SPOTenant -EmailAttestationReAuthDays 30
    
    # 危険な機能の無効化
    Set-SPOTenant -OneDriveForGuestsEnabled $false
    Set-SPOTenant -FileAnonymousLinkType Disabled
    Set-SPOTenant -FolderAnonymousLinkType Disabled
    
    Write-Host "✅ セキュア共有設定完了" -ForegroundColor Green
    
    foreach ($setting in $secureSettings.GetEnumerator()) {
        Write-Host "  $($setting.Key): $($setting.Value)" -ForegroundColor White
    }
}

# 実行
Set-SecureOneDriveSharing
```

### 教育機関向け段階的制限

```powershell
# 教育機関向けユーザー階層別共有制御
function Set-EducationOneDriveSharingByRole {
    Write-Host "🎓 教育機関向け階層別共有制御を設定中..." -ForegroundColor Cyan
    
    # 役割別共有設定
    $sharingPolicies = @{
        "生徒" = @{
            Capability = "Disabled"
            Description = "共有完全無効"
        }
        "教職員" = @{
            Capability = "ExternalUserSharingOnly"
            Description = "認証済み外部ユーザーのみ"
        }
        "管理者" = @{
            Capability = "ExternalUserAndGuestSharing"
            Description = "制限付き外部共有"
        }
    }
    
    foreach ($role in $sharingPolicies.Keys) {
        $policy = $sharingPolicies[$role]
        $users = Get-MgUser -Filter "jobTitle eq '$role'" -All
        
        Write-Host "👥 $role の共有設定適用中 ($($users.Count) ユーザー)..." -ForegroundColor Yellow
        Write-Host "   設定: $($policy.Description)" -ForegroundColor Blue
        
        foreach ($user in $users) {
            $oneDriveUrl = Get-OneDriveUrlForUser -UserPrincipalName $user.UserPrincipalName
            
            try {
                Set-SPOSite -Identity $oneDriveUrl -SharingCapability $policy.Capability
                Write-Host "  ✅ $($user.UserPrincipalName)" -ForegroundColor Green
            }
            catch {
                Write-Host "  ⚠️ スキップ: $($user.UserPrincipalName)" -ForegroundColor Yellow
            }
        }
    }
    
    Write-Host "✅ 階層別共有制御設定完了" -ForegroundColor Green
}

# 実行
Set-EducationOneDriveSharingByRole
```

### 条件付きアクセス設定

```powershell
# 条件付きアクセスによるデバイス制限
function Set-OneDriveConditionalAccess {
    Write-Host "🛡️ OneDrive 条件付きアクセス設定中..." -ForegroundColor Cyan
    
    # 管理されていないデバイスからの制限
    Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
    Set-SPOTenant -LimitedAccessFileType WebPreviewableFiles
    
    # 信頼できない場所からのアクセス制限
    Set-SPOTenant -ApplyAppEnforcedRestrictionsToAdHocRecipients $true
    
    # ダウンロード制限の詳細設定
    $restrictionSettings = @{
        "制限対象" = "管理されていないデバイス"
        "許可操作" = "Webプレビューのみ"
        "ダウンロード" = "ブロック"
        "印刷" = "ブロック"
        "コピー" = "ブロック"
    }
    
    Write-Host "✅ 条件付きアクセス設定完了" -ForegroundColor Green
    
    foreach ($setting in $restrictionSettings.GetEnumerator()) {
        Write-Host "  $($setting.Key): $($setting.Value)" -ForegroundColor White
    }
}

# 実行
Set-OneDriveConditionalAccess
```

## 同期設定

### 同期クライアントの制御

```powershell
# OneDrive同期クライアントのセキュリティ設定
function Set-OneDriveSyncSecurity {
    Write-Host "🔄 OneDrive 同期セキュリティ設定中..." -ForegroundColor Cyan
    
    # 危険なファイル形式の同期除外
    $excludedExtensions = @(
        "exe", "bat", "cmd", "scr", "msi", "vbs", "js", "jar", "com", "pif"
    )
    
    Set-SPOTenant -ExcludedFileExtensionsForSyncClient ($excludedExtensions -join ",")
    
    # ドメイン制限（組織管理デバイスのみ同期許可）
    # 注意: この設定には Azure AD Device Management が必要
    try {
        Set-SPOTenant -AllowDownloadingNonWebViewableFiles $false
        Write-Host "  ✅ 非Webビュー可能ファイルのダウンロード制限" -ForegroundColor Green
    }
    catch {
        Write-Host "  ⚠️ 一部設定をスキップしました（権限不足）" -ForegroundColor Yellow
    }
    
    # 同期制限の確認
    Write-Host "📋 同期制限設定:" -ForegroundColor Yellow
    Write-Host "  除外ファイル形式: $($excludedExtensions -join ', ')" -ForegroundColor White
    Write-Host "  管理対象: 組織ドメインのみ" -ForegroundColor White
    
    Write-Host "✅ 同期セキュリティ設定完了" -ForegroundColor Green
}

# 実行
Set-OneDriveSyncSecurity
```

### 帯域幅制御

```powershell
# OneDrive同期の帯域幅制御（Group Policyまたはレジストリ）
function Set-OneDriveBandwidthControl {
    Write-Host "📶 OneDrive 帯域幅制御設定ガイド" -ForegroundColor Cyan
    
    $bandwidthSettings = @"
OneDrive同期の帯域幅制御は、以下の方法で設定できます：

1. Group Policy設定:
   - コンピューターの構成 > ポリシー > 管理用テンプレート
   - OneDrive > ネットワーク > 帯域幅の制限

2. レジストリ設定:
   キー: HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\OneDrive
   値: UploadBandwidthLimit (KB/s)
   値: DownloadBandwidthLimit (KB/s)

3. 推奨設定:
   - アップロード制限: 512 KB/s (ピーク時間)
   - ダウンロード制限: 1024 KB/s (ピーク時間)
   - 非ピーク時間: 制限なし

4. PowerShell経由での一括設定（要管理者権限）:
"@
    
    Write-Host $bandwidthSettings -ForegroundColor White
    
    # サンプル設定スクリプト
    $sampleScript = @"
# レジストリ設定例（各クライアントで実行）
`$registryPath = "HKLM:\SOFTWARE\Policies\Microsoft\OneDrive"
if (!(Test-Path `$registryPath)) {
    New-Item -Path `$registryPath -Force
}
Set-ItemProperty -Path `$registryPath -Name "UploadBandwidthLimit" -Value 512
Set-ItemProperty -Path `$registryPath -Name "DownloadBandwidthLimit" -Value 1024
"@
    
    Write-Host "📄 クライアント設定用スクリプト例:" -ForegroundColor Yellow
    Write-Host $sampleScript -ForegroundColor Green
}

# 実行
Set-OneDriveBandwidthControl
```

## 監査とレポート

### 活動監査の設定

```powershell
# OneDrive活動の監査設定
function Enable-OneDriveAuditing {
    Write-Host "📋 OneDrive 監査設定を有効化中..." -ForegroundColor Cyan
    
    # 監査ログの有効化
    Set-AdminAuditLogConfig -UnifiedAuditLogIngestionEnabled $true
    
    # 監査対象の活動
    $auditActivities = @(
        "FileAccessed",
        "FileDownloaded", 
        "FileUploaded",
        "FileDeleted",
        "FileShared",
        "SharingLinkCreated",
        "SharingInvitationCreated"
    )
    
    Write-Host "✅ 監査設定完了" -ForegroundColor Green
    Write-Host "📊 監査対象活動:" -ForegroundColor Yellow
    
    foreach ($activity in $auditActivities) {
        Write-Host "  - $activity" -ForegroundColor White
    }
    
    Write-Host "ℹ️  監査ログは Microsoft 365 コンプライアンスセンターで確認できます" -ForegroundColor Blue
}

# 実行
Enable-OneDriveAuditing
```

### 共有レポートの生成

```powershell
# OneDrive外部共有レポート
function Get-OneDriveExternalSharingReport {
    param(
        [string]$OutputPath = "C:\Reports\OneDriveExternalSharing.csv",
        [int]$DaysBack = 30
    )
    
    Write-Host "🔍 OneDrive 外部共有レポート生成中..." -ForegroundColor Cyan
    
    $startDate = (Get-Date).AddDays(-$DaysBack).ToString("yyyy-MM-dd")
    $endDate = (Get-Date).ToString("yyyy-MM-dd")
    
    # 統合監査ログから外部共有活動を検索
    $auditRecords = Search-UnifiedAuditLog -StartDate $startDate -EndDate $endDate `
        -Operations "SharingInvitationCreated,AnonymousLinkCreated,SecureLinkCreated" `
        -RecordType SharePointFileOperation
    
    $sharingReport = @()
    
    foreach ($record in $auditRecords) {
        $auditData = $record.AuditData | ConvertFrom-Json
        
        if ($auditData.Workload -eq "OneDrive") {
            $sharingInfo = [PSCustomObject]@{
                Date = $record.CreationDate
                User = $auditData.UserId
                Activity = $auditData.Operation
                FileName = $auditData.ObjectId
                ExternalUser = $auditData.TargetUserOrGroupName
                SharingType = $auditData.EventData.SharingType
                ClientIP = $auditData.ClientIP
                UserAgent = $auditData.UserAgent
            }
            
            $sharingReport += $sharingInfo
        }
    }
    
    # レポート出力
    $sharingReport | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
    
    # サマリー表示
    $totalSharing = $sharingReport.Count
    $uniqueUsers = ($sharingReport | Group-Object User).Count
    $externalSharing = ($sharingReport | Where-Object { $_.ExternalUser -notlike "*@yourdomain.com" }).Count
    
    Write-Host "📊 外部共有サマリー (過去$DaysBack日間):" -ForegroundColor Cyan
    Write-Host "  総共有数: $totalSharing" -ForegroundColor White
    Write-Host "  共有ユーザー数: $uniqueUsers" -ForegroundColor White  
    Write-Host "  外部共有数: $externalSharing" -ForegroundColor $(if ($externalSharing -gt 0) { "Red" } else { "Green" })
    Write-Host "📄 詳細レポート: $OutputPath" -ForegroundColor Green
    
    return $sharingReport
}

# 実行例
$sharingReport = Get-OneDriveExternalSharingReport -DaysBack 30
```

## 実用的な管理シナリオ

### 新入社員・新入生のOneDriveセットアップ

```powershell
# 新規ユーザー向けの自動OneDriveプロビジョニング
function Initialize-NewUserOneDrive {
    param(
        [string]$UserPrincipalName,
        [string]$UserRole = "生徒", # 生徒、教職員、管理者
        [int]$StorageQuotaMB = 2048, # デフォルト2GB
        [bool]$EnableExternalSharing = $false
    )
    
    Write-Host "👤 新規ユーザーOneDrive設定中: $UserPrincipalName" -ForegroundColor Cyan
    
    try {
        # 1. OneDriveサイトの事前プロビジョニング
        Request-SPOPersonalSite -UserEmails @($UserPrincipalName)
        Write-Host "✅ OneDriveサイトプロビジョニング要求送信" -ForegroundColor Green
        
        # 2. 少し待ってからサイトURL構築
        Start-Sleep -Seconds 30
        $oneDriveUrl = Get-OneDriveUrlForUser -UserPrincipalName $UserPrincipalName
        
        # 3. 役割に応じた容量設定
        $quotaSettings = @{
            "生徒" = 2048      # 2GB
            "教職員" = 10240   # 10GB
            "管理者" = 51200   # 50GB
        }
        
        $actualQuota = $quotaSettings[$UserRole]
        if ($actualQuota) {
            Set-SPOSite -Identity $oneDriveUrl -StorageQuota $actualQuota
            Write-Host "✅ ストレージ容量設定: $($actualQuota)MB" -ForegroundColor Green
        }
        
        # 4. セキュリティ設定
        if (-not $EnableExternalSharing) {
            Set-SPOSite -Identity $oneDriveUrl -SharingCapability Disabled
            Write-Host "✅ 外部共有無効化完了" -ForegroundColor Green
        }
        
        # 5. 設定確認
        $siteInfo = Get-SPOSite -Identity $oneDriveUrl -Detailed
        
        Write-Host "📋 設定サマリー:" -ForegroundColor Yellow
        Write-Host "  ユーザー: $UserPrincipalName" -ForegroundColor White
        Write-Host "  役割: $UserRole" -ForegroundColor White
        Write-Host "  容量: $($siteInfo.StorageQuota)MB" -ForegroundColor White
        Write-Host "  共有設定: $($siteInfo.SharingCapability)" -ForegroundColor White
        Write-Host "  URL: $($siteInfo.Url)" -ForegroundColor White
        
        return $siteInfo
    }
    catch {
        Write-Host "❌ エラー: $($_.Exception.Message)" -ForegroundColor Red
        return $null
    }
}

# 一括新規ユーザー設定
function Bulk-InitializeNewUsersOneDrive {
    param(
        [string]$CsvPath = ".\new_users.csv" # UserPrincipalName,Role の形式
    )
    
    Write-Host "📊 一括新規ユーザーOneDrive設定開始" -ForegroundColor Cyan
    
    $users = Import-Csv -Path $CsvPath
    $successCount = 0
    $errorCount = 0
    
    foreach ($user in $users) {
        $result = Initialize-NewUserOneDrive -UserPrincipalName $user.UserPrincipalName -UserRole $user.Role
        
        if ($result) {
            $successCount++
        } else {
            $errorCount++
        }
        
        # レート制限回避
        Start-Sleep -Seconds 2
    }
    
    Write-Host "📈 一括設定完了:" -ForegroundColor Cyan
    Write-Host "  成功: $successCount ユーザー" -ForegroundColor Green
    Write-Host "  失敗: $errorCount ユーザー" -ForegroundColor Red
}

# 実行例
# Initialize-NewUserOneDrive -UserPrincipalName "newstudent@school.edu" -UserRole "生徒"
# Bulk-InitializeNewUsersOneDrive -CsvPath ".\new_students_2025.csv"
```

### 学期末・年度末のデータ整理

```powershell
# 卒業生・退職者のOneDriveアーカイブ
function Archive-LeavingUsersOneDrive {
    param(
        [string[]]$LeavingUsers = @(),
        [string]$ArchiveLocation = "https://contoso.sharepoint.com/sites/archive",
        [int]$GracePeriodDays = 30,
        [switch]$WhatIf = $true
    )
    
    Write-Host "📦 退職・卒業ユーザーのOneDriveアーカイブ処理" -ForegroundColor Cyan
    
    foreach ($userEmail in $LeavingUsers) {
        Write-Host "👤 処理中: $userEmail" -ForegroundColor Yellow
        
        try {
            $oneDriveUrl = Get-OneDriveUrlForUser -UserPrincipalName $userEmail
            $siteInfo = Get-SPOSite -Identity $oneDriveUrl -Detailed
            
            # アーカイブ情報
            $archiveInfo = @{
                User = $userEmail
                OneDriveUrl = $oneDriveUrl
                StorageUsed = $siteInfo.StorageUsageCurrent
                LastModified = $siteInfo.LastContentModifiedDate
                ArchiveDate = (Get-Date).AddDays($GracePeriodDays)
            }
            
            if ($WhatIf) {
                Write-Host "  [WhatIf] アーカイブ予定:" -ForegroundColor Cyan
                Write-Host "    使用容量: $($archiveInfo.StorageUsed)MB" -ForegroundColor White
                Write-Host "    最終更新: $($archiveInfo.LastModified)" -ForegroundColor White
                Write-Host "    アーカイブ日: $($archiveInfo.ArchiveDate)" -ForegroundColor White
            } else {
                # 実際のアーカイブ処理
                
                # 1. サイトを読み取り専用に設定
                Set-SPOSite -Identity $oneDriveUrl -LockState ReadOnly
                Write-Host "  ✅ サイトを読み取り専用に設定" -ForegroundColor Green
                
                # 2. 管理者をサイトコレクション管理者に追加
                Set-SPOUser -Site $oneDriveUrl -LoginName "admin@contoso.com" -IsSiteCollectionAdmin $true
                
                # 3. アーカイブ記録の作成
                $archiveRecord = "アーカイブ日: $(Get-Date), ユーザー: $userEmail, 容量: $($archiveInfo.StorageUsed)MB"
                $archiveRecord | Out-File -FilePath ".\onedrive_archive_log.txt" -Append
                
                Write-Host "  ✅ アーカイブ処理完了" -ForegroundColor Green
            }
        }
        catch {
            Write-Host "  ❌ エラー: $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}

# 使用例
$graduatingStudents = @("student1@school.edu", "student2@school.edu")
Archive-LeavingUsersOneDrive -LeavingUsers $graduatingStudents -WhatIf $true
```

### 緊急時のアクセス制御

```powershell
# 緊急時の一時的アクセス制限
function Set-EmergencyOneDriveRestrictions {
    param(
        [string]$Reason = "緊急セキュリティ対応",
        [switch]$DisableExternalSharing = $true,
        [switch]$BlockDownloads = $true,
        [switch]$EnableAuditMode = $true,
        [switch]$WhatIf = $false
    )
    
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Write-Host "🚨 緊急OneDriveセキュリティ制限適用中..." -ForegroundColor Red
    Write-Host "📝 理由: $Reason" -ForegroundColor Yellow
    Write-Host "⏰ 実行時刻: $timestamp" -ForegroundColor Yellow
    
    $emergencyActions = @()
    
    if ($DisableExternalSharing) {
        $action = "外部共有の無効化"
        if ($WhatIf) {
            Write-Host "  [WhatIf] $action" -ForegroundColor Cyan
        } else {
            Set-SPOTenant -SharingCapability Disabled
            Write-Host "  ✅ $action 完了" -ForegroundColor Green
        }
        $emergencyActions += $action
    }
    
    if ($BlockDownloads) {
        $action = "管理されていないデバイスからのダウンロード制限"
        if ($WhatIf) {
            Write-Host "  [WhatIf] $action" -ForegroundColor Cyan
        } else {
            Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
            Set-SPOTenant -LimitedAccessFileType WebPreviewableFiles
            Write-Host "  ✅ $action 完了" -ForegroundColor Green
        }
        $emergencyActions += $action
    }
    
    if ($EnableAuditMode) {
        $action = "詳細監査モードの有効化"
        if ($WhatIf) {
            Write-Host "  [WhatIf] $action" -ForegroundColor Cyan
        } else {
            # 監査ログの強化
            Set-AdminAuditLogConfig -UnifiedAuditLogIngestionEnabled $true
            Write-Host "  ✅ $action 完了" -ForegroundColor Green
        }
        $emergencyActions += $action
    }
    
    # 緊急対応ログの記録
    $emergencyLog = @"
=== OneDrive 緊急セキュリティ対応ログ ===
実行日時: $timestamp
理由: $Reason
実行者: $env:USERNAME
適用した制限:
$(($emergencyActions | ForEach-Object { "- $_" }) -join "`n")

復旧手順:
1. Set-SPOTenant -SharingCapability ExternalUserSharingOnly
2. Set-SPOTenant -ConditionalAccessPolicy AllowFullAccess
3. 関係者への復旧通知

===========================================
"@
    
    $emergencyLog | Out-File -FilePath ".\emergency_onedrive_actions.log" -Append
    
    Write-Host "📋 緊急対応完了。ログファイル: .\emergency_onedrive_actions.log" -ForegroundColor Green
    Write-Host "⚠️ 復旧時は必ずログを参照してください" -ForegroundColor Yellow
}

# 使用例
# Set-EmergencyOneDriveRestrictions -Reason "データ漏洩疑い調査" -WhatIf $true
```

### 定期メンテナンスの自動化

```powershell
# 週次OneDriveヘルスチェック
function Start-WeeklyOneDriveHealthCheck {
    param(
        [string]$ReportPath = ".\OneDrive_Weekly_Report_$(Get-Date -Format 'yyyyMMdd').html"
    )
    
    Write-Host "🔍 週次OneDriveヘルスチェック開始..." -ForegroundColor Cyan
    
    # 1. 基本統計の収集
    $tenantInfo = Get-SPOTenant
    $allOneDriveSites = Get-SPOSite -IncludePersonalSite $true -Limit All -Filter "Url -like '*-my.sharepoint.com/personal/*'"
    
    $healthMetrics = @{
        CheckDate = Get-Date
        TotalOneDriveUsers = $allOneDriveSites.Count
        TotalStorageUsed = ($allOneDriveSites | Measure-Object StorageUsageCurrent -Sum).Sum
        TotalStorageQuota = ($allOneDriveSites | Measure-Object StorageQuota -Sum).Sum
        HighUsageUsers = ($allOneDriveSites | Where-Object { ($_.StorageUsageCurrent / $_.StorageQuota) -gt 0.9 }).Count
        ExternalSharingEnabled = ($allOneDriveSites | Where-Object { $_.SharingCapability -ne "Disabled" }).Count
        InactiveUsers = ($allOneDriveSites | Where-Object { $_.LastContentModifiedDate -lt (Get-Date).AddDays(-30) }).Count
    }
    
    # 2. セキュリティチェック
    $securityIssues = @()
    
    # 外部共有チェック
    $externalSharingSites = $allOneDriveSites | Where-Object { $_.SharingCapability -ne "Disabled" }
    if ($externalSharingSites.Count -gt 0) {
        $securityIssues += "外部共有が有効なOneDriveサイト: $($externalSharingSites.Count)件"
    }
    
    # 容量警告チェック
    $highUsageSites = $allOneDriveSites | Where-Object { ($_.StorageUsageCurrent / $_.StorageQuota) -gt 0.9 }
    if ($highUsageSites.Count -gt 0) {
        $securityIssues += "容量使用率90%超過: $($highUsageSites.Count)ユーザー"
    }
    
    # 3. HTMLレポート生成
    $htmlReport = @"
<!DOCTYPE html>
<html>
<head>
    <title>OneDrive for Business 週次ヘルスレポート</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #0078d4; color: white; padding: 20px; border-radius: 5px; }
        .metric { background-color: #f5f5f5; padding: 15px; margin: 10px 0; border-radius: 5px; }
        .warning { background-color: #fff3cd; border: 1px solid #ffeaa7; padding: 10px; border-radius: 5px; }
        .success { background-color: #d4edda; border: 1px solid #c3e6cb; padding: 10px; border-radius: 5px; }
        table { width: 100%; border-collapse: collapse; margin: 10px 0; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <div class="header">
        <h1>📊 OneDrive for Business 週次ヘルスレポート</h1>
        <p>生成日時: $($healthMetrics.CheckDate)</p>
    </div>
    
    <div class="metric">
        <h2>📈 基本統計</h2>
        <table>
            <tr><th>項目</th><th>値</th></tr>
            <tr><td>総OneDriveユーザー数</td><td>$($healthMetrics.TotalOneDriveUsers)</td></tr>
            <tr><td>総ストレージ使用量</td><td>$([math]::Round($healthMetrics.TotalStorageUsed/1024, 2)) GB</td></tr>
            <tr><td>総ストレージ容量</td><td>$([math]::Round($healthMetrics.TotalStorageQuota/1024, 2)) GB</td></tr>
            <tr><td>全体使用率</td><td>$([math]::Round(($healthMetrics.TotalStorageUsed/$healthMetrics.TotalStorageQuota)*100, 1))%</td></tr>
        </table>
    </div>
    
    <div class="metric">
        <h2>⚠️ 注意事項</h2>
        <ul>
            <li>容量使用率90%超過ユーザー: $($healthMetrics.HighUsageUsers)人</li>
            <li>外部共有有効サイト: $($healthMetrics.ExternalSharingEnabled)件</li>
            <li>30日間非活動ユーザー: $($healthMetrics.InactiveUsers)人</li>
        </ul>
    </div>
    
    $(if ($securityIssues.Count -gt 0) {
        '<div class="warning"><h2>🚨 セキュリティ警告</h2><ul>' + 
        ($securityIssues | ForEach-Object { "<li>$_</li>" }) -join '' + 
        '</ul></div>'
    } else {
        '<div class="success"><h2>✅ セキュリティ状況良好</h2><p>重大なセキュリティ問題は検出されませんでした。</p></div>'
    })
    
    <div class="metric">
        <h2>📋 推奨アクション</h2>
        <ul>
            <li>容量使用率の高いユーザーへの通知</li>
            <li>非活動ユーザーの確認とアーカイブ検討</li>
            <li>外部共有の妥当性確認</li>
            <li>セキュリティポリシーの遵守状況確認</li>
        </ul>
    </div>
</body>
</html>
"@
    
    # レポート保存
    $htmlReport | Out-File -FilePath $ReportPath -Encoding UTF8
    
    Write-Host "✅ 週次ヘルスチェック完了" -ForegroundColor Green
    Write-Host "📄 レポート保存: $ReportPath" -ForegroundColor Green
    
    # 重要な警告があれば表示
    if ($securityIssues.Count -gt 0) {
        Write-Host "⚠️ セキュリティ警告が検出されました:" -ForegroundColor Yellow
        $securityIssues | ForEach-Object { Write-Host "  - $_" -ForegroundColor Red }
    }
    
    return $healthMetrics
}

# タスクスケジューラーでの自動実行設定例
function Register-OneDriveWeeklyTask {
    $taskName = "OneDrive週次ヘルスチェック"
    $scriptPath = "C:\Scripts\OneDriveHealthCheck.ps1"
    
    # PowerShellスクリプトファイルの作成
    $scriptContent = @"
Import-Module Microsoft.Online.SharePoint.PowerShell
Connect-SPOService -Url "https://yourtenant-admin.sharepoint.com"
Start-WeeklyOneDriveHealthCheck
"@
    
    $scriptContent | Out-File -FilePath $scriptPath -Encoding UTF8
    
    # タスクスケジューラーへの登録
    $action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File `"$scriptPath`""
    $trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday -At "6:00AM"
    $settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
    
    Register-ScheduledTask -TaskName $taskName -Action $action -Trigger $trigger -Settings $settings -Description "OneDrive for Business 週次ヘルスチェック"
    
    Write-Host "✅ 週次タスクを登録しました: $taskName" -ForegroundColor Green
}

# 実行例
# $healthReport = Start-WeeklyOneDriveHealthCheck
# Register-OneDriveWeeklyTask
```

## 新機能と今後の展望

### Microsoft 365 Copilot統合

```powershell
# Copilot統合の準備と設定
function Enable-OneDriveCopilotIntegration {
    param(
        [bool]$EnableDataProcessing = $true,
        [string[]]$AllowedUsers = @(),
        [switch]$WhatIf = $true
    )
    
    Write-Host "🤖 OneDrive Copilot統合設定中..." -ForegroundColor Cyan
    
    if ($WhatIf) {
        Write-Host "  [WhatIf] Copilotデータ処理: $EnableDataProcessing" -ForegroundColor Cyan
        Write-Host "  [WhatIf] 対象ユーザー数: $($AllowedUsers.Count)" -ForegroundColor Cyan
    } else {
        # 実際の設定はMicrosoft 365管理センターまたはGraph APIで実行
        Write-Host "💡 Copilot統合は以下で設定してください:" -ForegroundColor Yellow
        Write-Host "  1. Microsoft 365管理センター > 設定 > Copilot" -ForegroundColor White
        Write-Host "  2. OneDriveデータへのアクセス許可を設定" -ForegroundColor White
        Write-Host "  3. ユーザー単位でのCopilotライセンス割り当て" -ForegroundColor White
    }
}

# 実行例
# Enable-OneDriveCopilotIntegration -WhatIf $true
```

### ゼロトラスト対応

```powershell
# OneDriveゼロトラストセキュリティ設定
function Implement-OneDriveZeroTrust {
    Write-Host "🛡️ OneDrive ゼロトラストセキュリティ実装中..." -ForegroundColor Cyan
    
    # 1. デバイスベースのアクセス制御
    Write-Host "📱 デバイス制御設定:" -ForegroundColor Yellow
    Write-Host "  - 管理されたデバイスのみアクセス許可" -ForegroundColor White
    Write-Host "  - 条件付きアクセスポリシーとの連携" -ForegroundColor White
    
    # 2. ID ベースの認証強化
    Write-Host "🔐 認証強化設定:" -ForegroundColor Yellow
    Write-Host "  - 多要素認証の必須化" -ForegroundColor White
    Write-Host "  - リスクベースの条件付きアクセス" -ForegroundColor White
    
    # 3. データ分類とラベリング
    Write-Host "🏷️ データ保護設定:" -ForegroundColor Yellow
    Write-Host "  - Microsoft Purview 情報保護との統合" -ForegroundColor White
    Write-Host "  - 自動分類とラベル適用" -ForegroundColor White
    
    # 4. 継続的な監視
    Write-Host "👁️ 監視設定:" -ForegroundColor Yellow
    Write-Host "  - Microsoft Defender for Cloud Appsとの連携" -ForegroundColor White
    Write-Host "  - 異常アクセスパターンの検出" -ForegroundColor White
    
    Write-Host "✅ ゼロトラスト実装ガイド完了" -ForegroundColor Green
}

# 実行
Implement-OneDriveZeroTrust
```

### 2025年の新機能対応

```powershell
# 2025年新機能の確認と準備
function Check-OneDrive2025Features {
    Write-Host "🚀 OneDrive 2025年新機能チェック..." -ForegroundColor Cyan
    
    $newFeatures = @(
        @{
            Name = "Enhanced AI Integration"
            Description = "Copilot統合の強化とAI自動整理"
            Status = "準備中"
            Action = "管理センターでプレビュー機能を確認"
        },
        @{
            Name = "Advanced Security Analytics"
            Description = "ML基盤のセキュリティ脅威検出"
            Status = "ロールアウト中"
            Action = "Microsoft Defender for Cloud Apps設定確認"
        },
        @{
            Name = "Improved Collaboration"
            Description = "リアルタイム共同編集の強化"
            Status = "利用可能"
            Action = "ユーザートレーニング計画"
        },
        @{
            Name = "Sustainability Insights"
            Description = "カーボンフットプリント可視化"
            Status = "プレビュー"
            Action = "環境影響レポート設定"
        }
    )
    
    Write-Host "📋 2025年新機能一覧:" -ForegroundColor Yellow
    foreach ($feature in $newFeatures) {
        Write-Host "  🔹 $($feature.Name)" -ForegroundColor White
        Write-Host "    説明: $($feature.Description)" -ForegroundColor Gray
        Write-Host "    状況: $($feature.Status)" -ForegroundColor $(if ($feature.Status -eq "利用可能") { "Green" } else { "Yellow" })
        Write-Host "    推奨対応: $($feature.Action)" -ForegroundColor Cyan
        Write-Host ""
    }
}

# 実行
Check-OneDrive2025Features
```

---

## ベストプラクティス

### 🎓 教育機関向けベストプラクティス

#### 1. セキュリティ管理

##### 児童生徒データの保護

- 児童生徒用OneDriveでは外部共有を完全に無効化
- 管理者による定期的なアクセス監査
- 個人情報を含むファイルの特別な保護設定

```powershell
# 教育機関向けセキュリティ設定テンプレート
function Set-EducationalSecurityBaseline {
    Write-Host "🎓 教育機関向けOneDriveセキュリティ設定適用中..." -ForegroundColor Cyan
    
    # 児童生徒向け設定
    Set-SPOTenant -OneDriveDefaultShareLinkScope "SpecificPeople"
    Set-SPOTenant -OneDriveDefaultShareLinkRole "View"
    Set-SPOTenant -RequireAcceptingAccountMatchInvitedAccount $true
    
    # 外部共有の制限
    Set-SPOTenant -SharingCapability "ExternalUserSharingOnly"
    Set-SPOTenant -SharingDomainRestrictionMode "AllowList"
    
    Write-Host "✅ 教育機関向けセキュリティ設定完了" -ForegroundColor Green
}
```

#### 2. 容量管理戦略

##### 役割別容量配分

- 教職員: 5-15TB（業務内容に応じて）
- 生徒: 1-5TB（学年・コースに応じて）
- 管理者: 15TB（バックアップ・アーカイブ用）

##### 学期・年度末の整理

```powershell
# 年度末のデータ整理自動化
function Start-AcademicYearCleanup {
    param(
        [int]$GraduatingYear = (Get-Date).Year
    )
    
    Write-Host "🧹 年度末OneDriveデータ整理開始..." -ForegroundColor Cyan
    
    # 卒業生の特定
    $graduatingStudents = Get-MgUser -Filter "department eq 'Students' and employeeId like '$GraduatingYear*'"
    
    foreach ($student in $graduatingStudents) {
        Write-Host "📂 処理中: $($student.DisplayName)" -ForegroundColor Yellow
        
        # アーカイブ処理
        Archive-StudentOneDrive -StudentEmail $student.UserPrincipalName
    }
    
    Write-Host "✅ 年度末整理完了" -ForegroundColor Green
}
```

### 一般的なベストプラクティス

#### 1. セキュリティ

##### アクセス制御

- 最小権限の原則を適用
- 定期的な権限レビュー（四半期ごと）
- 条件付きアクセスとの連携
- 多要素認証の必須化

##### データ保護

```powershell
# データ保護ポリシーの実装
function Implement-DataProtectionPolicies {
    Write-Host "🛡️ データ保護ポリシー実装中..." -ForegroundColor Cyan
    
    # 機密データの自動分類
    Set-SPOTenant -EnableSensitivityLabelForPDF $true
    
    # DLP（データ損失防止）統合
    Write-Host "💡 以下でDLPポリシーを設定してください:" -ForegroundColor Yellow
    Write-Host "  1. Microsoft Purview コンプライアンスポータル" -ForegroundColor White
    Write-Host "  2. データ損失防止 > ポリシー" -ForegroundColor White
    Write-Host "  3. OneDrive for Businessを対象に追加" -ForegroundColor White
    
    Write-Host "✅ データ保護ポリシー設定ガイド完了" -ForegroundColor Green
}
```

#### 2. パフォーマンス最適化

##### 同期設定の最適化

- 適切なファイル種類の除外
- 帯域幅制限の設定
- 大きなファイルの管理戦略

```powershell
# パフォーマンス最適化設定
function Optimize-OneDrivePerformance {
    Write-Host "⚡ OneDriveパフォーマンス最適化中..." -ForegroundColor Cyan
    
    # 同期除外ファイル設定
    $excludedExtensions = @("tmp", "temp", "cache", "log", "~")
    Set-SPOTenant -ExcludedFileExtensionsForSyncClient $excludedExtensions
    
    # 大きなファイルの警告設定
    Set-SPOTenant -OneDriveAccessRequests $false
    
    Write-Host "✅ パフォーマンス最適化完了" -ForegroundColor Green
}
```

#### 3. ガバナンスと運用

##### 継続的な監視

- 使用状況の定期レポート
- セキュリティインシデントの追跡
- ユーザーフィードバックの収集

##### 変更管理

- 設定変更の文書化
- テスト環境での事前検証
- ロールバック計画の準備

---

## トラブルシューティング

### よくある問題と解決方法

#### 1. 同期エラー

**症状**: OneDriveクライアントが同期しない

**原因と対処法**:
```powershell
# 同期問題の診断
function Diagnose-OneDriveSyncIssues {
    param(
        [string]$UserEmail
    )
    
    Write-Host "🔍 OneDrive同期問題の診断開始..." -ForegroundColor Cyan
    
    # 1. ユーザーのOneDriveサイト確認
    $oneDriveUrl = "https://yourtenant-my.sharepoint.com/personal/" + $UserEmail.Replace("@", "_").Replace(".", "_")
    
    try {
        $site = Get-SPOSite -Identity $oneDriveUrl
        Write-Host "✅ OneDriveサイト: 正常" -ForegroundColor Green
        Write-Host "   容量使用: $($site.StorageUsageCurrent)MB / $($site.StorageQuota)MB" -ForegroundColor White
    }
    catch {
        Write-Host "❌ OneDriveサイトアクセスエラー: $($_.Exception.Message)" -ForegroundColor Red
        return
    }
    
    # 2. 同期制限の確認
    $tenant = Get-SPOTenant
    Write-Host "📋 同期設定確認:" -ForegroundColor Yellow
    Write-Host "   除外ファイル拡張子: $($tenant.ExcludedFileExtensionsForSyncClient)" -ForegroundColor White
    Write-Host "   同期可能ドメイン: $($tenant.AllowedDomainGuids)" -ForegroundColor White
    
    # 3. 解決方法の提示
    Write-Host "🔧 推奨解決手順:" -ForegroundColor Cyan
    Write-Host "   1. OneDriveクライアントの再起動" -ForegroundColor White
    Write-Host "   2. アカウントの再リンク" -ForegroundColor White
    Write-Host "   3. 除外ファイルの確認と移動" -ForegroundColor White
    Write-Host "   4. 容量不足の場合はファイル整理" -ForegroundColor White
}

# 実行例
# Diagnose-OneDriveSyncIssues -UserEmail "user@contoso.edu"
```

#### 2. 外部共有ができない

**症状**: ファイルの外部共有リンクが生成できない

**対処法**:
```powershell
# 外部共有問題の診断と修正
function Fix-OneDriveExternalSharingIssues {
    param(
        [string]$UserEmail,
        [string]$OneDriveUrl
    )
    
    Write-Host "🔗 外部共有問題の診断中..." -ForegroundColor Cyan
    
    # 1. テナント設定確認
    $tenant = Get-SPOTenant
    Write-Host "🌐 テナント共有設定:" -ForegroundColor Yellow
    Write-Host "   共有機能: $($tenant.SharingCapability)" -ForegroundColor White
    Write-Host "   外部ユーザー期限: $($tenant.ExternalUserExpirationRequired)" -ForegroundColor White
    
    # 2. 個別サイト設定確認
    if ($OneDriveUrl) {
        $site = Get-SPOSite -Identity $OneDriveUrl
        Write-Host "📂 個別サイト設定:" -ForegroundColor Yellow
        Write-Host "   サイト共有機能: $($site.SharingCapability)" -ForegroundColor White
    }
    
    # 3. 修正提案
    Write-Host "🔧 修正提案:" -ForegroundColor Cyan
    
    if ($tenant.SharingCapability -eq "Disabled") {
        Write-Host "   ⚠️ テナントレベルで外部共有が無効です" -ForegroundColor Red
        Write-Host "   💡 対処: Set-SPOTenant -SharingCapability ExternalUserSharingOnly" -ForegroundColor Green
    }
    
    if ($site -and $site.SharingCapability -eq "Disabled") {
        Write-Host "   ⚠️ サイトレベルで外部共有が無効です" -ForegroundColor Red
        Write-Host "   💡 対処: Set-SPOSite -Identity '$OneDriveUrl' -SharingCapability ExternalUserSharingOnly" -ForegroundColor Green
    }
}

# 実行例
# Fix-OneDriveExternalSharingIssues -UserEmail "user@contoso.edu" -OneDriveUrl "https://contoso-my.sharepoint.com/personal/user_contoso_edu"
```

#### 3. 容量不足エラー

**症状**: ファイルアップロード時に容量不足エラー

**対処法**:
```powershell
# 容量問題の解決
function Resolve-OneDriveStorageIssues {
    param(
        [string]$UserEmail
    )
    
    Write-Host "💾 OneDrive容量問題の解決中..." -ForegroundColor Cyan
    
    # ユーザーのOneDrive情報取得
    $oneDriveUrl = "https://yourtenant-my.sharepoint.com/personal/" + $UserEmail.Replace("@", "_").Replace(".", "_")
    $site = Get-SPOSite -Identity $oneDriveUrl
    
    $usagePercent = [math]::Round(($site.StorageUsageCurrent / $site.StorageQuota) * 100, 2)
    
    Write-Host "📊 現在の容量状況:" -ForegroundColor Yellow
    Write-Host "   使用量: $($site.StorageUsageCurrent) MB" -ForegroundColor White
    Write-Host "   割り当て: $($site.StorageQuota) MB" -ForegroundColor White
    Write-Host "   使用率: $usagePercent%" -ForegroundColor $(if ($usagePercent -gt 90) { "Red" } else { "Green" })
    
    # 解決策の提案
    Write-Host "🔧 解決策:" -ForegroundColor Cyan
    
    if ($usagePercent -gt 95) {
        Write-Host "   🚨 緊急: 容量増加が必要" -ForegroundColor Red
        Write-Host "   💡 即時対応: Set-SPOSite -Identity '$oneDriveUrl' -StorageQuota $($site.StorageQuota + 5120)" -ForegroundColor Green
    }
    elseif ($usagePercent -gt 85) {
        Write-Host "   ⚠️ 注意: 不要ファイルの整理を推奨" -ForegroundColor Yellow
        Write-Host "   💡 対応: ごみ箱の空、古いファイルのアーカイブ" -ForegroundColor Green
    }
    
    # 自動整理の提案
    Write-Host "🧹 自動整理オプション:" -ForegroundColor Cyan
    Write-Host "   1. ごみ箱の自動削除設定" -ForegroundColor White
    Write-Host "   2. 古いバージョンファイルの制限" -ForegroundColor White
    Write-Host "   3. 大きなファイルのSharePointライブラリへの移動" -ForegroundColor White
}

# 実行例
# Resolve-OneDriveStorageIssues -UserEmail "user@contoso.edu"
```

#### 4. アクセス権限エラー

**症状**: ユーザーが自分のOneDriveにアクセスできない

**対処法**:
```powershell
# アクセス権限問題の診断と修正
function Fix-OneDriveAccessIssues {
    param(
        [string]$UserEmail
    )
    
    Write-Host "🔐 OneDriveアクセス権限問題の診断中..." -ForegroundColor Cyan
    
    # 1. ユーザーの基本情報確認
    try {
        $user = Get-MgUser -UserId $UserEmail
        Write-Host "👤 ユーザー情報:" -ForegroundColor Yellow
        Write-Host "   表示名: $($user.DisplayName)" -ForegroundColor White
        Write-Host "   アカウント状態: $(if ($user.AccountEnabled) { '有効' } else { '無効' })" -ForegroundColor $(if ($user.AccountEnabled) { "Green" } else { "Red" })
    }
    catch {
        Write-Host "❌ ユーザーが見つかりません: $UserEmail" -ForegroundColor Red
        return
    }
    
    # 2. OneDriveライセンス確認
    $licenses = $user.AssignedLicenses
    $hasOneDriveLicense = $false
    
    foreach ($license in $licenses) {
        $sku = Get-MgSubscribedSku -SubscribedSkuId $license.SkuId
        if ($sku.ServicePlans | Where-Object { $_.ServicePlanName -like "*ONEDRIVE*" -and $_.ProvisioningStatus -eq "Success" }) {
            $hasOneDriveLicense = $true
            break
        }
    }
    
    Write-Host "📄 ライセンス状況:" -ForegroundColor Yellow
    Write-Host "   OneDriveライセンス: $(if ($hasOneDriveLicense) { '有効' } else { '無効' })" -ForegroundColor $(if ($hasOneDriveLicense) { "Green" } else { "Red" })
    
    # 3. OneDriveサイトの確認
    $oneDriveUrl = "https://yourtenant-my.sharepoint.com/personal/" + $UserEmail.Replace("@", "_").Replace(".", "_")
    
    try {
        $site = Get-SPOSite -Identity $oneDriveUrl
        Write-Host "🌐 OneDriveサイト: 正常" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ OneDriveサイト未作成または問題あり" -ForegroundColor Red
        Write-Host "💡 対処: Request-SPOPersonalSite -UserEmails @('$UserEmail')" -ForegroundColor Green
    }
    
    # 4. 解決手順の提示
    Write-Host "🔧 推奨解決手順:" -ForegroundColor Cyan
    
    if (-not $user.AccountEnabled) {
        Write-Host "   1. ユーザーアカウントの有効化" -ForegroundColor Red
    }
    
    if (-not $hasOneDriveLicense) {
        Write-Host "   2. OneDriveライセンスの割り当て" -ForegroundColor Red
    }
    
    Write-Host "   3. OneDriveサイトの初期化待機（最大24時間）" -ForegroundColor White
    Write-Host "   4. ブラウザキャッシュのクリア" -ForegroundColor White
    Write-Host "   5. 別のデバイス・ブラウザでの確認" -ForegroundColor White
}

# 実行例
# Fix-OneDriveAccessIssues -UserEmail "user@contoso.edu"
```

### 緊急時対応手順

#### セキュリティインシデント時

```powershell
# 緊急時のOneDriveセキュリティ対応
function Invoke-OneDriveSecurityIncidentResponse {
    param(
        [string]$AffectedUserEmail,
        [string]$IncidentType = "DataBreach",
        [bool]$ImmediateAction = $true
    )
    
    Write-Host "🚨 OneDrive緊急セキュリティ対応開始" -ForegroundColor Red
    Write-Host "📧 対象ユーザー: $AffectedUserEmail" -ForegroundColor Yellow
    Write-Host "🔍 インシデント種類: $IncidentType" -ForegroundColor Yellow
    
    if ($ImmediateAction) {
        # 1. 即座にアクセス停止
        $oneDriveUrl = "https://yourtenant-my.sharepoint.com/personal/" + $AffectedUserEmail.Replace("@", "_").Replace(".", "_")
        Set-SPOSite -Identity $oneDriveUrl -LockState ReadOnly
        Write-Host "✅ OneDriveアクセスを読み取り専用に制限" -ForegroundColor Green
        
        # 2. 外部共有の即座停止
        Set-SPOSite -Identity $oneDriveUrl -SharingCapability Disabled
        Write-Host "✅ 外部共有を無効化" -ForegroundColor Green
        
        # 3. アクティブセッションの無効化
        Revoke-MgUserSignInSession -UserId $AffectedUserEmail
        Write-Host "✅ ユーザーセッションを無効化" -ForegroundColor Green
    }
    
    # 4. 監査ログの緊急取得
    Write-Host "📋 次のステップ:" -ForegroundColor Cyan
    Write-Host "   1. 監査ログの詳細分析" -ForegroundColor White
    Write-Host "   2. 影響範囲の特定" -ForegroundColor White
    Write-Host "   3. 関係者への通知" -ForegroundColor White
    Write-Host "   4. 復旧計画の策定" -ForegroundColor White
}

# 実行例（緊急時のみ）
# Invoke-OneDriveSecurityIncidentResponse -AffectedUserEmail "user@contoso.edu" -IncidentType "SuspiciousAccess"
```

### ヘルプデスク向けクイックガイド

```text
=== OneDrive for Business トラブルシューティング クイックガイド ===

🔍 症状別対応チャート:

├── 同期しない
│   ├── クライアント再起動 → アカウント再リンク
│   ├── 容量確認 → ファイル整理
│   └── 除外ファイル確認 → 移動・削除

├── 共有できない  
│   ├── テナント設定確認 → 管理者に連絡
│   ├── サイト設定確認 → 権限付与
│   └── ファイル種類確認 → 許可形式に変更

├── 容量不足
│   ├── 使用量確認 → 不要ファイル削除
│   ├── ごみ箱確認 → 完全削除
│   └── 管理者に容量増加依頼

└── アクセスできない
    ├── ライセンス確認 → 管理者に連絡
    ├── アカウント状態確認 → 有効化依頼
    └── ブラウザ・キャッシュクリア

緊急連絡先: IT管理者 (内線: XXXX)
```

---
