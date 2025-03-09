# Clineメイン処理フロー

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Extension as extension.ts
    participant Provider as ClineProvider.ts
    participant Core as Cline.ts
    participant Context as コンテキスト管理
    participant API as APIハンドラー
    participant Tools as 各種ツール
    participant Models as AIモデル

    %% 拡張機能の起動
    User->>Extension: VSCode拡張機能を起動
    Extension->>Extension: activate() %% 拡張機能の起動時に呼び出される初期化関数
    Extension->>Provider: new ClineProvider() %% Cline機能を提供するプロバイダーを作成
    Provider->>Provider: 初期化処理 %% 設定の読み込みや状態の初期化
    Provider->>Provider: loadSettings() %% ユーザー設定の読み込み
    Provider->>Provider: loadCustomModes() %% カスタムモードの読み込み
    Provider->>Provider: initializeFileWatchers() %% 設定ファイルの変更監視
    Extension->>Extension: コマンド登録 %% VSCodeコマンドパレットからの操作を可能にする
    Extension->>Extension: WebViewProvider登録 %% UIを表示するためのWebViewを登録

    %% サイドバー表示
    User->>Extension: サイドバーを開く
    Extension->>Provider: resolveWebviewView() %% サイドバーのWebView表示を解決する関数
    Provider->>Provider: Webviewを初期化 %% HTML/CSSの読み込みとUIの準備
    Provider->>Provider: setWebviewMessageListener() %% WebViewとの通信用リスナーを設定
    Provider->>Provider: setupThemeListener() %% テーマ変更の監視と適用

    %% タスク開始
    User->>Provider: タスク入力
    Provider->>Provider: webviewMessageListener処理 %% ユーザー入力メッセージを受信して処理
    Provider->>Provider: initClineWithTask() %% タスクに基づいてClineを初期化
    Provider->>Provider: validateMode() %% 選択されたモードの検証
    Provider->>Core: new Cline() %% コア処理を担当するClineインスタンスを作成
    Core->>Core: 初期化処理 %% コンテキスト設定やモデル準備
    Core->>Core: startTask() %% タスク処理を開始
    Core->>Core: initiateTaskLoop() %% AIとの対話ループを開始

    %% コンテキスト準備
    Core->>Context: loadContext() %% タスクに必要なコンテキスト情報を読み込み
    Context->>Context: parseMentions() %% @ファイルパスなどのメンションを解析
    Context->>Context: getEnvironmentDetails() %% 環境情報を取得
    Note over Context: VSCode表示ファイル、開いているタブ、<br>アクティブなターミナル情報、<br>現在の時刻、ファイル構造など
    Context-->>Core: 解析済みコンテキスト

    %% プロンプト準備
    Core->>Core: preparePrompt() %% システムプロンプトとユーザーメッセージを構築
    Core->>Core: buildSystemPrompt() %% 基本システムプロンプトの構築
    Core->>Core: addUserInstructions() %% カスタム指示の追加
    Core->>Core: addLanguagePreference() %% 言語設定の追加
    Core->>Core: addClineRules() %% .clinerules設定の追加
    Core->>Core: addClineIgnore() %% .clineignore設定の追加
    Core->>Core: addModeSpecificInstructions() %% モード固有の指示を追加

    %% モデル選択
    Core->>Core: selectModel() %% 設定に基づいて適切なAIモデルを選択
    Core->>Core: checkModelAvailability() %% モデルの利用可能性を確認
    Core->>Core: applyModelFallback() %% 必要に応じて代替モデルを選択
    Note over Core: モデル選択基準：<br>ユーザー設定、タスク複雑さ、<br>利用可能なAPIキー

    %% コンテキスト管理
    Core->>Context: manageContextWindow() %% コンテキストウィンドウサイズを管理
    Context->>Context: calculateTokenUsage() %% 現在のトークン使用量を計算
    Context->>Context: getTruncationRange() %% 切り詰め範囲を決定
    Context->>Context: getTruncatedMessages() %% 切り詰められた会話履歴を取得
    Context-->>Core: 最適化されたコンテキスト

    %% APIリクエスト
    Core->>API: createMessage() %% AIモデルへのリクエストを作成して送信
    API->>API: setModelParameters() %% モデルパラメータを設定
    Note over API: 温度、最大トークン数、<br>トップP/K、頻度ペナルティなど
    API->>Models: sendRequest() %% 選択されたモデルにリクエスト送信
    Note over API,Models: Anthropic, OpenAI, Google,<br>Mistral, ローカルモデルなど
    Models-->>API: ストリーミングレスポンス開始

    %% レスポンス処理ループ
    loop ストリーミング処理
        Models-->>API: レスポンスチャンク
        API-->>Core: チャンク転送
        Core->>Core: processStreamingResponse() %% ストリーミングデータを処理
        Core->>Core: updateTokenUsage() %% トークン使用量を更新
        Core->>Core: renderMarkdown() %% マークダウンをレンダリング
        Core->>User: incrementalDisplay() %% 増分表示
    end

    %% ツール使用検出
    Core->>Core: parseToolCalls() %% レスポンス内のツール使用要求を解析
    Core->>Core: detectToolUse() %% AIレスポンス内のツール使用要求を識別
    Core->>Core: validateToolRequest() %% ツールリクエストの形式と権限を検証
    Core->>Core: checkToolPermissions() %% 現在のモードでツールが使用可能か確認
    Core->>User: requestToolApproval() %% ユーザーにツール実行の許可を求める
    Note over Core,User: ツール名、パラメータ、潜在的な影響を表示
    
    %% ユーザー承認と実行
    alt ユーザーが承認した場合
        User-->>Core: 承認
        Core->>Tools: executeToolRequest() %% 指定されたツールを実行
        Note over Core,Tools: ファイル操作、コマンド実行、ブラウザ操作など
        
        %% ツール実行の分岐
        alt ファイル操作ツール
            Tools->>Tools: read_file/write_to_file/apply_diff
            Tools->>Tools: validateFilePath() %% ファイルパスの検証
            Tools->>Tools: checkFilePermissions() %% ファイルアクセス権限の確認
        else コマンド実行ツール
            Tools->>Tools: execute_command
            Tools->>Tools: sanitizeCommand() %% コマンドの安全性確認
            Tools->>Tools: setupTerminal() %% ターミナル環境の準備
        else ブラウザ操作ツール
            Tools->>Tools: browser_action
            Tools->>Tools: initializePuppeteer() %% ブラウザエンジンの初期化
            Tools->>Tools: executeAction() %% ブラウザアクションの実行
        else MCP関連ツール
            Tools->>Tools: use_mcp_tool/access_mcp_resource
            Tools->>Tools: validateMcpRequest() %% MCPリクエストの検証
            Tools->>Tools: connectToMcpServer() %% MCPサーバーへの接続
        end
        
        Tools-->>Core: 実行結果
        Core->>Core: processToolResult() %% ツール実行結果を処理
        Core->>Core: formatToolResult() %% ツール結果のフォーマット
        Core->>Core: displayToolResult() %% ツール実行結果をUIに表示
        Core->>User: 実行結果を表示
    else ユーザーが拒否した場合
        User-->>Core: 拒否
        Core->>Core: handleRejectedTool() %% 拒否されたツールの処理
        Core->>Core: notifyAIOfRejection() %% AIにツール拒否を通知
        Core->>User: 拒否メッセージを表示
    end

    %% エラー処理とリトライ
    alt エラー発生時
        Core->>Core: handleApiError() %% APIエラーの処理
        Core->>Core: categorizeError() %% エラータイプの分類
        
        alt 接続エラーの場合
            Core->>Core: retryWithBackoff() %% バックオフ戦略でリトライ
            Core->>API: 再リクエスト
        else コンテキスト長超過の場合
            Core->>Context: reduceContextSize() %% コンテキストサイズを削減
            Core->>API: 縮小したコンテキストで再リクエスト
        else レート制限の場合
            Core->>Core: applyRateLimitBackoff() %% レート制限バックオフ
            Core->>User: レート制限通知
            Core->>API: 待機後に再リクエスト
        else モデルエラーの場合
            Core->>Core: switchToFallbackModel() %% 代替モデルに切り替え
            Core->>API: 代替モデルで再リクエスト
        end
    end

    %% 次のAPIリクエスト
    Core->>Context: updateContext() %% ツール実行結果をコンテキストに追加
    Context->>Context: prioritizeContent() %% 重要なコンテンツを優先
    Context->>Context: manageContextWindow() %% コンテキストウィンドウサイズを管理
    Context-->>Core: 更新されたコンテキスト
    Core->>API: createFollowUpMessage() %% ツール実行結果を含めた次のAIリクエスト
    Note over Core,API: 前回の会話履歴とツール実行結果を含める
    API->>Models: sendRequest() %% 更新されたコンテキストでリクエスト
    Models-->>API: ストリーミングレスポンス
    API-->>Core: ストリーミングレスポンス転送
    Core->>Core: handleModelResponse() %% モデルレスポンスを処理

    %% タスク完了
    Core->>Core: detectCompletionTool() %% attempt_completionツール検出
    Note over Core: タスク完了を示す特殊ツールを検出
    Core->>Core: validateCompletionResult() %% 完了結果の検証
    Core->>Core: finalizeTask() %% タスクの最終処理を実行
    Core->>Core: saveTaskHistory() %% タスク履歴を保存
    Core->>User: displayCompletionResult() %% タスク完了結果をユーザーに表示
    Note over Core,User: 完了結果とデモコマンド（オプション）を表示
    
    %% 新しいタスクまたは終了
    User->>Provider: 新しいタスク開始または終了 %% ユーザーが次のアクションを選択
    Provider->>Core: clearTask() %% 現在のタスクをクリアして新しいタスクの準備
    Provider->>Provider: resetState() %% プロバイダーの状態をリセット
    Provider->>Provider: prepareForNewTask() %% 新しいタスクの準備
```

## 主要コンポーネントの説明

### extension.ts
VSCode拡張機能のエントリーポイント。拡張機能の起動時に実行され、ClineProviderの初期化やコマンド登録を行います。

### ClineProvider.ts
Webviewの管理とユーザーインターフェースの処理を担当します。ユーザーからの入力を受け取り、Clineクラスに処理を委譲します。

### Cline.ts
コア処理ロジックとツール実行を担当します。APIリクエストの送信、レスポンスの処理、ツールの実行などを行います。

### コンテキスト管理
会話履歴やファイル情報などのコンテキストを管理します。トークン使用量の最適化や重要情報の優先順位付けを行います。

### APIハンドラー
AIモデルとの通信を担当します。リクエストの送信、レスポンスの受信、エラー処理などを行います。

### 各種ツール
ファイル操作、コマンド実行、ブラウザ操作などのツールを提供します。AIからの要求に基づいて実行されます。

### AIモデル
Anthropic（Claude）、OpenAI（GPT-4、GPT-3.5）、Google（Gemini）、Mistral AIなどの外部AIモデルとの連携を行います。

## 主要な処理フロー

1. **拡張機能の起動**：
   - VSCode拡張機能が起動すると、`extension.ts`の`activate()`関数が実行されます
   - `ClineProvider`のインスタンスが作成され、WebViewProviderとして登録されます
   - ユーザー設定、カスタムモード、ファイル監視などが初期化されます

2. **サイドバー表示**：
   - ユーザーがサイドバーを開くと、`resolveWebviewView()`が呼び出されます
   - Webviewが初期化され、メッセージリスナーが設定されます
   - テーマ変更の監視と適用が設定されます

3. **タスク開始**：
   - ユーザーがタスクを入力すると、`webviewMessageListener`がメッセージを処理します
   - `initClineWithTask()`が呼び出され、選択されたモードが検証されます
   - `Cline`インスタンスが作成され、`startTask()`が実行されます
   - `initiateTaskLoop()`が呼び出され、AIとの対話ループが開始されます

4. **コンテキスト準備**：
   - `loadContext()`が呼び出され、タスクに必要なコンテキスト情報が読み込まれます
   - ファイルメンション（@ファイルパスなど）が解析されます
   - 環境情報（VSCode表示ファイル、開いているタブ、ターミナル情報など）が取得されます

5. **プロンプト準備**：
   - `preparePrompt()`が呼び出され、システムプロンプトとユーザーメッセージが構築されます
   - 基本システムプロンプト、カスタム指示、言語設定、.clinerules、.clineignoreなどが追加されます
   - モード固有の指示が追加されます

6. **モデル選択**：
   - `selectModel()`が呼び出され、設定に基づいて適切なAIモデルが選択されます
   - モデルの利用可能性が確認され、必要に応じて代替モデルが選択されます
   - 選択基準には、ユーザー設定、タスク複雑さ、利用可能なAPIキーなどが含まれます

7. **コンテキスト管理**：
   - `manageContextWindow()`が呼び出され、コンテキストウィンドウサイズが管理されます
   - 現在のトークン使用量が計算され、切り詰め範囲が決定されます
   - 切り詰められた会話履歴が取得され、最適化されたコンテキストが返されます

8. **APIリクエスト**：
   - `createMessage()`が呼び出され、AIモデルへのリクエストが作成されて送信されます
   - モデルパラメータ（温度、最大トークン数、トップP/Kなど）が設定されます
   - 選択されたモデル（Anthropic、OpenAI、Googleなど）にリクエストが送信されます
   - ストリーミングレスポンスの受信が開始されます

9. **レスポンス処理ループ**：
   - レスポンスチャンクが受信され、`processStreamingResponse()`で処理されます
   - トークン使用量が更新され、マークダウンがレンダリングされます
   - 増分表示によりユーザーにリアルタイムでレスポンスが表示されます

10. **ツール使用検出**：
    - `parseToolCalls()`が呼び出され、レスポンス内のツール使用要求が解析されます
    - ツールリクエストの形式と権限が検証されます
    - 現在のモードでツールが使用可能か確認されます
    - ユーザーにツール実行の許可が求められます

11. **ユーザー承認と実行**：
    - ユーザーがツール実行を承認または拒否します
    - 承認された場合、対応するツール（ファイル操作、コマンド実行、ブラウザ操作など）が実行されます
    - ツール実行結果が処理され、ユーザーに表示されます
    - 拒否された場合、AIにツール拒否が通知されます

12. **エラー処理とリトライ**：
    - APIエラーが発生した場合、エラータイプが分類されます
    - 接続エラーの場合、バックオフ戦略でリトライされます
    - コンテキスト長超過の場合、コンテキストサイズが削減されて再リクエストされます
    - レート制限の場合、バックオフが適用され、待機後に再リクエストされます
    - モデルエラーの場合、代替モデルに切り替えられて再リクエストされます

13. **次のAPIリクエスト**：
    - ツール実行結果がコンテキストに追加されます
    - 重要なコンテンツが優先され、コンテキストウィンドウサイズが管理されます
    - 更新されたコンテキストで次のAIリクエストが送信されます
    - 新しいストリーミングレスポンスが処理されます

14. **タスク完了**：
    - `attempt_completion`ツールが検出されると、タスク完了と見なされます
    - 完了結果が検証され、タスクの最終処理が実行されます
    - タスク履歴が保存され、完了結果がユーザーに表示されます

15. **新しいタスクまたは終了**：
    - ユーザーが新しいタスクを開始するか終了するかを選択します
    - 現在のタスクがクリアされ、プロバイダーの状態がリセットされます
    - 新しいタスクの準備が行われます

## AIモデルとの連携

### AIモデルの選択と設定

Clineは複数のAIプロバイダーとモデルをサポートしています：

1. **サポートされるプロバイダー**：
   - Anthropic（Claude）
   - OpenAI（GPT-4、GPT-3.5）
   - Google（Gemini）
   - Mistral AI
   - その他のローカルモデルやサードパーティプロバイダー

2. **モデル選択メカニズム**：
   - ユーザー設定に基づいて自動的に選択
   - タスクの複雑さに応じて適切なモデルを選択
   - フォールバックメカニズムによる代替モデルの使用

3. **モデルパラメータ**：
   - 温度（Temperature）：創造性と決定論のバランスを制御
   - 最大トークン数：レスポンスの長さを制限
   - トップP/トップK：サンプリング方法の設定
   - 頻度ペナルティ：繰り返しを減らす設定

### プロンプトエンジニアリング

Clineは高度なプロンプトエンジニアリング技術を使用してAIの能力を最大化します：

1. **システムプロンプト構造**：
   - 役割定義：AIの専門知識と行動範囲を定義
   - ツール使用ガイドライン：利用可能なツールとその使用方法
   - 制約条件：AIの行動に関する制限事項
   - モード固有の指示：現在のモードに応じた特別な指示

2. **コンテキスト管理**：
   - スライディングウィンドウ：コンテキスト長の制限内で重要な情報を保持
   - 優先順位付け：タスク関連情報の重要度に基づく選択
   - メタデータ埋め込み：ファイル構造やシステム情報の効率的な提供

3. **カスタマイズ**：
   - ユーザー定義のカスタム指示：特定のプロジェクトやワークフローに合わせた調整
   - モードごとの最適化：コード、アーキテクト、質問応答などの特化したモード

### AIレスポンス処理

AIからのレスポンスは複数の段階で処理されます：

1. **ストリーミング処理**：
   - チャンク単位の受信：小さな部分ごとにレスポンスを受信
   - 増分表示：受信したチャンクをリアルタイムでUIに表示
   - マークダウンレンダリング：適切な書式でコンテンツを表示

2. **ツール使用検出**：
   - XMLパターン認識：ツール使用を示す特定のパターンを検出
   - パラメータ抽出：ツールに必要なパラメータを解析
   - 検証：ツールリクエストの形式と権限を検証

3. **エラー処理とリトライ**：
   - 接続エラー：ネットワーク問題の検出と再試行
   - モデルエラー：AIモデルからのエラーレスポンスの処理
   - コンテキスト長超過：コンテキスト長が制限を超えた場合の対応
   - レート制限：APIレート制限に達した場合のバックオフ戦略

### AIとのインタラクションループ

Clineは再帰的なインタラクションループを通じてタスクを進行します：

1. **初期リクエスト**：
   - ユーザータスクとシステムプロンプトを含む初期リクエスト
   - 環境情報（ファイル構造、システム情報など）の提供

2. **ツール使用サイクル**：
   - AIがツール使用を提案
   - ユーザーが承認または拒否
   - ツール実行結果をAIに返送
   - AIが結果を分析して次のステップを提案

3. **タスク完了**：
   - AIが`attempt_completion`ツールを使用してタスク完了を示す
   - 最終結果とデモコマンド（オプション）の提示
   - ユーザーフィードバックに基づく改善（必要に応じて）

4. **コンテキスト管理**：
   - 長期的なタスクでのコンテキスト管理
   - 重要な情報の保持と不要な情報の削除
   - トークン使用量の最適化

## 主要関数の詳細実装

### initiateTaskLoop() - AIとの対話ループを開始

この関数は、ユーザーのタスク入力からAIとの対話ループを開始する役割を担っています。

```typescript
private async initiateTaskLoop(userContent: UserContent, isNewTask: boolean): Promise<void> {
    let nextUserContent = userContent;
    let includeFileDetails = true;
    while (!this.abort) {
        const didEndLoop = await this.recursivelyMakeClineRequests(nextUserContent, includeFileDetails, isNewTask);
        includeFileDetails = false; // 最初のリクエストでのみファイル詳細を含める

        if (didEndLoop) {
            // タスクが完了した場合（attempt_completionが呼ばれた場合）、ループを抜ける
            break;
        } else {
            // AIがツールを使用せずにテキストのみで応答した場合、タスク継続を促す
            nextUserContent = [
                {
                    type: "text",
                    text: formatResponse.noToolsUsed(),
                }
            ];
            this.consecutiveMistakeCount++;
        }
    }
}
```

主な処理内容：
- ユーザーのタスク内容（userContent）を受け取り、AIとの対話ループを開始します
- 最初のリクエストでは環境情報（ファイル構造など）を含めます（includeFileDetails = true）
- `recursivelyMakeClineRequests()`を呼び出してAIリクエストを処理します
- AIがツールを使用せずにテキストのみで応答した場合、タスク継続を促すメッセージを送信します
- ユーザーがタスクをキャンセルするか、AIが`attempt_completion`ツールを使用するまでループを継続します
- 連続してツールを使用しない場合、consecutiveMistakeCountをインクリメントして、一定回数以上になると警告を表示します

### recursivelyMakeClineRequests() - 再帰的にAIリクエストを処理

この関数は、AIとのリクエスト・レスポンスサイクルを管理する中核的な関数です。

```typescript
async recursivelyMakeClineRequests(
    userContent: UserContent,
    includeFileDetails: boolean = false,
    isNewTask: boolean = false
): Promise<boolean> {
    // コンテキスト情報のロード
    const [parsedUserContent, environmentDetails] = await this.loadContext(userContent, includeFileDetails);
    userContent = parsedUserContent;
    userContent.push({ type: "text", text: environmentDetails });
    
    // APIリクエストの送信と応答の処理
    await this.addToApiConversationHistory({
        role: "user",
        content: userContent,
    });
    
    // APIストリームの初期化と処理
    const stream = this.attemptApiRequest(previousApiReqIndex);
    
    // ストリーミングレスポンスの処理
    for await (const chunk of stream) {
        // チャンクの処理（使用量、推論、テキスト）
        // アシスタントメッセージの解析とツール使用の検出
        // presentAssistantMessage()を呼び出してUIに表示
    }
    
    // アシスタントの応答をAPIの会話履歴に追加
    await this.addToApiConversationHistory({
        role: "assistant",
        content: [{ type: "text", text: assistantMessage }],
    });
    
    // ツール使用の有無を確認して次のステップを決定
    const didToolUse = this.assistantMessageContent.some((block) => block.type === "tool_use");
    
    if (!didToolUse) {
        // ツール使用がない場合、タスク継続を促す
        this.userMessageContent.push({
            type: "text",
            text: formatResponse.noToolsUsed(),
        });
        this.consecutiveMistakeCount++;
    }
    
    // 再帰的に次のリクエストを処理
    const recDidEndLoop = await this.recursivelyMakeClineRequests(this.userMessageContent);
    return recDidEndLoop;
}
```

主な処理内容：
- `loadContext()`を呼び出してユーザーコンテンツと環境情報を準備します
- APIリクエストを送信し、ストリーミングレスポンスを処理します
- レスポンスを解析してアシスタントメッセージとツール使用を検出します
- ツール使用があれば実行し、結果をユーザーに表示します
- ツール実行結果を含めて次のAPIリクエストを再帰的に処理します
- エラー処理、トークン使用量の追跡、会話履歴の管理なども行います
- 連続してエラーが発生した場合、ユーザーに通知して対応を求めます
- チェックポイント機能を使用して、タスクの状態を保存します

### loadContext() - タスクに必要なコンテキスト情報を読み込み

この関数は、AIに提供するコンテキスト情報を準備します。

```typescript
async loadContext(userContent: UserContent, includeFileDetails: boolean = false) {
    return await Promise.all([
        // ユーザーコンテンツ内のメンション（@ファイルパスなど）を解析
        Promise.all(
            userContent.map(async (block) => {
                if (block.type === "text") {
                    if (
                        block.text.includes("<feedback>") ||
                        block.text.includes("<answer>") ||
                        block.text.includes("<task>") ||
                        block.text.includes("<user_message>")
                    ) {
                        return {
                            ...block,
                            text: await parseMentions(block.text, cwd, this.urlContentFetcher),
                        }
                    }
                }
                return block;
            }),
        ),
        // 環境詳細情報の取得
        this.getEnvironmentDetails(includeFileDetails),
    ]);
}
```

主な処理内容：
- ユーザーコンテンツ内のメンション（@ファイルパスなど）を解析して実際のコンテンツに置き換えます
- `getEnvironmentDetails()`を呼び出して環境情報を取得します
  - VSCodeの表示ファイル、開いているタブ
  - アクティブなターミナル情報と出力
  - 現在の時刻とタイムゾーン
  - 現在の作業ディレクトリのファイル構造（includeFileDetails=trueの場合）
  - 現在のモード（PlanモードかActモード）の情報
- ユーザーが生成したコンテンツ（フィードバック、回答、タスク、メッセージ）を特定のタグで囲み、メンション解析の対象とします

### preparePrompt() - システムプロンプトとユーザーメッセージを構築

この関数は直接実装されていませんが、`recursivelyMakeClineRequests()`内で以下のようにプロンプトを構築しています：

```typescript
// システムプロンプトの構築
let systemPrompt = await SYSTEM_PROMPT(cwd, supportsComputerUse, mcpHub, this.browserSettings);

// カスタム指示、言語設定、.clinerules、.clineignoreなどの追加
if (
    settingsCustomInstructions ||
    clineRulesFileInstructions ||
    clineIgnoreInstructions ||
    preferredLanguageInstructions
) {
    systemPrompt += addUserInstructions(
        settingsCustomInstructions,
        clineRulesFileInstructions,
        clineIgnoreInstructions,
        preferredLanguageInstructions,
    );
}

// コンテキストウィンドウの管理（トークン数が多い場合に会話履歴を切り詰める）
if (previousApiReqIndex >= 0) {
    const previousRequest = this.clineMessages[previousApiReqIndex];
    if (previousRequest && previousRequest.text) {
        const { tokensIn, tokensOut, cacheWrites, cacheReads } = JSON.parse(previousRequest.text);
        const totalTokens = (tokensIn || 0) + (tokensOut || 0) + (cacheWrites || 0) + (cacheReads || 0);
        let contextWindow = this.api.getModel().info.contextWindow || 128_000;
        
        // コンテキストウィンドウサイズに基づいて会話履歴を切り詰める
        if (totalTokens >= maxAllowedSize) {
            this.conversationHistoryDeletedRange = getNextTruncationRange(
                this.apiConversationHistory,
                this.conversationHistoryDeletedRange,
                keep,
            );
        }
    }
}

// 切り詰められた会話履歴の取得
const truncatedConversationHistory = getTruncatedMessages(
    this.apiConversationHistory,
    this.conversationHistoryDeletedRange,
);

// APIリクエストの作成と送信
let stream = this.api.createMessage(systemPrompt, truncatedConversationHistory);
```

主な処理内容：
- システムプロンプトの構築（AIの役割、ツール使用ガイドライン、制約条件など）
- カスタム指示、言語設定、.clinerules、.clineignoreなどの追加
- コンテキストウィンドウの管理（トークン数が多い場合に会話履歴を切り詰める）
- モデルの特性に応じたコンテキストウィンドウサイズの調整
- 切り詰められた会話履歴を使用してAPIリクエストを作成
- 異なるAIモデル間での切り替えに対応するためのコンテキスト管理