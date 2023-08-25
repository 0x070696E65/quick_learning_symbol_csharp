# 12.オフライン署名

ロック機構の章で、アナウンスしたトランザクションをハッシュ値指定でロックして、  
複数の署名（オンライン署名）を集めるアグリゲートトランザクションを紹介しました。    
この章では、トランザクションを事前に署名を集めてノードにアナウンスするオフライン署名について説明します。  

## 手順

Aliceが起案者となりトランザクションを作成し、署名します。  
次にBobが署名してAliceに返します。  
最後にAliceがトランザクションを結合してネットワークにアナウンスします。  


## 12.1 トランザクション作成
```cs
var innerTx1 = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),
    SignerPublicKey = alicePublicKey,
    Message = Converter.Utf8ToPlainMessage("tx1"),
};
var innerTx2 = new EmbeddedTransferTransactionV1()
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(aliceAddress.bytes),
    SignerPublicKey = bobPublicKey,
    Message = Converter.Utf8ToPlainMessage("tx2"),
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
TransactionHelper.SetMaxFee(aggregateTx, 100, 1);

var signature = facade.SignTransaction(aliceKeyPair, aggregateTx);
TransactionsFactory.AttachSignature(aggregateTx, signature);
var signedPayload = TransactionHelper.GetPayload(aggregateTx);
Console.WriteLine(signedPayload);
```
###### 出力例
```cs
>5801000000000000F95F520DF1E08FA936F3C57E7150062BD8710FB8290F5076540DFBFB22D22B32AED865F591C5AFA06D3A3A4208317973AE4CA24988C6C34EACE44B62EA40700C13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F000000000298414100AF000000000000272F7B5403000000E1B817E8E6186A67925A0C02DC514689F6684845E8F5A6564F4E7C37072E2D84B000000000000000540000000000000013B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F0000000001985441983AB360969797AB6030FF53A1995F43B27C56C5B456E2D90400000000000000007478310000000054000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B0000000001985441982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF04000000000000000074783200000000
```

署名を行い、signedPayloadを出力します。signedPayloadをBobに渡して署名を促します。  

## 12.2 Bobによる連署


Aliceから受け取ったsignedPayloadでトランザクションを復元します。

```cs
var signedPayload =
    "5801000000000000F95F520DF1E08FA936F3C57E7150062BD8710FB8290F5076540DFBFB22D22B32AED865F591C5AFA06D3A3A4208317973AE4CA24988C6C34EACE44B62EA40700C13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F000000000298414100AF000000000000272F7B5403000000E1B817E8E6186A67925A0C02DC514689F6684845E8F5A6564F4E7C37072E2D84B000000000000000540000000000000013B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F0000000001985441983AB360969797AB6030FF53A1995F43B27C56C5B456E2D90400000000000000007478310000000054000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B0000000001985441982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF04000000000000000074783200000000";
var tx = TransactionFactory.Deserialize(signedPayload);
Console.WriteLine(tx);
```
###### 出力例
```cs
> (signature: F95F520DF1E08FA936F3C57E7150062BD8710FB8290F5076540DFBFB22D22B32AED865F591C5AFA06D3A3A4208317973AE4CA24988C6C34EACE44B62EA40700C, signerPublicKey: 13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F, version: 0x2, network: NetworkType.TESTNET, type: TransactionType.AGGREGATE_COMPLETE, fee: 0x000000000000AF00, deadline: 0x00000003547B2F27, transactionsHash: E1B817E8E6186A67925A0C02DC514689F6684845E8F5A6564F4E7C37072E2D84, transactions: [(signerPublicKey: 13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F, version: 0x1, network: NetworkType.TESTNET, type: TransactionType.TRANSFER, recipientAddress: 983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9, mosaics: [], message: hex(00747831), ),(signerPublicKey: 4C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B, version: 0x1, network: NetworkType.TESTNET, type: TransactionType.TRANSFER, recipientAddress: 982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF, mosaics: [], message: hex(00747832), )], cosignatures: [], )
```

念のため、Aliceがすでに署名したトランザクション（ペイロード）かどうかを検証します。
```cs
var res = facade.VerifyTransaction(tx, tx.Signature);
Console.WriteLine(res);
```
###### 出力例
```cs
> True
```

ペイロードがsigner、つまりAliceによって署名されたものであることが確認できました。
次にBobが連署します。
```cs
var hash = facade.HashTransaction(tx, tx.Signature);
var cosignature = new Cosignature(){
    Signature = bobKeyPair.Sign(hash.bytes),
    SignerPublicKey = bobPublicKey
};
Console.WriteLine(Converter.BytesToHex(cosignature.Serialize()));
```
###### 出力例
```cs
> 00000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B42C6293F8E8C2490260A452E2E8493D2CF03F733AD787C500941CC0043BCFE8E577D8480588094EB878696122809F8BB6CC285F6CB7E959733A1E2BBF8860B0B
```

hashに対して署名を行い、Cosignatureクラスの生成、シリアライズして16進数文字列で返します。
Bobが全ての署名を揃えられる場合は、Aliceに返却しなくてもBobがアナウンスすることも可能です。

## 12.3 アナウンス

AliceはBobからcosignatureHexを受け取ります。
また事前にAlice自身で作成したsignedPayloadを用意します。  

```cs
var signedPayload = "5801000000000000F95F520DF1E08FA936F3C57E7150062BD8710FB8290F5076540DFBFB22D22B32AED865F591C5AFA06D3A3A4208317973AE4CA24988C6C34EACE44B62EA40700C13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F000000000298414100AF000000000000272F7B5403000000E1B817E8E6186A67925A0C02DC514689F6684845E8F5A6564F4E7C37072E2D84B000000000000000540000000000000013B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F0000000001985441983AB360969797AB6030FF53A1995F43B27C56C5B456E2D90400000000000000007478310000000054000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B0000000001985441982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF04000000000000000074783200000000";
var tx = TransactionFactory.Deserialize(signedPayload);
var cosignatureHex = "00000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B42C6293F8E8C2490260A452E2E8493D2CF03F733AD787C500941CC0043BCFE8E577D8480588094EB878696122809F8BB6CC285F6CB7E959733A1E2BBF8860B0B";
var cosignature = TransactionHelper.CosignatureDeserializer(cosignatureHex);
(tx as AggregateCompleteTransactionV2)!.Cosignatures = new[] {cosignature};

var payload = TransactionsFactory.CreatePayload(tx);
var result = await Announce(payload);
Console.WriteLine(result);
```

## 12.4 現場で使えるヒント

### マーケットプレイスレス
ボンデッドトランザクションと異なりハッシュロックのための手数料(10XYM)を気にする必要がありません。    
ペイロードを共有できる場が存在する場合、売り手は考えられるすべての買い手候補に対してペイロードを作成して交渉開始を待つことができます。
（複数のトランザクションが個別に実行されないように、1つしか存在しない領収書NFTをアグリゲートトランザクションに混ぜ込むなどして排他制御をしてください）。
この交渉に専用のマーケットプレイスを構築する必要はありません。
SNSのタイムラインをマーケットプレイスにしたり、必要に応じて任意の時間や空間でワンタイムマーケットプレイスを展開することができます。  

ただ、オフラインで署名を交換するため、なりすましのハッシュ署名要求には気を付けましょう。  
（必ず検証可能なペイロードからハッシュを生成して署名するようにしてください）  
