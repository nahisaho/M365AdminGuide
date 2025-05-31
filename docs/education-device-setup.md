# 教育機関向けデバイス初期設定ガイド

## 📋 概要

このガイドでは、教育機関でのWindowsおよびiOSデバイスの初期セットアップについて、包括的に説明します。Provisioning Package、Windows Autopilot、Microsoft Intuneを活用した効率的なデバイス展開・管理手法を学習できます。

## 🎯 対象読者

- 教育機関のIT管理者
- デバイス展開を担当する方
- Microsoft 365 Education環境の管理者

## 📚 前提条件

- Microsoft 365 Education ライセンス
- Microsoft Intune ライセンス
- Azure AD Premium P1 以上
- 管理者権限を持つアカウント

---

## 🖥️ Windows デバイス初期設定

### Windows Autopilot 概要

Windows Autopilotは、新しいWindowsデバイスを簡単にセットアップ、事前構成、および再利用するためのサービスです。

#### 主な利点
- **ゼロタッチ展開**: 管理者の手作業を最小限に抑制
- **ブランド化**: 学校のロゴとカスタム設定で統一感のある体験
- **セキュリティ**: デバイスが自動的に組織のセキュリティポリシーに準拠

### Windows Autopilot セットアップ手順

#### 1. デバイス登録

```powershell
# デバイスハードウェアハッシュの取得
Get-WindowsAutoPilotInfo -OutputFile C:\temp\AutopilotHWID.csv

# 複数デバイスの場合
Get-WindowsAutoPilotInfo -GroupTag "Education-Students" -OutputFile C:\temp\Students-HWID.csv
```

#### 2. Microsoft Endpoint Manager Admin Center での設定

1. **Microsoft Endpoint Manager Admin Center** にアクセス
2. **デバイス** > **Windows** > **Windows の登録** > **デバイス**
3. **インポート** をクリックしてCSVファイルをアップロード
4. グループタグを設定（例：Education-Students、Education-Staff）

#### 3. Autopilot プロファイルの作成

```json
{
  "displayName": "Education Student Profile",
  "description": "学生向けAutopilotプロファイル",
  "outOfBoxExperienceSettings": {
    "hidePrivacySettings": true,
    "hideEULA": true,
    "userType": "standard",
    "deviceUsageType": "shared",
    "skipKeyboardSelectionPage": true,
    "hideEscapeLink": true
  },
  "deviceNameTemplate": "EDU-STU-%SERIAL%",
  "enableWhiteGlove": false,
  "extractHardwareHash": false,
  "deviceType": "windowsPc"
}
```

#### 4. プロファイルの割り当て

1. **デバイス** > **Windows** > **Windows の登録** > **プロファイル**
2. 作成したプロファイルを選択
3. **割り当て** タブで対象グループを選択

### Provisioning Package作成

Windows構成デザイナーを使用してプロビジョニングパッケージを作成します。

#### 教育機関向け推奨設定

```xml
<!-- 基本設定例 -->
<Settings>
  <Runtime>
    <!-- デバイス名設定 -->
    <ComputerName>EDU-%SERIAL%</ComputerName>
    
    <!-- ローカル管理者アカウント -->
    <Accounts>
      <Users>
        <User wcm:action="add">
          <Name>SchoolAdmin</Name>
          <Password>SecurePassword123!</Password>
          <Group>Administrators</Group>
        </User>
      </Users>
    </Accounts>
    
    <!-- WiFi設定 -->
    <WiFi>
      <Profile>
        <Name>School-WiFi</Name>
        <SSIDConfig>
          <SSID>SchoolNetwork</SSID>
        </SSIDConfig>
        <Security>
          <Authentication>WPA2PSK</Authentication>
          <Encryption>AES</Encryption>
          <KeyMaterial>WiFiPassword123!</KeyMaterial>
        </Security>
      </Profile>
    </WiFi>
    
    <!-- 証明書のインストール -->
    <Certificates>
      <ClientCertificates>
        <ClientCertificate>
          <CertificateStore>Root</CertificateStore>
          <CertificatePath>C:\temp\school-root-ca.cer</CertificatePath>
        </ClientCertificate>
      </ClientCertificates>
    </Certificates>
  </Runtime>
</Settings>
```

#### プロビジョニングパッケージ作成手順

1. **Windows Configuration Designer** を起動
2. **新しいプロジェクト** > **プロビジョニングパッケージ**
3. **共通のすべてのWindowsデスクトップエディション** を選択
4. 必要な設定を構成：
   - **アカウント管理** > ローカル管理者作成
   - **ネットワーク接続** > WiFiプロファイル
   - **証明書** > ルート証明書のインストール
   - **ポリシー** > セキュリティ設定

5. **エクスポート** > **プロビジョニングパッケージ**

### Intune による Windows デバイス管理

#### デバイスコンプライアンスポリシー

```json
{
  "@odata.type": "#microsoft.graph.windowsCompliancePolicy",
  "displayName": "Education Windows Compliance",
  "description": "教育機関Windows端末コンプライアンスポリシー",
  "passwordRequired": true,
  "passwordMinimumLength": 8,
  "passwordRequiredType": "alphanumeric",
  "passwordMinutesOfInactivityBeforeLock": 15,
  "passwordExpirationDays": 90,
  "passwordPreviousPasswordBlockCount": 5,
  "passwordRequiredToUnlockFromIdle": true,
  "requireHealthyDeviceReport": true,
  "osMinimumVersion": "10.0.19041",
  "osMaximumVersion": null,
  "mobileOsMinimumVersion": null,
  "mobileOsMaximumVersion": null,
  "earlyLaunchAntiMalwareDriverEnabled": true,
  "bitLockerEnabled": true,
  "secureBootEnabled": true,
  "codeIntegrityEnabled": true,
  "storageRequireEncryption": true
}
```

#### 構成プロファイル（デバイス制限）

```json
{
  "@odata.type": "#microsoft.graph.windowsDeviceRestrictionConfiguration",
  "displayName": "Education Device Restrictions",
  "description": "教育機関デバイス制限ポリシー",
  "defenderBlockEndUserAccess": true,
  "defenderRequireRealTimeMonitoring": true,
  "defenderRequireBehaviorMonitoring": true,
  "defenderRequireNetworkInspectionSystem": true,
  "defenderScanDownloads": true,
  "defenderScanScriptsLoadedInInternetExplorer": true,
  "defenderSignatureUpdateIntervalInHours": 4,
  "defenderMonitorFileActivity": "enable",
  "defenderDaysBeforeDeletingQuarantinedMalware": 30,
  "defenderScanMaxCpu": 50,
  "defenderScanArchiveFiles": true,
  "defenderScanIncomingMail": true,
  "defenderScanRemovableDrivesDuringFullScan": true,
  "defenderScanMappedNetworkDrivesDuringFullScan": false,
  "defenderScanNetworkFiles": false,
  "defenderRequireCloudProtection": true,
  "defenderCloudBlockLevel": "high",
  "defenderPromptForSampleSubmission": "sendSafeSamplesAutomatically",
  "defenderScheduledQuickScanTime": "02:00:00.0000000",
  "defenderScanType": "userDefined",
  "defenderSystemScanSchedule": "userDefined",
  "defenderScheduledScanTime": "02:00:00.0000000",
  "defenderDetectedMalwareActions": {
    "lowSeverity": "quarantine",
    "moderateSeverity": "quarantine", 
    "highSeverity": "quarantine",
    "severeSeverity": "quarantine"
  }
}
```

---

## 📱 iOS デバイス初期設定

### Apple School Manager連携

#### 前提条件
- Apple School Manager アカウント
- DEP（Device Enrollment Program）への登録
- Apple Push Notification service証明書

#### セットアップ手順

1. **Apple School Manager** でのMDMサーバー設定
2. **Microsoft Endpoint Manager** でのApple証明書設定
3. デバイスの自動登録設定

#### Apple School Manager での設定

```bash
# Apple Configurator 2 での設定例
# 設定プロファイル作成
defaults write ~/Desktop/ios-config.plist PayloadDisplayName "Education iOS Config"
defaults write ~/Desktop/ios-config.plist PayloadDescription "教育機関向けiOS設定"
defaults write ~/Desktop/ios-config.plist PayloadIdentifier "com.school.ios.config"
defaults write ~/Desktop/ios-config.plist PayloadType "Configuration"
defaults write ~/Desktop/ios-config.plist PayloadUUID "$(uuidgen)"
defaults write ~/Desktop/ios-config.plist PayloadVersion 1
```

### iOS デバイス登録プロファイル

```json
{
  "@odata.type": "#microsoft.graph.depIOSEnrollmentProfile",
  "displayName": "Education iOS Enrollment",
  "description": "教育機関向けiOS登録プロファイル",
  "requiresUserAuthentication": true,
  "configurationEndpointUrl": "https://enrollment.manage.microsoft.com/enrollmentserver/discovery.svc",
  "enableAuthenticationViaCompanyPortal": true,
  "requireCompanyPortalOnSetupAssistantEnrolledDevices": true,
  "isDefault": false,
  "supervisedModeEnabled": true,
  "supportDepartment": "School IT Support",
  "supportPhoneNumber": "+81-3-1234-5678",
  "passCodeDisabled": false,
  "profileRemovalDisabled": true,
  "activationLockDisabled": false,
  "enableSingleAppEnrollmentMode": false,
  "deviceNameTemplate": "School-iPad-{{SERIAL}}",
  "skipSetupItems": [
    "appleID",
    "biometric",
    "deviceToDeviceMigration",
    "displayTone",
    "privacy",
    "restore",
    "screenTime",
    "siri",
    "softwareUpdate",
    "tos"
  ],
  "iTunesPairingMode": "disallow",
  "managementCertificates": [],
  "restoreBlocked": true,
  "restoreFromAndroidBlocked": true,
  "appleIdDisabled": true,
  "termsAndConditionsDisabled": true,
  "touchIdDisabled": false,
  "applePayDisabled": true,
  "siriDisabled": false,
  "diagnosticsDisabled": false,
  "displayToneSetupDisabled": true,
  "privacyPaneDisabled": true,
  "screenTimeScreenDisabled": true,
  "deviceToDeviceMigrationDisabled": true,
  "welcomeScreenDisabled": false,
  "passcodeLockGracePeriodInSeconds": 0,
  "carrierActivationUrl": "",
  "userlessSharedAadModeEnabled": false
}
```

### iOS コンプライアンスポリシー

```json
{
  "@odata.type": "#microsoft.graph.iosCompliancePolicy",
  "displayName": "Education iOS Compliance",
  "description": "教育機関iOS端末コンプライアンスポリシー",
  "passcodeBlockSimple": true,
  "passcodeExpirationDays": 90,
  "passcodeMinimumLength": 6,
  "passcodeMinutesOfInactivityBeforeLock": 15,
  "passcodeMinutesOfInactivityBeforeScreenTimeout": 5,
  "passcodePreviousPasscodeBlockCount": 5,
  "passcodeMinimumCharacterSetCount": 3,
  "passcodeRequiredType": "alphanumeric",
  "passcodeRequired": true,
  "osMinimumVersion": "14.0",
  "osMaximumVersion": null,
  "systemIntegrityProtectionEnabled": true,
  "deviceThreatProtectionEnabled": false,
  "deviceThreatProtectionRequiredSecurityLevel": "unavailable",
  "advancedThreatProtectionRequiredSecurityLevel": "unavailable",
  "securityBlockJailbrokenDevices": true,
  "managedEmailProfileRequired": false,
  "restrictedApps": [
    {
      "name": "Social Media Apps",
      "publisher": "Various",
      "appStoreUrl": "",
      "appId": "com.facebook.Facebook"
    }
  ]
}
```

### iOS デバイス制限プロファイル

```json
{
  "@odata.type": "#microsoft.graph.iosDeviceRestrictionsConfiguration",
  "displayName": "Education iOS Device Restrictions",
  "description": "教育機関iOS端末制限ポリシー",
  "appsBlockClipboardSharing": true,
  "appsBlockYouTube": true,
  "appsBlockiTunes": true,
  "appsBlockRadio": true,
  "appStoreBlockAutomaticDownloads": true,
  "appStoreBlocked": false,
  "appStoreBlockInAppPurchases": true,
  "appStoreBlockUIAppInstallation": true,
  "appStoreRequirePassword": true,
  "bluetoothBlockModification": true,
  "cameraBlocked": false,
  "cellularBlockDataRoaming": true,
  "cellularBlockGlobalBackgroundFetchWhileRoaming": true,
  "cellularBlockPerAppDataModification": true,
  "cellularBlockVoiceRoaming": true,
  "certificatesBlockUntrustedTlsCertificates": true,
  "classroomAppBlockRemoteScreenObservation": false,
  "classroomAppForceUnpromptedScreenObservation": true,
  "compliantAppsList": [
    {
      "name": "Microsoft Teams",
      "publisher": "Microsoft Corporation",
      "appStoreUrl": "https://apps.apple.com/app/microsoft-teams/id1113153706",
      "appId": "com.microsoft.skype.teams"
    },
    {
      "name": "Microsoft Word",
      "publisher": "Microsoft Corporation", 
      "appStoreUrl": "https://apps.apple.com/app/microsoft-word/id586447913",
      "appId": "com.microsoft.Office.Word"
    },
    {
      "name": "Microsoft PowerPoint",
      "publisher": "Microsoft Corporation",
      "appStoreUrl": "https://apps.apple.com/app/microsoft-powerpoint/id586449534", 
      "appId": "com.microsoft.Office.PowerPoint"
    }
  ],
  "compliantAppListType": "appsInListCompliant",
  "configurationProfileBlockChanges": true,
  "definitionLookupBlocked": false,
  "deviceBlockEnableRestrictions": true,
  "deviceBlockEraseContentAndSettings": true,
  "deviceBlockNameModification": true,
  "diagnosticDataSubmissionBlocked": false,
  "documentsBlockManagedDocumentsInUnmanagedApps": true,
  "documentsBlockUnmanagedDocumentsInManagedApps": true,
  "emailInDomainSuffixes": ["school.edu.jp"],
  "enterpriseAppBlockTrust": true,
  "enterpriseAppBlockTrustModification": true,
  "faceTimeBlocked": true,
  "findMyFriendsBlocked": true,
  "gamingBlockGameCenterFriends": true,
  "gamingBlockMultiplayer": true,
  "gameCenterBlocked": true,
  "hostPairingBlocked": true,
  "iBooksStoreBlocked": false,
  "iBooksStoreBlockErotica": true,
  "iCloudBlockActivityContinuation": true,
  "iCloudBlockBackup": false,
  "iCloudBlockDocumentSync": false,
  "iCloudBlockManagedAppsSync": true,
  "iCloudBlockPhotoLibrary": true,
  "iCloudBlockPhotoStreamSync": true,
  "iCloudBlockSharedPhotoStream": true,
  "iCloudRequireEncryptedBackup": true,
  "iTunesBlockExplicitContent": true,
  "iTunesBlockMusicService": false,
  "iTunesBlockRadio": true,
  "keyboardBlockAutoCorrect": false,
  "keyboardBlockDictation": true,
  "keyboardBlockPredictive": false,
  "keyboardBlockShortcuts": false,
  "keyboardBlockSpellCheck": false,
  "kioskModeAllowAssistiveSpeak": false,
  "kioskModeAllowAssistiveTouchSettings": false,
  "kioskModeAllowAutoLock": true,
  "kioskModeAllowColorInversionSettings": false,
  "kioskModeAllowRingerSwitch": false,
  "kioskModeAllowScreenRotation": true,
  "kioskModeAllowSleepButton": true,
  "kioskModeAllowTouchscreen": true,
  "kioskModeAllowVoiceOverSettings": false,
  "kioskModeAllowVolumeButtons": true,
  "kioskModeAllowZoomSettings": false,
  "lockScreenBlockControlCenter": true,
  "lockScreenBlockNotificationView": false,
  "lockScreenBlockPassbook": true,
  "lockScreenBlockTodayView": true,
  "mediaContentRatingAustralia": null,
  "mediaContentRatingCanada": null,
  "mediaContentRatingFrance": null,
  "mediaContentRatingGermany": null,
  "mediaContentRatingIreland": null,
  "mediaContentRatingJapan": {
    "movieRating": "allBlocked",
    "tvRating": "allBlocked"
  },
  "mediaContentRatingNewZealand": null,
  "mediaContentRatingUnitedKingdom": null,
  "mediaContentRatingUnitedStates": null,
  "networkUsageRules": [],
  "newsBlocked": false,
  "nfcBlocked": false,
  "notificationsBlockSettingsModification": true,
  "passcodeBlockFingerprintUnlock": false,
  "passcodeBlockFingerprintModification": true,
  "passcodeBlockModification": true,
  "passcodeBlockSimple": true,
  "passcodeExpirationDays": 90,
  "passcodeMinimumLength": 6,
  "passcodeMinutesOfInactivityBeforeLock": 15,
  "passcodeMinutesOfInactivityBeforeScreenTimeout": 5,
  "passcodePreviousPasscodeBlockCount": 5,
  "passcodeSignInFailureCountBeforeWipe": 10,
  "passcodeRequiredType": "alphanumeric",
  "passcodeRequired": true,
  "podcastsBlocked": false,
  "safariBlockAutofill": true,
  "safariBlockJavaScript": false,
  "safariBlockPopups": true,
  "safariBlocked": false,
  "safariCookieSettings": "blockAlways",
  "safariManagedDomains": ["school.edu.jp", "*.microsoft.com"],
  "safariPasswordAutoFillDomains": ["school.edu.jp"],
  "safariRequireFraudWarning": true,
  "screenCaptureBlocked": false,
  "siriBlocked": false,
  "siriBlockedWhenLocked": true,
  "siriBlockUserGeneratedContent": true,
  "siriRequireProfanityFilter": true,
  "spotlightBlockInternetResults": true,
  "voiceDialingBlocked": false,
  "wallpaperBlockModification": true,
  "wiFiConnectOnlyToConfiguredNetworks": true
}
```

---

## 🔧 最低限必要なIntune設定

### 1. 基本的なポリシー設定

#### Microsoft Intune 基本構成

```powershell
# Microsoft Graph PowerShell を使用した設定例

# 接続
Connect-MgGraph -Scopes "DeviceManagementConfiguration.ReadWrite.All"

# デバイスコンプライアンスポリシーの作成
$compliancePolicy = @{
    "@odata.type" = "#microsoft.graph.windowsCompliancePolicy"
    displayName = "Education Basic Compliance"
    description = "教育機関基本コンプライアンスポリシー"
    passwordRequired = $true
    passwordMinimumLength = 8
    requireHealthyDeviceReport = $true
    osMinimumVersion = "10.0.19041"
}

New-MgDeviceManagementDeviceCompliancePolicy -BodyParameter $compliancePolicy
```

### 2. 条件付きアクセスポリシー

```json
{
  "displayName": "Education Device Compliance",
  "state": "enabled",
  "conditions": {
    "applications": {
      "includeApplications": ["All"]
    },
    "users": {
      "includeGroups": ["All Students", "All Staff"]
    },
    "devices": {
      "includeDevices": ["All"]
    },
    "platforms": {
      "includePlatforms": ["windows", "iOS"]
    }
  },
  "grantControls": {
    "operator": "AND",
    "builtInControls": ["compliantDevice", "domainJoinedDevice"]
  },
  "sessionControls": null
}
```

### 3. アプリケーション管理

#### 必須アプリケーション一覧

```json
{
  "requiredApps": [
    {
      "name": "Microsoft Teams",
      "platform": "both",
      "assignmentType": "required",
      "description": "授業・会議用"
    },
    {
      "name": "Microsoft Office Apps",
      "platform": "both", 
      "assignmentType": "required",
      "description": "文書作成・編集"
    },
    {
      "name": "Microsoft Authenticator",
      "platform": "both",
      "assignmentType": "required", 
      "description": "多要素認証"
    },
    {
      "name": "Company Portal",
      "platform": "both",
      "assignmentType": "required",
      "description": "デバイス管理"
    }
  ],
  "blockedApps": [
    {
      "name": "Social Media Apps",
      "platform": "both",
      "reason": "教育環境での集中阻害防止"
    },
    {
      "name": "Gaming Apps", 
      "platform": "both",
      "reason": "授業時間中の不適切使用防止"
    }
  ]
}
```

### 4. セキュリティ基準設定

#### Windows セキュリティベースライン

```json
{
  "displayName": "Education Security Baseline",
  "description": "教育機関向けセキュリティベースライン設定",
  "settings": [
    {
      "settingName": "DefenderAvStatus",
      "value": "enabled"
    },
    {
      "settingName": "FirewallProfileDomain",
      "value": "enabled"
    },
    {
      "settingName": "FirewallProfilePrivate", 
      "value": "enabled"
    },
    {
      "settingName": "FirewallProfilePublic",
      "value": "enabled"
    },
    {
      "settingName": "SmartScreenEnabled",
      "value": "enabled"
    },
    {
      "settingName": "DefenderCloudProtection",
      "value": "enabled"
    },
    {
      "settingName": "DefenderRealTimeProtection",
      "value": "enabled"
    }
  ]
}
```

---

## 📊 展開チェックリスト

### 事前準備

- [ ] Microsoft 365 Education ライセンス確認
- [ ] Azure AD Premium ライセンス確認  
- [ ] Intune ライセンス確認
- [ ] ネットワーク要件確認
- [ ] Apple School Manager アカウント設定（iOS デバイスの場合）
- [ ] デバイス調達・受領確認

### Windows Autopilot 展開

- [ ] ハードウェアハッシュ収集
- [ ] Autopilot プロファイル作成
- [ ] デバイス登録
- [ ] グループタグ設定
- [ ] プロファイル割り当て
- [ ] テストデバイスでの動作確認

### iOS デバイス展開

- [ ] Apple School Manager 設定
- [ ] DEP トークン設定
- [ ] APN 証明書設定
- [ ] 登録プロファイル作成
- [ ] デバイス割り当て
- [ ] Supervised モード確認
- [ ] テストデバイスでの動作確認

### Intune ポリシー設定

- [ ] コンプライアンスポリシー作成
- [ ] デバイス制限ポリシー作成
- [ ] アプリケーション管理ポリシー作成
- [ ] 条件付きアクセスポリシー作成
- [ ] セキュリティベースライン適用
- [ ] ポリシー割り当て確認

### 運用開始後

- [ ] デバイス登録状況監視
- [ ] コンプライアンス状況確認
- [ ] アプリケーション展開状況確認
- [ ] ユーザーサポート体制確立
- [ ] トラブルシューティング手順確認
- [ ] 定期的なポリシー見直し

---

## 🔍 トラブルシューティング

### よくある問題と解決方法

#### Windows Autopilot 関連

**問題**: デバイスがAutopilot プロファイルを認識しない

**解決方法**:
1. ハードウェアハッシュが正しく登録されているか確認
2. デバイスがインターネットに接続されているか確認
3. プロファイルの割り当てが正しいか確認

```powershell
# Autopilot 状態確認
Get-AutopilotDiagnostics

# ログ確認
Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin"
```

#### iOS 登録関連

**問題**: iOS デバイスが登録されない

**解決方法**:
1. DEP トークンの有効期限確認
2. APN 証明書の有効性確認
3. デバイスのシリアル番号がDEP に登録されているか確認

```bash
# iOS デバイス状態確認
# 設定 > 一般 > VPN とデバイス管理 で確認
```

#### コンプライアンス問題

**問題**: デバイスがコンプライアンス非準拠となる

**解決方法**:
1. 具体的な非準拠項目を確認
2. ポリシー設定の妥当性確認
3. デバイス固有の制限事項確認

### ログ収集手順

#### Windows

```powershell
# MDM 診断情報収集
mdmdiagnosticstool.exe -area DeviceEnrollment -cab C:\temp\EnrollmentLogs.cab

# Autopilot ログ収集  
Get-AutopilotDiagnostics -Online -OutputPath C:\temp\AutopilotDiag.zip
```

#### iOS

1. **設定** > **プライバシーとセキュリティ** > **分析と改善**
2. **分析データ** から該当ログを確認
3. MDM 関連ログを確認

---

## 📚 参考資料

### Microsoft 公式ドキュメント

- [Windows Autopilot documentation](https://docs.microsoft.com/windows/deployment/windows-autopilot/)
- [Microsoft Intune documentation](https://docs.microsoft.com/mem/intune/)
- [iOS device enrollment](https://docs.microsoft.com/mem/intune/enrollment/ios-enroll)

### Apple 公式ドキュメント

- [Apple School Manager User Guide](https://support.apple.com/school-manager/)
- [Device Enrollment Program](https://www.apple.com/business/dep/)

### PowerShell モジュール

```powershell
# 必要なモジュールのインストール
Install-Module Microsoft.Graph.Intune
Install-Module WindowsAutoPilotIntune
Install-Module Microsoft.Graph.Authentication
```

---

## ✅ 次のステップ

このガイドを完了したら、以下の関連ドキュメントもご確認ください：

- [Intune デバイス管理](intune-device-management.md) - より詳細なIntune設定
- [セキュリティ管理](security-management.md) - 追加のセキュリティ対策
- [ユーザー管理](user-management.md) - 教育機関向けユーザー管理

---

**注意**: このガイドの設定例は教育機関での一般的な使用を想定しています。各機関の具体的な要件に応じて設定を調整してください。
