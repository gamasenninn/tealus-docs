# TTS 読み上げ

AIエージェントの回答を音声で読み上げる機能です。音声合成のエンジンは **Aivis Cloud / OpenAI / Gemini** から選べ、サーバー側で合成しない設定（各端末のブラウザ標準音声）もあります（管理者が `TTS_PROVIDER` で選択、[AIエージェント設定](../../admin/agent.md)）。

## 仕組み

```
AIが回答テキストを生成
  → agent-server が音声合成（Aivis / OpenAI / Gemini、部屋ごとに選べる）
  → server に音声を渡す（メモリに 5 分保持）
  → Socket.IO `tts:audio` で、音声の URL をルーム全員に知らせる
  → 各端末がその URL から取得し、<audio> 要素で再生
      （Web Audio API GainNode で音量補正）

TTS_PROVIDER=browser のときは合成せず、Socket.IO `tts:speak` で文を送り、
各端末のブラウザ標準音声（Web Speech API）が読み上げる
```

テキストメッセージの送信と音声合成は並行して処理されるため、チャットへのテキスト表示がブロックされることはありません。

!!! info "配信経路の切替"
    上記は default の経路です。環境変数 `TTS_BROADCAST_MEDIASOUP=true` を設定すると、旧来の **mediasoup PlainTransport 経由**の legacy 経路に切り替わります。詳細は [アーキテクチャ](../../dev/architecture.md) の「TTS 音声合成パイプライン」セクションを参照してください。

!!! warning "Aivis 3000 文字 hard cap（v0.3.0 後、[#300](https://github.com/gamasenninn/tealus/issues/300)）"
    Aivis Cloud API は 1 回の合成リクエストで **3000 文字上限** があります。終礼議事録のような長文を AI agent が読み上げようとして上限超過で失敗するケースが採用者環境で発生していたため、v0.3.0 後で **3000 文字 hard cap** を server 側で実装しました。

    | 状況 | 動作 |
    |---|---|
    | 3000 文字以下 | 全文読み上げ |
    | 3000 文字超過 | 先頭 3000 文字までを読み上げ、以降は **truncation**（chat 上のテキスト表示は完全） |

    AI 回答の TTS 読み上げ用途では実害は稀ですが、長文 voice メッセージの「全文読み上げ」ボタンや bubble button 経由の個人 TTS では truncate skip（[#758fa58](https://github.com/gamasenninn/tealus/commit/758fa58)）が併用される設計です。

### TTS bubble button — 全文読み上げ（v0.3.0 後、`758fa58`）

メッセージ吹き出しの **TTS button** を直接タップして再生する **個人 TTS** では、500 文字 truncate を **skip** して全文を読み上げます。AI 回答の自動読み上げ（500 文字 truncate あり）と区別され、user 側で能動的に読ませたい長文に対応します。

## 使い方

### 1. 読み上げを有効にする

プロフィール設定の「AI回答の読み上げ」をONにします。

### 2. 音声で回答を聞く

AI の回答がチャットに表示されると同時に、**その端末のスピーカーから読み上げられます**。サーバー側で音声合成を設定していればその音声、未設定ならブラウザ標準音声（Web Speech API）です。

!!! note "トランシーバーには接続しません"
    以前このページには「送信するとトランシーバーが自動接続され、30 秒で自動切断される」と書いていましたが、**誤りでした**（このページ冒頭の「Socket.IO で配信」という説明とも食い違っていました）。読み上げはトランシーバーを使いません。

## テキストの前処理

読み上げ前にテキストは以下の前処理が行われます。

| 処理 | 内容 |
|---|---|
| Markdown除去 | 見出し・太字等のMarkdown記法を除去 |
| URL省略 | URLは「URL省略」に置換 |
| コード省略 | コードブロックは「コード省略」に置換 |
| 文字数制限 | 500文字を超える部分は切り詰め |

## 音声モデル

ルームごとに、読み上げの**エンジン**と**声**を選べます。設定はトーク画面右上の **≡** → ルーム設定の「**読み上げエンジン**」「**声**」から行います（v0.10.0、[#444](https://github.com/gamasenninn/tealus/issues/444)）。変えられるのは、**1:1 のトークでは本人、グループでは部屋の管理者**です。管理者はダッシュボード（`/system`）のルーム設定からも同じ設定ができます。

!!! note
    全体の設定 `TTS_PROVIDER` が `browser` か `none` のときは、部屋ごとの選択は効きません（画面にもその旨が出ます）。鍵が未設定のエンジンは選べません。

エンジンは **Aivis Cloud / OpenAI / Gemini** の 3 つで、未設定の部屋は全体の設定（`TTS_PROVIDER`）に従います。下の表は Aivis の声（10 種）です。

- **OpenAI**（13 種）: marin・cedar（おすすめ）、alloy、ash、ballad、coral、echo、fable、nova、onyx、sage、shimmer、verse
- **Gemini**（30 種）: Kore、Zephyr、Puck、Charon、Fenrir、Leda、Orus、Aoede、Callirrhoe、Autonoe、Enceladus、Iapetus、Umbriel、Algieba、Despina、Erinome、Algenib、Rasalgethi、Laomedeia、Achernar、Alnilam、Schedar、Gacrux、Pulcherrima、Achird、Zubenelgenubi、Vindemiatrix、Sadachbia、Sadaltager、Sulafat

### 選択可能なモデル

| モデル名 | 声の特徴 |
|---|---|
| 凛音エル（デフォルト） | 青年女性 |
| まお | 青年女性 |
| にせ | 青年男性 |
| まい | 青年女性 |
| 阿井田 茂 | 中年男性 |
| fumifumi | 成人男性 |
| morioki | 成人女性 |
| ろてじん | 長老ボイス（老年男性） |
| 花音 | 青年女性 |
| コハク | 青年女性 |

!!! tip "声はエージェントの性格の一部"
    音声モデルはプロンプトと同じレイヤで管理されています。エージェントのキャラクター設定に合わせて声を選ぶことで、より自然なコミュニケーションが実現できます。

## 音量設定

TTS 音声の音量は **ユーザー単位の voiceVolume**（0-100%）と **クライアント全体の boost 定数** の組み合わせで決まります（[#198](https://github.com/gamasenninn/tealus/issues/198)）。トランシーバー音声と TTS 音声のバランス調整に活用してください。

### voiceVolume（Profile から設定）

各ユーザーが Profile の音量スライダーで 0-100% を指定します。値は `localStorage.voiceVolume` に保存されるため、ブラウザごとに独立した設定となります（同一ユーザーが別ブラウザで使うと値は共有されません）。トランシーバー / 音声メッセージ / TTS すべての再生にこの倍率が適用されます。

### TTS_VOLUME_BOOST（クライアントの組み込み倍率）

client 側ソース（`client/src/constants/ui.ts`）に定義された定数で、TTS 経路にのみ追加で掛かるブースト値です（現在 `3.0`）。録音音声やトランシーバーと比較して TTS の loudness が低く感じる傾向への補正として用いられます。

!!! warning "環境変数ではありません"
    `TTS_VOLUME_BOOST` は **クライアント側のビルド時定数** です。サーバー側の環境変数でもユーザー設定でもなく、変更には **クライアントの再ビルド**が必要です。

### Web Audio API GainNode による 1.0 超ブースト

ブラウザ標準の `<audio>.volume` プロパティは上限が `1.0` ですが、Tealus は **Web Audio API の GainNode** を介して再生することで `1.0` を超える実効音量を出せるようにしています。これにより `voiceVolume × TTS_VOLUME_BOOST > 1.0` の組み合わせも正しく反映されます。

!!! tip "音量バランス調整の手順"
    まず Profile の **voiceVolume** を調整してみてください。それでもバランスが悪い場合に限り `TTS_VOLUME_BOOST` を上下する、という手順が運用しやすいです。極端に大きい値（例: 4.0 以上）はクリッピング（音割れ）の原因になるため避けてください。

## 長文メッセージの折りたたみ

300文字を超えるメッセージは自動的に折りたたまれ、「もっとみる」ボタンが表示されます。クリックで展開、「閉じる」で再折りたたみできます。これはTTSとは独立した表示機能ですが、AIの長文回答が読みやすくなります。
