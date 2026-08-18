# VRC Relator

VRChatと外部AIサービスを接続する方法を検討するための設計メモです。

## 現在の状態

**このrepositoryには、動作確認済みのVRChat/UdonSharp実装、OSC bridge、AI API client、Prefab、package manifest、test、CIはありません。** 現在の正準treeはREADMEとdocsだけです。そのため、ここに書かれた会話経路を「実装済み」「2秒以内で動作する」「Prefabを置くだけで使える」とは扱いません。

現時点で確認できるのは、下記の公式仕様を組み合わせれば外部アプリケーションとVRChatを接続する設計余地があることまでです。

## 何を作りたいか

目的は「AIキャラクターが自然に会話できる」と先に宣言することではありません。

**入力 → 外部bridge → AI API → VRChat側の表示**を別々に確認し、どこで失敗したかを切り分けられる最小構成を作ることが目的です。

### 設計方針

- 設計例と動作確認済み実装を分ける
- VRChatで公式に確認できるAPIだけを前提にする
- 入力、外部通信、AI応答、表示を独立して検証する
- model名やprovider名を固定しない
- latencyを保証値にしない
- API keyや会話本文の保存方針を実装前に明示する
- 実装が存在しない段階では、toolingや抽象化を先に増やさない

## 公式仕様から確認できること

### VRChat OSC

VRChat公式ドキュメントでは、OSCで利用できる主要APIとして **Input** と **Avatar Parameters** が案内されています。

- OSC overview: https://docs.vrchat.com/docs/osc-overview
- OSC as Input Controller: https://docs.vrchat.com/docs/osc-as-input-controller
- OSC Avatar Parameters: https://docs.vrchat.com/docs/osc-avatar-parameters

既定では、VRChatは **UDP 9000で受信し、9001へ送信**します。外部アプリケーション側からVRChatへ送る場合は9000、VRChatからの値を外部アプリケーションで受ける場合は9001が既定です。

Avatar Parametersで公式に説明されている値型は **Int / Bool / Float** です。したがって、以前のREADMEにあった

```text
/avatar/parameters/userSpeech = "任意の文字列"
/avatar/parameters/aiResponse = "任意の文字列"
```

という方式は、公式Avatar Parameters仕様に基づく実装として扱いません。

OSC自体は任意のaddress/valueを扱えるプロトコルですが、VRChatが公開しているOSC APIにはVRChat側の契約があります。外部bridgeを作る場合は、その契約と独自アプリ間の通信を区別する必要があります。

### Udon / UdonSharp

VRChat Creator Documentationでは、`OnPlayerTriggerEnter` / `OnPlayerTriggerExit` は正式なPlayer Eventとして掲載されています。

- Event Nodes: https://creators.vrchat.com/worlds/udon/graph/event-nodes/
- Player Collisions: https://creators.vrchat.com/worlds/udon/players/player-collisions/

一方、以前のREADMEにあった `OnChatBoxSubmit` と `OnOSCMessageReceived` は、今回確認したVRChat公式Udon event一覧では実装根拠を確認できませんでした。このrepositoryでは、これらを動作済みcallbackとして説明しません。

## AI APIとの境界

外部AIサービスを使う場合、ユーザー入力はVRChat/ローカル環境の外へ送信されます。これは明示的なprivacy boundaryです。

OpenAI APIを例にすると、現在の公式quickstartはserver-side JavaScriptで公式SDKを使い、API keyを環境変数から読み、Responses APIを呼び出す例を掲載しています。

- OpenAI API quickstart: https://platform.openai.com/docs/quickstart
- API authentication: https://platform.openai.com/docs/api-reference/authentication
- Data controls: https://platform.openai.com/docs/guides/your-data

以前のREADMEに固定されていた `gpt-3.5-turbo` と `/v1/chat/completions` を、このprojectの正準仕様にはしません。provider/model/endpointは実装時に現在の公式documentationへ再照合します。

API keyはsource codeやVRChat worldへ埋め込まず、外部bridge側のenvironment variableまたはsecret managementで扱う必要があります。

## 検証したい最小の経路

実装する場合は、次の順で一段ずつ証拠を残します。

1. VRChat内で、公式Udon eventを使ってlocal player interactionを検出できる
2. 外部OSC applicationがVRChatの公式OSC endpointを送受信できる
3. 外部bridgeが入力をAI providerへ送り、responseを取得できる
4. AI responseをVRChatで表示するための、公式にサポートされた経路を確定する
5. API failure / timeout時に明示的なfallbackを表示できる
6. 会話本文・credential・provider側data retentionの境界を確認する
7. Unity/VRChatの実機またはBuild & Testでend-to-endを再現する

これらが揃うまでは、MVP完成、導入容易性、応答時間、会話品質を保証しません。

## 現在の確認表

| 項目 | 状態 | 根拠 |
|---|---|---|
| VRChat OSCの既定port 9000/9001 | 公式仕様で確認 | VRChat OSC Overview |
| Avatar ParametersのInt/Bool/Float | 公式仕様で確認 | VRChat OSC Avatar Parameters |
| `OnPlayerTriggerEnter/Exit` | 公式仕様で確認 | VRChat Udon Event Nodes |
| `OnChatBoxSubmit` | 未確認 | 公式Udon event一覧で根拠を確認できず |
| `OnOSCMessageReceived` | 未確認 | 公式Udon event一覧で根拠を確認できず |
| VRChat側C#実装 | 未実装 | repositoryにsourceなし |
| OSC bridge | 未実装 | repositoryにsourceなし |
| AI API client | 未実装 | repositoryにsourceなし |
| Prefab | 未実装 | repositoryにassetなし |
| Unity / VRChat runtime test | 未実施 | test evidenceなし |
| 2秒以内のresponse | 未検証 | measurementなし |

## 次に実装するなら

最初にAI provider adapterや販売用設定schemaを増やすのではなく、**VRChat ↔ 外部applicationの1往復を公式仕様だけで再現**するのが先です。その後にAI APIを外部bridgeへ追加します。

この順番なら、VRChat側、network、provider側のどこで問題が起きたかを分けて確認できます。
