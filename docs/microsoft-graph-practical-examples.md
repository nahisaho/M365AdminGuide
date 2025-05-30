# Microsoft Graph PowerShell 実践例集

Microsoft Graph PowerShell を使用した日本の組織でよく使用される実践的なシナリオとコード例集です。

## 目次

- [新年度の組織変更対応](#新年度の組織変更対応)
- [退職者のアカウント処理](#退職者のアカウント処理)
- [セキュリティ監査レポート](#セキュリティ監査レポート)
- [包括的な管理クラス](#包括的な管理クラス)
- [定期実行タスク](#定期実行タスク)

## 新年度の組織変更対応

### 部署名の一括変更

```powershell
function Update-DepartmentNames {
    param(
        [Parameter(Mandatory)]
        [hashtable]$DepartmentMapping  # @{"旧部署名" = "新部署名"}
    )
    
    foreach ($oldDept in $DepartmentMapping.Keys) {
        $newDept = $DepartmentMapping[$oldDept]
        
        # 該当部署のユーザーを取得
        $users = Get-MgUser -Filter "department eq '$oldDept'" -Property DisplayName,UserPrincipalName,Department
        
        Write-Host "部署変更: $oldDept → $newDept (対象ユーザー: $($users.Count)人)" -ForegroundColor Yellow
        
        foreach ($user in $users) {
            try {
                Update-MgUser -UserId $user.Id -Department $newDept
                Write-Host "  ✓ $($user.DisplayName)" -ForegroundColor Green
            }
            catch {
                Write-Warning "  ✗ $($user.DisplayName): $($_.Exception.Message)"
            }
        }
        
        # 部署グループの名前変更
        $oldGroup = Get-MgGroup -Filter "displayName eq '$oldDept'"
        if ($oldGroup) {
            Update-MgGroup -GroupId $oldGroup.Id -DisplayName $newDept
            Write-Host "グループ名変更: $oldDept → $newDept" -ForegroundColor Cyan
        }
    }
}

# 使用例：2024年度組織改編
$deptChanges = @{
    "総務部" = "総務・人事部"
    "営業第一部" = "営業統括部"
    "営業第二部" = "営業統括部"
    "システム部" = "デジタル推進部"
}

Update-DepartmentNames -DepartmentMapping $deptChanges
```

## 退職者のアカウント処理

### 退職者の無効化と引き継ぎ処理

```powershell
function Disable-EmployeeAccount {
    param(
        [Parameter(Mandatory)]
        [string]$EmployeeUPN,
        
        [Parameter(Mandatory)]
        [string]$SuccessorUPN,
        
        [switch]$ForwardEmail = $true,
        [switch]$ConvertToSharedMailbox = $false,
        [int]$MailboxRetentionDays = 30
    )
    
    try {
        # 退職者情報の取得
        $employee = Get-MgUser -UserId $EmployeeUPN -Property DisplayName,AssignedLicenses,Department,JobTitle
        $successor = Get-MgUser -UserId $SuccessorUPN -Property DisplayName
        
        Write-Host "退職者処理開始: $($employee.DisplayName) → 引き継ぎ先: $($successor.DisplayName)" -ForegroundColor Yellow
        
        # 1. アカウントの無効化
        Update-MgUser -UserId $employee.Id -AccountEnabled $false
        Write-Host "  ✓ アカウント無効化完了" -ForegroundColor Green
        
        # 2. ライセンスの削除
        if ($employee.AssignedLicenses.Count -gt 0) {
            $removeLicenses = $employee.AssignedLicenses | ForEach-Object { $_.SkuId }
            
            $licenseUpdate = @{
                AddLicenses = @()
                RemoveLicenses = $removeLicenses
            }
            
            Set-MgUserLicense -UserId $employee.Id -BodyParameter $licenseUpdate
            Write-Host "  ✓ ライセンス削除完了 ($($removeLicenses.Count)個)" -ForegroundColor Green
        }
        
        # 3. グループメンバーシップの確認と削除
        $membershipGroups = Get-MgUserMemberOf -UserId $employee.Id
        foreach ($group in $membershipGroups) {
            if ($group.AdditionalProperties["@odata.type"] -eq "#microsoft.graph.group") {
                $groupInfo = Get-MgGroup -GroupId $group.Id
                Write-Host "  ! グループメンバーシップ確認必要: $($groupInfo.DisplayName)" -ForegroundColor Yellow
            }
        }
        
        # 4. 処理結果のレポート
        $report = @{
            Timestamp = Get-Date
            Employee = $employee.DisplayName
            EmployeeUPN = $EmployeeUPN
            Successor = $successor.DisplayName
            SuccessorUPN = $SuccessorUPN
            AccountDisabled = $true
            LicensesRemoved = $employee.AssignedLicenses.Count
            Department = $employee.Department
            JobTitle = $employee.JobTitle
            RetentionDate = (Get-Date).AddDays($MailboxRetentionDays)
        }
        
        $report | Export-Csv -Path "C:\temp\employee-offboarding-$(Get-Date -Format 'yyyyMMdd').csv" -Append -NoTypeInformation -Encoding UTF8
        
        Write-Host "退職者処理完了: $($employee.DisplayName)" -ForegroundColor Green
        return $report
    }
    catch {
        Write-Error "退職者処理エラー: $($_.Exception.Message)"
        return $null
    }
}

# 使用例
$result = Disable-EmployeeAccount -EmployeeUPN "yamada.taro@contoso.com" -SuccessorUPN "tanaka.hanako@contoso.com" -ForwardEmail
```

## セキュリティ監査レポート

### 定期的なセキュリティ監査

```powershell
function Get-SecurityAuditReport {
    param(
        [string]$OutputPath = "C:\temp\security-audit-$(Get-Date -Format 'yyyyMMdd-HHmm').csv"
    )
    
    Write-Host "セキュリティ監査レポート生成中..." -ForegroundColor Yellow
    
    $auditResults = @()
    
    # 1. 管理者権限を持つユーザーの確認
    $adminRoles = Get-MgDirectoryRole | Where-Object { 
        $_.DisplayName -match "管理者|Administrator" 
    }
    
    foreach ($role in $adminRoles) {
        $roleMembers = Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id
        foreach ($member in $roleMembers) {
            if ($member.AdditionalProperties["@odata.type"] -eq "#microsoft.graph.user") {
                $user = Get-MgUser -UserId $member.Id -Property DisplayName,UserPrincipalName,AccountEnabled,LastSignInDateTime
                
                $auditResults += [PSCustomObject]@{
                    Type = "管理者権限"
                    User = $user.DisplayName
                    UPN = $user.UserPrincipalName
                    Role = $role.DisplayName
                    AccountEnabled = $user.AccountEnabled
                    LastSignIn = $user.LastSignInDateTime
                    Risk = if (-not $user.AccountEnabled) { "無効アカウント" } 
                           elseif (-not $user.LastSignInDateTime -or $user.LastSignInDateTime -lt (Get-Date).AddDays(-90)) { "長期未使用" }
                           else { "正常" }
                }
            }
        }
    }
    
    # 2. ゲストユーザーの確認
    $guestUsers = Get-MgUser -Filter "userType eq 'Guest'" -Property DisplayName,UserPrincipalName,CreatedDateTime,LastSignInDateTime
    
    foreach ($guest in $guestUsers) {
        $auditResults += [PSCustomObject]@{
            Type = "ゲストユーザー"
            User = $guest.DisplayName
            UPN = $guest.UserPrincipalName
            Role = "ゲスト"
            AccountEnabled = $true
            LastSignIn = $guest.LastSignInDateTime
            CreatedDate = $guest.CreatedDateTime
            Risk = if (-not $guest.LastSignInDateTime -or $guest.LastSignInDateTime -lt (Get-Date).AddDays(-30)) { "未使用ゲスト" }
                   elseif ($guest.CreatedDateTime -gt (Get-Date).AddDays(-7)) { "新規ゲスト" }
                   else { "正常" }
        }
    }
    
    # 3. 非アクティブユーザーの確認
    $inactiveUsers = Get-MgUser -Filter "accountEnabled eq true" -Property DisplayName,UserPrincipalName,LastSignInDateTime,CreatedDateTime |
        Where-Object { 
            $_.LastSignInDateTime -and 
            $_.LastSignInDateTime -lt (Get-Date).AddDays(-60) 
        }
    
    foreach ($user in $inactiveUsers) {
        $auditResults += [PSCustomObject]@{
            Type = "非アクティブユーザー"
            User = $user.DisplayName
            UPN = $user.UserPrincipalName
            Role = "一般ユーザー"
            AccountEnabled = $true
            LastSignIn = $user.LastSignInDateTime
            CreatedDate = $user.CreatedDateTime
            Risk = "長期未使用"
        }
    }
    
    # 4. レポートの出力
    $auditResults | Export-Csv -Path $OutputPath -NoTypeInformation -Encoding UTF8
    
    # 5. サマリーの表示
    $riskCounts = $auditResults | Group-Object Risk | Select-Object Name, Count
    
    Write-Host "`n📊 セキュリティ監査サマリー" -ForegroundColor Cyan
    Write-Host "================================" -ForegroundColor Cyan
    foreach ($risk in $riskCounts) {
        $color = switch ($risk.Name) {
            "正常" { "Green" }
            "新規ゲスト" { "Yellow" }
            default { "Red" }
        }
        Write-Host "$($risk.Name): $($risk.Count)件" -ForegroundColor $color
    }
    
    Write-Host "`n📄 詳細レポート: $OutputPath" -ForegroundColor Green
    
    return $auditResults
}

# 使用例
$auditReport = Get-SecurityAuditReport
```

## 包括的な管理クラス

### GraphManager クラス（エラー処理・ログ機能付き）

```powershell
class GraphManager {
    [string]$LogPath
    [string]$TenantId
    [bool]$Connected
    
    # コンストラクタ
    GraphManager([string]$LogDirectory) {
        $this.LogPath = Join-Path $LogDirectory "GraphManagement-$(Get-Date -Format 'yyyyMMdd').log"
        $this.Connected = $false
        
        # ログディレクトリの作成
        if (-not (Test-Path (Split-Path $this.LogPath))) {
            New-Item -ItemType Directory -Path (Split-Path $this.LogPath) -Force
        }
    }
    
    # ログ記録メソッド
    [void]WriteLog([string]$Level, [string]$Message) {
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $logEntry = "[$timestamp] [$Level] $Message"
        Add-Content -Path $this.LogPath -Value $logEntry -Encoding UTF8
        
        # コンソール出力
        $color = switch ($Level) {
            "INFO" { "White" }
            "SUCCESS" { "Green" }
            "WARNING" { "Yellow" }
            "ERROR" { "Red" }
            default { "Gray" }
        }
        Write-Host $logEntry -ForegroundColor $color
    }
    
    # Microsoft Graph への接続
    [bool]Connect([string[]]$Scopes) {
        try {
            $this.WriteLog("INFO", "Microsoft Graph への接続を開始します")
            
            Connect-MgGraph -Scopes $Scopes -NoWelcome
            $context = Get-MgContext
            
            if ($context) {
                $this.Connected = $true
                $this.TenantId = $context.TenantId
                $this.WriteLog("SUCCESS", "Microsoft Graph に正常に接続しました (テナント: $($this.TenantId))")
                return $true
            }
            else {
                $this.WriteLog("ERROR", "Microsoft Graph への接続に失敗しました")
                return $false
            }
        }
        catch {
            $this.WriteLog("ERROR", "Microsoft Graph 接続エラー: $($_.Exception.Message)")
            return $false
        }
    }
    
    # 安全な切断
    [void]Disconnect() {
        if ($this.Connected) {
            try {
                Disconnect-MgGraph
                $this.Connected = $false
                $this.WriteLog("INFO", "Microsoft Graph から切断しました")
            }
            catch {
                $this.WriteLog("WARNING", "切断時にエラーが発生しました: $($_.Exception.Message)")
            }
        }
    }
    
    # パスワード生成
    [string]GeneratePassword() {
        $lowercase = "abcdefghijklmnopqrstuvwxyz"
        $uppercase = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
        $numbers = "0123456789"
        $symbols = "!@#$%^&*"
        
        $password = ""
        $password += ($lowercase.ToCharArray() | Get-Random -Count 3) -join ""
        $password += ($uppercase.ToCharArray() | Get-Random -Count 3) -join ""
        $password += ($numbers.ToCharArray() | Get-Random -Count 2) -join ""
        $password += ($symbols.ToCharArray() | Get-Random -Count 2) -join ""
        
        # パスワードをシャッフル
        $passwordArray = $password.ToCharArray()
        for ($i = $passwordArray.Length - 1; $i -gt 0; $i--) {
            $j = Get-Random -Maximum ($i + 1)
            $temp = $passwordArray[$i]
            $passwordArray[$i] = $passwordArray[$j]
            $passwordArray[$j] = $temp
        }
        
        return -join $passwordArray
    }
    
    # 新入社員の一括登録
    [object[]]BulkCreateUsers([string]$CsvPath) {
        if (-not $this.Connected) {
            $this.WriteLog("ERROR", "Microsoft Graph に接続されていません")
            return @()
        }
        
        $this.WriteLog("INFO", "CSV ファイルからユーザーの一括作成を開始: $CsvPath")
        
        if (-not (Test-Path $CsvPath)) {
            $this.WriteLog("ERROR", "CSV ファイルが見つかりません: $CsvPath")
            return @()
        }
        
        $users = Import-Csv -Path $CsvPath -Encoding UTF8
        $results = @()
        $successCount = 0
        $errorCount = 0
        
        foreach ($user in $users) {
            try {
                # パスワードの生成
                $tempPassword = $this.GeneratePassword()
                
                # ユーザープリンシパル名の生成
                $userName = "$($user.FirstName).$($user.LastName)".ToLower() -replace '[^a-z0-9.]', ''
                $userPrincipalName = "$userName@$($user.Domain)"
                
                $passwordProfile = @{
                    ForceChangePasswordNextSignIn = $true
                    Password = $tempPassword
                }
                
                $newUserParams = @{
                    DisplayName = "$($user.FirstName) $($user.LastName)"
                    UserPrincipalName = $userPrincipalName
                    MailNickname = $userName
                    PasswordProfile = $passwordProfile
                    AccountEnabled = $true
                    Department = $user.Department
                    JobTitle = $user.JobTitle
                    OfficeLocation = $user.Office
                    Country = "Japan"
                    UsageLocation = "JP"
                }
                
                $newUser = New-MgUser @newUserParams
                
                $results += [PSCustomObject]@{
                    Status = "Success"
                    DisplayName = $newUser.DisplayName
                    UserPrincipalName = $newUser.UserPrincipalName
                    TempPassword = $tempPassword
                    Department = $user.Department
                    Error = $null
                }
                
                $successCount++
                $this.WriteLog("SUCCESS", "ユーザー作成成功: $($newUser.DisplayName) ($($newUser.UserPrincipalName))")
                
                # ライセンス割り当て（指定されている場合）
                if ($user.License) {
                    $this.AssignLicense($newUser.Id, $user.License)
                }
                
                # 部署グループへの追加
                if ($user.Department) {
                    $this.AddUserToDepartmentGroup($newUser.Id, $user.Department)
                }
                
            }
            catch {
                $errorMessage = $_.Exception.Message
                $results += [PSCustomObject]@{
                    Status = "Error"
                    DisplayName = "$($user.FirstName) $($user.LastName)"
                    UserPrincipalName = $userPrincipalName
                    TempPassword = $null
                    Department = $user.Department
                    Error = $errorMessage
                }
                
                $errorCount++
                $this.WriteLog("ERROR", "ユーザー作成失敗: $($user.FirstName) $($user.LastName) - $errorMessage")
            }
        }
        
        # 結果の出力
        $outputPath = "C:\temp\user-creation-results-$(Get-Date -Format 'yyyyMMdd-HHmm').csv"
        $results | Export-Csv -Path $outputPath -NoTypeInformation -Encoding UTF8
        
        $this.WriteLog("INFO", "一括ユーザー作成完了: 成功 $successCount 件, 失敗 $errorCount 件")
        $this.WriteLog("INFO", "結果ファイル: $outputPath")
        
        return $results
    }
    
    # ライセンス割り当て
    [void]AssignLicense([string]$UserId, [string]$LicenseSkuPartNumber) {
        try {
            $sku = Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq $LicenseSkuPartNumber}
            if ($sku -and $sku.PrepaidUnits.Enabled -gt $sku.ConsumedUnits) {
                $license = @{
                    AddLicenses = @(@{ SkuId = $sku.SkuId })
                    RemoveLicenses = @()
                }
                
                Set-MgUserLicense -UserId $UserId -BodyParameter $license
                $this.WriteLog("SUCCESS", "ライセンス割り当て成功: $LicenseSkuPartNumber")
            }
            else {
                $this.WriteLog("WARNING", "ライセンス不足または見つかりません: $LicenseSkuPartNumber")
            }
        }
        catch {
            $this.WriteLog("ERROR", "ライセンス割り当てエラー: $($_.Exception.Message)")
        }
    }
    
    # 部署グループへの追加
    [void]AddUserToDepartmentGroup([string]$UserId, [string]$Department) {
        try {
            $group = Get-MgGroup -Filter "displayName eq '$Department'"
            if ($group) {
                New-MgGroupMember -GroupId $group.Id -DirectoryObjectId $UserId
                $this.WriteLog("SUCCESS", "部署グループに追加: $Department")
            }
            else {
                # グループが存在しない場合は作成
                $newGroup = $this.CreateDepartmentGroup($Department)
                if ($newGroup) {
                    New-MgGroupMember -GroupId $newGroup.Id -DirectoryObjectId $UserId
                }
            }
        }
        catch {
            $this.WriteLog("ERROR", "グループ追加エラー: $($_.Exception.Message)")
        }
    }
    
    # 部署グループの作成
    [object]CreateDepartmentGroup([string]$Department) {
        try {
            $groupParams = @{
                DisplayName = $Department
                Description = "$Department のメンバー"
                GroupTypes = @()
                SecurityEnabled = $true
                MailEnabled = $false
                MailNickname = ($Department -replace '\s', '').ToLower()
            }
            
            $group = New-MgGroup @groupParams
            $this.WriteLog("SUCCESS", "部署グループ作成: $Department")
            return $group
        }
        catch {
            $this.WriteLog("ERROR", "グループ作成エラー: $($_.Exception.Message)")
            return $null
        }
    }
}

# 使用例
function Start-GraphManagement {
    # GraphManager の初期化
    $graphMgr = [GraphManager]::new("C:\temp\GraphLogs")
    
    try {
        # 接続
        $scopes = @(
            "User.ReadWrite.All",
            "Group.ReadWrite.All",
            "Directory.ReadWrite.All"
        )
        
        if ($graphMgr.Connect($scopes)) {
            
            # 新入社員の一括登録（CSVファイルがある場合）
            $csvPath = "C:\temp\new-employees.csv"
            if (Test-Path $csvPath) {
                $results = $graphMgr.BulkCreateUsers($csvPath)
                
                # 成功したユーザーのみを表示
                $successfulUsers = $results | Where-Object { $_.Status -eq "Success" }
                if ($successfulUsers) {
                    Write-Host "`n✅ 作成成功ユーザー:" -ForegroundColor Green
                    $successfulUsers | Format-Table DisplayName, UserPrincipalName, Department -AutoSize
                }
            }
        }
    }
    finally {
        # 必ず切断
        $graphMgr.Disconnect()
    }
}
```

## 定期実行タスク

### スケジュールタスクの設定

```powershell
function Register-GraphManagementTask {
    param(
        [Parameter(Mandatory)]
        [string]$TaskName,
        
        [Parameter(Mandatory)]
        [string]$ScriptPath,
        
        [string]$Schedule = "Daily",
        [string]$StartTime = "09:00"
    )
    
    # タスクアクション（PowerShell スクリプトの実行）
    $action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-ExecutionPolicy Bypass -File `"$ScriptPath`""
    
    # トリガー（スケジュール）の設定
    $trigger = switch ($Schedule) {
        "Daily" { New-ScheduledTaskTrigger -Daily -At $StartTime }
        "Weekly" { New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday -At $StartTime }
        "Monthly" { New-ScheduledTaskTrigger -Weekly -WeeksInterval 4 -DaysOfWeek Monday -At $StartTime }
    }
    
    # タスクの設定
    $settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
    
    # プリンシパル（実行ユーザー）の設定
    $principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
    
    # タスクの登録
    Register-ScheduledTask -TaskName $TaskName -Action $action -Trigger $trigger -Settings $settings -Principal $principal -Description "Microsoft Graph 管理の自動実行タスク"
    
    Write-Host "スケジュールタスク '$TaskName' を登録しました" -ForegroundColor Green
    Write-Host "スケジュール: $Schedule ($StartTime)" -ForegroundColor Cyan
    Write-Host "スクリプト: $ScriptPath" -ForegroundColor Cyan
}

# 使用例
# Register-GraphManagementTask -TaskName "GraphHealthCheck" -ScriptPath "C:\Scripts\GraphHealthCheck.ps1" -Schedule "Daily" -StartTime "08:00"
```

## CSVテンプレート

### 新入社員一括登録用 CSV (new-employees.csv)

```csv
FirstName,LastName,Domain,Department,JobTitle,Office,License
太郎,山田,contoso.com,営業部,営業担当,東京本社,ENTERPRISEPACK
花子,田中,contoso.com,総務部,総務担当,東京本社,ENTERPRISEPACK
次郎,佐藤,contoso.com,開発部,エンジニア,大阪支社,ENTERPRISEPACK_FACULTY
```

## 関連情報

- [Microsoft Graph PowerShell セットアップガイド](microsoft-graph-powershell.md)
- [ユーザー管理](user-management.md)
- [セキュリティ管理](security-management.md)
- [ライセンス管理](license-management.md)

## 注意事項

### 実行前の確認

1. **テスト環境での検証**: 本番環境での実行前は必ずテスト環境で動作確認
2. **権限の確認**: 必要な Microsoft Graph 権限が割り当てられているか確認
3. **バックアップ**: 重要なデータの変更前はバックアップを取得
4. **ログの確認**: 実行後は必ずログファイルを確認

### セキュリティ考慮事項

- 生成された一時パスワードは安全に管理し、使用後は削除
- 実行アカウントには必要最小限の権限のみを付与
- ログファイルに機密情報が含まれないよう注意
- 定期的にスクリプトとログの内容を監査
