# 開発環境について

Windows、macOS、Linux（Ubuntu）から共通の Ubuntu 24.04 環境を利用するための Dev Container 設定です。言語やフレームワークは固定していません。

## 前提

- Visual Studio Code
- Docker コンテナを実行できる環境（Windows・macOS では Docker Desktop など）
- Docker Compose v2 以降（`docker compose` が利用可能なこと）
- VS Code の Dev Containers 拡張機能
- コンテナイメージと拡張機能を取得できるネットワーク接続

Windows では Linux コンテナを利用してください。

## 起動方法

1. Docker を起動します。
2. VS Code でリポジトリのルートを開きます。
3. 拡張機能画面で `@recommended` を検索し、Microsoft の **Dev Containers**（`ms-vscode-remote.remote-containers`）をホスト側にインストールします。
4. 左下のリモート接続ボタンから **Reopen in Container** を選ぶか、コマンドパレットから `Dev Containers: Reopen in Container` を実行します。既存環境を更新する場合は `Dev Containers: Rebuild Container` を実行します。
5. 左下に `Dev Container: Multi-LLM-Project` と表示されたら、拡張機能画面のコンテナ側で Claude Code・Codex・GitHub Copilot の導入を確認し、それぞれサインインします。

Compose を手動で起動する必要はありません。拡張機能が `devcontainer.json` を読み、開発用コンテナとプロキシをまとめて起動します。起動に失敗した場合は `Dev Containers: Show Container Log` で確認してください。

`customizations.vscode.settings` にコンテナ内の VS Code 用プロキシ設定を指定しています。ホスト側の VS Code や認証用ブラウザはホストのネットワーク設定を使います。`proxy:3128` は Compose 内の名前なので、ホストのユーザ設定には指定しないでください。企業ネットワークではホストからの拡張機能取得やブラウザ認証にも通信許可が必要です。

コンテナ内のユーザは `vscode` です。エージェント拡張機能は `devcontainer.json` の `customizations.vscode.extensions` に記載しています。各サービスの契約・認証は別途必要です。この雛形にはエージェントの CLI を追加する処理は含めていません。

作成時に `postCreateCommand` で共有スキルの登録スクリプトを実行します。コマンドの利用方法は [エージェント構成](../.llm-agents/README.md#コマンドで呼び出す) を参照してください。

## 認証の引き継ぎ

Codex の `/home/vscode/.codex` と Claude Code の `/home/vscode/.claude` は名前付きボリュームに保存します。Claude Code は `CLAUDE_CONFIG_DIR` で保存先を指定します。コンテナ内で初回ログインすると、ファイルに保存された認証・設定を再構築後も使えます。既存コンテナの書き込み層に認証を保存していた場合は、再構築前に退避するか、再構築後に再ログインしてください。`docker compose down --volumes` は認証用ボリュームも削除します。

### ホストの認証ファイルを共有する

1. `.devcontainer/compose.auth.example.yaml` を同じ場所の `compose.auth.yaml` にコピーします。
2. `volumes: []` を削除し、必要なマウントだけコメントを外します。`source` はホスト上の既存ディレクトリの絶対パスに変更してください。Windows は `C:/Users/名前/.codex` のように指定します。リモート Docker では Docker ホスト上のパスになります。
3. `devcontainer.json` の `dockerComposeFile` を次に変更します。

   ```json
   "dockerComposeFile": ["compose.yaml", "compose.auth.yaml"]
   ```

4. `Dev Containers: Rebuild Container` を実行します。

追加ファイルは自動では読み込みません。ホストから Compose を操作する場合も、`docker compose -f .devcontainer/compose.yaml -f .devcontainer/compose.auth.yaml ...` と両方を指定してください。設定確認は末尾に `config --quiet` を付けます。

| ツール | ホスト認証の利用方法 |
| --- | --- |
| Codex | ホストの `~/.codex`（`CODEX_HOME` を変更している場合はそのディレクトリ）を共有します。`auth.json` が必要です。Keychain／keyring のみの場合は、ホストの `config.toml` で `cli_auth_credentials_store = "file"` を設定してログインし直すか、コンテナ内でログインしてください。 |
| Claude Code | Linux などでファイル保存された `~/.claude/.credentials.json` を含むディレクトリを共有できます。macOS Keychain の認証はディレクトリ共有では移行できないため、コンテナ側の拡張機能でログインするか API キーを利用します。ホストの `~/.claude.json` にある初期設定・信頼状態は自動移行しないため、初回の確認が表示される場合があります。 |
| GitHub Copilot | ホストの VS Code の「アカウント」で GitHub にサインインし、コンテナの Copilot で同じアカウントを選びます。認証を求められたら VS Code のサインイン操作を完了してください。Git の credential helper や SSH 鍵は Copilot のサインインの代わりにはなりません。 |

共有ディレクトリはトークンの更新を可能にするため書き込み可能です。認証だけでなく設定・履歴・ログアウトもホスト側に影響します。同じ認証の同時利用で更新が競合する場合は、共有を外してコンテナ内で個別にログインしてください。コンテナ内の UID がホストの所有者と異なる場合は、書き込み権限の調整か名前付きボリュームでの個別ログインが必要です。

仕様は [Codex の認証](https://developers.openai.com/codex/auth/)、[Claude Code の認証](https://code.claude.com/docs/en/authentication)、[Claude Code の設定](https://code.claude.com/docs/en/settings)、[Copilot のセットアップ](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-extension) を参照してください。

### API キーを使う場合

`auth.env.example` を `auth.env` にコピーして利用するキーだけ設定し、`compose.auth.yaml` の `env_file` を有効にします。上記の Compose ファイル追加と再構築も必要です。`auth.env` と `compose.auth.yaml` は Git 管理対象外です。API 利用はサブスクリプション認証とは別の課金・権限になります。

Claude Code は `ANTHROPIC_API_KEY` を利用できます。Codex CLI を別途インストールしている場合は、コンテナ内で次を実行します。環境変数を渡すだけでは、Codex 拡張機能のサインインが完了するとは限りません。

```sh
printenv OPENAI_API_KEY | codex login --with-api-key
```

キーはコンテナの環境変数になります。値を表示する `docker inspect` や `docker compose config` の出力を共有しないでください。構成確認には `config --quiet` を使います。

### Git: SSH agent を利用する（推奨）

Dev Containers は起動中のホストの SSH agent を自動転送します。ホストで鍵を登録してからコンテナを開き直してください。秘密鍵ファイルのマウントは不要です。

```sh
# ホストで実行する。鍵のパスは利用中のものに変更する。
ssh-add ~/.ssh/id_ed25519
ssh-add -l
```

Linux で agent が未起動の場合は `eval "$(ssh-agent -s)"` の後で登録し、その環境から VS Code を起動します。Windows は管理者 PowerShell で `Set-Service ssh-agent -StartupType Automatic`、`Start-Service ssh-agent` を実行し、通常の PowerShell で `ssh-add "$env:USERPROFILE/.ssh/id_ed25519"` を実行します。

コンテナ内の `ssh-add -l` で鍵の転送、`git ls-remote origin HEAD` で現在の remote へのアクセスを確認できます。初回の SSH 接続では、接続先が公開するホスト鍵のフィンガープリントと照合してください。Compose 単独起動では VS Code による agent・credential helper の転送はありません。

### Git: 秘密鍵ファイルを利用する

agent を使えない場合は `compose.auth.yaml` の鍵マウント例を有効にします。鍵をリポジトリには置かず、ホスト上の鍵を読み取り専用で `/mnt/git-ssh-key` にマウントします。再構築後、コンテナ内で次を実行します。

```sh
mkdir -p ~/.ssh
chmod 700 ~/.ssh
# 上書きを避けるため、この名前の鍵が存在しないことを確認する。
test ! -e ~/.ssh/devcontainer_git_key && install -m 600 /mnt/git-ssh-key ~/.ssh/devcontainer_git_key
export GIT_SSH_COMMAND='ssh -i /home/vscode/.ssh/devcontainer_git_key -o IdentitiesOnly=yes'
git ls-remote origin HEAD
```

コピー先の名前は既存の鍵と重複させないでください。この環境変数は現在のシェルだけに適用されます。継続利用する場合はコンテナ内の `~/.bashrc` に `export` 行を追加します。鍵のコピーとシェル設定は再構築後に再実行します。パスフレーズ付きの鍵は入力を求められます。ホストの SSH 設定・踏み台・証明書は自動移行しないので、必要なものを個別に設定してください。

### Git: HTTPS 認証に切り替える

ホストの Git で OS の credential helper を設定し、HTTPS で認証しておくと、Dev Containers がその認証をコンテナから利用できるようにします。SSH 用の remote を変更する場合は、コンテナ内で次を実行してください。

```sh
git remote set-url origin https://github.com/OWNER/REPO.git
git ls-remote origin HEAD
```

`OWNER/REPO` は対象リポジトリに置き換えます。remote の変更は共有ワークスペースのホスト側にも反映されます。URL にトークンを埋め込まないでください。HTTPS は設定済みの HTTP プロキシを利用します。SSH は `HTTP_PROXY` を自動利用しないため、SSH が禁止されたネットワークでは HTTPS を選びます。GitHub Enterprise などの独自ホストはプロキシの許可リストにも追加してください。

Git の転送機能の詳細は [VS Code の認証情報共有](https://code.visualstudio.com/remote/advancedcontainers/sharing-git-credentials) を参照してください。

## 外部アクセス用プロキシ

`compose.yaml` で開発用の `devcontainer` と Squid の `proxy` を一緒に起動します。Squid のヘルスチェックが成功すると開発用コンテナを起動し、VS Code から環境を閉じると両サービスを停止します。

開発用コンテナには `HTTP_PROXY` / `HTTPS_PROXY` と小文字の同名変数を `http://proxy:3128` に設定しています。これらを参照する curl、Git などの HTTP／HTTPS 通信が Squid を経由します。HTTPS は CONNECT で中継し、TLS の復号や独自 CA の配布は行いません。

`NO_PROXY` / `no_proxy` には `localhost,127.0.0.1,::1,proxy,devcontainer` を設定しています。別のサービスを追加して直接接続したい場合は、両方の変数にサービス名を追加してください。

プロキシのポートはホストに公開していません。同じ Compose ネットワーク内では認証なしで利用でき、接続先ポートは HTTP の 80 と HTTPS の 443 に限定しています。変更が必要な場合は `proxy/squid.conf` の ACL を編集して再ビルドしてください。キャッシュは無効で、アクセスログはコンテナの標準出力に記録します。

これは環境変数を利用する明示的なプロキシ設定です。直接通信をネットワークで禁止する構成ではありません。環境変数を参照しない拡張機能やツール、SSH などには個別の設定が必要です。また、イメージ取得・Dockerfile のビルド中の通信はこのプロキシを経由しません。それらにもプロキシが必要なネットワークでは、Docker 側のプロキシ設定を別途用意してください。

設定後は `Dev Containers: Rebuild Container` を実行します。開発用コンテナ内で次を実行すると、プロキシの利用と HTTPS CONNECT を確認できます。

```sh
curl -v --fail --head http://registry.npmjs.org
curl -v --fail --head https://pypi.org/simple/pip/
```

ホストのリポジトリルートから状態とアクセスログを確認できます。`compose.yaml` の `name` で Compose のプロジェクト名を `multi-llm-project_devcontainer` に固定しているため、Dev Containers 拡張が起動したコンテナも次のコマンドで操作できます。実際のプロジェクト名は `docker compose ls` で確認できます。

```sh
docker compose -f .devcontainer/compose.yaml ps
docker compose -f .devcontainer/compose.yaml logs --tail=50 proxy
```

構成の仕様は [VS Code の Dev Container 作成ガイド](https://code.visualstudio.com/docs/devcontainers/create-dev-container) と [Squid のアクセス制御](https://www.squid-cache.org/Doc/config/http_access/) を参照してください。

### ドメインのホワイトリスト・ブラックリスト

- `proxy/lists/whitelist.txt`: 許可するドメイン。有効な行がある場合、そのドメインだけを許可します。空の場合はブラックリスト以外を許可します。
- `proxy/lists/blacklist.txt`: 拒否するドメイン。両方に一致する場合は拒否を優先します。

初期状態では Python（uv / PyPI）、Node.js（npm）、Claude Code・Codex・GitHub Copilot・GitHub・VS Code のホストを許可しています。ブラックリストはコメントのみです。**許可リストにない宛先へのプロキシ通信は拒否されます。**

| 用途 | 許可する主なホスト |
| --- | --- |
| Python パッケージの取得 | `pypi.org`、`files.pythonhosted.org` |
| uv のインストーラと配布先 | `astral.sh`、`releases.astral.sh` |
| GitHub、uv・CPython 配布物、GitHub 依存、nvm | `.github.com`、`.githubusercontent.com`、`.githubassets.com`、`github-cloud.s3.amazonaws.com` |
| Node.js 本体・ヘッダ、npm パッケージ | `nodejs.org`、`registry.npmjs.org` |
| Claude Code | `api.anthropic.com`、`claude.ai`、`platform.claude.com`、`downloads.claude.ai` など |
| Codex | `.chatgpt.com`、`api.openai.com`、`.auth.openai.com`、配布・コンテンツ用 CDN など |
| GitHub Copilot | `.githubcopilot.com`、GitHub 共通ホスト、`default.exp-tas.com` |
| VS Code Server・拡張機能 | `marketplace.visualstudio.com`、`.gallery.vsassets.io`、`.gallerycdn.vsassets.io`、VS Code 配布用 CDN など |

配布元の根拠は [uv のインストール](https://docs.astral.sh/uv/getting-started/installation/)、[uv の Python 配布物](https://docs.astral.sh/uv/concepts/python-versions/)、[PyPI の Index API](https://docs.pypi.org/api/index-api/)、[npm のレジストリ](https://docs.npmjs.com/cli/v11/using-npm/registry/) を参照してください。この変更は通信先の許可のみで、uv や Node.js 自体はインストールしません。

PyPy、Yarn、起動後の Ubuntu パッケージ取得用の候補は、許可リスト内にコメントで用意しています。社内レジストリや npm のインストールスクリプトが取得する追加バイナリ（Playwright、Cypress、Electron など）は、利用するものに応じて追加してください。GitHub Enterprise の独自ドメイン、企業 SSO の IdP、Bedrock・Vertex AI・Azure などの外部モデルプロバイダは別途追加してください。エージェントが Web 検索や外部サイト閲覧でアクセスする宛先も、利用先に応じた追加が必要です。接続が拒否された場合は `docker compose -f .devcontainer/compose.yaml logs --tail=100 proxy` の `TCP_DENIED/403` から宛先を確認できます。

接続先の確認には [Claude Code のネットワーク要件](https://code.claude.com/docs/en/network-config)、[Codex の認証](https://developers.openai.com/codex/auth/)、[Copilot の許可リスト](https://docs.github.com/en/copilot/reference/copilot-allowlist-reference)、[VS Code のネットワーク要件](https://code.visualstudio.com/docs/setup/network) を利用しています。Codex の一覧は通常の ChatGPT / API キー認証向けの基本構成で、全機能・外部連携を網羅するものではありません。サービス側の配布先変更時はログから追記してください。

GitHub の Git 操作をプロキシ経由にする場合は `https://github.com/OWNER/REPO.git` 形式を使ってください。SSH 形式の `git@github.com:OWNER/REPO.git` はこの HTTP プロキシを自動では利用しません。GitHub Packages の npm ホストは `.github.com` に含まれます。Copilot は全契約プラン向けのドメインを許可しています。Claude のプラグイン情報・旧インストーラ向けに `storage.googleapis.com` も許可するため、このホスト上の他のバケットにも通信できます。

GitHub の許可は特定のリポジトリだけに限定するものではありません。また、レジストリを許可しても個々のパッケージの安全性を判定するものではありません。

1行に1ドメインを記載します。`example.com` はそのホストだけ、`.example.com` はドメイン自身とサブドメインに一致します。URL、パス、ポート、`*.example.com` のようなワイルドカードは記載しません。空行、`#` 以降のコメント、Windows の改行に対応しています。

例えばホワイトリストに `.example.com`、ブラックリストに `blocked.example.com` を記載すると、`example.com` とそのサブドメインのうち `blocked.example.com` 以外を許可します。判定は HTTP の宛先ホスト名と HTTPS CONNECT の宛先ホスト名に適用します。IP アドレスからの逆引きは行いません。ヘルスチェックはリストの制限対象外です。ドメインの記法は [Squid の ACL 仕様](https://www.squid-cache.org/Doc/config/acl/) に従います。

リストのディレクトリは読み取り専用でマウントしています。初回は `Dev Containers: Rebuild Container` を実行してください。その後のリスト変更は、ホストのリポジトリルートから次を実行して反映します（イメージの再ビルドは不要）。再起動時には既存のプロキシ接続が切断されます。

```sh
docker compose -f .devcontainer/compose.yaml restart proxy
docker compose -f .devcontainer/compose.yaml ps
```

起動時にリストを読み込んで ACL を生成するため、`squid -k reconfigure` だけでは変更を反映できません。ファイルの削除や読み取り権限の不足がある場合は起動に失敗します。この制限が適用されるのは Squid を経由する通信です。

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

ベースイメージの既定値は更新可能な `ubuntu-24.04` タグです。厳密に固定する場合は、`compose.yaml` の `services.devcontainer.build.args.BASE_IMAGE` に確認済みの `mcr.microsoft.com/devcontainers/base:ubuntu-24.04@sha256:<digest>` を指定してください。Dockerfile の `BASE_IMAGE` ビルド引数で切り替えられます。プロキシのベースイメージは `proxy/Dockerfile` の `ubuntu:24.04` です。ベースイメージを固定しても、ビルド時に取得する Ubuntu パッケージまで同一になる保証はありません。

## カスタマイズ

- 使用言語が決まったら、`devcontainer.json` に Features を追加するか、既存の `Dockerfile` に必要なツールを追加してください。
- 開発サーバを利用する場合は、必要に応じて `forwardPorts` を追加してください。
- 設定変更後は `Dev Containers: Rebuild Container` を実行してください。
- API キーや認証情報をイメージ・設定ファイル・Git に保存しないでください。

Dev Container を使わず、各 OS 上で直接開発することもできます。その場合は必要な開発ツールを個別に用意してください。
