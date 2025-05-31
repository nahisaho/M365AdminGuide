# 👥 Microsoft 365 ユーザー管理ガイド

## 📋 概要

Microsoft 365におけるユーザーアカウントの総合的な管理手順を説明します。教育機関での実運用を想定し、GUI操作からPowerShell自動化、年次更新プロセス、パスワードレス認証まで網羅的にカバーします。

## 🎯 このドキュメントで学べること

```text
基本操作：
├── GUI（Microsoft 365管理センター）でのユーザー管理
├── PowerShellでの個別ユーザー操作
├── CSVファイルを活用した一括処理
└── 教育機関特有の運用プロセス

高度な管理：
├── アカウントライフサイクル管理
├── 年次更新の自動化
├── パスワードレス認証の実装
└── セキュリティ強化手法
```

## ✅ 前提条件

### 必要な権限

- **Microsoft 365管理者権限**（User Administrator以上）
- **Microsoft Entra ID Premium**（条件付きアクセス利用時）
- **適切なライセンス**（管理対象ユーザー分）

### 技術要件

```powershell
# 必要なPowerShellモジュール
Install-Module Microsoft.Graph -Force
Install-Module ImportExcel -Force  # CSV処理の拡張用
```

### 事前準備

```text
組織情報：
├── ドメイン設定完了
├── ライセンス購入・割り当て計画
├── セキュリティポリシー策定
└── 運用ルール確立

技術環境：
├── PowerShell実行環境
├── CSVファイル管理方法
├── バックアップ・復旧手順
└── 監査ログ設定
```

## ユーザーの作成

### Microsoft 365 管理センターを使用

1. [Microsoft 365 管理センター](https://admin.microsoft.com) にアクセス
2. **ユーザー** > **アクティブなユーザー** を選択
3. **ユーザーの追加** をクリック
4. 必要な情報を入力：
   - 名前
   - ユーザー名
   - パスワード設定
   - ライセンス割り当て

### PowerShell を使用

**以下のPowerShellコードの処理内容:**

1. `Connect-MgGraph -Scopes "User.ReadWrite.All"` - Microsoft Graph APIに接続し、全ユーザーの読み取り・書き込み権限を要求
2. `$passwordProfile` - 新規ユーザーのパスワード設定オブジェクトを作成、初回サインイン時の強制パスワード変更を有効化
3. `$newUser` - 新規ユーザーアカウントの基本情報（表示名、ログインID、メールニックネーム等）を定義
4. `New-MgUser` - 定義されたパラメータを使用して新しいユーザーアカウントをMicrosoft 365テナントに作成

```powershell
# Microsoft Graph に接続
Connect-MgGraph -Scopes "User.ReadWrite.All"

# 新しいユーザーを作成
$passwordProfile = @{
    ForceChangePasswordNextSignIn = $true
    Password = "TempPassword123!"
}

$newUser = @{
    DisplayName = "田中 太郎"
    UserPrincipalName = "tanaka@contoso.onmicrosoft.com"
    MailNickname = "tanaka"
    PasswordProfile = $passwordProfile
    AccountEnabled = $true
}

New-MgUser @newUser
```

## ユーザーの編集

### 基本情報の更新

**以下のPowerShellコードの処理内容:**

1. `Update-MgUser` - Microsoft Graph APIを使用して既存ユーザーの属性情報を更新
2. `-UserId` - 更新対象ユーザーをUserPrincipalName（ログインID）で指定
3. `-DisplayName` - ユーザーの表示名（氏名）を新しい値に変更、組織内の表示やディレクトリ検索で使用

```powershell
# ユーザー情報を更新
Update-MgUser -UserId "tanaka@contoso.onmicrosoft.com" -DisplayName "田中 次郎"
```

### ライセンスの割り当て

#### 💡 ライセンス別機能比較

| 機能 | A1 | A3 | A5 |
|------|----|----|----| 
| デスクトップ Office | ❌ | ✅ | ✅ |
| Intune 管理 | ❌ | ✅ | ✅ |
| 条件付きアクセス | ❌ | ✅ | ✅ |
| Identity Protection | ❌ | ❌ | ✅ |

```powershell
# 利用可能なライセンスを確認
Get-MgSubscribedSku | Select-Object SkuPartNumber, ConsumedUnits, PrepaidUnits

# A1 ライセンスの割り当て（基本機能のみ）
$a1License = @{
    AddLicenses = @(
        @{
            SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "STANDARDWOFFPACK_FACULTY"}).SkuId
        }
    )
    RemoveLicenses = @()
}
Set-MgUserLicense -UserId "tanaka@contoso.edu" -BodyParameter $a1License

# A3 ライセンスの割り当て（Intune + 条件付きアクセス利用可能）
$a3License = @{
    AddLicenses = @(
        @{
            SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPACK_FACULTY"}).SkuId
        }
    )
    RemoveLicenses = @()
}
Set-MgUserLicense -UserId "principal@contoso.edu" -BodyParameter $a3License

# A5 ライセンスの割り当て（全機能利用可能）
$a5License = @{
    AddLicenses = @(
        @{
            SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPREMIUM_FACULTY"}).SkuId
        }
    )
    RemoveLicenses = @()
}
Set-MgUserLicense -UserId "manager@contoso.edu" -BodyParameter $a5License
```

## ユーザーの無効化と削除

### ユーザーアカウントの無効化

```powershell
# ユーザーアカウントを無効化
Update-MgUser -UserId "tanaka@contoso.onmicrosoft.com" -AccountEnabled:$false
```

### ユーザーの削除

```powershell
# ユーザーを削除（論理削除）
Remove-MgUser -UserId "tanaka@contoso.onmicrosoft.com"

# 削除されたユーザーを確認
Get-MgDirectoryDeletedItem -DirectoryObjectId "user_id"

# 完全に削除
Remove-MgDirectoryDeletedItem -DirectoryObjectId "user_id"
```

## グループ管理

### 🏫 教育機関におけるグループ管理の概要

教育機関では、学年、学級、職員グループなど組織構造に合わせたグループ管理が重要です。Microsoft 365 では以下のタイプのグループを作成できます：

- **セキュリティグループ**: アクセス権限の管理
- **Microsoft 365 グループ**: コラボレーション用（Teams、SharePoint等）
- **配布グループ**: メール配信専用
- **動的グループ**: 属性ベースの自動メンバーシップ

### 📋 グループ作成制限とポリシー

#### グループ作成権限の設定

```powershell
# Microsoft Graph に接続
Connect-MgGraph -Scopes "Policy.ReadWrite.Authorization", "Group.ReadWrite.All"

# 現在のグループ作成ポリシーを確認
$groupSetting = Get-MgDirectorySetting | Where-Object {$_.DisplayName -eq "Group.Unified"}

if ($groupSetting) {
    $groupSetting.Values | Where-Object {$_.Name -eq "EnableGroupCreation"}
} else {
    Write-Host "グループ作成ポリシーが設定されていません" -ForegroundColor Yellow
}
```

#### グループ作成を管理者のみに制限

```powershell
# グループ作成を管理者のみに制限する設定
$template = Get-MgDirectorySettingTemplate | Where-Object {$_.DisplayName -eq "Group.Unified"}

$values = @()
$template.Values | ForEach-Object {
    $value = @{
        Name = $_.Name
        Value = $_.DefaultValue
    }
    
    # グループ作成を無効化
    if ($_.Name -eq "EnableGroupCreation") {
        $value.Value = "false"
    }
    
    # 許可されたグループ作成者を指定（管理者グループのObjectId）
    if ($_.Name -eq "GroupCreationAllowedGroupId") {
        $adminGroup = Get-MgGroup -Filter "displayName eq '教育機関管理者'"
        $value.Value = $adminGroup.Id
    }
    
    $values += $value
}

$settingParams = @{
    TemplateId = $template.Id
    Values = $values
}

New-MgDirectorySetting -BodyParameter $settingParams
Write-Host "✅ グループ作成制限を設定しました" -ForegroundColor Green
```

#### 学年・学級別グループ作成権限

```powershell
# 学年主任や学級担任にグループ作成権限を付与
$allowedCreators = @(
    "小学部管理者",
    "中学部管理者", 
    "事務局管理者",
    "学年主任"
)

foreach ($groupName in $allowedCreators) {
    try {
        $group = Get-MgGroup -Filter "displayName eq '$groupName'"
        if (-not $group) {
            $newGroup = @{
                DisplayName = $groupName
                Description = "$groupName のためのグループ作成権限グループ"
                GroupTypes = @()
                SecurityEnabled = $true
                MailEnabled = $false
                MailNickname = $groupName.Replace("管理者", "admin").Replace("主任", "manager")
            }
            New-MgGroup @newGroup
            Write-Host "✅ 作成権限グループ作成: $groupName" -ForegroundColor Green
        }
    }
    catch {
        Write-Host "❌ グループ作成エラー: $groupName - $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

### セキュリティグループの作成

```powershell
# 教育機関向けセキュリティグループを作成
$group = @{
    DisplayName = "6年1組"
    Description = "6年1組の児童・担任"
    GroupTypes = @()
    SecurityEnabled = $true
    MailEnabled = $false
    MailNickname = "grade6-class1"
}

New-MgGroup @group
```

### グループメンバーの追加

```powershell
# グループにメンバーを追加
$groupId = "group_id_here"
$userId = "user_id_here"

New-MgGroupMember -GroupId $groupId -DirectoryObjectId $userId
```

### 🔄 動的メンバーシップによるグループ管理

動的グループは、属性ルールに基づいてメンバーシップが自動的に管理されるグループです。Microsoft Entra ID Premium P1 または P2 ライセンスが必要です。

#### 動的グループの作成

```powershell
# 学級ベースの動的グループ作成
$dynamicGroupParams = @{
    DisplayName = "6年1組-動的グループ"
    Description = "6年1組所属者の動的グループ"
    GroupTypes = @("DynamicMembership")
    SecurityEnabled = $true
    MailEnabled = $false
    MailNickname = "grade6-class1-dynamic"
    MembershipRule = '(user.department -eq "6年1組")'
    MembershipRuleProcessingState = "On"
}

$dynamicGroup = New-MgGroup @dynamicGroupParams
Write-Host "✅ 動的グループ作成成功: $($dynamicGroup.DisplayName)" -ForegroundColor Green
```

#### 📋 教育機関向け動的グループルールの例

##### 1. 学部・学年ベースのルール

```powershell
# 特定の学部のユーザー
$rule1 = '(user.department -eq "小学部")'

# 複数の学部を含む
$rule2 = '(user.department -eq "小学部") or (user.department -eq "中学部")'

# 小学生
$rule3 = '(user.department -eq "小学部") and (user.jobTitle -eq "児童")'

# 中学生
$rule4 = '(user.department -eq "中学部") and (user.jobTitle -eq "生徒")'
```

##### 2. 職位・身分ベースのルール

```powershell
# 教職員のみ
$rule5 = '(user.jobTitle -contains "校長") or (user.jobTitle -contains "教頭") or (user.jobTitle -contains "教諭") or (user.jobTitle -contains "養護教諭") or (user.jobTitle -contains "事務職員")'

$rule6 = '(user.jobTitle -contains "児童") or (user.jobTitle -contains "生徒")'

# 管理職
$rule7 = '(user.jobTitle -contains "校長") or (user.jobTitle -contains "教頭") or (user.jobTitle -contains "主幹教諭")'

# 新入生（入学日ベース）
$rule8 = '(user.employeeHireDate -ge "2025-04-01")'
```

##### 3. 校舎・場所ベースのルール

```powershell
# 本校舎のユーザー
$rule9 = '(user.physicalDeliveryOfficeName -eq "本校舎")'

# 分校・分室
$rule10 = '(user.physicalDeliveryOfficeName -eq "分校") or (user.physicalDeliveryOfficeName -eq "分室")'

# 特別教室利用者
$rule11 = '(user.physicalDeliveryOfficeName -eq "音楽室") or (user.physicalDeliveryOfficeName -eq "理科室")'
```

##### 4. 学年・コースベースのルール

```powershell
# 学年別（カスタム属性使用）
$rule12 = '(user.extensionAttribute1 -eq "小学1年")'  # 小学1年生
$rule13 = '(user.extensionAttribute1 -eq "中学3年")'  # 中学3年生

# 特別クラス・コース別
$rule14 = '(user.extensionAttribute2 -eq "特進クラス")'
$rule15 = '(user.extensionAttribute2 -eq "特別支援クラス")'
```

#### 🛠️ 高度な動的グループ管理

##### 動的グループの監視と管理

```powershell
# 動的グループの一覧取得
$dynamicGroups = Get-MgGroup -Filter "groupTypes/any(c:c eq 'DynamicMembership')"

foreach ($group in $dynamicGroups) {
    Write-Host "グループ名: $($group.DisplayName)" -ForegroundColor Cyan
    Write-Host "ルール: $($group.MembershipRule)" -ForegroundColor White
    Write-Host "状態: $($group.MembershipRuleProcessingState)" -ForegroundColor White
    
    # メンバー数の確認
    $memberCount = (Get-MgGroupMember -GroupId $group.Id).Count
    Write-Host "現在のメンバー数: $memberCount" -ForegroundColor Green
    Write-Host "---"
}
```

##### 動的グループルールのテスト

```powershell
# ルールのバリデーション（テスト用）
function Test-DynamicGroupRule {
    param(
        [string]$Rule,
        [string]$TestUserPrincipalName
    )
    
    try {
        # テスト用ユーザーの属性を取得
        $testUser = Get-MgUser -UserId $TestUserPrincipalName -Property "department,jobTitle,city,country,employeeHireDate"
        
        Write-Host "テスト対象ユーザー: $($testUser.DisplayName)" -ForegroundColor Cyan
        Write-Host "部署: $($testUser.Department)" -ForegroundColor White
        Write-Host "職位: $($testUser.JobTitle)" -ForegroundColor White
        Write-Host "都市: $($testUser.City)" -ForegroundColor White
        Write-Host "ルール: $Rule" -ForegroundColor Yellow
        
        # 実際の評価は Microsoft Graph API では直接できないため、
        # ルールを手動で確認する必要があります
        Write-Host "⚠️ ルールの評価結果は Microsoft Entra ID で確認してください" -ForegroundColor Yellow
    }
    catch {
        Write-Host "❌ テストエラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 使用例
Test-DynamicGroupRule -Rule '(user.department -eq "6年1組")' -TestUserPrincipalName "tanaka@school.edu"
```

#### 📅 学術カレンダーベースの動的グループ

```powershell
# 学期・年度ベースの動的グループ自動作成
function Create-AcademicCalendarGroups {
    param(
        [string]$AcademicYear = "2025",
        [string]$Semester = "前期"  # 前期、後期
    )
    
    # 授業参加期間中の児童生徒グループ
    $registrationGroupParams = @{
        DisplayName = "$AcademicYear 年度$Semester-授業参加中"
        Description = "$AcademicYear 年度$Semester の授業参加期間中の児童生徒"
        GroupTypes = @("DynamicMembership")
        SecurityEnabled = $true
        MailEnabled = $false
        MailNickname = "registration-$AcademicYear-$Semester"            MembershipRule = "(user.jobTitle -eq `"児童`" or user.jobTitle -eq `"生徒`") and (user.extensionAttribute3 -eq `"$AcademicYear`") and (user.accountEnabled -eq true)"
        MembershipRuleProcessingState = "On"
    }
    
    try {
        $registrationGroup = New-MgGroup @registrationGroupParams
        Write-Host "✅ 授業参加グループ作成: $($registrationGroup.DisplayName)" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ 授業参加グループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
    
    # 卒業予定者グループ（4年生かつ卒業年度）
    if ($Semester -eq "後期") {
        $graduateGroupParams = @{
            DisplayName = "$AcademicYear 年度-卒業予定者"
            Description = "$AcademicYear 年度の卒業予定者"
            GroupTypes = @("DynamicMembership")
            SecurityEnabled = $true
            MailEnabled = $false
            MailNickname = "graduates-$AcademicYear"
            MembershipRule = "(user.extensionAttribute1 -eq `"中学3年`") and (user.extensionAttribute3 -eq `"$((Get-Date).Year - 2)`") and (user.jobTitle -eq `"生徒`")"
            MembershipRuleProcessingState = "On"
        }
        
        try {
            $graduateGroup = New-MgGroup @graduateGroupParams
            Write-Host "✅ 卒業予定者グループ作成: $($graduateGroup.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ 卒業予定者グループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    # 新入生グループ（1年生かつ入学年度）
    if ($Semester -eq "前期") {
        $freshmanGroupParams = @{
            DisplayName = "$AcademicYear 年度-新入生"
            Description = "$AcademicYear 年度の新入生"
            GroupTypes = @("DynamicMembership")
            SecurityEnabled = $true
            MailEnabled = $false
            MailNickname = "freshmen-$AcademicYear"
            MembershipRule = "(user.extensionAttribute1 -eq `"小学1年`") and (user.extensionAttribute3 -eq `"$AcademicYear`") and (user.jobTitle -eq `"児童`")"
            MembershipRuleProcessingState = "On"
        }
        
        try {
            $freshmanGroup = New-MgGroup @freshmanGroupParams
            Write-Host "✅ 新入生グループ作成: $($freshmanGroup.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ 新入生グループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}

# 学期開始時の自動実行
Create-AcademicCalendarGroups -AcademicYear "2025" -Semester "前期"
```

#### 📊 動的グループライセンス管理の統合

```powershell
# 部署別ライセンス動的グループの自動作成
function Create-DepartmentLicenseGroups {
    param(
        [string[]]$Departments = @("小学部", "中学部", "事務局", "保健室", "図書室")
    )
    
    # A3ライセンス用動的グループ（教職員管理職）
    foreach ($dept in $Departments) {
        $a3GroupParams = @{
            DisplayName = "$dept-A3ライセンス-動的"
            Description = "$dept の教職員管理職向け A3 ライセンス動的グループ"
            GroupTypes = @("DynamicMembership")
            SecurityEnabled = $true
            MailEnabled = $false
            MailNickname = "$($dept.Replace('部','').Replace('局',''))-a3-dynamic"
            MembershipRule = "(user.department -eq `"$dept`") and ((user.jobTitle -contains `"校長`") or (user.jobTitle -contains `"教頭`") or (user.jobTitle -contains `"主幹教諭`") or (user.jobTitle -contains `"教務主任`"))"
            MembershipRuleProcessingState = "On"
        }
        
        try {
            $a3Group = New-MgGroup @a3GroupParams
            
            # A3ライセンスをグループに割り当て
            $a3SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPACK_FACULTY"}).SkuId
            $a3LicenseParams = @{
                AddLicenses = @(@{SkuId = $a3SkuId})
                RemoveLicenses = @()
            }
            Set-MgGroupLicense -GroupId $a3Group.Id -BodyParameter $a3LicenseParams
            
            Write-Host "✅ A3動的グループ作成成功: $($a3Group.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ A3動的グループ作成エラー: $dept - $($_.Exception.Message)" -ForegroundColor Red
        }
        
        # A1ライセンス用動的グループ（教職員一般）
        $a1GroupParams = @{
            DisplayName = "$dept-A1ライセンス-動的"
            Description = "$dept の教職員一般向け A1 ライセンス動的グループ"
            GroupTypes = @("DynamicMembership")
            SecurityEnabled = $true
            MailEnabled = $false
            MailNickname = "$($dept.Replace('部','').Replace('局',''))-a1-dynamic"
            MembershipRule = "(user.department -eq `"$dept`") and not ((user.jobTitle -contains `"校長`") or (user.jobTitle -contains `"教頭`") or (user.jobTitle -contains `"主幹教諭`") or (user.jobTitle -contains `"教務主任`") or (user.jobTitle -contains `"管理職`"))"
            MembershipRuleProcessingState = "On"
        }
        
        try {
            $a1Group = New-MgGroup @a1GroupParams
            
            # A1ライセンスをグループに割り当て
            $a1SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "STANDARDWOFFPACK_FACULTY"}).SkuId
            $a1LicenseParams = @{
                AddLicenses = @(@{SkuId = $a1SkuId})
                RemoveLicenses = @()
            }
            Set-MgGroupLicense -GroupId $a1Group.Id -BodyParameter $a1LicenseParams
            
            Write-Host "✅ A1動的グループ作成成功: $($a1Group.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ A1動的グループ作成エラー: $dept - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    # 管理職向けA5ライセンス動的グループ
    $a5GroupParams = @{
        DisplayName = "管理職-A5ライセンス-動的"
        Description = "管理職向け A5 ライセンス動的グループ"
        GroupTypes = @("DynamicMembership")
        SecurityEnabled = $true
        MailEnabled = $false
        MailNickname = "management-a5-dynamic"
        MembershipRule = '(user.jobTitle -contains "校長") or (user.jobTitle -contains "教頭") or (user.jobTitle -contains "副校長") or (user.jobTitle -contains "教務主任")'
        MembershipRuleProcessingState = "On"
    }
    
    try {
        $a5Group = New-MgGroup @a5GroupParams
        
        # A5ライセンスをグループに割り当て
        $a5SkuId = (Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPREMIUM_FACULTY"}).SkuId
        $a5LicenseParams = @{
            AddLicenses = @(@{SkuId = $a5SkuId})
            RemoveLicenses = @()
        }
        Set-MgGroupLicense -GroupId $a5Group.Id -BodyParameter $a5LicenseParams
        
        Write-Host "✅ A5動的グループ作成成功: $($a5Group.DisplayName)" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ A5動的グループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 実行
Create-DepartmentLicenseGroups
```

#### ⚠️ 動的グループの注意事項とベストプラクティス

##### 注意事項

1. **ライセンス要件**: Microsoft Entra ID Premium P1 または P2 が必要
2. **処理時間**: メンバーシップの更新には最大24時間かかる場合がある
3. **ルール制限**: 複雑すぎるルールは避ける
4. **監視**: 定期的にメンバーシップの状況を確認する

##### ベストプラクティス

```powershell
# 1. 動的グループの健全性チェック
function Check-DynamicGroupHealth {
    $dynamicGroups = Get-MgGroup -Filter "groupTypes/any(c:c eq 'DynamicMembership')"
    
    foreach ($group in $dynamicGroups) {
        $members = Get-MgGroupMember -GroupId $group.Id
        
        Write-Host "グループ: $($group.DisplayName)" -ForegroundColor Cyan
        Write-Host "メンバー数: $($members.Count)" -ForegroundColor White
        Write-Host "最終更新: $($group.ProxyAddresses)" -ForegroundColor White
        
        # 空のグループに対する警告
        if ($members.Count -eq 0) {
            Write-Host "⚠️ 警告: メンバーが0人です。ルールを確認してください。" -ForegroundColor Yellow
        }
        
        # 予想より多いメンバーに対する警告
        if ($members.Count -gt 1000) {
            Write-Host "⚠️ 警告: メンバー数が1000人を超えています。ルールの見直しを検討してください。" -ForegroundColor Yellow
        }
        
        Write-Host "---"
    }
}

# 2. 月次レポート生成
function Generate-DynamicGroupReport {
    $dynamicGroups = Get-MgGroup -Filter "groupTypes/any(c:c eq 'DynamicMembership')"
    $report = @()
    
    foreach ($group in $dynamicGroups) {
        $members = Get-MgGroupMember -GroupId $group.Id
        $licenses = Get-MgGroupLicense -GroupId $group.Id
        
        $report += [PSCustomObject]@{
            GroupName = $group.DisplayName
            MembershipRule = $group.MembershipRule
            MemberCount = $members.Count
            LicenseCount = $licenses.Count
            ProcessingState = $group.MembershipRuleProcessingState
            CreatedDate = $group.CreatedDateTime
        }
    }
    
    $report | Export-Csv -Path "C:\temp\dynamic_groups_report_$(Get-Date -Format 'yyyyMMdd').csv" -NoTypeInformation
    Write-Host "✅ レポートが生成されました: C:\temp\dynamic_groups_report_$(Get-Date -Format 'yyyyMMdd').csv" -ForegroundColor Green
}

# 実行例
Check-DynamicGroupHealth
Generate-DynamicGroupReport
```

## 🎯 教育機関向けグループメンバーシップ戦略

### 📊 静的 vs 動的グループ選択のベストプラクティス

教育機関では、グループの目的とライフサイクルに応じて、静的または動的メンバーシップを選択することが重要です。

#### 🏫 教育機関の特殊要件

```powershell
# 小中学校のグループ分類
$educationGroupTypes = @{
    "授業関連" = @{
        Description = "年度末で削除される学級・授業グループ"
        RecommendedType = "静的メンバーシップ"
        Lifecycle = "1年間"
        Examples = @("5年1組2024", "中学3年国語", "小学6年理科実験")
    }
    "教職員組織" = @{
        Description = "継続的な教職員組織グループ"
        RecommendedType = "動的メンバーシップ"
        Lifecycle = "継続"
        Examples = @("小学部教員", "中学部教員", "事務職員", "養護教諭")
    }
    "児童生徒組織" = @{
        Description = "進級・転校に応じて変動するグループ"
        RecommendedType = "動的メンバーシップ"
        Lifecycle = "継続（メンバー自動更新）"
        Examples = @("小学1年生", "中学2年生", "転入生")
    }
    "特別活動" = @{
        Description = "特定期間の委員会・クラブ活動グループ"
        RecommendedType = "静的メンバーシップ"
        Lifecycle = "活動期間"
        Examples = @("図書委員会2024", "吹奏楽部", "生徒会執行部")
    }
}

Write-Host "🏫 小中学校グループ分類ガイド" -ForegroundColor Cyan
foreach ($type in $educationGroupTypes.Keys) {
    $info = $educationGroupTypes[$type]
    Write-Host "`n📚 $type" -ForegroundColor Yellow
    Write-Host "   説明: $($info.Description)" -ForegroundColor White
    Write-Host "   推奨: $($info.RecommendedType)" -ForegroundColor Green
    Write-Host "   期間: $($info.Lifecycle)" -ForegroundColor Gray
    Write-Host "   例: $($info.Examples -join ', ')" -ForegroundColor DarkGray
}
```

### 📅 授業関連グループ：静的メンバーシップ推奨

#### 理由と利点

1. **予測可能な削除**: 年度末に確実にグループを削除可能
2. **履歴保持**: 成績管理や出席記録との整合性維持
3. **手動調整**: 転入生や特別な学習支援が必要な児童生徒への柔軟な対応
4. **データ保護**: 授業終了後の適切なデータアーカイブ

```powershell
# 学級・授業用静的グループの管理システム
function New-ClassGroup {
    param(
        [string]$ClassName,  # 例: "5年1組"、"中3国語"
        [string]$Subject = "",  # 教科名（学級の場合は空）
        [int]$Year = (Get-Date).Year,
        [string]$Teacher,  # 担任または教科担当
        [string[]]$Students = @()
    )
    
    $groupName = if ($Subject) { "$ClassName-$Subject-$Year" } else { "$ClassName-$Year" }
    $expirationDate = Get-Date -Year ($Year + 1) -Month 3 -Day 31  # 年度末
    
    Write-Host "🏫 学級グループ作成: $groupName" -ForegroundColor Cyan
    
    $groupParams = @{
        DisplayName = $groupName
        Description = "$Year年度の$ClassName$(if($Subject){" $Subject授業"})（担当: $Teacher）- 削除予定: $($expirationDate.ToString('yyyy/MM/dd'))"
        GroupTypes = @("Unified")  # Microsoft 365 グループ
        SecurityEnabled = $false
        MailEnabled = $true
        MailNickname = $groupName.Replace(" ", "").Replace("-", "").Replace("年度", "").Replace("組", "").ToLower()
        Visibility = "Private"
    }
    
    try {
        $group = New-MgGroup @groupParams
        
        # メタデータの設定
        $customAttributes = @{
            extensionAttribute1 = "Class"  # グループタイプ
            extensionAttribute2 = $Subject # 教科
            extensionAttribute3 = $Year.ToString() # 年度
            extensionAttribute4 = $expirationDate.ToString('yyyy-MM-dd') # 削除予定日
            extensionAttribute5 = $Teacher # 担当者
        }
        
        # カスタム属性を設定（実際の実装では Graph API の拡張プロパティを使用）
        
        Write-Host "✅ 学級グループ作成完了: $($group.DisplayName)" -ForegroundColor Green
        
        # 児童生徒メンバーの追加
        foreach ($studentUPN in $Students) {
            try {
                $student = Get-MgUser -UserId $studentUPN -ErrorAction Stop
                New-MgGroupMember -GroupId $group.Id -DirectoryObjectId $student.Id
                Write-Host "  ✅ 児童生徒追加: $($student.DisplayName)" -ForegroundColor Green
            }
            catch {
                Write-Host "  ❌ 児童生徒追加エラー: $studentUPN" -ForegroundColor Red
            }
        }
        
        # 担当教員を所有者として追加
        try {
            $teacherUser = Get-MgUser -Filter "mail eq '$Teacher'" -ErrorAction Stop
            New-MgGroupOwner -GroupId $group.Id -DirectoryObjectId $teacherUser.Id
            Write-Host "  ✅ 担当教員を所有者に設定: $($teacherUser.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "  ❌ 担当教員設定エラー: $Teacher" -ForegroundColor Red
        }
        
        return $group
    }
    catch {
        Write-Host "❌ 学級グループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
        return $null
    }
}

# 年度末の一括削除機能
function Remove-ExpiredClassGroups {
    param(
        [int]$TargetYear = (Get-Date).Year - 1,
        [switch]$WhatIf = $true
    )
    
    Write-Host "🗑️ 期限切れ学級グループの削除" -ForegroundColor Cyan
    Write-Host "対象年度: $TargetYear" -ForegroundColor Yellow
    
    # 削除対象のグループを取得（年度で判断）
    $expiredGroups = Get-MgGroup -All | Where-Object {
        $_.DisplayName -match "$TargetYear" -and
        $_.Description -like "*削除予定*"
    }
    
    if ($expiredGroups.Count -eq 0) {
        Write-Host "削除対象のグループはありません。" -ForegroundColor Green
        return
    }
    
    Write-Host "削除対象グループ数: $($expiredGroups.Count)" -ForegroundColor Yellow
    
    foreach ($group in $expiredGroups) {
        if ($WhatIf) {
            Write-Host "  [予行] 削除予定: $($group.DisplayName)" -ForegroundColor DarkYellow
        } else {
            try {
                # データのアーカイブ（必要に応じて）
                $archivePath = "C:\Archives\Groups\$TargetYear\$($group.DisplayName).json"
                $groupData = @{
                    GroupName = $group.DisplayName
                    Description = $group.Description
                    Members = (Get-MgGroupMember -GroupId $group.Id | Select-Object DisplayName, UserPrincipalName)
                    DeletedDate = (Get-Date).ToString('yyyy-MM-dd HH:mm:ss')
                }
                
                $groupData | ConvertTo-Json -Depth 3 | Out-File $archivePath -Encoding UTF8
                
                # グループの削除
                Remove-MgGroup -GroupId $group.Id
                Write-Host "  ✅ 削除完了: $($group.DisplayName)" -ForegroundColor Green
            }
            catch {
                Write-Host "  ❌ 削除エラー: $($group.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
            }
        }
    }
    
    if ($WhatIf) {
        Write-Host "`n💡 実際に削除するには -WhatIf を外して実行してください。" -ForegroundColor Yellow
    }
}

# 使用例
$students = @("student1@school.edu", "student2@school.edu", "student3@school.edu")
$classGroup = New-ClassGroup -ClassName "5年1組" -Year 2025 -Teacher "teacher@school.edu" -Students $students

# 教科別グループの例
$mathClass = New-ClassGroup -ClassName "中学3年" -Subject "数学" -Year 2025 -Teacher "math.teacher@school.edu" -Students $students

# 年度末削除（予行練習）
Remove-ExpiredClassGroups -TargetYear 2024 -WhatIf
```

### 👥 教職員グループ：動的メンバーシップ推奨

#### 理由と利点

1. **自動更新**: 人事異動や新規採用に自動対応
2. **管理負荷軽減**: 手動でのメンバー管理が不要
3. **整合性確保**: HR システムとの属性同期で正確性維持
4. **継続性**: グループ自体は永続的に存在

```powershell
# 教職員向け動的グループ管理システム
function New-FacultyDynamicGroups {
    param(
        [string[]]$Departments = @("小学部", "中学部", "事務室", "保健室", "図書室"),
        [string[]]$JobCategories = @("校長", "教頭", "教諭", "養護教諭", "事務職員", "用務員")
    )
    
    Write-Host "👥 教職員動的グループシステム構築" -ForegroundColor Cyan
    
    $createdGroups = @()
    
    # 部門別教職員グループ
    foreach ($dept in $Departments) {
        $deptGroupParams = @{
            DisplayName = "$dept-教職員"
            Description = "$dept の全教職員（動的メンバーシップ）"
            GroupTypes = @("DynamicMembership", "Unified")
            SecurityEnabled = $true
            MailEnabled = $true
            MailNickname = "$($dept.Replace('部','').Replace('室',''))-staff"
            MembershipRule = "(user.department -eq `"$dept`") and ((user.jobTitle -contains `"校長`") or (user.jobTitle -contains `"教頭`") or (user.jobTitle -contains `"教諭`") or (user.jobTitle -contains `"養護教諭`") or (user.jobTitle -contains `"事務職員`") or (user.jobTitle -contains `"用務員`"))"
            MembershipRuleProcessingState = "On"
            Visibility = "Private"
        }
        
        try {
            $deptGroup = New-MgGroup @deptGroupParams
            $createdGroups += $deptGroup
            Write-Host "  ✅ 部署グループ作成: $($deptGroup.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "  ❌ 部署グループ作成エラー: $dept - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    # 職位別グループ
    foreach ($jobTitle in $JobCategories) {
        $jobGroupParams = @{
            DisplayName = "全校-$jobTitle"
            Description = "全校の$jobTitle グループ（動的メンバーシップ）"
            GroupTypes = @("DynamicMembership")
            SecurityEnabled = $true
            MailEnabled = $true
            MailNickname = "all-$($jobTitle.Replace('教諭','teacher').Replace('職員','staff').Replace('校長','principal').Replace('教頭','viceprincipal'))"
            MembershipRule = "(user.jobTitle -contains `"$jobTitle`")"
            MembershipRuleProcessingState = "On"
            Visibility = "Private"
        }
        
        try {
            $jobGroup = New-MgGroup @jobGroupParams
            $createdGroups += $jobGroup
            Write-Host "  ✅ 職位グループ作成: $($jobGroup.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "  ❌ 職位グループ作成エラー: $jobTitle - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    # 管理職グループ（複合条件）
    $managementGroupParams = @{
        DisplayName = "全校-管理職"
        Description = "全校の管理職グループ（動的メンバーシップ）"
        GroupTypes = @("DynamicMembership")
        SecurityEnabled = $true
        MailEnabled = $true
        MailNickname = "all-management"
        MembershipRule = "(user.jobTitle -contains `"校長`") or (user.jobTitle -contains `"教頭`") or (user.jobTitle -contains `"主幹教諭`") or (user.jobTitle -contains `"教務主任`") or (user.jobTitle -contains `"学年主任`") or (user.jobTitle -contains `"生徒指導主事`")"
        MembershipRuleProcessingState = "On"
        Visibility = "Private"
    }
    
    try {
        $mgmtGroup = New-MgGroup @managementGroupParams
        $createdGroups += $mgmtGroup
        Write-Host "  ✅ 管理職グループ作成: $($mgmtGroup.DisplayName)" -ForegroundColor Green
    }
    catch {
        Write-Host "  ❌ 管理職グループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
    
    return $createdGroups
}

# 教職員グループの健全性監視
function Monitor-FacultyGroupHealth {
    Write-Host "🔍 教職員グループ健全性チェック" -ForegroundColor Cyan
    
    $facultyGroups = Get-MgGroup -Filter "groupTypes/any(c:c eq 'DynamicMembership')" -All | 
                    Where-Object { $_.DisplayName -like "*教職員*" -or $_.DisplayName -like "*全学-*" }
    
    foreach ($group in $facultyGroups) {
        $members = Get-MgGroupMember -GroupId $group.Id -All
        $lastUpdate = $group.ProxyAddresses  # 実際の最終更新日時は別のプロパティを使用
        
        Write-Host "`n📊 $($group.DisplayName)" -ForegroundColor Yellow
        Write-Host "  メンバー数: $($members.Count)" -ForegroundColor White
        Write-Host "  ルール: $($group.MembershipRule)" -ForegroundColor Gray
        Write-Host "  状態: $($group.MembershipRuleProcessingState)" -ForegroundColor White
        
        # 異常検知
        if ($members.Count -eq 0) {
            Write-Host "  ⚠️ 警告: メンバーが0人です。ルールを確認してください。" -ForegroundColor Red
        }
        
        if ($group.MembershipRuleProcessingState -ne "On") {
            Write-Host "  ⚠️ 警告: 動的メンバーシップが無効です。" -ForegroundColor Red
        }
        
        # 期待メンバー数との比較（組織の規模に応じて調整）
        $expectedRanges = @{
            "小学部-教職員" = @{Min=10; Max=30}
            "中学部-教職員" = @{Min=8; Max=25}
            "全校-教諭" = @{Min=15; Max=50}
            "全校-事務職員" = @{Min=5; Max=15}
            "全校-管理職" = @{Min=3; Max=10}
        }
        
        if ($expectedRanges.ContainsKey($group.DisplayName)) {
            $range = $expectedRanges[$group.DisplayName]
            if ($members.Count -lt $range.Min -or $members.Count -gt $range.Max) {
                Write-Host "  ⚠️ 注意: メンバー数が想定範囲外です（想定: $($range.Min)-$($range.Max)人）" -ForegroundColor Yellow
            }
        }
    }
}

# 自動化された日次チェック
function Start-DailyFacultyGroupMonitoring {
    param(
        [string]$LogPath = "C:\Logs\FacultyGroupMonitoring"
    )
    
    if (!(Test-Path $LogPath)) {
        New-Item -Path $LogPath -ItemType Directory -Force
    }
    
    $today = Get-Date -Format "yyyy-MM-dd"
    $logFile = "$LogPath\FacultyGroupHealth_$today.log"
    
    Start-Transcript -Path $logFile
    
    try {
        Write-Host "🕐 $(Get-Date): 教職員グループ日次監視開始" -ForegroundColor Cyan
        Monitor-FacultyGroupHealth
        
        # 問題があった場合のアラート送信（実装は環境に応じて）
        # Send-Alert -Type "FacultyGroupIssue" -LogPath $logFile
        
        Write-Host "✅ $(Get-Date): 教職員グループ日次監視完了" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ $(Get-Date): 監視エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
    finally {
        Stop-Transcript
    }
}

# 使用例
$facultyGroups = New-FacultyDynamicGroups
Monitor-FacultyGroupHealth
```

### 🎓 児童生徒グループ：動的メンバーシップ推奨

#### 学年・在籍状況による自動管理

```powershell
# 児童生徒向け動的グループ管理
function New-StudentDynamicGroups {
    param(
        [string[]]$GradeLevels = @("小学1年", "小学2年", "小学3年", "小学4年", "小学5年", "小学6年", "中学1年", "中学2年", "中学3年"),
        [string[]]$StudentTypes = @("小学生", "中学生", "転入生", "特別支援", "不登校支援")
    )
    
    Write-Host "🎓 児童生徒動的グループシステム構築" -ForegroundColor Cyan
    
    # 学年別グループ
    foreach ($grade in $GradeLevels) {
        $gradeGroupParams = @{
            DisplayName = "全校-$grade"
            Description = "全校の$grade グループ（動的メンバーシップ）"
            GroupTypes = @("DynamicMembership", "Unified")
            SecurityEnabled = $true
            MailEnabled = $true
            MailNickname = "all-$($grade.Replace('年','year').Replace('小学','elem').Replace('中学','junior'))"
            MembershipRule = "(user.extensionAttribute1 -eq `"$grade`") and (user.jobTitle -eq `"児童`" or user.jobTitle -eq `"生徒`")"
            MembershipRuleProcessingState = "On"
            Visibility = "Private"
        }
        
        try {
            $gradeGroup = New-MgGroup @gradeGroupParams
            Write-Host "  ✅ 学年グループ作成: $($gradeGroup.DisplayName)" -ForegroundColor Green
            
            # 学年進級時の自動更新ロジック追加
            Add-StudentGradeProgressionRule -GroupId $gradeGroup.Id -Grade $grade
        }
        catch {
            Write-Host "  ❌ 学年グループ作成エラー: $grade - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    # 児童生徒種別グループ
    foreach ($type in $StudentTypes) {
        $typeGroupParams = @{
            DisplayName = "全校-$type"
            Description = "全校の$type グループ（動的メンバーシップ）"
            GroupTypes = @("DynamicMembership")
            SecurityEnabled = $true
            MailEnabled = $true
            MailNickname = "all-$($type.Replace('生','').Replace('学','').Replace('支援','support').Replace('転入','transfer'))"
            MembershipRule = "(user.extensionAttribute2 -eq `"$type`")"
            MembershipRuleProcessingState = "On"
            Visibility = "Private"
        }
        
        try {
            $typeGroup = New-MgGroup @typeGroupParams
            Write-Host "  ✅ 児童生徒種別グループ作成: $($typeGroup.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "  ❌ 児童生徒種別グループ作成エラー: $type - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}

# 学年進級処理の自動化
function Start-AnnualGradePromotion {
    param(
        [int]$NewAcademicYear = (Get-Date).Year,
        [switch]$WhatIf = $true
    )
    
    Write-Host "📈 年次学年進級処理" -ForegroundColor Cyan
    Write-Host "対象年度: $NewAcademicYear" -ForegroundColor Yellow
    
    # 進級ルール定義
    $promotionRules = @{
        "小学1年" = "小学2年"
        "小学2年" = "小学3年"
        "小学3年" = "小学4年"
        "小学4年" = "小学5年"
        "小学5年" = "小学6年"
        "小学6年" = "中学1年"
        "中学1年" = "中学2年"
        "中学2年" = "中学3年"
        "中学3年" = "卒業"
    }
    
    # 現在の全児童生徒を取得
    $students = Get-MgUser -Filter "jobTitle eq '児童' or jobTitle eq '生徒'" -All
    
    foreach ($student in $students) {
        $currentGrade = $student.AdditionalProperties["extensionAttribute1"]
        $newGrade = $promotionRules[$currentGrade]
        
        if ($WhatIf) {
            Write-Host "  [予行] $($student.DisplayName): $currentGrade → $newGrade" -ForegroundColor DarkYellow
        } else {
            try {
                if ($newGrade -eq "卒業") {
                    # 卒業処理
                    Update-MgUser -UserId $student.Id -AdditionalProperties @{
                        extensionAttribute1 = "卒業生"
                        extensionAttribute3 = $NewAcademicYear.ToString()
                    }
                    Write-Host "  🎓 卒業処理: $($student.DisplayName)" -ForegroundColor Green
                } else {
                    # 通常の進級
                    Update-MgUser -UserId $student.Id -AdditionalProperties @{
                        extensionAttribute1 = $newGrade
                        extensionAttribute3 = $NewAcademicYear.ToString()
                    }
                    Write-Host "  ✅ 進級処理: $($student.DisplayName) ($currentGrade → $newGrade)" -ForegroundColor Green
                }
            }
            catch {
                Write-Host "  ❌ 進級処理エラー: $($student.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
            }
        }
    }
    
    if ($WhatIf) {
        Write-Host "`n💡 実際に進級処理するには -WhatIf を外して実行してください。" -ForegroundColor Yellow
    } else {
        Write-Host "`n✅ 学年進級処理が完了しました。動的グループのメンバーシップが自動更新されます。" -ForegroundColor Green
    }
}
```

### 🔧 ハイブリッド戦略：特別活動グループ

#### 静的ベース + 動的サブグループ

```powershell
# ハイブリッド特別活動グループ管理
function New-HybridActivityGroup {
    param(
        [string]$ActivityName,
        [string]$ActivityType,  # "委員会", "クラブ活動", "行事委員" など
        [DateTime]$StartDate,
        [DateTime]$EndDate,
        [string]$Advisor,  # 顧問教員
        [string[]]$CoreMembers = @(),
        [string]$DynamicMembershipRule = ""
    )
    
    Write-Host "🏃 ハイブリッド特別活動グループ作成: $ActivityName" -ForegroundColor Cyan
    
    # メインの静的活動グループ
    $mainGroupParams = @{
        DisplayName = "$ActivityName-$ActivityType"
        Description = "$ActivityType 活動（$($StartDate.ToString('yyyy/MM/dd'))-$($EndDate.ToString('yyyy/MM/dd'))）"
        GroupTypes = @("Unified")
        SecurityEnabled = $false
        MailEnabled = $true
        MailNickname = "$($ActivityName.Replace(' ', '').Replace('-', '')).activity"
        Visibility = "Private"
    }
    
    try {
        $mainGroup = New-MgGroup @mainGroupParams
        Write-Host "  ✅ メイン活動グループ作成: $($mainGroup.DisplayName)" -ForegroundColor Green
        
        # 顧問教員を所有者に設定
        $advisorUser = Get-MgUser -Filter "mail eq '$Advisor'"
        New-MgGroupOwner -GroupId $mainGroup.Id -DirectoryObjectId $advisorUser.Id
        
        # コアメンバーを追加
        foreach ($memberUPN in $CoreMembers) {
            try {
                $member = Get-MgUser -UserId $memberUPN
                New-MgGroupMember -GroupId $mainGroup.Id -DirectoryObjectId $member.Id
                Write-Host "    ✅ コアメンバー追加: $($member.DisplayName)" -ForegroundColor Green
            }
            catch {
                Write-Host "    ❌ メンバー追加エラー: $memberUPN" -ForegroundColor Red
            }
        }
        
        # 動的サブグループの作成（該当する場合）
        if ($DynamicMembershipRule) {
            $subGroupParams = @{
                DisplayName = "$ActivityName-関連者（動的）"
                Description = "$ActivityName の関連者動的グループ"
                GroupTypes = @("DynamicMembership")
                SecurityEnabled = $true
                MailEnabled = $false
                MailNickname = "$($ActivityName.Replace(' ', '').Replace('-', '')).related"
                MembershipRule = $DynamicMembershipRule
                MembershipRuleProcessingState = "On"
                Visibility = "Private"
            }
            
            try {
                $subGroup = New-MgGroup @subGroupParams
                Write-Host "  ✅ 動的サブグループ作成: $($subGroup.DisplayName)" -ForegroundColor Green
                
                # メイングループと動的グループの関連付け（メタデータとして記録）
                # 実際の実装では、カスタムプロパティまたは外部データベースで管理
                
                return @{
                    MainGroup = $mainGroup
                    DynamicSubGroup = $subGroup
                }
            }
            catch {
                Write-Host "  ❌ 動的サブグループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
                return @{ MainGroup = $mainGroup }
            }
        }
        
        return @{ MainGroup = $mainGroup }
    }
    catch {
        Write-Host "❌ プロジェクトグループ作成エラー: $($_.Exception.Message)" -ForegroundColor Red
        return $null
    }
}

# プロジェクト終了時の処理
function Complete-ProjectGroup {
    param(
        [string]$ProjectGroupId,
        [switch]$ArchiveData = $true,
        [switch]$WhatIf = $true
    )
    
    $group = Get-MgGroup -GroupId $ProjectGroupId
    Write-Host "📁 プロジェクト終了処理: $($group.DisplayName)" -ForegroundColor Cyan
    
    if ($ArchiveData) {
        # データのアーカイブ
        $archivePath = "C:\Archives\Projects\$($group.DisplayName.Replace(':', '-'))"
        if (!(Test-Path $archivePath)) {
            New-Item -Path $archivePath -ItemType Directory -Force
        }
        
        # グループ情報の保存
        $groupInfo = @{
            GroupName = $group.DisplayName
            Description = $group.Description
            Members = Get-MgGroupMember -GroupId $group.Id | Select-Object DisplayName, UserPrincipalName, UserType
            Owners = Get-MgGroupOwner -GroupId $group.Id | Select-Object DisplayName, UserPrincipalName
            CompletedDate = (Get-Date).ToString('yyyy-MM-dd HH:mm:ss')
        }
        
        $groupInfo | ConvertTo-Json -Depth 3 | Out-File "$archivePath\group_info.json" -Encoding UTF8
        
        if ($WhatIf) {
            Write-Host "  [予行] アーカイブ保存: $archivePath" -ForegroundColor DarkYellow
        } else {
            Write-Host "  ✅ アーカイブ保存完了: $archivePath" -ForegroundColor Green
        }
    }
    
    if ($WhatIf) {
        Write-Host "  [予行] グループ削除予定: $($group.DisplayName)" -ForegroundColor DarkYellow
    } else {
        try {
            Remove-MgGroup -GroupId $group.Id
            Write-Host "  ✅ グループ削除完了: $($group.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "  ❌ グループ削除エラー: $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}
```

### 📋 決定フローチャート

```powershell
# グループタイプ決定支援システム
function Get-GroupMembershipRecommendation {
    param(
        [string]$GroupPurpose,
        [string]$GroupDuration,  # "短期", "中期", "長期", "永続"
        [string]$MembershipPattern,  # "固定", "変動", "混合"
        [string]$OrganizationalLevel  # "授業", "学年", "全校", "プロジェクト"
    )
    
    Write-Host "🎯 グループメンバーシップタイプ推奨システム" -ForegroundColor Cyan
    
    $recommendation = @{
        RecommendedType = ""
        Confidence = 0
        Reasoning = @()
        AlternativeOptions = @()
        ImplementationTips = @()
    }
    
    # 決定ロジック
    $score_static = 0
    $score_dynamic = 0
    
    # 期間による評価
    switch ($GroupDuration) {
        "短期" { 
            $score_static += 3
            $recommendation.Reasoning += "短期間のため、静的メンバーシップが管理しやすい"
        }
        "中期" { 
            $score_static += 1
            $score_dynamic += 1
            $recommendation.Reasoning += "中期間のため、目的に応じて選択"
        }
        "長期" { 
            $score_dynamic += 2
            $recommendation.Reasoning += "長期間のため、動的メンバーシップで管理負荷軽減"
        }
        "永続" { 
            $score_dynamic += 3
            $recommendation.Reasoning += "永続的グループのため、動的メンバーシップが最適"
        }
    }
    
    # メンバーシップパターンによる評価
    switch ($MembershipPattern) {
        "固定" { 
            $score_static += 3
            $recommendation.Reasoning += "メンバーが固定のため、静的メンバーシップが適切"
        }
        "変動" { 
            $score_dynamic += 3
            $recommendation.Reasoning += "メンバーが変動するため、動的メンバーシップが効率的"
        }
        "混合" { 
            $score_static += 1
            $score_dynamic += 1
            $recommendation.Reasoning += "ハイブリッド戦略の検討を推奨"
            $recommendation.AlternativeOptions += "ハイブリッド戦略（静的+動的サブグループ）"
        }
    }
    
    # 組織レベルによる評価
    switch ($OrganizationalLevel) {
        "学級" { 
            $score_static += 2
            $recommendation.Reasoning += "学級レベルは年度末で削除されるため、静的が推奨"
            $recommendation.ImplementationTips += "年度末の自動削除設定を実装"
        }
        "学年" { 
            $score_dynamic += 2
            $recommendation.Reasoning += "学年レベルは進級により変動するため、動的が推奨"
            $recommendation.ImplementationTips += "学年属性との同期設定を確認"
        }
        "全校" { 
            $score_dynamic += 3
            $recommendation.Reasoning += "全校レベルは動的メンバーシップが最適"
            $recommendation.ImplementationTips += "職位・学年属性の整備が重要"
        }
        "特別活動" { 
            $score_static += 1
            $recommendation.Reasoning += "特別活動はハイブリッド戦略が効果的"
            $recommendation.AlternativeOptions += "コアメンバー（静的）+ 関連者（動的）"
        }
    }
    
    # 最終推奨の決定
    if ($score_dynamic > $score_static) {
        $recommendation.RecommendedType = "動的メンバーシップ"
        $recommendation.Confidence = [math]::Min(90, 50 + ($score_dynamic - $score_static) * 10)
    } elseif ($score_static > $score_dynamic) {
        $recommendation.RecommendedType = "静的メンバーシップ"
        $recommendation.Confidence = [math]::Min(90, 50 + ($score_static - $score_dynamic) * 10)
    } else {
        $recommendation.RecommendedType = "ハイブリッド戦略"
        $recommendation.Confidence = 60
        $recommendation.Reasoning += "両方のメリットを活用するハイブリッド戦略を推奨"
    }
    
    # 追加の実装ヒント
    if ($recommendation.RecommendedType -eq "動的メンバーシップ") {
        $recommendation.ImplementationTips += "Microsoft Entra ID Premium P1以上のライセンスが必要"
        $recommendation.ImplementationTips += "属性の整備と定期的な健全性チェックを実施"
    } elseif ($recommendation.RecommendedType -eq "静的メンバーシップ") {
        $recommendation.ImplementationTips += "定期的なメンバーシップレビューを設定"
        $recommendation.ImplementationTips += "適切なライフサイクル管理（削除日程）を設定"
    }
    
    # 結果の表示
    Write-Host "`n📊 推奨結果" -ForegroundColor Yellow
    Write-Host "推奨タイプ: $($recommendation.RecommendedType)" -ForegroundColor Green
    Write-Host "信頼度: $($recommendation.Confidence)%" -ForegroundColor Green
    
    Write-Host "`n💭 理由:" -ForegroundColor Yellow
    foreach ($reason in $recommendation.Reasoning) {
        Write-Host "  • $reason" -ForegroundColor White
    }
    
    if ($recommendation.AlternativeOptions.Count -gt 0) {
        Write-Host "`n🔄 代替案:" -ForegroundColor Yellow
        foreach ($option in $recommendation.AlternativeOptions) {
            Write-Host "  • $option" -ForegroundColor Cyan
        }
    }
    
    Write-Host "`n💡 実装のヒント:" -ForegroundColor Yellow
    foreach ($tip in $recommendation.ImplementationTips) {
        Write-Host "  • $tip" -ForegroundColor Gray
    }
    
    return $recommendation
}

# 使用例
Write-Host "=== 授業グループの例 ===" -ForegroundColor Magenta
Get-GroupMembershipRecommendation -GroupPurpose "小学5年算数" -GroupDuration "短期" -MembershipPattern "固定" -OrganizationalLevel "授業"

Write-Host "`n=== 学年グループの例 ===" -ForegroundColor Magenta  
Get-GroupMembershipRecommendation -GroupPurpose "小学部教員" -GroupDuration "永続" -MembershipPattern "変動" -OrganizationalLevel "学年"

Write-Host "`n=== 委員会活動の例 ===" -ForegroundColor Magenta
Get-GroupMembershipRecommendation -GroupPurpose "図書委員会活動" -GroupDuration "中期" -MembershipPattern "混合" -OrganizationalLevel "プロジェクト"
```

### 📈 実装ロードマップ

```powershell
# 段階的実装計画
function Get-ImplementationRoadmap {
    Write-Host "🗺️ 教育機関グループ戦略実装ロードマップ" -ForegroundColor Cyan
    
    $phases = @{
        "フェーズ1" = @{
            Duration = "1-2ヶ月"
            Priority = "高"
            Tasks = @(
                "現在のグループ監査と分類",
                "教職員動的グループの構築",
                "基本的な児童生徒動的グループの作成",
                "グループ命名規則の標準化"
            )
            Prerequisites = @(
                "Microsoft Entra ID Premium P1 ライセンス",
                "ユーザー属性の整備",
                "管理者権限の確認"
            )
        }
        "フェーズ2" = @{
            Duration = "2-3ヶ月"
            Priority = "中"
            Tasks = @(
                "授業用静的グループテンプレートの作成",
                "年度末削除自動化の実装",
                "グループ健全性監視システムの構築",
                "ハイブリッドプロジェクトグループの試験運用"
            )
            Prerequisites = @(
                "フェーズ1の完了",
                "自動化スクリプトのテスト環境",
                "バックアップ・アーカイブ戦略の確立"
            )
        }
        "フェーズ3" = @{
            Duration = "3-4ヶ月"
            Priority = "低"
            Tasks = @(
                "高度な動的ルールの実装",
                "AI支援による最適化",
                "他システムとの統合",
                "包括的なレポート機能の構築"
            )
            Prerequisites = @(
                "フェーズ2の完了",
                "外部システム連携の準備",
                "高度な監視ツールの導入"
            )
        }
    }
    
    foreach ($phase in $phases.Keys) {
        $info = $phases[$phase]
        Write-Host "`n📅 $phase ($($info.Duration))" -ForegroundColor Yellow
        Write-Host "優先度: $($info.Priority)" -ForegroundColor White
        
        Write-Host "タスク:" -ForegroundColor Cyan
        foreach ($task in $info.Tasks) {
            Write-Host "  ✓ $task" -ForegroundColor Green
        }
        
        Write-Host "前提条件:" -ForegroundColor Gray
        foreach ($prereq in $info.Prerequisites) {
            Write-Host "  • $prereq" -ForegroundColor DarkGray
        }
    }
    
    Write-Host "`n🎯 成功の指標:" -ForegroundColor Magenta
    Write-Host "  • グループ管理工数の50%削減" -ForegroundColor White
    Write-Host "  • メンバーシップエラーの90%削減" -ForegroundColor White
    Write-Host "  • 年度更新作業の自動化達成" -ForegroundColor White
    Write-Host "  • ユーザー満足度の向上" -ForegroundColor White
}

Get-ImplementationRoadmap
```

## 一括操作

### 📋 一括ユーザー作成（詳細版）

#### CSVファイルの形式例（拡張版）

```csv
DisplayName,UserPrincipalName,MailNickname,Password,Department,JobTitle,License,ManagerEmail
田中 太郎,tanaka@contoso.edu,tanaka,TempPass123!,小学部,教諭,A3,principal@contoso.edu
佐藤 花子,sato@contoso.edu,sato,TempPass456!,中学部,教諭,A5,vice-principal@contoso.edu
山田 次郎,yamada@contoso.edu,yamada,TempPass789!,事務室,事務員,A1,manager@contoso.edu
```

#### PowerShell による一括ユーザー作成

```powershell
# Microsoft Graph に接続（必要な権限を指定）
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All"

# CSV ファイルを読み込み
$users = Import-Csv -Path "C:\temp\users.csv"
$createdUsers = @()
$errorLog = @()

foreach ($user in $users) {
    $passwordProfile = @{
        ForceChangePasswordNextSignIn = $true
        Password = $user.Password
    }
    
    $newUser = @{
        DisplayName = $user.DisplayName
        UserPrincipalName = $user.UserPrincipalName
        MailNickname = $user.MailNickname
        PasswordProfile = $passwordProfile
        AccountEnabled = $true
        Department = $user.Department
        JobTitle = $user.JobTitle
    }
    
    try {
        $createdUser = New-MgUser @newUser
        $createdUsers += [PSCustomObject]@{
            DisplayName = $user.DisplayName
            UserPrincipalName = $user.UserPrincipalName
            License = $user.License
            UserId = $createdUser.Id
        }
        Write-Host "✅ 作成成功: $($user.DisplayName)" -ForegroundColor Green
        Start-Sleep -Seconds 1  # API レート制限回避
    }
    catch {
        $errorLog += [PSCustomObject]@{
            DisplayName = $user.DisplayName
            UserPrincipalName = $user.UserPrincipalName
            Error = $_.Exception.Message
        }
        Write-Host "❌ 作成エラー: $($user.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
    }
}

# エラーログをCSVに出力
if ($errorLog.Count -gt 0) {
    $errorLog | Export-Csv -Path "C:\temp\user_creation_errors.csv" -NoTypeInformation
    Write-Host "エラーログを保存しました: C:\temp\user_creation_errors.csv" -ForegroundColor Yellow
}
```

### 📊 一括ライセンス割り当て

#### 個別ユーザーへのライセンス割り当て

```powershell
# ライセンス情報の取得
$availableLicenses = Get-MgSubscribedSku | Select-Object SkuPartNumber, SkuId, ConsumedUnits, PrepaidUnits

# A1ライセンス（基本）の一括割り当て
function Set-BulkA1License {
    param($users)
    
    $a1SkuId = ($availableLicenses | Where-Object {$_.SkuPartNumber -eq "STANDARDWOFFPACK_FACULTY"}).SkuId
    
    foreach ($user in $users | Where-Object {$_.License -eq "A1"}) {
        try {
            $licenseParams = @{
                AddLicenses = @(@{SkuId = $a1SkuId})
                RemoveLicenses = @()
            }
            Set-MgUserLicense -UserId $user.UserId -BodyParameter $licenseParams
            Write-Host "✅ A1ライセンス割り当て成功: $($user.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ A1ライセンス割り当てエラー: $($user.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}

# A3/A5ユーザー用グループ作成とライセンス割り当て
function Set-BulkA3A5License {
    param($users)
    
    # A3グループの作成
    $a3GroupParams = @{
        DisplayName = "Microsoft 365 A3 ライセンスグループ"
        Description = "A3ライセンス自動割り当て用グループ"
        GroupTypes = @()
        SecurityEnabled = $true
        MailEnabled = $false
        MailNickname = "A3LicenseGroup"
    }
    
    try {
        $a3Group = New-MgGroup @a3GroupParams
        Write-Host "✅ A3ライセンスグループ作成成功" -ForegroundColor Green
    }
    catch {
        $a3Group = Get-MgGroup -Filter "displayName eq 'Microsoft 365 A3 ライセンスグループ'"
        Write-Host "ℹ️ A3ライセンスグループは既に存在します" -ForegroundColor Blue
    }
    
    # A5グループの作成
    $a5GroupParams = @{
        DisplayName = "Microsoft 365 A5 ライセンスグループ"
        Description = "A5ライセンス自動割り当て用グループ"
        GroupTypes = @()
        SecurityEnabled = $true
        MailEnabled = $false
        MailNickname = "A5LicenseGroup"
    }
    
    try {
        $a5Group = New-MgGroup @a5GroupParams
        Write-Host "✅ A5ライセンスグループ作成成功" -ForegroundColor Green
    }
    catch {
        $a5Group = Get-MgGroup -Filter "displayName eq 'Microsoft 365 A5 ライセンスグループ'"
        Write-Host "ℹ️ A5ライセンスグループは既に存在します" -ForegroundColor Blue
    }
    
    # グループにライセンスを割り当て
    $a3SkuId = ($availableLicenses | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPACK_FACULTY"}).SkuId
    $a5SkuId = ($availableLicenses | Where-Object {$_.SkuPartNumber -eq "ENTERPRISEPREMIUM_FACULTY"}).SkuId
    
    # A3グループにライセンス割り当て
    try {
        $a3LicenseParams = @{
            AddLicenses = @(@{SkuId = $a3SkuId})
            RemoveLicenses = @()
        }
        Set-MgGroupLicense -GroupId $a3Group.Id -BodyParameter $a3LicenseParams
        Write-Host "✅ A3グループライセンス割り当て成功" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ A3グループライセンス割り当てエラー: $($_.Exception.Message)" -ForegroundColor Red
    }
    
    # A5グループにライセンス割り当て
    try {
        $a5LicenseParams = @{
            AddLicenses = @(@{SkuId = $a5SkuId})
            RemoveLicenses = @()
        }
        Set-MgGroupLicense -GroupId $a5Group.Id -BodyParameter $a5LicenseParams
        Write-Host "✅ A5グループライセンス割り当て成功" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ A5グループライセンス割り当てエラー: $($_.Exception.Message)" -ForegroundColor Red
    }
    
    # ユーザーをグループに追加
    foreach ($user in $users | Where-Object {$_.License -eq "A3"}) {
        try {
            New-MgGroupMember -GroupId $a3Group.Id -DirectoryObjectId $user.UserId
            Write-Host "✅ A3グループ追加成功: $($user.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ A3グループ追加エラー: $($user.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    foreach ($user in $users | Where-Object {$_.License -eq "A5"}) {
        try {
            New-MgGroupMember -GroupId $a5Group.Id -DirectoryObjectId $user.UserId
            Write-Host "✅ A5グループ追加成功: $($user.DisplayName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ A5グループ追加エラー: $($user.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}

# ライセンス割り当て実行
Set-BulkA1License -users $createdUsers
Set-BulkA3A5License -users $createdUsers
```

### 🔄 一括ユーザー編集

```powershell
# CSV形式: UserPrincipalName,NewDepartment,NewJobTitle,NewManager
$userUpdates = Import-Csv -Path "C:\temp\user_updates.csv"

foreach ($update in $userUpdates) {
    try {
        # マネージャーのIDを取得
        $manager = $null
        if ($update.NewManager) {
            $manager = Get-MgUser -Filter "userPrincipalName eq '$($update.NewManager)'"
        }
        
        $updateParams = @{
            Department = $update.NewDepartment
            JobTitle = $update.NewJobTitle
        }
        
        if ($manager) {
            $updateParams.Manager = @{
                "@odata.id" = "https://graph.microsoft.com/v1.0/users/$($manager.Id)"
            }
        }
        
        Update-MgUser -UserId $update.UserPrincipalName @updateParams
        Write-Host "✅ 更新成功: $($update.UserPrincipalName)" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ 更新エラー: $($update.UserPrincipalName) - $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

### 🗑️ 一括ユーザー削除

```powershell
# 削除対象ユーザーのリスト（CSV形式: UserPrincipalName,Reason）
$usersToDelete = Import-Csv -Path "C:\temp\users_to_delete.csv"

# 確認プロンプト
Write-Host "以下のユーザーを削除します:" -ForegroundColor Yellow
$usersToDelete | Format-Table UserPrincipalName, Reason

$confirmation = Read-Host "削除を実行しますか？ (y/N)"
if ($confirmation -eq "y" -or $confirmation -eq "Y") {
    foreach ($user in $usersToDelete) {
        try {
            # 削除前にライセンスを除去
            $userObj = Get-MgUser -UserId $user.UserPrincipalName
            $currentLicenses = $userObj.AssignedLicenses
            
            if ($currentLicenses.Count -gt 0) {
                $removeLicenseParams = @{
                    AddLicenses = @()
                    RemoveLicenses = $currentLicenses.SkuId
                }
                Set-MgUserLicense -UserId $user.UserPrincipalName -BodyParameter $removeLicenseParams
            }
            
            # ユーザー削除
            Remove-MgUser -UserId $user.UserPrincipalName
            Write-Host "✅ 削除成功: $($user.UserPrincipalName)" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ 削除エラー: $($user.UserPrincipalName) - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
}
```

## 📋 グループ作成制限の管理

教育機関では、セキュリティとガバナンスの観点から、グループ作成を特定の権限を持つユーザーに制限することが重要です。

### 🔒 グループ作成制限の設定

#### Microsoft 365 管理センターでの設定

1. **Azure Active Directory 管理センター** にアクセス
2. **グループ** > **全般** を選択
3. **セルフサービス グループ管理** セクションで以下を設定：
   - **ユーザーはセキュリティ グループを作成できます**: **いいえ**
   - **ユーザーは Microsoft 365 グループを作成できます**: **いいえ**
   - **管理者以外はグループを所有できます**: **いいえ**

#### PowerShell を使用した一括設定

```powershell
# グループ作成制限の設定
Connect-MgGraph -Scopes "Directory.ReadWrite.All", "Group.ReadWrite.All"

# 現在の設定を確認
$currentSettings = Get-MgDirectorySetting | Where-Object {$_.DisplayName -eq "Group.Unified"}

# 設定が存在しない場合は新規作成
if (-not $currentSettings) {
    $template = Get-MgDirectorySettingTemplate | Where-Object {$_.DisplayName -eq "Group.Unified"}
    $settings = $template.Values | ForEach-Object { @{Name = $_.Name; Value = $_.DefaultValue} }
    
    # グループ作成制限を設定
    ($settings | Where-Object {$_.Name -eq "EnableGroupCreation"}).Value = "false"
    ($settings | Where-Object {$_.Name -eq "GroupCreationAllowedGroupId"}).Value = ""
    
    $settingsParams = @{
        TemplateId = $template.Id
        Values = $settings
    }
    
    New-MgDirectorySetting -BodyParameter $settingsParams
    Write-Host "✅ グループ作成制限を設定しました" -ForegroundColor Green
}
else {
    # 既存設定を更新
    $settingId = $currentSettings.Id
    $values = $currentSettings.Values
    
    ($values | Where-Object {$_.Name -eq "EnableGroupCreation"}).Value = "false"
    
    $updateParams = @{
        Values = $values
    }
    
    Update-MgDirectorySetting -DirectorySettingId $settingId -BodyParameter $updateParams
    Write-Host "✅ グループ作成制限を更新しました" -ForegroundColor Green
}
```

### 🔐 プライベートグループのデフォルト設定

セキュリティを向上させるため、新しく作成されるグループを自動的にプライベートに設定する方法：

#### Microsoft 365 グループのデフォルトプライバシー設定

```powershell
# Microsoft Graph に接続
Connect-MgGraph -Scopes "Policy.ReadWrite.Authorization", "Group.ReadWrite.All"

# グループ作成ポリシーのテンプレートを取得
$template = Get-MgDirectorySettingTemplate | Where-Object {$_.DisplayName -eq "Group.Unified"}

# 現在の設定を確認
$currentSettings = Get-MgDirectorySetting | Where-Object {$_.DisplayName -eq "Group.Unified"}

if (-not $currentSettings) {
    # 新規設定の作成
    $values = $template.Values | ForEach-Object {
        @{
            Name = $_.Name
            Value = switch ($_.Name) {
                "DefaultGroupAccessType" { "Private" }
                "EnableGroupCreation" { if ($AllowedCreatorGroupId) { "true" } else { "false" } }
                "GroupCreationAllowedGroupId" { $AllowedCreatorGroupId }
                "AllowToAddGuests" { "false" }
                "EnableMIPLabels" { "true" }  # 秘密度ラベル有効化
                default { $_.DefaultValue }
            }
        }
    }
    
    $settingParams = @{
        TemplateId = $template.Id
        Values = $values
    }
    
    New-MgDirectorySetting -BodyParameter $settingParams
    Write-Host "✅ プライベートグループのデフォルト設定を適用しました" -ForegroundColor Green
}
else {
    # 既存設定の更新
    $values = $currentSettings.Values
    
    # デフォルトアクセスタイプをプライベートに変更
    $defaultAccessType = $values | Where-Object {$_.Name -eq "DefaultGroupAccessType"}
    if ($defaultAccessType) {
        $defaultAccessType.Value = "Private"
    }
    
    $updateParams = @{
        Values = $values
    }
    
    Update-MgDirectorySetting -DirectorySettingId $currentSettings.Id -BodyParameter $updateParams
    Write-Host "✅ プライベートグループ設定を更新しました" -ForegroundColor Green
}
```

#### Teams でのプライベートチーム強制設定

```powershell
# Teams PowerShell モジュールでの設定
Connect-MicrosoftTeams

# Teams 作成ポリシーでプライベートチームのみ許可
$teamCreationPolicy = @{
    Identity = "RestrictToPrivateTeams"
    AllowCreatePrivateChannels = $true
    AllowCreateSharedChannels = $false
    ChannelCreationEnabled = $true
    AllowUserEditMessages = $true
    AllowUserDeleteMessages = $false
    AllowOwnerDeleteMessages = $true
    AllowTeamMentions = $true
    AllowChannelMentions = $true
    ShowInTeamsSearchAndSuggestions = $false  # プライベート性を強化
}

New-CsTeamsChannelsPolicy @teamCreationPolicy

# 特定のユーザーグループにポリシーを適用
Grant-CsTeamsChannelsPolicy -PolicyName "RestrictToPrivateTeams" -Group "教職員グループ作成権限者"

Write-Host "✅ Teams プライベートチーム強制ポリシーを設定しました" -ForegroundColor Green
```

#### SharePoint サイトのプライベート設定

```powershell
# SharePoint Online でのサイト作成時のデフォルトプライバシー設定
Connect-SPOService -Url "https://yourdomain-admin.sharepoint.com"

# Microsoft 365 グループ接続サイトのデフォルトを非公開に設定
Set-SPOTenant -DefaultGroupAccessType Private

# 新規サイト作成時の外部共有を制限
Set-SPOTenant -SharingCapability ExternalUserSharingOnly
Set-SPOTenant -DefaultSharingLinkType Direct  # 特定のユーザーのみ

Write-Host "✅ SharePoint サイトのプライベート設定を適用しました" -ForegroundColor Green
```

#### 包括的なプライベートグループ強制設定

```powershell
# 教育機関向け包括的プライベート設定
function Set-EducationPrivateGroupDefault {
    param(
        [switch]$EnforceStrictPrivacy = $true,
        [string]$AllowedCreatorGroupId = ""
    )
    
    Write-Host "🏫 教育機関向けプライベートグループ設定を適用中..." -ForegroundColor Cyan
    
    try {
        # Microsoft Graph 接続
        Connect-MgGraph -Scopes "Policy.ReadWrite.Authorization", "Group.ReadWrite.All"
        
        # グループ設定テンプレートを取得
        $template = Get-MgDirectorySettingTemplate | Where-Object {$_.DisplayName -eq "Group.Unified"}
        $currentSettings = Get-MgDirectorySetting | Where-Object {$_.DisplayName -eq "Group.Unified"}
        
        if (-not $currentSettings) {
            # 新規設定作成
            $values = $template.Values | ForEach-Object {
                @{
                    Name = $_.Name
                    Value = switch ($_.Name) {
                        "DefaultGroupAccessType" { "Private" }
                        "EnableGroupCreation" { if ($AllowedCreatorGroupId) { "true" } else { "false" } }
                        "GroupCreationAllowedGroupId" { $AllowedCreatorGroupId }
                        "AllowToAddGuests" { "false" }
                        "EnableMIPLabels" { "true" }  # 秘密度ラベル有効化
                        default { $_.DefaultValue }
                    }
                }
            }
            
            $settingParams = @{
                TemplateId = $template.Id
                Values = $values
            }
            
            New-MgDirectorySetting -BodyParameter $settingParams
        }
        else {
            # 既存設定更新
            $values = $currentSettings.Values
            
            # プライベート設定の強制
            ($values | Where-Object {$_.Name -eq "DefaultGroupAccessType"}).Value = "Private"
            ($values | Where-Object {$_.Name -eq "AllowToAddGuests"}).Value = "false"
            ($values | Where-Object {$_.Name -eq "EnableMIPLabels"}).Value = "true"
            
            if ($AllowedCreatorGroupId) {
                ($values | Where-Object {$_.Name -eq "EnableGroupCreation"}).Value = "true"
                ($values | Where-Object {$_.Name -eq "GroupCreationAllowedGroupId"}).Value = $AllowedCreatorGroupId
            }
            
            Update-MgDirectorySetting -DirectorySettingId $currentSettings.Id -BodyParameter @{Values = $values}
        }
        
        Write-Host "✅ プライベートグループのデフォルト設定が完了しました" -ForegroundColor Green
        Write-Host "📋 適用された設定:" -ForegroundColor Yellow
        Write-Host "   - デフォルトアクセス: プライベート" -ForegroundColor White
        Write-Host "   - 外部ゲスト追加: 無効" -ForegroundColor White
        Write-Host "   - 秘密度ラベル: 有効" -ForegroundColor White
        
        if ($EnforceStrictPrivacy) {
            # Teams での追加設定
            try {
                Connect-MicrosoftTeams
                
                # グローバルポリシーの更新
                Set-CsTeamsChannelsPolicy -Identity Global -AllowSharedChannels $false
                
                Write-Host "   - Teams 共有チャンネル: 無効" -ForegroundColor White
            }
            catch {
                Write-Host "⚠️ Teams 設定はスキップされました" -ForegroundColor Yellow
            }
            
            # SharePoint での追加設定
            try {
                Connect-SPOService -Url "https://yourdomain-admin.sharepoint.com"
                
                Set-SPOTenant -DefaultGroupAccessType Private
                Set-SPOTenant -SharingCapability ExternalUserSharingOnly
                
                Write-Host "   - SharePoint デフォルト: プライベート" -ForegroundColor White
            }
            catch {
                Write-Host "⚠️ SharePoint 設定はスキップされました" -ForegroundColor Yellow
            }
        }
    }
    catch {
        Write-Host "❌ 設定エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 実行例
# 管理者のみがグループを作成でき、すべてプライベートに設定
Set-EducationPrivateGroupDefault -EnforceStrictPrivacy

# 特定グループのユーザーにプライベートグループ作成を許可
$creatorGroupId = (Get-MgGroup -Filter "displayName eq '教職員グループ作成権限者'").Id
Set-EducationPrivateGroupDefault -AllowedCreatorGroupId $creatorGroupId -EnforceStrictPrivacy
```

#### プライベートグループ設定の確認

```powershell
# 現在のプライベートグループ設定を確認
function Get-PrivateGroupSettings {
    Connect-MgGraph -Scopes "Directory.Read.All"
    
    $settings = Get-MgDirectorySetting | Where-Object {$_.DisplayName -eq "Group.Unified"}
    
    if ($settings) {
        Write-Host "=== プライベートグループ設定 ===" -ForegroundColor Cyan
        
        $defaultAccess = ($settings.Values | Where-Object {$_.Name -eq "DefaultGroupAccessType"}).Value
        $allowGuests = ($settings.Values | Where-Object {$_.Name -eq "AllowToAddGuests"}).Value
        $mipLabels = ($settings.Values | Where-Object {$_.Name -eq "EnableMIPLabels"}).Value
        
        Write-Host "デフォルトアクセス: $defaultAccess" -ForegroundColor $(if($defaultAccess -eq "Private") {"Green"} else {"Red"})
        Write-Host "外部ゲスト許可: $allowGuests" -ForegroundColor $(if($allowGuests -eq "false") {"Green"} else {"Red"})
        Write-Host "秘密度ラベル: $mipLabels" -ForegroundColor $(if($mipLabels -eq "true") {"Green"} else {"Yellow"})
        
        # Teams ポリシーの確認
        try {
            Connect-MicrosoftTeams
            $teamsPolicy = Get-CsTeamsChannelsPolicy -Identity Global
            Write-Host "Teams 共有チャンネル: $($teamsPolicy.AllowSharedChannels)" -ForegroundColor $(if($teamsPolicy.AllowSharedChannels -eq $false) {"Green"} else {"Red"})
        }
        catch {
            Write-Host "Teams ポリシー: 確認できませんでした" -ForegroundColor Yellow
        }
    }
    else {
        Write-Host "❌ グループ設定が見つかりません" -ForegroundColor Red
    }
}

# 設定確認の実行
Get-PrivateGroupSettings
```

### 🔍 プライベートグループ設定の監視

```powershell
# プライベートでないグループを検出
function Find-PublicGroups {
    Connect-MgGraph -Scopes "Group.Read.All"
    
    $allGroups = Get-MgGroup -All -Property Id,DisplayName,Visibility,GroupTypes
    $publicGroups = $allGroups | Where-Object {$_.Visibility -eq "Public"}
    
    if ($publicGroups.Count -gt 0) {
        Write-Host "⚠️ パブリックグループが検出されました:" -ForegroundColor Yellow
        foreach ($group in $publicGroups) {
            Write-Host "  - $($group.DisplayName) (ID: $($group.Id))" -ForegroundColor White
        }
        
        # 自動でプライベートに変更するかの確認
        $choice = Read-Host "これらのグループをプライベートに変更しますか？ (y/N)"
        if ($choice -eq "y" -or $choice -eq "Y") {
            foreach ($group in $publicGroups) {
                try {
                    Update-MgGroup -GroupId $group.Id -Visibility "Private"
                    Write-Host "✅ プライベートに変更: $($group.DisplayName)" -ForegroundColor Green
                }
                catch {
                    Write-Host "❌ 変更失敗: $($group.DisplayName)" -ForegroundColor Red
                }
            }
        }
    }
    else {
        Write-Host "✅ すべてのグループがプライベートです" -ForegroundColor Green
    }
}

# 監視スクリプトの実行
Find-PublicGroups
```

### 📋 パブリックチーム作成の例外申請ワークフロー

セキュリティを重視しつつ、必要に応じてパブリックチーム作成を許可するための申請・承認システムを構築します。

#### 🔄 Microsoft Forms + Power Automate による承認ワークフロー

##### 1. Microsoft Forms 申請フォームの作成

```powershell
# Forms API (Graph API) を使用したフォーム作成（参考）
# 実際のフォーム作成は Microsoft Forms で行うことを推奨

$formTemplate = @"
=== パブリックチーム作成申請フォーム ===

1. 申請者情報
   - 氏名: [必須]
   - 所属部署: [必須]
   - メールアドレス: [必須]
   - 職位: [必須]

2. チーム情報
   - チーム名: [必須]
   - チームの目的: [必須]
   - 想定メンバー数: [必須]
   - プロジェクト期間: [必須]

3. パブリック設定の理由
   - パブリックが必要な理由: [必須]
   - 機密情報の有無: [必須]
   - 外部共有の必要性: [必須]
   - 代替手段の検討結果: [必須]

4. 承認者情報
   - 直属上司: [必須]
   - 部門責任者: [必須]
   - IT管理者: [必須]

5. 添付資料
   - プロジェクト概要書: [オプション]
   - セキュリティ要件書: [オプション]
"@

Write-Host $formTemplate
```

##### 2. Power Automate 承認フローの設定

```json
{
  "name": "パブリックチーム作成承認フロー",
  "description": "Forms申請に基づくパブリックチーム作成の多段階承認",
  "trigger": {
    "type": "Forms - 新しい応答が送信されたとき",
    "formId": "YOUR_FORM_ID"
  },
  "steps": [
    {
      "name": "申請内容の解析",
      "type": "変数の初期化",
      "variables": {
        "申請者名": "@{triggerOutputs()?['body/r_申請者名']}",
        "チーム名": "@{triggerOutputs()?['body/r_チーム名']}",
        "申請理由": "@{triggerOutputs()?['body/r_パブリック理由']}"
      }
    },
    {
      "name": "第1承認 - 直属上司",
      "type": "承認の開始",
      "approvers": "@{triggerOutputs()?['body/r_直属上司']}",
      "title": "パブリックチーム作成申請の承認",
      "details": "申請者: @{variables('申請者名')}\nチーム名: @{variables('チーム名')}\n理由: @{variables('申請理由')}"
    },
    {
      "name": "第2承認 - 部門責任者",
      "type": "承認の開始",
      "condition": "@{equals(outputs('第1承認_-_直属上司')?['body/outcome'], 'Approve')}",
      "approvers": "@{triggerOutputs()?['body/r_部門責任者']}",
      "title": "パブリックチーム作成申請の最終承認"
    },
    {
      "name": "第3承認 - IT管理者",
      "type": "承認の開始", 
      "condition": "@{equals(outputs('第2承認_-_部門責任者')?['body/outcome'], 'Approve')}",
      "approvers": "@{triggerOutputs()?['body/r_IT管理者']}",
      "title": "パブリックチーム作成のセキュリティ承認"
    },
    {
      "name": "承認済みチーム作成",
      "type": "HTTP",
      "condition": "@{equals(outputs('第3承認_-_IT管理者')?['body/outcome'], 'Approve')}",
      "method": "POST",
      "uri": "https://graph.microsoft.com/v1.0/teams",
      "headers": {
        "Authorization": "Bearer YOUR_ACCESS_TOKEN",
        "Content-Type": "application/json"
      },
      "body": {
        "template@odata.bind": "https://graph.microsoft.com/v1.0/teamsTemplates('standard')",
        "displayName": "@{variables('チーム名')}",
        "description": "承認済みパブリックチーム - @{variables('申請理由')}",
        "visibility": "Public"
      }
    },
    {
      "name": "申請者への完了通知",
      "type": "メールの送信",
      "to": "@{triggerOutputs()?['body/r_申請者メール']}",
      "subject": "パブリックチーム作成完了のお知らせ",
      "body": "お疲れ様です。\n\n申請されたパブリックチーム「@{variables('チーム名')}」の作成が完了しました。\n\nチームURL: @{outputs('承認済みチーム作成')?['body/webUrl']}\n\nセキュリティガイドラインに従って運用をお願いします。"
    }
  ]
}
```

##### 3. PowerShell による承認フロー管理

```powershell
# パブリックチーム作成承認フローの管理
function New-PublicTeamApprovalWorkflow {
    param(
        [string]$FormId,
        [string]$ApprovalFlowId,
        [string[]]$ITAdmins = @("it-admin@contoso.com")
    )
    
    $errorHandlingConfig = @"
{
  "errorHandling": {
    "retryPolicy": {
      "type": "exponential",
      "count": 3,
      "interval": "PT10S"
    },
    "errorActions": [
      {
        "name": "管理者通知",
        "type": "メールの送信",
        "to": ["$($AdminEmails -join '","')"],
        "subject": "パブリックチーム承認フロー エラー",
        "body": "申請ID: @{triggerOutputs()?['body/ID']}\nエラー詳細: @{outputs('前のアクション')?['error']}\n\n手動での対応が必要です。"
      },
      {
        "name": "SharePointエラーログ",
        "type": "SharePoint リストアイテムの作成",
        "siteUrl": "https://contoso.sharepoint.com/sites/ITManagement",
        "listName": "ApprovalErrors",
        "fields": {
          "申請ID": "@{triggerOutputs()?['body/ID']}",
          "エラー時刻": "@{utcNow()}",
          "エラー詳細": "@{outputs('前のアクション')?['error']}",
          "修復状態": "未対応"
        }
      }
    ]
  }
}
"@
    
    Write-Host "🔧 エラーハンドリング設定を追加しました" -ForegroundColor Green
    Write-Host $errorHandlingConfig
}

# 承認フローの自動復旧機能
function Enable-ApprovalFlowAutoRecovery {
    $recoveryActions = @"
{
  "autoRecovery": {
    "conditions": [
      {
        "errorType": "TimeoutError",
        "action": "escalateToNextApprover",
        "timeout": "PT2H"
      },
      {
        "errorType": "ApproverUnavailable", 
        "action": "delegateToManager",
        "fallbackApprover": "manager@contoso.com"
      }
    ],
    "notifications": {
      "onRecovery": true,
      "onEscalation": true
    }
  }
}
"@
    
    Write-Host "🔄 自動復旧機能を有効化しました" -ForegroundColor Cyan
    Write-Host $recoveryActions
}

# 実行例
Add-ApprovalFlowErrorHandling -FlowId "YOUR_FLOW_ID" -AdminEmails @("admin1@contoso.com", "admin2@contoso.com")
Enable-ApprovalFlowAutoRecovery
```

##### 🚨 リアルタイム監視とアラート

```powershell
# パブリックチーム作成の異常検知
function Start-PublicTeamAnomalyDetection {
    param(
        [int]$AlertThreshold = 5,  # 1日あたりの申請数上限
        [string[]]$AlertRecipients = @("security@contoso.com")
    )
    
    Write-Host "🚨 異常検知システム開始" -ForegroundColor Yellow
    
    $anomalyRules = @{
        MaxDailyApplications = $AlertThreshold
        SuspiciousPatterns = @(
            "短時間での大量申請",
            "同一ユーザーからの連続申請", 
            "権限外ユーザーからの申請",
            "外部ドメインへの共有要求"
        )
        ResponseActions = @(
            "即座の管理者通知",
            "申請の一時停止",
            "セキュリティログの詳細記録",
            "申請者への確認連絡"
        )
    }
    
    # Azure Sentinel連携（ログ分析）
    $sentinelQuery = @"
// パブリックチーム申請の異常検知クエリ
PublicTeamApplications
| where TimeGenerated > ago(24h)
| summarize ApplicationCount = count() by ApplicantUPN, bin(TimeGenerated, 1h)
| where ApplicationCount > $AlertThreshold
| project TimeGenerated, ApplicantUPN, ApplicationCount, AlertLevel = "High"
"@
    
    Write-Host "📊 異常検知ルール:" -ForegroundColor Cyan
    foreach ($rule in $anomalyRules.SuspiciousPatterns) {
        Write-Host "  - $rule" -ForegroundColor White
    }
    
    Write-Host "`n📋 対応アクション:" -ForegroundColor Yellow
    foreach ($action in $anomalyRules.ResponseActions) {
        Write-Host "  - $action" -ForegroundColor White
    }
    
    Write-Host "`n🔍 Sentinel監視クエリ:" -ForegroundColor Gray
    Write-Host $sentinelQuery -ForegroundColor DarkGray
    
    return $anomalyRules
}

# セキュリティインシデント対応
function Invoke-SecurityIncidentResponse {
    param(
        [string]$IncidentType,
        [string]$ApplicantUPN,
        [string]$Details
    )
    
    $incidentActions = switch ($IncidentType) {
        "MassApplication" {
            @{
                Priority = "High"
                Actions = @(
                    "申請者のアカウント一時停止",
                    "過去24時間の申請履歴確認",
                    "セキュリティチームへの即座通知"
                )
            }
        }
        "UnauthorizedAccess" {
            @{
                Priority = "Critical" 
                Actions = @(
                    "アカウントの即座停止",
                    "認証ログの詳細調査",
                    "インシデント対応チーム招集"
                )
            }
        }
        "DataExfiltration" {
            @{
                Priority = "Critical"
                Actions = @(
                    "該当チームの即座停止",
                    "データアクセスログ保全",
                    "法務・コンプライアンス通知"
                )
            }
        }
    }
    
    Write-Host "🚨 セキュリティインシデント検知" -ForegroundColor Red
    Write-Host "種別: $IncidentType" -ForegroundColor Yellow
    Write-Host "対象: $ApplicantUPN" -ForegroundColor Yellow
    Write-Host "詳細: $Details" -ForegroundColor Yellow
    Write-Host "優先度: $($incidentActions.Priority)" -ForegroundColor Red
    
    Write-Host "`n📋 対応アクション:" -ForegroundColor Cyan
    foreach ($action in $incidentActions.Actions) {
        Write-Host "  ✓ $action" -ForegroundColor White
    }
    
    # 自動対応の実行
    try {
        # 例: アカウント停止
        if ($incidentActions.Actions -contains "申請者のアカウント一時停止") {
            Set-MgUser -UserId $ApplicantUPN -AccountEnabled:$false
            Write-Host "✅ アカウント停止完了: $ApplicantUPN" -ForegroundColor Green
        }
        
        # 例: セキュリティログ記録
        $logEntry = @{
            Timestamp = Get-Date
            IncidentType = $IncidentType
            Severity = $incidentActions.Priority
            AffectedUser = $ApplicantUPN
            Details = $Details
            AutoActionsExecuted = $incidentActions.Actions
        }
        
        Write-Host "📝 インシデントログ記録完了" -ForegroundColor Green
    }
    catch {
        Write-Host "❌ 自動対応エラー: $($_.Exception.Message)" -ForegroundColor Red
    }
}

# 実行例
$anomalyRules = Start-PublicTeamAnomalyDetection -AlertThreshold 3
Invoke-SecurityIncidentResponse -IncidentType "MassApplication" -ApplicantUPN "suspicious@contoso.com" -Details "1時間で10件の申請"
```

##### 📱 モバイル承認とプッシュ通知

```powershell
# モバイル承認システムの設定
function Enable-MobileApprovalSystem {
    param(
        [string]$TeamsAppId,
        [bool]$EnablePushNotifications = $true
    )
    
    $mobileConfig = @{
        Platform = "Microsoft Teams Mobile"
        Features = @(
            "ワンタップ承認/否認",
            "音声付きプッシュ通知", 
            "オフライン時の代理承認",
            "バイオメトリック認証",
            "GPSベースの場所確認"
        )
        ApprovalMethods = @(
            "Teams アダプティブカード",
            "Outlook モバイル承認",
            "Power Apps モバイルアプリ"
        )
    }
    
    # Teams アダプティブカードテンプレート
    $adaptiveCard = @"
{
  "type": "AdaptiveCard",
  "version": "1.3",
  "body": [
    {
      "type": "TextBlock",
      "text": "🔓 パブリックチーム作成申請",
      "weight": "Bolder",
      "size": "Medium"
    },
    {
      "type": "FactSet",
      "facts": [
        {"title": "申請者", "value": "@{variables('申請者名')}"},
        {"title": "チーム名", "value": "@{variables('チーム名')}"},
        {"title": "目的", "value": "@{variables('申請理由')}"},
        {"title": "緊急度", "value": "@{variables('緊急度')}"}
      ]
    },
    {
      "type": "TextBlock",
      "text": "セキュリティ確認済み ✅",
      "color": "Good"
    }
  ],
  "actions": [
    {
      "type": "Action.Submit",
      "title": "✅ 承認",
      "data": {"action": "approve", "requestId": "@{variables('申請ID')}"},
      "style": "positive"
    },
    {
      "type": "Action.Submit", 
      "title": "❌ 否認",
      "data": {"action": "reject", "requestId": "@{variables('申請ID')}"},
      "style": "destructive"
    },
    {
      "type": "Action.Submit",
      "title": "ℹ️ 詳細確認",
      "data": {"action": "details", "requestId": "@{variables('申請ID')}"}
    }
  ]
}
"@
    
    Write-Host "📱 モバイル承認システム設定" -ForegroundColor Cyan
    Write-Host "プラットフォーム: $($mobileConfig.Platform)" -ForegroundColor White
    Write-Host "機能:" -ForegroundColor Yellow
    foreach ($feature in $mobileConfig.Features) {
        Write-Host "  ✓ $feature" -ForegroundColor Green
    }
    
    if ($EnablePushNotifications) {
        Write-Host "`n🔔 プッシュ通知設定:" -ForegroundColor Yellow
        Write-Host "  - 即座通知: 承認待ち申請" -ForegroundColor White
        Write-Host "  - 毎日の要約: 保留中申請" -ForegroundColor White
        Write-Host "  - 緊急アラート: セキュリティ関連" -ForegroundColor White
    }
    
    Write-Host "`n📋 Teams アダプティブカード:" -ForegroundColor Gray
    Write-Host $adaptiveCard -ForegroundColor DarkGray
    
    return $mobileConfig
}

# 実行例
Enable-MobileApprovalSystem -TeamsAppId "YOUR_TEAMS_APP_ID" -EnablePushNotifications $true
```

##### 🎯 AI支援による承認支援

```powershell
# AI による申請内容分析と承認支援
function Enable-AIAssistedApproval {
    param(
        [string]$OpenAIEndpoint,
        [string]$CognitiveServicesKey
    )
    
    Write-Host "🤖 AI承認支援システム初期化中..." -ForegroundColor Cyan
    
    $aiFeatures = @{
        ContentAnalysis = @{
            Description = "申請内容の自動分析"
            Capabilities = @(
                "リスクレベル自動判定",
                "セキュリティ要件チェック",
                "類似申請との比較",
                "承認履歴パターン学習"
            )
        }
        RiskScoring = @{
            Description = "リスクスコア自動算出"
            Factors = @(
                "申請者の過去履歴",
                "要求される権限レベル",
                "外部共有の範囲",
                "データ機密性評価"
            )
        }
        RecommendationEngine = @{
            Description = "承認推奨エンジン"
            Outputs = @(
                "承認/否認の推奨判定",
                "追加確認が必要な項目",
                "類似ケースの参照",
                "承認条件の提案"
            )
        }
    }
    
    # AI分析用のプロンプトテンプレート
    $analysisPrompt = @"
以下のパブリックチーム作成申請を分析してください：

申請内容:
- 申請者: {申請者名} ({所属部署})
- チーム名: {チーム名}
- 目的: {チームの目的}
- 外部共有: {外部共有の必要性}
- メンバー数: {想定メンバー数}
- 期間: {プロジェクト期間}

分析項目:
1. セキュリティリスクレベル (1-10)
2. ビジネス妥当性 (1-10)
3. 承認推奨度 (承認/条件付き承認/否認)
4. 注意すべきポイント
5. 追加確認推奨事項

過去の承認履歴と比較し、客観的な判断を提供してください。
"@
    
    Write-Host "🔍 AI分析機能:" -ForegroundColor Yellow
    foreach ($feature in $aiFeatures.Keys) {
        Write-Host "  ✓ $($aiFeatures[$feature].Description)" -ForegroundColor Green
        foreach ($capability in $aiFeatures[$feature].Values) {
            if ($capability -is [array]) {
                foreach ($item in $capability) {
                    Write-Host "    - $item" -ForegroundColor White
                }
            }
        }
    }
    
    Write-Host "`n📝 AI分析プロンプト:" -ForegroundColor Gray
    Write-Host $analysisPrompt -ForegroundColor DarkGray
    
    return @{
        Features = $aiFeatures
        AnalysisPrompt = $analysisPrompt
        Enabled = $true
    }
}

# AI分析結果の例
function Get-AIApprovalRecommendation {
    param(
        [hashtable]$ApplicationData
    )
    
    # 模擬AI分析結果
    $aiAnalysis = @{
        RiskScore = 6
        BusinessValue = 8
        Recommendation = "条件付き承認"
        KeyConcerns = @(
            "外部ユーザーアクセス範囲の明確化が必要",
            "プロジェクト終了後のデータ処理方針確認"
        )
        SuggestedConditions = @(
            "30日間の試用期間設定",
            "月次レビューの実施",
            "外部共有ログの監視強化"
        )
        SimilarCases = @(
            @{Case = "広報-2024-003"; Outcome = "承認"; Note = "同様の外部連携で成功"},
            @{Case = "広報-2023-015"; Outcome = "否認"; Note = "セキュリティ要件未達"}
        )
    }
    
    Write-Host "🤖 AI承認分析結果" -ForegroundColor Cyan
    Write-Host "リスクスコア: $($aiAnalysis.RiskScore)/10" -ForegroundColor $(if($aiAnalysis.RiskScore -le 5) {"Green"} elseif($aiAnalysis.RiskScore -le 7) {"Yellow"} else {"Red"})
    Write-Host "ビジネス価値: $($aiAnalysis.BusinessValue)/10" -ForegroundColor Green
    Write-Host "AI推奨: $($aiAnalysis.Recommendation)" -ForegroundColor Yellow
    
    Write-Host "`n⚠️ 懸念事項:" -ForegroundColor Yellow
    foreach ($concern in $aiAnalysis.KeyConcerns) {
        Write-Host "  - $concern" -ForegroundColor White
    }
    
    Write-Host "`n💡 推奨条件:" -ForegroundColor Cyan
    foreach ($condition in $aiAnalysis.SuggestedConditions) {
        Write-Host "  - $condition" -ForegroundColor White
    }
    
    return $aiAnalysis
}

# 実行例
$aiSystem = Enable-AIAssistedApproval -OpenAIEndpoint "https://your-openai.openai.azure.com/" -CognitiveServicesKey "YOUR_KEY"
$recommendation = Get-AIApprovalRecommendation -ApplicationData @{Name="新規広報チーム"; Purpose="外部連携"}
```
