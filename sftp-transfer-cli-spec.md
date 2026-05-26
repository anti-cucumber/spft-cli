# SFTP転送CLI 仕様書

## 1. 概要

本書は、SFTPによるファイル転送をラップする自作CLI部品の仕様を定義する。

本CLIはGo言語で実装し、以下の環境で動作することを想定する。

- Azure上のWindows Server
- AWS上のLinux

本CLIは以下の操作を提供する。

- `GET`: SFTPサーバーから1ファイルを取得する。
- `PUT`: SFTPサーバーへ1ファイルを格納する。
- `LIST`: 指定したリモートパス直下の一覧情報を取得する。

SFTP接続情報は、実行時に各クラウドのSecret管理サービスから取得する。

- AWS: AWS Secrets Manager
- Azure: Azure Key Vault

クラウド側の認証情報はCLI引数では渡さず、CLIを実行する筐体に付与されたロールまたはManaged Identityに任せる。

テスト用途に限り、ローカルJSONファイルからSFTP接続情報を取得する `local` プロバイダーを提供する。`local` は実運用用途では使用しない。

## 2. コマンド形式

```bash
sftp-util <操作> [オプション]
```

`<操作>` には以下のいずれかを指定する。

- `GET`
- `PUT`
- `LIST`

操作名は大文字・小文字を区別しない。ただし、利用時の表記は大文字を推奨する。

## 3. 共通オプション

| オプション | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `--provider` | 必須 | - | 接続情報プロバイダー。指定可能値は `aws`、`azure`、`local`。 |
| `--secret-name` | 条件付き必須 | - | SFTP接続情報を格納したSecret名。`aws` / `azure` で必須。 |
| `--secret-file` | 条件付き必須 | - | SFTP接続情報を格納したローカルJSONファイルパス。`local` で必須。 |
| `--remote-path` | 必須 | - | リモートファイルパスまたはディレクトリプレフィックス。SFTPログインユーザーのホームディレクトリからの相対パスとして扱う。 |
| `--connect-timeout` | 任意 | `30s` | SFTP接続確立までのタイムアウト。 |
| `--timeout` | 任意 | `10m` | 1回の操作試行全体のタイムアウト。 |
| `--retry` | 任意 | `2` | 初回失敗後のリトライ回数。 |
| `--retry-interval` | 任意 | `5s` | リトライ前の待機時間。 |

## 4. プロバイダー別オプション

### 4.1 AWS

| オプション | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `--region` | 任意 | 実行環境から解決 | AWS Secrets Managerへアクセスするリージョン。 |
| `--aws-endpoint-url` | 任意 | - | AWS Secrets ManagerのエンドポイントURL。LocalStackなどのローカル検証で利用する。通常運用では指定しない。 |

AWSの認証は、AWS SDKの標準認証チェーンを利用する。

EC2上で実行する場合は、EC2インスタンスに付与されたIAMロールを利用する想定とする。

IAMロールには、対象Secretを読み取るために以下の権限が必要となる。

- `secretsmanager:GetSecretValue`
- SecretがカスタマーマネージドKMSキーで暗号化されている場合は `kms:Decrypt`

### 4.2 Azure

| オプション | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `--vault-name` | 必須 | - | Azure Key Vault名。CLI内部でVault URLを組み立てる。 |

Azureの認証は `DefaultAzureCredential` を利用する。

Azure VM上で実行する場合は、VMに割り当てられたManaged Identityを利用する想定とする。

Managed Identityには、対象Key Vault Secretを読み取るための権限が必要となる。

### 4.3 local

`local` はテスト用途専用のプロバイダーとする。

| オプション | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- |
| `--secret-file` | 必須 | - | SFTP接続情報を格納したローカルJSONファイルパス。 |

`--secret-file` には、Secret JSON形式と同じJSONオブジェクトを記載する。

`local` はクラウドSecret管理サービスへアクセスしない。

実行例:

```bash
sftp-util LIST --provider local --secret-file ./local-sftp-secret.json --remote-path data/
```

## 5. 操作別仕様

### 5.1 GET

必須オプション:

| オプション | 説明 |
| --- | --- |
| `--remote-path` | 取得元のリモートファイルパス。 |
| `--local-path` | 格納先のローカルファイルパス。 |

実行例:

```bash
sftp-util GET --provider aws --secret-name my-sftp-secret --remote-path data/input.txt --local-path ./input.txt
```

Azureでの実行例:

```bash
sftp-util GET --provider azure --vault-name my-vault --secret-name my-sftp-secret --remote-path data/input.txt --local-path ./input.txt
```

localでの実行例:

```bash
sftp-util GET --provider local --secret-file ./local-sftp-secret.json --remote-path data/input.txt --local-path ./input.txt
```

動作:

- SFTPサーバーから1ファイルを取得する。
- ローカルファイルが既に存在する場合は上書きする。
- `--local-path` の親ディレクトリは自動作成しない。
- ローカル側の親ディレクトリが存在しない場合はエラーとする。

### 5.2 PUT

必須オプション:

| オプション | 説明 |
| --- | --- |
| `--remote-path` | 格納先のリモートファイルパス。 |
| `--local-path` | 格納元のローカルファイルパス。 |

実行例:

```bash
sftp-util PUT --provider aws --secret-name my-sftp-secret --remote-path data/output.txt --local-path ./output.txt
```

Azureでの実行例:

```bash
sftp-util PUT --provider azure --vault-name my-vault --secret-name my-sftp-secret --remote-path data/output.txt --local-path ./output.txt
```

localでの実行例:

```bash
sftp-util PUT --provider local --secret-file ./local-sftp-secret.json --remote-path data/output.txt --local-path ./output.txt
```

動作:

- SFTPサーバーへ1ファイルを格納する。
- リモートファイルが既に存在する場合は上書きする。
- `--remote-path` の親ディレクトリは自動作成しない。
- リモート側の親ディレクトリが存在しない場合はエラーとする。
- 格納後、ローカルファイルサイズとリモートファイルサイズを比較する。
- サイズチェックに失敗した場合は、格納したリモートファイルを削除し、エラーとして終了する。

### 5.3 LIST

必須オプション:

| オプション | 説明 |
| --- | --- |
| `--remote-path` | 一覧取得対象のリモートディレクトリプレフィックス。 |

実行例:

```bash
sftp-util LIST --provider aws --secret-name my-sftp-secret --remote-path data/
```

Azureでの実行例:

```bash
sftp-util LIST --provider azure --vault-name my-vault --secret-name my-sftp-secret --remote-path data/
```

localでの実行例:

```bash
sftp-util LIST --provider local --secret-file ./local-sftp-secret.json --remote-path data/
```

動作:

- 指定したリモートパス直下のエントリを一覧取得する。
- 初期実装では再帰的な一覧取得は行わない。
- 出力形式はJSON Linesとする。

出力項目:

| 項目 | 型 | 説明 |
| --- | --- | --- |
| `name` | string | エントリ名。 |
| `path` | string | SFTPログインユーザーのホームディレクトリからの相対パス。 |
| `size` | number | エントリサイズ。単位はバイト。 |
| `modTime` | string | 最終更新日時。RFC 3339形式。 |
| `isDir` | boolean | ディレクトリの場合は `true`。 |

出力例:

```jsonl
{"name":"input.txt","path":"data/input.txt","size":12345,"modTime":"2026-05-26T10:00:00Z","isDir":false}
{"name":"archive","path":"data/archive","size":0,"modTime":"2026-05-26T10:05:00Z","isDir":true}
```

## 6. Secret JSON形式

Secretの値はJSONオブジェクトとする。

```json
{
  "host": "sftp.example.com",
  "port": 22,
  "username": "user01",
  "privateKey": "-----BEGIN OPENSSH PRIVATE KEY-----\n...",
  "privateKeyPassphrase": "optional-passphrase",
  "hostKeySHA256": "SHA256:xxxxxxxx..."
}
```

項目:

| 項目 | 必須 | 説明 |
| --- | --- | --- |
| `host` | 必須 | SFTPサーバーのホスト名またはIPアドレス。 |
| `port` | 必須 | SFTPサーバーのポート番号。通常は `22`。 |
| `username` | 必須 | SFTPログインユーザー名。 |
| `privateKey` | 必須 | 公開鍵認証で利用する秘密鍵の内容。 |
| `privateKeyPassphrase` | 任意 | 秘密鍵のパスフレーズ。省略時はパスフレーズなしの秘密鍵として解析する。 |
| `hostKeySHA256` | 必須 | 期待するSFTPサーバーホスト鍵のSHA256フィンガープリント。 |

パスワード認証はサポートしない。

## 7. リモートパスの扱い

リモートパスは、SFTPログインユーザーに紐づくホームディレクトリからの相対パスとして扱う。

呼び出し元はホームディレクトリを指定しない。

例:

| 指定値 | 意味 |
| --- | --- |
| `data/input.txt` | ログインユーザーのホームディレクトリ配下の `data/input.txt`。 |
| `data/` | ログインユーザーのホームディレクトリ配下の `data` ディレクトリ。 |

初期実装では、絶対パス指定は対象外とする。

リモートパスの区切り文字は `/` とする。Windows上で実行する場合も、リモートパスに `\` は使用しない。

親ディレクトリ参照である `..` を含むリモートパスは指定不可とする。

## 8. ホスト鍵検証

SFTPサーバーのホスト鍵検証は必須とする。

CLIは、接続先サーバーのホスト鍵をSecret JSON内の `hostKeySHA256` と照合する。

以下の場合、CLIはエラーとして終了する。

- `hostKeySHA256` がSecret JSONに存在しない。
- 接続先サーバーのホスト鍵フィンガープリントが期待値と一致しない。
- 接続先サーバーのホスト鍵を取得または検証できない。

ホスト鍵検証をスキップするオプションは提供しない。

## 9. リトライとタイムアウト

CLIは、後始末を伴う制御された失敗を可能にするため、リトライとタイムアウトのオプションを持つ。

デフォルト値:

| オプション | デフォルト |
| --- | --- |
| `--connect-timeout` | `30s` |
| `--timeout` | `10m` |
| `--retry` | `2` |
| `--retry-interval` | `5s` |

リトライ方針:

- `GET`: リトライ対象とする。
- `PUT`: リトライ対象とする。失敗した試行で不完全なリモートファイルが残った場合、CLIはそれを削除してからリトライする。
- `LIST`: リトライ対象とする。
- Secret取得は、一時的なエラーのみリトライ対象とする。
- 権限エラー、Secret不存在、Secret JSON不正、SFTP認証失敗、ホスト鍵検証失敗はリトライしない。

呼び出し元でジョブ全体のタイムアウトを別途設定してもよい。ただし、不完全ファイル削除などの後始末をCLI側で行わせたい場合、CLIのタイムアウトは呼び出し元の外部タイムアウトより短く設定することが望ましい。

## 10. 上書き方針

上書きは常に有効とする。

- `GET` は既存のローカルファイルを上書きする。
- `PUT` は既存のリモートファイルを上書きする。

初期実装では、上書きを禁止する `--no-overwrite` オプションは提供しない。

## 11. ディレクトリ作成方針

CLIはディレクトリを自動作成しない。

- `GET` は、ローカル側の親ディレクトリが存在しない場合にエラーとする。
- `PUT` は、リモート側の親ディレクトリが存在しない場合にエラーとする。

## 12. 終了ステータス

| 終了コード | 意味 |
| --- | --- |
| `0` | 正常終了。 |
| `1` | 一般的なエラー。 |
| `2` | コマンドライン引数不正。 |

詳細なエラー情報は標準エラーに出力する。

`LIST` のデータ出力は標準出力に出力する。

## 13. 将来拡張

以下の機能は初期実装では対象外とし、必要に応じて将来追加する。

- 再帰的な `LIST`
- 再帰的な一覧取得用の `--max-depth`
- 一覧取得時の安全装置としての `--max-entries`
- パスワード認証
- 絶対リモートパス
- ディレクトリ自動作成
- 上書き禁止モード

## 14. ビルドおよび配布方針

本CLIは、GitHub Actions上のUbuntu Runnerでビルドすることを想定する。

ビルド時の前提:

- Goのバージョンは、利用するAWS SDK for Go v2およびAzure SDK for Goがサポートするバージョンを利用する。
- Goのバージョンは `go.mod` およびGitHub Actionsワークフロー内で明示する。
- `ubuntu-latest` は将来指すUbuntuバージョンが変わる可能性があるため、安定運用では `ubuntu-24.04` など具体的なRunnerイメージの指定を推奨する。
- Linux向け実行体とWindows向け実行体を同一ソースからクロスコンパイルする。
- OS側のCライブラリ依存を避けるため、可能な限り `CGO_ENABLED=0` でビルドする。
- x86_64向けLinux実行体は、古めのx86_64 CPUでも動作しやすいように `GOAMD64=v1` 相当でビルドする。

想定する成果物:

| 対象OS | 対象アーキテクチャ | 成果物例 |
| --- | --- | --- |
| Linux | x86_64 | `sftp-util-linux-amd64` |
| Windows | x86_64 | `sftp-util-windows-amd64.exe` |

必要に応じて、ARM64環境向けに以下の成果物を追加する。

| 対象OS | 対象アーキテクチャ | 成果物例 |
| --- | --- | --- |
| Linux | arm64 | `sftp-util-linux-arm64` |
| Windows | arm64 | `sftp-util-windows-arm64.exe` |

ビルド時にSecret値やクラウド認証情報は埋め込まない。

SFTP接続情報およびクラウド側の認証情報は、実行時にSecret管理サービスおよび実行筐体のロールまたはManaged Identityから取得する。

実行環境の前提:

- AWS Secrets ManagerまたはAzure Key VaultへHTTPS通信できること。
- 実行OSの信頼済みCA証明書ストアが利用可能であること。
- 実行筐体の時刻が正しく同期されていること。
- 実行筐体のロールまたはManaged Identityに、対象Secretを読み取る権限が付与されていること。
