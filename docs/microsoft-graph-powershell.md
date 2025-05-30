# Microsoft Graph PowerShell セットアップガイド

Microsoft Graph PowerShell を使用して Microsoft 365 環境を管理するための包括的なセットアップガイドです。

## 概要

Microsoft Graph PowerShell は、Microsoft Graph API にアクセスするための PowerShell モジュールです。Microsoft 365 のリソース（ユーザー、グループ、デバイス、アプリケーションなど）をプログラムで管理できます。

## 前提条件

### システム要件

- **PowerShell**: 5.1 以上（PowerShell 7.x 推奨）
- **オペレーティングシステム**:
  - Windows 10/11
  - Windows Server 2016 以降
  - Linux (Ubuntu, CentOS, RHEL)
  - macOS 10.13 以降
- **.NET Framework**: 4.7.2 以降
- **インターネット接続**: Microsoft Graph エンドポイントへのアクセス

### 必要な権限

- **Microsoft Entra ID テナント**: 管理者権限
- **PowerShell 実行ポリシー**: RemoteSigned 以上
- **ネットワーク**: HTTPSアウトバウンド接続（ポート443）

## インストール手順

### 1. PowerShell バージョンの確認

**処理内容:** システムの PowerShell バージョンと .NET Framework のバージョンを確認し、Microsoft Graph PowerShell の動作要件を満たしているかを検証します。

**詳細な処理説明:**

1. `$PSVersionTable.PSVersion` - PowerShellの自動変数から現在のバージョン情報を取得
2. Windows環境では、レジストリから.NET Frameworkのリリース番号を取得
3. バージョン情報を分析して、Microsoft Graph PowerShell の最小要件（PowerShell 5.1+、.NET Framework 4.7.2+）を満たしているかを確認

**コマンド詳細解説:**

- `$PSVersionTable` : PowerShellセッションの詳細情報を含む自動変数
- `Get-ItemProperty` : レジストリキーのプロパティ値を取得するコマンドレット
- `"HKLM:SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full\"` : .NET Framework 4.x系の情報が格納されているレジストリパス

**実行前の準備:**

- PowerShell 5.1 以上または PowerShell 7.x の環境
- Windows 環境では管理者権限でレジストリアクセスが可能

**期待される結果:**

- PowerShell のメジャー・マイナーバージョン情報の表示（例：7.3.0）
- .NET Framework のリリース番号表示（Windows環境、例：528040）
- バージョンが要件を満たしていない場合のエラーメッセージ

**以下のPowerShellコードの処理内容:**

1. `$PSVersionTable.PSVersion` - PowerShell自動変数からバージョン情報を取得し、メジャー・マイナー・ビルド番号を表示
2. `Get-ItemProperty` - Windowsレジストリから.NET Framework 4.x系のリリース番号を取得
3. バージョン確認により、Microsoft Graph PowerShell の最小要件（PowerShell 5.1+、.NET 4.7.2+）を満たしているかを判定

```powershell
# PowerShell バージョンの確認
$PSVersionTable.PSVersion

# .NET Framework バージョンの確認（Windows のみ）
Get-ItemProperty "HKLM:SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full\" -Name Release
```

### 2. 実行ポリシーの設定

**処理内容:** PowerShell の実行ポリシーを確認・変更し、外部スクリプトやモジュールの実行を可能にします。Microsoft Graph PowerShell モジュールのインストールと実行に必要な設定です。

**詳細な処理説明:**

1. `Get-ExecutionPolicy` - 現在のPowerShell実行ポリシーを確認
2. PowerShellがスクリプトやモジュールの実行を許可する範囲を制御
3. Microsoft Graph PowerShellモジュールのインストールと実行には RemoteSigned 以上が必要

**パラメータ説明:**

- `Get-ExecutionPolicy`: 現在の実行ポリシー設定を表示
- `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned`: リモートからダウンロードしたスクリプトの署名を要求
- `-Scope CurrentUser`: 現在のユーザーのみに設定を適用（管理者権限不要）

**実行前の準備:**

- 管理者権限（システム全体の設定変更の場合）
- または現在ユーザーの権限（CurrentUser スコープの場合）

**期待される結果:**

- 現在の実行ポリシー表示（例：Restricted、RemoteSigned）
- 実行ポリシーの変更確認メッセージ

**以下のPowerShellコードの処理内容:**

1. `Get-ExecutionPolicy` - 現在のPowerShell実行ポリシー設定を表示し、スクリプト実行の制限レベルを確認
2. `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned` - 実行ポリシーをRemoteSignedに変更、リモートからダウンロードしたスクリプトの署名を要求
3. `-Scope CurrentUser` - 変更を現在のユーザーのみに適用し、管理者権限なしで設定可能

```powershell
# 現在の実行ポリシーを確認
Get-ExecutionPolicy

# 実行ポリシーの変更（管理者権限が必要）
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### 3. Microsoft Graph PowerShell モジュールのインストール

#### オプション A: 全モジュールのインストール（推奨）

**処理内容:** Microsoft Graph PowerShell SDK 全体をインストールし、Microsoft 365 のすべてのリソースにアクセス可能なコマンドレットを使用できるようにします。

**パラメータ説明:**

- `Install-Module Microsoft.Graph`: Microsoft Graph PowerShell SDK のメインモジュール
- `-Scope CurrentUser`: 現在のユーザーのみにインストール（管理者権限不要）
- `-Repository PSGallery`: PowerShell Gallery からダウンロード
- `-Force`: 既存モジュールの上書きインストール

**処理内容:** Microsoft Graph PowerShell SDK の全モジュールを一括でインストールします。包括的な Microsoft 365 管理機能にアクセスできる推奨方法です。

**詳細な処理説明:**

1. `Install-Module Microsoft.Graph` - Microsoft Graph PowerShell SDK のメインモジュールをインストール
2. PowerShell Gallery から最新版のモジュールをダウンロード
3. 認証、ユーザー管理、グループ管理、デバイス管理など全機能が含まれる
4. 依存関係のあるサブモジュールも自動的にインストール

**パラメータ説明:**

- `-Scope CurrentUser`: 現在のユーザーのみにインストール（管理者権限不要）
- `-Repository PSGallery`: PowerShell Gallery からダウンロード
- `-Force`: 確認なしでインストールを実行

**実行前の準備:**

- インターネット接続（PowerShell Gallery へのアクセス）
- PowerShell 5.1 以上または PowerShell 7.x
- 十分なディスク容量（約 200MB）

**期待される結果:**

- Microsoft Graph PowerShell SDK の全モジュールがインストール
- Get-InstalledModule でインストール状況の確認
- 数百のコマンドレットが利用可能

**以下のPowerShellコードの処理内容:**

1. `Install-Module Microsoft.Graph` - Microsoft Graph PowerShell SDK全体をPowerShell Galleryからダウンロード・インストール
2. `-Scope CurrentUser` - 現在のユーザーのみにインストールし、管理者権限不要でインストール実行
3. `-Repository PSGallery -Force` - 公式のPowerShell Galleryリポジトリから強制的にインストール、確認プロンプトを省略
4. `Get-InstalledModule Microsoft.Graph` - インストール完了の確認とバージョン情報の表示

```powershell
# Microsoft Graph PowerShell SDK のインストール
Install-Module Microsoft.Graph -Scope CurrentUser -Repository PSGallery -Force

# インストールの確認
Get-InstalledModule Microsoft.Graph
```

#### オプション B: 個別モジュールのインストール

**処理内容:** 必要な機能のみを個別にインストールし、システムリソースの使用量を最小限に抑えます。特定の管理タスクに限定する場合に有効です。

**詳細な処理説明:**

1. `Microsoft.Graph.Authentication` - すべての Graph モジュールの基盤となる認証機能（必須）
2. 各機能別モジュールを個別にインストールしてディスク容量を節約
3. 必要な機能のみをロードすることでパフォーマンスを向上
4. 段階的な導入やテスト環境での利用に適している

**モジュール説明:**

- `Microsoft.Graph.Authentication`: 認証機能（必須）- Microsoft Graph API への接続と認証処理
- `Microsoft.Graph.Users`: ユーザー管理機能 - ユーザーアカウントの作成・編集・削除
- `Microsoft.Graph.Groups`: グループ管理機能 - セキュリティグループ・Microsoft 365グループの管理
- `Microsoft.Graph.DeviceManagement`: デバイス管理機能 - Intune デバイス管理とポリシー設定
- `Microsoft.Graph.Identity.SignIns`: サインイン・ID 管理機能 - 認証ログとアクセス監査

**実行前の準備:**

- 使用する機能の事前計画
- インターネット接続
- モジュール間の依存関係の理解

**期待される結果:**

- 選択したモジュールのみがインストール
- ディスク使用量の最適化（約30-50MB）
- 必要な機能のコマンドレットが利用可能

```powershell
# 特定の機能のみ必要な場合
Install-Module Microsoft.Graph.Authentication -Scope CurrentUser -Force
Install-Module Microsoft.Graph.Users -Scope CurrentUser -Force
Install-Module Microsoft.Graph.Groups -Scope CurrentUser -Force
Install-Module Microsoft.Graph.DeviceManagement -Scope CurrentUser -Force
Install-Module Microsoft.Graph.Identity.SignIns -Scope CurrentUser -Force
```

### 4. モジュールの確認

**処理内容:** インストールされたMicrosoft Graph PowerShell モジュールの一覧と利用可能なコマンドレットを確認し、正常にインストールが完了したかを検証します。

**詳細な処理説明:**

1. `Get-Module Microsoft.Graph* -ListAvailable` - インストール済みの全Graph関連モジュールとバージョン情報を取得
2. `Get-Command -Module Microsoft.Graph*` - 利用可能な全コマンドレットを列挙して機能確認
3. モジュールの依存関係とバージョン互換性をチェック
4. インストールの完全性とアクセス可能性を検証

**コマンド説明:**

- `Get-Module Microsoft.Graph* -ListAvailable`: インストール済みのGraph関連モジュールを表示
- `Get-Command -Module Microsoft.Graph*`: 利用可能なコマンドレットの一覧表示
- `Get-Command -Module Microsoft.Graph* | Measure-Object`: コマンドレット総数の確認

**実行前の準備:**

- Microsoft Graph PowerShell モジュールのインストール完了

**期待される結果:**

- インストールされたモジュールのバージョン情報表示
- 利用可能なコマンドレットの総数確認（通常数百から千以上）
- モジュールの正常性確認

**以下のPowerShellコードの処理内容:**

1. `Get-Module Microsoft.Graph* -ListAvailable` - インストール済みの全Graph関連モジュールとバージョン情報を一覧表示
2. `Get-Command -Module Microsoft.Graph*` - 利用可能な全コマンドレットを列挙してGraph機能の確認
3. `Measure-Object` - コマンドレット総数のカウントでインストールの完全性を検証

```powershell
# インストール済みモジュールの確認
Get-Module Microsoft.Graph* -ListAvailable

# 利用可能なコマンドの確認
Get-Command -Module Microsoft.Graph*

# コマンドレット数の確認
Get-Command -Module Microsoft.Graph* | Measure-Object | Select-Object Count
```

- インストールされたモジュールのバージョン情報表示
- 利用可能なコマンドレットの総数確認（通常数百から千以上）
- モジュールの正常性確認

```powershell
# インストール済みモジュールの確認
Get-Module Microsoft.Graph* -ListAvailable

# 利用可能なコマンドの確認
Get-Command -Module Microsoft.Graph*
```

## 初期設定と認証

### 1. 基本的な接続

**処理内容:** Microsoft Graph API への最も基本的な接続を行い、現在の認証状況とユーザー情報を確認します。初回実行時はブラウザでの認証が必要です。

**詳細な処理説明:**

1. `Connect-MgGraph` - Microsoft Graph API への認証プロセスを開始
2. ブラウザが自動起動し、Microsoft 365 ログイン画面が表示
3. 認証成功後、アクセストークンがローカルに保存される
4. `Get-MgContext` でセッション情報（テナントID、アカウント、権限）を確認
5. 接続中のユーザーアカウントの詳細情報を取得

**コマンド説明:**

- `Connect-MgGraph`: Microsoft Graph への認証・接続を実行
- `Get-MgContext`: 現在の接続コンテキスト（テナント情報、権限など）を表示
- `Get-MgUser -UserId (Get-MgContext).Account`: 接続中のユーザーアカウント情報を取得

**実行前の準備:**

- Microsoft 365 テナントの有効なユーザーアカウント
- Web ブラウザでの認証画面アクセス
- 基本的な Graph API 権限

**期待される結果:**

- ブラウザでの認証画面表示と成功
- テナント ID、アカウント情報、取得済み権限の表示
- 接続ユーザーの詳細情報（DisplayName、UPN など）

```powershell
# Microsoft Graph への接続
Connect-MgGraph

# 接続状況の確認
Get-MgContext

# 現在のユーザー情報を取得
Get-MgUser -UserId (Get-MgContext).Account
```

### 2. 管理者同意での接続

**処理内容:** 管理者レベルの権限を要求して Microsoft Graph に接続し、ユーザーやグループの読み書き権限を取得します。組織全体の管理作業に必要な高レベル権限を取得できます。

**パラメータ説明:**

- `-Scopes`: 要求する権限の範囲を指定
  - `User.ReadWrite.All`: 全ユーザーの読み書き権限
  - `Group.ReadWrite.All`: 全グループの読み書き権限
  - `Directory.ReadWrite.All`: ディレクトリ全体の読み書き権限
- `-UseDeviceAuthentication`: デバイス認証フローを使用（サーバー環境等で有効）

**実行前の準備:**

- Microsoft 365 テナントの管理者権限
- 管理者による事前の同意プロセス
- 組織のセキュリティポリシーへの準拠

**期待される結果:**

- 管理者権限での認証成功
- 高レベル権限の取得確認
- 組織全体のリソースへのアクセス許可

```powershell
# 管理者同意を含む接続
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All" -UseDeviceAuthentication

# または、対話的認証
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All"
```

### 3. スコープ（権限）の指定

**処理内容:** Microsoft Graph API の特定の権限スコープを事前に定義し、包括的な管理権限を一度に要求します。複数の管理タスクを効率的に実行するための高度な権限設定です。

**詳細な処理説明:**

1. 必要な権限スコープを配列として事前定義
2. `Connect-MgGraph -Scopes` で複数の権限を一度に要求
3. テナント管理者による組織レベルでの権限承認が必要
4. 権限取得後は指定したスコープ内でのAPI操作が可能
5. セッション中は取得した権限が維持される

**権限スコープ説明:**

- `User.ReadWrite.All`: ユーザーアカウントの作成・更新・削除
- `Group.ReadWrite.All`: グループの作成・管理・メンバーシップ変更
- `Directory.ReadWrite.All`: ディレクトリ全体の読み書きアクセス
- `DeviceManagementManagedDevices.ReadWrite.All`: Intune デバイス管理
- `Policy.ReadWrite.ConditionalAccess`: 条件付きアクセスポリシー管理
- `RoleManagement.ReadWrite.Directory`: 管理者ロールの割り当て・管理

**実行前の準備:**

- テナント管理者権限
- 各権限に対する組織承認
- セキュリティポリシーとの整合性確認

**期待される結果:**

- 指定したすべての権限スコープの取得
- 包括的な Microsoft 365 管理権限の確立
- 複数の管理タスクの実行準備完了

```powershell
# 特定のスコープでの接続
$scopes = @(
    "User.ReadWrite.All"
    "Group.ReadWrite.All"
    "Directory.ReadWrite.All"
    "DeviceManagementManagedDevices.ReadWrite.All"
    "Policy.ReadWrite.ConditionalAccess"
    "RoleManagement.ReadWrite.Directory"
)

Connect-MgGraph -Scopes $scopes
```
- `Policy.ReadWrite.ConditionalAccess`: 条件付きアクセスポリシー管理
- `RoleManagement.ReadWrite.Directory`: 管理者ロールの割り当て・管理

**実行前の準備:**

- テナント管理者権限
- 各権限に対する組織承認
- セキュリティポリシーとの整合性確認

**期待される結果:**

- 指定したすべての権限スコープの取得
- 包括的な Microsoft 365 管理権限の確立
- 複数の管理タスクの実行準備完了

```powershell
# 特定のスコープでの接続
$scopes = @(
    "User.ReadWrite.All"
    "Group.ReadWrite.All"
    "Directory.ReadWrite.All"
    "DeviceManagementManagedDevices.ReadWrite.All"
    "Policy.ReadWrite.ConditionalAccess"
    "RoleManagement.ReadWrite.Directory"
)

Connect-MgGraph -Scopes $scopes
```

## アプリケーション登録（上級者向け）

### 1. Microsoft Entra ID アプリケーションの作成

**処理内容:** Microsoft Entra ID に新しいアプリケーション登録を作成し、PowerShell スクリプトからの自動認証に使用できるアプリケーション ID を取得します。無人実行やサービスアカウントでの運用に必要です。

**パラメータ説明:**

- `New-MgApplication`: 新しいアプリケーション登録を作成
- `-DisplayName`: アプリケーションの表示名（管理画面で識別用）
- `-SignInAudience AzureADMyOrg`: 単一テナント（組織内のみ）でのサインインに制限

**実行前の準備:**

- Microsoft Entra ID の管理者権限
- Application.ReadWrite.All または Application.ReadWrite.OwnedBy 権限
- アプリケーション名の事前計画

**期待される結果:**

- 新しいアプリケーション登録の作成
- アプリケーション ID（クライアント ID）の取得
- オブジェクト ID の取得（後続の設定で使用）

**注意事項:**

- 取得したアプリケーション ID は安全に保管
- 適切な権限のみを付与（最小権限の原則）

```powershell
# アプリケーション登録
$app = New-MgApplication -DisplayName "Microsoft Graph PowerShell App" -SignInAudience AzureADMyOrg

# アプリケーション ID の確認
Write-Host "Application ID: $($app.AppId)"
Write-Host "Object ID: $($app.Id)"
```

### 2. API アクセス許可の設定

```powershell
# Microsoft Graph API のサービスプリンシパルを取得
$graphSP = Get-MgServicePrincipal -Filter "DisplayName eq 'Microsoft Graph'"

# 必要な権限の設定
$permissions = @(
    @{
        Id = "df021288-bdef-4463-88db-98f22de89214"  # User.Read.All
        Type = "Role"
    },
    @{
        Id = "62a82d76-70ea-41e2-9197-370581804d09"  # Group.ReadWrite.All
        Type = "Role"
    }
)

# API アクセス許可の追加
foreach ($permission in $permissions) {
    $requiredResourceAccess = @{
        ResourceAppId = $graphSP.AppId
        ResourceAccess = @($permission)
    }
    
    Update-MgApplication -ApplicationId $app.Id -RequiredResourceAccess $requiredResourceAccess
}
```

### 3. 証明書ベース認証の設定

```powershell
# 自己署名証明書の作成（テスト環境用）
$cert = New-SelfSignedCertificate -Subject "CN=GraphPowerShellCert" -CertStoreLocation "Cert:\CurrentUser\My" -KeyExportPolicy Exportable -KeySpec Signature

# 証明書の公開キーをアプリケーションに追加
$keyCredential = @{
    Type = "AsymmetricX509Cert"
    Usage = "Verify"
    Key = $cert.RawData
}

Update-MgApplication -ApplicationId $app.Id -KeyCredentials $keyCredential

# 証明書を使用した接続
Connect-MgGraph -ClientId $app.AppId -TenantId "your-tenant-id" -CertificateThumbprint $cert.Thumbprint
```

## よく使用するコマンド例

### ユーザー管理

```powershell
# 全ユーザーの取得
Get-MgUser -All

# 特定ユーザーの詳細情報
Get-MgUser -UserId "user@contoso.com" -Property DisplayName,Department,JobTitle,Manager

# 新しいユーザーの作成
$passwordProfile = @{
    ForceChangePasswordNextSignIn = $true
    Password = "TempPassword123!"
}

$newUser = @{
    DisplayName = "新規 ユーザー"
    UserPrincipalName = "newuser@contoso.com"
    MailNickname = "newuser"
    PasswordProfile = $passwordProfile
    AccountEnabled = $true
}

New-MgUser @newUser

# ユーザー情報の更新
Update-MgUser -UserId "user@contoso.com" -Department "営業部" -JobTitle "マネージャー"

# ユーザーの削除
Remove-MgUser -UserId "user@contoso.com"
```

### グループ管理

```powershell
# 全グループの取得
Get-MgGroup -All

# セキュリティグループの作成
$group = @{
    DisplayName = "営業部グループ"
    Description = "営業部のメンバー"
    GroupTypes = @()
    SecurityEnabled = $true
    MailEnabled = $false
    MailNickname = "sales-group"
}

New-MgGroup @group

# グループメンバーの追加
New-MgGroupMember -GroupId "group-id" -DirectoryObjectId "user-id"

# グループメンバーの確認
Get-MgGroupMember -GroupId "group-id"
```

### ライセンス管理

```powershell
# 利用可能なライセンスの確認
Get-MgSubscribedSku | Select-Object SkuPartNumber, ConsumedUnits, PrepaidUnits

# ユーザーへのライセンス割り当て
$license = @{
    AddLicenses = @(
        @{
            SkuId = "sku-id-here"
        }
    )
    RemoveLicenses = @()
}

Set-MgUserLicense -UserId "user@contoso.com" -BodyParameter $license

# ユーザーのライセンス確認
Get-MgUser -UserId "user@contoso.com" -Property AssignedLicenses
```

## トラブルシューティング

### 1. 接続エラー

```powershell
# 現在の接続状況を確認
Get-MgContext

# 接続をリセット
Disconnect-MgGraph
Connect-MgGraph

# デバッグモードでの接続
Connect-MgGraph -Debug
```

### 2. 権限エラー

```powershell
# 現在のスコープ（権限）を確認
(Get-MgContext).Scopes

# 不足している権限がある場合は再接続
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All", "Directory.ReadWrite.All"
```

### 3. モジュールの更新

```powershell
# モジュールのバージョン確認
Get-InstalledModule Microsoft.Graph

# 最新版への更新
Update-Module Microsoft.Graph -Force

# 古いバージョンの削除
Uninstall-Module Microsoft.Graph -AllVersions -Force
Install-Module Microsoft.Graph -Force
```

### 4. キャッシュのクリア

```powershell
# Microsoft Entra ID 認証キャッシュのクリア
Clear-AzContext -Force

# PowerShell モジュールキャッシュのクリア
Remove-Module Microsoft.Graph* -Force
Import-Module Microsoft.Graph
```

## セキュリティベストプラクティス

### 1. 最小権限の原則

```powershell
# 必要最小限のスコープのみを指定
$minimalScopes = @(
    "User.Read.All"      # ユーザー情報の読み取りのみ
    "Group.Read.All"     # グループ情報の読み取りのみ
)

Connect-MgGraph -Scopes $minimalScopes
```

### 2. セッション管理

```powershell
# セッション情報の確認
Get-MgContext | Select-Object Account, Scopes, TenantId

# 作業完了後の切断
Disconnect-MgGraph

# タイムアウト設定
$context = Get-MgContext
if ($context -and (Get-Date) -gt $context.ExpiresOn) {
    Write-Warning "セッションが期限切れです。再接続してください。"
    Disconnect-MgGraph
}
```

### 3. ログとモニタリング

```powershell
# 操作ログの有効化
$DebugPreference = "Continue"
$VerbosePreference = "Continue"

# 重要な操作の記録
function Write-AuditLog {
    param($Action, $Target, $Result)
    
    $logEntry = @{
        Timestamp = Get-Date
        User = (Get-MgContext).Account
        Action = $Action
        Target = $Target
        Result = $Result
    }
    
    $logEntry | Export-Csv -Path "C:\Logs\GraphOperations.csv" -Append -NoTypeInformation
}

# 使用例
try {
    $user = New-MgUser @newUserParams
    Write-AuditLog -Action "CreateUser" -Target $newUserParams.UserPrincipalName -Result "Success"
}
catch {
    Write-AuditLog -Action "CreateUser" -Target $newUserParams.UserPrincipalName -Result "Failed: $($_.Exception.Message)"
}
```

## 自動化スクリプトの例

### 1. 新入社員のオンボーディング

```powershell
function New-EmployeeOnboarding {
    param(
        [Parameter(Mandatory)]
        [string]$FirstName,
        
        [Parameter(Mandatory)]
        [string]$LastName,
        
        [Parameter(Mandatory)]
        [string]$Department,
        
        [Parameter(Mandatory)]
        [string]$JobTitle,
        
        [string]$ManagerUPN
    )
    
    # ユーザープリンシパル名の生成
    $userName = "$FirstName.$LastName".ToLower()
    $userPrincipalName = "$userName@contoso.com"
    
    # パスワードの生成
    $tempPassword = -join ((65..90) + (97..122) + (48..57) | Get-Random -Count 12 | ForEach-Object {[char]$_})
    
    # ユーザーの作成
    $passwordProfile = @{
        ForceChangePasswordNextSignIn = $true
        Password = $tempPassword
    }
    
    $userParams = @{
        DisplayName = "$FirstName $LastName"
        UserPrincipalName = $userPrincipalName
        MailNickname = $userName
        PasswordProfile = $passwordProfile
        AccountEnabled = $true
        Department = $Department
        JobTitle = $JobTitle
    }
    
    try {
        $newUser = New-MgUser @userParams
        Write-Host "ユーザー作成成功: $userPrincipalName" -ForegroundColor Green
        
        # 部署グループへの追加
        $departmentGroup = Get-MgGroup -Filter "displayName eq '$Department'"
        if ($departmentGroup) {
            New-MgGroupMember -GroupId $departmentGroup.Id -DirectoryObjectId $newUser.Id
            Write-Host "部署グループに追加: $Department" -ForegroundColor Green
        }
        
        # マネージャーの設定
        if ($ManagerUPN) {
            $manager = Get-MgUser -UserId $ManagerUPN
            Set-MgUserManagerByRef -UserId $newUser.Id -BodyParameter @{"@odata.id" = "https://graph.microsoft.com/v1.0/users/$($manager.Id)"}
            Write-Host "マネージャー設定: $ManagerUPN" -ForegroundColor Green
        }
        
        # 結果の返却
        return @{
            User = $newUser
            TempPassword = $tempPassword
            Success = $true
        }
    }
    catch {
        Write-Error "ユーザー作成エラー: $($_.Exception.Message)"
        return @{
            Success = $false
            Error = $_.Exception.Message
        }
    }
}

# 使用例
$result = New-EmployeeOnboarding -FirstName "太郎" -LastName "田中" -Department "営業部" -JobTitle "営業担当" -ManagerUPN "manager@contoso.com"
```

### 2. 一括ライセンス割り当て

```powershell
function Set-BulkLicenseAssignment {
    param(
        [Parameter(Mandatory)]
        [string]$CsvPath,
        
        [Parameter(Mandatory)]
        [string]$LicenseSkuPartNumber
    )
    
    # ライセンス SKU ID の取得
    $sku = Get-MgSubscribedSku | Where-Object {$_.SkuPartNumber -eq $LicenseSkuPartNumber}
    if (-not $sku) {
        Write-Error "指定されたライセンス SKU が見つかりません: $LicenseSkuPartNumber"
        return
    }
    
    # CSV ファイルの読み込み
    $users = Import-Csv -Path $CsvPath
    $results = @()
    
    foreach ($user in $users) {
        try {
            $license = @{
                AddLicenses = @(@{ SkuId = $sku.SkuId })
                RemoveLicenses = @()
            }
            
            Set-MgUserLicense -UserId $user.UserPrincipalName -BodyParameter $license
            
            $results += @{
                User = $user.UserPrincipalName
                Status = "成功"
                License = $LicenseSkuPartNumber
            }
            
            Write-Host "ライセンス割り当て成功: $($user.UserPrincipalName)" -ForegroundColor Green
        }
        catch {
            $results += @{
                User = $user.UserPrincipalName
                Status = "失敗"
                Error = $_.Exception.Message
            }
            
            Write-Warning "ライセンス割り当て失敗: $($user.UserPrincipalName) - $($_.Exception.Message)"
        }
    }
    
    # 結果をCSVで出力
    $results | Export-Csv -Path "C:\temp\license-assignment-results.csv" -NoTypeInformation -Encoding UTF8
    return $results
}

# 使用例
Set-BulkLicenseAssignment -CsvPath "C:\temp\users.csv" -LicenseSkuPartNumber "ENTERPRISEPACK_FACULTY"
```

## パフォーマンス最適化

### 1. バッチ処理

```powershell
# 大量データの効率的な取得
$users = Get-MgUser -All -Property DisplayName,Department,JobTitle -PageSize 999

# フィルタリングの活用
$salesUsers = Get-MgUser -Filter "department eq '営業部'" -Property DisplayName,UserPrincipalName

# 並列処理（PowerShell 7以降）
$userIds = @("user1@contoso.com", "user2@contoso.com", "user3@contoso.com")
$userIds | ForEach-Object -Parallel {
    $user = Get-MgUser -UserId $_
    Write-Host "処理完了: $($user.DisplayName)"
} -ThrottleLimit 5
```

### 2. キャッシュの活用

```powershell
# セッション中のデータキャッシュ
if (-not $global:GroupCache) {
    $global:GroupCache = @{}
    Get-MgGroup -All | ForEach-Object {
        $global:GroupCache[$_.DisplayName] = $_.Id
    }
}

# キャッシュからグループIDを取得
$groupId = $global:GroupCache["営業部"]
```

## 注意事項

### 重要な制限事項

1. **レート制限**: Microsoft Graph API には呼び出し回数の制限があります
2. **権限管理**: 必要最小限の権限のみを使用してください
3. **データ保護**: 機密情報の取り扱いには十分注意してください
4. **セッション管理**: 長時間の操作では定期的な再認証が必要です

### 推奨事項

- 本番環境での作業前は必ずテスト環境で検証
- 重要な操作の前はバックアップの取得
- 定期的なモジュールの更新
- 監査ログの確認

## 参考資料

### 公式ドキュメント

- [Microsoft Graph PowerShell SDK](https://docs.microsoft.com/ja-jp/powershell/microsoftgraph/)
- [Microsoft Graph API リファレンス](https://docs.microsoft.com/ja-jp/graph/api/overview)
- [Microsoft Graph Explorer](https://developer.microsoft.com/ja-jp/graph/graph-explorer)

### コミュニティリソース

- [Microsoft Graph PowerShell GitHub](https://github.com/microsoftgraph/msgraph-sdk-powershell)
- [Microsoft Tech Community](https://techcommunity.microsoft.com/t5/microsoft-graph/ct-p/microsoftgraph)

## 関連情報

- [📋 Microsoft Graph PowerShell 実践例集](microsoft-graph-practical-examples.md)
- [👥 ユーザー管理](user-management.md)
- [🔐 セキュリティ管理](security-management.md)
- [📱 Intune 端末管理](intune-device-management.md)
- [🎫 ライセンス管理](license-management.md)
