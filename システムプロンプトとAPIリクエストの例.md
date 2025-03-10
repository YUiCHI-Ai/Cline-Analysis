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

1. `parseAssistantMessage`関数がレスポンスを解析
2. テキストコンテンツとツール使用に分割
3. ツール使用（`read_file`）が検出され、パラメータ（`path: "index.html"`）が抽出

### 3.3 ツールの実行

1. `presentAssistantMessage`メソッドがツール使用を処理
2. ユーザーに承認を求める（または自動承認設定に従う）
3. ツール（`read_file`）を実行し、結果を取得
4. 結果をユーザーに表示し、次のAPIリクエストに含める

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