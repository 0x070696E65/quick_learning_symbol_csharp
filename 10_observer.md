# 10.監視
SymbolのノードはWebSocket通信でブロックチェーンの状態変化を監視することが可能です。  

## 10.1 リスナー設定

WebSocketを生成してリスナーの設定を行います。本書では以下のクラスを用意しました。C#のSDKにはwebsocketについては実装されていません。理由は使用するWebSocketライブラリ等が環境や作成するアプリケーションによって変更されると想定するからです。

以下のクラスはC#コンソールアプリで使用することを想定しています。Unityや他のアプリケーションなどは各自環境に合わせて実装してください。本書にてWebsocketについて詳しくは説明しません。<br>
https://docs.symbol.dev/ja/api.html#websockets

```cs
class Listener
{
    private readonly string uri;
    private readonly ClientWebSocket webSocket;
    private string uid;
    
    public Listener(string _uri, ClientWebSocket _webSocket)
    {
        uri = _uri;
        webSocket = _webSocket;
        uid = "";
    }

    public async Task Open()
    {
        var serverUri = new Uri(uri);
        await webSocket.ConnectAsync(serverUri, CancellationToken.None);
        Console.WriteLine("Connected to Symbol node WebSocket server.");

        var buffer = new byte[256];
        var receiveResult = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
        var receivedMessage = Encoding.UTF8.GetString(buffer, 0, receiveResult.Count);
        var json = JsonNode.Parse(receivedMessage);
        uid = (string)json["uid"];
    }
    
    public async Task<bool> NewBlock(Action<JsonNode> callback = null)
    {
        var body = Encoding.UTF8.GetBytes("{\"uid\":\"" + uid + "\", \"subscribe\":\"block\"}");
        await webSocket.SendAsync(new ArraySegment<byte>(body), WebSocketMessageType.Text, true, CancellationToken.None);
        while (webSocket.State == WebSocketState.Open)
        {
            var buffer = new byte[4096];
            var receiveResult = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            var receivedMessage = Encoding.UTF8.GetString(buffer, 0, receiveResult.Count);
            var json = JsonNode.Parse(receivedMessage);
            if ((string) json["topic"] == "block")
                callback?.Invoke(json["data"]);
        }
        return true;
    }
    
    public async Task Confirmed(string address, Action<JsonNode> callback = null)
    {
        var body = Encoding.UTF8.GetBytes("{\"uid\":\"" + uid + "\", \"subscribe\":\"confirmedAdded/" + address + "\"}");
        await webSocket.SendAsync(new ArraySegment<byte>(body), WebSocketMessageType.Text, true, CancellationToken.None);
        while (webSocket.State == WebSocketState.Open)
        {
            var buffer = new byte[4096];
            var receiveResult = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            var receivedMessage = Encoding.UTF8.GetString(buffer, 0, receiveResult.Count);
            var json = JsonNode.Parse(receivedMessage);
            if((string)json["topic"] == "confirmedAdded/" + address)
                callback?.Invoke(json["data"]);
        }
    }
    
    public async Task Unconfirmed(string address, Action<JsonNode> callback = null)
    {
        var body = Encoding.UTF8.GetBytes("{\"uid\":\"" + uid + "\", \"subscribe\":\"unconfirmedAdded/" + address + "\"}");
        await webSocket.SendAsync(new ArraySegment<byte>(body), WebSocketMessageType.Text, true, CancellationToken.None);
        while (webSocket.State == WebSocketState.Open)
        {
            var buffer = new byte[4096];
            var receiveResult = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            var receivedMessage = Encoding.UTF8.GetString(buffer, 0, receiveResult.Count);
            var json = JsonNode.Parse(receivedMessage);
            if((string)json["topic"] == "unconfirmedAdded/" + address)
                callback?.Invoke(json["data"]);
        }
    }
    
    public async Task AggregateBondedAdded(string address, Action<JsonNode> callback = null)
    {
        var body = Encoding.UTF8.GetBytes("{\"uid\":\"" + uid + "\", \"subscribe\":\"partialAdded/" + address + "\"}");
        await webSocket.SendAsync(new ArraySegment<byte>(body), WebSocketMessageType.Text, true, CancellationToken.None);
        while (webSocket.State == WebSocketState.Open)
        {
            var buffer = new byte[4096];
            var receiveResult = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            var receivedMessage = Encoding.UTF8.GetString(buffer, 0, receiveResult.Count);
            var json = JsonNode.Parse(receivedMessage);
            if((string)json["topic"] == "partialAdded/" + address)
                callback?.Invoke(json["data"]);
        }
    }
    
    public void Close(string reason = "Closing")
    {
        webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, reason, CancellationToken.None);
        webSocket.Dispose();
    }
}
```

エンドポイントのフォーマットは以下の通りです。
- wss://{node url}:3001/ws

## 10.2 受信検知

アカウントが受信した未承認トランザクションを検知します。

```cs
var listener = new Listener("wss://mikun-testnet.tk:3001/ws", new ClientWebSocket());
await listener.Open();
//未承認トランザクションの検知
await listener.Unconfirmed(aliceAddress.ToString(), (unconfirmedTx) =>
{
    //受信後の処理を記述
    Console.WriteLine($"UnconfirmedTx: {unconfirmedTx}");
});
```
上記リスナーを実行後、aliceへの送信トランザクションをアナウンスしてください。

###### 出力例
```cs
UnconfirmedTx: {
  "transaction": {
    "signature": "3CAC524FB79138955221934C57847F1565EC8364D6CA0FF31519EF57C6DD0C81A30C6420F2AA9E2A80B829CF5B95CC96DEA57D0F14D1EBE5F15983147E784E0F",
    "signerPublicKey": "13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F",
    "version": 1,
    "network": 152,
    "type": 16724,
    "maxFee": "19360",
    "deadline": "14223106946",
    "recipientAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
    "mosaics": [
      {
        "id": "72C0212E67A08BCE",
        "amount": "1000000"
      }
    ]
  },
  "meta": {
    "hash": "2FC13AA2E0FC5E8D48AE494F2058F0FA64AE7A6DE5FF6CE772947B7C39E06A95",
    "merkleComponentHash": "2FC13AA2E0FC5E8D48AE494F2058F0FA64AE7A6DE5FF6CE772947B7C39E06A95",
    "height": "0"
  }
}
```

未承認トランザクションは meta.height=0　で受信します。

アカウントが受信した承認トランザクションを検知します。

```cs
var listener = new Listener("wss://mikun-testnet.tk:3001/ws", new ClientWebSocket());
await listener.Open();
//未承認トランザクションの検知
await listener.Confirmed(aliceAddress.ToString(), (confirmedTx) =>
{
    //受信後の処理を記述
    Console.WriteLine($"ConfirmedTx: {confirmedTx}");
});
```
上記リスナーを実行後、aliceへの送信トランザクションをアナウンスしてください。

###### 出力例
```cs
> ConfirmedTx: {
  "transaction": {
    "signature": "1C50D39E92639C2957818225F4895E86B8181A157D435763B1C79EEE770E5EB6B9D04E57CE845B2A0AC7A0E19C7B4CBC6D08DCB363E8CCF76223BDD420B6620F",
    "signerPublicKey": "13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F",
    "version": 1,
    "network": 152,
    "type": 16724,
    "maxFee": "19888",
    "deadline": "14220836102",
    "recipientAddress": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9",
    "mosaics": [
      {
        "id": "72C0212E67A08BCE",
        "amount": "1000000"
      }
    ]
  },
  "meta": {
    "hash": "63A6E0A10758B10DA4B396E2C038D8190B33EE4E5F14488667F6808C52889059",
    "merkleComponentHash": "63A6E0A10758B10DA4B396E2C038D8190B33EE4E5F14488667F6808C52889059",
    "height": "379346"
  }
}
```

受信後の処理内等で`listener.Close();`とすることで監視を終えることができます。

##### 注意事項
受信先アドレスやモザイクIDで受信検知をする場合は送信者がネームスペースを利用して送信している場合もあるのでご注意ください。
たとえば、メインネットでXYMのモザイクIDは`6BED913FA20223F8`ですが、ユーザーがネームスペースID(symbol.xym)で送信した場合はトランザクションには
`E74B99BA41F4AFEE`というIDが記録されています。

## 10.3 ブロック監視

新規に生成されたブロックを検知します。

```js
var listener = new Listener("wss://mikun-testnet.tk:3001/ws", new ClientWebSocket());
await listener.Open();
await listener.NewBlock((x)=>
{
    Console.WriteLine("New block: " + x);
});
```
###### 出力例
```cs
> New block: {
  "block": {
    "signature": "E638C8E8D1A842AA8A35F5C79733787407EBAB7033CB068923FFCFC597F050927B19DC895B66BC1082ACE1BCC0E0E7B315A6DDCF6FCB204E35FE888E65316F0B",
    "signerPublicKey": "104CCEA2A65EA60F84A45704084EF93BAD423932085000A13B4E9279788933FF",
    "version": 1,
    "network": 152,
    "type": 33091,
    "height": "379421",
    "timestamp": "14216082496",
    "difficulty": "10000000000000",
    "proofGamma": "C06AEE0E1A34D53AEF7CDC6A40EC3A8E6778F3F33C5CFF63FC0C0EF6394C5809",
    "proofVerificationHash": "5B564280F3EE6A28029D04DDD96C63E0",
    "proofScalar": "978E5AA7246642929C6F2870B802F0D4A0B5D0E26BE59161EB71EA3B95D8D803",
    "previousBlockHash": "8BB43BE0477BBF010D99D2AB622B9C1EE4B5A02B0EB8FC53C471E5D46D00C7E1",
    "transactionsHash": "0000000000000000000000000000000000000000000000000000000000000000",
    "receiptsHash": "0F2332B8351B6C51B350C8F14D05C77D0D6B26D72D13B595772F5913A6A80D6C",
    "stateHash": "0ADDFF5514CC4CF229FFDED9764E190910D3EC302FD058C67BF7D4767C22E2F2",
    "beneficiaryAddress": "98A4B6731C0BC7E2778B0A0F1878F8EDF24E933EEEF6D153",
    "feeMultiplier": 0
  },
  "meta": {
    "hash": "719C06EE6ED9E796FC05D84164EBDF3EE60DA16B83C50581C2644C749C8FB4B3",
    "generationHash": "C308B1A5B00EEB24890FD8478B18D8AF46DAE13C7A9FE479744B4230A97CAD8E"
  }
}
```

listener.NewBlock()をしておくと、約30秒ごとに通信が発生するのでWebSocketの切断が起こりにくくなります。  
まれに、ブロック生成が1分を超える場合があるのでその場合はリスナーを再接続する必要があります。
（その他の事象で切断される可能性もあるので、万全を期したい場合は後述するoncloseで補足しましょう）

## 10.4 署名要求

署名が必要なトランザクションが発生すると検知します。

```cs
await listener.Open();
await listener.AggregateBondedAdded(bobAddress.ToString(),(x)=>
{
    Console.WriteLine("AggregateTransaction: " + x);
});
```
###### 出力例
```cs

> AggregateTransaction: {
  "transaction": {
    "signature": "3A64753DCAA8FEC8254D9BDD1C82B36BDA78225EB8F54BE777FB91A8E4908A0B965C7D1225CB1A79D0252F1DAFFE190453CFA20EFEF422DDD3ACC539F37D8002",
    "signerPublicKey": "13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F",
    "version": 2,
    "network": 152,
    "type": 16961,
    "maxFee": "48000",
    "deadline": "14229739681",
    "transactionsHash": "C3327C865541E91BAA68D1CCE3A75E425BC7C5A16E51D87A1D6A9600880ACE2E",
    "transactions": [
      {
        "transaction": {
          "signerPublicKey": "4C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B",
          "version": 1,
          "network": 152,
          "type": 16724,
          "recipientAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
          "mosaics": [
            {
              "id": "E74B99BA41F4AFEE",
              "amount": "1000000"
            }
          ],
          "message": "0074657374"
        }
      },
      {
        "transaction": {
          "signerPublicKey": "13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F",
          "version": 1,
          "network": 152,
          "type": 16724,
          "recipientAddress": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9",
          "mosaics": [
            {
              "id": "E74B99BA41F4AFEE",
              "amount": "1000000"
            }
          ],
          "message": "0074657374"
        }
      }
    ]
  },
  "meta": {
    "hash": "156F449DBEE2E1D8C2E9C883F4DDE7DF0A92C719C096F4B9C25258B04E494001",
    "merkleComponentHash": "0000000000000000000000000000000000000000000000000000000000000000",
    "height": "0"
  }
}
```

指定アドレスが関係するすべてのアグリゲートトランザクションが検知されます。
連署が必要かどうかは別途フィルターして判断します。

## 10.5 現場で使えるヒント
### 常時コネクション

一覧からランダムに選択し、接続を試みます。

##### ノードへの接続チェック関数
```cs
static async Task<string> ConnectNode(string[] nodes)
{
    var httpClient = new HttpClient();
    var node = nodes[new Random().Next(nodes.Length)];
    httpClient.Timeout = TimeSpan.FromSeconds(2);
    Console.WriteLine($"try: {node}");

    HttpResponseMessage response = null;

    try
    {
        httpClient.Timeout = TimeSpan.FromSeconds(2);
        response = await httpClient.GetAsync($"{node}/node/health");

        if (response.IsSuccessStatusCode)
        {
            var content = await response.Content.ReadAsStringAsync();
            var status = JsonNode.Parse(content);
            if ((string)status["status"]["apiNode"] == "up" && (string)status["status"]["db"] == "up")
            {
                var res = await httpClient.GetAsync($"{node}/network/properties");
                var properties = JsonNode.Parse(await res.Content.ReadAsStringAsync());
                if(properties?["network"]?["epochAdjustment"] != null)
                    return node;

                Console.WriteLine("fail network properties");
                return await ConnectNode(nodes);
            }
            else
            {
                Console.WriteLine($"fail node status: {status}");
                return await ConnectNode(nodes);
            }
        }
        else
        {
            Console.WriteLine($"fail request status: {response.StatusCode}");
            return await ConnectNode(nodes);
        }
    }
    catch (TaskCanceledException e)
    {
        Console.WriteLine($"ontimeout: {e}");
        return await ConnectNode(nodes);
    }
    catch (Exception e)
    {
        Console.WriteLine($"onerror: {e}");
        return await ConnectNode(nodes);
    }
}
```

タイムアウト値を設定しておき、応答の悪いノードに接続した場合は選びなおします。
エンドポイント /node/health　を確認してステータス異常の場合はノードを選びなおします。

まれに /network/properties のエンドポイントが解放されていないノードが存在するため、
EpochAdjustmentの情報を取得してチェックを行います。取得できない場合は再帰的にConnectNodeを読み込みます。

##### 正常なノードの検索
```js
//ノード一覧
var NODES = new []
{
    "https://node.com:3001",
    ,,,
};
var connectNode = await ConnectNode(NODES);
Console.WriteLine(connectNode);
```

### 未署名トランザクション自動連署

未署名のトランザクションを検知して、署名＆ネットワークにアナウンスします。  
どのタイミングで検知するかはアプリケーションによって変わります。あくまでも本書では一例を載せますので、あくまでもサンプルとして活用してください。WebSocket、Http接続ライブラリなどは変わります。適切なNullチェックやエラー処理も必要です。

```cs
var NODES = new []
{
    "https://node.com:3001",
    ,,,
};
var connectNode = await ConnectNode(NODES);
var wsNode = connectNode.Replace("https", "wss") + "/ws";
var listener = new Listener(wsNode, new ClientWebSocket());
await listener.Open();
await listener.AggregateBondedAdded(bobAddress.ToString(),async (tx)=>
{
    //bobに関連するアグリゲートトランザクション検知
    var hash = tx["meta"]?["hash"]?.ToString();
    //該当Hashのトランザクション情報取得
    var txInfo = JsonNode.Parse(await GetDataFromApi(node, $"/transactions/partial/{hash}"));
    foreach (JsonNode t in (IEnumerable) txInfo["transaction"]["transactions"])
    {
        // bobの署名が必要なトランザクションのフィルタリング
        if (t["transaction"]["signerPublicKey"].ToString() == bobPublicKey.ToString())
        {
            // 連署作成
            var data = new Dictionary<string, string>()
            {
                {"parentHash", hash},
                {"signature", bobKeyPair.Sign(hash).ToString()},
                {"signerPublicKey", bobPublicKey.ToString()},
                {"version", "0"}
            };
            var json = JsonSerializer.Serialize(data);
            // Cosignatureトランザクションのアナウンス
            var result = await AnnounceCosignature(json);
            Console.WriteLine(result);
        }
    }
});
```

##### 注意事項
スキャムトランザクションを自動署名しないように、
送信元のアカウントを確認するなどのチェック処理を必ず実施するようにしてください。
