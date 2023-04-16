# 11.制限

アカウントに対する制限とモザイクのグローバル制限についての方法を紹介します。
本章では、既存アカウントの権限を制限してしまうので、使い捨てのアカウントを新規に作成してお試しください。
※秘密鍵から復元するためにメモしておくことを忘れずに。

```cs
//使い捨てアカウントCarolの生成
var carolKeyPair = KeyPair.GenerateNewKeyPair();
Console.WriteLine(carolKeyPair.PrivateKey);
var carolAddress = facade.Network.PublicKeyToAddress(carolKeyPair.PublicKey);
Console.WriteLine(carolAddress);

//FAUCET URL出力
Console.WriteLine("https://testnet.symbol.tools/?recipient=" + carolAddress +"&amount=100");
```
## 11.1 アカウント制限

### 指定アドレスからの受信制限・指定アドレスへの送信制限
```cs
var tx = new AccountAddressRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.ADDRESS,AccountRestrictionFlags.BLOCK}.ToList().Select(flag => (int)flag.Value).Sum()), // Flag
    RestrictionAdditions = new UnresolvedAddress[] { new (bobAddress.bytes) },　//設定アドレス
    //解除アドレスは空
};
TransactionHelper.SetMaxFee(tx, 100);

var signature = facade.SignTransaction(carolKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

RestrictionFlagsについては以下の通りです。
```cs
{1: 'AllowIncomingAddress', 16385: 'AllowOutgoingAddress', 32769: 'BlockIncomingAddress', 49153: 'BlockOutgoingAddress'}
```
ただし実際に上記のようなものはSDKには存在せず以下のEnum、AccountRestrictionFlagsがあります。
```cs
{1: 'ADDRESS', 2: 'MOSAIC_ID', 4: 'TRANSACTION_TYPE', 16384: 'OUTGOING', 32768: 'BLOCK'}
```

この設定を解除する際は`RestrictionDeletions = new UnresolvedAddress[] { new (bobAddress.bytes) },`として同じFlagsでトランザクションを構築します。

AddressRestrictionFlagにはBlockIncomingAddressのほか、下記のようなフラグが使用できます。
- AllowIncomingAddress(1)：指定アドレスからのみ受信許可
- AllowOutgoingAddress(16385)：指定アドレス宛のみ送信許可
- BlockIncomingAddress(32769)：指定アドレスからの受信拒否
- BlockOutgoingAddress(49153)：指定アドレス宛への送信禁止

これらのRestrictionFlags指定は以下を参考にしてください。
```cs
// AllowIncomingAddress
RestrictionFlags = new AccountRestrictionFlags(AccountRestrictionFlags.ADDRESS.Value),
// AllowOutgoingAddress
RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.ADDRESS,AccountRestrictionFlags.OUTGOING}.ToList().Select(flag => (int)flag.Value).Sum()),
// BlockIncomingAddress
RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.ADDRESS,AccountRestrictionFlags.BLOCK}.ToList().Select(flag => (int)flag.Value).Sum()),
// BlockOutgoingAddress
RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.ADDRESS,AccountRestrictionFlags.OUTGOING,AccountRestrictionFlags.BLOCK}.ToList().Select(flag => (int)flag.Value).Sum()),
```

### 指定モザイクの受信制限
```cs
var tx = new AccountMosaicRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.MOSAIC_ID,AccountRestrictionFlags.BLOCK}.ToList().Select(flag => (int)flag.Value).Sum()), //モザイク制限フラグ
    RestrictionAdditions = new UnresolvedMosaicId[]{new (IdGenerator.GenerateNamespaceId("symbol.xym"))} //設定モザイク
};
TransactionHelper.SetMaxFee(tx, 100);

var signature = facade.SignTransaction(carolKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

MosaicRestrictionFlagについては以下の通りです。
```cs
{2: 'AllowMosaic', 32770: 'BlockMosaic'}
```

- AllowMosaic：指定モザイクを含むトランザクションのみ受信許可
- BlockMosaic：指定モザイクを含むトランザクションを受信拒否

```cs
// AllowMosaic
RestrictionFlags = new AccountRestrictionFlags(AccountRestrictionFlags.MOSAIC_ID.Value),
// BlockMosaic
RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.MOSAIC_ID,AccountRestrictionFlags.BLOCK}.ToList().Select(flag => (int)flag.Value).Sum()),
```

解除する際は同じく`RestrictionDeletions = new UnresolvedMosaicId[]{new (IdGenerator.GenerateNamespaceId("symbol.xym"))}`です。

モザイク送信の制限機能はありません。
また、後述するモザイクのふるまいを制限するグローバルモザイク制限と混同しないようにご注意ください。

### 指定トランザクションの送信制限

```cs
var tx = new AccountOperationRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.TRANSACTION_TYPE,AccountRestrictionFlags.OUTGOING}.ToList().Select(flag => (int)flag.Value).Sum()),
    RestrictionAdditions = new TransactionType[]{new (TransactionType.ACCOUNT_OPERATION_RESTRICTION.Value)} //設定トランザクション
};
TransactionHelper.SetMaxFee(tx, 100);

var signature = facade.SignTransaction(carolKeyPair, tx);
var payload = TransactionsFactory.AttachSignature(tx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

OperationRestrictionFlagについては以下の通りです。
```js
{16388: 'AllowOutgoingTransactionType', 49156: 'BlockOutgoingTransactionType'}
```

- AllowOutgoingTransactionType：指定トランザクションの送信のみ許可
- BlockOutgoingTransactionType：指定トランザクションの送信を禁止

```cs
// AllowOutgoingTransactionType
RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.TRANSACTION_TYPE,AccountRestrictionFlags.OUTGOING}.ToList().Select(flag => (int)flag.Value).Sum()),
// BlockOutgoingTransactionType
RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.TRANSACTION_TYPE,AccountRestrictionFlags.OUTGOING,AccountRestrictionFlags.BLOCK}.ToList().Select(flag => (int)flag.Value).Sum()),
```

トランザクション受信の制限機能はありません。指定できるオペレーションは以下の通りです。

TransactionTypeについては以下の通りです。
```cs
{16705: 'AGGREGATE_COMPLETE', 16707: 'VOTING_KEY_LINK', 16708: 'ACCOUNT_METADATA', 16712: 'HASH_LOCK', 16716: 'ACCOUNT_KEY_LINK', 16717: 'MOSAIC_DEFINITION', 16718: 'NAMESPACE_REGISTRATION', 16720: 'ACCOUNT_ADDRESS_RESTRICTION', 16721: 'MOSAIC_GLOBAL_RESTRICTION', 16722: 'SECRET_LOCK', 16724: 'TRANSFER', 16725: 'MULTISIG_ACCOUNT_MODIFICATION', 16961: 'AGGREGATE_BONDED', 16963: 'VRF_KEY_LINK', 16964: 'MOSAIC_METADATA', 16972: 'NODE_KEY_LINK', 16973: 'MOSAIC_SUPPLY_CHANGE', 16974: 'ADDRESS_ALIAS', 16976: 'ACCOUNT_MOSAIC_RESTRICTION', 16977: 'MOSAIC_ADDRESS_RESTRICTION', 16978: 'SECRET_PROOF', 17220: 'NAMESPACE_METADATA', 17229: 'MOSAIC_SUPPLY_REVOCATION', 17230: 'MOSAIC_ALIAS'}
```

##### 注意事項
17232: 'ACCOUNT_OPERATION_RESTRICTION' の制限は許可されていません。
つまり、AllowOutgoingTransactionTypeを指定する場合は、ACCOUNT_OPERATION_RESTRICTIONを必ず含める必要があり、
BlockOutgoingTransactionTypeを指定する場合は、ACCOUNT_OPERATION_RESTRICTIONを含めることはできません。


### 確認

設定した制限情報を確認します<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Restriction-Account-routes/operation/getAccountRestrictions

```cs
var restrictionsinfo = JsonNode.Parse(await GetDataFromApi(node, $"/restrictions/account/{carolAddress}"));
Console.WriteLine($"AccountRestrictions: {restrictionsinfo}");
```
###### 出力例
```cs
> AccountRestrictions: {
  "accountRestrictions": {
    "version": 1,
    "address": "984F5658989C315F34E5BFC72834E2D3C13D7236334DC6F7",
    "restrictions": [
      {
        "restrictionFlags": 1,
        "values": [
          "983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9"
        ]
      },
      {
        "restrictionFlags": 32770,
        "values": [
          "72C0212E67A08BCE"
        ]
      },
      {
        "restrictionFlags": 16388,
        "values": [
          17232
        ]
      }
    ]
  }
}

```

## 11.2 グローバルモザイク制限

グローバルモザイク制限はモザイクに対して送信可能な条件を設定します。  
その後、各アカウントに対してグローバルモザイク制限専用の数値メタデータを付与します。  
送信アカウント・受信アカウントの両方が条件を満たした場合のみ、該当モザイクを送信することができます。  

なお、ここで先ほどと同じCarolを使用する場合は現在は`TransactionType.ACCOUNT_OPERATION_RESTRICTION`のみが許可されているため、これから使用するトランザクションを許可するか新たにKeyPairを作成するなどの対応をしてください。

```cs
var tx = new AccountOperationRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    RestrictionFlags = new AccountRestrictionFlags((ushort)new[] {AccountRestrictionFlags.TRANSACTION_TYPE,AccountRestrictionFlags.OUTGOING}.ToList().Select(flag => (int)flag.Value).Sum()),
    RestrictionAdditions = new TransactionType[]{new (TransactionType.MOSAIC_DEFINITION.Value),new (TransactionType.MOSAIC_SUPPLY_CHANGE.Value),new (TransactionType.MOSAIC_GLOBAL_RESTRICTION.Value),new (TransactionType.AGGREGATE_COMPLETE.Value),new (TransactionType.MOSAIC_ADDRESS_RESTRICTION.Value),new (TransactionType.TRANSFER.Value)} //設定トランザクション
};
```

### グローバル制限機能つきモザイクの作成
restrictableをtrueにしてCarolでモザイクを作成します。

```cs
var supplyMutable = true; //供給量変更の可否
var transferable = true; //第三者への譲渡可否
var restrictable = true; //制限設定の可否
var revokable = true; //発行者からの還収可否

var nonce = BitConverter.ToUInt32(Crypto.RandomBytes(8), 0);
var mosaicId = IdGenerator.GenerateMosaicId(carolAddress, nonce);

//モザイク定義
var mosaicDefTx = new EmbeddedMosaicDefinitionTransactionV1()
{
    Network = NetworkType.TESTNET,
    Nonce = new MosaicNonce(nonce),
    SignerPublicKey = carolKeyPair.PublicKey,
    Id = new MosaicId(mosaicId),
    Duration = new BlockDuration(0), //duration
    Divisibility = 0, //divisibility
    Flags = new MosaicFlags(Converter.CreateMosaicFlags(supplyMutable, transferable, restrictable, revokable)),
};

//モザイク変更
var mosaicChangeTx = new EmbeddedMosaicSupplyChangeTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    MosaicId = new UnresolvedMosaicId(mosaicId),
    Action = MosaicSupplyChangeAction.INCREASE,
    Delta = new Amount(1000000),
};

//グローバルモザイク制限
var key = IdGenerator.GenerateUlongKey("KYC"); // restrictionKey 
var mosaicGlobalResTx = new EmbeddedMosaicGlobalRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    MosaicId = new UnresolvedMosaicId(mosaicId),
    RestrictionKey = key,
    NewRestrictionValue = 1,
    NewRestrictionType = MosaicRestrictionType.EQ,
};

var innerTransactions = new IBaseTransaction[] { mosaicDefTx, mosaicChangeTx, mosaicGlobalResTx };
var merkleHash = SymbolFacade.HashEmbeddedTransactions(innerTransactions);
var aggregateTx = new AggregateCompleteTransactionV2()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    Transactions = innerTransactions,
    TransactionsHash = merkleHash,
};
TransactionHelper.SetMaxFee(aggregateTx, 100);

var signature = facade.SignTransaction(carolKeyPair, aggregateTx);
var payload = TransactionsFactory.AttachSignature(aggregateTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

MosaicRestrictionTypeについては以下の通りです。

```cs
{0: 'NONE', 1: 'EQ', 2: 'NE', 3: 'LT', 4: 'LE', 5: 'GT', 6: 'GE'}
```

| 演算子  | 略称  | 英語  |
|---|---|---|
| =  | EQ  | equal to  |
| !=  | NE  | not equal to  |
| <  | LT  | less than  |
| <=  | LE  | less than or equal to  |
| >  | GT  | greater than  |
| <=  | GE  | greater than or equal to  |


### アカウントへのモザイク制限適用

Carol,Bobに対してグローバル制限モザイクに対しての適格情報を追加します。  
送信・受信についてかかる制限なので、すでに所有しているモザイクについての制限はありません。  
送信を成功させるためには、送信者・受信者双方が条件をクリアしている必要があります。  
モザイク作成者の秘密鍵があればどのアカウントに対しても承諾の署名を必要とせずに制限をつけることができます。  

```cs
//Carolに適用
var carolMosaicAddressResTx = new MosaicAddressRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    MosaicId = new UnresolvedMosaicId(0x388F6F55BE55BB6E), // mosaicId
    RestrictionKey = IdGenerator.GenerateUlongKey("KYC"), // restrictionKey
    TargetAddress = new UnresolvedAddress(carolAddress.bytes), // address
    NewRestrictionValue = 1, // newRestrictionValue
    PreviousRestrictionValue = 0xFFFFFFFFFFFFFFFF // previousRestrictionValue
};
TransactionHelper.SetMaxFee(carolMosaicAddressResTx, 100);

var signature1 = facade.SignTransaction(carolKeyPair, carolMosaicAddressResTx);
var payload1 = TransactionsFactory.AttachSignature(carolMosaicAddressResTx, signature1);
var result1 = await Announce(payload1);
Console.WriteLine(result1);

//Bobに適用
var bobMosaicAddressResTx = new MosaicAddressRestrictionTransactionV1()
{
    Network = NetworkType.TESTNET,
    SignerPublicKey = carolKeyPair.PublicKey,
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp),
    MosaicId = new UnresolvedMosaicId(0x388F6F55BE55BB6E), // mosaicId
    RestrictionKey = IdGenerator.GenerateUlongKey("KYC"), // restrictionKey
    TargetAddress = new UnresolvedAddress(bobAddress.bytes), // address
    NewRestrictionValue = 1, // newRestrictionValue
    PreviousRestrictionValue = 0xFFFFFFFFFFFFFFFF // previousRestrictionValue
};
TransactionHelper.SetMaxFee(bobMosaicAddressResTx, 100);

var signature2 = facade.SignTransaction(carolKeyPair, bobMosaicAddressResTx);
var payload2 = TransactionsFactory.AttachSignature(bobMosaicAddressResTx, signature2);
var result2 = await Announce(payload2);
Console.WriteLine(result2);
```

### 制限状態確認

ノードに問い合わせて制限状態を確認します。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Restriction-Mosaic-routes/operation/searchMosaicRestrictions

```cs
var mosaicRestrictionsInfo = JsonNode.Parse(await GetDataFromApi(node, $"/restrictions/mosaic?mosaicId=388F6F55BE55BB6E"));
Console.WriteLine($"MosaicRestrictions: {mosaicRestrictionsInfo}");
```

###### 出力例
```cs
> MosaicRestrictions: {
  "data": [
    {
      "mosaicRestrictionEntry": {
        "version": 1,
        "compositeHash": "2276FFC8D893A8D3B1A4C99911E2E426361AFA5A7B0D6086F9AC51D858C371AF",
        "entryType": 1,
        "mosaicId": "388F6F55BE55BB6E",
        "restrictions": [
          {
            "key": "9300605567124626807",
            "restriction": {
              "referenceMosaicId": "0000000000000000",
              "restrictionValue": "1",
              "restrictionType": 1
            }
          }
        ]
      },
      "id": "643A4B197AF243DEB65DAE8C"
    },
    {
      "mosaicRestrictionEntry": {
        "version": 1,
        "compositeHash": "05AF65CB9ADC6DE0A69872B13AF5C980976AF0B63E6FE859B42C745FFE4BA9D1",
        "entryType": 0,
        "mosaicId": "388F6F55BE55BB6E",
        "targetAddress": "984F5658989C315F34E5BFC72834E2D3C13D7236334DC6F7",
        "restrictions": [
          {
            "key": "9300605567124626807",
            "value": "1"
          }
        ]
      },
      "id": "643A4FC57AF243DEB65DB58D"
    },
  ...
```

### 送信確認

実際にモザイクを送信してみて、制限状態を確認します。

```cs
//成功（CarolからBobに送信）
var trTx = new TransferTransactionV1
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(bobAddress.bytes),
    SignerPublicKey = carolKeyPair.PublicKey,
    Mosaics = new UnresolvedMosaic[]
    {
        new ()
        {
            MosaicId = new UnresolvedMosaicId(0x388F6F55BE55BB6E),
            Amount = new Amount(1)
        }
    },
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(trTx, 100); 
var signature = facade.SignTransaction(carolKeyPair, trTx);
var payload = TransactionsFactory.AttachSignature(trTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);

//失敗（CarolからDaveに送信）
var dave = KeyPair.GenerateNewKeyPair();
var daveAddress = facade.Network.PublicKeyToAddress(dave.PublicKey);

var trTx = new TransferTransactionV1
{
    Network = NetworkType.TESTNET,
    RecipientAddress = new UnresolvedAddress(daveAddress.bytes),
    SignerPublicKey = carolKeyPair.PublicKey,
    Mosaics = new UnresolvedMosaic[]
    {
        new ()
        {
            MosaicId = new UnresolvedMosaicId(0x388F6F55BE55BB6E),
            Amount = new Amount(1)
        }
    },
    Deadline = new Timestamp(facade.Network.FromDatetime<NetworkTimestamp>(DateTime.UtcNow).AddHours(2).Timestamp)
};
TransactionHelper.SetMaxFee(trTx, 100); 
var signature = facade.SignTransaction(carolKeyPair, trTx);
var payload = TransactionsFactory.AttachSignature(trTx, signature);
var result = await Announce(payload);
Console.WriteLine(result);
```

失敗した場合以下のようなエラーステータスになります。

```js
{"hash":"E3402FB7AE21A6A64838DDD0722420EC67E61206C148A73B0DFD7F8C098062FA","code":"Failure_RestrictionMosaic_Account_Unauthorized","deadline":"12371602742","group":"failed"}
```

## 11.3 現場で使えるヒント

ブロックチェーンの社会実装などを考えたときに、法律や信頼性の見地から
一つの役割のみを持たせたいアカウント、関係ないアカウントを巻き込みたくないと思うことがあります。
そんな場合にアカウント制限とグローバルモザイク制限を使いこなすことで、
モザイクのふるまいを柔軟にコントロールすることができます。

### アカウントバーン

AllowIncomingAddressによって指定アドレスからのみ受信可能にしておいて、  
XYMを全量送信すると、秘密鍵を持っていても自力では操作困難なアカウントを明示的に作成することができます。  
（最小手数料を0に設定したノードによって承認されることもあり、その可能性はゼロではありません）  

### モザイクロック
譲渡不可設定のモザイクを配布し、配布者側のアカウントで受け取り拒否を行うとモザイクをロックさせることができます。

### 所属証明
モザイクの章で所有の証明について説明しました。グローバルモザイク制限を活用することで、
KYCが済んだアカウント間でのみ所有・流通させることが可能なモザイクを作り、所有者のみが所属できる独自経済圏を構築することが可能です。


