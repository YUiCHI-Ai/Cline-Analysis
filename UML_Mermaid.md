# Clineメイン処理フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Extension as extension.ts
    participant Provider as ClineProvider.ts
    participant Core as Cline.ts
    participant API as APIハンドラー
    participant Tools as 各種ツール

    %% 拡張機能の起動
    User->>Extension: VSCode拡張機能を起動
    Extension->>Extension: activate() %% 拡張機能の起動時に呼び出される初期化関数
    Extension->>Provider: new ClineProvider() %% Cline機能を提供するプロバイダーを作成
    Provider->>Provider: 初期化処理 %% 設定の読み込みや状態の初期化
    Extension->>Extension: コマンド登録 %% VSCodeコマンドパレットからの操作を可能にする
    Extension->>Extension: WebViewProvider登録 %% UIを表示するためのWebViewを登録

    %% サイドバー表示
    User->>Extension: サイドバーを開く
    Extension->>Provider: resolveWebviewView() %% サイドバーのWebView表示を解決する関数
    Provider->>Provider: Webviewを初期化 %% HTML/CSSの読み込みとUIの準備
    Provider->>Provider: setWebviewMessageListener() %% WebViewとの通信用リスナーを設定

    %% タスク開始
    User->>Provider: タスク入力
    Provider->>Provider: webviewMessageListener処理 %% ユーザー入力メッセージを受信して処理
    Provider->>Provider: initClineWithTask() %% タスクに基づいてClineを初期化
    Provider->>Core: new Cline() %% コア処理を担当するClineインスタンスを作成
    Core->>Core: 初期化処理 %% コンテキスト設定やモデル準備
    Core->>Core: startTask() %% タスク処理を開始
    Core->>Core: initiateTaskLoop() %% AIとの対話ループを開始

    %% APIリクエスト
    Core->>Core: recursivelyMakeClineRequests() %% 再帰的にAIリクエストを処理
    Core->>Core: loadContext() %% タスクに必要なコンテキスト情報を読み込み
    Core->>API: createMessage() %% AIモデルへのリクエストを作成して送信
    API-->>Core: ストリーミングレスポンス %% AIからの応答をリアルタイムで受信
    Core->>Core: presentAssistantMessage() %% AIの応答をユーザーに表示

    %% ツール使用
    Core->>Core: ツール使用を検出 %% AIレスポンス内のツール使用要求を識別
    Core->>User: ツール承認リクエスト %% ユーザーにツール実行の許可を求める
    User-->>Core: 承認 %% ユーザーがツール実行を承認
    Core->>Tools: ツール実行 %% 指定されたツールを実行
    Tools-->>Core: 実行結果 %% ツール実行の結果を取得
    Core->>Core: 結果をユーザーに表示 %% ツール実行結果をUIに表示

    %% 次のAPIリクエスト
    Core->>API: 次のリクエスト %% ツール実行結果を含めた次のAIリクエスト
    API-->>Core: ストリーミングレスポンス %% 新しい応答をストリーミングで受信

    %% タスク完了
    Core->>Core: attempt_completionツール検出 %% タスク完了を示す特殊ツールを検出
    Core->>User: 完了結果表示 %% タスク完了結果をユーザーに表示
    User->>Provider: 新しいタスク開始または終了 %% ユーザーが次のアクションを選択
    Provider->>Core: clearTask() %% 現在のタスクをクリアして新しいタスクの準備
```

## 主要コンポーネントの説明

### extension.ts
VSCode拡張機能のエントリーポイント。拡張機能の起動時に実行され、ClineProviderの初期化やコマンド登録を行います。

### ClineProvider.ts
Webviewの管理とユーザーインターフェースの処理を担当します。ユーザーからの入力を受け取り、Clineクラスに処理を委譲します。

### Cline.ts
コア処理ロジックとツール実行を担当します。APIリクエストの送信、レスポンスの処理、ツールの実行などを行います。

## 主要な処理フロー

1. **拡張機能の起動**：
   - VSCode拡張機能が起動すると、`extension.ts`の`activate()`関数が実行されます
   - `ClineProvider`のインスタンスが作成され、WebViewProviderとして登録されます

2. **サイドバー表示**：
   - ユーザーがサイドバーを開くと、`resolveWebviewView()`が呼び出されます
   - Webviewが初期化され、メッセージリスナーが設定されます

3. **タスク開始**：
   - ユーザーがタスクを入力すると、`webviewMessageListener`がメッセージを処理します
   - `initClineWithTask()`が呼び出され、`Cline`インスタンスが作成されます
   - `startTask()`が実行され、タスクループが開始されます

4. **APIリクエスト**：
   - `recursivelyMakeClineRequests()`が実行され、APIリクエストが送信されます
   - APIからのレスポンスがストリーミングで受信され、`presentAssistantMessage()`で処理されます

5. **ツール使用**：
   - レスポンス内でツール使用が検出されると、ユーザーに承認を求めます
   - 承認されると、対応するツールが実行され、結果がユーザーに表示されます
   - ツール実行結果を含めて次のAPIリクエストが送信されます

6. **タスク完了**：
   - `attempt_completion`ツールが検出されると、タスク完了と見なされます
   - 完了結果がユーザーに表示され、新しいタスクを開始するか終了するかを選択できます