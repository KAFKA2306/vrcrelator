## VRChat AIコンパニオン通信システム コード全体設計（OpenAI Realtime APIミニマル対応）

---

### 1. 構成概要

- **VRChat層（UdonSharp）**  
  ユーザー音声入力をOSC経由で通信ブリッジへ送信し、AI応答音声をOSC経由で受信・再生
- **通信ブリッジ層（Node.js）**  
  VRChat層とOpenAI Realtime API（WebSocket）間で音声データを中継
- **AIサービス層（OpenAI Realtime API）**  
  音声ストリームでAI対話を実現

---

### 2. VRChat層（UdonSharpスクリプト設計）

#### 2.1 必要要素
- マイク音声取得（UdonSharpで直接は不可、外部アプリ連携を想定）
- OSC送信：ユーザー音声チャンク（Base64文字列）
- OSC受信：AI音声チャンク（Base64文字列）→ AudioSource再生
- セッション開始/終了OSC送信

#### 2.2 クラス構成例（擬似コード）

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

public class AIRealtimeCompanion : UdonSharpBehaviour
{
    public AudioSource aiAudioSource;
    public string oscIpAddress = "127.0.0.1";
    public int oscOutPort = 9000;
    public int oscInPort = 9001;
    public float detectionRadius = 3.0f;
    private bool isUserNearby = false;

    // ユーザー接近/離脱検知
    public override void OnPlayerTriggerEnter(VRCPlayerApi player)
    {
        if (player.isLocal)
        {
            isUserNearby = true;
            SendOscSessionControl("start");
        }
    }
    public override void OnPlayerTriggerExit(VRCPlayerApi player)
    {
        if (player.isLocal)
        {
            isUserNearby = false;
            SendOscSessionControl("stop");
        }
    }

    // セッション制御OSC送信
    private void SendOscSessionControl(string command)
    {
        // OSC送信処理（外部OSC送信コンポーネントを利用）
        // 例: oscSender.Send(oscIpAddress, oscOutPort, "/avatar/parameters/sessionControl", command);
    }

    // 音声チャンク送信（外部アプリから呼び出し）
    public void SendUserAudioChunk(string base64Audio)
    {
        if (isUserNearby)
        {
            // 例: oscSender.Send(oscIpAddress, oscOutPort, "/avatar/parameters/userAudioChunk", base64Audio);
        }
    }

    // AI音声チャンク受信（OSC→UdonSharp連携が必要。UdonOSC等を利用）
    public void OnAIAudioChunkReceived(string base64Audio)
    {
        // Base64デコード→float[]変換→AudioClip生成→aiAudioSource.PlayOneShot
    }
}
```
> ※UdonSharp単体ではマイク音声の直接取得やOSCバイナリ受信は困難なため、外部アプリ（例：Unity外部のC#やNode.js）でマイク音声を取得・OSC送信する構成が現実的です[1][5]。

---

### 3. 通信ブリッジ層（Node.js設計）

#### 3.1 必要要素
- OSCサーバー（VRChatから音声チャンク/セッション制御受信）
- WebSocketクライアント（OpenAI Realtime API接続）
- 音声データのBase64デコード/エンコード
- AI音声チャンクをOSCでVRChatに送信

#### 3.2 構成例（主要部抜粋）

```javascript
import osc from "node-osc";
import WebSocket from "ws";
import dotenv from "dotenv";
dotenv.config();

const oscServer = new osc.Server(process.env.OSC_LISTEN_PORT, "0.0.0.0");
const oscClient = new osc.Client(process.env.VRC_OSC_IP, process.env.VRC_OSC_PORT);

let ws = null;

function connectRealtimeAPI() {
    const url = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-10-01";
    ws = new WebSocket(url, {
        headers: {
            "Authorization": "Bearer " + process.env.OPENAI_API_KEY,
            "OpenAI-Beta": "realtime=v1",
        }
    });
    ws.on("open", () => {
        console.log("Connected to OpenAI Realtime API");
        // 必要に応じてセッション初期設定送信
    });
    ws.on("message", (data) => {
        // AI音声バイナリ→Base64変換→OSCでVRChatへ
        const base64Audio = Buffer.from(data).toString("base64");
        oscClient.send("/avatar/parameters/aiAudioChunk", base64Audio);
    });
    ws.on("close", () => { ws = null; });
}

oscServer.on("message", (msg) => {
    const [address, ...args] = msg;
    if (address === "/avatar/parameters/sessionControl") {
        if (args[0] === "start") connectRealtimeAPI();
        if (args[0] === "stop" && ws) ws.close();
    }
    if (address === "/avatar/parameters/userAudioChunk" && ws && ws.readyState === WebSocket.OPEN) {
        const audioBuffer = Buffer.from(args[0], "base64");
        ws.send(audioBuffer);
    }
});
```
> ※WebSocket接続・認証ヘッダー・モデル指定などは[2][3]参照。  
> ※音声データの形式・チャンクサイズ・エンコード方法はOpenAI Realtime APIの仕様に従う必要があります。

---

### 4. AIサービス層（OpenAI Realtime API）

- WebSocketサーバーとして動作
- モデル指定例:  
  `wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-10-01`  
- 認証ヘッダー:  
  `"Authorization": "Bearer " + API_KEY`  
  `"OpenAI-Beta": "realtime=v1"`  
- 音声ストリーム送受信、会話状態維持  
- 詳細は公式ドキュメント[2][3]を参照

---

### 5. 外部アプリ（マイク音声→OSC送信用）

- Unity外部でマイク音声を取得し、チャンク化してOSCで `/avatar/parameters/userAudioChunk` へ送信
- 例：PythonやNode.jsで実装可能

---

### 6. フローまとめ

1. ユーザーがAIキャラクターに近づく→OSCで`sessionControl:start`送信
2. マイク音声をチャンク化しOSCで`userAudioChunk`送信
3. 通信ブリッジがOpenAI Realtime APIへ音声ストリーム送信
4. AI応答音声がWebSocketで返却→Base64化しOSCで`aiAudioChunk`送信
5. VRChat層が受信しAudioSourceで再生
6. 離脱時は`sessionControl:stop`でWebSocket切断

---

**備考：**
- マイク音声のOSC送信はVRChat/UdonSharp単体では困難なため、外部アプリの併用が現実的です
- VRChat側のOSC受信/再生はUdonOSC等の拡張が必要な場合があります
- 各パラメータ名やデータ形式はプロトタイピング時に調整してください

---

**参考:**
- [2] UnityでRealtime APIを使う例（WebSocket接続・認証方法等）
- [3] Node.jsでのWebSocketクライアント例
- [5] VRChat Udonでの外部通信・OSC活用例
