# 7.メタデータ

アカウント・モザイク・ネームスペースに対してKey-Value形式のデータを登録することができます。  
Valueの最大値は1024バイトです。
本章ではモザイク・ネームスペースの作成アカウントとメタデータの作成アカウントがどちらもAliceであることを前提に説明します。

## 7.1 アカウントに登録

アカウントに対して、Key-Value値を登録します。

```cs
var key = IdGenerator.GenerateUlongKey("key_account");
var valueBytes = Converter.Utf8ToBytes("test");

var tx = new EmbeddedAccountMetadataTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    TargetAddress = new UnresolvedAddress(aliceAddress.bytes),
    ScopedMetadataKey = key,
    Value = valueBytes,
    ValueSizeDelta = (byte) valueBytes.Length
};
var innerTransactions = new IBaseTransaction[] {tx};
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);

var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(aggregateTx, 100);

var signature = facade.SignTransaction(aliceKeyPair, aggregateTx);
var payload = TransactionsFactory.AttachSignature(aggregateTx, signature);
var hash = facade.HashTransaction(aggregateTx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```

メタデータの登録には記録先アカウントが承諾を示す署名が必要です。
また、記録先アカウントと記録者アカウントが同一でもアグリゲートトランザクションにする必要があります。

異なるアカウントのメタデータに登録する場合は連署者の署名が必要です。
なお、手数料をfeeMultiplierにて設定する場合は連署者の数×104が追加の数量として必要です。

```cs
var key = IdGenerator.GenerateUlongKey("key_account");
var valueBytes = Converter.Utf8ToBytes("test");

var tx = new EmbeddedAccountMetadataTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    TargetAddress = new UnresolvedAddress(bobAddress.bytes),
    ScopedMetadataKey = key,
    Value = valueBytes,
    ValueSizeDelta = (byte) valueBytes.Length
};
var innerTransactions = new IBaseTransaction[] {tx};
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);

var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(aggregateTx, 100, 1/*連署者の数*/);

// 署名
var aliceSignature = facade.SignTransaction(aliceKeyPair, aggregateTx);
TransactionsFactory.AttachSignature(aggregateTx, aliceSignature);

var hash = facade.HashTransaction(aggregateTx);

// 連署者による署名
var bobCosignature = new Cosignature
{
    Signature = bobKeyPair.Sign(hash.bytes),
    SignerPublicKey = bobKeyPair.PublicKey
};
aggregateTx.Cosignatures = new [] {bobCosignature};

var payload = TransactionsFactory.CreatePayload(aggregateTx);
var result = await Announce(payload);
Console.WriteLine(result);
```

bobの秘密鍵が分からない場合はこの後の章で説明する
アグリゲートボンデッドトランザクション、あるいはオフライン署名を使用する必要があります。

## 7.2 モザイクに登録

ターゲットとなるモザイクに対して、Key値・ソースアカウントの複合キーでValue値を登録します。
登録・更新にはモザイクを作成したアカウントの署名が必要です。

```cs
var key = IdGenerator.GenerateUlongKey("key_mosaic");
var valueBytes = Converter.Utf8ToBytes("test");

var tx = new EmbeddedMosaicMetadataTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    TargetAddress = new UnresolvedAddress(aliceAddress.bytes), //モザイク作成者アドレス
    TargetMosaicId = new UnresolvedMosaicId(0x1F75D061E31413F7), // mosaic id
    ScopedMetadataKey = key, // Key
    Value = valueBytes, // Value
    ValueSizeDelta = (byte) valueBytes.Length
};
var innerTransactions = new IBaseTransaction[] {tx};
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);

var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(aggregateTx, 100);

var signature = facade.SignTransaction(aliceKeyPair, aggregateTx);
var payload = TransactionsFactory.AttachSignature(aggregateTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

## 7.3 ネームスペースに登録

ネームスペースに対して、Key-Value値を登録します。
登録・更新にはネームスペースを作成したアカウントの署名が必要です。

```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook");
var key = IdGenerator.GenerateUlongKey("key_namespace");
var valueBytes = Converter.Utf8ToBytes("test");

var tx = new EmbeddedNamespaceMetadataTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey, //メタデータの登録者
    TargetAddress = new UnresolvedAddress(aliceAddress.bytes), //ネームスペースの作成者アドレス
    TargetNamespaceId = new NamespaceId(namespaceId),
    ScopedMetadataKey = key, //Key
    Value = valueBytes, //Value
    ValueSizeDelta = (byte) valueBytes.Length
};
var innerTransactions = new IBaseTransaction[] {tx};
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);

var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(aggregateTx, 100);

var signature = facade.SignTransaction(aliceKeyPair, aggregateTx);
var payload = TransactionsFactory.AttachSignature(aggregateTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

## 7.4 確認
登録したメタデータを確認します。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Metadata-routes/operation/searchMetadataEntries

```cs
var metadatas = JsonNode.Parse(await GetDataFromApi(node, $"/metadata?sourceAddress={aliceAddress}&targetAddress={aliceAddress}"));
Console.WriteLine(metadatas);
```
###### 出力例
```cs
{
  "data": [
    {
      "metadataEntry": {
        "version": 1,
        "compositeHash": "19EB6F74E30C1016A826938957F1746C794DB14DC7763DB2C74F30271DCC0F2B",
        "sourceAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
        "targetAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
        "scopedMetadataKey": "DF16D14E750D0048",
        "targetId": "0000000000000000",
        "metadataType": 0,
        "valueSize": 6,
        "value": "616161616161"
      },
      "id": "636CF1F45ED31320FDD28430"
    },
    {
      "metadataEntry": {
        "version": 1,
        "compositeHash": "9E8B1392C31D76F0E0108EFFC331ABFCC58C0DC116A8539361482C40301B98B3",
        "sourceAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
        "targetAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
        "scopedMetadataKey": "E15492B77F0B0D44",
        "targetId": "0000000000000000",
        "metadataType": 0,
        "valueSize": 6,
        "value": "616161616161"
      },
      "id": "636CF1F45ED31320FDD28432"
    },
```
metadataTypeは以下の通りです。
```cs
{0: 'Account', 1: 'Mosaic', 2: 'Namespace'}
```

### 注意事項
メタデータはキー値で素早く情報にアクセスできるというメリットがある一方で更新可能であることに注意しておく必要があります。
更新には、発行者アカウントと登録先アカウントの署名が必要のため、それらのアカウントの管理状態が信用できる場合のみ使用するようにしてください。


## 7.5 現場で使えるヒント

### 有資格証明

モザイクの章で所有証明、ネームスペースの章でドメインリンクの説明をしました。
実社会で信頼性の高いドメインからリンクされたアカウントが発行したメタデータの付与を受けることで
そのドメイン内での有資格情報の所有を証明することができます。

#### DID

分散型アイデンティティと呼ばれます。
エコシステムは発行者、所有者、検証者に分かれ、例えば大学が発行した卒業証書を学生が所有し、
企業は学生から提示された証明書を大学が公表している公開鍵をもとに検証します。
このやりとりにプラットフォームに依存する情報はありません。
メタデータを活用することで、大学は学生の所有するアカウントにメタデータを発行することができ、
企業は大学の公開鍵と学生のモザイク(アカウント)所有証明でメタデータに記載された卒業証明を検証することができます。


