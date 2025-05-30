# SharePoint Online 管理

SharePoint Online のサイト管理、ファイル共有制御に関するガイドです。

## 概要

SharePoint Online は、組織内でのファイル共有、コラボレーション、コンテンツ管理を行うためのクラウドプラットフォームです。適切な設定により、セキュリティを維持しながら効率的な情報共有を実現できます。

## 前提条件

- Microsoft 365 管理者権限
- SharePoint 管理者権限
- PowerShell モジュール：Microsoft.Online.SharePoint.PowerShell

## PowerShell 接続

**以下のPowerShellコードの処理内容:**

1. `Install-Module -Name Microsoft.Online.SharePoint.PowerShell` - SharePoint Online管理用PowerShellモジュールをインストール
2. `Connect-SPOService -Url` - SharePoint Online管理センターに接続、テナント固有の管理URLを指定
3. `Connect-MgGraph -Scopes` - Microsoft Graph PowerShell接続（推奨方法）、サイトとグループの完全制御権限を要求

```powershell
# SharePoint Online PowerShell モジュールのインストール
Install-Module -Name Microsoft.Online.SharePoint.PowerShell -Scope CurrentUser

# SharePoint Online 管理センターへの接続
Connect-SPOService -Url "https://yourtenant-admin.sharepoint.com"

# Microsoft Graph PowerShell（推奨）
Connect-MgGraph -Scopes "Sites.FullControl.All", "Group.ReadWrite.All"
```

## サイトコレクション管理

### サイトコレクションの作成

**以下のPowerShellコードの処理内容:**

1. `New-SPOSite` - SharePoint Onlineに新しいサイトコレクションを作成
2. `-Url` - サイトのURLパスを指定、プロジェクト名に合わせた識別可能なURL
3. `-Title` - サイトの表示タイトル、日本語での分かりやすい名称
4. `-Owner` - サイト管理者となるユーザーのメールアドレス
5. `-StorageQuota 1024` - サイトのストレージ容量を1GB（1024MB）に制限
6. `-Template "STS#3"` - モダンチームサイトテンプレートを使用
7. `New-MgSite` - Microsoft Graph APIを使用したサイト作成（新しい推奨方法）

```powershell
# モダンサイトの作成
New-SPOSite -Url "https://contoso.sharepoint.com/sites/project-alpha" `
    -Title "プロジェクト Alpha" `
    -Owner "admin@contoso.com" `
    -StorageQuota 1024 `
    -Template "STS#3"

# Microsoft Graph を使用したサイト作成
$site = @{
    '@odata.type' = "#microsoft.graph.site"
    displayName = "プロジェクト Alpha"
    name = "project-alpha"
}
New-MgSite -BodyParameter $site
```

### サイトコレクションの設定変更

**以下のPowerShellコードの処理内容:**

1. `Set-SPOSite -Identity` - 既存サイトコレクションの設定を変更
2. `-StorageQuota 2048` - サイトのストレージ容量制限を2GB（2048MB）に増加
3. `-DenyAddAndCustomizePages $true` - サイトのカスタマイズとページ追加を禁止してセキュリティを強化

```powershell
# ストレージ容量の変更
Set-SPOSite -Identity "https://contoso.sharepoint.com/sites/project-alpha" -StorageQuota 2048

# サイトの削除保護
Set-SPOSite -Identity "https://contoso.sharepoint.com/sites/project-alpha" -DenyAddAndCustomizePages $true
```

## 外部共有制御

### 🔒 外部共有のデフォルト設定

**重要**: ファイル共有のデフォルト設定で、テナント内の全員に許可しない設定を行います。

```powershell
# テナントレベルでの外部共有制御
Set-SPOTenant -SharingCapability Disabled

# より制限的な設定（認証されたユーザーのみ）
Set-SPOTenant -SharingCapability ExternalUserSharingOnly

# ドメイン制限の設定
Set-SPOTenant -SharingDomainRestrictionMode AllowList
Set-SPOTenant -SharingAllowedDomainList "partner1.com,partner2.com"
```

### 🎯 ファイル共有のデフォルト権限設定

```powershell
# デフォルトの共有リンク設定を制限
# 「テナント内のユーザー」から「特定のユーザー」に変更
Set-SPOTenant -DefaultSharingLinkType Direct

# 共有リンクの有効期限を設定
Set-SPOTenant -RequireAnonymousLinksExpireInDays 30

# ゲストユーザーの再認証期間を設定
Set-SPOTenant -EmailAttestationRequired $true
Set-SPOTenant -EmailAttestationReAuthDays 30
```

### 🚫 危険な設定の無効化

```powershell
# 「組織内のすべてのユーザー」リンクを無効化
Set-SPOTenant -FileAnonymousLinkType View
Set-SPOTenant -FolderAnonymousLinkType View

# 編集可能な匿名リンクを無効化
Set-SPOTenant -FileAnonymousLinkType View

# デフォルトリンク権限を「表示」に制限
Set-SPOTenant -DefaultLinkPermission View
```

### サイト別の外部共有設定

```powershell
# 特定サイトの外部共有を無効化
Set-SPOSite -Identity "https://contoso.sharepoint.com/sites/confidential" -SharingCapability Disabled

# 機密サイトの追加制限
Set-SPOSite -Identity "https://contoso.sharepoint.com/sites/confidential" `
    -ConditionalAccessPolicy AllowLimitedAccess `
    -LimitedAccessFileType WebPreviewableFiles
```

## 🔐 詳細なセキュリティ設定

### ダウンロードブロック設定

```powershell
# 管理されていないデバイスからのダウンロード制限
Set-SPOTenant -ConditionalAccessPolicy AllowLimitedAccess
Set-SPOTenant -LimitedAccessFileType OfficeOnlineFilesOnly

# 特定ファイル形式のダウンロード制限
Set-SPOTenant -ExcludedFileExtensionsForSyncClient "exe,bat,cmd"
```

### 🎓 教育機関向けファイル共有制御

```powershell
# 教育機関向けの厳格な設定
function Set-EducationFileSharingRestrictions {
    param(
        [string[]]$StudentSites = @(),
        [string[]]$FacultySites = @(),
        [string[]]$AdminSites = @()
    )
    
    # 児童生徒サイトの制限（最も厳格）
    foreach ($site in $StudentSites) {
        Set-SPOSite -Identity $site `
            -SharingCapability Disabled `
            -ConditionalAccessPolicy AllowLimitedAccess `
            -LimitedAccessFileType WebPreviewableFiles
        
        Write-Host "✅ 児童生徒サイト制限設定完了: $site" -ForegroundColor Green
    }
    
    # 教職員サイトの設定（中程度の制限）
    foreach ($site in $FacultySites) {
        Set-SPOSite -Identity $site `
            -SharingCapability ExternalUserSharingOnly `
            -DefaultSharingLinkType Direct `
            -DefaultLinkPermission View
        
        Write-Host "✅ 教職員サイト設定完了: $site" -ForegroundColor Green
    }
    
    # 管理サイトの設定（管理者のみアクセス）
    foreach ($site in $AdminSites) {
        Set-SPOSite -Identity $site `
            -SharingCapability Disabled `
            -DenyAddAndCustomizePages $true
        
        Write-Host "✅ 管理サイト設定完了: $site" -ForegroundColor Green
    }
}

# 実行例
$studentSites = @("https://contoso.sharepoint.com/sites/students")
$facultySites = @("https://contoso.sharepoint.com/sites/faculty")
$adminSites = @("https://contoso.sharepoint.com/sites/admin")

Set-EducationFileSharingRestrictions -StudentSites $studentSites -FacultySites $facultySites -AdminSites $adminSites
```

## 権限設定

### サイト権限の管理

```powershell
# サイトコレクション管理者の追加
Set-SPOUser -Site "https://contoso.sharepoint.com/sites/project-alpha" -LoginName "admin@contoso.com" -IsSiteCollectionAdmin $true

# ユーザーグループの権限設定
Grant-SPOSiteDesignRights -Identity "Design-ID" -Principals "group@contoso.com" -Rights View
```

### 権限継承の制御

```powershell
# 権限継承の停止
$site = Get-SPOSite -Identity "https://contoso.sharepoint.com/sites/project-alpha"
Set-SPOSite -Identity $site.Url -DenyAddAndCustomizePages $false

# カスタム権限の設定
# これはSharePoint Online Management Shellでは制限があるため、
# PnP PowerShellまたはサイト内で手動設定を推奨
```

## コンテンツタイプ管理

### コンテンツタイプの作成

```powershell
# Microsoft Graph を使用したコンテンツタイプ管理
$contentType = @{
    name = "プロジェクト文書"
    description = "プロジェクト関連の文書テンプレート"
    base = @{
        name = "Document"
    }
}

# PowerShell での直接的なコンテンツタイプ管理は制限があるため、
# SharePoint管理センターまたはPnP PowerShellを使用
```

## ワークフロー管理

### Power Automate 統合

```powershell
# ワークフローの有効化設定
Set-SPOTenant -DisableCustomAppAuthentication $false

# サイトでのフロー実行許可
Set-SPOSite -Identity "https://contoso.sharepoint.com/sites/project-alpha" -DisableFlows $false
```

## 検索設定

### 検索スキーマの管理

```powershell
# 検索設定の確認
Get-SPOTenant | Select-Object SearchResolveExactEmailOrUPN, UsePersistentCookiesForExplorerView

# 検索結果の制御
Set-SPOTenant -SearchResolveExactEmailOrUPN $false
```

## 監査とコンプライアンス

### 監査ログの有効化

```powershell
# SharePoint 監査ログの有効化
Set-SPOTenant -EnableSharePointAuditLog $true

# 特定サイトの詳細監査
Set-SPOSite -Identity "https://contoso.sharepoint.com/sites/confidential" -EnablePWA $false
```

### 🔍 ファイル共有の監査設定

```powershell
# 外部共有の監査強化
function Enable-SharingAudit {
    param([string[]]$SiteUrls)
    
    foreach ($site in $SiteUrls) {
        # サイトの監査設定を確認
        $auditConfig = Get-SPOSite -Identity $site | Select-Object Title, SharingCapability, ConditionalAccessPolicy
        
        Write-Host "📊 監査設定: $($auditConfig.Title)" -ForegroundColor Cyan
        Write-Host "   共有機能: $($auditConfig.SharingCapability)" -ForegroundColor Yellow
        Write-Host "   条件付きアクセス: $($auditConfig.ConditionalAccessPolicy)" -ForegroundColor Yellow
    }
    
    # 統合監査ログで外部共有イベントを確認
    Write-Host "💡 Microsoft Purview コンプライアンスポータルで以下を確認してください:" -ForegroundColor Cyan
    Write-Host "   - ファイル共有イベント" -ForegroundColor White
    Write-Host "   - 外部ユーザーアクセス" -ForegroundColor White
    Write-Host "   - 権限変更イベント" -ForegroundColor White
}

# 実行例
Enable-SharingAudit -SiteUrls @("https://contoso.sharepoint.com/sites/project-alpha")
```

## Data Loss Prevention (DLP)

### DLP ポリシーとの連携

```powershell
# DLP に対応した SharePoint の設定
Set-SPOTenant -OneDriveForGuestsEnabled $false
Set-SPOTenant -IPAddressEnforcement $true
Set-SPOTenant -IPAddressWACTokenLifetime 15
```

## レポート作成

### 使用状況レポート

```powershell
# サイト使用状況の確認
Get-SPOSite -Limit All | Select-Object Url, StorageUsageCurrent, StorageWarningLevel, LastContentModifiedDate

# 外部共有レポート
Get-SPOExternalUser | Select-Object DisplayName, InvitedAs, WhenCreated, AcceptedAs
```

### セキュリティレポート

```powershell
# 外部共有が有効なサイトの一覧
Get-SPOSite -Limit All | Where-Object {$_.SharingCapability -ne "Disabled"} | 
    Select-Object Url, Title, SharingCapability, ConditionalAccessPolicy
```

## バックアップと復旧

### サイトの削除と復旧

```powershell
# サイトの削除
Remove-SPOSite -Identity "https://contoso.sharepoint.com/sites/old-project" -Confirm:$false

# ごみ箱からの復旧
Restore-SPODeletedSite -Identity "https://contoso.sharepoint.com/sites/old-project"
```

## ベストプラクティス

### 🎓 教育機関向けベストプラクティス

1. **児童生徒データ保護**
   - 児童生徒用サイトでは外部共有を完全に無効化
   - 個人情報を含むファイルには追加の保護を設定
   - 定期的な権限監査を実施

2. **教職員の適切な共有**
   - デフォルトリンクを「特定のユーザー」に設定
   - 編集権限は必要最小限に制限
   - 外部との共有は事前承認制

3. **管理・運用**
   - サイトテンプレートの標準化
   - 命名規則の統一
   - ライフサイクル管理の自動化

### 一般的なベストプラクティス

1. **セキュリティ**
   - 最小権限の原則を適用
   - 条件付きアクセスとの連携
   - 定期的なセキュリティ監査

2. **パフォーマンス**
   - ストレージ容量の適切な管理
   - 不要なファイルの定期削除
   - 検索インデックスの最適化

3. **ガバナンス**
   - サイト作成ポリシーの策定
   - データ保持ポリシーの設定
   - コンプライアンス要件への対応

## トラブルシューティング

### よくある問題と解決方法

1. **外部共有ができない**
   ```powershell
   # テナントと サイトの共有設定を確認
   Get-SPOTenant | Select-Object SharingCapability
   Get-SPOSite -Identity "site-url" | Select-Object SharingCapability
   ```

2. **権限エラー**
   ```powershell
   # ユーザーの権限を確認
   Get-SPOUser -Site "site-url" -LoginName "user@contoso.com"
   ```

3. **同期の問題**
   ```powershell
   # OneDrive 同期設定を確認
   Get-SPOTenant | Select-Object OneDriveStorageQuota, ExcludedFileExtensionsForSyncClient
   ```

## 注意事項

- SharePoint Online の設定変更は即座に反映されない場合があります（最大24時間）
- 外部共有の制限は既存の共有には影響しません
- PowerShell コマンドの実行前は必ずテスト環境で検証してください
- コンプライアンス要件に応じて適切な設定を選択してください

## 関連情報

- [OneDrive for Business 管理](onedrive-management.md)
- [セキュリティ管理](security-management.md)
- [コンプライアンス管理](compliance-management.md)
- [Microsoft Graph PowerShell](microsoft-graph-powershell.md)

## 共有リンクの管理・削除

### 🔗 共有リンクの確認と削除手順

SharePoint Online でファイルやフォルダーの共有リンクを確認・削除する方法を説明します。

#### 1. SharePoint 管理センターでの一括確認

```powershell
# 外部共有されているファイルの確認
function Get-SharedLinksReport {
    param(
        [string[]]$SiteUrls = @(),
        [string]$OutputPath = ".\SharedLinksReport.csv"
    )
    
    Write-Host "🔍 共有リンクレポートを生成中..." -ForegroundColor Cyan
    
    $allSharedItems = @()
    
    if ($SiteUrls.Count -eq 0) {
        # すべてのサイトを対象
        $SiteUrls = (Get-SPOSite -Limit All).Url
    }
    
    foreach ($siteUrl in $SiteUrls) {
        Write-Host "📋 サイト確認中: $siteUrl" -ForegroundColor Yellow
        
        try {
            # 外部ユーザーの確認
            $externalUsers = Get-SPOExternalUser -SiteUrl $siteUrl
            
            foreach ($user in $externalUsers) {
                $sharedItem = [PSCustomObject]@{
                    SiteUrl = $siteUrl
                    ExternalUserEmail = $user.Email
                    InvitedAs = $user.InvitedAs
                    AcceptedAs = $user.AcceptedAs
                    WhenCreated = $user.WhenCreated
                    DisplayName = $user.DisplayName
                }
                $allSharedItems += $sharedItem
            }
        }
        catch {
            Write-Host "⚠️ サイトアクセスエラー: $siteUrl" -ForegroundColor Red
        }
    }
    
    # レポート出力
    $allSharedItems | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
    Write-Host "✅ 共有リンクレポート出力完了: $OutputPath" -ForegroundColor Green
    
    return $allSharedItems
}

# 実行例
$sharedReport = Get-SharedLinksReport
```

#### 2. 特定サイトの共有リンク削除

```powershell
# サイト内の外部共有を一括削除
function Remove-SiteExternalSharing {
    param(
        [string]$SiteUrl,
        [switch]$WhatIf = $true,
        [switch]$Force = $false
    )
    
    Write-Host "🗑️ 外部共有削除処理開始: $SiteUrl" -ForegroundColor Cyan
    
    if (-not $Force) {
        $confirmation = Read-Host "❓ 本当に外部共有を削除しますか？ (y/N)"
        if ($confirmation -ne 'y') {
            Write-Host "❌ 処理をキャンセルしました" -ForegroundColor Yellow
            return
        }
    }
    
    try {
        # 外部ユーザーの取得と削除
        $externalUsers = Get-SPOExternalUser -SiteUrl $SiteUrl
        
        if ($externalUsers.Count -eq 0) {
            Write-Host "📋 外部共有は見つかりませんでした" -ForegroundColor Green
            return
        }
        
        Write-Host "📊 削除対象の外部ユーザー数: $($externalUsers.Count)" -ForegroundColor Yellow
        
        foreach ($user in $externalUsers) {
            if ($WhatIf) {
                Write-Host "🔍 [WhatIf] 削除対象: $($user.Email)" -ForegroundColor Cyan
            } else {
                try {
                    Remove-SPOExternalUser -UniqueId $user.UniqueId -SiteUrl $SiteUrl -Confirm:$false
                    Write-Host "✅ 削除完了: $($user.Email)" -ForegroundColor Green
                }
                catch {
                    Write-Host "❌ 削除エラー: $($user.Email) - $($_.Exception.Message)" -ForegroundColor Red
                }
            }
        }
        
        if ($WhatIf) {
            Write-Host "💡 実際に削除するには -WhatIf $false を指定してください" -ForegroundColor Cyan
        } else {
            Write-Host "✅ 外部共有削除処理完了" -ForegroundColor Green
        }
    }
    catch {
        Write-Host "❌ 処理エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 実行例（確認のみ）
Remove-SiteExternalSharing -SiteUrl "https://contoso.sharepoint.com/sites/project-alpha" -WhatIf $true

# 実際の削除実行
# Remove-SiteExternalSharing -SiteUrl "https://contoso.sharepoint.com/sites/project-alpha" -WhatIf $false -Force
```

#### 3. 特定ユーザーの外部共有削除

```powershell
# 特定の外部ユーザーをテナント全体から削除
function Remove-ExternalUserFromTenant {
    param(
        [string]$ExternalUserEmail,
        [switch]$WhatIf = $true
    )
    
    Write-Host "🔍 外部ユーザー削除処理: $ExternalUserEmail" -ForegroundColor Cyan
    
    try {
        # 外部ユーザーの検索
        $externalUser = Get-SPOExternalUser | Where-Object { $_.Email -eq $ExternalUserEmail }
        
        if (-not $externalUser) {
            Write-Host "❌ 外部ユーザーが見つかりません: $ExternalUserEmail" -ForegroundColor Red
            return
        }
        
        Write-Host "📋 ユーザー情報:" -ForegroundColor Yellow
        Write-Host "   Email: $($externalUser.Email)" -ForegroundColor White
        Write-Host "   DisplayName: $($externalUser.DisplayName)" -ForegroundColor White
        Write-Host "   InvitedAs: $($externalUser.InvitedAs)" -ForegroundColor White
        Write-Host "   AcceptedAs: $($externalUser.AcceptedAs)" -ForegroundColor White
        
        if ($WhatIf) {
            Write-Host "🔍 [WhatIf] 削除対象ユーザー: $($externalUser.Email)" -ForegroundColor Cyan
        } else {
            $confirmation = Read-Host "❓ このユーザーを削除しますか？ (y/N)"
            if ($confirmation -eq 'y') {
                Remove-SPOExternalUser -UniqueId $externalUser.UniqueId -Confirm:$false
                Write-Host "✅ 外部ユーザー削除完了: $ExternalUserEmail" -ForegroundColor Green
            } else {
                Write-Host "❌ 削除をキャンセルしました" -ForegroundColor Yellow
            }
        }
    }
    catch {
        Write-Host "❌ 削除エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 実行例
Remove-ExternalUserFromTenant -ExternalUserEmail "external.user@partner.com" -WhatIf $true
```

#### 4. 共有リンクの有効期限設定と管理

```powershell
# 既存の共有リンクに有効期限を設定
function Set-ExistingSharingLinksExpiration {
    param(
        [string]$SiteUrl,
        [int]$ExpirationDays = 30,
        [switch]$WhatIf = $true
    )
    
    Write-Host "⏰ 共有リンク有効期限設定: $SiteUrl" -ForegroundColor Cyan
    Write-Host "📅 有効期限: $ExpirationDays 日" -ForegroundColor Yellow
    
    # 新しい共有リンクの有効期限を設定
    if ($WhatIf) {
        Write-Host "🔍 [WhatIf] サイトの匿名リンク有効期限を設定: $ExpirationDays 日" -ForegroundColor Cyan
    } else {
        try {
            Set-SPOSite -Identity $SiteUrl -OverrideTenantAnonymousLinkExpirationPolicy $true
            Write-Host "✅ 有効期限設定完了" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ 設定エラー: $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    Write-Host "💡 既存の共有リンクの管理は個別にサイト内で行う必要があります" -ForegroundColor Cyan
}

# 実行例
Set-ExistingSharingLinksExpiration -SiteUrl "https://contoso.sharepoint.com/sites/project-alpha" -ExpirationDays 7 -WhatIf $true
```

### 📱 ブラウザでの手動削除手順

#### サイト管理者による削除

1. **SharePoint サイトにアクセス**
   - 対象のSharePointサイトにアクセス
   - 右上の⚙️（設定）をクリック
   - 「サイトの権限」を選択

2. **外部ユーザーの確認**

   ```text
   サイトの権限 → 外部ユーザー タブ
   - 外部ユーザーの一覧が表示される
   - 各ユーザーの詳細を確認
   ```

3. **外部ユーザーの削除**

   ```text
   削除したいユーザーを選択 → 削除ボタンをクリック
   ⚠️ 注意: この操作は元に戻せません
   ```

#### ファイル・フォルダー別の共有削除

1. **ファイル/フォルダーを選択**
   - 対象のファイルまたはフォルダーを右クリック
   - 「共有」を選択

2. **共有状況の確認**

   ```text
   共有ダイアログ → 「アクセス許可の管理」
   - 現在の共有状況が表示される
   - 各共有リンクの詳細を確認
   ```

3. **共有リンクの削除**

   ```text
   削除したいリンクの「...」→「リンクの削除」
   または
   「アクセス許可の停止」→ ユーザー/グループを削除
   ```

### 🎓 教育機関向け共有リンク管理

```powershell
# 教育機関向けの包括的な共有リンク管理
function Manage-EducationSharingLinks {
    param(
        [string[]]$StudentSites = @(),
        [string[]]$FacultySites = @(),
        [switch]$RemoveAllStudentSharing = $true,
        [switch]$AuditFacultySharing = $true,
        [switch]$WhatIf = $true
    )
    
    Write-Host "🎓 教育機関向け共有リンク管理開始" -ForegroundColor Cyan
    
    # 児童生徒サイトの外部共有を完全削除
    if ($RemoveAllStudentSharing) {
        Write-Host "📚 児童生徒サイトの外部共有削除中..." -ForegroundColor Yellow
        
        foreach ($site in $StudentSites) {
            Write-Host "🔍 処理中: $site" -ForegroundColor White
            
            if ($WhatIf) {
                Write-Host "   [WhatIf] 外部共有を削除予定" -ForegroundColor Cyan
            } else {
                Remove-SiteExternalSharing -SiteUrl $site -WhatIf $false -Force
            }
        }
    }
    
    # 教職員サイトの監査
    if ($AuditFacultySharing) {
        Write-Host "👨‍🏫 教職員サイトの共有監査中..." -ForegroundColor Yellow
        
        foreach ($site in $FacultySites) {
            $externalUsers = Get-SPOExternalUser -SiteUrl $site -ErrorAction SilentlyContinue
            
            if ($externalUsers.Count -gt 0) {
                Write-Host "⚠️ $site に外部共有あり ($($externalUsers.Count)件)" -ForegroundColor Yellow
                
                foreach ($user in $externalUsers) {
                    Write-Host "   📧 $($user.Email) - $($user.InvitedAs)" -ForegroundColor White
                }
            } else {
                Write-Host "✅ $site - 外部共有なし" -ForegroundColor Green
            }
        }
    }
    
    Write-Host "📋 共有リンク管理完了" -ForegroundColor Green
}

# 実行例
$studentSites = @("https://contoso.sharepoint.com/sites/students-class1")
$facultySites = @("https://contoso.sharepoint.com/sites/faculty-math")

Manage-EducationSharingLinks -StudentSites $studentSites -FacultySites $facultySites -WhatIf $true
```

### 🔄 定期的な共有リンククリーンアップ

```powershell
# 定期実行用の共有リンククリーンアップスクリプト
function Start-SharingLinksCleanup {
    param(
        [int]$DaysToExpire = 30,
        [switch]$RemoveExpiredOnly = $true,
        [string]$LogPath = ".\SharingCleanupLog.txt"
    )
    
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "[$timestamp] 共有リンククリーンアップ開始"
    
    Write-Host "🧹 共有リンククリーンアップ開始" -ForegroundColor Cyan
    $logEntry | Out-File -FilePath $LogPath -Append
    
    try {
        # テナントレベルの設定確認
        $tenantSettings = Get-SPOTenant | Select-Object RequireAnonymousLinksExpireInDays, DefaultLinkPermission, DefaultSharingLinkType
        
        Write-Host "📋 現在のテナント設定:" -ForegroundColor Yellow
        Write-Host "   匿名リンク有効期限: $($tenantSettings.RequireAnonymousLinksExpireInDays) 日" -ForegroundColor White
        Write-Host "   デフォルト権限: $($tenantSettings.DefaultLinkPermission)" -ForegroundColor White
        Write-Host "   デフォルトリンクタイプ: $($tenantSettings.DefaultSharingLinkType)" -ForegroundColor White
        
        # すべてのサイトの外部共有確認
        $allSites = Get-SPOSite -Limit All
        $sitesWithSharing = @()
        
        foreach ($site in $allSites) {
            if ($site.SharingCapability -ne "Disabled") {
                $externalUsers = Get-SPOExternalUser -SiteUrl $site.Url -ErrorAction SilentlyContinue
                
                if ($externalUsers.Count -gt 0) {
                    $sitesWithSharing += [PSCustomObject]@{
                        SiteUrl = $site.Url
                        SiteTitle = $site.Title
                        ExternalUserCount = $externalUsers.Count
                        SharingCapability = $site.SharingCapability
                    }
                }
            }
        }
        
        Write-Host "📊 外部共有があるサイト数: $($sitesWithSharing.Count)" -ForegroundColor Yellow
        
        # 結果をログに記録
        $logEntry = "[$timestamp] 外部共有サイト数: $($sitesWithSharing.Count)"
        $logEntry | Out-File -FilePath $LogPath -Append
        
        return $sitesWithSharing
    }
    catch {
        $errorLog = "[$timestamp] エラー: $($_.Exception.Message)"
        Write-Host "❌ エラー: $($_.Exception.Message)" -ForegroundColor Red
        $errorLog | Out-File -FilePath $LogPath -Append
    }
}

# 定期実行例（週次）
# Start-SharingLinksCleanup -DaysToExpire 7
```

### ⚠️ 重要な注意事項

1. **削除前の確認**
   - **必ず `-WhatIf $true` で事前確認**
   - バックアップまたは記録を取得
   - 関係者への事前通知

2. **権限の確認**
   - SharePoint管理者権限が必要
   - サイトコレクション管理者権限でも一部操作可能
   - Microsoft 365 全体管理者権限があれば確実

3. **影響範囲**
   - 外部ユーザーの削除は**元に戻せません**
   - 共有されているファイルへのアクセスが即座に停止
   - 業務への影響を事前に評価

4. **法的・コンプライアンス考慮**
   - データ保持ポリシーとの整合性確認
   - 監査ログの保管
   - 関連法規制への準拠

### 📋 共有リンク削除チェックリスト

```markdown
## 共有リンク削除前チェックリスト

### 事前準備
- [ ] 現在の共有状況をレポート出力
- [ ] 関係者への通知完了
- [ ] バックアップ・記録の取得
- [ ] テスト環境での動作確認

### 実行段階
- [ ] WhatIfモードで動作確認
- [ ] 段階的削除（重要度の低いものから）
- [ ] 実行ログの記録
- [ ] エラー発生時の対応確認

### 事後確認
- [ ] 削除結果の確認
- [ ] 業務への影響確認
- [ ] 監査ログの確認
- [ ] 関係者への完了報告
```
