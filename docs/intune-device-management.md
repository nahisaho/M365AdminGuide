# Microsoft Intune による端末管理

Microsoft Intune を使用したデバイス管理に関する包括的なガイドです。

## 概要

Microsoft Intune は、クラウドベースのエンドポイント管理サービスです。組織のデバイス、アプリ、データを管理し、セキュリティポリシーを適用することができます。

## 前提条件

### 📋 ライセンス要件

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| **Microsoft Intune** | ❌ | ✅ | ✅ |
| **デバイス管理** | ❌ | ✅ | ✅ |
| **アプリ保護ポリシー** | ❌ | ✅ | ✅ |
| **条件付きアクセス連携** | ❌ | ✅ | ✅ |
| **高度な脅威保護** | ❌ | ❌ | ✅ |

⚠️ **重要**: Microsoft Intune はA3またはA5ライセンスが必要です。

- Microsoft Intune ライセンス（A3/A5に含まれる）
- Microsoft Entra ID Premium P1 または P2（推奨）
- Intune 管理者権限
- Microsoft Graph PowerShell モジュール

## Intune の基本概念

### デバイス管理の種類

1. **MDM（モバイルデバイス管理）**
   - デバイス全体を管理
   - 企業所有デバイス向け

2. **MAM（モバイルアプリケーション管理）**
   - アプリとデータのみを管理
   - BYOD（個人デバイス）向け

3. **共同管理**
   - Configuration Manager と Intune の併用
   - ハイブリッド環境向け

## PowerShell による Intune 管理

### 接続とセットアップ

**以下のPowerShellコードの処理内容:**

1. `Install-Module Microsoft.Graph.Intune -Force` - Microsoft Graph Intune PowerShellモジュールをインストール、既存があれば強制上書き
2. `Import-Module Microsoft.Graph.Intune` - インストールしたIntuneモジュールを現在のPowerShellセッションに読み込み
3. `Connect-MSGraph -AdminConsent` - Microsoft Graph APIに管理者同意付きで接続、Intune管理権限を取得
4. `Get-MSGraphEnvironment` - 接続状態と環境情報を確認、正常に接続されているかを検証

```powershell
# Microsoft Graph PowerShell モジュールのインストール
Install-Module Microsoft.Graph.Intune -Force
Import-Module Microsoft.Graph.Intune

# Intune に接続
Connect-MSGraph -AdminConsent

# 接続状態の確認
Get-MSGraphEnvironment
```

## デバイス登録

### 自動登録の設定

**以下のPowerShellコードの処理内容:**

1. `$mdmSettings` - MDM（モバイルデバイス管理）の自動登録設定オブジェクトを作成
2. `MdmUserScope = "All"` - すべてのユーザーに対してMDM登録を有効化
3. `MamUserScope = "All"` - すべてのユーザーに対してMAM（モバイルアプリケーション管理）を有効化
4. Microsoft Entra ID参加デバイスが自動的にIntuneに登録されるよう設定

```powershell
# Microsoft Entra ID 参加デバイスの自動 MDM 登録を有効化
$mdmSettings = @{
    MdmUserScope = "All"  # または "Some", "None"
    MamUserScope = "All"  # または "Some", "None"
}

# 設定を適用（実際の API コール例）
# Update-IntuneDeviceEnrollmentConfiguration
```

### 登録制限の設定

**以下のPowerShellコードの処理内容:**

1. `$deviceTypeRestriction` - デバイスタイプ制限ポリシーのオブジェクトを作成
2. `DisplayName` - ポリシーの表示名、管理者が識別しやすい名称を設定
3. `Description` - ポリシーの説明、承認されたデバイスタイプのみを許可する旨を記載
4. 企業セキュリティポリシーに準拠したデバイスのみの登録を制限

```powershell
# デバイス タイプ制限の作成
$deviceTypeRestriction = @{
    DisplayName = "企業デバイス制限"
    Description = "承認されたデバイスタイプのみ許可"
    PlatformSettings = @{
        iOS = @{
            PersonalDeviceEnrollmentBlocked = $true
            OsMinimumVersion = "14.0"
            OsMaximumVersion = "17.0"
        }
        Android = @{
            PersonalDeviceEnrollmentBlocked = $true
            OsMinimumVersion = "8.0"
            OsMaximumVersion = "14.0"
        }
        Windows = @{
            PersonalDeviceEnrollmentBlocked = $false
            OsMinimumVersion = "10.0.19041"
        }
    }
}

# 制限ポリシーの作成
New-IntuneDeviceEnrollmentPlatformRestriction @deviceTypeRestriction
```

## コンプライアンス ポリシー

### Windows デバイス コンプライアンス

```powershell
# Windows コンプライアンスポリシーの作成
$windowsCompliance = @{
    '@odata.type' = "#microsoft.graph.windowsCompliancePolicy"
    displayName = "Windows コンプライアンス"
    description = "Windows デバイスのセキュリティ要件"
    
    # OS バージョン要件
    osMinimumVersion = "10.0.19041"
    osMaximumVersion = "10.0.22631"
    
    # セキュリティ設定
    passwordRequired = $true
    passwordMinimumLength = 8
    passwordMinutesOfInactivityBeforeLock = 15
    passwordRequiredType = "alphanumeric"
    passwordPreviousPasswordBlockCount = 5
    
    # デバイス正常性
    requireHealthyDeviceReport = $true
    deviceThreatProtectionEnabled = $true
    deviceThreatProtectionRequiredSecurityLevel = "medium"
    
    # BitLocker 暗号化
    bitLockerEnabled = $true
    secureBootEnabled = $true
    codeIntegrityEnabled = $true
    
    # ファイアウォール
    firewallEnabled = $true
    antivirusRequired = $true
    antiSpywareRequired = $true
    
    # システム更新
    defenderEnabled = $true
    defenderVersion = ""
    signatureOutOfDate = $false
    rtpEnabled = $true
    
    # 追加設定
    earlyLaunchAntiMalwareDriverEnabled = $true
    validOperatingSystemBuildRanges = @()
}

# ポリシーの作成
New-IntuneDeviceCompliancePolicy -bodyParameter $windowsCompliance
```

### iOS/Android コンプライアンス

```powershell
# iOS コンプライアンスポリシー
$iosCompliance = @{
    '@odata.type' = "#microsoft.graph.iosCompliancePolicy"
    displayName = "iOS コンプライアンス"
    description = "iOS デバイスのセキュリティ要件"
    
    # OS バージョン
    osMinimumVersion = "15.0"
    osMaximumVersion = "17.0"
    
    # パスコード設定
    passcodeRequired = $true
    passcodeMinimumLength = 6
    passcodeMinutesOfInactivityBeforeLock = 5
    passcodeRequiredType = "numeric"
    passcodePreviousPasscodeBlockCount = 5
    
    # セキュリティ機能
    jailbrokenDevicesBlocked = $true
    deviceThreatProtectionEnabled = $true
    deviceThreatProtectionRequiredSecurityLevel = "medium"
    
    # 制限事項
    managedEmailProfileRequired = $true
    restrictedAppsBlocked = $true
}

# Android コンプライアンスポリシー
$androidCompliance = @{
    '@odata.type' = "#microsoft.graph.androidCompliancePolicy"
    displayName = "Android コンプライアンス"
    description = "Android デバイスのセキュリティ要件"
    
    # OS バージョン
    osMinimumVersion = "8.0"
    osMaximumVersion = "14.0"
    
    # パスワード設定
    passwordRequired = $true
    passwordMinimumLength = 6
    passwordMinutesOfInactivityBeforeLock = 15
    passwordRequiredType = "alphabetic"
    passwordPreviousPasswordBlockCount = 5
    
    # セキュリティ
    securityBlockJailbrokenDevices = $true
    deviceThreatProtectionEnabled = $true
    deviceThreatProtectionRequiredSecurityLevel = "medium"
    
    # Google Play Protect
    securityRequireGooglePlayServices = $true
    securityRequireUpToDateSecurityProviders = $true
    securityRequireVerifyApps = $true
}
```

## 構成プロファイル

### Wi-Fi プロファイル

```powershell
# Wi-Fi 構成プロファイルの作成
$wifiProfile = @{
    '@odata.type' = "#microsoft.graph.windowsWifiConfiguration"
    displayName = "企業Wi-Fi設定"
    description = "自動接続用Wi-Fi設定"
    
    # 基本設定
    networkName = "Corporate-WiFi"
    ssid = "Corporate-Wi-Fi"
    connectAutomatically = $true
    connectWhenNetworkNameIsHidden = $false
    
    # セキュリティ設定
    wiFiSecurityType = "wpa2Enterprise"
    authenticationMethod = "certificate"
    
    # 証明書設定
    rootCertificateForServerValidation = $null
    identityCertificateForClientAuthentication = $null
    
    # EAP 設定
    eapType = "eapTls"
    trustedRootCertificate = $null
    authenticationMethod = "certificate"
}

New-IntuneDeviceConfigurationPolicy -bodyParameter $wifiProfile
```

### VPN プロファイル

```powershell
# VPN 構成プロファイル
$vpnProfile = @{
    '@odata.type' = "#microsoft.graph.windowsVpnConfiguration"
    displayName = "企業VPN設定"
    description = "リモートアクセス用VPN設定"
    
    # 基本設定
    connectionName = "Corporate VPN"
    servers = @(
        @{
            description = "Primary VPN Server"
            address = "vpn.contoso.com"
            isDefaultServer = $true
        }
    )
    
    # 接続設定
    connectionType = "ikEv2"
    authenticationMethod = "certificate"
    splitTunneling = $false
    
    # 認証設定
    enableSplitTunneling = $false
    enableAlwaysOn = $true
    enableDeviceTunnel = $false
    
    # DNS設定
    dnsSuffixes = @("contoso.com", "internal.contoso.com")
    dnsServers = @("10.0.0.1", "10.0.0.2")
}

New-IntuneDeviceConfigurationPolicy -bodyParameter $vpnProfile
```

## アプリケーション管理

### Microsoft Store アプリの展開

```powershell
# Microsoft Store アプリの追加
$storeApp = @{
    '@odata.type' = "#microsoft.graph.winGetApp"
    displayName = "Microsoft Teams"
    description = "Microsoft Teams デスクトップアプリ"
    publisher = "Microsoft Corporation"
    packageIdentifier = "Microsoft.Teams"
    
    # 割り当て設定
    assignments = @(
        @{
            intent = "required"
            target = @{
                '@odata.type' = "#microsoft.graph.allLicensedUsersAssignmentTarget"
            }
        }
    )
}

New-IntuneApplication -bodyParameter $storeApp
```

### LOB（基幹業務）アプリの展開

```powershell
# MSI パッケージのアップロード例
$lobApp = @{
    '@odata.type' = "#microsoft.graph.win32LobApp"
    displayName = "社内システム"
    description = "社内基幹システムクライアント"
    publisher = "Contoso Corporation"
    
    # インストール設定
    installCommandLine = "setup.exe /S"
    uninstallCommandLine = "setup.exe /uninstall /S"
    
    # 検出ルール
    detectionRules = @(
        @{
            '@odata.type' = "#microsoft.graph.win32LobAppFileSystemDetection"
            path = "C:\Program Files\ContosoApp"
            fileOrFolderName = "ContosoApp.exe"
            check32BitOn64System = $false
            detectionType = "exists"
        }
    )
    
    # 要件
    minimumSupportedOperatingSystem = @{
        v10_1607 = $true
    }
    
    # 割り当て
    assignments = @(
        @{
            intent = "required"
            target = @{
                '@odata.type' = "#microsoft.graph.groupAssignmentTarget"
                groupId = "group-id-here"
            }
        }
    )
}
```

## アプリ保護ポリシー（MAM）

### iOS アプリ保護ポリシー

```powershell
# iOS アプリ保護ポリシー
$iosMAMPolicy = @{
    '@odata.type' = "#microsoft.graph.iosManagedAppProtection"
    displayName = "iOS アプリ保護"
    description = "iOS デバイスのアプリとデータ保護"
    
    # データ保護設定
    dataBackupBlocked = $true
    deviceComplianceRequired = $true
    managedBrowserToOpenLinksRequired = $true
    saveAsBlocked = $true
    
    # アクセス設定
    pinRequired = $true
    maximumPinRetries = 5
    simplePinBlocked = $true
    minimumPinLength = 4
    pinCharacterSet = "numeric"
    
    # 条件付きアクセス
    periodOfflineBeforeAccessCheck = "PT12H"
    periodOnlineBeforeAccessCheck = "PT30M"
    allowedInboundDataTransferSources = "managedApps"
    allowedOutboundDataTransferDestinations = "managedApps"
    
    # アプリ制限
    allowedOutboundClipboardSharingLevel = "managedAppsWithPasteIn"
    allowedInboundClipboardSharingLevel = "managedAppsWithPasteIn"
    contactSyncBlocked = $false
    printBlocked = $true
    
    # 対象アプリ
    apps = @(
        @{
            mobileAppIdentifier = @{
                '@odata.type' = "#microsoft.graph.iosMobileAppIdentifier"
                bundleId = "com.microsoft.msedge"
            }
        },
        @{
            mobileAppIdentifier = @{
                '@odata.type' = "#microsoft.graph.iosMobileAppIdentifier"
                bundleId = "com.microsoft.office.outlook"
            }
        }
    )
}

New-IntuneAppProtectionPolicy -bodyParameter $iosMAMPolicy
```

## 条件付きアクセス連携

### Intune コンプライアンス要求

```powershell
# 条件付きアクセスポリシーでのコンプライアンス要求
$conditionalAccessPolicy = @{
    displayName = "コンプライアント デバイス要求"
    state = "enabled"
    
    conditions = @{
        users = @{
            includeUsers = @("all")
            excludeUsers = @()
        }
        applications = @{
            includeApplications = @("Office365")
        }
        platforms = @{
            includePlatforms = @("windows", "iOS", "android")
        }
    }
    
    grantControls = @{
        operator = "AND"
        builtInControls = @("compliantDevice", "mfa")
    }
}
```

## レポートと監視

### デバイス コンプライアンス レポート

```powershell
# コンプライアンス状況の取得
Get-IntuneDeviceComplianceDeviceStatus | Where-Object {$_.status -eq "nonCompliant"} | Select-Object deviceDisplayName, userName, status, lastReportedDateTime

# デバイス登録レポート
Get-IntuneDevice | Group-Object deviceEnrollmentType | Select-Object Name, Count

# アプリ保護ポリシー レポート
Get-IntuneAppProtectionStatus | Select-Object userDisplayName, appDisplayName, complianceStatus, lastCheckInDateTime
```

### PowerBI ダッシュボード

```powershell
# Intune データウェアハウス API を使用したレポート
$intuneDataWarehouse = "https://graph.microsoft.com/beta/deviceManagement/reports"

# カスタムレポートの作成例
$reportRequest = @{
    reportName = "DeviceComplianceReport"
    filter = ""
    select = @("DeviceName", "UserName", "ComplianceStatus", "LastContact")
    orderBy = @("LastContact desc")
}
```

## トラブルシューティング

### 一般的な問題と解決方法

1. **デバイス登録の失敗**
   ```powershell
   # 登録エラーの確認
   Get-IntuneDeviceEnrollmentFailure | Select-Object deviceId, failureReason, errorCode
   
   # Microsoft Entra ID デバイス登録状況の確認
   Get-AzureADDevice -Filter "displayName eq 'DeviceName'"
   ```

2. **コンプライアンス評価の遅延**
   ```powershell
   # デバイスの同期を強制実行
   Sync-IntuneDevice -deviceId "device-id-here"
   
   # コンプライアンス状況の手動更新
   Update-IntuneDeviceComplianceDeviceStatus -deviceId "device-id-here"
   ```

3. **アプリ展開の問題**
   ```powershell
   # アプリインストール状況の確認
   Get-IntuneApplicationInstallStatus -applicationId "app-id-here"
   
   # 失敗したインストールの詳細
   Get-IntuneApplicationInstallStatus | Where-Object {$_.installState -eq "failed"}
   ```

## ベストプラクティス

### セキュリティ

1. **段階的な展開**
   - パイロットグループでテスト
   - 段階的にユーザーグループを拡大
   - 問題発生時の迅速なロールバック計画

2. **コンプライアンス設計**
   - 業務要件とセキュリティのバランス
   - ユーザビリティを考慮した設定
   - 定期的なポリシーの見直し

3. **監視とレポート**
   - 定期的なコンプライアンス状況の確認
   - 異常なデバイス動作の監視
   - インシデント対応計画の策定

### 運用管理

1. **ライフサイクル管理**
   ```powershell
   # デバイスの退職処理
   Remove-IntuneDevice -deviceId "device-id-here" -Confirm:$false
   
   # 企業データのワイプ
   Clear-IntuneDeviceCompanyData -deviceId "device-id-here"
   ```

2. **自動化スクリプト**
   ```powershell
   # 定期的なコンプライアンス確認スクリプト
   $nonCompliantDevices = Get-IntuneDeviceComplianceDeviceStatus | Where-Object {$_.status -eq "nonCompliant"}
   
   foreach ($device in $nonCompliantDevices) {
       # 管理者への通知
       Send-MailMessage -To "admin@contoso.com" -Subject "非準拠デバイス検出" -Body "デバイス: $($device.deviceDisplayName)"
       
       # ユーザーへの通知
       Send-IntuneNotificationMessage -deviceId $device.deviceId -message "デバイスがコンプライアンス要件を満たしていません。"
   }
   ```

## 注意事項

- ポリシー変更は段階的に実装してください
- 重要なビジネスアプリケーションの動作確認を必ず行ってください
- エンドユーザーへの事前通知と教育を実施してください
- 緊急時のポリシー無効化手順を準備してください

## 関連情報

- [Microsoft Intune 公式ドキュメント](https://docs.microsoft.com/ja-jp/mem/intune/)
- [Microsoft Graph Intune API](https://docs.microsoft.com/ja-jp/graph/api/resources/intune-graph-overview)
- [セキュリティ管理](security-management.md)
- [ユーザー管理](user-management.md)

# Windows Autopilot による自動展開

## 🚀 Windows Autopilot とは

Windows Autopilot は、新しいデバイスをゼロタッチでセットアップし、組織の設定を自動的に適用する機能です。デバイスが工場出荷時の状態からすぐにビジネス対応できるように構成されます。

### Autopilot の利点

- **ゼロタッチ展開**: IT担当者による手動設定が不要
- **ユーザー主導**: エンドユーザーが自分でセットアップ可能
- **コスト削減**: 物理的なイメージ作成・管理が不要
- **セキュリティ**: デバイスが企業ポリシーに準拠した状態で起動
- **復旧支援**: デバイスのリセット時も自動で組織設定を復元

### 配布シナリオ

| シナリオ | 説明 | 用途 |
|----------|------|------|
| **User-driven** | ユーザーが自分で初期設定を実行 | 一般的な企業デバイス |
| **Self-deploying** | 完全自動での展開（ユーザー入力なし） | 共有デバイス・キオスク |
| **White glove** | IT 部門による事前設定後にユーザーへ渡す | 高セキュリティ環境 |
| **Pre-provisioning** | 部分的な設定を IT 部門で完了 | ハイブリッド展開 |

## 📋 前提条件と要件

### ライセンス要件

| 機能 | Business Premium | A3 | A5 |
|------|-------------------|----|----|
| **Windows Autopilot** | ✅ | ✅ | ✅ |
| **Microsoft Intune** | ✅ | ✅ | ✅ |
| **Microsoft Entra ID Premium** | P1 | P1 | P2 |
| **Self-deploying mode** | ✅ | ✅ | ✅ |
| **White glove** | ✅ | ✅ | ✅ |

⚠️ **重要**: Autopilot は Windows 10/11 Pro 以上のエディションが必要です。

### ハードウェア要件

```powershell
# ハードウェア要件チェック
function Test-AutopilotRequirements {
    Write-Host "🔍 Autopilot ハードウェア要件チェック" -ForegroundColor Cyan
    
    # TPM バージョン確認
    $tpm = Get-WmiObject -Namespace "root\cimv2\security\microsofttpm" -Class Win32_Tpm
    if ($tpm) {
        $tpmVersion = $tpm.PhysicalPresenceVersionInfo
        Write-Host "✅ TPM バージョン: $tpmVersion" -ForegroundColor Green
    } else {
        Write-Host "❌ TPM が検出されませんでした" -ForegroundColor Red
    }
    
    # UEFI 確認
    $firmware = Get-ComputerInfo | Select-Object BiosFirmwareType
    if ($firmware.BiosFirmwareType -eq "Uefi") {
        Write-Host "✅ UEFI ファームウェア" -ForegroundColor Green
    } else {
        Write-Host "❌ Legacy BIOS (Autopilot 非対応)" -ForegroundColor Red
    }
    
    # Windows バージョン確認
    $build = [System.Environment]::OSVersion.Version.Build
    if ($build -ge 18362) {
        Write-Host "✅ Windows ビルド: $build (対応)" -ForegroundColor Green
    } else {
        Write-Host "❌ Windows ビルド: $build (要更新)" -ForegroundColor Red
    }
}

Test-AutopilotRequirements
```

**必須要件:**
- **TPM 2.0** チップ搭載
- **UEFI** ファームウェア (レガシーBIOSは非対応)
- **安定したインターネット接続**
- **Windows 10 1903** 以降 または **Windows 11**

### ネットワーク要件

```powershell
# Autopilot に必要なエンドポイント
$autopilotEndpoints = @{
    "Windows Autopilot" = @(
        "*.delivery.mp.microsoft.com",
        "*.prod.do.dsp.mp.microsoft.com",
        "emdl.ws.microsoft.com",
        "*.dl.delivery.mp.microsoft.com"
    )
    "Windows Update" = @(
        "*.windowsupdate.com",
        "download.windowsupdate.com",
        "*.download.windowsupdate.com"
    )
    "認証" = @(
        "login.live.com",
        "login.microsoftonline.com",
        "*.login.microsoftonline.com"
    )
    "デバイス診断" = @(
        "cs.dds.microsoft.com",
        "dmd.metaservices.microsoft.com"
    )
    "Teams" = @(
        "config.teams.microsoft.com",
        "*.teams.microsoft.com"
    )
}

Write-Host "🌐 Autopilot 必要エンドポイント:" -ForegroundColor Cyan
foreach ($category in $autopilotEndpoints.Keys) {
    Write-Host "`n📂 $category" -ForegroundColor Yellow
    foreach ($endpoint in $autopilotEndpoints[$category]) {
        Write-Host "  - $endpoint" -ForegroundColor White
    }
}
```

## ⚙️ Autopilot セットアップ手順

### ステップ 1: PowerShell 環境の準備

```powershell
# 必要なモジュールのインストール
$modules = @(
    "Microsoft.Graph.Authentication",
    "Microsoft.Graph.Identity.DirectoryManagement", 
    "Microsoft.Graph.DeviceManagement",
    "WindowsAutoPilotIntune"
)

foreach ($module in $modules) {
    if (!(Get-Module -ListAvailable -Name $module)) {
        Write-Host "📦 $module をインストール中..." -ForegroundColor Yellow
        Install-Module $module -Force -Scope CurrentUser
    } else {
        Write-Host "✅ $module は既にインストール済み" -ForegroundColor Green
    }
}

# Microsoft Graph に接続
$scopes = @(
    "DeviceManagementServiceConfig.ReadWrite.All",
    "DeviceManagementConfiguration.ReadWrite.All", 
    "DeviceManagementManagedDevices.ReadWrite.All"
)

Connect-MgGraph -Scopes $scopes
Write-Host "✅ Microsoft Graph に接続しました" -ForegroundColor Green
```

### ステップ 2: デバイス登録

#### 方法 1: ハードウェア ID による登録

```powershell
# デバイスのハードウェア ID を取得するスクリプト
function Get-AutopilotHardwareId {
    param(
        [string]$OutputPath = "C:\Temp\autopilot-hardware.csv"
    )
    
    Write-Host "🔍 ハードウェア ID を取得中..." -ForegroundColor Cyan
    
    # ディレクトリ作成
    $dir = Split-Path $OutputPath -Parent
    if (!(Test-Path $dir)) {
        New-Item -Path $dir -ItemType Directory -Force
    }
    
    # Get-WindowsAutoPilotInfo スクリプトをダウンロード・実行
    $scriptUrl = "https://www.powershellgallery.com/packages/Get-WindowsAutoPilotInfo"
    Install-Script -Name Get-WindowsAutoPilotInfo -Force
    
    # ハードウェア ID 取得
    Get-WindowsAutoPilotInfo -OutputFile $OutputPath
    
    Write-Host "✅ ハードウェア ID を $OutputPath に保存しました" -ForegroundColor Green
    Write-Host "📁 ファイルをIntune管理センターにアップロードしてください" -ForegroundColor Yellow
}

# ハードウェア ID 取得実行
Get-AutopilotHardwareId
```

#### 方法 2: PowerShell による直接登録

```powershell
# Microsoft Graph を使用してデバイスを直接登録
function Register-AutopilotDevice {
    param(
        [string]$SerialNumber,
        [string]$HardwareHash,
        [string]$GroupTag = "Corporate",
        [string]$AssignedUser = ""
    )
    
    $deviceData = @{
        "@odata.type" = "#microsoft.graph.importedWindowsAutopilotDeviceIdentity"
        hardwareIdentifier = $HardwareHash
        serialNumber = $SerialNumber
        groupTag = $GroupTag
        assignedUserPrincipalName = $AssignedUser
    }
    
    try {
        $device = New-MgDeviceManagementImportedWindowsAutopilotDeviceIdentity -BodyParameter $deviceData
        Write-Host "✅ デバイス登録完了: $SerialNumber" -ForegroundColor Green
        return $device
    }
    catch {
        Write-Host "❌ デバイス登録エラー: $_" -ForegroundColor Red
    }
}

# 使用例
# Register-AutopilotDevice -SerialNumber "ABC12345" -HardwareHash "HASH_VALUE"
```

### ステップ 3: Autopilot プロファイル作成

```powershell
# User-driven プロファイルの作成
function New-AutopilotProfile {
    param(
        [string]$ProfileName = "Corporate Autopilot Profile",
        [string]$Description = "標準的な企業デバイス向けプロファイル"
    )
    
    $profileData = @{
        "@odata.type" = "#microsoft.graph.azureADWindowsAutopilotDeploymentProfile"
        displayName = $ProfileName
        description = $Description
        deviceNameTemplate = "CORP-%SERIAL%"
        deviceType = "windowsPc"
        enableWhiteGlove = $true
        
        # OOBE 設定
        outOfBoxExperienceSettings = @{
            hidePrivacySettings = $true
            hideEULA = $true
            userType = "standard"
            deviceUsageType = "singleUser"
            skipKeyboardSelectionPage = $true
            hideEscapeLink = $true
        }
        
        # 言語設定
        language = "ja-JP"
        
        # その他設定
        extractHardwareHash = $false
        hybridAzureADJoinSkipConnectivityCheck = $false
    }
    
    try {
        $profile = New-MgDeviceManagementWindowsAutopilotDeploymentProfile -BodyParameter $profileData
        Write-Host "✅ Autopilot プロファイル作成完了: $ProfileName" -ForegroundColor Green
        return $profile
    }
    catch {
        Write-Host "❌ プロファイル作成エラー: $_" -ForegroundColor Red
    }
}

# Self-deploying プロファイルの作成
function New-AutopilotSelfDeployingProfile {
    param(
        [string]$ProfileName = "Kiosk Autopilot Profile",
        [string]$Description = "キオスク・共有デバイス向けプロファイル"
    )
    
    $profileData = @{
        "@odata.type" = "#microsoft.graph.azureADWindowsAutopilotDeploymentProfile"
        displayName = $ProfileName
        description = $Description
        deviceNameTemplate = "KIOSK-%SERIAL%"
        deviceType = "windowsPc"
        
        # Self-deploying 設定
        outOfBoxExperienceSettings = @{
            hidePrivacySettings = $true
            hideEULA = $true
            userType = "administrator"
            deviceUsageType = "shared"
            skipKeyboardSelectionPage = $true
            hideEscapeLink = $true
        }
        
        language = "ja-JP"
        extractHardwareHash = $false
    }
    
    try {
        $profile = New-MgDeviceManagementWindowsAutopilotDeploymentProfile -BodyParameter $profileData
        Write-Host "✅ Self-deploying プロファイル作成完了: $ProfileName" -ForegroundColor Green
        return $profile
    }
    catch {
        Write-Host "❌ プロファイル作成エラー: $_" -ForegroundColor Red
    }
}

# プロファイル作成実行
$userDrivenProfile = New-AutopilotProfile
$selfDeployingProfile = New-AutopilotSelfDeployingProfile
```

### ステップ 4: プロファイル割り当て

```powershell
# プロファイルをデバイスグループに割り当て
function Set-AutopilotProfileAssignment {
    param(
        [string]$ProfileId,
        [string]$GroupId,
        [string]$AssignmentType = "include"
    )
    
    $assignmentData = @{
        "@odata.type" = "#microsoft.graph.windowsAutopilotDeploymentProfileAssignment"
        target = @{
            "@odata.type" = "#microsoft.graph.groupAssignmentTarget"
            groupId = $GroupId
        }
    }
    
    try {
        $assignment = New-MgDeviceManagementWindowsAutopilotDeploymentProfileAssignment -WindowsAutopilotDeploymentProfileId $ProfileId -BodyParameter $assignmentData
        Write-Host "✅ プロファイル割り当て完了" -ForegroundColor Green
        return $assignment
    }
    catch {
        Write-Host "❌ プロファイル割り当てエラー: $_" -ForegroundColor Red
    }
}

# デバイスグループの作成
$deviceGroup = @{
    displayName = "Autopilot Devices"
    description = "Windows Autopilot 管理デバイス"
    groupTypes = @()
    securityEnabled = $true
    mailEnabled = $false
}

$group = New-MgGroup -BodyParameter $deviceGroup
Set-AutopilotProfileAssignment -ProfileId $userDrivenProfile.Id -GroupId $group.Id
```

## 🎓 教育機関向け設定

### 児童生徒デバイス向けプロファイル

```powershell
# 教育機関向け Autopilot プロファイル
function New-EducationAutopilotProfile {
    param(
        [string]$SchoolName = "Example School",
        [ValidateSet("Student", "Teacher", "Lab")]
        [string]$DeviceType = "Student"
    )
    
    $profileName = "$SchoolName - $DeviceType Devices"
    $deviceNameTemplate = switch ($DeviceType) {
        "Student" { "STU-%SERIAL%" }
        "Teacher" { "TCH-%SERIAL%" }
        "Lab" { "LAB-%SERIAL%" }
    }
    
    $profileData = @{
        "@odata.type" = "#microsoft.graph.azureADWindowsAutopilotDeploymentProfile"
        displayName = $profileName
        description = "$SchoolName の$DeviceType 向けデバイス"
        deviceNameTemplate = $deviceNameTemplate
        deviceType = "windowsPc"
        enableWhiteGlove = $false
        
        # OOBE 設定
        outOfBoxExperienceSettings = @{
            hidePrivacySettings = $true
            hideEULA = $true
            userType = if ($DeviceType -eq "Teacher") { "administrator" } else { "standard" }
            deviceUsageType = if ($DeviceType -eq "Lab") { "shared" } else { "singleUser" }
            skipKeyboardSelectionPage = $true
            hideEscapeLink = $true
        }
        
        language = "ja-JP"
    }
    
    $profile = New-MgDeviceManagementWindowsAutopilotDeploymentProfile -BodyParameter $profileData
    Write-Host "✅ 教育機関プロファイル作成: $profileName" -ForegroundColor Green
    return $profile
}

# 学年別デバイスグループ作成
function New-EducationDeviceGroups {
    param(
        [string]$SchoolName = "Example School"
    )
    
    $grades = @("小学1年", "小学2年", "小学3年", "小学4年", "小学5年", "小学6年", "中学1年", "中学2年", "中学3年", "教員", "管理")
    $groups = @()
    
    foreach ($grade in $grades) {
        $groupData = @{
            displayName = "$SchoolName - $grade - デバイス"
            description = "$SchoolName の $grade 向けデバイスグループ"
            groupTypes = @()
            securityEnabled = $true
            mailEnabled = $false
        }
        
        $group = New-MgGroup -BodyParameter $groupData
        $groups += $group
        Write-Host "✅ グループ作成: $($group.DisplayName)" -ForegroundColor Green
    }
    
    return $groups
}

# 教育機関設定実行
$educationGroups = New-EducationDeviceGroups -SchoolName "サンプル小中学校"
$studentProfile = New-EducationAutopilotProfile -SchoolName "サンプル小中学校" -DeviceType "Student"
$teacherProfile = New-EducationAutopilotProfile -SchoolName "サンプル小中学校" -DeviceType "Teacher"
```

### 学習端末向け設定ポリシー

```powershell
# 教育機関向けデバイス設定プロファイル
function New-EducationDeviceConfiguration {
    param(
        [string]$ProfileName = "教育機関 - 学習端末設定"
    )
    
    $configData = @{
        "@odata.type" = "#microsoft.graph.windows10GeneralConfiguration"
        displayName = $ProfileName
        description = "学習に最適化されたデバイス設定"
        
        # ブラウザ設定
        internetSharingBlocked = $true
        bluetoothBlocked = $false
        cameraBlocked = $false
        
        # Microsoft Store 制限
        appsBlockWindowsStoreOriginatedApps = $false
        developerUnlockSetting = "blocked"
        
        # Windows Update 設定
        automaticUpdateMode = "autoInstallAndRebootAtMaintenanceTime"
        automaticUpdateInstallTime = "03:00:00.0000000"
        
        # デバイスロック設定
        passwordRequired = $true
        passwordMinimumLength = 6
        passwordMinutesOfInactivityBeforeScreenTimeout = 15
        
        # 学習環境向け設定
        settingsBlockGamingPage = $true
        settingsBlockPrivacyPage = $true
        settingsBlockAccountsPage = $false
        
        # ネットワーク設定
        wiFiBlocked = $false
        bluetoothAdvertisingBlocked = $true
    }
    
    $profile = New-MgDeviceManagementDeviceConfiguration -BodyParameter $configData
    Write-Host "✅ 教育機関向け設定プロファイル作成完了" -ForegroundColor Green
    return $profile
}

$educationConfig = New-EducationDeviceConfiguration
```

## 📊 デバイス管理と監視

### Autopilot デバイス状況確認

```powershell
# Autopilot デバイス一覧取得
function Get-AutopilotDeviceStatus {
    param(
        [string]$Filter = ""
    )
    
    Write-Host "📊 Autopilot デバイス状況確認" -ForegroundColor Cyan
    
    # 登録済みデバイス取得
    $devices = Get-MgDeviceManagementWindowsAutopilotDeviceIdentity
    
    if ($Filter) {
        $devices = $devices | Where-Object { 
            $_.SerialNumber -like "*$Filter*" -or 
            $_.Model -like "*$Filter*" 
        }
    }
    
    $summary = @{
        Total = $devices.Count
        Assigned = ($devices | Where-Object { $_.DeploymentProfileAssignmentStatus -eq "assigned" }).Count
        Pending = ($devices | Where-Object { $_.DeploymentProfileAssignmentStatus -eq "pending" }).Count
        Failed = ($devices | Where-Object { $_.DeploymentProfileAssignmentStatus -eq "failed" }).Count
    }
    
    Write-Host "`n📈 デバイス統計" -ForegroundColor Yellow
    Write-Host "   総数: $($summary.Total)" -ForegroundColor White
    Write-Host "   割り当て済み: $($summary.Assigned)" -ForegroundColor Green
    Write-Host "   保留中: $($summary.Pending)" -ForegroundColor Yellow
    Write-Host "   エラー: $($summary.Failed)" -ForegroundColor Red
    
    return $devices | Select-Object SerialNumber, Model, Manufacturer, DeploymentProfileAssignmentStatus, LastContactedDateTime
}

# デプロイメント履歴確認
function Get-AutopilotDeploymentHistory {
    param(
        [string]$SerialNumber
    )
    
    Write-Host "📋 デプロイメント履歴: $SerialNumber" -ForegroundColor Cyan
    
    # デバイス検索
    $device = Get-MgDeviceManagementWindowsAutopilotDeviceIdentity | Where-Object { $_.SerialNumber -eq $SerialNumber }
    
    if ($device) {
        # 関連するマネージドデバイス情報取得
        $managedDevice = Get-MgDeviceManagementManagedDevice | Where-Object { $_.SerialNumber -eq $SerialNumber }
        
        $history = @{
            SerialNumber = $device.SerialNumber
            Model = $device.Model
            Manufacturer = $device.Manufacturer
            EnrollmentStatus = $device.DeploymentProfileAssignmentStatus
            LastContact = $device.LastContactedDateTime
            ManagedDeviceId = $managedDevice.Id
            ComplianceState = $managedDevice.ComplianceState
            LastSyncDateTime = $managedDevice.LastSyncDateTime
        }
        
        return $history
    } else {
        Write-Host "❌ デバイスが見つかりません: $SerialNumber" -ForegroundColor Red
    }
}

# 例: デバイス状況確認
$deviceStatus = Get-AutopilotDeviceStatus
$deviceStatus | Format-Table -AutoSize
```

### エラー監視とアラート

```powershell
# Autopilot エラー監視
function Watch-AutopilotErrors {
    param(
        [int]$IntervalMinutes = 30
    )
    
    Write-Host "🔍 Autopilot エラー監視を開始..." -ForegroundColor Cyan
    
    while ($true) {
        $errorDevices = Get-MgDeviceManagementWindowsAutopilotDeviceIdentity | 
                       Where-Object { $_.DeploymentProfileAssignmentStatus -eq "failed" }
        
        if ($errorDevices.Count -gt 0) {
            Write-Host "⚠️  $($errorDevices.Count) 台のデバイスでエラーが発生しています" -ForegroundColor Red
            
            foreach ($device in $errorDevices) {
                Write-Host "   - $($device.SerialNumber) ($($device.Model))" -ForegroundColor Yellow
            }
            
            # アラート処理（メール送信等）
            Send-AutopilotAlert -ErrorDevices $errorDevices
        } else {
            Write-Host "✅ エラーは検出されませんでした" -ForegroundColor Green
        }
        
        Start-Sleep -Seconds ($IntervalMinutes * 60)
    }
}

# アラート送信機能
function Send-AutopilotAlert {
    param(
        $ErrorDevices
    )
    
    # Microsoft Graph を使用してメール送信
    $subject = "Windows Autopilot エラーアラート"
    $body = "以下のデバイスでAutopilotエラーが発生しています:`n`n"
    
    foreach ($device in $ErrorDevices) {
        $body += "• $($device.SerialNumber) - $($device.Model)`n"
    }
    
    # メール送信処理（実装は環境に応じて調整）
    Write-Host "📧 アラートメールを送信しました" -ForegroundColor Yellow
}
```

## 🔧 トラブルシューティング

### よくある問題と解決方法

```powershell
# Autopilot トラブルシューティング
function Start-AutopilotTroubleshooting {
    param(
        [string]$SerialNumber = ""
    )
    
    Write-Host "🔧 Autopilot トラブルシューティング開始" -ForegroundColor Cyan
    
    # 1. デバイス登録確認
    Write-Host "`n1️⃣  デバイス登録確認" -ForegroundColor Yellow
    if ($SerialNumber) {
        $device = Get-MgDeviceManagementWindowsAutopilotDeviceIdentity | Where-Object { $_.SerialNumber -eq $SerialNumber }
        if ($device) {
            Write-Host "✅ デバイスは登録されています: $SerialNumber" -ForegroundColor Green
        } else {
            Write-Host "❌ デバイスが登録されていません: $SerialNumber" -ForegroundColor Red
            Write-Host "   解決策: ハードウェア ID を再取得して登録してください" -ForegroundColor Yellow
        }
    }
    
    # 2. プロファイル割り当て確認
    Write-Host "`n2️⃣  プロファイル割り当て確認" -ForegroundColor Yellow
    $profiles = Get-MgDeviceManagementWindowsAutopilotDeploymentProfile
    if ($profiles.Count -gt 0) {
        Write-Host "✅ $($profiles.Count) 個のプロファイルが設定されています" -ForegroundColor Green
        foreach ($profile in $profiles) {
            Write-Host "   - $($profile.DisplayName)" -ForegroundColor White
        }
    } else {
        Write-Host "❌ Autopilot プロファイルが設定されていません" -ForegroundColor Red
        Write-Host "   解決策: Autopilot プロファイルを作成してください" -ForegroundColor Yellow
    }
    
    # 3. ライセンス確認
    Write-Host "`n3️⃣  ライセンス確認" -ForegroundColor Yellow
    $licenses = Get-MgSubscribedSku | Where-Object { 
        $_.SkuPartNumber -like "*INTUNE*" -or 
        $_.SkuPartNumber -like "*EMS*" -or
        $_.SkuPartNumber -like "*EDUCATION*"
    }
    
    if ($licenses.Count -gt 0) {
        Write-Host "✅ Intune ライセンスが利用可能です" -ForegroundColor Green
        foreach ($license in $licenses) {
            $available = $license.PrepaidUnits.Enabled - $license.ConsumedUnits
            Write-Host "   - $($license.SkuPartNumber): $available ライセンス利用可能" -ForegroundColor White
        }
    } else {
        Write-Host "❌ Intune ライセンスが見つかりません" -ForegroundColor Red
        Write-Host "   解決策: A3/A5またはBusiness Premiumライセンスを確認してください" -ForegroundColor Yellow
    }
    
    # 4. ネットワーク接続確認
    Write-Host "`n4️⃣  ネットワーク接続確認" -ForegroundColor Yellow
    $testEndpoints = @(
        "login.microsoftonline.com",
        "enrollment.manage.microsoft.com",
        "portal.manage.microsoft.com"
    )
    
    foreach ($endpoint in $testEndpoints) {
        try {
            $result = Test-NetConnection -ComputerName $endpoint -Port 443 -InformationLevel Quiet
            if ($result) {
                Write-Host "✅ $endpoint : 接続OK" -ForegroundColor Green
            } else {
                Write-Host "❌ $endpoint : 接続NG" -ForegroundColor Red
            }
        }
        catch {
            Write-Host "❌ $endpoint : テストエラー" -ForegroundColor Red
        }
    }
}

# デバイスリセット機能
function Reset-AutopilotDevice {
    param(
        [string]$DeviceId,
        [switch]$KeepUserData = $false
    )
    
    Write-Host "🔄 Autopilot デバイスリセット: $DeviceId" -ForegroundColor Cyan
    
    $resetParams = @{
        keepUserData = $KeepUserData.IsPresent
    }
    
    try {
        Invoke-MgWipeDeviceManagementManagedDevice -ManagedDeviceId $DeviceId -BodyParameter $resetParams
        Write-Host "✅ デバイスリセット要求を送信しました" -ForegroundColor Green
        Write-Host "   デバイスの再起動後、Autopilot プロセスが開始されます" -ForegroundColor Yellow
    }
    catch {
        Write-Host "❌ デバイスリセットエラー: $_" -ForegroundColor Red
    }
}

# ログ収集機能
function Get-AutopilotLogs {
    param(
        [string]$OutputPath = "C:\Temp\AutopilotLogs"
    )
    
    Write-Host "📋 Autopilot ログ収集" -ForegroundColor Cyan
    
    if (!(Test-Path $OutputPath)) {
        New-Item -Path $OutputPath -ItemType Directory -Force
    }
    
    # Windows イベントログ
    $events = Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Provisioning-Diagnostics-Provider/Admin'; StartTime=(Get-Date).AddDays(-7)}
    $events | Export-Csv "$OutputPath\AutopilotEvents.csv" -NoTypeInformation
    
    # デバイス情報
    Get-ComputerInfo | Out-File "$OutputPath\ComputerInfo.txt"
    
    # TPM 情報
    Get-Tpm | Out-File "$OutputPath\TPMInfo.txt"
    
    Write-Host "✅ ログを $OutputPath に保存しました" -ForegroundColor Green
}

# トラブルシューティング実行例
Start-AutopilotTroubleshooting
```

### エラーコード対応表

| エラーコード | 説明 | 対処法 |
|-------------|------|--------|
| **0x80180014** | デバイスが既に登録済み | デバイスの重複登録を確認・削除 |
| **0x801c03ea** | プロファイルが見つからない | プロファイル割り当てを確認 |
| **0x80180002** | デバイス認証エラー | ハードウェア ID を再取得 |
| **0x80180018** | ネットワーク接続エラー | ファイアウォール・プロキシ設定確認 |
| **0x80070002** | ファイルが見つからない | Windows Update 実行 |

## 🎯 ベストプラクティス

### 1. 段階的展開

```powershell
# 段階的展開の実装
function Start-AutopilotPhaseDeployment {
    param(
        [int]$Phase1Users = 10,
        [int]$Phase2Users = 50,
        [int]$WaitDays = 7
    )
    
    Write-Host "🎯 Autopilot 段階的展開開始" -ForegroundColor Cyan
    
    # フェーズ1: パイロットグループ
    Write-Host "`n📍 フェーズ1: パイロット展開 ($Phase1Users ユーザー)" -ForegroundColor Yellow
    $pilotGroup = New-MgGroup -BodyParameter @{
        displayName = "Autopilot Pilot Group"
        description = "Autopilot パイロットテストグループ"
        groupTypes = @()
        securityEnabled = $true
        mailEnabled = $false
    }
    
    # フェーズ2以降のグループも事前作成
    $phase2Group = New-MgGroup -BodyParameter @{
        displayName = "Autopilot Phase 2 Group"
        description = "Autopilot フェーズ2展開グループ"
        groupTypes = @()
        securityEnabled = $true
        mailEnabled = $false
    }
    
    Write-Host "✅ 段階的展開グループを作成しました" -ForegroundColor Green
    Write-Host "   $WaitDays 日後にフェーズ2展開を実行してください" -ForegroundColor Yellow
}
```

### 2. 監視とレポート

```powershell
# 週次レポート生成
function New-AutopilotWeeklyReport {
    param(
        [string]$ReportPath = "C:\Reports\Autopilot"
    )
    
    Write-Host "📊 Autopilot 週次レポート生成" -ForegroundColor Cyan
    
    if (!(Test-Path $ReportPath)) {
        New-Item -Path $ReportPath -ItemType Directory -Force
    }
    
    $reportDate = Get-Date -Format "yyyy-MM-dd"
    $reportFile = "$ReportPath\Autopilot-WeeklyReport-$reportDate.html"
    
    # データ収集
    $devices = Get-MgDeviceManagementManagedDevice
    $profiles = Get-MgDeviceManagementWindowsAutopilotDeploymentProfile
    $managedDevices = Get-MgDeviceManagementManagedDevice | Where-Object { $_.OperatingSystem -eq "Windows" }
    
    # HTML レポート生成
    $html = @"
    <!DOCTYPE html>
    <html>
    <head>
        <title>Windows Autopilot 週次レポート</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 20px; }
            table { border-collapse: collapse; width: 100%; }
            th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
            th { background-color: #f2f2f2; }
            .success { color: green; }
            .warning { color: orange; }
            .error { color: red; }
        </style>
    </head>
    <body>
        <h1>Windows Autopilot 週次レポート</h1>
        <p>レポート日時: $(Get-Date)</p>
        
        <h2>サマリー</h2>
        <table>
            <tr><th>項目</th><th>数量</th></tr>
            <tr><td>登録済みデバイス</td><td>$($devices.Count)</td></tr>
            <tr><td>プロファイル数</td><td>$($profiles.Count)</td></tr>
            <tr><td>管理対象デバイス</td><td>$($managedDevices.Count)</td></tr>
        </table>
        
        <h2>デバイス状況</h2>
        <table>
            <tr><th>シリアル番号</th><th>モデル</th><th>ステータス</th><th>最終接続</th></tr>
"@
    
    foreach ($device in $devices | Select-Object -First 20) {
        $statusClass = switch ($device.DeploymentProfileAssignmentStatus) {
            "assigned" { "success" }
            "pending" { "warning" }
            "failed" { "error" }
            default { "" }
        }
        
        $html += "<tr><td>$($device.SerialNumber)</td><td>$($device.Model)</td><td class='$statusClass'>$($device.DeploymentProfileAssignmentStatus)</td><td>$($device.LastContactedDateTime)</td></tr>"
    }
    
    $html += @"
        </table>
    </body>
    </html>
"@
    
    $html | Out-File $reportFile -Encoding UTF8
    Write-Host "✅ レポートを生成しました: $reportFile" -ForegroundColor Green
}
```

### 3. セキュリティ強化

```powershell
# セキュリティ設定の強化
function Set-AutopilotSecurityHardening {
    Write-Host "🔒 Autopilot セキュリティ強化設定" -ForegroundColor Cyan
    
    # BitLocker 自動暗号化設定
    $bitlockerPolicy = @{
        "@odata.type" = "#microsoft.graph.bitLockerSystemDrivePolicy"
        encryptionMethod = "aesCbc256"
        startupAuthenticationRequired = $true
        startupAuthenticationBlockWithoutTpmChip = $true
        minimumPinLength = 6
        recoveryOptions = @{
            blockDataRecoveryAgent = $false
            recoveryPasswordUsage = "required"
            recoveryKeyUsage = "required"
            enableRecoveryInformationSaveToStore = $true
            recoveryInformationToStore = "passwordAndKey"
            enableBitLockerAfterRecoveryInformationToStore = $true
        }
    }
    
    # 条件付きアクセス連携
    $conditionalAccessPolicy = @{
        displayName = "Autopilot デバイス - 条件付きアクセス"
        state = "enabled"
        conditions = @{
            users = @{
                includeUsers = @("all")
                excludeUsers = @()
                includeGroups = @()
                excludeGroups = @()  # 緊急アクセスアカウントを除外
            }
            
            applications = @{
                includeApplications = @("All")
                excludeApplications = @()
            }
            
            platforms = @{
                includePlatforms = @("windows", "iOS", "android", "macOS")
                excludePlatforms = @()
            }
            
            locations = @{
                includeLocations = @("All")
                excludeLocations = @()  # 信頼できるIPアドレスを除外
            }
            
            signInRiskLevels = @("medium", "high")
            userRiskLevels = @("medium", "high")
            
            deviceStates = @{
                includeStates = @("All")
                excludeStates = @("compliant", "domainJoined")
            }
        }
        
        grantControls = @{
            operator = "OR"
            builtInControls = @("mfa", "compliantDevice")
            customAuthenticationFactors = @()
            termsOfUse = @()
        }
        
        sessionControls = @{
            applicationEnforcedRestrictions = $null
            cloudAppSecurity = @{
                cloudAppSecurityType = "monitorOnly"
                isEnabled = $true
            }
            signInFrequency = @{
                value = 4
                type = "hours"
                isEnabled = $true
            }
            persistentBrowser = @{
                mode = "never"
                isEnabled = $true
            }
        }
    }
    
    try {
        $policy = New-MgIdentityConditionalAccessPolicy -BodyParameter $riskBasedPolicy
        Write-Host "✅ リスクベース条件付きアクセスポリシーを作成しました" -ForegroundColor Green
    } catch {
        Write-Host "❌ 条件付きアクセス設定エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

## 📊 高度な監視とレポート

### カスタムダッシュボードの作成

```powershell
# PowerBI 用データエクスポート
function Export-IntuneDataForPowerBI {
    param(
        [string]$ExportPath = "C:\Reports\PowerBI"
    )
    
    Write-Host "📊 PowerBI 用データエクスポート" -ForegroundColor Cyan
    
    if (!(Test-Path $ExportPath)) {
        New-Item -Path $ExportPath -ItemType Directory -Force
    }
    
    $exportDate = Get-Date -Format "yyyy-MM-dd"
    
    try {
        # デバイス情報
        Write-Host "📱 デバイス情報をエクスポート中..." -ForegroundColor Yellow
        $devices = Get-MgDeviceManagementManagedDevice | Select-Object @(
            'Id', 'DeviceName', 'OperatingSystem', 'OSVersion', 'DeviceType',
            'ComplianceState', 'ManagementState', 'EnrolledDateTime', 'LastSyncDateTime',
            'UserDisplayName', 'UserPrincipalName', 'SerialNumber', 'Manufacturer', 'Model',
            'TotalStorageSpaceInBytes', 'FreeStorageSpaceInBytes', 'PartnerReportedThreatState'
        )
        $devices | Export-Csv "$ExportPath\Devices_$exportDate.csv" -NoTypeInformation -Encoding UTF8
        
        # コンプライアンス詳細
        Write-Host "📋 コンプライアンス情報をエクスポート中..." -ForegroundColor Yellow
        $compliance = Get-MgDeviceManagementDeviceComplianceDeviceStatus | Select-Object @(
            'DeviceDisplayName', 'UserName', 'DeviceModel', 'Platform', 'ComplianceGracePeriodExpirationDateTime',
            'Status', 'LastReportedDateTime', 'UserPrincipalName'
        )
        $compliance | Export-Csv "$ExportPath\Compliance_$exportDate.csv" -NoTypeInformation -Encoding UTF8
        
        # アプリケーション状況
        Write-Host "📱 アプリケーション情報をエクスポート中..." -ForegroundColor Yellow
        $apps = Get-MgDeviceManagementMobileApp | Select-Object @(
            'Id', 'DisplayName', 'Description', 'Publisher', 'CreatedDateTime', 'LastModifiedDateTime'
        )
        $apps | Export-Csv "$ExportPath\Applications_$exportDate.csv" -NoTypeInformation -Encoding UTF8
        
        # ユーザー情報
        Write-Host "👥 ユーザー情報をエクスポート中..." -ForegroundColor Yellow
        $users = Get-MgDeviceManagementManagedDevice | Group-Object UserPrincipalName | ForEach-Object {
            [PSCustomObject]@{
                UserPrincipalName = $_.Name
                DeviceCount = $_.Count
                LastActiveDevice = ($_.Group | Sort-Object LastSyncDateTime -Descending | Select-Object -First 1).LastSyncDateTime
                ComplianceStatus = ($_.Group | Where-Object {$_.ComplianceState -eq "compliant"}).Count
                NonComplianceStatus = ($_.Group | Where-Object {$_.ComplianceState -eq "noncompliant"}).Count
            }
        }
        $users | Export-Csv "$ExportPath\Users_$exportDate.csv" -NoTypeInformation -Encoding UTF8
        
        # サマリーレポート
        $summary = [PSCustomObject]@{
            ExportDate = Get-Date
            TenantId = (Get-MgContext).TenantId
            DeviceConfigurationCount = $deviceConfigs.Count
            CompliancePolicyCount = $compliancePolicies.Count
            ApplicationCount = $applications.Count
            AutopilotProfileCount = $autopilotProfiles.Count
        }
        $summary | Export-Csv "$ExportPath\Summary_$exportDate.csv" -NoTypeInformation -Encoding UTF8
        
        Write-Host "✅ データエクスポート完了: $ExportPath" -ForegroundColor Green
        
        # PowerBI テンプレートファイルの作成
        $pbiTemplate = @"
{
    "version": "1.0",
    "queries": [
        {
            "name": "Devices",
            "source": "$ExportPath\\Devices_$exportDate.csv"
        },
        {
            "name": "Compliance", 
            "source": "$ExportPath\\Compliance_$exportDate.csv"
        },
        {
            "name": "Applications",
            "source": "$ExportPath\\Applications_$exportDate.csv"
        },
        {
            "name": "Users",
            "source": "$ExportPath\\Users_$exportDate.csv"
        },
        {
            "name": "Summary",
            "source": "$ExportPath\\Summary_$exportDate.csv"
        }
    ]
}
"@
        $pbiTemplate | Out-File "$ExportPath\PowerBI_Template.json" -Encoding UTF8
        
    } catch {
        Write-Host "❌ データエクスポートエラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# リアルタイム監視アラート
function Start-IntuneMonitoring {
    param(
        [int]$IntervalMinutes = 15,
        [string]$WebhookUrl = "",
        [string[]]$AlertTypes = @("NonCompliance", "FailedSync", "SecurityThreat")
    )
    
    Write-Host "🔍 Intune リアルタイム監視開始" -ForegroundColor Cyan
    Write-Host "監視間隔: $IntervalMinutes 分" -ForegroundColor Yellow
    
    $lastCheck = Get-Date
    
    while ($true) {
        try {
            Write-Host "`n⏰ 監視チェック実行: $(Get-Date)" -ForegroundColor Blue
            
            if ("NonCompliance" -in $AlertTypes) {
                # 新しい非準拠デバイスの検出
                $newNonCompliant = Get-MgDeviceManagementDeviceComplianceDeviceStatus | Where-Object {
                    $_.Status -eq "noncompliant" -and $_.LastReportedDateTime -gt $lastCheck
                }
                
                if ($newNonCompliant.Count -gt 0) {
                    $alertMessage = "🚨 新しい非準拠デバイス検出: $($newNonCompliant.Count) 台"
                    Write-Host $alertMessage -ForegroundColor Red
                    
                    if ($WebhookUrl) {
                        $webhook = @{
                            text = $alertMessage
                            attachments = @(
                                @{
                                    color = "danger"
                                    fields = @(
                                        $newNonCompliant | ForEach-Object {
                                            @{
                                                title = $_.DeviceDisplayName
                                                value = "ユーザー: $($_.UserName)`n最終レポート: $($_.LastReportedDateTime)"
                                                short = $true
                                            }
                                        }
                                    )
                                }
                            )
                        } | ConvertTo-Json -Depth 10
                        
                        Invoke-RestMethod -Uri $WebhookUrl -Method Post -Body $webhook -ContentType "application/json"
                    }
                }
            }
            
            if ("FailedSync" -in $AlertTypes) {
                # 同期失敗デバイスの検出
                $failedSync = Get-MgDeviceManagementManagedDevice | Where-Object {
                    $_.LastSyncDateTime -lt (Get-Date).AddHours(-24)
                }
                
                if ($failedSync.Count -gt 0) {
                    Write-Host "⚠️  24時間以上同期していないデバイス: $($failedSync.Count) 台" -ForegroundColor Yellow
                }
            }
            
            if ("SecurityThreat" -in $AlertTypes) {
                # セキュリティ脅威の検出
                $threats = Get-MgDeviceManagementManagedDevice | Where-Object {
                    $_.PartnerReportedThreatState -ne "cleared" -and $_.PartnerReportedThreatState -ne "unknown"
                }
                
                if ($threats.Count -gt 0) {
                    $threatAlert = "🛡️ セキュリティ脅威検出: $($threats.Count) 台"
                    Write-Host $threatAlert -ForegroundColor Red
                }
            }
            
            $lastCheck = Get-Date
            Start-Sleep -Seconds ($IntervalMinutes * 60)
            
        } catch {
            Write-Host "❌ 監視エラー: $($_.Exception.Message)" -ForegroundColor Red
            Start-Sleep -Seconds 300  # エラー時は5分待機
        }
    }
}
```

## 🔄 災害復旧とバックアップ

### 設定のバックアップと復元

```powershell
# Intune 設定の完全バックアップ
function Backup-IntuneConfiguration {
    param(
        [string]$BackupPath = "C:\Backups\Intune\$(Get-Date -Format 'yyyy-MM-dd')"
    )
    
    Write-Host "💾 Intune 設定バックアップ開始" -ForegroundColor Cyan
    
    if (!(Test-Path $BackupPath)) {
        New-Item -Path $BackupPath -ItemType Directory -Force
    }
    
    try {
        # デバイス構成プロファイル
        Write-Host "📱 デバイス構成プロファイルをバックアップ中..." -ForegroundColor Yellow
        $deviceConfigs = Get-MgDeviceManagementDeviceConfiguration
        $deviceConfigs | ConvertTo-Json -Depth 10 | Out-File "$BackupPath\DeviceConfigurations.json" -Encoding UTF8
        
        Write-Host "✅ Intune設定の包括的な管理ガイドが完成しました" -ForegroundColor Green
        
    } catch {
        Write-Host "❌ エラーが発生しました: $($_.Exception.Message)" -ForegroundColor Red
    }
}
```