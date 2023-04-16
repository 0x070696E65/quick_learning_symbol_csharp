# 6.ネームスペース

Symbolブロックチェーンではネームスペースをレンタルしてアドレスやモザイクに視認性の高い単語をリンクさせることができます。
ネームスペースは最大64文字、利用可能な文字は a, b, c, …, z, 0, 1, 2, …, 9, _ , - です。

## 6.1 手数料の計算

ネームスペースのレンタルにはネットワーク手数料とは別にレンタル手数料が発生します。
ネットワークの活性度に比例して価格が変動しますので、取得前に確認するようにしてください。

ルートネームスペースを365日レンタルする場合の手数料を計算します。

https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Network-routes/operation/getRentalFees

```cs
var rentalFees = JsonNode.Parse(await GetDataFromApi(node, $"/network/fees/rental"));
var rootNsperBlock = int.Parse((string)rentalFees["effectiveRootNamespaceRentalFeePerBlock"]);
const int rentalDays = 365;
const int rentalBlock = rentalDays * 24 * 60 * 60 / 30;
var rootNsRentalFeeTotal = rentalBlock * rootNsperBlock;
Console.WriteLine($"rentalBlock: {rentalBlock}");
Console.WriteLine($"rootNsRentalFeeTotal: {rootNsRentalFeeTotal}");
```
###### 出力例
```cs
> rentalBlock:1051200
> rootNsRenatalFeeTotal:210240000 //約210XYM
```

期間はブロック数で指定します。1ブロックを30秒として計算しました。
最低で30日分はレンタルする必要があります（最大で1825日分）。

サブネームスペースの取得手数料を計算します。

```cs
var childNamespaceRentalFee = int.Parse((string)rentalFees["effectiveChildNamespaceRentalFee"]);
Console.WriteLine($"childNamespaceRentalFee: {childNamespaceRentalFee}");
```
###### 出力例
```cs
> childNamespaceRentalFee: 10000000 //10XYM
```

サブネームスペースに期間指定はありません。ルートネームスペースをレンタルしている限り使用できます。

## 6.2 レンタル

ルートネームスペースをレンタルします(例:xembook)
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook");
var tx = new NamespaceRegistrationTransactionV1()
{
    Network = NetworkType.TESTNET,
    Id = new NamespaceId(namespaceId),
    Name = Converter.Utf8ToBytes("xembook"),
    RegistrationType = NamespaceRegistrationType.ROOT,
    Duration = new BlockDuration(86400),
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp) //Deadline:有効期限
};
TransactionHelper.SetMaxFee(tx, 100);

var signature = facade.SignTransaction(aliceKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var hash = facade.HashTransaction(tx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```

サブネームスペースをレンタルします(例:xembook.tomato)
```cs
var parentId = IdGenerator.GenerateNamespaceId("xembook");
var namespaceId = IdGenerator.GenerateNamespaceId("tomato", parentId);
var subNamespaceTx = new NamespaceRegistrationTransactionV1()
{
    Network = NetworkType.TESTNET,
    ParentId = new NamespaceId(parentId),
    Id = new NamespaceId(namespaceId),
    Name = Converter.Utf8ToBytes("tomato"),
    RegistrationType = NamespaceRegistrationType.CHILD,
    Duration = new BlockDuration(86400),
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp) //Deadline:有効期限
};
TransactionHelper.SetMaxFee(subNamespaceTx, 100);

var signature = facade.SignTransaction(aliceKeyPair, subNamespaceTx);
var payload = TransactionsFactory.AttachSignature(subNamespaceTx, signature);
var hash = facade.HashTransaction(subNamespaceTx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```

2階層目のサブネームスペースを作成したい場合は
例えば、xembook.tomato.morningを定義したい場合は以下のようにします。

```cs
var parentId = IdGenerator.GenerateNamespaceId("xembook.tomato"); //紐づけたいルートネームスペース
var namespaceId = IdGenerator.GenerateNamespaceId("morning", parentId); //作成するサブネームスペース
var subNamespaceTx = new NamespaceRegistrationTransactionV1()
{
    Network = NetworkType.TESTNET,
    ParentId = new NamespaceId(parentId),
    Id = new NamespaceId(namespaceId),
    Name = Converter.Utf8ToBytes("morning"),
    RegistrationType = NamespaceRegistrationType.CHILD,
    Duration = new BlockDuration(86400),
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp) //Deadline:有効期限
};
```

### 有効期限の計算

レンタル済みルートネームスペースの有効期限を計算します。

namespace<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Namespace-routes/operation/getNamespace

chain<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Chain-routes/operation/getChainInfo

block<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Block-routes

```cs
var namespaceInfo = JsonNode.Parse(await GetDataFromApi(node, $"/namespaces/{namespaceId:X16}"));
var endHeight = ulong.Parse((string)namespaceInfo["namespace"]["endHeight"]);

var chainInfo = JsonNode.Parse(await GetDataFromApi(node, "/chain/info"));
var lastHeight = ulong.Parse((string)chainInfo["height"]);

var blockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{lastHeight}"));
var timestamp = ulong.Parse((string)blockInfo["block"]["timestamp"]);

var remainHeight = endHeight - lastHeight;

if (Network.TestNet.epocTime == null) throw new NullReferenceException("epocTime is null");
var epochAdjustment = (ulong)new DateTimeOffset(Network.TestNet.epocTime.Value).ToUnixTimeSeconds();
var endDateTimeOffset = DateTimeOffset.FromUnixTimeMilliseconds((long)(timestamp + remainHeight * 30000 + epochAdjustment * 1000));
var endDate = endDateTimeOffset.DateTime;

Console.WriteLine(endDate.ToString("yyyy/MM/dd HH:mm:ss", CultureInfo.InvariantCulture));
```

ネームスペース情報の終了ブロックを取得し、現在のブロック高から差し引いた残ブロック数に30秒(平均ブロック生成間隔)を掛け合わせた日時を出力します。
テストネットでは設定した有効期限よりも1日程度更新期限が猶予されます。メインネットはこの値が30日となっていますのでご留意ください

###### 出力例
```cs
> 2023/05/12 13:33:43
```
## 6.3 リンク

### アカウントへのリンク
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook");
var address = Converter.StringToAddress("TA5LGYEWS6L2WYBQ75J2DGK7IOZHYVWFWRLOFWI");
var tx = new AddressAliasTransactionV1()
{
    Network = NetworkType.TESTNET,
    NamespaceId = new NamespaceId(namespaceId),
    Address = new Address(address),
    AliasAction = AliasAction.LINK,
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(tx, 100);

var signature = facade.SignTransaction(aliceKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var hash = facade.HashTransaction(tx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```
リンク先のアドレスは自分が所有していなくても問題ありません。
※但し既にネットワークに認識されているアドレスに限ります。

### モザイクへリンク
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook.tomato");

var tx = new MosaicAliasTransactionV1()
{
    Network = NetworkType.TESTNET,
    NamespaceId = new NamespaceId(namespaceId),
    MosaicId = new MosaicId(0x5E033AC6CE11E654),
    AliasAction = AliasAction.LINK,
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(tx, 100);

var signature = facade.SignTransaction(aliceKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var hash = facade.HashTransaction(tx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```
モザイクを作成したアドレスと同一の場合のみリンクできるようです。

※この時、MosaicIdの引数はulong型になります。
そのため`Convert.ToUInt64("5E033AC6CE11E654", 16);`のように16進数文字列をulong型にコンバートするか`0x5E033AC6CE11E654`のように指定してください。

## 6.4 未解決で使用

送信先にUnresolvedAccountとして指定して、アドレスを特定しないままトランザクションを署名・アナウンスします。
チェーン側で解決されたアカウントに対しての送信が実施されます。
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook");
var tx = new TransferTransactionV1
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(Converter.AliasToRecipient(namespaceId, Network.TestNet.Identifier)), //UnresolvedAccount:未解決アカウントアドレス
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(tx, 100);
```
送信モザイクにUnresolvedMosaicとして指定して、モザイクIDを特定しないままトランザクションを署名・アナウンスします。

```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook.tomato");
var tx = new TransferTransactionV1
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Mosaics = new []
    {
        new UnresolvedMosaic()
        {
            MosaicId = new UnresolvedMosaicId(namespaceId), //UnresolvedMosaic:未解決モザイク
            Amount = new Amount(1) //送信量
        }
    },
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(tx, 100);
```

XYMをネームスペースで使用する場合は以下のように指定します。

```cs
var namespaceId = IdGenerator.GenerateNamespaceId("symbol.xym");
```

namespaceIdはulongですが、以下のように16進数文字列に変換することも可能です
```cs
namespaceId.ToString("X8")
```

## 6.5 参照

アドレスへリンクしたネームスペースの参照します
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook");
var namespaceInfo = JsonNode.Parse(await GetDataFromApi(node, $"/namespaces/{namespaceId:X8}"));
Console.WriteLine($"NamespaceInfo: {namespaceInfo}");
```
###### 出力例
```cs
> NamespaceInfo: {
  "meta": {
    "index": 0,
    "active": true
  },
  "namespace": {
    "version": 1,
    "registrationType": 0,
    "depth": 1,
    "level0": "DE3D3C9CD38EBCFF",
    "alias": {
      "type": 2,
      "address": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9"
    },
    "parentId": "0000000000000000",
    "ownerAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
    "startHeight": "371665",
    "endHeight": "460945"
  },
  "id": "64364422D7D26E76F92946C9"
}
```

aliasは以下の通りです。
```cs
{0: 'None', 1: 'Mosaic', 2: 'Address'}
```

registrationTypeは以下の通りです。
```cs
{0: 'RootNamespace', 1: 'SubNamespace'}
```

モザイクへリンクしたネームスペースを参照します。
```cs
var namespaceId = IdGenerator.GenerateNamespaceId("xembook.tomato");
var namespaceInfo = JsonNode.Parse(await GetDataFromApi(node, $"/namespaces/{namespaceId:X8}"));
Console.WriteLine($"NamespaceInfo: {namespaceInfo}");
```
###### 出力例
```cs
NamespaceInfo: {
  "meta": {
    "index": 0,
    "active": true
  },
  "namespace": {
    "version": 1,
    "registrationType": 1,
    "depth": 2,
    "level0": "DE3D3C9CD38EBCFF",
    "level1": "F2C38795AB40A6A0",
    "alias": {
      "type": 1,
      "mosaicId": "5E033AC6CE11E654"
    },
    "parentId": "DE3D3C9CD38EBCFF",
    "ownerAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
    "startHeight": "371665",
    "endHeight": "460945"
  },
  "id": "64364422D7D26E76F92946CB"
}

```

### 逆引き

アドレスに紐づけられたネームスペースを全て調べます。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Namespace-routes/operation/getAccountsNames

```cs
var obj = new Dictionary<string, string[]>()
{
    {
        "addresses", new[]
        {
            "TA5LGYEWS6L2WYBQ75J2DGK7IOZHYVWFWRLOFWI"
        }
    }
};

var namespaceInfo = JsonNode.Parse(await PostDataFromApi(node, $"/namespaces/account/names", obj));
foreach (var name in (IEnumerable)namespaceInfo["accountNames"][0]["names"])
{
    Console.WriteLine(name);
}
```

モザイクに紐づけられたネームスペースを全て調べます。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Namespace-routes/operation/getMosaicsNames
```cs
var obj = new Dictionary<string, string[]>()
{
    {
        "mosaicIds", new[]
        {
            "72C0212E67A08BCE"
        }
    }
};

var namespaceInfo = JsonNode.Parse(await PostDataFromApi(node, $"/namespaces/mosaic/names", obj));
foreach (var name in (IEnumerable)namespaceInfo["mosaicNames"][0]["names"])
{
    Console.WriteLine(name);
}
```


### レシートの参照

トランザクションに使用されたネームスペースをブロックチェーン側がどう解決したかを確認します。<br>

#### アドレス
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Receipt-routes/operation/searchAddressResolutionStatements

```cs
var state = JsonNode.Parse(await GetDataFromApi(node, $"/statements/resolutions/address?height=373690"));
Console.WriteLine(state);
```
###### 出力例
```js
{
  "data": [
    {
      "statement": {
        "height": "373690",
        "unresolved": "99FFBC8ED39C3C3DDE000000000000000000000000000000",
        "resolutionEntries": [
          {
            "source": {
              "primaryId": 1,
              "secondaryId": 0
            },
            "resolved": "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9"
          }
        ]
      },
      "id": "64365D88D7D26E76F92948B5",
      "meta": {
        "timestamp": "14034020645"
      }
    }
  ],
  "pagination": {
    "pageNumber": 1,
    "pageSize": 20
  }
}
```

#### モザイク
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Receipt-routes/operation/searchMosaicResolutionStatements

```cs
var state = JsonNode.Parse(await GetDataFromApi(node, $"/statements/resolutions/mosaic?height=373694"));
Console.WriteLine(state);
```
###### 出力例
```js
{
  "data": [
    {
      "statement": {
        "height": "373694",
        "unresolved": "F2C38795AB40A6A0",
        "resolutionEntries": [
          {
            "source": {
              "primaryId": 1,
              "secondaryId": 0
            },
            "resolved": "5E033AC6CE11E654"
          }
        ]
      },
      "id": "64365E1ED7D26E76F92948C1",
      "meta": {
        "timestamp": "14034170941"
      }
    }
  ],
  "pagination": {
    "pageNumber": 1,
    "pageSize": 20
  }
}
```

#### 注意事項
ネームスペースはレンタル制のため、過去のトランザクションで使用したネームスペースのリンク先と
現在のネームスペースのリンク先が異なる可能性があります。
過去のデータを参照する際などに、その時どのアカウントにリンクしていたかなどを知りたい場合は
必ずレシートを参照するようにしてください。

## 6.6 現場で使えるヒント

### 外部ドメインとの相互リンク

ネームスペースは重複取得がプロトコル上制限されているため、
インターネットドメインや実世界で周知されている商標名と同一のネームスペースを取得し、
外部(公式サイトや印刷物など)からネームスペース存在の認知を公表することで、
Symbol上のアカウントのブランド価値を構築することができます
(法的な効力については調整が必要です)。
外部ドメイン側のハッキングあるいは、Symbol側でのネームスペース更新忘れにはご注意ください。


#### ネームスペースを取得するアカウントについての注意
ネームスペースはレンタル期限という概念をもつ機能です。
今のところ、取得したネームスペースは放棄か延長の選択肢しかありません。
運用譲渡などが発生する可能性のあるシステムでネームスペース活用を検討する場合は
マルチシグ化(9章)したアカウントでネームスペースを取得することをおすすめします。

