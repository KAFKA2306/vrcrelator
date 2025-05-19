## VRChat AIコンパニオン通信システム 詳細設計仕様書 (Realtime API ミニマル版)

### 1. はじめに

#### 1.1. ドキュメントの目的
本ドキュメントは、「VRChat AIコンパニオン通信システム (Realtime API ミニマル版)」の各コンポーネントの内部動作、データ構造、インターフェース、および非機能要件について詳細に規定します。OpenAI Realtime APIを活用し、低遅延の音声対話体験を実現することを目的とします。

#### 1.2. 対象読者
- 本システムの開発担当者
- 本システムのテスト担当者
- 本システムの保守担当者

#### 1.3. 前提知識
- VRChatおよびUdonSharpの基本的な知識
- Node.jsおよびJavaScriptの基本的な知識
- OSC (Open Sound Control) プロトコルの概要
- WebSocketプロトコルの基本的な知識
- OpenAI Realtime APIの概要 [1]

#### 1.4. 用語定義
| 用語                | 説明                                                                      |
| ------------------- | ------------------------------------------------------------------------- |
| VRChat層            | VRChat内で動作し、音声入出力とOSC通信を担当するUdonSharpスクリプト             |
| 通信ブリッジ層      | VRChat層(OSC)とAIサービス層(WebSocket)を中継するNode.jsアプリケーション        |
| AIサービス層        | OpenAI Realtime APIを利用し、音声対話処理を行う外部サービス                     |
| OSC                 | Open Sound Control。本システムではVRChat層と通信ブリッジ層間の音声データ等通信に使用 |
| WebSocket           | 通信ブリッジ層とAIサービス層間の双方向リアルタイム通信に使用                    |
| APIキー             | OpenAI APIを利用するための認証キー                                           |
| `.env`ファイル      | APIキーなどの機密情報を格納する環境変数ファイル                               |
| `userAudioChunk`    | ユーザーの音声データチャンクを示すOSCパラメータ                               |
| `aiAudioChunk`      | AIの応答音声データチャンクを示すOSCパラメータ                                 |
| `sessionControl`    | Realtime APIセッションの開始/終了を指示するOSCパラメータ                    |
| `gpt-4o-realtime-preview` | Realtime APIで使用されるOpenAIモデル [1]                                |

#### 1.5. 参考ドキュメント
- VRChat AIコンパニオン通信システム 全体設計仕様書 (ミニマル版)
- OpenAI Realtime API ドキュメント [1]

---

### 2. システム概要

#### 2.1. システム目的
VRChat内でユーザーがAIキャラクターと低遅延の音声対話を行える、シンプルかつ拡張容易なシステムを最小限の機能で実現します。テキストチャット機能は本ミニマル版では対象外とします。

#### 2.2. システム構成図
```mermaid
graph LR
    A[ユーザーマイク] -- 音声入力 --> B{VRChat層 (UdonSharp)};
    B -- OSC (UDP - 音声チャンク/制御) --> C{通信ブリッジ層 (Node.js)};
    C -- WebSocket (音声ストリーム/制御) --> D[AIサービス層 (OpenAI Realtime API)];
    D -- WebSocket (音声ストリーム/制御) --> C;
    C -- OSC (UDP - 音声チャンク) --> B;
    B -- 音声出力 --> E[ユーザースピーカー];

    subgraph VRChatワールド
        B
    end

    subgraph ローカルPC/サーバー
        C
    end

    subgraph クラウド (OpenAI)
        D
    end
```

---

### 3. 機能要件詳細

#### 3.1. VRChat層 機能詳細
- **ユーザー音声入力**: VRChatのマイク入力から音声データを取得し、適切なサイズのチャンクに分割する。
- **音声チャンク送信**: 分割した音声データチャンクを `userAudioChunk` OSCメッセージとして通信ブリッジ層へ送信する。
- **セッション制御送信**: ユーザーがAIキャラクターに接近/話しかけ始めた際にセッション開始 (`sessionControl="start"`)、離れた/会話終了時にセッション終了 (`sessionControl="stop"`) をOSCで送信する。
- **AI音声チャンク受信・再生**: `aiAudioChunk` OSCメッセージを受信し、音声データチャンクをバッファリングしてVRChat内の `AudioSource` で再生する。

#### 3.2. 通信ブリッジ層 機能詳細
- **OSCメッセージ受信**: VRChat層から `userAudioChunk` および `sessionControl` OSCメッセージを受信する。
- **WebSocketセッション管理**: `sessionControl` に基づき、OpenAI Realtime APIとのWebSocketセッションを確立・維持・終了する。
- **音声データ中継 (VRC→AI)**: 受信した `userAudioChunk` をRealtime APIのWebSocketセッションへストリーミング送信する。
- **音声データ中継 (AI→VRC)**: Realtime APIから受信した音声データをチャンク化し、`aiAudioChunk` OSCメッセージとしてVRChat層へ送信する。
- **APIキー管理**: `.env`ファイルからOpenAI APIキーを安全に読み込む。

#### 3.3. AIサービス層 機能詳細 (OpenAI Realtime API)
- **音声認識 (STT)**: 受信した音声ストリームをリアルタイムでテキストに変換。
- **自然言語理解・応答生成 (NLU/LLM)**: 認識されたテキストと会話コンテキストに基づき、`gpt-4o-realtime-preview` モデルを使用して応答を生成。
- **音声合成 (TTS)**: 生成されたテキスト応答をリアルタイムで音声に変換しストリーミング送信。
- **割り込み処理**: 会話中の割り込みに対応 [1]。
- **関数呼び出しサポート**: (本ミニマル版では直接利用しないが、APIの機能として存在) [1]。

---

### 4. コンポーネント詳細設計

#### 4.1. VRChat層 (例: `AIRealtimeCompanion.cs`)

##### 4.1.1. クラス概要
AIコンパニオンとのリアルタイム音声対話を制御するUdonSharp Behaviour。音声入出力、OSC通信、セッション管理トリガーを担当します。

##### 4.1.2. 主要プロパティ (Inspector設定項目)
| プロパティ名        | 型                  | 説明                                                                 | Inspector公開 |
| ------------------- | ------------------- | -------------------------------------------------------------------- | ------------- |
| `oscSender`         | `VRC.SDK3.Components.VRCOscSender` | OSCメッセージ送信に使用                                                  | Public        |
| `audioSourceForAI`  | `UnityEngine.AudioSource` | AIの応答音声を再生するAudioSource                                        | Public        |
| `detectionRadius`   | `float`             | ユーザー接近を検知する半径 (例: `3.0f`)                                      | Public        |
| `oscIpAddress`      | `string`            | 通信ブリッジ層のIPアドレス (デフォルト: `"127.0.0.1"`)                        | Public        |
| `oscOutPort`        | `int`               | OSC送信ポート (通信ブリッジ層の受信ポート。デフォルト: `9000`)                   | Public        |
| `audioChunkSize`    | `int`               | 音声チャンクのサンプル数 (例: `1024`)                                       | Public        |
| `micSampleRate`     | `int`               | マイクのサンプルレート (例: `16000`Hz、Realtime APIの推奨に合わせる)             | Public        |
| `isUserNearby`      | `bool`              | ユーザーが近くにいるか (内部状態管理用)                                     | Private       |
| `audioBuffer`       | `System.Collections.Generic.Queue` | 受信したAI音声チャンクを格納するバッファ                               | Private       |

*注意: OSC受信はVRChatの標準機能ではOSCパラメータマッピングが主。生のバイト配列や高頻度のチャンク受信はUdonOSC等のカスタムライブラリが必要になる場合がある。本仕様では、OSCパラメータとして文字列にエンコード(Base64等)して送信し、Udonでデコードすることを想定するが、パフォーマンスに注意が必要。より効率的なのは、ブリッジ側で再生可能なオーディオファイル(wavなど)を一時的に生成し、そのURLをOSCで送りVRChat側でロード・再生する方式だが、ミニマル版ではチャンク伝送を試みる。*

##### 4.1.3. 主要メソッドとロジック

-   **`Start()`**:
    -   `audioBuffer = new Queue();`
    -   `audioSourceForAI` の設定 (Loopしない、PlayOnAwakeしないなど)。
    -   マイク入力の準備 (VRChat標準のマイクアクセス権限に依存)。`Microphone.Start(null, true, 1, micSampleRate);` (ただし、VRChat内でのUdonからの直接的なマイク制御は制限が多い。アバターマイクの音声ストリームをUdonでフックできるかが鍵。代替として、ユーザーに外部アプリでマイク音声をOSC送信させる構成も考えられるが、UXは低下する。ここでは理想的なケースとしてUdonがマイクデータにアクセスできる前提で進める。もし不可なら、この部分は大幅な設計変更が必要)。

-   **`Update()` または `FixedUpdate()` (音声処理)**:
    -   マイクから音声データを取得し、`audioChunkSize` 単位で `float[]` として分割。
    -   分割したチャンクをBase64エンコードし、`SendOscAudioChunk(encodedChunk)` を呼び出す。
    -   `audioBuffer` にデータがあれば `PlayBufferedAudio()` を呼び出す。

-   **`OnPlayerTriggerEnter(VRCPlayerApi player)`**:
    -   `player` がローカルプレイヤーの場合、`isUserNearby = true; SendOscSessionControl("start");`

-   **`OnPlayerTriggerExit(VRCPlayerApi player)`**:
    -   `player` がローカルプレイヤーの場合、`isUserNearby = false; SendOscSessionControl("stop");`

-   **`private void SendOscSessionControl(string command)`**:
    -   `oscSender.Send(oscIpAddress, oscOutPort, "/avatar/parameters/sessionControl", command);`

-   **`private void SendOscAudioChunk(string base64EncodedAudioChunk)`**:
    -   `if (isUserNearby) { oscSender.Send(oscIpAddress, oscOutPort, "/avatar/parameters/userAudioChunk", base64EncodedAudioChunk); }`

-   **OSCメッセージ受信処理 (`OnAvatarParameterChanged` またはUdonOSC利用)**:
    -   `/avatar/parameters/aiAudioChunk` (string - Base64エンコードされた音声チャンク) を受信。
    -   Base64デコードし、`float[]` に変換。
    -   `lock (audioBuffer) { audioBuffer.Enqueue(decodedChunk); }`

-   **`private void PlayBufferedAudio()`**:
    -   `if (!audioSourceForAI.isPlaying && audioBuffer.Count > 0) { lock (audioBuffer) { float[] chunkToPlay = audioBuffer.Dequeue(); AudioClip clip = AudioClip.Create("aiVoiceChunk", chunkToPlay.Length, 1, micSampleRate, false); clip.SetData(chunkToPlay, 0); audioSourceForAI.clip = clip; audioSourceForAI.Play(); } }`

##### 4.1.4. OSCメッセージ送受信仕様 (VRChat層観点)

-   **送信メッセージ**:
    -   アドレス: `/avatar/parameters/sessionControl`, 値: `string` (`"start"` or `"stop"`)
    -   アドレス: `/avatar/parameters/userAudioChunk`, 値: `string` (Base64エンコードされた音声チャンク)
-   **受信メッセージ**:
    -   アドレス: `/avatar/parameters/aiAudioChunk`, 値: `string` (Base64エンコードされた音声チャンク)

#### 4.2. 通信ブリッジ層 (`bridge_realtime.js`)

##### 4.2.1. ファイル概要
VRChat (OSC) とOpenAI Realtime API (WebSocket) 間の音声データおよび制御メッセージを中継するNode.jsスクリプト。

##### 4.2.2. 使用ライブラリとバージョン (例)
-   `node-osc`: ^7.0.0
-   `ws`: ^8.10.0 (WebSocketクライアント)
-   `dotenv`: ^16.3.0

##### 4.2.3. 環境変数 (`.env` ファイル)
```
OPENAI_API_KEY="sk-YourOpenAIapiKey"
OSC_LISTEN_PORT=9000
VRC_OSC_IP="127.0.0.1"
VRC_OSC_PORT=9001
OPENAI_REALTIME_API_URL="wss://api.openai.com/v1/realtime/sessions" # 仮のURL、公式ドキュメント参照
SYSTEM_PROMPT_REALTIME="あなたはVRChatのフレンドリーなAIコンパニオンです。自然な声で応答してください。"
VOICE_ID_REALTIME="alloy" # Realtime APIでサポートされる音声ID [1]
AUDIO_FORMAT_REALTIME="pcm_s16le" # Realtime APIが期待する音声フォーマット
SAMPLE_RATE_REALTIME=16000 # Realtime APIが期待するサンプルレート
```

##### 4.2.4. 主要関数とロジック

-   **グローバル変数**: `let wsClient = null; let sessionId = null;`

-   **初期化処理**:
    -   `dotenv.config()`
    -   `osc.Server` を初期化。
    -   `osc.Client` を初期化。
    -   OSCサーバーの `message` イベントリスナーを設定。

-   **`oscServer.on('message', async (msg) => { ... })`**:
    -   `/avatar/parameters/sessionControl`:
        -   `"start"` なら `await startRealtimeSession();`
        -   `"stop"` なら `await stopRealtimeSession();`
    -   `/avatar/parameters/userAudioChunk`:
        -   `if (wsClient && wsClient.readyState === WebSocket.OPEN) { const audioData = Buffer.from(base64EncodedChunk, 'base64'); wsClient.send(audioData); }`

-   **`async function startRealtimeSession()`**:
    -   `if (wsClient) { await stopRealtimeSession(); }`
    -   `wsClient = new WebSocket(process.env.OPENAI_REALTIME_API_URL, { headers: { 'Authorization': \`Bearer ${process.env.OPENAI_API_KEY}\` } });`
    -   `wsClient.on('open', () => { console.log('Realtime API WebSocket connected.'); // セッション開始メッセージを送信 (API仕様による) wsClient.send(JSON.stringify({ action: "start_session", session_config: { system_prompt: process.env.SYSTEM_PROMPT_REALTIME, voice_id: process.env.VOICE_ID_REALTIME, audio_format: process.env.AUDIO_FORMAT_REALTIME, sample_rate: parseInt(process.env.SAMPLE_RATE_REALTIME) } })); });`
    -   `wsClient.on('message', (data) => { // data が音声データ (Buffer) であると仮定 const base64AudioChunk = data.toString('base64'); oscClient.send('/avatar/parameters/aiAudioChunk', base64AudioChunk); });`
    -   `wsClient.on('close', () => { console.log('Realtime API WebSocket disconnected.'); wsClient = null; });`
    -   `wsClient.on('error', (error) => { console.error('Realtime API WebSocket error:', error); if (wsClient) wsClient.close(); wsClient = null; });`

-   **`async function stopRealtimeSession()`**:
    -   `if (wsClient) { wsClient.send(JSON.stringify({ action: "end_session" })); // API仕様による wsClient.close(); wsClient = null; console.log('Realtime API WebSocket session stopped.'); }`

##### 4.2.5. API連携仕様 (OpenAI Realtime API - WebSocket)

-   **接続URL**: `process.env.OPENAI_REALTIME_API_URL` (公式ドキュメントで確認)
-   **認証**: `Authorization: Bearer YOUR_OPENAI_API_KEY` ヘッダー
-   **メッセージ形式**:
    -   クライアント → サーバー:
        -   セッション開始: JSONオブジェクト (設定含む)
        -   音声データ: バイナリフレーム (PCMなど)
        -   セッション終了: JSONオブジェクト
    -   サーバー → クライアント:
        -   音声データ: バイナリフレーム
        -   (その他、テキストトランスクリプトやエラーメッセージ等もAPI仕様によりあり得る)
    *実際のメッセージ形式はOpenAI Realtime APIの公式ドキュメントに厳密に従う必要があります。上記は想定です。*

##### 4.2.6. OSCメッセージ送受信仕様 (ブリッジ層観点)

-   **受信メッセージ**:
    -   アドレス: `/avatar/parameters/sessionControl`, 引数: `string` (`"start"` or `"stop"`)
    -   アドレス: `/avatar/parameters/userAudioChunk`, 引数: `string` (Base64エンコード音声チャンク)
-   **送信メッセージ**:
    -   アドレス: `/avatar/parameters/aiAudioChunk`, 引数: `string` (Base64エンコード音声チャンク)

#### 4.3. AIサービス層 (OpenAI Realtime API)

##### 4.3.1. 利用API
OpenAI Realtime API (WebSocketベース) [1]

##### 4.3.2. 主要パラメータ設定 (ブリッジ層の `.env` またはコード内で設定)
-   `model`: `gpt-4o-realtime-preview` (API側で指定またはデフォルト) [1]
-   `system_prompt`: キャラクター設定、応答スタイル指示。
-   `voice_id`: 応答音声の種類 (例: "alloy", "echo" などAPIがサポートするID) [1]。
-   `audio_format`: 送受信する音声データのフォーマット (例: "pcm_s16le")。
-   `sample_rate`: 音声データのサンプルレート (例: 16000, 24000)。
*これらのパラメータはRealtime APIのセッション開始時に設定すると想定。*

---

### 5. データ構造詳細

#### 5.1. OSCメッセージペイロード詳細
-   `/avatar/parameters/sessionControl` (VRC → ブリッジ):
    -   型: `string`
    -   値: `"start"`, `"stop"`
-   `/avatar/parameters/userAudioChunk` (VRC → ブリッジ):
    -   型: `string`
    -   値: Base64エンコードされた音声データチャンク (例: 1024サンプルのfloat配列をエンコード)
-   `/avatar/parameters/aiAudioChunk` (ブリッジ → VRC):
    -   型: `string`
    -   値: Base64エンコードされた音声データチャンク

#### 5.2. WebSocketメッセージ (OpenAI Realtime API)
OpenAI Realtime APIの公式ドキュメントで定義されるJSONオブジェクトおよびバイナリオーディオフレームに従います。

---

### 6. インターフェース詳細

#### 6.1. VRChat OSCインターフェース
「詳細設計仕様書 (ミニマル版)」と同様、ただし送受信するパラメータが音声関連に変更。

#### 6.2. OpenAI Realtime API (WebSocket) インターフェース
「4.2.5. API連携仕様 (OpenAI Realtime API - WebSocket)」セクションを参照。

---

### 7. 非機能要件詳細

#### 7.1. パフォーマンス
-   **応答遅延目標値**: ユーザーの発話終了からAIの応答音声が聞こえ始めるまで、平均500ms以内、最大1秒以内 (ネットワーク状況およびOpenAI Realtime APIの性能に依存)。
-   **音声品質**: サンプルレート `16000Hz` 以上、PCM 16bitモノラルを基本とする (Realtime APIの仕様に準拠)。

#### 7.2. セキュリティ
「詳細設計仕様書 (ミニマル版)」と同様。WebSocket通信はWSS (Secure WebSocket) を使用。

#### 7.3. 拡張性
-   システムプロンプト、ボイスIDの変更容易性。
-   将来的にRealtime APIがサポートする他モダリティ (視覚など) への拡張可能性を考慮した疎結合設計 [1]。

#### 7.4. 保守性
「詳細設計仕様書 (ミニマル版)」と同様。

---

### 8. エラーハンドリングと例外処理

#### 8.1. VRChat層でのエラー
-   OSC送信失敗。
-   マイクアクセス不可 (VRChatの権限設定やプラットフォーム制限による)。
-   受信音声チャンクのデコード失敗。

#### 8.2. 通信ブリッジ層でのエラー
-   OSC受信エラー。
-   WebSocket接続失敗 (URL不正、認証失敗、ネットワーク問題)。
-   WebSocketセッション中のエラー (APIからのエラーメッセージ受信、予期せぬ切断)。
-   音声データのエンコード/デコードエラー。

#### 8.3. AIサービス層でのエラー
OpenAI Realtime API側で発生するエラー (モデルエラー、過負荷など)。ブリッジ層でWebSocket経由でエラーメッセージを受信し処理。

#### 8.4. ユーザーへのエラー通知方法
本ミニマル版では、VRChat内でユーザーに直接的なエラーフィードバックを行う仕組みは限定的 (例: AIが無言になる、特定のエラー音声再生など)。デバッグはコンソールログに依存。

---

### 9. デプロイと運用
「詳細設計仕様書 (ミニマル版)」の該当セクションに準じつつ、音声関連の設定 (マイク、AudioSource) が追加される。

---

### 10. テスト計画 (概要)
「詳細設計仕様書 (ミニマル版)」の該当セクションに準じつつ、音声対話の品質 (遅延、明瞭度、自然さ、割り込み性能) に関するテスト項目を追加。

---

**本再設計ドキュメントの注意点:**
-   OpenAI Realtime APIは比較的新しいAPIであり、本ドキュメント作成時点での情報に基づいています。APIの仕様、特にWebSocketメッセージの詳細な形式やセッション管理方法は、必ず最新の公式ドキュメントを参照してください [1]。
-   VRChat Udonからのマイク音声ストリームの直接的かつ効率的な取得、およびOSC経由での高頻度な音声チャンク送受信は技術的課題を含む可能性があります。パフォーマンスと実現可能性を慎重に検証する必要があります。Base64エンコード/デコードはCPU負荷が高いため、代替手段 (圧縮、より効率的なバイナリOSC伝送ライブラリなど) の検討が必要になる場合があります。

Citations:
[1] https://openai.com/index/introducing-the-realtime-api/

---
Perplexity の Eliot より: pplx.ai/share
