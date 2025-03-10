# システムプロンプトからAPIリクエストへの変換フロー

このドキュメントでは、CoolClineのシステムプロンプトがAPIリクエストに変換される基本的なフローをUML図（Mermaid記法）で説明します。

## 1. 全体的なフロー図

```mermaid
flowchart TD
    A[ユーザー入力受信] --> B[システムプロンプト生成]
    B --> C[ユーザーカスタム指示追加]
    C --> D[環境情報収集]
    D --> E[会話履歴取得]
    E --> F[APIリクエスト構築]
    F --> G[APIリクエスト送信]
    G --> H[APIレスポンス受信]
    H --> I{ツール使用検出?}
    I -->|はい| J[ツール実行]
    J --> K[結果を次のリクエストに含める]
    K --> F
    I -->|いいえ| L[応答をユーザーに表示]
    L --> M{タスク完了?}
    M -->|いいえ| F
    M -->|はい| N[処理終了]
```

## 2. クラス図

```mermaid
classDiagram
    class Cline {
        -config: ClineConfig
        -apiConversationHistory: ApiConversationHistory
        -isProcessing: boolean
        -currentConversationId: string
        +recursivelyMakeClineRequests(userMessage: string, options: RecursiveRequestOptions): Promise~void~
        -buildInitialApiRequest(systemPrompt: string, userMessage: string, environmentDetails: any, conversationHistory: ApiMessage[], options: any): ApiRequest
        -executeRecursiveApiRequests(apiRequest: ApiRequest): Promise~void~
        -handleApiError(error: any): void
        -getEnvironmentDetails(): Promise~EnvironmentDetails~
        -executeToolUse(toolUse: ToolUse): Promise~ToolResult~
    }

    class ApiConversationHistory {
        -history: ApiMessage[]
        -maxHistoryLength: number
        +getHistory(): ApiMessage[]
        +addMessage(message: ApiMessage): void
        +clearHistory(): void
    }

    class ApiService {
        +sendApiRequest(request: ApiRequest): Promise~ApiResponse~
        -preProcessRequest(request: ApiRequest): void
        -selectProvider(model: string): ApiProvider
        -postProcessResponse(response: ApiResponse): ApiResponse
    }

    class ToolRegistry {
        -tools: Map~string, ToolImplementation~
        +registerTool(name: string, implementation: ToolImplementation): void
        +getTool(name: string): ToolImplementation
        +hasToolWithName(name: string): boolean
        +getAllToolNames(): string[]
    }

    Cline --> ApiConversationHistory: 使用
    Cline --> ApiService: 使用
    Cline --> ToolRegistry: 使用
```

## 3. シーケンス図

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant Cline as CoolCline
    participant SystemPrompt as SYSTEM_PROMPT関数
    participant UserInst as addUserInstructions関数
    participant EnvDetails as getEnvironmentDetails関数
    participant ConvHistory as apiConversationHistory
    participant ApiReq as APIリクエスト構築
    participant ApiService as ApiService
    participant ToolExec as ツール実行

    User->>Cline: ユーザー入力
    activate Cline
    Cline->>SystemPrompt: システムプロンプト生成
    activate SystemPrompt
    SystemPrompt-->>Cline: 基本システムプロンプト
    deactivate SystemPrompt
    
    Cline->>UserInst: ユーザーカスタム指示追加
    activate UserInst
    UserInst-->>Cline: 拡張システムプロンプト
    deactivate UserInst
    
    Cline->>EnvDetails: 環境情報収集
    activate EnvDetails
    EnvDetails-->>Cline: 環境情報
    deactivate EnvDetails
    
    Cline->>ConvHistory: 会話履歴取得
    activate ConvHistory
    ConvHistory-->>Cline: 会話履歴
    deactivate ConvHistory
    
    Cline->>ApiReq: APIリクエスト構築
    activate ApiReq
    ApiReq-->>Cline: 構築されたAPIリクエスト
    deactivate ApiReq
    
    Cline->>ApiService: APIリクエスト送信
    activate ApiService
    ApiService-->>Cline: APIレスポンス
    deactivate ApiService
    
    alt ツール使用検出
        Cline->>ToolExec: ツール実行
        activate ToolExec
        ToolExec-->>Cline: ツール実行結果
        deactivate ToolExec
        
        Cline->>ConvHistory: 結果を会話履歴に追加
        activate ConvHistory
        ConvHistory-->>Cline: 更新された会話履歴
        deactivate ConvHistory
        
        Cline->>ApiReq: 次のAPIリクエスト構築
        activate ApiReq
        ApiReq-->>Cline: 構築されたAPIリクエスト
        deactivate ApiReq
        
        Cline->>ApiService: 次のAPIリクエスト送信
        activate ApiService
        ApiService-->>Cline: 次のAPIレスポンス
        deactivate ApiService
    else タスク完了
        Cline-->>User: 最終応答
    end
    
    deactivate Cline
```

## 4. APIリクエスト構造

```mermaid
classDiagram
    class ApiRequest {
        +model: string
        +system: string
        +messages: ApiMessage[]
        +max_tokens: number
        +temperature: number
    }
    
    class ApiMessage {
        +role: string
        +content: ApiContent[]
    }
    
    class ApiContent {
        <<interface>>
    }
    
    class TextContent {
        +type: "text"
        +text: string
    }
    
    class ToolUseContent {
        +type: "tool_use"
        +name: string
        +params: Record~string, any~
    }
    
    ApiRequest --> ApiMessage: 含む
    ApiMessage --> ApiContent: 含む
    ApiContent <|-- TextContent: 実装
    ApiContent <|-- ToolUseContent: 実装
```

## 5. システムプロンプト生成プロセス

```mermaid
flowchart LR
    A[SYSTEM_PROMPT関数] --> B[基本的な自己紹介部分]
    A --> C[ツール使用セクション]
    A --> D[能力セクション]
    A --> E[ルールセクション]
    A --> F[システム情報セクション]
    A --> G[目標セクション]
    
    C --> C1[generateToolUseSection]
    D --> D1[generateCapabilitiesSection]
    E --> E1[generateRulesSection]
    F --> F1[generateSystemInfoSection]
    G --> G1[generateObjectiveSection]
    
    B & C1 & D1 & E1 & F1 & G1 --> H[セクションを結合]
    H --> I[完成したシステムプロンプト]
```

## 6. 再帰的なAPIリクエスト処理

```mermaid
stateDiagram-v2
    [*] --> 初期化
    初期化 --> システムプロンプト生成
    システムプロンプト生成 --> ユーザーカスタム指示追加
    ユーザーカスタム指示追加 --> 環境情報収集
    環境情報収集 --> 会話履歴取得
    会話履歴取得 --> APIリクエスト構築
    APIリクエスト構築 --> APIリクエスト送信
    APIリクエスト送信 --> APIレスポンス解析
    
    APIレスポンス解析 --> ツール使用検出
    ツール使用検出 --> ツール実行: ツールが検出された
    ツール実行 --> 結果処理
    結果処理 --> 会話履歴更新
    会話履歴更新 --> 環境情報更新
    環境情報更新 --> APIリクエスト構築: タスク未完了
    
    ツール使用検出 --> タスク完了確認: ツールが検出されない
    タスク完了確認 --> APIリクエスト構築: タスク未完了
    タスク完了確認 --> 処理終了: タスク完了
    
    処理終了 --> [*]
```

## 7. ツール実行プロセス

```mermaid
flowchart TD
    A[ツール使用検出] --> B{自動承認設定?}
    B -->|自動承認| D[ツール実行]
    B -->|承認必要| C[ユーザーに承認要求]
    C --> C1{ユーザー承認?}
    C1 -->|承認| D
    C1 -->|拒否| E[ツール実行キャンセル]
    D --> F[結果取得]
    F --> G[結果フォーマット]
    G --> H[ユーザーに結果表示]
    H --> I[会話履歴更新]
    I --> J[環境情報更新]
    E --> K[エラーメッセージ生成]
    K --> H
```

## 8. APIレスポンス解析プロセス

```mermaid
flowchart TD
    A[APIレスポンス受信] --> B[レスポンス構造検証]
    B --> C[コンテンツタイプ識別]
    C --> D[テキストとツール使用の分離]
    D --> E[思考プロセス抽出]
    E --> F[マークダウン処理]
    F --> G[テキストセグメント化]
    D --> H[ツール使用のXMLパース]
    H --> I[パラメータの型変換]
    I --> J[構造化データへの変換]
    J --> K[ツール名の検証]
    K --> L[必須パラメータの検証]
    L --> M[パラメータ型の検証]
    M --> N[セキュリティ検証]
    G & N --> O[メッセージオブジェクト構築]
    O --> P[メタデータ付加]
    P --> Q[最終検証]
    Q --> R[解析完了]
```

## 9. 完全なループ例（Todoアプリ作成）

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant Cline as CoolCline
    participant API as AIプロバイダーAPI
    participant FS as ファイルシステム
    
    User->>Cline: "シンプルなTodoアプリを作成して"
    activate Cline
    
    Cline->>API: 初期APIリクエスト送信
    activate API
    API-->>Cline: レスポンス（read_fileツール使用）
    deactivate API
    
    Cline->>User: ツール使用の承認要求
    User->>Cline: 承認
    
    Cline->>FS: index.htmlファイル読み込み
    activate FS
    FS-->>Cline: ファイル内容
    deactivate FS
    
    Cline->>User: ファイル内容表示
    
    Cline->>API: 次のAPIリクエスト送信（ファイル内容含む）
    activate API
    API-->>Cline: レスポンス（write_to_fileツール使用）
    deactivate API
    
    Cline->>User: ツール使用の承認要求
    User->>Cline: 承認
    
    Cline->>FS: index.htmlファイル更新
    activate FS
    FS-->>Cline: 成功
    deactivate FS
    
    Cline->>User: 更新成功を表示
    
    Cline->>API: 次のAPIリクエスト送信（更新結果含む）
    activate API
    API-->>Cline: レスポンス（write_to_fileツール使用）
    deactivate API
    
    Cline->>User: ツール使用の承認要求
    User->>Cline: 承認
    
    Cline->>FS: style.cssファイル作成
    activate FS
    FS-->>Cline: 成功
    deactivate FS
    
    Cline->>User: 作成成功を表示
    
    Cline->>API: 次のAPIリクエスト送信（作成結果含む）
    activate API
    API-->>Cline: レスポンス（write_to_fileツール使用）
    deactivate API
    
    Cline->>User: ツール使用の承認要求
    User->>Cline: 承認
    
    Cline->>FS: script.jsファイル作成
    activate FS
    FS-->>Cline: 成功
    deactivate FS
    
    Cline->>User: 作成成功を表示
    
    Cline->>API: 次のAPIリクエスト送信（作成結果含む）
    activate API
    API-->>Cline: レスポンス（attempt_completionツール使用）
    deactivate API
    
    Cline->>User: タスク完了報告
    
    deactivate Cline
```

## 10. まとめ

CoolClineのシステムプロンプトからAPIリクエストへの変換プロセスは、複数のステップから構成される複雑なフローです。このプロセスは、静的なシステムプロンプトと動的な会話履歴・環境情報を組み合わせ、ユーザーの依頼に応じて適切なツールを使用し、タスクを効率的に実行する仕組みを提供しています。

上記のUML図は、このプロセスの主要なコンポーネントと相互作用を視覚的に表現したものです。システムプロンプトの生成から始まり、APIリクエストの構築、送信、レスポンスの解析、ツールの実行、そして次のリクエストへのフィードバックという一連のサイクルが、タスクが完了するまで繰り返されます。