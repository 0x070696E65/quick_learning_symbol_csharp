# 5.モザイク

本章ではモザイクの設定とその生成方法について解説します。
Symbolではトークンのことをモザイクと表現します。

> Wikipediaによると、トークンとは「紀元前8000年頃から紀元前3000年までのメソポタミアの地層から出土する直径が1cm前後の粘土で作られたさまざまな形状の物体」のことを指します。一方でモザイクとは「小片を寄せあわせ埋め込んで、絵（図像）や模様を表す装飾美術の技法。石、陶磁器（モザイクタイル）、有色無色のガラス、貝殻、木などが使用され、建築物の床や壁面、あるいは工芸品の装飾のために施される。」とあります。SymbolにおいてモザイクとはSymbolが作りなすエコシステムの様相を表すさまざまな構成要素、と考えることができます。

## 5.1 モザイク生成

モザイク生成には
作成するモザイクを定義します。
```cs
var supplyMutable = true; //供給量変更の可否
var transferable = false; //第三者への譲渡可否
var restrictable = true; //制限設定の可否
var revokable = true; //発行者からの還収可否

//モザイク定義
var nonce = BitConverter.ToUInt32(Crypto.RandomBytes(8), 0);
var mosaicId = IdGenerator.GenerateMosaicId(aliceAddress, nonce);

var mosaicDefTx = new EmbeddedMosaicDefinitionTransactionV1()
{
    Network = NetworkType.TESTNET,
    Nonce = new MosaicNonce(nonce),
    SignerPublicKey = alicePublicKey,
    Id = new MosaicId(mosaicId),
    Duration = new BlockDuration(0),
    Divisibility = 2,
    Flags = new MosaicFlags(Converter.CreateMosaicFlags(supplyMutable, transferable, restrictable, revokable)),
};
```

MosaicFlagsは以下の通りです。

```js
MosaicFlags {
  supplyMutable: false, transferable: false, restrictable: false, revokable: false
}
```
数量変更、第三者への譲渡、モザイクグローバル制限の適用、発行者からの還収の可否について指定します。
この項目は後で変更することはできません。

#### divisibility:可分性

可分性は小数点第何位まで数量の単位とするかを決めます。データは整数値として保持されます。

divisibility:0 = 1  
divisibility:1 = 1.0  
divisibility:2 = 1.00  

#### duration:有効期限

0を指定した場合、無期限に使用することができます。
モザイク有効期限を設定した場合、期限が切れた後も消滅することはなくデータとしては残ります。
アカウント1つにつき1000までしか所有することはできませんのでご注意ください。


次に数量を変更します
```cs
//モザイク変更
var mosaicChangeTx = new EmbeddedMosaicSupplyChangeTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    MosaicId = new UnresolvedMosaicId(mosaicId),
    Action = MosaicSupplyChangeAction.INCREASE,
    Delta = new Amount(1000000),
};
```
supplyMutable:falseの場合、全モザイクが発行者にある場合だけ数量の変更が可能です。
divisibility > 0 の場合は、最小単位を1として整数値で定義してください。
（divisibility:2 で 1.00 作成したい場合は100と指定）

MosaicSupplyChangeActionは以下の通りです。
```cs
{0: 'Decrease', 1: 'Increase'}
```
増やしたい場合はIncreaseを指定します。
上記2つのトランザクションをまとめてアグリゲートトランザクションを作成します。

```cs
var innerTransactions = new IBaseTransaction[] { mosaicDefTx, mosaicChangeTx };
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

アグリゲートトランザクションの特徴として、
まだ存在していないモザイクの数量を変更しようとしている点に注目してください。
配列化した時に、矛盾点がなければ1つのブロック内で問題なく処理することができます。

### 確認
作成したモザイク情報を確認します。

https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Mosaic-routes/operation/getMosaic

```cs
var mosaic = JsonNode.Parse( await GetDataFromApi(node, $"/mosaics/{mosaicId:X8}"));
Console.WriteLine($"Mosaic: {mosaic}");
```
###### 出力例
```js
> Mosaic: {
  "mosaic": {
    "version": 1,
    "id": "780EFB7E05B64285",
    "supply": "1000000",
    "startHeight": "371149",
    "ownerAddress": "982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF",
    "revision": 1,
    "flags": 13,
    "divisibility": 2,
    "duration": "0"
  },
  "id": "6435204D5ED31320FDCE1D55"
}
```

## 5.2 モザイク送信

作成したモザイクを送信します。
よく、ブロックチェーンに初めて触れる方は、
モザイク送信について「クライアント端末に保存されたモザイクを別のクライアント端末へ送信」することとイメージされている人がいますが、
モザイク情報はすべてのノードで常に共有・同期化されており、送信先に未知のモザイク情報を届けることではありません。
正確にはブロックチェーンへ「トランザクションを送信」することにより、アカウント間でのトークン残量を組み替える操作のことを言います。

```cs
var tx = new TransferTransactionV1
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Mosaics = new UnresolvedMosaic[]
    {
        new ()
          {
              MosaicId = new UnresolvedMosaicId(0x72C0212E67A08BCE), //テストネットXYM
              Amount = new Amount(1000000) //1XYM(divisibility:6)
          },
        new ()
        {
            MosaicId = new UnresolvedMosaicId(0x780EFB7E05B64285), // 5.1 で作成したモザイク
            Amount = new Amount(1) // 数量:0.01(divisibility:2 の場合)
        }
    },
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp) //Deadline:有効期限
};
tx.Sort();
TransactionHelper.SetMaxFee(tx, 100); //手数料

var signature = facade.SignTransaction(aliceKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var hash = facade.HashTransaction(tx, signature);
Console.WriteLine(hash);
var result = await Announce(payload);
Console.WriteLine(result);
```
※一つのトランザクションで複数のモザイクを送信する場合は`tx.Sort();`が必要です。これはモザイクIDが昇順で無ければいけないというルールのためです。

##### 送信モザイクリスト

複数のモザイクを一度に送信できます。
XYMを送信するには以下のモザイクIDを指定します。
- メインネット：6BED913FA20223F8
- テストネット：72C0212E67A08BCE

#### 送信量
小数点もすべて整数にして指定します。
XYMは可分性6なので、1XYM=1000000で指定します。

### 送信確認

```cs
var transaction = JsonNode.Parse(await GetDataFromApi(node, $"/transactions/confirmed/{hash}"));
Console.WriteLine($"Transaction: {transaction}");
```
###### 出力例
```cs
> Transaction: {
  "meta": {
    "height": "371175",
    "hash": "E234822F69B786818BDAC4B9E80960441361D5E910A1C7D59E66B56D93096D75",
    "merkleComponentHash": "E234822F69B786818BDAC4B9E80960441361D5E910A1C7D59E66B56D93096D75",
    "index": 0,
    "timestamp": "13953669388",
    "feeMultiplier": 100
  },
  "transaction": {
    "size": 192,
    "signature": "B5F16E03910042BE7863F3C37B7CA50EE4C095B6A0E041ABB3F8368A3D4C948CE5E283C782EC540C40CC3DB05B8DAEDD9971AF85F94E932747CCD0B7DD29F00D",
    "signerPublicKey": "13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F",
    "version": 1,
    "network": 152,
    "type": 16724,
    "maxFee": "19200",
    "deadline": "13960857981",
    "recipientAddress": "9864F4D6958183958449BE499B59236238022F3985BD6B47",
    "mosaics": [
      {
        "id": "72C0212E67A08BCE",
        "amount": "1000000"
      },
      {
        "id": "780EFB7E05B64285",
        "amount": "1"
      }
    ]
  },
  "id": "643523A9D7D26E76F9292C72"
}

```
TransferTransactionのmosaicsに2種類のモザイクが送信されていることが確認できます。また、承認されたブロックの情報が記載されています。

## 5.3 現場で使えるヒント

### 所有証明

前章でトランザクションによる存在証明について説明しました。
アカウントの作成した送信指示が消せない形で残せるので、絶対につじつまの合う台帳を作ることができます。
すべてのアカウントの「絶対に消せない送信指示」の蓄積結果として、各アカウントは自分のモザイク所有を証明することができます。
（本ドキュメントでは所有を「自分の意思で手放すことができる状態」とします。少し話題がそれますが、法律的にはデジタルデータに所有権が認められていないのも、一度知ってしまったデータは自分の意志では忘れたことを他人に証明することができない点に注目すると「手放すことができる状態」の意味に納得がいくかもしれません。ブロックチェーンによりそのデータの放棄を明確に示すことができるのですが、詳しくは法律の専門の方にお任せします。）

#### NFT(non fungible token)

発行枚数を1に限定し、supplyMutableをfalseに設定することで、1つだけしか存在しないトークンを発行できます。
モザイクは作成したアカウントアドレスを改ざんできない情報として保有しているので、
そのアカウントの送信トランザクションをメタ情報として利用できます。
7章で説明するメタデータをモザイクに登録する方法もありますが、その方法は登録アカウントとモザイク作成者の連署によって更新可能なことにご注意ください。

NFTの実現方法はいろいろありますが、その一例の処理概要を以下に例示します（実行するためにはnonceやフラグ情報を適切に設定してください）。
```cs
var supplyMutable = false; //供給量変更の可否

//モザイク定義
var mosaicDefTx = new EmbeddedMosaicDefinitionTransactionV1()
{
    Network = NetworkType.TESTNET,
    Nonce = new MosaicNonce(nonce),
    SignerPublicKey = alicePublicKey,
    Id = new MosaicId(mosaicId),
    Duration = new BlockDuration(0),  //duration:無期限
    Divisibility = 0, //divisibility:可分性
    Flags = new MosaicFlags(Converter.CreateMosaicFlags(supplyMutable, transferable, restrictable, revokable)),
};

//モザイク数量固定
var mosaicChangeTx = new EmbeddedMosaicSupplyChangeTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    MosaicId = new UnresolvedMosaicId(mosaicId),
    Action = MosaicSupplyChangeAction.INCREASE,  //増やす
    Delta = new Amount(1),  //数量1
};

//NFTデータ
var nftTx = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(aliceAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Message = Converter.Utf8ToPlainMessage("Hello Symbol!")  //NFTデータ実体
};

//モザイクの生成とNFTデータをアグリゲートしてブロックに登録
var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = alicePublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
};
TransactionHelper.SetMaxFee(aggregateTx, 100);
```

モザイク生成時のブロック高と作成アカウントがモザイク情報に含まれているので同ブロック内のトランザクションを検索することにより、
紐づけられたNFTデータを取得することができます。

##### 注意事項
モザイクの作成者が全数量を所有している場合、供給量を変更することが可能です。
またトランザクションに分割してデータを記録した場合、改ざんできませんがデータの追記は可能です。
NFTを運用する場合はモザイク作成者の秘密鍵を厳重に管理・あるいは破棄するなど、適切な運用にご注意ください。


#### 回収可能なポイント運用

transferableをfalseに設定することで転売が制限されるため、資金決済法の影響を受けにくいポイントを定義することができます。
またrevokableをtrueに設定することで、ユーザ側が秘密鍵を管理しなくても使用分を回収できるような中央管理型のポイント運用を行うことができます。

```cs
var transferable = false; //第三者への譲渡可否
var revokable = true; //発行者からの還収可否
```

トランザクションは以下のように記述します。

```cs
var revocationTx = new MosaicSupplyRevocationTransactionV1()
{
    Network = NetworkType.TESTNET,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp), //Deadline:有効期限
    SourceAddress = new UnresolvedAddress(bobAddress.bytes), //回収先アドレス
    Mosaic = new UnresolvedMosaic()
    {
        MosaicId = new UnresolvedMosaicId(mosaicId),
        Amount = new Amount(3)
    }
};
TransactionHelper.SetMaxFee(revocationTx, 100);
```




