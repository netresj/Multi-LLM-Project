# Multi-LLM-Project

## このリポジトリについて

このリポジトリは複数のLLMをVSCodeのコーディングエージェントで利用するためのテンプレートリポジトリです.
対象とするLLMサービスは Claude Code, Codex, および Github Copilot です.
Windows, Mac OS, Linux(Ubuntu) での開発を対象とします.

## はじめ方

1. このリポジトリをテンプレートとして利用するか、クローンします。
2. VS Code でリポジトリのルートを開き、必要な推奨拡張機能をインストールします。
3. 利用する Claude Code、Codex、GitHub Copilot のアカウントでサインインします。
4. [共通ルール](AGENTS.md) を確認し、[共通プロンプト](.llm-agents/instructions.md) の未設定項目をプロジェクトに合わせて記入します。
5. 共通のコンテナ環境を使う場合は、[Dev Container の手順](.devcontainer/README.md) に従います。

## ディレクトリ構成

```text
.
├── AGENTS.md                       # 共通ルールの正本・Codex の入口
├── CLAUDE.md                       # Claude Code の入口
├── .github/
│   └── copilot-instructions.md      # GitHub Copilot の入口
├── .llm-agents/
│   ├── README.md                   # エージェント構成の説明
│   ├── instructions.md             # 共通プロンプトの本体
│   ├── skills/                     # スキルの本体の配置先
│   └── agents/                     # エージェント定義の本体の配置先
├── .devcontainer/
│   ├── README.md                   # 開発環境の説明
│   ├── devcontainer.json           # Ubuntu ベースの共通環境
│   └── Dockerfile                  # RTK・検索補助ツールの導入
└── .vscode/
    ├── extensions.json             # 推奨拡張機能
    └── settings.json               # ワークスペース設定
```

共通プロンプトは1ファイルに集約し、各エージェントの入口から参照します。アプリケーションの言語やディレクトリ構成は、利用するプロジェクトに合わせて追加してください。

共有スキルを追加する場合は、いずれのエージェントでも「このリポジトリの `create-shared-skill` スキルを使って、○○用のスキルを作成してください」と依頼できます。作成したスキルは共通ディレクトリに配置され、他のエージェントでも同じ本体を読み込んで利用できます。

エージェントについての詳細は [.llm-agents/README.md](.llm-agents/README.md) を参照してください。
開発環境についての詳細は [.devcontainer/README.md](.devcontainer/README.md) を参照してください。

共有スキル作成コマンドは Claude Code・Copilot では `/create-shared-skill`、Codex では `$create-shared-skill` です。[登録と呼び出しの手順](.llm-agents/README.md#コマンドで呼び出す) を参照してください。
