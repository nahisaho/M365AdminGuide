# Exchange Online 管理

Exchange Online のメール管理に関するガイドです。

## 概要

Exchange Online は Microsoft 365 のクラウドベースのメールサービスです。メールボックス管理、メールフロー制御、セキュリティ機能などを提供します。

## 前提条件

- Exchange Online 管理者権限
- Exchange Online PowerShell モジュール
- 適切なライセンス

## PowerShell 接続

**以下のPowerShellコードの処理内容:**

1. `Install-Module -Name ExchangeOnlineManagement -Force` - Exchange Online管理用PowerShellモジュールをインストール、既存バージョンがある場合は強制上書き
2. `Import-Module ExchangeOnlineManagement` - インストールしたモジュールを現在のPowerShellセッションに読み込み
3. `Connect-ExchangeOnline -UserPrincipalName` - 指定した管理者アカウントでExchange Onlineサービスに認証・接続

```powershell
# Exchange Online に接続
Install-Module -Name ExchangeOnlineManagement -Force
Import-Module ExchangeOnlineManagement
Connect-ExchangeOnline -UserPrincipalName admin@contoso.onmicrosoft.com
```

## メールボックス管理

### メールボックスの作成

**以下のPowerShellコードの処理内容:**

1. `New-Mailbox` - Exchange Onlineに新しいメールボックスを作成するコマンドレット
2. `-Name` - メールボックスの内部名称を指定
3. `-DisplayName` - アドレス帳や組織内で表示される氏名
4. `-UserPrincipalName` - ユーザーのログインIDとメールアドレス
5. `-Password` - 初期パスワードをSecureString形式で設定、ConvertTo-SecureStringで文字列を暗号化

```powershell
# 新しいメールボックスを作成
New-Mailbox -Name "田中太郎" -DisplayName "田中 太郎" -UserPrincipalName "tanaka@contoso.com" -Password (ConvertTo-SecureString -String "TempPassword123!" -AsPlainText -Force)
```

### メールボックス設定の変更

**以下のPowerShellコードの処理内容:**

1. `Set-Mailbox` - 既存メールボックスの設定を変更するコマンドレット
2. `-ProhibitSendQuota 50GB` - 送信禁止容量制限を50GBに設定、この容量に達すると新しいメール送信が不可
3. `-ProhibitSendReceiveQuota 55GB` - 送受信禁止容量制限、この容量に達すると完全にメール機能停止
4. `-IssueWarningQuota 45GB` - 警告発生容量、45GBに達するとユーザーに容量警告を通知
5. `Set-MailboxAutoReplyConfiguration` - 自動返信（不在時メッセージ）の設定変更

```powershell
# メールボックス容量の変更
Set-Mailbox -Identity "tanaka@contoso.com" -ProhibitSendQuota 50GB -ProhibitSendReceiveQuota 55GB -IssueWarningQuota 45GB

# 自動返信の設定
Set-MailboxAutoReplyConfiguration -Identity "tanaka@contoso.com" -AutoReplyState Enabled -InternalMessage "現在、外出しております。" -ExternalMessage "I am currently out of office."
```

### 共有メールボックス

共有メールボックスは、複数のユーザーが共通のメールアドレスからメールを送受信するために使用されます。教育機関では、学年事務、児童生徒支援、図書室などで活用されます。

#### 🎓 教育機関での共有メールボックス活用例

- **学年事務**: `info@5th.school.edu.jp` (小学5年)
- **児童生徒支援**: `student-support@school.edu.jp` (児童生徒相談)
- **図書室**: `library@school.edu.jp` (図書室サービス)
- **保健室**: `health@school.edu.jp` (保健管理関連)

#### 共有メールボックスの作成と設定

```powershell
# 教育機関向け共有メールボックスの作成
function New-EducationSharedMailbox {
    param(
        [string]$Department,
        [string]$EmailAddress,
        [string[]]$AccessUsers
    )
    
    # 共有メールボックスの作成
    $sharedMailbox = New-Mailbox -Name $Department -DisplayName $Department -Shared -PrimarySmtpAddress $EmailAddress
    Write-Host "✅ 共有メールボックス作成成功: $EmailAddress" -ForegroundColor Green
    
    # アクセス権の一括付与
    foreach ($user in $AccessUsers) {
        try {
            # フルアクセス権限（メール読み取り・管理）
            Add-MailboxPermission -Identity $EmailAddress -User $user -AccessRights FullAccess -InheritanceType All -AutoMapping $true
            
            # 送信権限
            Add-RecipientPermission -Identity $EmailAddress -Trustee $user -AccessRights SendAs -Confirm:$false
            
            Write-Host "✅ アクセス権付与成功: $user" -ForegroundColor Green
        }
        catch {
            Write-Host "❌ アクセス権付与エラー: $user - $($_.Exception.Message)" -ForegroundColor Red
        }
    }
    
    # 共有メールボックスの基本設定
    Set-Mailbox -Identity $EmailAddress -ProhibitSendQuota 50GB -ProhibitSendReceiveQuota 55GB -IssueWarningQuota 45GB
    
    return $sharedMailbox
}

# 使用例：小学部の共有メールボックス
$elemUsers = @("tanaka@school.edu.jp", "yamada@school.edu.jp", "suzuki@school.edu.jp")
New-EducationSharedMailbox -Department "小学部事務" -EmailAddress "elem-office@school.edu.jp" -AccessUsers $elemUsers

# 児童生徒支援室の共有メールボックス
$supportUsers = @("counselor1@school.edu.jp", "counselor2@school.edu.jp", "admin@school.edu.jp")
New-EducationSharedMailbox -Department "児童生徒支援室" -EmailAddress "student-support@school.edu.jp" -AccessUsers $supportUsers
```

#### 共有メールボックスの高度な設定

```powershell
# 自動返信とルール設定
Set-MailboxAutoReplyConfiguration -Identity "elem-office@school.edu.jp" -AutoReplyState Enabled -InternalMessage "
お問い合わせありがとうございます。
小学部事務室では、平日9:00-17:00にお返事いたします。
緊急の場合は、内線1234までお電話ください。
" -ExternalMessage "
Thank you for your inquiry.
Elementary Department Office will respond during weekdays 9:00-17:00 JST.
For urgent matters, please call extension 1234.
"

# 送信時制限の設定（学内のみに制限）
Set-Mailbox -Identity "student-support@school.edu.jp" -RequireSenderAuthenticationEnabled $false
```

#### 共有メールボックスの権限管理

```powershell
# 権限の確認
Get-MailboxPermission -Identity "elem-office@school.edu.jp" | Where-Object {$_.User -ne "NT AUTHORITY\SELF"}

# 権限の削除
Remove-MailboxPermission -Identity "elem-office@school.edu.jp" -User "olduser@school.edu.jp" -AccessRights FullAccess -Confirm:$false
Remove-RecipientPermission -Identity "elem-office@school.edu.jp" -Trustee "olduser@school.edu.jp" -AccessRights SendAs -Confirm:$false

# 部分的な権限付与（読み取り専用）
Add-MailboxPermission -Identity "elem-office@school.edu.jp" -User "readonly@school.edu.jp" -AccessRights ReadPermission -InheritanceType All
```

## 配布グループ管理

### 教育機関向け配布グループの作成

```powershell
# 学年別配布グループの作成
New-DistributionGroup -Name "小学部教職員" -DisplayName "小学部 教職員" -PrimarySmtpAddress "elem-faculty@school.edu.jp" -Type Distribution

# 学年別児童生徒グループ
New-DistributionGroup -Name "2024年度小学6年" -DisplayName "2024年度 小学6年" -PrimarySmtpAddress "grade6-2024@school.edu.jp" -Type Distribution

# メンバーの追加
Add-DistributionGroupMember -Identity "elem-faculty@school.edu.jp" -Member "teacher1@school.edu.jp"
Add-DistributionGroupMember -Identity "grade1-2024@school.edu.jp" -Member "student001@school.edu.jp"
```

### 動的配布グループ（教育機関向け）

```powershell
# 学年ベースの動的配布グループ
New-DynamicDistributionGroup -Name "小学部_動的" -DisplayName "小学部（動的）" -PrimarySmtpAddress "elem-dynamic@school.edu.jp" -RecipientFilter {Department -eq "小学部"}

# 学年ベースの動的配布グループ
New-DynamicDistributionGroup -Name "小学6年_動的" -DisplayName "小学6年（動的）" -PrimarySmtpAddress "sixth-grade@school.edu.jp" -RecipientFilter {CustomAttribute1 -eq "小学6年"}

# 教職員のみの動的グループ
New-DynamicDistributionGroup -Name "全教職員_動的" -DisplayName "全教職員（動的）" -PrimarySmtpAddress "all-faculty@school.edu.jp" -RecipientFilter {RecipientType -eq "UserMailbox" -and CustomAttribute2 -eq "教職員"}
```

## メールフロー管理

教育機関では、児童生徒の安全性確保と教職員のセキュリティ強化のため、特別なメールフロー制御が必要です。

### 🎓 教育機関向けメールフロー戦略

#### 1. 児童生徒の外部メール送受信禁止

児童生徒の安全を確保するため、外部ドメインとのメール送受信を完全に禁止します。

```powershell
# 児童生徒用の外部メール禁止ルール
function Set-StudentMailRestriction {
    param(
        [string[]]$StudentGroups = @("小学生", "中学生"),
        [string[]]$AllowedDomains = @("school.edu.jp")
    )
    
    foreach ($group in $StudentGroups) {
        # 外部への送信禁止ルール
        $outboundRuleName = "$group-外部送信禁止"
        New-TransportRule -Name $outboundRuleName `
            -FromMemberOf $group `
            -RecipientDomainIsNot $AllowedDomains `
            -RejectMessageEnhancedStatusCode "5.7.1" `
            -RejectMessageReasonText "学外へのメール送信は禁止されています。担任の先生にご相談ください。" `
            -Priority 1
            
        # 外部からの受信禁止ルール
        $inboundRuleName = "$group-外部受信禁止"
        New-TransportRule -Name $inboundRuleName `
            -SentToMemberOf $group `
            -SenderDomainIsNot $AllowedDomains `
            -DeleteMessage $true `
            -Priority 2
            
        Write-Host "✅ $group の外部メール制限を設定しました" -ForegroundColor Green
    }
}

# 実行例
Set-StudentMailRestriction

# 学年別の詳細制限設定
New-TransportRule -Name "小学生-完全外部禁止" `
    -FromMemberOf "小学生" `
    -RecipientDomainIsNot @("elementary.school.jp", "school.edu.jp") `
    -RejectMessageEnhancedStatusCode "5.7.1" `
    -RejectMessageReasonText "小学生は学校内でのみメール送信が可能です。" `
    -Priority 1

New-TransportRule -Name "中学生-教職員経由承認" `
    -FromMemberOf @("中学生") `
    -RecipientDomainIsNot @("school.edu.jp") `
    -ModerateMessageByUser @("teacher1@school.edu.jp", "teacher2@school.edu.jp") `
    -Priority 3
```

#### 2. 教職員の添付ファイル承認フロー

教職員が添付ファイル付きメールを外部に送信する際の承認フローを設定します。

```powershell
# 教職員の添付ファイル承認フロー設定
function Set-FacultyAttachmentApprovalFlow {
    param(
        [string[]]$FacultyGroups = @("教職員", "非常勤講師"),
        [string[]]$ApprovalManagers = @("principal@school.edu.jp", "vice-principal@school.edu.jp"),
        [string[]]$ExternalDomains = @(),  # 空の場合は全外部ドメインが対象
        [int]$AttachmentSizeThresholdMB = 5
    )
    
    # 大容量添付ファイルの承認フロー
    $largAttachmentRule = "教職員-大容量添付承認"
    New-TransportRule -Name $largAttachmentRule `
        -FromMemberOf $FacultyGroups `
        -AttachmentSizeOver "$($AttachmentSizeThresholdMB)MB" `
        -RecipientDomainIsNot @("school.edu.jp", "mext.go.jp") `
        -ModerateMessageByUser $ApprovalManagers `
        -SenderNotificationType Never `
        -RejectMessageEnhancedStatusCode "5.7.1" `
        -RejectMessageReasonText "大容量ファイル($($AttachmentSizeThresholdMB)MB以上)の外部送信は管理者の承認が必要です。"
    
    # 特定ファイル形式の承認フロー
    $sensitiveFileRule = "教職員-機密ファイル承認"
    New-TransportRule -Name $sensitiveFileRule `
        -FromMemberOf $FacultyGroups `
        -AttachmentNameMatchesPatterns @("*.xlsx", "*.docx", "*.pdf", "*.zip") `
        -RecipientDomainIsNot @("school.edu.jp", "mext.go.jp") `
        -ModerateMessageByUser $ApprovalManagers `
        -SenderNotificationType Never
    
    # 個人情報含有可能性のある添付ファイル
    $personalDataRule = "教職員-個人情報添付承認"
    New-TransportRule -Name $personalDataRule `
        -FromMemberOf $FacultyGroups `
        -AttachmentContainsWords @("成績", "評価", "個人情報", "学籍", "住所", "電話番号") `
        -RecipientDomainIsNot @("school.edu.jp") `
        -ModerateMessageByUser $ApprovalManagers `
        -RejectMessageEnhancedStatusCode "5.7.1" `
        -RejectMessageReasonText "個人情報を含む可能性のあるファイルの外部送信は事前承認が必要です。"
    
    Write-Host "✅ 教職員の添付ファイル承認フローを設定しました" -ForegroundColor Green
    Write-Host "📋 承認者: $($ApprovalManagers -join ', ')" -ForegroundColor Yellow
}

# 実行例
Set-FacultyAttachmentApprovalFlow -AttachmentSizeThresholdMB 10

# 緊急時の一時的な承認バイパス設定
New-TransportRule -Name "緊急時-添付承認バイパス" `
    -FromMemberOf @("校長", "教頭", "事務長") `
    -SenderIpRanges @("192.168.1.0/24") `
    -AttachmentSizeOver "50MB" `
    -RecipientDomainIsNot @("school.edu.jp") `
    -SetHeaderName "X-Bypass-Reason" `
    -SetHeaderValue "Emergency-Administrative" `
    -Priority 0 `
    -Enabled $false  # 通常は無効、緊急時のみ有効化
```

#### 3. 承認プロセスの管理と監視

```powershell
# 承認待ちメッセージの確認
function Get-PendingApprovalMessages {
    param(
        [int]$DaysBack = 7
    )
    
    $startDate = (Get-Date).AddDays(-$DaysBack)
    $endDate = Get-Date
    
    # 承認待ちメッセージトレース
    $pendingMessages = Get-MessageTrace -StartDate $startDate -EndDate $endDate -Status "Pending"
    
    Write-Host "=== 承認待ちメッセージ ($DaysBack日間) ===" -ForegroundColor Cyan
    foreach ($msg in $pendingMessages) {
        Write-Host "送信者: $($msg.SenderAddress)" -ForegroundColor Yellow
        Write-Host "受信者: $($msg.RecipientAddress)" -ForegroundColor Yellow
        Write-Host "件名: $($msg.Subject)" -ForegroundColor White
        Write-Host "日時: $($msg.Received)" -ForegroundColor Gray
        Write-Host "---" -ForegroundColor Gray
    }
    
    return $pendingMessages
}

# 承認決定の一括処理
function Process-PendingApprovals {
    param(
        [string]$Action = "Approve", # "Approve" or "Reject"
        [string]$ApproverEmail,
        [string]$Reason = "自動承認"
    )
    
    # 実装は環境に応じてカスタマイズ
    Write-Host "📧 承認処理: $Action - 理由: $Reason" -ForegroundColor Green
}

# 実行例
Get-PendingApprovalMessages -DaysBack 3
```

### 従来のトランスポートルール

```powershell
# 外部メール警告ルール
New-TransportRule -Name "外部メール警告" -SenderDomainIs "external.com" -PrependSubject "[EXTERNAL] "

# 大容量添付ファイル制限
New-TransportRule -Name "添付ファイル容量制限" -AttachmentSizeOver 25MB -RejectMessageEnhancedStatusCode "5.7.1" -RejectMessageReasonText "添付ファイルが25MBを超えています。"
```

### 承認済みドメイン

```powershell
# 新しい承認済みドメインの追加
New-AcceptedDomain -Name "contoso.com" -DomainName "contoso.com" -DomainType Authoritative
```

### メールフロー コネクタ

```powershell
# オンプレミス環境への送信コネクタ
New-OutboundConnector -Name "To OnPremises" -RecipientDomains "internal.contoso.com" -SmartHosts "mail.contoso.com" -TlsSettings EncryptionOnly
```

## セキュリティ機能

### マルウェア対策

```powershell
# マルウェア対策ポリシーの設定
Set-MalwareFilterPolicy -Identity Default -Action DeleteMessage -EnableFileFilter $true -FileTypes @("exe","bat","cmd","scr")
```

### スパム対策

```powershell
# スパム対策ポリシーの設定
Set-HostedContentFilterPolicy -Identity Default -SpamAction MoveToJmf -HighConfidenceSpamAction Quarantine -PhishSpamAction Quarantine -BulkThreshold 6
```

### セーフ リンク（Defender for Office 365）

```powershell
# セーフ リンク ポリシーの作成
New-SafeLinksPolicy -Name "会社ポリシー" -IsEnabled $true -AllowClickThrough $false -ScanUrls $true -EnableForInternalSenders $true
New-SafeLinksRule -Name "会社ルール" -SafeLinksPolicy "会社ポリシー" -RecipientDomainIs "contoso.com"
```

## メールボックス権限管理

### 代理アクセス

```powershell
# 代理送信権限の付与
Add-RecipientPermission -Identity "manager@contoso.com" -Trustee "assistant@contoso.com" -AccessRights SendAs

# 代理人の設定
Set-Mailbox -Identity "manager@contoso.com" -GrantSendOnBehalfTo "assistant@contoso.com"
```

### フォルダー権限

```powershell
# 予定表フォルダーの権限設定
Add-MailboxFolderPermission -Identity "manager@contoso.com:\Calendar" -User "assistant@contoso.com" -AccessRights Editor
```

## 監査とコンプライアンス

### メールボックス監査

```powershell
# メールボックス監査の有効化
Set-Mailbox -Identity "tanaka@contoso.com" -AuditEnabled $true -AuditLogAgeLimit 90

# 監査ログの確認
Search-MailboxAuditLog -Identity "tanaka@contoso.com" -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) -LogonTypes Owner,Delegate,Admin
```

### メッセージトレース

```powershell
# メッセージトレースの実行
Get-MessageTrace -SenderAddress "external@example.com" -RecipientAddress "internal@contoso.com" -StartDate (Get-Date).AddDays(-1) -EndDate (Get-Date)
```

## バックアップと復旧

### 削除済みアイテムの復旧

```powershell
# 削除済みメールボックスの復元
Get-Mailbox -SoftDeletedMailbox | Where-Object {$_.DisplayName -eq "田中太郎"}
Undo-SoftDeletedMailbox -SoftDeletedMailboxId "mailbox-guid" -WindowsLiveID "tanaka@contoso.com"
```

### 訴訟ホールド

```powershell
# 訴訟ホールドの有効化
Set-Mailbox -Identity "tanaka@contoso.com" -LitigationHoldEnabled $true -LitigationHoldDuration 2555 # 7年間
```

## モバイル デバイス管理

### ActiveSync ポリシー

```powershell
# モバイル デバイス メールボックス ポリシーの作成
New-MobileDeviceMailboxPolicy -Name "企業ポリシー" -RequireDeviceEncryption $true -RequireStorageCardEncryption $true -PasswordEnabled $true -MinPasswordLength 6

# ユーザーへの適用
Set-CASMailbox -Identity "tanaka@contoso.com" -MobileDeviceMailboxPolicy "企業ポリシー"
```

## レポート作成

### メールフロー レポート

```powershell
# 上位送信者レポート
Get-MailTrafficTopReport -Category TopMailSender -StartDate (Get-Date).AddDays(-30) -EndDate (Get-Date)

# マルウェア検出レポート
Get-MailTrafficSummaryReport -Category TopMalware -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date)
```

## 一括操作

### CSV ファイルを使用したメールボックス作成

```powershell
# CSV ファイルからメールボックスを一括作成
$users = Import-Csv -Path "C:\temp\mailboxes.csv"

foreach ($user in $users) {
    try {
        New-Mailbox -Name $user.Name -DisplayName $user.DisplayName -UserPrincipalName $user.UPN -Password (ConvertTo-SecureString -String $user.Password -AsPlainText -Force)
        Write-Host "作成成功: $($user.DisplayName)" -ForegroundColor Green
    }
    catch {
        Write-Host "作成エラー: $($user.DisplayName) - $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

## ベストプラクティス

### 🎓 教育機関特有のベストプラクティス

1. **児童生徒安全対策**
   - 児童生徒の外部メール完全禁止
   - 教職員による児童生徒メール監視体制
   - 不適切コンテンツのフィルタリング
   - 緊急連絡網の整備

2. **情報セキュリティ**
   - 個人情報を含むメールの承認フロー
   - 添付ファイルの自動スキャン
   - 外部ドメインへの送信制限
   - DLP（データ損失防止）ポリシーの徹底

3. **コンプライアンス**
   - 教育データのプライバシー保護
   - 保護者同意の管理
   - 法的要件（個人情報保護法等）への準拠
   - 監査ログの長期保管

4. **運用管理**
   - 学期ごとのアカウント整理
   - 卒業生アカウントの適切な処理
   - 共有メールボックスの定期見直し
   - メールフロールールの定期監査

### 一般的なベストプラクティス

1. **セキュリティ**
   - マルウェア・スパム対策の定期的な見直し
   - トランスポートルールの適切な設定
   - 監査ログの定期確認

2. **パフォーマンス**
   - メールボックス容量の適切な設定
   - 不要なメールの定期削除
   - アーカイブ機能の活用

3. **コンプライアンス**
   - データ保持ポリシーの設定
   - 法的要件への準拠
   - 監査ログの保管

## トラブルシューティング

### よくある問題

1. **メール配信の遅延**
   ```powershell
   # メッセージトレースで配信状況を確認
   Get-MessageTrace -MessageTraceId "trace-id"
   ```

2. **メールボックス容量超過**
   ```powershell
   # 容量使用状況の確認
   Get-MailboxStatistics -Identity "tanaka@contoso.com" | Select-Object TotalItemSize, ItemCount
   ```

## 注意事項

- PowerShell コマンド実行前は必ずテスト環境で確認
- 大量のデータ操作時は段階的に実行
- 重要な設定変更前はバックアップを取得

## 関連情報

- [Exchange Online 公式ドキュメント](https://docs.microsoft.com/ja-jp/exchange/exchange-online)
- [Exchange Online PowerShell](https://docs.microsoft.com/ja-jp/powershell/exchange/exchange-online-powershell)
- [メールフロー管理](docs/mail-flow-management.md)
