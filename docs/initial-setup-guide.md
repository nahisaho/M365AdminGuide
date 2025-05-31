# 🚀 Microsoft 365 初期セットアップガイド

## 📋 概要

Microsoft 365を教育機関に導入する際の初期セットアップ手順を詳しく説明します。このガイドでは、テナント作成から基本的な管理設定まで、ステップバイステップで実行できる包括的な手順を提供します。

## 🎯 セットアップの全体像

```
Phase 1: 準備段階（1-2週間）
├── 要件定義・計画策定
├── 既存システムの調査
├── ドメイン・DNS準備
└── 導入チーム編成

Phase 2: テナント作成（1-3日）
├── Microsoft 365テナント作成
├── 初期管理者設定
├── 基本的なセキュリティ設定
└── ドメイン追加・検証

Phase 3: 基盤設定（1-2週間）
├── ユーザー・グループ管理基盤
├── セキュリティポリシー設定
├── ライセンス管理設定
└── 基本サービス有効化
```

---

## 🛠️ Phase 1: 導入に必要な準備

### 📝 要件定義と事前調査

#### 1. 組織情報の整理

**必要な組織情報**
```
基本情報：
├── 組織名（正式名称・英語表記）
├── 所在地・連絡先情報
├── 教育機関種別（小学校/中学校/高等学校/大学等）
├── 学生数・教職員数
└── 既存ITシステム構成

技術要件：
├── 既存ドメイン名
├── DNS管理状況
├── 既存メールシステム
├── ネットワーク構成
└── セキュリティ要件
```

#### 2. ライセンス計画の策定

**ユーザー分類とライセンス計画**

| ユーザー種別 | 想定人数 | 推奨ライセンス | 理由 |
|---|---|---|---|
| **管理者** | 2-5名 | A5 | 高度な管理機能・セキュリティ |
| **教職員** | ○○名 | A3 | デスクトップアプリ・デバイス管理 |
| **学生（高学年）** | ○○名 | A3 | 本格的なICT活用 |
| **学生（低学年）** | ○○名 | A1 | 基本機能での導入 |

**ライセンス数計算例**
```powershell
# ライセンス必要数の計算テンプート

$organizationData = @{
    Administrators = 3
    Faculty = 50
    SeniorStudents = 200
    JuniorStudents = 300
    GuestUsers = 10
}

$licenseRequirement = @{
    A5_Licenses = $organizationData.Administrators
    A3_Licenses = $organizationData.Faculty + $organizationData.SeniorStudents
    A1_Licenses = $organizationData.JuniorStudents + $organizationData.GuestUsers
}

Write-Host "=== ライセンス必要数 ==="
Write-Host "A5 ライセンス: $($licenseRequirement.A5_Licenses) 個"
Write-Host "A3 ライセンス: $($licenseRequirement.A3_Licenses) 個"
Write-Host "A1 ライセンス: $($licenseRequirement.A1_Licenses) 個"
```

#### 3. ドメインとDNSの準備

**ドメイン要件の確認**
```
必要なドメイン情報：
├── 組織の公式ドメイン（例：school.edu.jp）
├── DNS管理権限の確認
├── 既存メールシステムとの連携要件
└── サブドメイン利用の検討

DNS設定準備：
├── MXレコード設定権限
├── TXTレコード設定権限
├── CNAMEレコード設定権限
└── 変更反映時間の確認
```

**DNS管理者への事前連絡テンプレート**
```
件名: Microsoft 365導入に伴うDNS設定変更のお願い

Microsoft 365の導入に伴い、以下のDNS設定変更が必要になります：

1. ドメイン所有権検証用TXTレコード
   - レコード名: [設定時に指定]
   - 値: [設定時に指定]

2. メール配信用MXレコード（段階的移行時）
   - 優先度: [設定時に指定]
   - 値: [テナント名].mail.protection.outlook.com

3. その他認証用レコード
   - SPF、DKIM、DMARC設定

設定日程について相談させてください。
```

#### 4. 既存システムとの統合計画

**既存システム調査チェックリスト**
```
メールシステム：
□ 現在のメールサーバー（Exchange Server等）
□ メールボックス数・サイズ
□ 既存メールアーカイブ
□ カスタム設定・ルール

認証システム：
□ Active Directory Domain Services
□ LDAP サーバー
□ シングルサインオン設定
□ 多要素認証システム

ファイルサーバー：
□ ファイルサーバー構成
□ 共有フォルダ構造
□ アクセス権設定
□ データ容量

学習管理システム：
□ LMS（Learning Management System）
□ 学生情報システム
□ 成績管理システム
□ 図書管理システム
```

### 👥 導入チームの編成

#### プロジェクトチーム構成

**推奨チーム構成**
```
プロジェクトマネージャー（1名）
├── 全体進行管理
├── 関係者調整
└── 意思決定支援

技術責任者（1-2名）
├── システム設計・構築
├── セキュリティ設定
└── 技術的課題解決

運用責任者（1-2名）
├── 日常運用設計
├── ユーザーサポート体制
└── 運用手順書作成

教育担当者（2-3名）
├── ユーザートレーニング計画
├── 利用ガイド作成
└── 段階的展開支援
```

**各担当者の事前準備**

| 役割 | 事前準備項目 |
|---|---|
| **プロジェクトマネージャー** | プロジェクト計画書、予算確保、上層部承認 |
| **技術責任者** | Microsoft 365技術習得、既存システム調査 |
| **運用責任者** | 運用ポリシー策定、サポート体制設計 |
| **教育担当者** | 教育計画策定、教材準備 |

---

## 🏗️ Phase 2: Microsoft 365テナントの作成と初期設定

### 🎫 テナント作成手順

#### 1. Microsoft 365 Education サインアップ

**アクセス手順**
1. **教育機関向けサインアップページにアクセス**
   ```
   URL: https://www.microsoft.com/ja-jp/education/products/office
   ```

2. **「無料で使ってみる」をクリック**
   - A1ライセンスから開始可能
   - 後からA3/A5にアップグレード可能

3. **組織情報の入力**
   ```
   必要情報：
   ├── 組織名（正式名称）
   ├── 国/地域
   ├── 教育機関種別
   ├── 学生数規模
   └── 管理者メールアドレス
   ```

#### 2. 初期テナント設定

**テナント作成時の重要な設定**

```powershell
# テナント作成後の初期確認コマンド

# Microsoft Graph PowerShell に接続
Connect-MgGraph -Scopes "Organization.Read.All", "Directory.ReadWrite.All"

# テナント基本情報の確認
Get-MgOrganization | Select-Object DisplayName, Id, CreatedDateTime, Country

# 初期ドメイン情報の確認
Get-MgDomain | Select-Object Id, IsDefault, IsInitial, IsVerified
```

**テナント基本設定の確認項目**
```
テナント情報：
□ テナント名の確認
□ 初期ドメイン（.onmicrosoft.com）の確認
□ 地域設定の確認
□ タイムゾーン設定

管理者設定：
□ 全体管理者アカウントの作成
□ セキュリティ設定の確認
□ 通知設定の確認
```

#### 3. Microsoft 365管理センターへのアクセス

**管理センターの場所とアクセス**
```
主要管理センター：
├── Microsoft 365管理センター: https://admin.microsoft.com
├── Microsoft Entra管理センター: https://entra.microsoft.com
├── セキュリティセンター: https://security.microsoft.com
├── コンプライアンスセンター: https://compliance.microsoft.com
└── Exchangeオンライン管理センター: https://admin.exchange.microsoft.com
```

### 🔐 管理者アカウントの作成と初期セキュリティ設定

#### 1. 管理者アカウントの作成

**緊急時アクセス用アカウント（Break Glass Account）の作成**

```powershell
# 緊急時アクセス用管理者アカウントの作成

# アカウント情報の定義
$emergencyAdmin = @{
    DisplayName = "Emergency Administrator"
    UserPrincipalName = "emergency-admin@yourdomain.onmicrosoft.com"
    PasswordProfile = @{
        Password = "ComplexPassword123!"
        ForceChangePasswordNextSignIn = $false
    }
    AccountEnabled = $true
}

# アカウント作成
$newUser = New-MgUser @emergencyAdmin

# 全体管理者ロールの割り当て
$globalAdminRole = Get-MgDirectoryRole | Where-Object {$_.DisplayName -eq "Global Administrator"}
New-MgDirectoryRoleMember -DirectoryRoleId $globalAdminRole.Id -BodyParameter @{
    "@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$($newUser.Id)"
}

Write-Host "緊急時アクセス用管理者アカウントを作成しました: $($emergencyAdmin.UserPrincipalName)"
```

**日常業務用管理者アカウントの作成**

```powershell
# 専門分野別管理者アカウントの作成例

$administrators = @(
    @{
        DisplayName = "IT Manager"
        UserPrincipalName = "it-manager@yourdomain.onmicrosoft.com"
        Department = "IT"
        Roles = @("Global Administrator", "Security Administrator")
    },
    @{
        DisplayName = "Security Manager"
        UserPrincipalName = "security-manager@yourdomain.onmicrosoft.com"
        Department = "IT"
        Roles = @("Security Administrator", "Compliance Administrator")
    },
    @{
        DisplayName = "User Manager"
        UserPrincipalName = "user-manager@yourdomain.onmicrosoft.com"
        Department = "HR"
        Roles = @("User Administrator", "License Administrator")
    }
)

foreach ($admin in $administrators) {
    # ユーザー作成
    $newAdmin = New-MgUser @{
        DisplayName = $admin.DisplayName
        UserPrincipalName = $admin.UserPrincipalName
        Department = $admin.Department
        PasswordProfile = @{
            Password = "TempPassword123!"
            ForceChangePasswordNextSignIn = $true
        }
        AccountEnabled = $true
    }
    
    # ロール割り当て
    foreach ($roleName in $admin.Roles) {
        $role = Get-MgDirectoryRole | Where-Object {$_.DisplayName -eq $roleName}
        if ($role) {
            New-MgDirectoryRoleMember -DirectoryRoleId $role.Id -BodyParameter @{
                "@odata.id" = "https://graph.microsoft.com/v1.0/directoryObjects/$($newAdmin.Id)"
            }
        }
    }
    
    Write-Host "管理者アカウントを作成しました: $($admin.DisplayName)"
}
```

#### 2. 初期セキュリティ設定

**多要素認証（MFA）の設定**

```powershell
# 全管理者に対するMFA要求の設定

# セキュリティデフォルトの有効化（推奨）
$securityDefaults = @{
    IsEnabled = $true
    Description = "Security defaults enabled for all users"
}

# 条件付きアクセスポリシーの作成（より詳細な制御が必要な場合）
$mfaPolicy = @{
    DisplayName = "Require MFA for Administrators"
    State = "enabled"
    Conditions = @{
        Users = @{
            IncludeRoles = @(
                "62e90394-69f5-4237-9190-012177145e10", # Global Administrator
                "194ae4cb-b126-40b2-bd5b-6091b380977d"  # Security Administrator
            )
        }
        Applications = @{
            IncludeApplications = @("All")
        }
    }
    GrantControls = @{
        Operator = "AND"
        BuiltInControls = @("mfa")
    }
}

# 条件付きアクセスポリシーの作成
New-MgIdentityConditionalAccessPolicy -BodyParameter $mfaPolicy
```

**パスワードポリシーの設定**

```powershell
# パスワードポリシーの設定

$passwordPolicy = @{
    DefaultPasswordRuleSet = @{
        BannedPasswordCheckOnPremisesMode = "Audit"
        EnableBannedPasswordCheck = $true
        BannedPasswordList = @(
            "password", "123456", "school", "student", "admin"
        )
    }
    PasswordComplexityRules = @{
        MinimumLength = 12
        RequireUppercase = $true
        RequireLowercase = $true
        RequireNumbers = $true
        RequireSymbols = $true
    }
}

Write-Host "パスワードポリシーを設定しました"
```

**アカウントロックアウトポリシーの設定**

```powershell
# サインインリスクベースのポリシー設定

$signInRiskPolicy = @{
    DisplayName = "Block high-risk sign-ins"
    State = "enabled"
    Conditions = @{
        Users = @{
            IncludeUsers = @("All")
        }
        SignInRiskLevels = @("high")
    }
    GrantControls = @{
        Operator = "OR"
        BuiltInControls = @("block")
    }
}

New-MgIdentityConditionalAccessPolicy -BodyParameter $signInRiskPolicy
```

#### 3. 監査ログとアラートの設定

**監査ログの有効化**

```powershell
# 統合監査ログの有効化と設定

# Exchange Online PowerShell への接続が必要
# Connect-ExchangeOnline

# 統合監査ログの有効化
Set-AdminAuditLogConfig -UnifiedAuditLogIngestionEnabled $true

# 監査ログ保持期間の設定（90日間）
Set-AdminAuditLogConfig -AdminAuditLogAgeLimit 90.00:00:00

# 重要なアクティビティの監査設定
$auditSettings = @{
    AuditEnabled = $true
    DefaultAuditSet = @(
        "UserLoggedIn",
        "UserLoginFailed", 
        "UserPasswordChanged",
        "UserAdded",
        "UserDeleted",
        "RoleAssignmentAdded",
        "AdminSubmittedQuarantineRelease"
    )
}

Write-Host "監査ログ設定を完了しました"
```

**セキュリティアラートの設定**

```powershell
# Microsoft 365 Defender でのアラート設定

$securityAlerts = @(
    @{
        Name = "High-risk sign-in detected"
        Description = "Alert when high-risk sign-in is detected"
        Severity = "High"
        Category = "Authentication"
    },
    @{
        Name = "Mass file download"
        Description = "Alert when unusual file download activity is detected"
        Severity = "Medium"
        Category = "Data Loss Prevention"
    },
    @{
        Name = "Admin role assignment"
        Description = "Alert when admin roles are assigned"
        Severity = "High"
        Category = "Identity and Access"
    }
)

foreach ($alert in $securityAlerts) {
    Write-Host "アラート設定: $($alert.Name)"
    # 実際のアラート作成はMicrosoft 365 Defenderコンソールから実行
}
```

---

## 🌐 Phase 3: ドメインの追加と検証手続き

### 📝 ドメイン追加の準備

#### 1. ドメイン追加前の確認事項

**ドメイン要件チェックリスト**
```
技術要件：
□ ドメインの所有権確認
□ DNS管理権限の確保
□ 既存メールシステムとの整合性
□ サブドメイン利用の検討

運用要件：
□ メール移行計画の策定
□ ユーザー通知計画
□ ダウンタイム許容時間
□ ロールバック計画
```

#### 2. ドメイン追加手順

**Microsoft 365管理センターでのドメイン追加**

1. **管理センターでのドメイン追加開始**
   ```
   手順：
   1. Microsoft 365管理センター (https://admin.microsoft.com) にアクセス
   2. [設定] > [ドメイン] を選択
   3. [ドメインの追加] をクリック
   4. ドメイン名を入力（例：school.edu.jp）
   ```

2. **PowerShellでのドメイン追加**
   ```powershell
   # PowerShellを使用したドメイン追加

   # ドメイン追加
   $domainName = "school.edu.jp"
   New-MgDomain -Id $domainName

   # ドメイン情報の確認
   Get-MgDomain -DomainId $domainName | Select-Object Id, IsVerified, SupportedServices
   ```

#### 3. ドメイン所有権の検証

**TXTレコードによる検証**

```powershell
# ドメイン検証用TXTレコードの取得

$domain = Get-MgDomain -DomainId $domainName
$verificationRecord = Get-MgDomainVerificationDnsRecord -DomainId $domainName

Write-Host "=== ドメイン検証情報 ==="
Write-Host "ドメイン名: $($domain.Id)"
Write-Host "検証レコード種別: $($verificationRecord.RecordType)"
Write-Host "検証値: $($verificationRecord.Text)"
Write-Host ""
Write-Host "DNS設定例:"
Write-Host "種別: TXT"
Write-Host "名前: @ または $domainName"
Write-Host "値: $($verificationRecord.Text)"
```

**DNS設定の実装例**

```bash
# DNS設定例（Bind形式）

# ドメイン検証用TXTレコード
school.edu.jp.    IN    TXT    "MS=ms12345678"

# メール配信用MXレコード（検証後に設定）
school.edu.jp.    IN    MX    10    school-edu-jp.mail.protection.outlook.com.

# SPFレコード（メールセキュリティ）
school.edu.jp.    IN    TXT    "v=spf1 include:spf.protection.outlook.com -all"

# DKIM設定用CNAMEレコード（Microsoft 365で自動生成される値を使用）
selector1._domainkey.school.edu.jp.    IN    CNAME    selector1-school-edu-jp._domainkey.yourtenant.onmicrosoft.com.
selector2._domainkey.school.edu.jp.    IN    CNAME    selector2-school-edu-jp._domainkey.yourtenant.onmicrosoft.com.
```

#### 4. ドメイン検証の実行と確認

**検証プロセスの自動化**

```powershell
# ドメイン検証の自動確認スクリプト

function Test-DomainVerification {
    param(
        [string]$DomainName,
        [int]$MaxRetries = 10,
        [int]$DelaySeconds = 30
    )
    
    $retryCount = 0
    
    while ($retryCount -lt $MaxRetries) {
        try {
            # ドメイン検証の実行
            Invoke-MgVerifyDomain -DomainId $DomainName
            
            # 検証状態の確認
            $domain = Get-MgDomain -DomainId $DomainName
            
            if ($domain.IsVerified) {
                Write-Host "✅ ドメイン検証成功: $DomainName" -ForegroundColor Green
                return $true
            }
            else {
                Write-Host "⏳ 検証待機中... ($($retryCount + 1)/$MaxRetries)" -ForegroundColor Yellow
                Start-Sleep -Seconds $DelaySeconds
                $retryCount++
            }
        }
        catch {
            Write-Host "❌ 検証エラー: $($_.Exception.Message)" -ForegroundColor Red
            Start-Sleep -Seconds $DelaySeconds
            $retryCount++
        }
    }
    
    Write-Host "❌ ドメイン検証失敗: $DomainName" -ForegroundColor Red
    return $false
}

# 使用例
$verificationResult = Test-DomainVerification -DomainName "school.edu.jp"
```

#### 5. メールルーティングの設定

**段階的メール移行の設定**

```powershell
# メールルーティング設定（段階的移行用）

# Exchange Online PowerShell接続
# Connect-ExchangeOnline

# 承認されたドメインの設定
New-AcceptedDomain -Name "school.edu.jp" -DomainName "school.edu.jp" -DomainType Authoritative

# メール配信ポリシーの設定
$emailPolicy = @{
    Name = "School Email Policy"
    Domains = @("school.edu.jp")
    MatchedSenderAttributes = @("Department")
    RouteBasedOnDepartment = $true
}

# 段階的移行用のコネクタ設定（既存メールサーバーとの連携）
$onPremConnector = @{
    Name = "OnPremises-Inbound"
    ConnectorType = "OnPremises"
    ConnectorSource = "Default"
    SmartHosts = @("mail.school.edu.jp")
    TlsSettings = @{
        RequireTls = $true
        CertificateValidation = "Enabled"
    }
}

Write-Host "メールルーティング設定を完了しました"
```

### 🔧 ドメイン設定の最適化

#### 1. セキュリティレコードの設定

**完全なDNSセキュリティ設定**

```powershell
# 推奨DNSセキュリティ設定の生成

function Generate-SecurityDNSRecords {
    param(
        [string]$DomainName,
        [string]$TenantName
    )
    
    $dnsRecords = @"
=== $DomainName セキュリティDNS設定 ===

# SPF レコード（メール送信元認証）
$DomainName.    IN    TXT    "v=spf1 include:spf.protection.outlook.com -all"

# DMARC レコード（メール認証ポリシー）
_dmarc.$DomainName.    IN    TXT    "v=DMARC1; p=quarantine; rua=mailto:dmarc@$DomainName; ruf=mailto:dmarc@$DomainName"

# DKIM セレクター1
selector1._domainkey.$DomainName.    IN    CNAME    selector1-$(($DomainName -replace '\.', '-'))._domainkey.$TenantName.onmicrosoft.com.

# DKIM セレクター2  
selector2._domainkey.$DomainName.    IN    CNAME    selector2-$(($DomainName -replace '\.', '-'))._domainkey.$TenantName.onmicrosoft.com.

# Exchange Autodiscover
autodiscover.$DomainName.    IN    CNAME    autodiscover.outlook.com.

# SIP連携（Teams）
sip.$DomainName.    IN    CNAME    sipdir.online.lync.com.
lyncdiscover.$DomainName.    IN    CNAME    webdir.online.lync.com.

# モバイルデバイス管理
enterpriseregistration.$DomainName.    IN    CNAME    enterpriseregistration.windows.net.
enterpriseenrollment.$DomainName.    IN    CNAME    enterpriseenrollment.manage.microsoft.com.
"@
    
    return $dnsRecords
}

# 使用例
$dnsConfig = Generate-SecurityDNSRecords -DomainName "school.edu.jp" -TenantName "schooledu"
Write-Host $dnsConfig
```

#### 2. ドメイン設定の検証

**包括的なドメイン設定チェック**

```powershell
# ドメイン設定の総合検証

function Test-DomainConfiguration {
    param([string]$DomainName)
    
    $results = @()
    
    # 基本ドメイン情報
    $domain = Get-MgDomain -DomainId $DomainName
    $results += [PSCustomObject]@{
        Check = "ドメイン検証"
        Status = if($domain.IsVerified) {"✅ 成功"} else {"❌ 失敗"}
        Details = "検証状態: $($domain.IsVerified)"
    }
    
    # DNS レコードの確認
    try {
        $mxRecords = Resolve-DnsName -Name $DomainName -Type MX -ErrorAction Stop
        $results += [PSCustomObject]@{
            Check = "MXレコード"
            Status = "✅ 設定済み"
            Details = "優先度: $($mxRecords[0].Preference), 値: $($mxRecords[0].NameExchange)"
        }
    }
    catch {
        $results += [PSCustomObject]@{
            Check = "MXレコード"
            Status = "❌ 未設定"
            Details = "MXレコードが見つかりません"
        }
    }
    
    # SPF レコードの確認
    try {
        $spfRecords = Resolve-DnsName -Name $DomainName -Type TXT -ErrorAction Stop | 
                     Where-Object {$_.Strings -match "v=spf1"}
        if ($spfRecords) {
            $results += [PSCustomObject]@{
                Check = "SPFレコード"
                Status = "✅ 設定済み"
                Details = $spfRecords.Strings[0]
            }
        }
        else {
            $results += [PSCustomObject]@{
                Check = "SPFレコード"
                Status = "⚠️ 未設定"
                Details = "SPFレコードが見つかりません"
            }
        }
    }
    catch {
        $results += [PSCustomObject]@{
            Check = "SPFレコード"
            Status = "❌ エラー"
            Details = "DNS解決エラー"
        }
    }
    
    return $results
}

# 使用例とレポート生成
$domainCheck = Test-DomainConfiguration -DomainName "school.edu.jp"
$domainCheck | Format-Table -AutoSize

# 結果をCSVに出力
$domainCheck | Export-Csv "domain-configuration-report.csv" -NoTypeInformation -Encoding UTF8
```

---

## 📋 セットアップ完了後のチェックリスト

### ✅ 必須確認項目

```
テナント基本設定：
□ テナント名・ドメイン設定確認
□ 管理者アカウント作成・権限設定
□ セキュリティデフォルト有効化
□ 多要素認証設定完了

ドメイン設定：
□ ドメイン検証完了
□ DNS設定（MX、SPF、DKIM）完了
□ メールルーティング正常動作
□ Autodiscover設定完了

セキュリティ設定：
□ 条件付きアクセスポリシー設定
□ 監査ログ有効化
□ アラート設定完了
□ パスワードポリシー設定

サービス有効化：
□ Exchange Online 動作確認
□ SharePoint Online アクセス確認
□ Teams 基本動作確認
□ OneDrive 同期確認
```

### 🔄 継続的な監視項目

**日次監視項目**
```powershell
# 日次チェックスクリプト例

function Daily-HealthCheck {
    $report = @()
    
    # サービス稼働状況
    $serviceHealth = Get-MgServicePrincipal | Where-Object {$_.DisplayName -like "*Office 365*"}
    $report += "サービス稼働状況: 正常"
    
    # 新規サインイン失敗
    $failedSignins = Get-MgAuditLogSignIn -Filter "status/errorCode ne 0 and createdDateTime ge $(Get-Date -Format yyyy-MM-dd)" 
    $report += "サインイン失敗: $($failedSignins.Count) 件"
    
    # ライセンス使用状況
    $skus = Get-MgSubscribedSku
    foreach ($sku in $skus) {
        $usage = [Math]::Round(($sku.ConsumedUnits / $sku.PrepaidUnits.Enabled) * 100, 1)
        $report += "ライセンス使用率 ($($sku.SkuPartNumber)): $usage%"
    }
    
    return $report
}

# 日次レポート実行
$dailyReport = Daily-HealthCheck
$dailyReport | ForEach-Object { Write-Host $_ }
```

**週次監視項目**
```powershell
# 週次チェックスクリプト例

function Weekly-SecurityCheck {
    # 高リスクサインインの確認
    $riskySigns = Get-MgIdentityConditionalAccessNamedLocation | 
                  Where-Object {$_.CreatedDateTime -ge (Get-Date).AddDays(-7)}
    
    # 新規管理者の確認
    $adminRoles = Get-MgDirectoryRole | Where-Object {$_.DisplayName -like "*Admin*"}
    
    # パスワード変更状況
    $passwordChanges = Get-MgAuditLogDirectoryAudit -Filter "category eq 'UserManagement' and activityDisplayName eq 'Change user password'"
    
    Write-Host "=== 週次セキュリティレポート ==="
    Write-Host "リスク検出数: $($riskySigns.Count)"
    Write-Host "管理者ロール数: $($adminRoles.Count)"
    Write-Host "パスワード変更: $($passwordChanges.Count) 件"
}

# 週次レポート実行
Weekly-SecurityCheck
```

---

## 🚨 トラブルシューティング

### よくある問題と解決方法

#### 1. ドメイン検証失敗

**症状**: TXTレコードを設定したがドメイン検証が通らない

**解決手順**:
```powershell
# DNS伝播状況の確認
function Test-DNSPropagation {
    param([string]$DomainName, [string]$RecordType = "TXT")
    
    $publicDNS = @("8.8.8.8", "1.1.1.1", "208.67.222.222")
    
    foreach ($dns in $publicDNS) {
        try {
            $result = Resolve-DnsName -Name $DomainName -Type $RecordType -Server $dns -ErrorAction Stop
            Write-Host "DNS $dns : ✅ 解決成功" -ForegroundColor Green
            Write-Host "  値: $($result.Strings -join ', ')"
        }
        catch {
            Write-Host "DNS $dns : ❌ 解決失敗" -ForegroundColor Red
        }
    }
}

# 使用例
Test-DNSPropagation -DomainName "school.edu.jp" -RecordType "TXT"
```

#### 2. メール配信問題

**症状**: メールが正常に配信されない

**診断手順**:
```powershell
# メール配信の診断

# Exchange Online でメッセージ追跡
# Connect-ExchangeOnline
$messageTrace = Get-MessageTrace -StartDate (Get-Date).AddHours(-24) -EndDate (Get-Date)
$messageTrace | Where-Object {$_.Status -eq "Failed"} | 
    Select-Object Received, SenderAddress, RecipientAddress, Status, Detail

# SPF/DKIM設定の確認
function Test-EmailAuthentication {
    param([string]$DomainName)
    
    # SPF確認
    $spfRecord = Resolve-DnsName -Name $DomainName -Type TXT | 
                Where-Object {$_.Strings -match "v=spf1"}
    
    # DKIM確認
    $dkimSelector1 = Resolve-DnsName -Name "selector1._domainkey.$DomainName" -Type CNAME
    $dkimSelector2 = Resolve-DnsName -Name "selector2._domainkey.$DomainName" -Type CNAME
    
    Write-Host "SPF設定: $(if($spfRecord){'✅'}else{'❌'})"
    Write-Host "DKIM Selector1: $(if($dkimSelector1){'✅'}else{'❌'})"
    Write-Host "DKIM Selector2: $(if($dkimSelector2){'✅'}else{'❌'})"
}

Test-EmailAuthentication -DomainName "school.edu.jp"
```

#### 3. サインイン問題

**症状**: ユーザーがサインインできない

**診断と解決**:
```powershell
# ユーザーサインイン状況の確認

function Diagnose-UserSignIn {
    param([string]$UserPrincipalName)
    
    # ユーザー基本情報
    $user = Get-MgUser -UserId $UserPrincipalName
    Write-Host "ユーザー状態: $(if($user.AccountEnabled){'有効'}else{'無効'})"
    
    # ライセンス状況
    $licenses = $user.AssignedLicenses
    Write-Host "ライセンス数: $($licenses.Count)"
    
    # 最近のサインイン履歴
    $signIns = Get-MgAuditLogSignIn -Filter "userPrincipalName eq '$UserPrincipalName'" -Top 10
    Write-Host "直近のサインイン試行: $($signIns.Count) 件"
    
    # エラーがある場合の詳細
    $errors = $signIns | Where-Object {$_.Status.ErrorCode -ne 0}
    if ($errors) {
        Write-Host "エラー詳細:"
        $errors | ForEach-Object {
            Write-Host "  - $($_.Status.FailureReason)"
        }
    }
}

# 使用例
Diagnose-UserSignIn -UserPrincipalName "student@school.edu.jp"
```

---

## 📚 次のステップと関連ドキュメント

### 🎯 セットアップ完了後の推奨作業

1. **[ユーザー管理設定](user-management.md)**
   - 一括ユーザー作成
   - グループ管理設定
   - ライセンス割り当て自動化

2. **[セキュリティ強化](security-management.md)**
   - 条件付きアクセス詳細設定
   - データ損失防止ポリシー
   - 脅威保護設定

3. **[サービス個別設定](teams-management.md)**
   - Teams for Education設定
   - SharePoint サイト作成
   - Exchange メール機能

4. **[監視・レポート設定](reports-analytics.md)**
   - 利用状況分析
   - セキュリティ監視
   - パフォーマンス監視

### 📖 関連ドキュメント

- [Microsoft 365 基本概念とライセンス](microsoft-365-basics-and-licensing.md)
- [概要と前提条件](overview-and-prerequisites.md)
- [ライセンス管理](license-management.md)
- [Microsoft Graph PowerShell](microsoft-graph-powershell.md)

---

**⚠️ 重要な注意事項**

1. **バックアップとロールバック**
   - 設定変更前は必ず現在の設定をバックアップ
   - ロールバック手順を事前に確認
   - 重要な変更は検証環境で事前テスト

2. **セキュリティ最優先**
   - 管理者アカウントのパスワードは定期変更
   - 多要素認証は全管理者で必須
   - 最小権限の原則を遵守

3. **段階的導入**
   - 一度に全機能を有効化しない
   - パイロットユーザーでの検証を推奨
   - ユーザートレーニングと並行実施

このガイドに従って初期セットアップを完了すれば、安全で効率的なMicrosoft 365環境の基盤が構築されます。
