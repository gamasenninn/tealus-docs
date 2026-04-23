# AI アーキテクチャ

Tealus の AI 統合の全体像を開発者向けに解説します。3層エージェント構造、Webhook ディスパッチ、TTS 音声合成パイプラインの内部実装を説明します。

!!! info
    運用・設定については [AIエージェント設定](../admin/agent.md)、MCP ツールの詳細は [MCP Server](mcp.md) を参照してください。

## 全体構成

```
ユーザーがメッセージ送信
    ↓
Tealus Server (port 3000)
    ↓ Webhook (message.created)
Agent Server (port 4000)
    ↓
┌─────────────────────────────────────────┐
│           Webhook Handler               │
│                                         │
│  botUserIds Set ← Bot? → スキップ       │
│         ↓ (人間のメッセージのみ)         │
│                                         │
│       Webhook Dispatcher                │
│  roomQueues Map ← ルーム別キュー         │
│         ↓                               │
│       応答モード判定                     │
│  (auto / mention / all / off)           │
│         ↓                               │
│        Router                           │
│  ┌─────────────────────────────┐        │
│  │ 第1段: ルールベース判定     │        │
│  │  /deep → Deep              │        │
│  │  /light → Light            │        │
│  │  挨拶パターン → 即応答      │        │
│  │  Deep キーワード → Deep     │        │
│  │  該当なし ↓                 │        │
│  │ 第2段: LLM 判定            │        │
│  │  GPT系 → light or deep     │        │
│  └──────┬──────────┬──────────┘        │
│         ↓          ↓                    │
│    Light Agent   Deep Agent             │
│   (GPT-4o-mini)  (Claude CLI)           │
│         ↓          ↓                    │
│      Bot API で回答送信                  │
│         ↓                               │
│    TTS 読み上げ (fire-and-forget)        │
└─────────────────────────────────────────┘
```

## Router — 2段階振り分け

ユーザーのメッセージを適切なエージェントに振り分けます。

### 第1段: ルールベース判定

高速で確実な判定をルールで行います。

| パターン | 振り分け先 |
|---|---|
| `/deep` コマンド | Deep Agent |
| `/light` コマンド | Light Agent |
| 挨拶パターン（こんにちは、ありがとう等） | Router が直接応答 |
| Deep キーワード（コード、リファクタ、デバッグ、実装、PR等） | Deep Agent |
| 該当なし | 第2段へ |

### 第2段: LLM 判定

第1段で判定できなかったメッセージを LLM で分類します。

- **モデル**: GPT系（GPT-4o-mini）
- **パラメータ**: `max_tokens: 10`, `temperature: 0`（決定論的に判定）
- **出力**: `light` または `deep` の1単語
- **フォールバック**: エラー時は Light Agent に振る

!!! tip "なぜ2段階か"
    ルールベース判定はコストゼロ・遅延ゼロで処理できます。LLM を呼ぶのは判定が難しいケースだけに限定することで、Router のコストを月 $3〜$13 に抑えています。

## Light Agent — OpenAI Agents SDK

日常的な質問やタスクを高速に処理する軽量エージェントです。

### セッション管理（TealusSession）

`TealusSession` クラスが Tealus の会話履歴を OpenAI Agents SDK の形式に変換します。

- Bot API からチャット履歴を取得
- 直近 **20件**（`LIGHT_CONTEXT_MESSAGES`）のメッセージをコンテキストとして使用
- 初回は DB から取得、以降はキャッシュに蓄積

### システムプロンプト

複数のプロンプトソースが階層的に合成されます。

| ソース | ファイル | 優先度 |
|---|---|---|
| デフォルト | `config/default_system_prompt.md` | 基本 |
| カスタム | `config/system_prompt.md` | グローバル上書き |
| ルーム固有 | `{workspace}/light_prompt.md` | ルーム個別の指示 |
| メモリ | `{workspace}/memory/MEMORY.md` | エージェントの記憶 |

### ツール

| ツール名 | 説明 |
|---|---|
| `write_memory` | エージェントのメモリに情報を書き込み |
| `read_memory` | 保存済みメモリの読み取り |
| `get_current_time` | 現在時刻の取得 |
| `list_workspace_files` | ワークスペース内のファイル一覧 |
| `generate_image` | DALL-E による画像生成（設定でON/OFF可） |
| `code_interpreter` | Python コード実行（設定でON/OFF可） |

ツール実行時には UI にステータスが自動通知されます（例: 「検索中...」「画像生成中...」）。

## Deep Agent — Claude Code CLI

複雑なコード生成、分析、長いワークフローを処理する高度なエージェントです。

### 起動方式

```bash
claude -p - \
  --dangerously-skip-permissions \
  --mcp-config <動的設定ファイル> \
  [--resume <sessionId>]
```

- プロンプトは **stdin** から送信
- `--resume tealus-room-{roomId}` で前のセッションから再開可能

### 動的 MCP 設定生成

実行時に MCP 設定 JSON を動的に組み立てます。

1. Tealus MCP Server（チャット読み書き用）を基本設定として追加
2. ルーム固有の MCP 設定（`room_settings.json` の `mcp` セクション）をマージ
3. `.deep_mcp_config.json` としてワークスペースに書き出し

これにより、Deep Agent は [MCP Server](mcp.md) 経由でチャット履歴の読み取り・メッセージ送信を自律的に行えます。

### 制約

| 項目 | 値 |
|---|---|
| タイムアウト | 5分（`DEEP_TIMEOUT`） |
| 出力分割 | 4000字単位でチャットに送信 |
| プロセス終了 | SIGTERM（Windows: `taskkill /T /F`） |

## Webhook Dispatcher

Agent Server がメッセージを受け取り、エージェントに渡すまでの制御を担います。

### ルーム別キュー

同一ルームへの並行応答を防止するため、`roomQueues`（Map）でルームごとにメッセージをシリアライズします。

```
ルーム A: メッセージ1 → メッセージ2 → メッセージ3（順次処理）
ルーム B: メッセージ1 → メッセージ2（並行して処理）
```

異なるルームのメッセージは並行処理されますが、同じルーム内のメッセージは前のメッセージの処理が完了してから次に進みます。

### 無限ループ防止

`botUserIds`（Set）に Bot ユーザー ID を保持し、Webhook 受信時に送信者をチェックします。Bot からのメッセージは即座にスキップされ、Bot 同士の無限応答ループを防止します。

### 対応イベント

| イベント | 処理 |
|---|---|
| `message.created` | Router 経由でエージェントに振り分け |
| `voice.transcription_completed` | 音声文字起こし完了時にエージェント処理 |
| `member.joined` / `member.left` | Bot のルーム参加状態を追跡 |

## TTS 音声合成パイプライン

AI の回答をリアルタイムで音声に変換し、トランシーバーに送信します。

!!! info
    PlainTransport・ffmpeg の技術詳細は [アーキテクチャ](architecture.md) の「TTS 音声合成パイプライン（PlainTransport）」セクションを参照してください。

### FIFO キュー管理

`isProcessing` フラグによる排他制御で、同時読み上げを防止します。

```
speakMessage() → queue に追加
                    ↓
              processQueue()
              isProcessing = true
                    ↓
              テキスト前処理
                    ↓
              Aivis Cloud API (0.3-0.5秒)
                    ↓
              ffmpeg → RTP 送信
                    ↓
              isProcessing = false
              → キューに次があれば再度 processQueue()
```

複数のメッセージが短時間に発生しても、キューに積まれた順序で逐次処理されます。

### テキスト前処理

読み上げに適したテキストに変換します。

| 処理 | 内容 |
|---|---|
| Markdown除去 | 見出し・太字・リンク等の記法を除去 |
| URL省略 | URL → 「URL省略」に置換 |
| コード省略 | コードブロック → 「コード省略」に置換 |
| 改行統一 | 連続改行を1行に統一 |
| 文字数制限 | 500文字（`TTS_MAX_LENGTH`）を超える部分をカット |

### ルーム別音声モデル

`room_settings.json` の `tts_model_uuid` フィールドでルームごとに異なる音声モデルを設定できます。未設定の場合は環境変数 `AIVIS_MODEL_UUID` のデフォルトモデル（凛音エル）が使用されます。

## 設計思想

### コスト最適化 — なぜ3層か

すべてのメッセージを Deep Agent（Claude）で処理すると高コストになります。3層に分けることで、大半のメッセージを低コストな Light Agent で処理し、複雑なタスクだけを Deep Agent に回します。

| 層 | 月額目安 | 処理割合 |
|---|---|---|
| Router | $3〜$13 | 全メッセージを判定 |
| Light Agent | $80〜$280 | 大半のメッセージ |
| Deep Agent | $100（MAXプラン固定） | 複雑なタスクのみ |
| **合計** | **$183〜$393** | |

!!! warning
    Deep Agent のコストは MAXプランの固定料金が前提です。API 課金に切り替える場合はコストが大幅に変動します。

### 声は人格の一部

Tealus ではプロンプト（性格・口調）と音声モデル（声）を `room_settings.json` で統合管理します。

- ルームごとに異なるエージェントの人格を定義
- 「丁寧な敬語で話すアシスタント」には落ち着いた声を
- 「フレンドリーなチームメイト」には明るい声を

声とプロンプトを同じレイヤで管理することで、エージェントのキャラクター設定が一貫します。

### AI 間協働

Tealus では AI 同士が GitHub Issue で作業指示を出し合う運用が実現しています。

- 開発 AI（Claude）がドキュメント AI に GitHub Issue でレビュー結果を記載
- ドキュメント AI が Issue を見て更新を実施
- 開発 AI がレビューして承認

複数の AI（GPT-4o-mini + Claude）が同じチャットルームで人間と対等に議論し、設計提案・実装・レビューを分担しています。
