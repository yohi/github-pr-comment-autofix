# サーバーレス・アーキテクチャによるGitHub PR自動修正Chrome拡張機能の包括的設計と実装戦略

## 1. エグゼクティブサマリー

### 1.1 プロジェクトの概要と目的

現代のソフトウェアエンジニアリングにおいて、Pull Request（PR）ベースの開発ワークフローは品質保証の要石となっている。しかし、コードレビューにおける指摘事項の修正作業、特に「命名規則の微調整」「型定義の修正」「可読性の向上」といった定型的なタスクは、開発者の認知的負荷を高め、コンテキストスイッチによる生産性低下を招く主要因となっている。本レポートは、GitHub Models APIの無料枠を活用し、これらの修正作業をブラウザ上で完結させるサーバーレスChrome拡張機能の設計仕様を包括的に論じるものである。

本設計の核心は、**「Thick Client（シッククライアント）」アーキテクチャ**の採用にある。従来の自動修正ツールがGitHub Actionsや外部のSaaS（Software as a Service）バックエンドを必要としていたのに対し、本設計ではGoogle ChromeのManifest V3環境とGitHubのAPIエコシステムのみを利用し、完全なサーバーレス構成を実現する。これにより、インフラコストをゼロに抑えつつ、個人開発者や小規模チームが即座に導入可能なソリューションを提供する。

### 1.2 技術的アプローチと主要な決定事項

本システムの実現には、以下の3つの技術的柱が存在する。

1. **GitHub Models API (Inference) の活用**: Azure AI基盤上でホストされる gpt-4o や gpt-4o-mini などの高性能モデルへ、GitHub Personal Access Token (PAT) を介して直接アクセスする。これにより、OpenAI等の外部APIキーを別途契約することなく、GitHubのエコシステム内で認証と推論を完結させる。
2. **Manifest V3 Service WorkerによるCORS回避**: Webブラウザのセキュリティモデルにおける最大の障壁であるCross-Origin Resource Sharing (CORS) 制約を打破するため、拡張機能のService Workerをプロキシとして機能させる。これにより、コンテンツスクリプト（ページ内JavaScript）からの直接通信ではブロックされるAPIリクエストを、特権的なバックグラウンドコンテキストから実行する。
3. **「Suggested Changes」機能へのネイティブ統合**: 生成された修正コードを単なるテキストとして提示するのではなく、GitHubのMarkdown拡張構文である suggestion コードブロックとしてフォーマットし、レビューコメントへの返信として投稿する。これにより、レビュワーやPR作成者はワンクリックで修正をコミットでき、Git操作の手間を大幅に削減する。

### 1.3 期待される効果と制約

本システムの導入により、レビュー指摘から修正完了までのリードタイムは数分から数秒へと短縮されることが期待される。一方で、GitHub Modelsの無料枠（Copilot Free等）には厳格なレート制限（RPM/TPM）が存在するため、設計段階でのスロットリング制御やエラーハンドリングが不可欠となる。本レポートでは、これらの制約を技術的に克服するための具体的な実装戦略についても詳述する。

---

## 2. 序論と技術的背景

### 2.1 コードレビュー自動化の進化と現状

コードレビュー自動化の歴史は、静的解析ツール（Linter）のCIへの統合から始まった。その後、Danger JSのようなルールベースのボットが登場し、近年ではLLM（Large Language Models）を用いた自律型エージェント（GitHub Copilot WorkspaceやOpenHandsなど）が注目を集めている。しかし、これらの高度なエージェントは依然として複雑なセットアップや高額なランニングコストを伴う場合が多い。

対照的に、本設計が目指すのは「軽量」かつ「オンデマンド」な自動化である。開発者がブラウザでPRを見ているその瞬間に、特定のコメントに対してピンポイントでAIの支援を受けられるUX（ユーザーエクスペリエンス）は、既存の重量級エージェントとは異なるニッチを埋めるものである。特に、GitHub Models APIの登場により、推論能力が「ユーティリティ」として GitHub プラットフォーム自体に組み込まれたことは、この軽量アプローチを現実的なものとした最大の転換点である。

### 2.2 サーバーレス・ブラウザ拡張機能の台頭

Chrome拡張機能の開発パラダイムは、Manifest V3の導入により大きく変化した。バックグラウンドページの廃止とService Workerへの移行は、常駐プロセスのメモリ消費を削減する一方で、長時間のタスク実行や永続的な状態保持を困難にした。しかし、API通信においては declarativeNetRequest や強化された fetch APIの権限管理により、よりセキュアかつ制御された通信が可能となっている。

本設計において「サーバーレス」とは、AWS LambdaやGoogle Cloud FunctionsのようなFaaS（Function as a Service）を使用することすら排除し、クライアント（ブラウザ）とAPIプロバイダ（GitHub）の1対1通信のみでシステムを成立させることを意味する。これは、ユーザーデータのプライバシー保護（中間サーバーにコードが送信されない）の観点からも極めて重要である。

### 2.3 Copilotエコシステムと「無料枠」の定義

本レポートで言及する「Copilotの無料枠」および「GitHub Modelsの無料枠」については、明確な定義が必要である。GitHubは個人開発者向けに GitHub Copilot Free プランを提供しており、これには月間2000回のコード補完や50回程度のチャットリクエストが含まれる場合がある。また、GitHub Models（Marketplace）のPlaygroundやAPI利用においても、Rate Limit Tierに基づいた無料利用枠が設定されている。

重要な洞察として、これらの無料枠は「実験やプロトタイピング」を主眼に置いているが、APIのエンドポイント自体は本番環境レベルの堅牢性を持っている点が挙げられる。したがって、適切なレート制限管理を行えば、個人の日常業務における補助ツールとして十分に機能する。

---

## 3. GitHub Models API の詳細分析と選定

### 3.1 推論エンドポイントの仕様と統合戦略

GitHub Models APIは、OpenAIなどのモデルプロバイダへの統一インターフェースを提供する。開発者は、モデルごとの固有API仕様を意識することなく、標準的な Chat Completions API（OpenAI互換）を利用できる。

| 項目 | 詳細仕様 | 備考 |
| :--- | :--- | :--- |
| **ベースURL** | https://models.github.ai/inference/chat/completions | |
| **認証方式** | GitHub PAT (Bearer Token) | models:read スコープが必須 |
| **プロトコル** | HTTP/2, REST | JSONペイロードを使用 |
| **APIバージョン** | 2022-11-28 などをヘッダー指定 | |

過去のドキュメントや一部のモデルでは models.inference.ai.azure.com というエンドポイントが参照される場合があるが、GitHubは models.github.ai への統一を進めており、本設計でも後者を採用する。これにより、将来的な認証フローの変更やモデル追加に柔軟に対応できる。

### 3.2 モデル選定：GPT-4o vs GPT-4o-mini

GitHub Modelsでは多数のモデルが利用可能であるが、PR修正タスクにおいては以下の2モデルが主要な候補となる。

1. **GPT-4o (Omni)**:
    * **特徴**: 最上位の推論能力を持ち、複雑なロジック修正や文脈理解に優れる。
    * **コンテキストウィンドウ**: 128kトークン。
    * **適用シナリオ**: リファクタリング、バグ修正、セキュリティ脆弱性の対応。
    * **制約**: 無料枠でのレート制限が厳しく（RPM/TPMが低い）、連続使用には不向き。
2. **GPT-4o-mini**:
    * **特徴**: 高速かつ低コスト。コードの整形、単純な修正、ドキュメント生成に最適。
    * **コンテキストウィンドウ**: 128kトークン（APIリクエスト制限は入力8k/出力4k程度の場合あり）。
    * **適用シナリオ**: 命名規則の修正、コメントの追加、型定義の微修正。
    * **利点**: 無料枠でも比較的緩やかなレート制限が適用される傾向にあり、本ツールのデフォルトモデルとして最適である。

**推奨戦略**: デフォルトでは GPT-4o-mini を使用し、ユーザーが明示的に「高度な修正」を選択した場合のみ GPT-4o を使用するハイブリッド構成とする。これにより、無料枠の制限内で最大限のUXを提供する。

### 3.3 レート制限 (Rate Limits) の詳細と対策

GitHub Modelsの無料枠におけるレート制限は、APIレスポンスヘッダーを通じて動的に通知される。

* x-ratelimit-limit: 期間あたりの最大リクエスト数。
* x-ratelimit-remaining: 残りのリクエスト数。
* x-ratelimit-reset: 制限がリセットされるUNIXタイムスタンプ。

特に無料枠（Free Tier）では、Requests Per Minute (RPM) だけでなく Tokens Per Minute (TPM) の制限も厳格である。例えば、GPT-4oの場合、TPM制限により長いファイルの内容をコンテキストとして送信すると、即座に制限に達するリスクがある。

**実装上の対策**:

1. **トークン計算のフロントエンド実装**: リクエスト送信前に概算トークン数を計算し、制限を超過しそうな場合はファイル内容をトリミングする、またはユーザーに警告を表示する。
2. **Exponential Backoff**: 429 Too Many Requests エラーを受け取った際、retry-after ヘッダーまたは x-ratelimit-reset の値に基づいて待機時間を計算し、自動的に再試行するロジックをService Workerに実装する。

---

## 4. アーキテクチャ設計：Manifest V3とセキュリティ境界

### 4.1 全体アーキテクチャ概要

本システムは、Chrome拡張機能のManifest V3仕様に準拠した3層構造を採用する。各コンポーネントは厳格なセキュリティ境界によって分離され、メッセージパッシングによって連携する。

| コンポーネント | 役割と責任 | 権限 (Permissions) |
| :--- | :--- | :--- |
| **Content Script** | DOMの読み取り（コード、コメント）、UI（修正ボタン）の注入、ユーザー操作の検知。 | activeTab (現在のページへのアクセス) |
| **Service Worker** | 外部API通信（GitHub API, Models API）、トークン管理、ビジネスロジックの実行。 | storage, host_permissions |
| **Options Page** | ユーザー設定（PAT入力）、モデル選択、プロンプトテンプレートの編集。 | storage |

### 4.2 CORS (Cross-Origin Resource Sharing) 回避メカニズム

ブラウザ拡張機能開発において最も一般的な課題の一つがCORSエラーである。通常、Webページ上のJavaScript（Content Scriptを含む）から異なるオリジン（この場合は api.github.com や models.github.ai）へ fetch リクエストを送信すると、ブラウザはセキュリティポリシーに基づきこれをブロックする。

Manifest V3において、この問題に対する解は **Service Workerへの通信委譲** である。

* **メカニズム**:
    1. manifest.json の host_permissions にアクセス先のURLパターン（https://api.github.com/* 等）を明記する。
    2. Content ScriptはAPIリクエストを行わず、chrome.runtime.sendMessage を使用してService Workerに「リクエスト依頼」を送信する。
    3. Service Workerは host_permissions によって特権的なネットワークアクセス権を持っているため、CORS制約を受けることなく外部サーバーと通信できる。
    4. 取得したデータはメッセージレスポンスとしてContent Scriptに返却される。

このアーキテクチャは、「CORS Unlocker」のような汎用拡張機能が採用している手法と同様であるが、本設計では特定のGitHubエンドポイントのみを許可することでセキュリティを担保する。

### 4.3 セキュリティと認証情報の管理

ユーザーのGitHub PAT（Personal Access Token）は非常に機微な情報であり、その取り扱いには細心の注意が必要である。

1. **トークンの保存**: chrome.storage.local を使用して保存する。chrome.storage.sync はGoogleアカウント経由で同期される利便性があるが、企業ポリシーによっては外部クラウドへのトークン保存が禁止されている場合があるため、デフォルトはローカル保存とする。
2. **メモリ内保持の最小化**: Service Workerはイベント駆動で起動・終了を繰り返す（Ephemeral）ため、変数は永続化されない。リクエストのたびにStorageからトークンを読み出し、使用後は直ちに破棄される設計とする。
3. **Content Scriptへの非開示**: **最も重要なセキュリティ原則**として、PATは決してContent Script（Webページ側のコンテキスト）に渡してはならない。もしContent Scriptが侵害された場合（例えばGitHub上で悪意のあるXSSが実行された場合）、DOM内にある情報は盗み出される可能性がある。したがって、認証ヘッダーの付与は必ずService Worker内部で行い、Content Scriptには「APIの実行結果」のみを返すアーキテクチャを徹底する。

---

## 5. 実装詳細：コンテキスト抽出とDOM解析

### 5.1 GitHub PR UIのDOM構造解析

GitHubのPR画面（Files changedタブ）はReactによって動的にレンダリングされており、DOM構造は複雑かつ変更されやすい。安定したデータ抽出のためには、視覚的なクラス名ではなく、データ属性（Data Attributes）を優先的に利用する戦略が必要である。

#### 必要な情報の特定手法

レビュー修正を行うためには、以下の情報をDOMから正確に抽出する必要がある。

1. **ファイルパス**:
    * セレクタ例: `.file-header[data-path]` や `div.js-file-header`。
    * 多くのファイルヘッダーには `data-path="src/utils.ts"` のような属性が付与されている。これを利用することで、表示上のテキスト（省略されている可能性がある）に依存せず正確なパスを取得できる。
2. **レビューコメントの内容**:
    * セレクタ例: `.review-comment-body`。
    * コメントIDは `div.review-comment` の `id` 属性（例: `pullrequestreview-123456` または `issuecomment-123456`）から解析する。
3. **対象行番号とDiff位置**: これが最も難易度の高い部分である。GitHubのAPIはコメントの位置を position（Diff内の相対位置）または line（ファイル内の絶対行番号）で管理しているが、APIのバージョンやエンドポイントによって要求されるパラメータが異なる。
    * **DOMからの取得**: 行番号セル（`<td class="blob-num">`）には `data-line-number` 属性が存在する。コメントが表示されている行の直近の `data-line-number` を探索することで、絶対行番号を取得する。

### 5.2 Rawコンテンツの取得戦略

DOMからコードをスクレイピングする方法（innerText の取得）は簡便だが、以下の欠点がある。

* 長い行が省略されている可能性がある。
* 構文ハイライトのためのHTMLタグが混入し、純粋なコードの抽出が困難。
* 前後の文脈（import文やクラス定義など）が画面外にある場合、AIが正確な修正を行えない。

**推奨戦略**: GitHub REST APIによるRawコンテンツ取得。DOMからは「ファイルパス」と「SHA（コミットハッシュ）」のみを取得し、実際のコード内容はGitHub REST APIの `GET /repos/{owner}/{repo}/contents/{path}` エンドポイントを使用して取得する。

* **パラメータ**: `ref` パラメータにPRのHEADコミットのSHAを指定する。これにより、PR時点での正確なファイル内容が得られる。
* **部分取得**: ファイル全体が巨大な場合、トークン制限を圧迫する。AST（抽象構文木）パーサを拡張機能内に組み込むのは重量過多となるため、簡易的に「指摘行の前後50行」を抽出するロジックを実装する。

### 5.3 MutationObserverによる動的インジェクション

GitHubのSPA遷移（例えば「Conversation」タブから「Files changed」タブへの切り替え）では、ページリロードが発生せずDOMのみが書き換わる。したがって、`window.onload` だけでは不十分である。

`MutationObserver` を使用し、PRのタイムラインやDiffが表示されるコンテナ要素（例: `#files` や `.js-diff-container`）の変更を監視する。新しいレビューコメントがDOMに追加されたことを検知した瞬間に、その横に「AI修正」ボタンを注入する。

```javascript
// 実装イメージ（概念コード）  
const observer = new MutationObserver((mutations) => {  
  for (const mutation of mutations) {  
    if (mutation.addedNodes.length) {  
      injectAIButtons(); // 未処理のコメントにボタンを追加  
    }  
  }  
});  
observer.observe(document.querySelector('.repository-content'), { childList: true, subtree: true });
```

---

## 6. コアロジック：プロンプトエンジニアリングと推論

### 6.1 プロンプト設計の原則

GitHub Models APIへ送信するプロンプトは、AIが「修正パッチ」を正確に生成できるように最適化されている必要がある。曖昧な指示は、不必要な解説やMarkdownフォーマットの欠落を招き、後続の自動処理を失敗させる原因となる。

#### システムプロンプト構成

AIの役割と出力形式を厳格に定義する。

**System Prompt**:

"You are an expert software engineer acting as an automated code repair agent. Your task is to apply the code review suggestion to the provided code snippet.

**Constraints**:
1. Output ONLY the modified code block wrapped in standard markdown fences (e.g., ```javascript ... ```).
2. Do NOT explain your changes.
3. Do NOT include existing code that is unchanged unless necessary for context.
4. Preserve indentation and coding style."

### 6.2 コンテキストの構築 (User Prompt)

ユーザープロンプトには、修正に必要な全ての情報を含める。

**User Prompt Template**:

"File: src/components/Button.tsx (Language: TypeScript)

**Review Comment**:
'${review_comment}'

**Original Code (Line ${start_line} to ${end_line})**:
```typescript
${original_code_snippet}
```

Please generate the fixed code to resolve the review comment."

### 6.3 幻覚 (Hallucination) の抑制と検証

AIは時として、存在しないメソッドを捏造したり、指示と異なる行を変更したりする（幻覚）。

これを防ぐための技術的アプローチとして以下を採用する。

1. **Temperature設定**: 推論パラメータの `temperature` を 0.0 〜 0.2 に設定し、創造性を抑制して決定論的な出力を促す。
2. **Diff検証（ポストプロセス）**: AIからのレスポンスを受け取った後、Service Worker内で簡易的な文字列比較を行う。元のコードと全く同じ場合や、指摘と無関係な行が大幅に変更されている場合は、ユーザーに警告を表示するか、再生成を促す。

---

## 7. 修正適用メカニズム: Suggestion vs Direct Commit

本システムにおける最大の価値提供は、生成されたコードをGitHubへどう反映させるかにある。ここでは2つの主要なアプローチを詳細に比較・設計する。

### 7.1 戦略A: Suggested Changes (推奨アプローチ)

GitHubには、コメント内に特定のMarkdown構文を記述することで、diffとして適用可能な「提案」を作成する機能がある。

#### 仕組みとAPIペイロード

AIが生成したコードブロックを `suggestion` 構文でラップし、REST APIを用いて「コメントへの返信」として投稿する。

**APIエンドポイント**: `POST /repos/{owner}/{repo}/pulls/{pull_number}/comments/{comment_id}/replies`

**ペイロード構成**:
```json
{  
  "body": "```suggestion\nconst fixedVariable = calculateValue();\n```"  
}
```

**メリット**:
* **安全性**: ユーザー（PR作成者）がGitHub UI上で差分を確認し、「Commit suggestion」ボタンを押すまでコードは変更されない。AIの誤りを人間がフィルタリングできる。
* **実装の単純さ**: 既存のコメントIDに対してテキストを投稿するだけであり、Gitの内部構造（Tree/Blob）を操作する必要がない。
* **競合回避**: PRが更新されていても、Suggestionは特定の行に対する提案として残るため、Direct Commitに比べて競合リスクが低い。

**デメリット**:
* ユーザーが手動で「Commit suggestion」を押す必要がある（完全自動ではない）。

### 7.2 戦略B: Direct Commit (サーバーレスGit操作)

ユーザーが「即座に適用」を望む場合、API経由で直接リポジトリ内のファイルを更新する。

#### 仕組みとAPIフロー

この処理は複雑であり、以下の手順をService Worker内でアトミックに近い形で実行する必要がある。

1. **最新のファイルBlobを取得**: `GET /repos/.../contents/{path}` で `sha` (Blob SHA) を取得。
2. **Base64エンコード**: 修正後のファイル全体をBase64形式にエンコードする。
3. **更新リクエスト**: `PUT /repos/.../contents/{path}`。

**APIペイロード**:
```json
{  
  "message": "fix: Apply AI suggestion for review comment",  
  "content": "b64_encoded_content...",  
  "sha": "original_blob_sha",  
  "branch": "feature-branch-name"  
}
```

**リスクと課題**:
* **Race Condition**: ユーザーが修正ボタンを押した直後に別のコミットがプッシュされると、`sha` の不一致でAPIが失敗する（409 Conflict）。
* **データ損失**: ファイル全体を書き換えるため、部分的な取得ロジック（5.2節）と組み合わせる場合、ファイル全体を再構築するロジックが必要となり、バグによるコード消失のリスクがある。

**結論**:

本設計では、**戦略A (Suggested Changes)** をデフォルトかつ推奨の実装とする。これは「AIアシスタント」としての立ち位置（最終決定権は人間にある）に合致し、実装リスクも最小化できるためである。戦略Bは「Advanced Option」として、リスクを理解したユーザー向けにのみ開放する。

---

## 8. ユーザーエクスペリエンス (UX) とエラーハンドリング

### 8.1 ユーザーインターフェース設計

ブラウザ拡張機能としてのUIは、可能な限りGitHubのネイティブUIに溶け込むべきである。

* **トリガー**: レビューコメントのツールバー（「Edit」「Delete」などが並ぶ場所）に「✨ AI Fix」ボタンを追加する。
* **ローディング状態**: 推論中はボタンをスピナーアイコンに変更し、ユーザーに処理中であることを明示する。
* **完了通知**:
    * Suggestion投稿成功時: ページをリロードするか、DOMを部分更新して、追加されたSuggestionコメントを即座に表示する。
    * エラー時: トースト通知（画面下部のポップアップ）でエラー内容を表示する。

### 8.2 エラーハンドリングの詳細戦略

サーバーレス環境ではネットワークエラーやAPI制限が頻発するため、堅牢なエラー処理が必要である。

| エラー種別 | 原因 | 対応策 |
| :--- | :--- | :--- |
| **401 Unauthorized** | PATが無効、またはスコープ不足。 | オプションページへの誘導リンクを表示し、正しいスコープ（models:read, repo）での再発行を促す。 |
| **403 Forbidden (CORS)** | host_permissions 設定漏れ。 | 開発者向けログに出力（ユーザーには「拡張機能エラー」と通知）。 |
| **429 Too Many Requests** | モデルAPIまたはGitHub APIのレート制限超過。 | x-ratelimit-reset ヘッダーを解析し、「API制限中です。あとXX分後に再試行可能です」と具体的に表示する。 |
| **422 Unprocessable Entity** | Suggestionの行位置不正など。 | 「修正箇所の特定に失敗しました。最新のコードをpullしてから再試行してください」と通知。 |

### 8.3 無料枠ユーザーへの配慮

GitHub Modelsの無料枠を利用する場合、トークン消費を抑える工夫がUXに求められる。

* **Diffのみ送信**: ファイル全体ではなく、変更箇所周辺（Hunk）のみを送信するモードを実装する。
* **モデル切り替えUI**: クォータが枯渇した場合、一時的に軽量モデル（例: gpt-4o-mini からさらに軽量なモデルがあればそれ、あるいはフォールバック）への切り替えを促す。

---

## 9. 将来の展望と拡張性

### 9.1 マルチファイル修正への対応

現在の設計は単一ファイル内の修正に焦点を当てているが、実際のレビューでは「関数名の変更に伴い、呼び出し元もすべて修正する」といったマルチファイル修正が求められる場合がある。これに対応するには、リポジトリ全体の依存関係解析が必要となり、現状のサーバーレス構成（クライアントサイドのみ）では計算資源的に困難である。将来的には、GitHub Copilot WorkspaceのようなクラウドベースのエージェントAPIとの連携が視野に入る。

### 9.2 エンタープライズ対応とセキュリティコンプライアンス

企業利用においては、以下の機能追加が必要となる。

* **管理ポリシー**: 組織管理者が拡張機能の利用を制御できるポリシー設定（Chrome Enterprise Policy）。
* **監査ログ**: AIによる修正履歴を監査ログとして保存する機能。
* **BYOK (Bring Your Own Key)**: GitHub Modelsの無料枠ではなく、組織契約のAzure OpenAI Serviceのエンドポイントを指定可能にする構成。

## 10. 結論

本レポートで提示したサーバーレスChrome拡張機能の設計は、GitHub Models APIの能力とManifest V3のセキュリティモデルを巧みに組み合わせることで、追加のインフラコストなしに高度な開発者支援ツールを実現するものである。

特に、Service WorkerをCORSプロキシとして利用するアーキテクチャと、GitHub標準のSuggested Changes機能を活用した修正適用フローは、セキュリティリスクを最小限に抑えつつ、既存の開発ワークフローへのシームレスな統合を可能にする。この「軽量かつ強力」なツールは、日々の開発業務における微細な摩擦（Friction）を取り除き、エンジニアがより創造的な問題解決に集中できる環境を提供する基盤となるだろう。

---

## 付録：データ表と参照仕様

### 表1: GitHub Models API (Free Tier) と Copilot プランの比較

| 機能 / プラン | GitHub Copilot Free | GitHub Copilot Pro | GitHub Models (Marketplace API) |
| :--- | :--- | :--- | :--- |
| **対象ユーザー** | 個人開発者 (学生・OSS等) | 個人・プロフェッショナル | 全ユーザー (PAT利用) |
| **モデルアクセス** | GPT-4o, GPT-4o-mini等 | 全モデルへの優先アクセス | Playground提供モデル (制限あり) |
| **レート制限** | 限定的 (月間補完数等) | 緩和されている | Tierに基づく (Low/High) |
| **API利用** | IDE経由が主 | IDE + Chat | REST API直接利用可 |

### 表2: 推奨されるAPIインタラクションフロー

| ステップ | アクション | APIエンドポイント / メソッド | 必須パラメータ |
| :--- | :--- | :--- | :--- |
| 1 | 認証確認 | GET /user | Authorization: Bearer <PAT> |
| 2 | ファイル取得 | GET /repos/{owner}/{repo}/contents/{path} | ref={pr_head_sha} |
| 3 | AI推論 | POST https://models.github.ai/.../chat/completions | messages, model |
| 4 | 提案投稿 | POST /repos/.../comments/{id}/replies | body (suggestion block) |

### 引用・参考文献リスト

本レポートの記述は、以下の調査資料に基づいている。

* API仕様:
* 認証・セキュリティ:
* レート制限・プラン:
* ブラウザ拡張機能・CORS:
* PR・レビュー機能:

#### 引用文献

1. Quickstart for GitHub Models - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/github-models/quickstart
2. REST API endpoints for models catalog - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest/models/catalog
3. Solving the inference problem for open source AI projects with GitHub Models, 2月 6, 2026にアクセス、 https://github.blog/ai-and-ml/llms/solving-the-inference-problem-for-open-source-ai-projects-with-github-models/
4. oe/cors-unlocker: enable web app cors in modern browser - GitHub, 2月 6, 2026にアクセス、 https://github.com/oe/cors-unlocker
5. Receiving CORS Policy on Chrome Extension Background Script - Stack Overflow, 2月 6, 2026にアクセス、 https://stackoverflow.com/questions/60461806/receiving-cors-policy-on-chrome-extension-background-script
6. How can I manually add suggestions in code reviews on GitHub? - Stack Overflow, 2月 6, 2026にアクセス、 https://stackoverflow.com/questions/54239733/how-can-i-manually-add-suggestions-in-code-reviews-on-github
7. Incorporating feedback in your pull request - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/articles/incorporating-feedback-in-your-pull-request
8. Rate limits for the REST API - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api
9. Rate Limits and File Uploads in GitHub Models · community · Discussion #149698, 2月 6, 2026にアクセス、 https://github.com/orgs/community/discussions/149698
10. software-agent-sdk/examples/03_github_workflows/02_pr_review/agent_script.py at main, 2月 6, 2026にアクセス、 https://github.com/OpenHands/software-agent-sdk/blob/main/examples/03_github_workflows/02_pr_review/agent_script.py
11. GitHub REST API documentation, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest?apiVersion=2022-11-28
12. Background scripts - Mozilla - MDN Web Docs, 2月 6, 2026にアクセス、 https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Background_scripts
13. About individual GitHub Copilot plans and benefits, 2月 6, 2026にアクセス、 https://docs.github.com/en/copilot/concepts/billing/individual-plans
14. GitHub Models billing, 2月 6, 2026にアクセス、 https://docs.github.com/billing/managing-billing-for-your-products/about-billing-for-github-models
15. Prototyping with AI models - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/github-models/prototyping-with-ai-models
16. GitHub Models Provider - Promptfoo, 2月 6, 2026にアクセス、 https://www.promptfoo.dev/docs/providers/github/
17. REST API endpoints for models inference - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest/models/inference?apiVersion=2022-11-28
18. Why does GitHub Models have two different endpoints? · community · Discussion #157126, 2月 6, 2026にアクセス、 https://github.com/orgs/community/discussions/157126
19. Supported AI models in GitHub Copilot, 2月 6, 2026にアクセス、 https://docs.github.com/copilot/reference/ai-models/supported-models
20. OpenAI GPT-4o mini · GitHub Models, 2月 6, 2026にアクセス、 https://github.com/marketplace/models/azure-openai/gpt-4o-mini
21. CORS error on login API · community · Discussion #169674 - GitHub, 2月 6, 2026にアクセス、 https://github.com/orgs/community/discussions/169674
22. Managing your personal access tokens - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
23. Filtering files in a pull request - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/articles/filtering-files-in-a-pull-request-by-file-type
24. Ability to make the file list wider · community · Discussion #67849 - GitHub, 2月 6, 2026にアクセス、 https://github.com/orgs/community/discussions/67849
25. CHANGELOG.md - uxsolutions/bootstrap-datepicker - GitHub, 2月 6, 2026にアクセス、 https://github.com/uxsolutions/bootstrap-datepicker/blob/master/CHANGELOG.md
26. Pull Request Review Comment wrong position from the diff? - Stack Overflow, 2月 6, 2026にアクセス、 https://stackoverflow.com/questions/52820680/pull-request-review-comment-wrong-position-from-the-diff
27. PR review comment position - github - Stack Overflow, 2月 6, 2026にアクセス、 https://stackoverflow.com/questions/37866485/pr-review-comment-position
28. GitHub-Dark/index.html at master, 2月 6, 2026にアクセス、 https://github.com/StylishThemes/GitHub-Dark/blob/master/index.html
29. REST API endpoints for repository contents - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/rest/repos/contents
30. use-models-at-scale.md - GitHub, 2月 6, 2026にアクセス、 https://github.com/github/docs/blob/main/content/github-models/github-models-at-scale/use-models-at-scale.md
31. Is there a way to suggest edits to a PR? - Stack Overflow, 2月 6, 2026にアクセス、 https://stackoverflow.com/questions/68148336/is-there-a-way-to-suggest-edits-to-a-pr
32. REST API endpoints for pull request review comments - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest/pulls/comments
33. Authenticating to the REST API - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest/authentication/authenticating-to-the-rest-api
34. Using CORS and JSONP to make cross-origin requests - GitHub Docs, 2月 6, 2026にアクセス、 https://docs.github.com/en/rest/using-the-rest-api/using-cors-and-jsonp-to-make-cross-origin-requests
