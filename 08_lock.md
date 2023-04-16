# 8.ロック

Symbolブロックチェーンにはハッシュロックとシークレットロックの２種類のロック機構があります。  

## 8.1 ハッシュロック

ハッシュロックは後でアナウンスされる予定のトランザクションを事前にハッシュ値で登録しておくことで、
該当トランザクションがアナウンスされた場合に、そのトランザクションをAPIノード上で処理せずにロックさせて、署名が集まってから処理を行うことができます。
アカウントが所有するモザイクを操作できないようにロックするわけではなく、ロックされるのはハッシュ値の対象となるトランザクションとなります。
ハッシュロックにかかる費用は10XYM、有効期限は最大約48時間です。ロックしたトランザクションが承認されれば10XYMは返却されます。

### アグリゲートボンデッドトランザクションの作成

```cs
var bobPublicKey = new PublicKey(Converter.HexToBytes("4C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B"));
var bobAddress = facade.Network.PublicKeyToAddress(bobPublicKey);

var namespaceId = IdGenerator.GenerateNamespaceId("symbol.xym");

var tx1 = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),  //Bobへの送信
    SignerPublicKey = alicePublicKey,
    Mosaics = new [] //1XYM
    {
        new UnresolvedMosaic()
        {
            MosaicId = new UnresolvedMosaicId(namespaceId),
            Amount = new Amount(1000000)
        }
    },
    //メッセージ無し
};
var tx2 = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(aliceAddress.bytes), // Aliceへの送信
    SignerPublicKey = bobPublicKey,
    Message = Converter.Utf8ToPlainMessage("thank you!") //メッセージ
};

var innerTransactions = new IBaseTransaction[] {tx1, tx2};
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);

var aggregateTx = new AggregateBondedTransactionV2() {
    Network = NetworkType.TESTNET,
    Transactions = 	innerTransactions,
    SignerPublicKey = aliceKeyPair.PublicKey,
    TransactionsHash = merkleHash,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
};
TransactionHelper.SetMaxFee(aggregateTx, 100, 2/*連署者の数*/);

//署名
var aliceSignature = facade.SignTransaction(aliceKeyPair, aggregateTx);
```
tx1,tx2の2つのトランザクション作成する際に送信元アカウントの公開鍵を`SignerPublicKey`に指定します。公開鍵はアカウントの章を参考に事前にAPIで取得しておきましょう。

配列化されたトランザクションはブロック承認時にその順序で整合性を検証されます。
例えば、tx1でNFTをAliceからBobへ送信した後、tx2でBobからCarolへ同じNFTを送信することは可能ですが、tx2,tx1の順序でアグリゲートトランザクションを通知するとエラーになります。
また、アグリゲートトランザクションの中に1つでも整合性の合わないトランザクションが存在していると、アグリゲートトランザクション全体がエラーとなってチェーンに承認されることはありません。

またハッシュロックトランザクション承認後payloadを指定してボンデッドトランザクションをアナウンスします。そのためボンデッドトランザクション構築時にペイロードを保管するようにします。

後ほどBobは連署するためにアグリゲートトランザクションのハッシュが必要になりますので、それもコンソールに出力し保存しておきます。

```cs
// Bonded用payload作成
var payloadBonded = TransactionsFactory.AttachSignature(aggregateTx, aliceSignature);
Console.WriteLine(payloadBonded);

var hash = facade.HashTransaction(aggregateTx);
Console.WriteLine(hash);
```

### ハッシュロックトランザクションの作成と署名、アナウンス
```cs

//ハッシュロックTX作成
var hashLockTx = new HashLockTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = aliceKeyPair.PublicKey,
    Mosaic = new UnresolvedMosaic() //10xym固定値
    {
        MosaicId = new UnresolvedMosaicId(namespaceId),
        Amount = new Amount(10 * 1000000)
    },
    Duration = new BlockDuration(480), // ロック有効期限
    Hash = hash, // このハッシュ値を登録
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
};
TransactionHelper.SetMaxFee(hashLockTx, 100);

//署名
var signature = facade.SignTransaction(aliceKeyPair, hashLockTx);
var payload = TransactionsFactory.AttachSignature(hashLockTx, signature);

//ハッシュロックTXをアナウンス
var result = await Announce(payload);
Console.WriteLine(result);
```

### アグリゲートボンデッドトランザクションのアナウンス

アグリゲートボンデッドトランザクションは通常のトランザクションとはアナウンス先が異なります。以下のような関数を用意しておくと良いでしょう。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/announcePartialTransaction

```cs
static async Task<string> AnnounceBonded(string payload)
{
    using var client = new HttpClient();
    var content = new StringContent(payload, Encoding.UTF8, "application/json");
    var response =  client.PutAsync(node + "/transactions/partial", content).Result;
    return await response.Content.ReadAsStringAsync();
}
```

エクスプローラーなどで確認した後、ボンデッドトランザクションをネットワークにアナウンスします。
```cs
var resultBonded = await AnnounceBonded(payloadBonded);
Console.WriteLine(resultBonded);
```

### 連署
ロックされたトランザクションを指定されたアカウント(Bob)で連署します。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/announceCosignatureTransaction

連署のトランザクションもアナウンス先が異なります。以下のような関数を用意しておくなど対応してください。

```cs
static async Task<string> AnnounceCosignature(string data)
{
    using var client = new HttpClient();
    var content = new StringContent(data, Encoding.UTF8, "application/json");
    var response =  client.PutAsync(node + "/transactions/cosignature", content).Result;
    return await response.Content.ReadAsStringAsync();
}
```

事前に保存しておいたハッシュに対して署名し連署トランザクションをアナウンスします。
```cs
const string hash = "111194F4FEC35A27F91BBAD2F37E2AAFE037F77A2D503B8D0DE402F2AD4017D3";
var data = new Dictionary<string, string>()
{
    {"parentHash", hash},
    {"signature", bobKeyPair.Sign(hash).ToString()},
    {"signerPublicKey", bobPublicKey.ToString()},
    {"version", "0"}
};
var json = JsonSerializer.Serialize(data);
var result = await AnnounceCosignature(json);
Console.WriteLine(result);
```

なお、以下のように特定のアカウントに対して連署を要求するトランザクションが存在しているか確認することも可能です。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Transaction-routes/operation/searchPartialTransactions

```cs
var partial = JsonNode.Parse(await GetDataFromApi(node, $"/transactions/partial?address={bobAddress}"));
Console.WriteLine(partial);
```

### 注意点
ハッシュロックトランザクションは起案者(トランザクションを作成し最初に署名するアカウント)に限らず、誰が作成してアナウンスしても大丈夫ですが、
アグリゲートトランザクションにそのアカウントがsignerとなるトランザクションを含めるようにしてください。
モザイク送信無し＆メッセージ無しのダミートランザクションでも問題ありません（パフォーマンスに影響が出るための仕様とのことです）

## 8.2 シークレットロック・シークレットプルーフ

シークレットロックは事前に共通パスワードを作成しておき、指定モザイクをロックします。
受信者が有効期限内にパスワードの所有を証明することができればロックされたモザイクを受け取ることができる仕組みです。

ここではAliceが1XYMをロックしてBobが解除することで受信する方法を説明します。

### シークレットロック

ロック・解除にかかわる共通暗号を作成します。今回はSHA3_256というハッシュアルゴリズムで作成します。

ロック・解除にかかわる共通暗号を作成します。
```cs
var (proof, secret) = Crypto.CreateHash256Pair();
Console.WriteLine($"secret: {Converter.BytesToHex(secret)}"); //ロック用キーワード
Console.WriteLine($"proof: {Converter.BytesToHex(proof)}"); //解除用キーワード
```

###### 出力例
```cs
> secret: 84DC2F922F1A7F42A29C0CF350A193BA035FB3B84DDF7F1E97979CD3AC90FEF5
proof: 99BA16BDC20BBE77617993AB21877D0C8E71C147
```

トランザクションを作成・署名・アナウンスします
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("symbol.xym");
var lockTx = new SecretLockTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes), //解除時の転送先:Bob
    Mosaic = new UnresolvedMosaic
    {
        MosaicId = new UnresolvedMosaicId(namespaceId),
        Amount = new Amount(1000000) //1XYM
    }, //ロックするモザイク
    Secret = new Hash256(secret), //ロック用キーワード
    Duration = new BlockDuration(480), //ロック期間(ブロック数)
    HashAlgorithm = new LockHashAlgorithm(0), //ロックキーワード生成に使用したアルゴリズム
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
};
TransactionHelper.SetMaxFee(lockTx, 100);

var signature = facade.SignTransaction(aliceKeyPair, lockTx);
var payload = TransactionsFactory.AttachSignature(lockTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

LockHashAlgorithmは以下の通りです。
```js
{0: 'SHA3_256', 1: 'HASH_160', 2: 'HASH_256'}
```

ロック時に解除先を指定するのでBob以外のアカウントが解除しても転送先（Bob）を変更することはできません。
ロック期間は最長で365日(ブロック数を日換算)までです。

承認されたトランザクションを確認します。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Secret-Lock-routes/operation/searchSecretLock
```cs
var secretLockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/lock/secret?secret=196191F74708E2B4A52AEB643A3BA7E19655A64E7FAC6FBBA4F267BC18EDFF9E"));
Console.WriteLine($"SecretLockInfo: {secretLockInfo}");
```
###### 出力例
```cs
> SecretLockInfo
    amount: UInt64 {lower: 1000000, higher: 0}
    compositeHash: "770F65CB0CC0CA17370DE961B2AA5B48B8D86D6DB422171AB00DF34D19DEE2F1"
    endHeight: UInt64 {lower: 323495, higher: 0}
    hashAlgorithm: 0
    mosaicId: MosaicId {id: Id}
    ownerAddress: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
    recipientAddress: Address {address: 'TBTWKXCNROT65CJHEBPL7F6DRHX7UKSUPD7EUGA', networkType: 152}
    recordId: "6260A1D3205E94BEA3D9E3E9"
    secret: "F260BFB53478F163EE61EE3E5FB7CFCAF7F0B663BC9DD4C537B958D4CE00E240"
    status: 0
    version: 1
> SecretLockInfo: {
  "data": [
    {
      "lock": {
        "version": 1,
        "ownerAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
        "mosaicId": "72C0212E67A08BCE",
        "amount": "1000000",
        "endHeight": "377073",
        "status": 0,
        "hashAlgorithm": 0,
        "secret": "196191F74708E2B4A52AEB643A3BA7E19655A64E7FAC6FBBA4F267BC18EDFF9E",
        "recipientAddress": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9",
        "compositeHash": "4848923EAC8E9F0874B9682075718E868F416ED73BD0A5CBBD3610034B5EB698"
      },
```
ロックしたAliceがownerAddress、受信予定のBobがrecipientAddressに記録されています。

secret情報が公開されていて、これに対応するproofをBobがネットワークに通知します。

### シークレットプルーフ

解除用キーワードを使用してロック解除します。
Bobは事前に解除用キーワードを入手しておく必要があります。

```cs
var secret = Converter.HexToBytes("196191F74708E2B4A52AEB643A3BA7E19655A64E7FAC6FBBA4F267BC18EDFF9E"); //ロックキーワード
var proof = Converter.HexToBytes("91B7E1E02D98C8DB6CFA90AF810D120FED9D854E"); //解除用キーワード
var proofTx = new SecretProofTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes), //解除アカウント（受信アカウント）
    Proof = proof,
    Secret = new Hash256(secret), //ロックキーワード
    HashAlgorithm = new LockHashAlgorithm(0), //ロック作成に使用したアルゴリズム
    SignerPublicKey = bobPublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
};
TransactionHelper.SetMaxFee(proofTx, 100);

var signature = facade.SignTransaction(bobKeyPair, proofTx);
var hash = facade.HashTransaction(proofTx);
Console.WriteLine(hash);
var payload = TransactionsFactory.AttachSignature(proofTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

承認結果を確認します。
```cs
var hash = "1FB012172203DAA7BEDB527E7123552B42B589D7A01A299368B49B2CA7EDB9B1";
var transactionInfo = JsonNode.Parse(await GetDataFromApi(node, $"/transactions/confirmed/{hash}"));
Console.WriteLine($"SecretProofTransaction: {transactionInfo}");
```
###### 出力例
```js
> SecretProofTransaction: {
  "meta": {
    "height": "376618",
    "hash": "1FB012172203DAA7BEDB527E7123552B42B589D7A01A299368B49B2CA7EDB9B1",
    "merkleComponentHash": "1FB012172203DAA7BEDB527E7123552B42B589D7A01A299368B49B2CA7EDB9B1",
    "index": 0,
    "timestamp": "14127260541",
    "feeMultiplier": 100
  },
  "transaction": {
    "size": 207,
    "signature": "BA95A3E60DF8050DEED546883EF952BAAD220D2D0E4EF4AF8E7282A399738613429369AACB14AFC9BB649E65CCB0142AD2BBCA0556CCFBBC03BC61C068B57300",
    "signerPublicKey": "4C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B",
    "version": 1,
    "network": 152,
    "type": 16978,
    "maxFee": "20700",
    "deadline": "14134446056",
    "hashAlgorithm": 0,
    "secret": "196191F74708E2B4A52AEB643A3BA7E19655A64E7FAC6FBBA4F267BC18EDFF9E",
    "recipientAddress": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9",
    "proof": "91B7E1E02D98C8DB6CFA90AF810D120FED9D854E"
  },
  "id": "6437C9C0D7D26E76F9296B3A"
}
```

SecretProofTransactionにはモザイクの受信量の情報は含まれていません。
ブロック生成時に作成されるレシートで受信量を確認します。
レシートタイプ:LockSecret_Completed(8786) でBob宛のレシートを検索してみます。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Receipt-routes/operation/searchReceipts

```cs
var receiptInfo = JsonNode.Parse(await GetDataFromApi(node, $"/statements/transaction?targetAddress={bobAddress}&receiptType=8786&order=desc"));
Console.WriteLine($"ReceiptInfo: {receiptInfo}");
```
###### 出力例
```cs
> ReceiptInfo: {
  "data": [
    {
      "statement": {
        "height": "376618",
        "source": {
          "primaryId": 1,
          "secondaryId": 0
        },
        "receipts": [
          {
            "version": 1,
            "type": 8786,
            "targetAddress": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9",
            "mosaicId": "72C0212E67A08BCE",
            "amount": "1000000"
          }
        ]
      },
      "id": "6437C9C0D7D26E76F9296B3C",
      "meta": {
        "timestamp": "14127260541"
      }
    },
```

ReceiptTypeは以下の通りです。

```cs
{4685: 'Mosaic_Rental_Fee', 4942: 'Namespace_Rental_Fee', 8515: 'Harvest_Fee', 8776: 'LockHash_Completed', 8786: 'LockSecret_Completed', 9032: 'LockHash_Expired', 9042: 'LockSecret_Expired', 12616: 'LockHash_Created', 12626: 'LockSecret_Created', 16717: 'Mosaic_Expired', 16718: 'Namespace_Expired', 16974: 'Namespace_Deleted', 20803: 'Inflation', 57667: 'Transaction_Group', 61763: 'Address_Alias_Resolution', 62019: 'Mosaic_Alias_Resolution'}

8786: 'LockSecret_Completed' :ロック解除完了
9042: 'LockSecret_Expired'　：ロック期限切れ
```

## 8.3 現場で使えるヒント


### 手数料代払い

一般的にブロックチェーンはトランザクション送信に手数料を必要とします。
そのため、ブロックチェーンを利用しようとするユーザは事前に手数料を取引所から入手しておく必要があります。
このユーザが企業である場合はその管理方法も加えてさらにハードルの高い問題となります。
アグリゲートトランザクションを使用することでハッシュロック費用とネットワーク手数料をサービス提供者が代理で負担することができます。

### タイマー送信

シークレットロックは指定ブロック数を経過すると元のアカウントへ払い戻されます。
この原理を利用して、シークレットロックしたアカウントにたいしてロック分の費用をサービス提供者が充足しておけば、
期限が過ぎた後ユーザ側がロック分のトークン所有量が増加することになります。
一方で、期限が過ぎる前にシークレット証明トランザクションをアナウンスすると、送信が完了し、サービス提供者に充当戻るためキャンセル扱いとなります。

### アトミックスワップ
シークレットロックを使用して、他のチェーンとのトークン・モザイクの交換を行うことができます。
他のチェーンではハッシュタイムロックコントラクト(HTLC)と呼ばれているためハッシュロックと間違えないようにご注意ください。


