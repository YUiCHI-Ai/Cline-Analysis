# システムプロンプトとAPIリクエストの例

このドキュメントでは、CoolClineのシステムプロンプトがどのようにAPIリクエストに変換されるかを具体的な例を用いて説明します。

## 1. システムプロンプトの構造

CoolClineのシステムプロンプトは、`cline/src/core/prompts/system.ts`で定義されており、以下のセクションで構成されています：

```
You are CoolCline, a highly skilled software engineer with extensive knowledge in many programming languages, frameworks, design patterns, and best practices.

====

TOOL USE

[ツールの使用方法と形式の説明]

# Tools

[各ツールの詳細な説明]

# Tool Use Guidelines

[ツール使用のガイドラインとベストプラクティス]

====

CAPABILITIES

[CoolClineの能力の説明]

====

RULES

[CoolClineの動作ルールの説明]

====

SYSTEM INFORMATION

[OS、シェル、ホームディレクトリなどの環境情報]

====

OBJECTIVE

[目標と作業方法の説明]
```

## 2. APIリクエストへの変換プロセス

### 2.1 基本的な変換フロー

1. `Cline.ts`の`recursivelyMakeClineRequests`メソッドが呼び出される
2. `SYSTEM_PROMPT`関数がシステムプロンプトを生成
3. ユーザーのカスタム指示があれば`addUserInstructions`関数で追加
4. 環境情報が`getEnvironmentDetails`メソッドで収集され追加
5. 会話履歴が`apiConversationHistory`から取得され追加
6. 最終的なプロンプトがAPIに送信される

### 2.2 具体的な変換例

以下は、ユーザーが「シンプルなTodoアプリを作成して」と依頼した場合の変換例です：

#### 2.2.1 ユーザーの依頼

```
シンプルなTodoアプリを作成して
```

#### 2.2.2 システムプロンプト（一部抜粋）

```
You are CoolCline, a highly skilled software engineer with extensive knowledge in many programming languages, frameworks, design patterns, and best practices.

====

TOOL USE

You have access to a set of tools that are executed upon the user's approval. You can use one tool per message, and will receive the result of that tool use in the user's response. You use tools step-by-step to accomplish a given task, with each tool use informed by the result of the previous tool use.

# Tool Use Formatting

Tool use is formatted using XML-style tags. The tool name is enclosed in opening and closing tags, and each parameter is similarly enclosed within its own set of tags. Here's the structure:

<tool_name>
<parameter1_name>value1</parameter1_name>
<parameter2_name>value2</parameter2_name>
...
</tool_name>

...

# Tools

## read_file
Description: Request to read the contents of a file at the specified path...
...

## write_to_file
Description: Request to write content to a file at the specified path...
...

...

# Tool Use Guidelines

1. In <thinking> tags, assess what information you already have and what information you need to proceed with the task.
2. Choose the most appropriate tool based on the task and the tool descriptions provided...
...
```

#### 2.2.3 環境情報（一部抜粋）

```
# VSCode Visible Files
index.html
style.css
script.js

# VSCode Open Tabs
index.html

# Current Working Directory (/Users/username/projects) Files
index.html
style.css
script.js
README.md
package.json
...

# Current Time
2025/3/10 午後12:15:00 (Asia/Tokyo, UTC+9:00)

# Current Mode
code
```

#### 2.2.4 最終的なAPIリクエスト

APIリクエストは以下の構造になります：

```json
{
  "model": "claude-3-7-sonnet-20250219",
  "system": "[システムプロンプト全文]",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "<task>\nシンプルなTodoアプリを作成して\n</task>"
        }
      ]
    }
  ],
  "max_tokens": 8192,
  "temperature": 0
}
```

## 3. APIレスポンスの処理

### 3.1 レスポンスの例

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "シンプルなTodoアプリを作成します。まずは現在のファイル構成を確認しましょう。\n\n<thinking>\nユーザーからシンプルなTodoアプリの作成依頼を受けました。まず、現在のディレクトリ構造を確認して、既存のファイルがあるかどうかを確認する必要があります。環境情報によると、index.html、style.css、script.jsファイルが既に存在するようです。これらのファイルの内容を確認してから、Todoアプリの実装を進めるべきでしょう。\n</thinking>"
    },
    {
      "type": "tool_use",
      "name": "read_file",
      "params": {
        "path": "index.html"
      }
    }
  ]
}
```

### 3.2 レスポンスの解析

APIからのレスポンスは、`parseAssistantMessage`関数によって詳細に解析されます。この処理は以下のステップで行われます：

#### 3.2.1 レスポンスの構造解析

APIレスポンスは以下のような構造を持っています：

```typescript
interface AssistantResponse {
  role: "assistant";
  content: ContentItem[];
}

type ContentItem =
  | { type: "text"; text: string }
  | { type: "tool_use"; name: string; params: Record<string, any> };
```

解析処理は`cline/src/core/assistant-message/parse.ts`で実装されており、以下のような流れで行われます：

1. **初期検証**：
   - レスポンスが`role: "assistant"`を持つことを確認
   - `content`配列が存在し、少なくとも1つの要素を含むことを確認
   - 無効な構造の場合は早期にエラーを返す

2. **コンテンツタイプの識別**：
   - 各コンテンツアイテムの`type`フィールドを検査
   - サポートされているタイプ（"text"または"tool_use"）かどうかを確認
   - 未知のタイプの場合は警告をログに記録し、可能な限り処理を継続

3. **コンテンツの正規化**：
   - テキストコンテンツの場合、空白の正規化や特殊文字のエスケープ処理を実施
   - ツール使用コンテンツの場合、パラメータの型変換や正規化を実施
   - 実際のコードでは以下のような処理が行われます：

```typescript
function normalizeContent(content: ContentItem[]): NormalizedContent[] {
  return content.map(item => {
    if (item.type === "text") {
      return {
        type: "text",
        text: normalizeText(item.text)
      };
    } else if (item.type === "tool_use") {
      return {
        type: "tool_use",
        name: item.name.trim(),
        params: normalizeParams(item.params)
      };
    }
    // 未知のタイプの場合
    logger.warn(`Unknown content type: ${(item as any).type}`);
    return null;
  }).filter(Boolean) as NormalizedContent[];
}
```

#### 3.2.2 テキストとツール使用の分離

テキストコンテンツとツール使用コンテンツは異なる処理が必要なため、分離して処理されます：

1. **思考プロセスの抽出**：
   - テキストコンテンツから`<thinking>...</thinking>`タグで囲まれた部分を正規表現で抽出
   - 抽出された思考プロセスは内部分析用に保存され、ユーザーには表示されない
   - 実装例：

```typescript
function extractThinking(text: string): { thinking: string | null; visibleText: string } {
  const thinkingRegex = /<thinking>([\s\S]*?)<\/thinking>/g;
  const matches = [...text.matchAll(thinkingRegex)];
  
  if (matches.length === 0) {
    return { thinking: null, visibleText: text };
  }
  
  const thinking = matches.map(match => match[1].trim()).join('\n\n');
  const visibleText = text.replace(thinkingRegex, '').trim();
  
  return { thinking, visibleText };
}
```

2. **マークダウン処理**：
   - 表示用テキストはマークダウンとして解析され、HTML要素に変換
   - コードブロックは構文ハイライト処理が適用される
   - 数式やダイアグラムなどの特殊なマークダウン拡張もサポート

3. **テキストセグメント化**：
   - 長いテキストは適切なセグメントに分割され、段階的に表示
   - セグメント化はセンテンス境界や段落境界を考慮して行われる
   - 実装例：

```typescript
function segmentText(text: string): string[] {
  // 段落で分割
  const paragraphs = text.split(/\n\s*\n/);
  
  // 長い段落をさらに分割
  return paragraphs.flatMap(paragraph => {
    if (paragraph.length < MAX_SEGMENT_LENGTH) {
      return [paragraph];
    }
    
    // 長い段落はセンテンス境界で分割
    return segmentLongParagraph(paragraph);
  });
}
```

#### 3.2.3 ツール使用の構造化

ツール使用コンテンツは、XMLパーサーを使用して詳細に解析されます：

1. **XMLパース処理**：
   - ツール使用は`<tool_name>...</tool_name>`形式のXMLとして表現
   - パラメータも`<param_name>...</param_name>`形式で表現
   - 実際のパース処理は以下のようなステップで行われます：
     1. XMLノードの抽出
     2. ルートノード（ツール名）の識別
     3. 子ノード（パラメータ）の抽出
     4. ネストされたパラメータの再帰的解析

2. **パラメータの型変換**：
   - 文字列として受け取ったパラメータ値を適切な型に変換
   - 数値、真偽値、配列、オブジェクトなどの型をサポート
   - JSON文字列の自動パースも実施
   - 実装例：

```typescript
function convertParameterType(value: string, expectedType: string): any {
  if (expectedType === 'number') {
    const num = Number(value);
    if (isNaN(num)) {
      throw new Error(`Expected number, got: ${value}`);
    }
    return num;
  } else if (expectedType === 'boolean') {
    if (value.toLowerCase() === 'true') return true;
    if (value.toLowerCase() === 'false') return false;
    throw new Error(`Expected boolean, got: ${value}`);
  } else if (expectedType === 'object' || expectedType === 'array') {
    try {
      return JSON.parse(value);
    } catch (e) {
      throw new Error(`Expected JSON ${expectedType}, got invalid JSON: ${value}`);
    }
  }
  // デフォルトは文字列
  return value;
}
```

3. **構造化データへの変換**：
   - パース結果は以下のような構造化データに変換されます：

```typescript
interface ToolUse {
  toolName: string;
  parameters: Record<string, any>;
  rawXml: string; // デバッグ用に元のXMLも保持
}

// 例：read_fileツールの場合
const toolUseExample: ToolUse = {
  toolName: "read_file",
  parameters: {
    path: "index.html"
  },
  rawXml: "<read_file>\n<path>index.html</path>\n</read_file>"
};
```

4. **特殊パラメータの処理**：
   - 複数行テキストやコードブロックなどの特殊なパラメータ形式を処理
   - XMLエスケープ文字（`&lt;`, `&gt;`, `&amp;`など）の適切な変換
   - CData セクションのサポート（`<![CDATA[...]]>`）

#### 3.2.4 バリデーション処理

抽出されたツール使用は、安全性と正確性を確保するために厳密にバリデーションされます：

1. **ツール名の検証**：
   - 抽出されたツール名が登録済みのツールリストに存在するか確認
   - 大文字小文字の違いや類似名を考慮した曖昧さ解消も実施
   - 実装例：

```typescript
function validateToolName(toolName: string): string {
  const registeredTools = getRegisteredTools();
  
  // 完全一致
  if (registeredTools.includes(toolName)) {
    return toolName;
  }
  
  // 大文字小文字を無視した一致
  const lowerToolName = toolName.toLowerCase();
  const match = registeredTools.find(t => t.toLowerCase() === lowerToolName);
  if (match) {
    logger.info(`Tool name case mismatch. Requested: ${toolName}, Using: ${match}`);
    return match;
  }
  
  // 類似ツールの提案
  const suggestions = findSimilarTools(toolName);
  throw new ToolValidationError(
    `Unknown tool: ${toolName}`,
    suggestions.length > 0 ? `Did you mean: ${suggestions.join(', ')}?` : undefined
  );
}
```

2. **必須パラメータの検証**：
   - 各ツールの必須パラメータが存在するかチェック
   - 不足しているパラメータがある場合はエラーを生成
   - ツールのスキーマ定義と照合して検証

3. **パラメータ型の検証**：
   - パラメータ値が期待される型と一致するかチェック
   - 型変換が可能な場合は自動変換を試みる
   - 変換できない場合はエラーを生成

4. **セキュリティ検証**：
   - ファイルパスやコマンドなどの危険なパラメータに対する追加検証
   - ディレクトリトラバーサル攻撃やコマンドインジェクションの防止
   - 許可リストや禁止リストとの照合

5. **エラーメッセージの生成**：
   - バリデーション失敗時の詳細なエラーメッセージを生成
   - ユーザーフレンドリーな修正提案を含める
   - エラーの種類に応じた適切なエラーコードを設定

#### 3.2.5 メッセージの構造化

解析とバリデーションが完了すると、最終的な構造化メッセージが生成されます：

1. **メッセージオブジェクトの構築**：
   - テキスト部分とツール使用部分を含む統合されたメッセージオブジェクトを生成
   - 思考プロセスは内部用プロパティとして保持
   - 実装例：

```typescript
interface ParsedAssistantMessage {
  text: string;
  thinking?: string;
  toolUse?: ToolUse;
  segments: string[];
  hasError: boolean;
  errorMessage?: string;
}

function buildFinalMessage(
  visibleText: string,
  thinking: string | null,
  toolUse: ToolUse | null,
  hasError: boolean = false,
  errorMessage?: string
): ParsedAssistantMessage {
  return {
    text: visibleText,
    ...(thinking ? { thinking } : {}),
    ...(toolUse ? { toolUse } : {}),
    segments: segmentText(visibleText),
    hasError,
    ...(errorMessage ? { errorMessage } : {})
  };
}
```

2. **メタデータの付加**：
   - タイムスタンプ、メッセージID、処理時間などのメタデータを追加
   - デバッグ情報や追跡情報も必要に応じて付加

3. **キャッシュと最適化**：
   - 頻繁に使用されるパース結果をキャッシュして処理を最適化
   - 大きなメッセージの場合は遅延処理や段階的処理を適用

4. **エラー回復メカニズム**：
   - 部分的なパースエラーが発生した場合の回復メカニズム
   - 可能な限り有効な情報を抽出し、エラー部分をスキップ

5. **最終検証**：
   - 構造化されたメッセージの整合性を最終確認
   - 必要なフィールドがすべて存在するか確認
   - 循環参照などの問題がないか確認

このように、APIレスポンスの解析は複数の詳細なステップを経て行われ、テキストとツール使用を適切に構造化し、後続の処理で使用できる形式に変換します。この処理により、CoolClineはAIの応答から正確にツール使用を抽出し、安全に実行することができます。

### 3.3 ツールの実行

ツール使用が検出されると、`presentAssistantMessage`メソッドによって以下の詳細なプロセスが実行されます：

1. **ツール実行の準備**：
   - `cline/src/core/Cline.ts`の`executeToolUse`メソッドがツール実行を担当
   - ツール名に基づいて適切なツール実装を`toolRegistry`から取得
   - ツールのパラメータを検証し、必要に応じて型変換を実施

2. **ユーザー承認プロセス**：
   - 自動承認設定（`AutoApprovalSettings`）をチェック
     - 特定のツールが自動承認リストに含まれているか確認
     - グローバル自動承認設定が有効かどうかを確認
   - 自動承認されない場合は、WebViewUIを通じてユーザーに承認を求める
     - ツール名、パラメータ、実行内容の説明をユーザーに表示
     - ユーザーが承認または拒否するまで待機

3. **ツールの実行メカニズム**：
   - 承認されたツールは`executeToolImplementation`メソッドで実行
   - 各ツールは専用のハンドラ関数を持ち、適切なサービスを呼び出す
     - 例：`read_file`は`FileSystemService`を使用してファイルを読み込む
     - 例：`execute_command`は`TerminalService`を使用してコマンドを実行
   - 実行中はプログレスインジケータを表示し、長時間実行の場合はタイムアウト処理も実施

4. **結果の処理と表示**：
   - ツール実行の結果（成功/失敗、出力データ）を取得
   - 結果をフォーマットし、WebViewUIを通じてユーザーに表示
     - 成功した場合は結果データを表示（ファイル内容、コマンド出力など）
     - 失敗した場合はエラーメッセージと理由を表示
   - 結果データは構造化され、次のAPIリクエストの`messages`配列に追加

5. **状態の更新と次のステップ準備**：
   - 会話履歴（`conversationHistory`）を更新
   - 環境情報（開いているファイル、ディレクトリ構造など）を再取得
   - 次のAPIリクエストのためのコンテキスト情報を準備
   - 必要に応じてVSCode拡張機能の状態を更新（ファイル変更の反映など）

6. **エラーハンドリングとリトライ**：
   - ツール実行中のエラーを捕捉し、適切なエラーメッセージを生成
   - 一時的なエラー（ネットワーク問題など）の場合は自動リトライを実施
   - 致命的なエラーの場合はユーザーに通知し、回復オプションを提示

このプロセスにより、CoolClineはAPIレスポンスからツール使用を正確に抽出し、ユーザーの承認を得た上で安全に実行し、結果を次のAPIリクエストに含めることで連続的な対話を実現しています。

## 4. 再帰的なAPIリクエスト

### 4.1 ツール実行結果を含むAPIリクエスト

```json
{
  "model": "claude-3-7-sonnet-20250219",
  "system": "[システムプロンプト全文]",
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "<task>\nシンプルなTodoアプリを作成して\n</task>"
        }
      ]
    },
    {
      "role": "assistant",
      "content": [
        {
          "type": "text",
          "text": "シンプルなTodoアプリを作成します。まずは現在のファイル構成を確認しましょう。"
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "[read_file] Result:\n<!DOCTYPE html>\n<html lang=\"en\">\n<head>\n    <meta charset=\"UTF-8\">\n    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n    <title>Document</title>\n    <link rel=\"stylesheet\" href=\"style.css\">\n</head>\n<body>\n    \n    <script src=\"script.js\"></script>\n</body>\n</html>"
        },
        {
          "type": "text",
          "text": "<environment_details>\n# VSCode Visible Files\nindex.html\nstyle.css\nscript.js\n\n# VSCode Open Tabs\nindex.html\n\n# Current Working Directory (/Users/username/projects) Files\nindex.html\nstyle.css\nscript.js\nREADME.md\npackage.json\n...\n\n# Current Time\n2025/3/10 午後12:16:00 (Asia/Tokyo, UTC+9:00)\n\n# Current Mode\ncode\n</environment_details>"
        }
      ]
    }
  ],
  "max_tokens": 8192,
  "temperature": 0
}
```

### 4.2 次のレスポンス例

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "次に、style.cssとscript.jsファイルの内容を確認します。"
    },
    {
      "type": "tool_use",
      "name": "read_file",
      "params": {
        "path": "style.css"
      }
    }
  ]
}
```

## 5. 完全なループ例

以下は、Todoアプリ作成の完全なループ例です：

1. ユーザーの依頼を受信
2. システムプロンプトを生成し、APIリクエストを送信
3. レスポンスを受信し、`read_file`ツールの使用を検出
4. ファイルを読み込み、結果をユーザーに表示
5. 結果を次のAPIリクエストに含めて送信
6. 新しいレスポンスを受信し、`write_to_file`ツールの使用を検出
7. ファイルを作成/更新し、結果をユーザーに表示
8. 結果を次のAPIリクエストに含めて送信
9. このプロセスを繰り返し、必要なファイルを作成/更新
10. 最終的に`attempt_completion`ツールを使用してタスク完了を報告

## 6. システムプロンプトとAPIリクエストの変換における重要なポイント

1. **システムプロンプトは静的**: システムプロンプトは基本的に静的で、タスクごとに大きく変わることはありません。
2. **会話履歴は動的**: 会話履歴は各ツール使用の結果に基づいて動的に更新されます。
3. **環境情報は更新される**: 各APIリクエストごとに最新の環境情報が収集され、追加されます。
4. **ツール使用の結果がコンテキストを形成**: ツール使用の結果が次のAPIリクエストのコンテキストとなり、CoolClineの判断に影響します。
5. **再帰的なループ**: タスクが完了するまで、APIリクエスト→レスポンス→ツール使用→結果→APIリクエスト...というループが繰り返されます。

## 7. まとめ

CoolClineのシステムプロンプトからAPIリクエストへの変換プロセスは、静的なシステムプロンプトと動的な会話履歴・環境情報の組み合わせによって実現されています。このプロセスにより、CoolClineはユーザーの依頼に応じて適切なツールを使用し、タスクを効率的に実行することができます。

システムプロンプトはCoolClineの基本的な動作を定義し、APIリクエストはその動作に基づいて具体的なタスクを実行します。ツールの使用により、CoolClineはファイルの読み書き、コマンドの実行、ブラウザの操作などの操作を行い、ユーザーの依頼に応じたコードを作成します。