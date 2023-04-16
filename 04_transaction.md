# 4.トランザクション
ブロックチェーン上のデータ更新はトランザクションをネットワークにアナウンスすることによって行います。

## 4.1 トランザクションのライフサイクル

トランザクションを作成してから、改ざんが困難なデータとなるまでを順に説明します。

- トランザクション作成
  - ブロックチェーンが受理できるフォーマットでトランザクションを作成します。
- 署名
  - アカウントの秘密鍵でトランザクションを署名します。
- アナウンス
  - 任意のノードに署名済みトランザクションを通知します。
- 未承認トランザクション
  - ノードに受理されたトランザクションは、未承認トランザクションとして全ノードに伝播します
    - トランザクションに設定した最大手数料が、各ノード毎に設定されている最低手数料を満たさない場合はそのノードへは伝播しません。
- 承認済みトランザクション
  - 約30秒に1度ごとに生成されるブロックに未承認トランザクションが取り込まれると、承認済みトランザクションとなります。
- ロールバック
  - ノード間の合意に達することができずロールバックされたブロックに含まれていたトランザクションは、未承認トランザクションに差し戻されます。
    - 有効期限切れや、キャッシュからあふれたトランザクションは切り捨てられます。
- ファイナライズ
  - 投票ノードによるファイナライズプロセスによりブロックが確定するとトランザクションはロールバック不可なデータとして扱うことができます。

### ブロックとは

ブロックは約30秒ごとに生成され、高い手数料を支払ったトランザクションから優先に取り込まれ、ブロック単位で他のノードと同期します。
同期に失敗するとロールバックして、ネットワークが全体で合意が取れるまでこの作業を繰り返します。

## 4.2 トランザクション作成

まずは最も基本的な転送トランザクションを作成してみます。

### Bobへの転送トランザクション

送信先のBobアドレスを秘密鍵から作成しておきます。
```cs
var bobPrivateKey = new PrivateKey("E3839324F3CD2FC194F6E1C501D4D2CFD0DC8CCAC4307AC328E3154FF009****");
var bobKeyPair = new KeyPair(bobPrivateKey);
var bobAddress = facade.Network.PublicKeyToAddress(bobKeyPair.PublicKey);
Console.WriteLine($"bobAddress: {bobAddress}");
```
```cs
> bobAddress: TCIWJYMLTTIPOYAXJD65XDOO7IFIZFXBRA3DS2Y
```

トランザクションを作成します。
```cs
var tx = new TransferTransactionV1
{
    Network = NetworkType.TESTNET, //テストネット・メインネット区分
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Message = Converter.Utf8ToPlainMessage("Hello, Symbol"), //メッセージ
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp) //Deadline:有効期限
};
TransactionHelper.SetMaxFee(tx, 100); //手数料
```

各設定項目について説明します。

#### 有効期限
この例では2時間で設定しています。
最大6時間まで指定可能です。
```cs
Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(6).Timestamp)
```

#### メッセージ
トランザクションに最大1023バイトのメッセージを添付することができます。
バイナリデータであってもrawdataとして送信することが可能です。

##### 空メッセージ
Messageプロパティを設定しないでください

##### 平文メッセージ
```cs
Message = Converter.Utf8ToPlainMessage("Hello, Symbol")
```

##### 暗号文メッセージ
```cs
var encrypted = Crypto.Encode("SENDER_PRIVATE_KEY", "RECIPIENT_PUBLIC_KEY", "MESSAGE");
var encryptedMessage = Converter.Utf8ToEncryptoMessage(encrypted);
  ,,,,
Message = encryptedMessage
```
Crypto.Encodeの第一引数に送信者の秘密鍵（16進数）、第二引数に受信者の公開鍵（16進数）、第三引数にString型の文字列を渡して暗号化された文字列を作成し、Converter.Utf8ToEncryptoMessageにて、「指定したメッセージが暗号化されています」という意味のフラグ（目印）をつけます。
エクスプローラーやウォレットはそのフラグを参考にメッセージを無用にデコードしなかったり、非表示にしたりなどの処理を行います。そのためこのフラグを使用することが暗黙の了解とされています。

##### 生データ
MessageプロパティにRawデータのByte[]を渡してください。

#### 最大手数料
ネットワーク手数料については、常に少し多めに払っておけば問題はないのですが、最低限の知識は持っておく必要があります。
アカウントはトランザクションを作成するときに、ここまでは手数料として払ってもいいという最大手数料を指定します。
一方で、ノードはその時々で最も高い手数料となるトランザクションのみブロックにまとめて収穫しようとします。
つまり、多く払ってもいいというトランザクションが他に多く存在すると承認されるまでの時間が長くなります。
逆に、より少なく払いたいというトランザクションが多く存在し、その総額が大きい場合は、設定した最大額に満たない手数料額で送信が実現します。

トランザクションサイズ x feeMultiprilerというもので決定されます。
176バイトだった場合 maxFee を100で設定すると 17600μXYM = 0.0176XYMを手数料として支払うことを許容します。
そのためFeeプロパティに直接手数料を設定するよりもサイズに合わせてトランザクション構築後にFeeを設定することをおすすめします。

##### feeMultiprier = 100として指定する方法
```cs
var tx = new TransferTransactionV1
{
  Network = NetworkType.TESTNET,
  ,,,,
);
TransactionHelper.SetMaxFee(tx, 100);
```

##### maxFee = 17600 として指定する方法
```cs
var tx = new TransferTransactionV1
{
  Network = NetworkType.TESTNET,
  ,,,,
  Fee = new Amount(17600);
);
```

本書では以後、feeMultiprier = 100として指定する方法で統一して説明します。

## 4.3 署名とアナウンス

作成したトランザクションを秘密鍵で署名して、任意のノードを通じてアナウンスします。

### 署名
```cs
var signature = facade.SignTransaction(aliceKeyPair, tx);
Console.WriteLine($"Signature: {signature}");
```
###### 出力例
```cs
> Signature: 4609EAB80ADD25B16710D00B712F7A7E4916E08B5F0F00ACB65365C179AFA851EC8D22DF9192321A745B20C032F477673EB73423EFC26A3FC6CFD8476097D90C
```

トランザクションの署名にはgenerationHash値が必要ですがfacade作成時の引数であるネットワーク設定にハードコーディングされています。

generationHash
- テストネット
    - 7FCCD304802016BEBBCD342A332F91FF1F3BB5E902988B352697BE245F48E836
- メインネット
    - 57F7DA205008026C776CB6AED843393F04CD458E0AA2D9F1D5F31A402072B2D6

generationHash値はそのブロックチェーンネットワークを一意に識別するための値です。
同じ秘密鍵をもつ他のネットワークに使いまわされないようにそのネットワーク個別のハッシュ値を織り交ぜて署名済みトランザクションを作成します。

今後トランザクションのアナウンスを繰り返すので以下のようにアナウンス用の関数を用意しておくと良いでしょう。なお、本書ではHttpClientを使用しますが環境等に合わせて修正してください。
```cs
static async Task<string> Announce(string payload)
{
    using var client = new HttpClient();
    var content = new StringContent(payload, Encoding.UTF8, "application/json");
    var response =  client.PutAsync(node + "/transactions", content).Result;
    return await response.Content.ReadAsStringAsync();
}
```
### アナウンス
```cs
var payload = TransactionsFactory.AttachSignature(tx, signature);
Console.WriteLine(payload);
var hash = facade.HashTransaction(tx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```
```cs
> {"payload": "A000000000000000472B014BC52C3A712104D1B0A2B90A41CAF68E3B30E93B918539013CCE7566F5F4E2AE68C16FA15A8CF52C7A8FB440E91309590E417B27836AA37A4ACA331F0C13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F0000000001985441803E00000000000067789C3F0300000098E02F5B8E1C4EDA515D9785CC4AB96B274ACC51890026A50000000000000000"}
B7F3EC314FB83A7D22D21431E9DE578BE3D41E7AAFC7D67E36048FB4AA6EDEFA
{"message":"packet 9 was pushed to the network via /transactions"}
```
トランザクションを受け付けるエンドポイントにペイロードを送信するためのJsonを作成しPut送信します。

上記のスクリプトのように `packet n was pushed to the network` というレスポンスがあれば、トランザクションはノードに受理されたことになります。
これはトランザクションのフォーマット等に異常が無かった程度の意味しかありません。
Symbolではノードの応答速度を極限に高めるため、トランザクションの内容を検証するまえに受信結果の応答を返し接続を切断します。
レスポンス値はこの情報を受け取ったにすぎません。フォーマットに異常があった場合は以下のようなメッセージ応答があります。

##### アナウンスに失敗した場合の応答例
```js
Uncaught Error: {"statusCode":409,"statusMessage":"Unknown Error","body":"{\"code\":\"InvalidArgument\",\"message\":\"payload has an invalid format\"}"}
```

## 4.4 確認


### ステータスの確認

ノードに受理されたトランザクションのステータスを確認

```cs
var transactionStatus = JsonNode.Parse(await GetDataFromApi(node, $"/transactionStatus/{hash}"));
Console.WriteLine(transactionStatus);
```
###### 出力例
```cs
> 
{
  "group": "confirmed",
  "code": "Success",
  "hash": "B7F3EC314FB83A7D22D21431E9DE578BE3D41E7AAFC7D67E36048FB4AA6EDEFA",
  "deadline": "13954168653",
  "height": "370966"
}
```

承認されると ` group: "confirmed"`となっています。

受理されたものの、エラーが発生していた場合は以下のような出力となります。トランザクションを書き直して再度アナウンスしてみてください。

```js
> 
{
  "group": "failed",
  "code": "Failure_Core_Insufficient_Balance",
  "hash": "A82507C6C46DF444E36AC94391EA2D0D7DD1A218948DED465A7A4F9D1B53CA0E",
  "deadline": "11990156766",
  "height": "undefined"
}
```

以下のようにResourceNotFoundエラーが発生した場合はトランザクションが受理されていません。
```js
Uncaught Error: {"statusCode":404,"statusMessage":"Unknown Error","body":"{\"code\":\"ResourceNotFound\",\"message\":\"no resource exists with id '18AEBC9866CD1C15270F18738D577CB1BD4B2DF3EFB28F270B528E3FE583F42D'\"}"}
```

考えられる可能性としては、トランザクションで指定した最大手数料が、ノードで設定された最低手数料に満たない場合や、
アグリゲートトランザクションとしてアナウンスすることが求められているトランザクションを単体のトランザクションでアナウンスした場合に発生するようです。

### 承認確認

トランザクションがブロックに承認されるまでに30秒程度かかります。

#### エクスプローラーで確認
hash で取得できたハッシュ値を使ってエクスプローラーで検索してみましょう。

```cs
Console.WriteLine(hash);
```
```cs
> "661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747"
```

- メインネット　
  - https://symbol.fyi/transactions/661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747
- テストネット　
  - https://testnet.symbol.fyi/transactions/661360E61C37E156B0BE18E52C9F3ED1022DCE846A4609D72DF9FA8A5B667747

#### REST APIで確認

```cs
var jsonString = await GetDataFromApi(node, $"/transactions/confirmed/{hash}");
Console.WriteLine(jsonString);
var transaction = JsonNode.Parse(jsonString);
Console.WriteLine($"Signature: {transaction["transaction"]["signature"]}");
```
デシリアライズする前の文字列とデシリアライズ後のSignatureをサンプルに出力してみます。

###### 出力例
```js
> 
{
	"meta": {
		"height": "370966",
		"hash": "B7F3EC314FB83A7D22D21431E9DE578BE3D41E7AAFC7D67E36048FB4AA6EDEFA",
		"merkleComponentHash": "B7F3EC314FB83A7D22D21431E9DE578BE3D41E7AAFC7D67E36048FB4AA6EDEFA",
		"index": 0,
		"timestamp": "13947005958",
		"feeMultiplier": 100
	},
	"transaction": {
		"size": 174,
		"signature": "3C2E15F2BAED8780A454737B168515422158378355EC6946C82A1178DE7D8BF12FC6897FB6EABF1079EDB8462AEE700A84D8FA9BFF5959FCD63EBBFD27A1980F",
		"signerPublicKey": "13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F",
		"version": 1,
		"network": 152,
		"type": 16724,
		"maxFee": "17400",
		"deadline": "13954168653",
		"recipientAddress": "98B8FBAF8D63C9E392382E56E4A8E015365BBA5AF95371EB",
		"message": "0048656C6C6F2C2053796D626F6C",
		"mosaics": []
	},
	"id": "643509A1D7D26E76F9292A94"
}
Signature: 3C2E15F2BAED8780A454737B168515422158378355EC6946C82A1178DE7D8BF12FC6897FB6EABF1079EDB8462AEE700A84D8FA9BFF5959FCD63EBBFD27A1980F
```
##### 注意点

トランザクションはブロックで承認されたとしても、ロールバックが発生するとトランザクションの承認が取り消される場合があります。
ブロックが承認された後、数ブロックの承認が進むと、ロールバックの発生する確率は減少していきます。
また、Votingノードの投票で実施されるファイナライズブロックを待つことで、記録されたデータは確実なものとなります。

## 4.5トランザクション履歴

Aliceが送受信したトランザクション履歴を一覧で取得します。

https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/searchConfirmedTransactions

以下のように取得します。例では最初のトランザクションのハッシュを出力していますがご自身の必要な情報に合わせてください。
```cs
var transactions = JsonNode.Parse(await GetDataFromApi(node, $"/transactions/confirmed?embedded=true&address={aliceAddress}"));
Console.WriteLine(transactions["data"]);
```
###### 出力例
```js
> 
[
  {
    "meta": {
      "height": "5384",
      "hash": "1B5DA615114774D75105258D8CFE50EB58B48519A40BA4A377679471ABECC993",
      "merkleComponentHash": "1B5DA615114774D75105258D8CFE50EB58B48519A40BA4A377679471ABECC993",
      "index": 2,
      "timestamp": "323445994",
      "feeMultiplier": 1136363
    },
    "transaction": {
      "size": 176,
      "signature": "EA758302AB948DFBE6DF1131DB7907ED48428FD08215CC187816151DA72E34DADFDE9DB1FB1C29A70BBB9719DD3857543264375A3756B5479D9DBCED5A4D9706",
      "signerPublicKey": "81EA7C15E7EC06261C9F654F54EAC4748CFCF00E09A8FE47779ACD14A7602004",
      "version": 1,
      "network": 152,
      "type": 16724,
      "maxFee": "200000000",
      "deadline": "330633658",
      "recipientAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
      "mosaics": [
        {
          "id": "72C0212E67A08BCE",
          "amount": "1000000000"
        }
      ]
    },
    "id": "636CF155D7D26E76F900C871"
  },
  ,,,,
```

TransactionTypeは以下の通りです。
```js
{0: 'RESERVED', 16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS', 17232: 'ACCOUNT_OPERATION_RESTRICTION'
```

## 4.6 アグリゲートトランザクション

Symbolでは複数のトランザクションを1ブロックにまとめてアナウンスすることができます。
最大で100件のトランザクションをまとめることができます（連署者が異なる場合は25アカウントまでを連署指定可能）。
以降の章で扱う内容にアグリゲートトランザクションへの理解が必要な機能が含まれますので、
本章ではアグリゲートトランザクションのうち、簡単なものだけを紹介します。
### 起案者の署名だけが必要な場合

```cs
var bobPrivateKey = new PrivateKey("E3839324F3CD2FC194F6E1C501D4D2CFD0DC8CCAC4307AC328E3154FF009****");
var bobKeyPair = new KeyPair(bobPrivateKey);
var bobAddress = facade.Network.PublicKeyToAddress(bobKeyPair.PublicKey);

var carolKeyPair = KeyPair.GenerateNewKeyPair();
var carolAddress = facade.Network.PublicKeyToAddress(carolKeyPair.PublicKey);

var innerTx1 = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Message = Converter.Utf8ToPlainMessage("tx1")
};
var innerTx2 = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(carolAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Message = Converter.Utf8ToPlainMessage("tx2")
};
var innerTransactions = new IBaseTransaction[] { innerTx1, innerTx2 };
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);
var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
};
TransactionHelper.SetMaxFee(aggregateTx, 100);

var signature = facade.SignTransaction(aliceKeyPair, aggregateTx);
var payload = TransactionsFactory.AttachSignature(aggregateTx, signature);
var hash = facade.HashTransaction(aggregateTx, signature);

var result = await Announce(payload);
Console.WriteLine(result);
```

まず、アグリゲートトランザクションに含めるトランザクションを作成します。
このときDeadlineを指定する必要はありません。
この時、SignerPublicKeyにインナートランザクションの送信元の公開鍵を指定します。
ちなみに送信元アカウントと署名アカウントが **必ずしも一致するとは限りません** 。
後の章での解説で「Bobの送信トランザクションをAliceが署名する」といった事が起こり得るためこのような書き方をします。
これはSymbolブロックチェーンでトランザクションを扱ううえで最も重要な概念になります。
なお、本章で扱うトランザクションは同じAliceですので、アグリゲートボンデッドトランザクションへの署名もAliceを指定します。

## 4.7 現場で使えるヒント

### 存在証明

アカウントの章でアカウントによるデータの署名と検証する方法について説明しました。
このデータをトランザクションに載せてブロックチェーンが承認することで、
アカウントがある時刻にあるデータの存在を認知したことを消すことができなくなります。
タイムスタンプの刻印された電子署名を利害関係者間で所有することと同じ意味があると考えることもできます。
（法律的な判断は他の方にお任せします）

ブロックチェーンは、この消せない「アカウントが認知したという事実」の存在をもって送信などのデータ更新を行います。
また、誰もがまだ知らないはずの事実を知っていたことの証明としてブロックチェーンを利用することもできます。
ここでは、その存在が証明されたデータをトランザクションに載せる２つの方法について説明します。

#### デジタルデータのハッシュ値(SHA256)出力方法

ファイルの要約値をブロックチェーンに記録することでそのファイルの存在を証明することができます。

各OSごとのファイルのSHA256でハッシュ値を計算する方法は下記の通りです。
```sh
#Windows
certutil -hashfile WINファイルパス SHA256
#Mac
shasum -a 256 MACファイルパス
#Linux
sha256sum Linuxファイルパス
```

#### 大きなデータの分割

トランザクションのペイロードには1023バイトしか格納できないため、
大きなデータは分割してペイロードに詰め込んでアグリゲートトランザクションにします。

```cs
const string bigdata = "C00200000000000093B0B985101C1BDD1BC2BF30D72F35E34265B3F381ECA464733E147A4F0A6B9353547E2E08189EF37E50D271BEB5F09B81CE5816BB34A153D2268520AF630A0A0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198414140770200000000002A769FB40000000076B455CFAE2CCDA9C282BF8556D3E9C9C0DE18B0CBE6660ACCF86EB54AC51B33B001000000000000DB000000000000000E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198544198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F22338B000000000000000066653465353435393833444430383935303645394533424446434235313637433046394232384135344536463032413837364535303734423641303337414643414233303344383841303630353343353345354235413835323835443639434132364235343233343032364244444331443133343139464435353438323930334242453038423832304100000000006800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F2233BC089179EBBE01A81400140035383435344434373631364336433635373237396800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F223345ECB996EDDB9BEB1400140035383435344434373631364336433635373237390000000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D5A71EBA9C924EFA146897BE6C9BB3DACEFA26A07D687AC4A83C9B03087640E2D1DDAE952E9DDBC33312E2C8D021B4CC0435852C0756B1EBD983FCE221A981D02";
var payloads = SplitString(bigdata, 1023);
foreach (var p in payloads)
{
    Console.WriteLine(p);
}

static List<string> SplitString(string str, int chunkSize)
{
    var chunks = new List<string>();
    var strLength = str.Length;
    for (var i = 0; i < strLength; i += chunkSize) {
        var length = Math.Min(chunkSize, strLength - i);
        var chunk = str.Substring(i, length);
        chunks.Add(chunk);
    }
    return chunks;
}
```
