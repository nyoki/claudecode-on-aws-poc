# Claude Code on Amazon Bedrock — Cognito 認証 PoC

Cognito ユーザ認証を経由して Claude Code を Amazon Bedrock 上で利用するための PoC スクリプト集。

IAM ユーザを作らずに、Cognito のユーザ管理だけで Bedrock を利用できる。Claude Code の `awsCredentialExport` 機能を使い、`~/.aws/` に一切触れない構成。

## 2 つの方式

| 項目 | [Identity Pool 方式](./id-pool/) | [IAM OIDC Federation 方式](./oidc/) |
|---|---|---|
| クレデンシャル取得 | Cognito Identity Pool 経由 | STS `AssumeRoleWithWebIdentity` 直接 |
| セッション時間 | 1 時間（固定） | 最大 12 時間（設定可能） |
| CloudTrail でのユーザ特定 | 間接的（`CognitoIdentityCredentials` と記録される） | 直接的（メールアドレスが記録される） |
| AWS リソース | User Pool, ドメイン, Client, **Identity Pool**, IAM ロール | User Pool, ドメイン, Client, **IAM OIDC Provider**, IAM ロール |
| ロールマッピング | 対応（グループ/属性ベース） | 非対応 |

**選び方:** 監査ログの追跡性や長時間セッションが必要なら IAM OIDC Federation 方式、グループベースのロールマッピングが必要なら Identity Pool 方式を選ぶ。

今回実装した両方式は Keychain サービス名が異なるため、同一マシンで併用できる。

### セッション時間の違い

Identity Pool 方式では AWS クレデンシャルの有効期限が 1 時間固定だが、OIDC Federation 方式では IAM ロールの `MaxSessionDuration`（デフォルト 12 時間）まで延長できる。

### CloudTrail でのユーザ特定

OIDC Federation 方式では、`bedrock-login` が ID Token からメールアドレスを抽出し、STS の `RoleSessionName` に設定する。これにより CloudTrail の `userIdentity.principalId` に以下のような形式で記録される:

```
AROAXXXXXXXXXXXXXXXXX:user@example.com
```

Identity Pool 方式では `CognitoIdentityCredentials` としか記録されず、個別ユーザの特定には追加の調査が必要になる。

## 前提条件

- macOS（Keychain + `open` コマンドを使用。Windows / Linux でも同等の仕組みで実現可能）
- AWS CLI v2
- Python 3
- curl

## クイックスタート

### 管理者

```bash
export ADMIN_AWS_PROFILE=your-profile

# Identity Pool 方式の場合
cd id-pool/admin && ./setup && ./add-user user@example.com

# IAM OIDC Federation 方式の場合
cd oidc/admin && ./setup && ./add-user user@example.com
```

各方式の AWS リソースや管理者の前提条件は方式ごとの README を参照:

- [Identity Pool 方式](./id-pool/README.md)
- [IAM OIDC Federation 方式](./oidc/README.md)

### 利用者: Claude Code の設定

Bedrock を利用したいプロジェクトの `.claude/settings.json` に以下を設定:

```json
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "1",
    "AWS_REGION": "ap-northeast-1"
  },
  "awsCredentialExport": "/path/to/<id-pool or oidc>/bedrock-login"
}
```

`awsCredentialExport` はクローンした場所に合わせてフルパスで指定する。

### 利用者: ログイン / ログアウト

```bash
# ログイン（ブラウザが開く）
bedrock-login

# Claude Code 起動
claude

# ログアウト（Keychain クリア + ブラウザセッションをクリア）
bedrock-logout

# ログアウト（Keychain クリアのみ、ブラウザのセッションは維持）
bedrock-logout --local
```

初回ログイン時は管理者から通知された仮パスワードでログインし、パスワード変更を求められる。ログアウト後に Claude Code を起動すると認証エラーになる。再利用するには `bedrock-login` を手動で実行して再認証する。

## bedrock-login のオプション

| オプション | 説明 |
|---|---|
| (なし) | 認証してクレデンシャルを取得。ターミナルではキーを非表示 |
| `--verbose` | クレデンシャル JSON をターミナルにも表示 |
| `--clear` | Keychain のトークンとログアウトフラグをクリア |

## トークンのライフサイクル

| トークン | 有効期限 | 期限切れ時の動作 |
|---|---|---|
| AWS 一時クレデンシャル | Identity Pool: 1 時間 / OIDC: 最大 12 時間 | Claude Code が自動で再取得（ユーザ操作なし） |
| Cognito ID Token | 1 時間 | リフレッシュトークンで自動更新（ユーザ操作なし） |
| Cognito リフレッシュトークン | 30 日 | ブラウザで再ログインが必要 |

日常の利用では、**30 日に 1 回だけブラウザでログイン**すれば自動更新される。

## 仕組み

```
ユーザ (Claude Code)
  │
  │  .claude/settings.json の awsCredentialExport に bedrock-login を指定
  │
  ├─ bedrock-login が呼ばれる
  │   ├─ Keychain にリフレッシュトークンがあれば ID Token を自動更新
  │   └─ なければブラウザで Cognito Hosted UI にログイン（PKCE 付き）
  │       └─ 認可コードをローカル HTTP サーバで受け取り → ID Token 取得
  │
  ├─ ID Token → AWS 一時クレデンシャルに変換
  │   ├─ Identity Pool 方式: get-id → get-credentials-for-identity
  │   └─ OIDC 方式: sts assume-role-with-web-identity
  │
  └─ 一時クレデンシャルを JSON で stdout に出力 → Claude Code が Bedrock API を呼び出し
```

### awsCredentialExport の出力形式

Claude Code は以下の形式を期待する:

```json
{
  "Credentials": {
    "AccessKeyId": "ASIA...",
    "SecretAccessKey": "...",
    "SessionToken": "..."
  }
}
```

## セキュリティ

| 情報 | 保存場所 | 平文ファイルへの書き出し |
|---|---|---|
| リフレッシュトークン（30 日有効） | macOS Keychain | しない |
| AWS 一時クレデンシャル | メモリのみ（stdout 出力） | しない |
| Cognito Client ID 等 | env.sh | する（秘密情報ではない） |

- パブリッククライアント + PKCE (S256) を採用。Client Secret は不要
- `env.sh` に秘密情報は含まれないため、Git にコミット可能
- ログアウト状態は Keychain のフラグで管理し、Claude Code からの自動再認証をブロック

## トラブルシューティング

### ポート競合エラー (`Address already in use`)

bedrock-login は以下のポートを順番に試行する: `19240`, `19336`, `19427`, `19777`, `19819`

全てが使用中の場合はエラーになる。使用中のプロセスを確認:

```bash
lsof -i :19240
```

### 認証がタイムアウトする

ブラウザでのログインは 2 分以内に完了する必要がある。

### リフレッシュが繰り返し失敗する

Keychain のトークンをクリアして再認証:

```bash
bedrock-login --clear
bedrock-login
```

### Keychain へのアクセス許可ダイアログ

初回実行時に macOS が Keychain へのアクセス許可を求めるダイアログを表示する場合がある。「許可」を選択すること。

### Claude Code で 403 エラーが出る

`awsCredentialExport` の出力形式を確認:

```bash
bedrock-login --verbose
```

出力が `{"Credentials": {"AccessKeyId": "ASIA...", ...}}` の形式であること。

## 注意事項

- このセットアップは PoC 用。本番環境では既存の Cognito User Pool との統合や、IP 制限の考慮が必要
- Cognito ドメインプレフィックスは `amazoncognito.com` のサブドメインを全 AWS ユーザで共有するため、グローバルに一意でなければならない（setup がランダム文字列で自動生成する）
- `awsCredentialExport` で取得したクレデンシャルは Claude Code が Bedrock API を呼び出す際に使われるが、Claude Code 内で実行される `aws` CLI 等の子プロセスには渡されない。`aws` CLI はデフォルトのクレデンシャルチェーン（`~/.aws/` の設定やプロファイル等）を参照する。一方、`.claude/settings.json` の `env` で設定した `AWS_REGION` は子プロセスに継承される。Claude Code 内で `aws` CLI を呼び出す場合は、`--profile` と `--region` を明示的に指定すること
