# 13.検証
ブロックチェーン上に記録されたさまざまな情報を検証します。
ブロックチェーンへのデータ記録は全ノードの合意を持って行われますが、
ブロックチェーンへの**データ参照**はノード単体からの情報取得であるため、
信用できないノードの情報を元にして新たな取引を行いたい場合は、ノードから取得したデータに対して検証を行う必要があります。


## 13.1 トランザクションの検証

トランザクションがブロックヘッダーに含まれていることを検証します。この検証が成功すれば、トランザクションがブロックチェーンの合意によって承認されたものとみなすことができます。

本章のサンプルスクリプトを実行する前に以下を実行して必要ライブラリを読み込んでおいてください。
```js
Buffer = require("/node_modules/buffer").Buffer;
cat = require("/node_modules/catbuffer-typescript");
sha3_256 = require('/node_modules/js-sha3').sha3_256;

accountRepo = repo.createAccountRepository();
blockRepo = repo.createBlockRepository();
stateProofService = new sym.StateProofService(repo);
```

### 検証するペイロード

今回検証するトランザクションペイロードとそのトランザクションが記録されているとされるブロック高です。

```js
payload = 'C00200000000000093B0B985101C1BDD1BC2BF30D72F35E34265B3F381ECA464733E147A4F0A6B9353547E2E08189EF37E50D271BEB5F09B81CE5816BB34A153D2268520AF630A0A0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198414140770200000000002A769FB40000000076B455CFAE2CCDA9C282BF8556D3E9C9C0DE18B0CBE6660ACCF86EB54AC51B33B001000000000000DB000000000000000E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198544198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F22338B000000000000000066653465353435393833444430383935303645394533424446434235313637433046394232384135344536463032413837364535303734423641303337414643414233303344383841303630353343353345354235413835323835443639434132364235343233343032364244444331443133343139464435353438323930334242453038423832304100000000006800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F2233BC089179EBBE01A81400140035383435344434373631364336433635373237396800000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D000000000198444198205C1A4CE06C45B3A896B1B2360E03633B9F36BF7F223345ECB996EDDB9BEB1400140035383435344434373631364336433635373237390000000000000000B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D5A71EBA9C924EFA146897BE6C9BB3DACEFA26A07D687AC4A83C9B03087640E2D1DDAE952E9DDBC33312E2C8D021B4CC0435852C0756B1EBD983FCE221A981D02';
height = 59639;
```


### payload確認

トランザクションの内容を確認します。

```js
tx = sym.TransactionMapping.createFromPayload(payload);
hash = sym.Transaction.createTransactionHash(payload,Buffer.from(generationHash, 'hex'));
console.log(hash);
console.log(tx);
```
###### 出力例
```js
> 257E2CAECF4B477235CA93C37090E8BE58B7D3812A012E39B7B55BA7D7FFCB20
> AggregateTransaction
    > cosignatures: Array(1)
      0: AggregateTransactionCosignature
        signature: "5A71EBA9C924EFA146897BE6C9BB3DACEFA26A07D687AC4A83C9B03087640E2D1DDAE952E9DDBC33312E2C8D021B4CC0435852C0756B1EBD983FCE221A981D02"
        signer: PublicAccount
          address: Address {address: 'TAQFYGSM4BWELM5IS2Y3ENQOANRTXHZWX57SEMY', networkType: 152}
          publicKey: "B2D4FD84B2B63A96AA37C35FC6E0A2341CEC1FD19C8FFC8D93CCCA2B028D1E9D"
      deadline: Deadline {adjustedValue: 3030349354}
    > innerTransactions: Array(3)
        0: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
        1: AccountMetadataTransaction {type: 16708, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
        2: AccountMetadataTransaction {type: 16708, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
      maxFee: UInt64 {lower: 161600, higher: 0}
      networkType: 152
      signature: "93B0B985101C1BDD1BC2BF30D72F35E34265B3F381ECA464733E147A4F0A6B9353547E2E08189EF37E50D271BEB5F09B81CE5816BB34A153D2268520AF630A0A"
    > signer: PublicAccount
        address: Address {address: 'TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ', networkType: 152}
        publicKey: "0E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26"
      transactionInfo: undefined
      type: 16705
```

### 署名者の検証

トランザクションがブロックに含まれていることが確認できれば自明ですが、  
念のため、アカウントの公開鍵でトランザクションの署名を検証しておきます。

```js
res = alice.publicAccount.verifySignature(
    tx.getSigningBytes([...Buffer.from(payload,'hex')],[...Buffer.from(generationHash,'hex')]),
    "93B0B985101C1BDD1BC2BF30D72F35E34265B3F381ECA464733E147A4F0A6B9353547E2E08189EF37E50D271BEB5F09B81CE5816BB34A153D2268520AF630A0A"
);
console.log(res);
```
```js
> true
```

getSigningBytesで署名の対象となる部分だけを取り出しています。  
通常のトランザクションとアグリゲートトランザクションでは取り出す部分が異なるので注意が必要です。  

### マークルコンポーネントハッシュの計算

トランザクションのハッシュ値には連署者の情報が含まれていません。  
一方でブロックヘッダーに格納されるマークルルートはトランザクションのハッシュに連署者の情報が含めたものが格納されます。  
そのためトランザクションがブロック内部に存在しているかどうかを検証する場合は、トランザクションハッシュをマークルコンポーネントハッシュに変換しておく必要があります。

```js
merkleComponentHash = hash;
if( tx.cosignatures !== undefined && tx.cosignatures.length > 0){
  
  const hasher = sha3_256.create();
  hasher.update(Buffer.from(hash, 'hex'));
  for (cosignature of tx.cosignatures ){
    hasher.update(Buffer.from(cosignature.signer.publicKey, 'hex'));
  }
  merkleComponentHash = hasher.hex().toUpperCase();
}
console.log(merkleComponentHash);
```
```js
> C8D1335F07DE05832B702CACB85B8EDAC2F3086543C76C9F56F99A0861E8F235
```

### InBlockの検証

ノードからマークルツリーを取得し、先ほど計算したmerkleComponentHashからブロックヘッダーのマークルルートが導出できることを確認します。

```js
function validateTransactionInBlock(leaf,HRoot,merkleProof){

  if (merkleProof.length === 0) {
    // There is a single item in the tree, so HRoot' = leaf.
    return leaf.toUpperCase() === HRoot.toUpperCase();
  }

  const HRoot0 = merkleProof.reduce((proofHash, pathItem) => {
    const hasher = sha3_256.create();
    if (pathItem.position === sym.MerklePosition.Left) {
      return hasher.update(Buffer.from(pathItem.hash + proofHash, 'hex')).hex();
    } else {
      return hasher.update(Buffer.from(proofHash + pathItem.hash, 'hex')).hex();
    }
  }, leaf);
  return HRoot.toUpperCase() === HRoot0.toUpperCase();
}

//トランザクションから計算
leaf = merkleComponentHash.toLowerCase();//merkleComponentHash

//ノードから取得
HRoot = (await blockRepo.getBlockByHeight(height).toPromise()).blockTransactionsHash;
merkleProof = (await blockRepo.getMerkleTransaction(height, leaf).toPromise()).merklePath;

result = validateTransactionInBlock(leaf,HRoot,merkleProof);
console.log(result);
```
```js
> true
```

トランザクションの情報がブロックヘッダーに含まれていることが確認できました。

## 13.2 ブロックヘッダーの検証

既知のブロックハッシュ値（例：ファイナライズブロック）から、検証中のブロックヘッダーまでたどれることを検証します。


### normalブロックの検証

```js
block = await blockRepo.getBlockByHeight(height).toPromise();
previousBlock = await blockRepo.getBlockByHeight(height - 1).toPromise();
if(block.type ===  sym.BlockType.NormalBlock){
    
  hasher = sha3_256.create();
  hasher.update(Buffer.from(block.signature,'hex')); //signature
  hasher.update(Buffer.from(block.signer.publicKey,'hex')); //publicKey
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.version, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.networkType, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.type, 2));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.height.lower    ,block.height.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.timestamp.lower ,block.timestamp.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.difficulty.lower,block.difficulty.higher]));
  hasher.update(Buffer.from(block.proofGamma,'hex'));
  hasher.update(Buffer.from(block.proofVerificationHash,'hex'));
  hasher.update(Buffer.from(block.proofScalar,'hex'));
  hasher.update(Buffer.from(previousBlock.hash,'hex'));
  hasher.update(Buffer.from(block.blockTransactionsHash,'hex'));
  hasher.update(Buffer.from(block.blockReceiptsHash,'hex'));
  hasher.update(Buffer.from(block.stateHash,'hex'));
  hasher.update(sym.RawAddress.stringToAddress(block.beneficiaryAddress.address));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.feeMultiplier, 4));
  hash = hasher.hex().toUpperCase();
  console.log(hash === block.hash);
}
```

true が出力されればこのブロックハッシュは前ブロックハッシュ値の存在を認知していることになります。  
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

```js
block = await blockRepo.getBlockByHeight(height).toPromise();
previousBlock = await blockRepo.getBlockByHeight(height - 1).toPromise();
if(block.type ===  sym.BlockType.ImportanceBlock){

  hasher = sha3_256.create();
  hasher.update(Buffer.from(block.signature,'hex')); //signature
  hasher.update(Buffer.from(block.signer.publicKey,'hex')); //publicKey
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.version, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.networkType, 1));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.type, 2));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.height.lower    ,block.height.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.timestamp.lower ,block.timestamp.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.difficulty.lower,block.difficulty.higher]));
  hasher.update(Buffer.from(block.proofGamma,'hex'));
  hasher.update(Buffer.from(block.proofVerificationHash,'hex'));
  hasher.update(Buffer.from(block.proofScalar,'hex'));
  hasher.update(Buffer.from(previousBlock.hash,'hex'));
  hasher.update(Buffer.from(block.blockTransactionsHash,'hex'));
  hasher.update(Buffer.from(block.blockReceiptsHash,'hex'));
  hasher.update(Buffer.from(block.stateHash,'hex'));
  hasher.update(sym.RawAddress.stringToAddress(block.beneficiaryAddress.address));
  hasher.update(cat.GeneratorUtils.uintToBuffer(   block.feeMultiplier, 4));
  hasher.update(cat.GeneratorUtils.uintToBuffer(block.votingEligibleAccountsCount,4));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.harvestingEligibleAccountsCount.lower,block.harvestingEligibleAccountsCount.higher]));
  hasher.update(cat.GeneratorUtils.uint64ToBuffer([block.totalVotingBalance.lower,block.totalVotingBalance.higher]));
  hasher.update(Buffer.from(block.previousImportanceBlockHash,'hex'));

  hash = hasher.hex().toUpperCase();
  console.log(hash === block.hash);
}
```

後述するアカウントやメタデータの検証のために、stateHashSubCacheMerkleRootsを検証しておきます。

### stateHashの検証
```js
console.log(block);
```
```js
> NormalBlockInfo
    height: UInt64 {lower: 59639, higher: 0}
    hash: "B5F765D388B5381AC93659F501D5C68C00A2EE7DF4548C988E97F809B279839B"
    stateHash: "9D6801C49FE0C31ADE5C1BB71019883378016FA35230B9813CA6BB98F7572758"
  > stateHashSubCacheMerkleRoots: Array(9)
        0: "4578D33DD0ED5B8563440DA88F627BBC95A174C183191C15EE1672C5033E0572"
        1: "2C76DAD84E4830021BE7D4CF661218973BA467741A1FC4663B54B5982053C606"
        2: "259FB9565C546BAD0833AD2B5249AA54FE3BC45C9A0C64101888AC123A156D04"
        3: "58D777F0AA670440D71FA859FB51F8981AF1164474840C71C1BEB4F7801F1B27"
        4: "C9092F0652273166991FA24E8B115ACCBBD39814B8820A94BFBBE3C433E01733"
        5: "4B53B8B0E5EE1EEAD6C1498CCC1D839044B3AE5F85DD8C522A4376C2C92D8324"
        6: "132324AF5536EC9AA85B2C1697F6B357F05EAFC130894B210946567E4D4E9519"
        7: "8374F46FBC759049F73667265394BD47642577F16E0076CBB7B0B9A92AAE0F8E"
        8: "45F6AC48E072992343254F440450EF4E840D8386102AD161B817E9791ABC6F7F"
```
```js
hasher = sha3_256.create();
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[0],'hex')); //AccountState
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[1],'hex')); //Namespace
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[2],'hex')); //Mosaic
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[3],'hex')); //Multisig
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[4],'hex')); //HashLockInfo
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[5],'hex')); //SecretLockInfo
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[6],'hex')); //AccountRestriction
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[7],'hex')); //MosaicRestriction
hasher.update(Buffer.from(block.stateHashSubCacheMerkleRoots[8],'hex')); //Metadata
hash = hasher.hex().toUpperCase();
console.log(block.stateHash === hash);
```
```js
> true
```

ブロックヘッダーの検証に利用した9個のstateがstateHashSubCacheMerkleRootsから構成されていることがわかります。


## 13.3 アカウント・メタデータの検証

マークルパトリシアツリーを利用して、トランザクションに紐づくアカウントやメタデータの存在を検証します。  
サービス提供者がマークルパトリシアツリーを提供すれば、利用者は自分の意志で選択したノードを使ってその真偽を検証することができます。

### 検証用共通関数

```js
//葉のハッシュ値取得関数
function getLeafHash(encodedPath, leafValue){
    const hasher = sha3_256.create();
    return hasher.update(sym.Convert.hexToUint8(encodedPath + leafValue)).hex().toUpperCase();
}

//枝のハッシュ値取得関数
function getBranchHash(encodedPath, links){
    const branchLinks = Array(16).fill(sym.Convert.uint8ToHex(new Uint8Array(32)));
    links.forEach((link) => {
        branchLinks[parseInt(`0x${link.bit}`, 16)] = link.link;
    });
    const hasher = sha3_256.create();
    const bHash = hasher.update(sym.Convert.hexToUint8(encodedPath + branchLinks.join(''))).hex().toUpperCase();
    return bHash;
}

//ワールドステートの検証
function checkState(stateProof,stateHash,pathHash,rootHash){

  const merkleLeaf = stateProof.merkleTree.leaf;
  const merkleBranches = stateProof.merkleTree.branches.reverse();
  const leafHash = getLeafHash(merkleLeaf.encodedPath,stateHash);

  let linkHash = leafHash; //最初のlinkHashはleafHash
  let bit="";
  for(let i = 0; i < merkleBranches.length; i++){
      const branch = merkleBranches[i];
      const branchLink = branch.links.find(x=>x.link === linkHash)
      linkHash = getBranchHash(branch.encodedPath,branch.links);
      bit = merkleBranches[i].path.slice(0,merkleBranches[i].nibbleCount) + branchLink.bit + bit ;
  }

  const treeRootHash = linkHash; //最後のlinkHashはrootHash
  let treePathHash = bit + merkleLeaf.path;

  if(treePathHash.length % 2 == 1){
    treePathHash = treePathHash.slice( 0, -1 );
  }
 
  //検証
  console.log(treeRootHash === rootHash);
  console.log(treePathHash === pathHash);
}
```


### 13.3.1 アカウント情報の検証

アカウント情報を葉として、
マークルツリー上の分岐する枝をアドレスでたどり、
ルートに到着できるかを確認します。

```js
stateProofService = new sym.StateProofService(repo);

aliceAddress = sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ");

hasher = sha3_256.create();
alicePathHash = hasher.update(
  sym.RawAddress.stringToAddress(aliceAddress.plain())
).hex().toUpperCase();

hasher = sha3_256.create();
aliceInfo = await accountRepo.getAccountInfo(aliceAddress).toPromise();
aliceStateHash = hasher.update(aliceInfo.serialize()).hex().toUpperCase();

//サービス提供者以外のノードから最新のブロックヘッダー情報を取得
blockInfo = await blockRepo.search({order:"desc"}).toPromise();
rootHash = blockInfo.data[0].stateHashSubCacheMerkleRoots[0];

//サービス提供者を含む任意のノードからマークル情報を取得
stateProof = await stateProofService.accountById(aliceAddress).toPromise();

//検証
checkState(stateProof,aliceStateHash,alicePathHash,rootHash);
```


### 13.3.2 モザイクへ登録したメタデータの検証

モザイクに登録したメタデータValue値を葉として、
マークルツリー上の分岐する枝をメタデータキーで構成されるハッシュ値でたどり、
ルートに到着できるかを確認します。

```js
srcAddress = Buffer.from(
    sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ").encoded(),
    'hex'
)

targetAddress = Buffer.from(
    sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ").encoded(),
    'hex'
)

hasher = sha3_256.create();    
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("CF217E116AA422E2")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("1275B0B7511D9161")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Mosaic])); // type: Account 0
compositeHash = hasher.hex();

hasher = sha3_256.create();   
hasher.update( Buffer.from(compositeHash,'hex'));

pathHash = hasher.hex().toUpperCase();

//stateHash(Value値)
hasher = sha3_256.create(); 
hasher.update(cat.GeneratorUtils.uintToBuffer(1, 2)); //version
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("CF217E116AA422E2")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("1275B0B7511D9161")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Mosaic])); //account

value = Buffer.from("test");

hasher.update(cat.GeneratorUtils.uintToBuffer(value.length, 2)); 
hasher.update(value); 
stateHash = hasher.hex();

//サービス提供者以外のノードから最新のブロックヘッダー情報を取得
blockInfo = await blockRepo.search({order:"desc"}).toPromise();
rootHash = blockInfo.data[0].stateHashSubCacheMerkleRoots[8];

//サービス提供者を含む任意のノードからマークル情報を取得
stateProof = await stateProofService.metadataById(compositeHash).toPromise();

//検証
checkState(stateProof,stateHash,pathHash,rootHash);
```

### 13.3.3 アカウントへ登録したメタデータの検証

アカウントに登録したメタデータValue値を葉として、
マークルツリー上の分岐する枝をメタデータキーで構成されるハッシュ値でたどり、
ルートに到着できるかを確認します。

```js
srcAddress = Buffer.from(
    sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ").encoded(),
    'hex'
)

targetAddress = Buffer.from(
    sym.Address.createFromRawAddress("TBIL6D6RURP45YQRWV6Q7YVWIIPLQGLZQFHWFEQ").encoded(),
    'hex'
)

//compositePathHash(Key値)
hasher = sha3_256.create();    
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("9772B71B058127D7")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("0000000000000000")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Account])); // type: Account 0
compositeHash = hasher.hex();

hasher = sha3_256.create();   
hasher.update( Buffer.from(compositeHash,'hex'));

pathHash = hasher.hex().toUpperCase();

//stateHash(Value値)
hasher = sha3_256.create(); 
hasher.update(cat.GeneratorUtils.uintToBuffer(1, 2)); //version
hasher.update(srcAddress);
hasher.update(targetAddress);
hasher.update(sym.Convert.hexToUint8Reverse("9772B71B058127D7")); // scopeKey
hasher.update(sym.Convert.hexToUint8Reverse("0000000000000000")); // targetId
hasher.update(Uint8Array.from([sym.MetadataType.Account])); //account
value = Buffer.from("test");
hasher.update(cat.GeneratorUtils.uintToBuffer(value.length, 2)); 
hasher.update(value); 
stateHash = hasher.hex();

//サービス提供者以外のノードから最新のブロックヘッダー情報を取得
blockInfo = await blockRepo.search({order:"desc"}).toPromise();
rootHash = blockInfo.data[0].stateHashSubCacheMerkleRoots[8];

//サービス提供者を含む任意のノードからマークル情報を取得
stateProof = await stateProofService.metadataById(compositeHash).toPromise();

//検証
checkState(stateProof,stateHash,pathHash,rootHash);
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
