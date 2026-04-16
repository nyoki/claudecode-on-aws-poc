# Claude Code on AWS Bedrock — Identity Pool 方式

Cognito ユーザ認証 + Identity Pool を経由して Claude Code を AWS Bedrock 上で利用するためのツール。

共通の利用方法（Claude Code の設定、ログイン/ログアウト、トラブルシューティング等）は [ルートの README](../README.md) を参照。

## 構成概要

```
ユーザ (Claude Code)
  │
  │  bedrock-login (awsCredentialExport ヘルパー)
  │
  ├─ ブラウザで Cognito Hosted UI にログイン（PKCE 付き）
  │   └─ 認可コードをローカル HTTP サーバで受け取り
  │
  ├─ Cognito Identity Pool で AWS 一時クレデンシャルに変換
  │
  └─ Bedrock API を呼び出し
```

## ファイル構成

```
id-pool/
├── README.md       # このファイル
├── bedrock-login   # 認証ヘルパー（Claude Code の awsCredentialExport から呼ばれる）
├── bedrock-logout  # ログアウト
├── env.sh          # setup が自動生成（秘密情報を含まない）
└── admin/
    ├── setup         # AWS リソース作成
    ├── teardown      # AWS リソース削除
    └── add-user      # ユーザ追加
```

## 管理者: セットアップ

### 管理者の前提条件

- AWS CLI v2
- 環境変数 `ADMIN_AWS_PROFILE` に AWS プロファイル名を設定すること
- そのプロファイルに以下の権限があること:
  - `cognito-idp:*`（User Pool 操作）
  - `cognito-identity:*`（Identity Pool 操作）
  - `iam:CreateRole`, `iam:PutRolePolicy`, `iam:DeleteRole`, `iam:DeleteRolePolicy`

### 1. AWS リソースの作成

```bash
export ADMIN_AWS_PROFILE=your-profile
cd admin
./setup
```

全リソースが作成され、設定値が `../env.sh` に保存される。

`setup` が作成する AWS リソース:

| # | リソース | 用途 |
|---|---|---|
| 1 | Cognito User Pool | ユーザ管理・認証 |
| 2 | Cognito User Pool ドメイン | Hosted UI（ブラウザログイン画面） |
| 3 | Cognito User Pool Client | OAuth 2.0 code フロー（パブリッククライアント + PKCE） |
| 4 | Cognito Identity Pool | ID Token → AWS 一時クレデンシャル変換 |
| 5 | IAM ロール | Bedrock の InvokeModel 権限 |

### 2. ユーザの追加

```bash
./add-user user@example.com
```

- ランダムな仮パスワードを生成
- Cognito の招待メール機能でユーザに送信
- 管理者の画面にも仮パスワードが表示される（メール不達時のバックアップ）
- 初回ログイン時にパスワード変更が求められる

> Cognito のデフォルトメール送信上限は AWS アカウント全体で 50 通/日（全 User Pool 合計、調整不可）。大量追加が必要な場合は SES 連携を検討すること。

### 3. 後片付け

```bash
./teardown   # 確認プロンプト後、全リソースを削除
```

## 補足

- 両方式の違い（セッション時間、CloudTrail でのユーザ特定等）は [ルートの README](../README.md#2-つの方式) を参照
- Keychain サービス名は `com.claudecode.bedrock-login`
