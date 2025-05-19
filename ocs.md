## VRChat OSC利用における「できること」「できないこと」とサンプルコード（今回の目的向け）

今回の目的は「OpenAI Realtime APIを活用した、VRChat内での低遅延音声対話」です。これを実現するためにOSCをどのように利用するか、その可能性と限界、そして具体的なサンプルコードの方向性を示します。

### VRChat OSCで「できること」（今回の目的に関連）

1.  **外部アプリケーションからVRChatへの入力送信**[1][5]:
    *   **アバターパラメータの制御**: 外部アプリケーション（今回は通信ブリッジ層）からOSCメッセージを送信し、VRChatアバターのパラメータを変更できます。
        *   **例**: AIが発話中であることを示すbool型パラメータをON/OFFする。
    *   **ChatBoxへのテキスト表示**: 特定のOSCアドレス (`/chatbox/input`) を使うことで、外部からVRChatのチャットボックスにテキストを表示できます[5][6]。
        *   **例**: AIの応答音声を文字起こししたものを表示する（今回は音声が主役なので補助的）。
2.  **VRChatから外部アプリケーションへの情報送信**[1][5]:
    *   **アバターパラメータの変更通知**: VRChatアバターのパラメータが変更された際に、その情報をOSCで外部アプリケーションに送信できます。
        *   **例**: ユーザーが特定のジェスチャー（パラメータ変更を伴う）をして会話開始/終了の意思表示をした場合、それを検知する。
    *   **基本的な入力状態の送信**: 移動やジャンプなどの基本的な入力状態を外部に送信できます。
        *   **例**: 今回の目的では直接利用しないが、ユーザーが特定の場所に移動したことをトリガーにするなど応用は可能。

### VRChat OSCで「できないこと」または「困難なこと」（今回の目的に関連）

1.  **UdonSharpからの直接的な高頻度・大容量バイナリデータ送信**:
    *   **マイク音声のリアルタイムストリーミング**: UdonSharp単体でマイク音声を取得し、それをOSC経由でリアルタイムに外部へストリーミングすることは非常に困難です。OSCは元々楽器制御などが主目的であり、VRChatのOSC実装も高帯域な音声ストリーミングを主眼としていません。
    *   **解決策**: マイク音声の取得とOSC送信は、VRChat外部のアプリケーション（例: Node.js、Python、C#のコンソールアプリなど）で行い、通信ブリッジ層へ送る。
2.  **UdonSharpでの直接的なOSCバイナリデータ（音声チャンク）受信と高忠実度再生**:
    *   **OSC経由での音声チャンク受信**: VRChatの標準OSC機能でUdonSharpが直接バイナリデータ（例: Base64エンコードされた音声チャンク）を高頻度に受信し、遅延なくデコードしてAudioSourceで再生するのは技術的なハードルが高いです。
        *   扱えるOSCパラメータ型は主に`int`, `float`, `bool`であり、`string`もChatBox用途が主です[5]。
        *   高頻度（例: 毎秒20回以上）のパラメータ更新は反映されない可能性があります[5]。
    *   **解決策/代替案**:
        *   **外部ライブラリ/ツール**: UdonOSCのようなコミュニティ製アセットを利用して、より柔軟なOSCメッセージ受信を試みる。
        *   **最小限のフィードバック**: MVPでは、AIの音声はPCのデフォルトスピーカーから直接再生し、VRChat内のアバターからのリップシンクや音源定位は諦めるか、非常に簡素なもの（発話中フラグで口パクON/OFF程度）にする。
        *   **テキスト補助**: 音声と同時に、認識されたテキストやAIの応答テキストを`/chatbox/input`経由で表示することで、音声が聞き取りにくい場合の補助とする。
3.  **複雑なセッション管理信号のやり取り**:
    *   WebSocketのようなきめ細かいセッション管理（接続確立、ハートビート、エラーコード詳細通知など）はOSCの標準機能ではありません。
    *   **解決策**: セッション開始/終了のような単純なトリガーは、特定のOSCアドレスとパラメータ（例: `/avatar/parameters/AI_SessionControl` に `int 1`で開始、`0`で終了）で行う。

### 今回の目的に合わせたOSCの具体的な使い方とサンプルコードの方向性

#### 1. VRChat → 通信ブリッジ層（Node.js）

**目的**: ユーザーがAIとの会話を開始/終了する意思を伝える。

**方法**: アバターの特定のジェスチャーやメニュー操作に紐づいたアバターパラメータ（例: `AI_SessionActive`, bool型）を用意する。このパラメータが変更された際にVRChatからOSCで通知する。

**UdonSharp側 (概念)**:
```csharp
// アバターパラメータ "AI_SessionActive" が変化した際にOSCを送信する設定が
// VRChatのOSC機能によって自動的に行われることを期待する。
// Udon側で特定のOSCメッセージを送信するコードは不要。
// ユーザーがメニューで「会話開始」ボタンを押すと、アニメーターコントローラーが
// "AI_SessionActive" パラメータをtrueにする。
```

**Node.js (通信ブリッジ) 側 (受信)**:
```javascript
// oscライブラリは 'node-osc' を想定
import { Server } from 'node-osc';

const oscServer = new Server(9001, '0.0.0.0', () => { // VRChatからの送信を受け取るポート
  console.log('OSC Server listening on port 9001');
});

oscServer.on('message', (msg) => {
  const address = msg[0];
  const args = msg.slice(1);

  if (address === '/avatar/parameters/AI_SessionActive') {
    const isActive = args[0] === 1; // boolはintの0か1で来る
    if (isActive) {
      console.log('AI Session Start requested from VRChat');
      // TODO: OpenAI Realtime APIへのWebSocket接続処理を開始
    } else {
      console.log('AI Session Stop requested from VRChat');
      // TODO: OpenAI Realtime APIへのWebSocket接続を終了
    }
  }
});
```

#### 2. 通信ブリッジ層（Node.js） → VRChat

**目的**: (MVPでは音声はPC直接再生を想定するため限定的) AIが発話中であるかどうかの簡単なフィードバックや、認識されたテキストの表示。

**方法**: 通信ブリッジからVRChatへOSCメッセージを送信する。

**Node.js (通信ブリッジ) 側 (送信)**:
```javascript
// oscライブラリは 'node-osc' を想定
import { Client } from 'node-osc';

const oscClient = new Client('127.0.0.1', 9000); // VRChatの受信ポート

// AIの発話開始/終了を通知する関数 (例)
function sendAISpeakingStatus(isSpeaking) {
  // アバターパラメータ "AI_IsSpeaking" (bool) に送信
  oscClient.send('/avatar/parameters/AI_IsSpeaking', isSpeaking ? 1 : 0, (err) => {
    if (err) console.error(err);
  });
}

// AIの応答テキストをVRChatのチャットボックスに表示する関数 (例)
function sendTextToChatbox(text) {
  // /chatbox/input に [テキスト(string), 即時送信フラグ(bool true), 通知音なしフラグ(bool true)]
  oscClient.send('/chatbox/input', text, true, true, (err) => {
    if (err) console.error(err);
    // console.log('Sent to chatbox:', text);
  });
}

// OpenAI Realtime APIからテキストが得られたと仮定
// sendTextToChatbox("AI: こんにちは！");
// sendAISpeakingStatus(true); // 発話開始
// setTimeout(() => sendAISpeakingStatus(false), 2000); // 2秒後に発話終了
```

**UdonSharp側 (受信とアバターへの反映 - `AI_IsSpeaking` パラメータ)**:
`AI_IsSpeaking` パラメータはアバターのアニメーターコントローラーに直接紐付け、口パクアニメーションなどを制御する。UdonSharpで直接このOSCメッセージを受信して何かをする必要はない場合が多い。

**ChatBoxへの表示**:
`/chatbox/input` へのOSCメッセージはVRChatクライアントが直接処理し、チャットボックスにテキストが表示される。UdonSharpでの特別な処理は不要。

#### 重要な注意点 (再掲)

-   **OSC設定ファイルの更新**: アバターに `AI_SessionActive` や `AI_IsSpeaking` といったパラメータを追加・変更した場合、VRChatが生成するOSC用JSONファイルを一度削除し、アバターを再ロードしてJSONファイルを再生成させる必要がある[5]。
-   **マイク音声とAI応答音声の経路**:
    -   **ユーザーのマイク音声**: VRChat外部アプリ → (OSC) → 通信ブリッジ層 → (WebSocket) → OpenAI Realtime API
    -   **AIの応答音声**: OpenAI Realtime API → (WebSocket) → 通信ブリッジ層 → (PCのスピーカーから直接再生、または非常に限定的なOSCフィードバックをVRChatへ)

この設計では、VRChatのOSC機能は主に「会話の開始/終了トリガー」と「補助的なテキスト表示/状態表示」に限定し、主要な音声ストリーミングはOSCの範囲外で扱うことで、実現可能性を高めています。
