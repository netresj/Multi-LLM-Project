# 開発環境について

Windows、macOS、Linux（Ubuntu）から共通の Ubuntu 24.04 環境を利用するための Dev Container 設定です。言語やフレームワークは固定していません。

## 前提

- Visual Studio Code
- Docker コンテナを実行できる環境（Windows・macOS では Docker Desktop など）
- VS Code の Dev Containers 拡張機能
- コンテナイメージと拡張機能を取得できるネットワーク接続

Windows では Linux コンテナを利用してください。

## 起動方法

1. Docker を起動します。
2. VS Code でリポジトリのルートを開きます。
3. コマンドパレットから `Dev Containers: Reopen in Container` を実行します。
4. コンテナ作成後、利用する各エージェントでサインインします。

コンテナ内のユーザは `vscode` です。エージェント拡張機能は `devcontainer.json` の `customizations.vscode.extensions` に記載しています。各サービスの契約・認証は別途必要です。この雛形にはエージェントの CLI を追加する処理は含めていません。

作成時に `postCreateCommand` で共有スキルの登録スクリプトを実行します。コマンドの利用方法は [エージェント構成](../.llm-agents/README.md#コマンドで呼び出す) を参照してください。

## トークン削減ツール

コンテナのビルド時に以下をインストールします。

| ツール | 用途 |
| --- | --- |
| RTK（Rust Token Killer） | 対応するコマンドの出力を圧縮し、LLM が読む量を削減 |
| ripgrep（`rg`） | 必要なファイルや行に検索を限定 |
| `jq` | JSON から必要な項目だけを抽出 |
| Python 3・PyYAML | 共有スキルの登録と YAML メタデータの検証 |

RTK は [公式リリース v0.49.0](https://github.com/rtk-ai/rtk/releases/tag/v0.49.0) に固定しています。[公式インストーラ](https://github.com/rtk-ai/rtk/blob/v0.49.0/install.sh) で Linux の x86_64 / ARM64 を判定し、SHA-256 の検証後に `/usr/local/bin/rtk` へ配置します。初回ビルドには Ubuntu のパッケージ配布元と GitHub への接続が必要です。

設定を反映するには `Dev Containers: Rebuild Container` を実行してください。コンテナ内のターミナルで確認できます。

```sh
rtk --version
rtk git status
rtk git diff
rtk git log -5
rtk gain
rg --version
jq --version
```

利用方針は [共通プロンプト](../.llm-agents/instructions.md) に集約しています。Claude Code、Codex、GitHub Copilot はその指示に従って `rtk` を明示的に呼び出します。通常のコマンドを自動変換するフックや `rtk init` による指示ファイルの生成は行っていません。エージェントが Dev Container 内のターミナルを使っていることを確認してください。

圧縮された出力だけで判断できない場合は、元のコマンドで必要な情報を確認してください。`rtk gain` は RTK 経由の実行による推定削減量を表示します。ファイル閲覧など別の経路で渡す情報には適用されず、実際の請求額の削減率を示すものではありません。

更新時は `Dockerfile` の `RTK_VERSION` を変更して再ビルドします。利用例と対応コマンドは [RTK 公式ドキュメント](https://www.rtk-ai.app/docs/getting-started/installation/) を参照してください。

Python 3 と PyYAML は Ubuntu のパッケージとして明示的に導入し、ビルド時に Python の起動と `yaml` のインポートを確認します。コンテナ外での依存導入と回帰テストは [エージェント構成](../.llm-agents/README.md#回帰テスト) を参照してください。

ベースイメージの既定値は更新可能な `ubuntu-24.04` タグです。厳密に固定する場合は、`devcontainer.json` の `build.args.BASE_IMAGE` に確認済みの `mcr.microsoft.com/devcontainers/base:ubuntu-24.04@sha256:<digest>` を指定してください。Dockerfile の `BASE_IMAGE` ビルド引数で切り替えられます。ベースイメージを固定しても、ビルド時に取得する Ubuntu パッケージまで同一になる保証はありません。

## カスタマイズ

- 使用言語が決まったら、`devcontainer.json` に Features を追加するか、既存の `Dockerfile` に必要なツールを追加してください。
- 開発サーバを利用する場合は、必要に応じて `forwardPorts` を追加してください。
- 設定変更後は `Dev Containers: Rebuild Container` を実行してください。
- API キーや認証情報をイメージ・設定ファイル・Git に保存しないでください。

Dev Container を使わず、各 OS 上で直接開発することもできます。その場合は必要な開発ツールを個別に用意してください。
