# 13.検証
ブロックチェーン上に記録されたさまざまな情報を検証します。
ブロックチェーンへのデータ記録は全ノードの合意を持って行われますが、
ブロックチェーンへの**データ参照**はノード単体からの情報取得であるため、
信用できないノードの情報を元にして新たな取引を行いたい場合は、ノードから取得したデータに対して検証を行う必要があります。


## 13.1 トランザクションの検証

トランザクションがブロックヘッダーに含まれていることを検証します。この検証が成功すれば、トランザクションがブロックチェーンの合意によって承認されたものとみなすことができます。

### 検証するペイロード

今回検証するトランザクションペイロードとそのトランザクションが記録されているとされるブロック高です。

```cs
var payload = "C001000000000000F95F520DF1E08FA936F3C57E7150062BD8710FB8290F5076540DFBFB22D22B32AED865F591C5AFA06D3A3A4208317973AE4CA24988C6C34EACE44B62EA40700C13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F000000000298414100AF000000000000272F7B5403000000E1B817E8E6186A67925A0C02DC514689F6684845E8F5A6564F4E7C37072E2D84B000000000000000540000000000000013B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F0000000001985441983AB360969797AB6030FF53A1995F43B27C56C5B456E2D90400000000000000007478310000000054000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B0000000001985441982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF0400000000000000007478320000000000000000000000004C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B42C6293F8E8C2490260A452E2E8493D2CF03F733AD787C500941CC0043BCFE8E577D8480588094EB878696122809F8BB6CC285F6CB7E959733A1E2BBF8860B0B";
var height = 381969;
```


### payload確認

トランザクションの内容を確認します。

```cs
var tx = TransactionFactory.Deserialize(payload);
var hash = facade.HashTransaction(tx, tx.Signature);
Console.WriteLine(hash);
Console.WriteLine(tx);
```
###### 出力例
```cs
> 2B104FF1E6F8B9DD643B17CE12D2B03601FA9D478E50F2BE8F77973D7C4AC4D9
> (signature: F95F520DF1E08FA936F3C57E7150062BD8710FB8290F5076540DFBFB22D22B32AED865F591C5AFA06D3A3A4208317973AE4CA24988C6C34EACE44B62EA40700C, signerPublicKey: 13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F, version: 0x2, network: NetworkType.TESTNET, type: TransactionType.AGGREGATE_COMPLETE, fee: 0x000000000000AF00, deadline: 0x00000003547B2F27, transactionsHash: E1B817E8E6186A67925A0C02DC514689F6684845E8F5A6564F4E7C37072E2D84, transactions: [(signerPublicKey: 13B00FBB13C7644E13BD786F0EA4F97820022A2606759793A5D3525A03F92A2F, version: 0x1, network: NetworkType.TESTNET, type: TransactionType.TRANSFER, recipientAddress: 983AB360969797AB6030FF53A1995F43B27C56C5B456E2D9, mosaics: [], message: hex(00747831), ),(signerPublicKey: 4C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B, version: 0x1, network: NetworkType.TESTNET, type: TransactionType.TRANSFER, recipientAddress: 982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF, mosaics: [], message: hex(00747832), )], cosignatures: [(version: 0x0, signerPublicKey: 4C4BD7F8E1E1AC61DB817089F9416A7EDC18339F06CDC851495B271533FAD13B, signature: 42C6293F8E8C2490260A452E2E8493D2CF03F733AD787C500941CC0043BCFE8E577D8480588094EB878696122809F8BB6CC285F6CB7E959733A1E2BBF8860B0B, )], )
```

### 署名者の検証

トランザクションがブロックに含まれていることが確認できれば自明ですが、  
念のため、アカウントの公開鍵でトランザクションの署名を検証しておきます。

```cs
var res = facade.VerifyTransaction(tx, tx.Signature, alicePublicKey);
Console.WriteLine(res);
```
```cs
> True
```

### マークルコンポーネントハッシュの計算

トランザクションのハッシュ値には連署者の情報が含まれていません。  
一方でブロックヘッダーに格納されるマークルルートはトランザクションのハッシュに連署者の情報が含めたものが格納されます。  
そのためトランザクションがブロック内部に存在しているかどうかを検証する場合は、トランザクションハッシュをマークルコンポーネントハッシュに変換しておく必要があります。

```cs
using Org.BouncyCastle.Crypto.Digests;

var merkleComponentHash = new byte[32];
if(tx.Cosignatures.Length != 0)
{
    var hasher = new Sha3Digest(256);
    hasher.BlockUpdate(hash.bytes, 0, hash.bytes.Length);
    foreach (var cosignature in tx.Cosignatures)
    {
        hasher.BlockUpdate(cosignature.SignerPublicKey.bytes, 0, cosignature.SignerPublicKey.bytes.Length);
    }
    hasher.DoFinal(merkleComponentHash, 0);
}
Console.WriteLine(Converter.BytesToHex(merkleComponentHash));

```
```cs
> 15FAE6AD62BE424C4E2D63A3553EBE0A7325CD3DE47F0BDC995386A59F08639A
```

### InBlockの検証

ノードからマークルツリーを取得し、先ほど計算したmerkleComponentHashからブロックヘッダーのマークルルートが導出できることを確認します。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Block-routes/operation/getBlockByHeight

https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Block-routes/operation/getMerkleTransaction

```cs
static string ComputeHash(string input)
{
    var outputBytes = new byte[32];
    var hasher = new Sha3Digest(256);
    var inputBytes = Converter.HexToBytes(input);
    hasher.BlockUpdate(inputBytes, 0, inputBytes.Length);
    hasher.DoFinal(outputBytes, 0);
    return Converter.BytesToHex(outputBytes);
}

static bool ValidateTransactionInBlock(string leaf, string hRoot, List<JsonNode> merkleProof)
{
    if (merkleProof.Count == 0)
        return leaf.ToUpper() == hRoot.ToUpper();

    var hRoot0 = merkleProof.Aggregate(leaf, (proofHash, pathItem) => {
        if ((string)pathItem["position"] == "left")
            return ComputeHash(pathItem["hash"] + proofHash);
        return ComputeHash(proofHash + pathItem["hash"]);
    });
    return hRoot.ToUpper() == hRoot0.ToUpper();
}

var blokInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{height}"));

var merkleInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{height}/transactions/{Converter.BytesToHex(merkleComponentHash)}/merkle"));
//トランザクションから計算
var leaf = Converter.BytesToHex(merkleComponentHash);
//ノードから取得
var HRoot = (string)blokInfo["block"]["transactionsHash"];
var merkleProof = ((JsonArray) merkleInfo["merklePath"]).ToList();
var result = ValidateTransactionInBlock(leaf, HRoot, merkleProof);
Console.WriteLine(result);
```
```cs
> True
```

トランザクションの情報がブロックヘッダーに含まれていることが確認できました。

## 13.2 ブロックヘッダーの検証

既知のブロックハッシュ値（例：ファイナライズブロック）から、検証中のブロックヘッダーまでたどれることを検証します。

BlockTypeは以下の３種類があります。
```cs
{32835: 'NemesisBlock', 33091: 'NormalBlock', 33347: 'ImportanceBlock'}
```

### normalブロックの検証

```cs
var height = 381959;
var blockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{height}"));
var previousBlockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{height - 1}"));
if ((ushort) blockInfo["block"]["type"] != 33091) return;
var hasher = new Sha3Digest(256);
var hash = new byte[32];
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["signature"]), 0, 64);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["signerPublicKey"]), 0, 32);
hasher.BlockUpdate(BitConverter.GetBytes((byte)blockInfo["block"]["version"]), 0, 1);
hasher.BlockUpdate(BitConverter.GetBytes((byte)blockInfo["block"]["network"]), 0, 1);
hasher.BlockUpdate(BitConverter.GetBytes((ushort)blockInfo["block"]["type"]), 0, 2);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["height"])), 0, 8);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["timestamp"])), 0, 8);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["difficulty"])), 0, 8);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["proofGamma"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["proofVerificationHash"]), 0, 16);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["proofScalar"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)previousBlockInfo["meta"]["hash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["transactionsHash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["receiptsHash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["stateHash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["beneficiaryAddress"]), 0, 24);
hasher.BlockUpdate(BitConverter.GetBytes((uint)blockInfo["block"]["feeMultiplier"]), 0, 4);
hasher.DoFinal(hash, 0);
Console.WriteLine((string)blockInfo["meta"]["hash"] == Converter.BytesToHex(hash));
```
True が出力されればこのブロックハッシュは前ブロックハッシュ値の存在を認知していることになります。  
同様にしてn番目のブロックがn-1番目のブロックを存在を確認し、最後に検証中のブロックにたどり着きます。  

これで、どのノードに問い合わせても確認可能な既知のファイナライズブロックが、  
検証したいブロックの存在に支えられていることが分かりました。  

### importanceブロックの検証

importanceBlockは、importance値の再計算が行われるブロック(720ブロック毎、テストネットは180ブロック毎)です。  
NormalBlockに加えて以下の情報が追加されています。  

- votingEligibleAccountsCount
- harvestingEligibleAccountsCount
- totalVotingBalance
- previousImportanceBlockHash

```cs
var height = 381960;
var blockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{height}"));
var previousBlockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks/{height - 1}"));
if ((ushort) blockInfo["block"]["type"] != 33347) return;
var hasher = new Sha3Digest(256);
var hash = new byte[32];
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["signature"]), 0, 64);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["signerPublicKey"]), 0, 32);
hasher.BlockUpdate(BitConverter.GetBytes((byte)blockInfo["block"]["version"]), 0, 1);
hasher.BlockUpdate(BitConverter.GetBytes((byte)blockInfo["block"]["network"]), 0, 1);
hasher.BlockUpdate(BitConverter.GetBytes((ushort)blockInfo["block"]["type"]), 0, 2);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["height"])), 0, 8);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["timestamp"])), 0, 8);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["difficulty"])), 0, 8);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["proofGamma"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["proofVerificationHash"]), 0, 16);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["proofScalar"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)previousBlockInfo["meta"]["hash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["transactionsHash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["receiptsHash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["stateHash"]), 0, 32);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["beneficiaryAddress"]), 0, 24);
hasher.BlockUpdate(BitConverter.GetBytes((uint)blockInfo["block"]["feeMultiplier"]), 0, 4);
hasher.BlockUpdate(BitConverter.GetBytes((uint)blockInfo["block"]["votingEligibleAccountsCount"]), 0, 4);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["harvestingEligibleAccountsCount"])), 0, 8);
hasher.BlockUpdate(BitConverter.GetBytes(ulong.Parse((string)blockInfo["block"]["totalVotingBalance"])), 0, 8);
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["block"]["previousImportanceBlockHash"]), 0, 32);
hasher.DoFinal(hash, 0);
Console.WriteLine((string)blockInfo["meta"]["hash"] == Converter.BytesToHex(hash));
```

後述するアカウントやメタデータの検証のために、stateHashSubCacheMerkleRootsを検証しておきます。

### stateHashの検証
```cs
Console.WriteLine($"NormalBlockInfo: {blockInfo}");
```
```cs
> NormalBlockInfo: {
  "meta": {
    "hash": "6B7E8D85CAD9178BB392876DAABE67A6B9836373A07F614714FAE8A0879B3F4C",
    "generationHash": "BF28FBA2F2C2C3F62897882CF3E7945DC8C5C27E9039F7647604CF91D488A7BA",
    "totalFee": "0",
    "totalTransactionsCount": 0,
    "stateHashSubCacheMerkleRoots": [
      "A82A1434A2115684EEEE1CE7FC17B5FA34172D62FAB42AF2365EF17A24E8FD8F",
      "18C3D9EC283D662E3EC83DD9122F3D78077B800A4665BB784D8483A8E8D1E77D",
      "D86FD52B9A0812F7F2EBEDA3957C3C1BF7BCD0DA5A56497C5B33DE372ADFC25F",
      "295DA4F751A3A51E09F430E90BD37CD935129646586224DD14E1BED7B4F38BB5",
      "0000000000000000000000000000000000000000000000000000000000000000",
      "0000000000000000000000000000000000000000000000000000000000000000",
      "D663DBB92948F056ADB0D62ACFF211993DFEB3C174386ACF6307BF4B20C18825",
      "23D1614C66F429EECF60F7437A4F2104F853A47F4E3119F6EC4D85A293F9F2A2",
      "9EA761073278891F8D962B3C88BC9F1922E5634575DC55820431DDCDBCF16137"
    ],
    "transactionsCount": 0,
    "statementsCount": 1
  },
```
```cs
var hasher = new Sha3Digest(256);
var hash = new byte[32];

hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][0]), 0, 32); // AccountState
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][1]), 0, 32); // Namespace
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][2]), 0, 32); // Mosaic
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][3]), 0, 32); // Multisig
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][4]), 0, 32); // HashLockInfo
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][5]), 0, 32); // SecretLockInfo
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][6]), 0, 32); // AccountRestriction
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][7]), 0, 32); // MosaicRestriction
hasher.BlockUpdate(Converter.HexToBytes((string)blockInfo["meta"]["stateHashSubCacheMerkleRoots"][8]), 0, 32); // Metadata

hasher.DoFinal(hash, 0);
Console.WriteLine((string)blockInfo["block"]["stateHash"] == Converter.BytesToHex(hash));
```
```cs
> True
```

ブロックヘッダーの検証に利用した9個のstateがstateHashSubCacheMerkleRootsから構成されていることがわかります。


## 13.3 アカウント・メタデータの検証

マークルパトリシアツリーを利用して、トランザクションに紐づくアカウントやメタデータの存在を検証します。  
サービス提供者がマークルパトリシアツリーを提供すれば、利用者は自分の意志で選択したノードを使ってその真偽を検証することができます。

### 検証用共通関数

```cs
// 葉のハッシュ値取得関数
static string GetLeafHash(string encodedPath, string leafValue)
{
    var hasher = new Sha3Digest(256);
    var input = Converter.HexToBytes(encodedPath + leafValue);
    hasher.BlockUpdate(input, 0, input.Length);
    var hash = new byte[hasher.GetDigestSize()];
    hasher.DoFinal(hash, 0);
    return Converter.BytesToHex(hash);
}

// 枝のハッシュ値取得関数
static string GetBranchHash(string encodedPath, List<JsonNode> links)
{
    var branchLinks = Enumerable.Repeat("0000000000000000000000000000000000000000000000000000000000000000", 16).ToList();
    foreach (var link in links)
    {
        branchLinks[int.Parse((string)link["bit"], NumberStyles.HexNumber)] = (string)link["link"];
    }
    var hasher = new Sha3Digest(256);
    var input = Converter.HexToBytes(encodedPath + string.Join("", branchLinks));
    hasher.BlockUpdate(input, 0, input.Length);
    var hash = new byte[hasher.GetDigestSize()];
    hasher.DoFinal(hash, 0);
    return Converter.BytesToHex(hash);
}

// ワールドステートの検証
static void CheckState(JsonNode stateProof, string stateHash, string pathHash, string rootHash)
{
    JsonNode? merkleLeaf = null;
    var merkleBranches = new List<JsonNode>();
    foreach (var tree in (JsonArray)stateProof["tree"])
    {
        if((byte)tree["type"] == 255)
        {
            merkleLeaf = tree;
        }
        else
        {
            merkleBranches.Add(tree);
        }
    }
    merkleBranches.Reverse();
    if(merkleLeaf == null) throw new NullReferenceException("merkleTree is null");
    var leafHash = GetLeafHash((string)merkleLeaf["encodedPath"], stateHash);
    var linkHash = leafHash; // 最初のlinkHashはleafHash
    
    var bit = "";
    foreach (var branch in merkleBranches)
    {
        var branchLink = ((JsonArray)branch["links"]).ToList().Find(x => (string)x["link"] == linkHash);
        linkHash = GetBranchHash((string)branch["encodedPath"], ((JsonArray)branch["links"]).ToList());
        var temp = (string) branch["path"] == ""
            ? ""
            : ((string) branch["path"]).Substring(0, (int) branch["nibbleCount"]);
        bit = temp + (string)branchLink["bit"] + bit;
    }

    var treeRootHash = linkHash; // 最後のlinkHashはrootHash
    var treePathHash = bit + merkleLeaf["path"];

    if (treePathHash.Length % 2 == 1)
    {
        treePathHash = treePathHash.Substring(0, treePathHash.Length - 1);
    }

    // 検証
    Console.WriteLine(treeRootHash == rootHash);
    Console.WriteLine(treePathHash == pathHash);
}
```

### 13.3.1 アカウント情報の検証

アカウント情報を葉として、
マークルツリー上の分岐する枝をアドレスでたどり、
ルートに到着できるかを確認します。

※本項はC#SDKにAccountInfoをシリアライズする関数がまだあらず、時間を要するので現時点ではオリジナル版にて学習してください。

https://github.com/xembook/quick_learning_symbol/blob/main/13_verify.md#1331-%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E6%83%85%E5%A0%B1%E3%81%AE%E6%A4%9C%E8%A8%BC


### 13.3.2 モザイクへ登録したメタデータの検証

モザイクに登録したメタデータValue値を葉として、
マークルツリー上の分岐する枝をメタデータキーで構成されるハッシュ値でたどり、
ルートに到着できるかを確認します。

```cs
var sourceAddress = Converter.StringToAddress("TAUYF774MZWLBEUI7S2LR6BA5CYLL53QSMDVV3Y");
var targetAddress = Converter.StringToAddress("TAUYF774MZWLBEUI7S2LR6BA5CYLL53QSMDVV3Y");

//compositePathHash(Key値)
var hasher = new Sha3Digest(256);
var compositeHash = new byte[32];
hasher.BlockUpdate(sourceAddress, 0, sourceAddress.Length);
hasher.BlockUpdate(targetAddress, 0, targetAddress.Length);
hasher.BlockUpdate(Converter.HexToBytes("90623009C157B457", true), 0, 8); // scopeKey
hasher.BlockUpdate(Converter.HexToBytes("173AC1E38CBAD11D", true), 0, 8); // targetId
hasher.BlockUpdate(new byte[]{1}, 0, 1); // type: Mosaic 1
hasher.DoFinal(compositeHash, 0);

var hasher2 = new Sha3Digest(256);
var pathHash = new byte[32];
hasher2.BlockUpdate(compositeHash, 0, compositeHash.Length);
hasher2.DoFinal(pathHash, 0);

//stateHash(Value値)
var hasher3 = new Sha3Digest(256);
var stateHash = new byte[32];
hasher3.BlockUpdate(new byte[]{1, 0}, 0, 2); // version
hasher3.BlockUpdate(sourceAddress, 0, sourceAddress.Length);
hasher3.BlockUpdate(targetAddress, 0, targetAddress.Length);
hasher3.BlockUpdate(Converter.HexToBytes("90623009C157B457", true), 0, 8); // scopeKey
hasher3.BlockUpdate(Converter.HexToBytes("173AC1E38CBAD11D", true), 0, 8); // targetId
hasher3.BlockUpdate(new byte[]{1}, 0, 1); // type: Mosaic 1

var value = Converter.Utf8ToBytes("test");
hasher3.BlockUpdate(BitConverter.GetBytes(value.Length), 0, 2);
hasher3.BlockUpdate(value, 0, value.Length); // value
hasher3.DoFinal(stateHash, 0);

//サービス提供者以外のノードから最新のブロックヘッダー情報を取得
var blockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks?order=desc"));
var rootHash = blockInfo["data"][0]["meta"]["stateHashSubCacheMerkleRoots"][8];

//サービス提供者を含む任意のノードからマークル情報を取得
var stateProof = JsonNode.Parse(await GetDataFromApi(node, $"/metadata/{Converter.BytesToHex(compositeHash)}/merkle"));

//検証
CheckState(stateProof, Converter.BytesToHex(stateHash), Converter.BytesToHex(pathHash), (string)rootHash);
```

### 13.3.3 アカウントへ登録したメタデータの検証

アカウントに登録したメタデータValue値を葉として、
マークルツリー上の分岐する枝をメタデータキーで構成されるハッシュ値でたどり、
ルートに到着できるかを確認します。

```cs
var sourceAddress = Converter.StringToAddress("TAUYF774MZWLBEUI7S2LR6BA5CYLL53QSMDVV3Y");
var targetAddress = Converter.StringToAddress("TAUYF774MZWLBEUI7S2LR6BA5CYLL53QSMDVV3Y");

//compositePathHash(Key値)
var hasher = new Sha3Digest(256);
var compositeHash = new byte[32];
hasher.BlockUpdate(sourceAddress, 0, sourceAddress.Length);
hasher.BlockUpdate(targetAddress, 0, targetAddress.Length);
hasher.BlockUpdate(Converter.HexToBytes("8705F15653EBBC43", true), 0, 8); // scopeKey
hasher.BlockUpdate(Converter.HexToBytes("0000000000000000", true), 0, 8); // targetId
hasher.BlockUpdate(new byte[]{0}, 0, 1); // type: Account 0
hasher.DoFinal(compositeHash, 0);

var hasher2 = new Sha3Digest(256);
var pathHash = new byte[32];
hasher2.BlockUpdate(compositeHash, 0, compositeHash.Length);
hasher2.DoFinal(pathHash, 0);

//stateHash(Value値)
var hasher3 = new Sha3Digest(256);
var stateHash = new byte[32];
hasher3.BlockUpdate(new byte[]{1, 0}, 0, 2); // version
hasher3.BlockUpdate(sourceAddress, 0, sourceAddress.Length);
hasher3.BlockUpdate(targetAddress, 0, targetAddress.Length);
hasher3.BlockUpdate(Converter.HexToBytes("8705F15653EBBC43", true), 0, 8); // scopeKey
hasher3.BlockUpdate(Converter.HexToBytes("0000000000000000", true), 0, 8); // targetId
hasher3.BlockUpdate(new byte[]{0}, 0, 1); // type: Account 0

var value = Converter.Utf8ToBytes("test");
hasher3.BlockUpdate(BitConverter.GetBytes(value.Length), 0, 2);
hasher3.BlockUpdate(value, 0, value.Length); // value
hasher3.DoFinal(stateHash, 0);

//サービス提供者以外のノードから最新のブロックヘッダー情報を取得
var blockInfo = JsonNode.Parse(await GetDataFromApi(node, $"/blocks?order=desc"));
var rootHash = blockInfo["data"][0]["meta"]["stateHashSubCacheMerkleRoots"][8];

//サービス提供者を含む任意のノードからマークル情報を取得
var stateProof = JsonNode.Parse(await GetDataFromApi(node, $"/metadata/{Converter.BytesToHex(compositeHash)}/merkle"));

// 検証
CheckState(stateProof, Converter.BytesToHex(stateHash), Converter.BytesToHex(pathHash), (string)rootHash);
```

## 13.4 現場で使えるヒント

### トラステッドウェブ

トラステッドウェブを簡単に説明すると、全てをプラットフォーマーに依存せず、かつ全てを検証せずに済むWebの実現です。

本章の検証で分かることは、ブロックチェーンが持つすべての情報はブロックヘッダーのハッシュ値によって検証可能ということです。
ブロックチェーンはみんなが認め合うブロックヘッダーの共有とそれを再現できるフルノードの存在で成り立っています。
しかし、ブロックチェーンを活用したいあらゆるシーンでこれらを検証するための環境を維持しておくことは非常に困難です。
最新のブロックヘッダーが複数の信頼できる機関から常時ブロードキャストされていれば、検証の手間を大きく省くことができます
このようなインフラが整えば、都会などの数千万人が密集する超過密地帯、あるいは基地局が十分に配置できない僻地や災害時の広域ネットワーク遮断時など
ブロックチェーンの能力を超えた場所においても信頼できる情報にアクセスできるようになります。
