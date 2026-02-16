# Requirements: core

## 概要

# cc-sdd プロジェクト初期化プロファイル

## 1. プロジェクト概要 (/kiro:spec-init 用)

以下のコマンドでプロジェクトを初期化してください:

> GitHub Models APIを活用し、PRレビューコメントに対する自動修正を行うChrome拡張機能を開発する。

## 2. 要件定義ドラフト (requirements.md 用)

以下の要件定義は **EARS (Easy Approach to Requirements Syntax)** に基づいています。

### 機能要件

* システムは **GitHub Personal Access Token (PAT)** を使用して認証を **shall** 行う。
* システムは GitHub Models API (gpt-4o-mini/gpt-4o) を使用してコード修正提案を **shall** 生成する。
* システムは **Manifest V3 Service Worker** を介してCORS制約を回避し通信を **shall** 行う。
* システムは PRページ上のDOM変更を検知し、修正ボタンを動的に **shall** 注入する。
* システムは修正提案を GitHub の Suggested Changes 形式でコメント返信として **shall** 投稿する。

### 技術的制約

* システムは **TypeScript** および **React** を使用して **must** 構築される。
* ビルドツールとして **Vite** を **must** 使用する。
* すべてのコマンド実行は **Devcontainer** 環境内で **must** 実行されなければならない。
* ホスト環境での直接実行は厳禁とする。

## 3. 環境構築 (Devcontainer)

以下の設定で `.devcontainer/devcontainer.json` を作成してください:

```json
{
  "name": "github-pr-autofix-chrome-extension",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:20",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-azuretools.vscode-docker",
        "esbenp.prettier-vscode",
        "dbaeumer.vscode-eslint"
      ]
    }
  },
  "remoteUser": "node"
}
```

## 4. テスト戦略

* テストは **Vitest** で実行する。
* コマンド: `npm test` (または `vitest run`)
* 実装タスク完了時にテストを自動実行する。


## ユーザーストーリー

- **役割** <ユーザー>
- **やりたいこと** <アクション>
- **理由・メリット** <目的/価値>

## 受入条件 (EARS)

- **前提** <前提条件>
- **もし** <トリガー/操作>
- **ならば** <期待される結果> 
