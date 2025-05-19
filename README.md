# VRChat AIコンパニオン通信システム（VRC Relator） 全体設計仕様書

## システム概要

本システムは、VRChat内でユーザーとAIキャラクターがテキスト会話できるミニマルな仕組みを提供します。関心の分離を徹底し、3つの明確な層に分けて設計しています。

### 目的
最小限の複雑さで、VRChat内でAIコンパニオンとの自然な対話を実現する。

## アーキテクチャ設計

### 全体構成（3層アーキテクチャ）

```
[VRChat層]  [通信ブリッジ層]  [AIサービス層]
```

| 層 | 主な責務 | 技術 | ファイル数 |
|---|---------|-----|----------|
| VRChat層 | ユーザー検知・UI表示 | UdonSharp, OSC | 1ファイル |
| 通信ブリッジ層 | OSC通信・API連携 | Node.js, node-osc | 1ファイル |
| AIサービス層 | 応答生成 | OpenAI API | 外部サービス |

## コンポーネント詳細設計

### 1. VRChat層（AICompanion.cs）

```csharp
// UdonSharpスクリプト
public class AICompanion : UdonSharpBehaviour
{
    [SerializeField] private Text responseText;  // AIの応答表示用テキスト
    [SerializeField] private float detectionRadius = 2.0f;  // 検知半径
    
    private bool isUserNearby = false;
    
    // ユーザー接近検知
    public override void OnPlayerTriggerEnter(VRCPlayerApi player) {
        if (player.isLocal) {
            isUserNearby = true;
            SendCustomNetworkEvent(NetworkEventTarget.Owner, "SendUserNearbyOSC");
        }
    }
    
    // ユーザー離脱検知
    public override void OnPlayerTriggerExit(VRCPlayerApi player) {
        if (player.isLocal) {
            isUserNearby = false;
            SendCustomNetworkEvent(NetworkEventTarget.Owner, "SendUserLeftOSC");
        }
    }
    
    // VRChatのテキスト入力検知（標準機能使用）
    public void OnChatBoxSubmit(string message) {
        SendOSCMessage("/avatar/parameters/userSpeech", message);
    }
    
    // OSC受信（AIからの応答）
    public void OnOSCMessageReceived(string address, string value) {
        if (address == "/avatar/parameters/aiResponse") {
            responseText.text = value;
        }
    }
}
```

### 2. 通信ブリッジ層（bridge.js）

```javascript
// Node.jsブリッジアプリケーション
const osc = require('node-osc');
const axios = require('axios');
require('dotenv').config();

// OSCサーバー（VRChatからの受信用）
const oscServer = new osc.Server(9001, '0.0.0.0');

// OSCクライアント（VRChatへの送信用）
const oscClient = new osc.Client('127.0.0.1', 9000);

// OSCメッセージ処理
oscServer.on('message', async (msg) => {
    const address = msg[0];
    const value = msg[1];
    
    if (address === '/avatar/parameters/userSpeech') {
        // AIへのリクエスト処理
        const aiResponse = await requestAIResponse(value);
        // 応答をVRChatへ送信
        oscClient.send('/avatar/parameters/aiResponse', aiResponse);
    }
});

// OpenAI APIリクエスト
async function requestAIResponse(userMessage) {
    try {
        const response = await axios.post('https://api.openai.com/v1/chat/completions', {
            model: 'gpt-3.5-turbo',
            messages: [
                {role: 'system', content: 'あなたはVRChat内の親しみやすいAIコンパニオンです。簡潔に応答してください。'},
                {role: 'user', content: userMessage}
            ],
            max_tokens: 100
        }, {
            headers: {
                'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`
            }
        });
        
        return response.data.choices[0].message.content;
    } catch (error) {
        console.error('APIエラー:', error);
        return 'すみません、応答できませんでした。';
    }
}

console.log('OSCブリッジ起動中...');
```

### 3. AIサービス層

OpenAI APIを直接利用するため、追加実装は不要。

## データフロー

1. **ユーザー接近時**：
   - ユーザーがAIの近くに来る → VRChat層でトリガー検知 → OSCで「userNearby=true」送信

2. **ユーザー発話時**：
   - ユーザーがテキスト入力 → VRChat層でイベント検知 → OSCで「userSpeech={テキスト}」送信
   - ブリッジ層受信 → OpenAI APIリクエスト → AI応答取得
   - ブリッジ層 → OSCで「aiResponse={応答テキスト}」送信
   - VRChat層受信 → UI更新（テキスト表示）

## インターフェース定義

### OSCパラメータ仕様

| パラメータ名 | 方向 | データ型 | 説明 |
|------------|------|---------|-----|
| userNearby | VRC→ブリッジ | bool | ユーザーが近くにいるか |
| userSpeech | VRC→ブリッジ | string | ユーザーの発話内容 |
| aiResponse | ブリッジ→VRC | string | AIの応答テキスト |

## 実装と運用ガイドライン

1. **セットアップ簡略化**：
   - VRChat: 単一Prefabをドラッグ＆ドロップのみ
   - Node.js: npmコマンド1行でインストール、.envファイルにAPIキーのみ設定

2. **制約条件**：
   - 応答速度：2秒以内を目標
   - テキスト長：VRChat OSC制限内に収める（100トークン程度）

3. **セキュリティ**：
   - APIキーは.envで管理
   - 個人情報はローカルのみで処理

## MVP定義（完成基準）

1. ユーザーがAIキャラクターに近づくと検知される
2. テキストで話しかけるとAIから自然な返答が返る
3. 会話がスムーズに続けられる（2秒以内の応答）

---

**設計思想：** 明確な関心分離、最小限の実装、ユーザー体験の最適化を原則としています。各層は単一ファイルで実装し、拡張性を確保しつつもシンプルさを維持します。
