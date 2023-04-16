# 3.キーペア

キーペアは秘密鍵と公開鍵のペアで公開鍵を定められた形式でコンバートした物がアドレスとなります。秘密鍵を使って署名することでのみブロックチェーンのデータを更新することができます。  

## 3.1 キーペア生成
まずはキーペアを作成します。

### 新規生成
```cs
var aliceKeyPair = KeyPair.GenerateNewKeyPair();
var alicePrivateKey = aliceKeyPair.PrivateKey;
var alicePublicKey = aliceKeyPair.PublicKey;
Console.WriteLine($"alicePrivateKey: {alicePrivateKey}");
Console.WriteLine($"alicePublicKey: {alicePublicKey}");
```
鍵はそれぞれbyte[]ですがコンソールに出力する際は16進数文字列にて出力します。

###### 出力例
```
> alicePrivateKey: 14311B7BEC946A28AD845FA8B565B90F52B6E0562FCC2D779D46FEB3A234****
> alicePublicKey: 5EDD03AA17FC3EF57F02271A979A0B30B6F350615DDDDC54D5F9DF3CD2690F04
```

#### 注意事項
秘密鍵を紛失するとそのアカウントに紐づけられたデータを操作することが出来なくなります。また、他人は知らないという秘密鍵の性質を利用してデータ操作の署名を行うので、秘密鍵を他人に教えてはいけません。組織のなかで秘密鍵を譲り受けて運用を続けるといった行為も控えましょう。
一般的なWebサービスでは「アカウントID」に対してパスワードが割り振られるため、パスワードの変更が可能ですが、ブロックチェーンではパスワードにあたる秘密鍵に対して一意に決まるID(アドレス)が割り振られるため、アカウントに紐づく秘密鍵を変更するということはできません。  


### アドレスの導出
```cs
var aliceAddress = facade.Network.PublicKeyToAddress(alicePublicKey);
Console.WriteLine($"aliceAddress: {aliceAddress}");
```
```cs
> aliceAddress: TC3YSUYEZLW66IGQ7SIDT4DVYYUKT6OMWHAYYYQ
```

これらがブロックチェーンを操作するための最も基本的な情報となります。また、既存の秘密鍵からキーペアを生成する方法は以下となります。

### 秘密鍵からキーペア生成
```cs
var alicePrivateKey = new PrivateKey("14311B7BEC946A28AD845FA8B565B90F52B6E0562FCC2D779D46FEB3A234****");
var aliceKeyPair = new KeyPair(alicePrivateKey);
```

## 3.2 アカウントへの送信

アカウントを作成しただけでは、ブロックチェーンにデータを送信することはできません。  
パブリックブロックチェーンはリソースを有効活用するためにデータ送信時に手数料を要求します。  
Symbolブロックチェーンでは、この手数料をXYMという共通トークンで支払うことになります。  
アカウントを生成したら、この後の章から説明するトランザクションを実行するために必要な手数料を送信しておきます。  

### フォーセットから送信

テストネットではフォーセット（蛇口）サービスから検証用のXYMを入手することができます。  
メインネットの場合は取引所などでXYMを購入するか、投げ銭サービス(NEMLOG,QUEST)などを利用して寄付を募りましょう。  

テストネット
- FAUCET(蛇口)
  - https://testnet.symbol.tools/

メインネット
- NEMLOG
  - https://nemlog.nem.social/
- QUEST
  - https://quest-bc.com/



### エクスプローラーで確認

フォーセットから作成したアカウントへ送信が成功したらエクスプローラーで確認してみましょう。

- テストネット
  - https://testnet.symbol.fyi/
- メインネット
  - https://symbol.fyi/

## 3.3 アカウント情報の確認

ノードに保存されているアカウント情報を取得します。

### 所有モザイク一覧の取得

ノードに保存されている情報を取得するにはREST APIから特定のエンドポイントにアクセスし取得します。

今後、REST APIからデータを取得することが多くなります。以下のような関数を用意しいつでも利用できるようにしておくと便利です。なお、UnityでWebGLビルドの場合はHttpClientが使えないようなのでUnityWebRequestなどに置き換えて利用してください

```cs
static async Task<string> GetDataFromApi(string _node, string _param)
{
    var url = $"{_node}{_param}";
    using var client = new HttpClient();
    try
    {
        var response = await client.GetAsync(url);

        if (response.IsSuccessStatusCode) {
            return await response.Content.ReadAsStringAsync();
        }
        throw new Exception($"Error: {response.StatusCode}");
    }
    catch (Exception ex) {
        throw new Exception(ex.Message);
    }
}

static async Task<string> PostDataFromApi(string _node, string _param, object _obj)
{
    var url = $"{_node}{_param}";
    using var client = new HttpClient();
    try
    {
        var json = JsonSerializer.Serialize(_obj);
        var data = new StringContent(json, Encoding.UTF8, "application/json");
        var response = await client.PostAsync(url, data);

        if (response.IsSuccessStatusCode) {
            return await response.Content.ReadAsStringAsync();
        }
        throw new Exception($"Error: {response.StatusCode}");
    }
    catch (Exception ex) {
        throw new Exception(ex.Message);
    }
}
```

取得できるデータは文字列で返ってくるので、取得したい情報に関するクラスを事前に作成する必要があります。

今回はアカウントが所有するモザイク一覧を取得するため以下エンドポイントを叩きます。<br>
https://symbol.github.io/symbol-openapi/v1.0.3/#tag/Account-routes/operation/getAccountInfo

ここのレスポンスの箇所に合わせてクラスを作成します。
※以下ではJsonをデシリアライズするためのクラスを作成しますが、本書ではこれ以降はAPIから取得するJSONを`System.Text.Json.Nodes.JsonNode.Parse()`によってデシリアライズします。
Symbolとは関係の無いところになりますので各自、Newtonsoft.Jsonを使用するなどご自身のアプリケーションや環境に合わせてご利用ください。
同様にNullチェックなども適切に処理してください（本書では端折ります）
```cs
namespace AccountApi
{
    public class Account
    {
        public int version { get; set; }
        public string address { get; set; }
        public string addressHeight { get; set; }
        public string publicKey { get; set; }
        public string publicKeyHeight { get; set; }
        public int accountType { get; set; }
        public SupplementalPublicKeys supplementalPublicKeys { get; set; }
        public List<object> activityBuckets { get; set; }
        public List<Mosaic> mosaics { get; set; }
        public string importance { get; set; }
        public string importanceHeight { get; set; }
    }

    public class Mosaic
    {
        public string id { get; set; }
        public string amount { get; set; }
    }

    public class SupplementalPublicKeys
    {
    }

    public class AccountRoot
    {
        public Account account { get; set; }
        public string id { get; set; }
    }   
}
```


```cs
var account = JsonSerializer.Deserialize<AccountApi.AccountRoot>(await GetDataFromApi(node, $"/accounts/{aliceAddress}"));
if (account == null) throw new NullReferenceException("account is null");
foreach (var mosaic in account.account.mosaics)
    Console.WriteLine($"{mosaic.id} : {mosaic.amount}");
```
###### 出力例
```cs
> 72C0212E67A08BCE : 1068026343
> 00EC51045EECD5EB : 1
> 173AC1E38CBAD11D : 499999997
```

※nodeから取得するアドレスは16進数表記です。これをNやTから始まるアドレスに変換するには以下で可能です。
```cs
Console.WriteLine(Converter.AddressToString(Converter.HexToBytes("982982FFFC666CB09288FCB4B8F820E8B0B5F77093075AEF")));

> TAUYF774MZWLBEUI7S2LR6BA5CYLL53QSMDVV3Y
```

#### 表示桁数の調整

所有するトークンの量は誤差の発生を防ぐため、整数値で扱います。トークンの定義から可分性を取得することができるので、その値を使って正確な所有量を表示してみます。  

```cs
using System.Text.Json.Nodes;

foreach (var mosaic in account.account.mosaics)
{
    var m = JsonNode.Parse(await GetDataFromApi(node, $"/mosaics/{mosaic.id}"));
    var divisibility = (int)m["mosaic"]["divisibility"];
    var displayAmount = "";
    displayAmount = divisibility > 0 ? (double.Parse(mosaic.amount) / MathF.Pow(10, divisibility)).ToString(CultureInfo.InvariantCulture) : mosaic.amount;
    Console.WriteLine($"{mosaic.id} : {displayAmount}");
}
```
###### 出力例
```cs
> 72C0212E67A08BCE : 1068.026343
> 00EC51045EECD5EB : 1
> 173AC1E38CBAD11D : 499999997
```
## 3.4 現場で使えるヒント
### 暗号化と署名

アカウントとして生成した秘密鍵や公開鍵は、そのまま従来の暗号化や電子署名として活用することができます。信頼性に問題点があるアプリケーションを使用する必要がある場合も、個人間（エンドツーエンド）でデータの秘匿性・正当性を検証することができます。  

#### 事前準備：対話のためのBobアカウントを生成
```cs
var bobKeyPair = KeyPair.GenerateNewKeyPair();
var bobPublicKey = bobKeyPair.PublicKey;
```

以降の章でもBobアカウントを多用しますので秘密鍵を保存して復元可能な状態にしておきましょう。
また、手数料のために多少のXYMをFaucetから取得しておきます。

```cs
Console.WriteLine(bobKeyPair.PrivateKey);
var bobAddress = facade.Network.PublicKeyToAddress(bobPublicKey);
Console.WriteLine("https://testnet.symbol.tools/?recipient=" + bobAddress +"&amount=10");
```

#### 暗号化

Aliceの秘密鍵・Bobの公開鍵で暗号化し、Aliceの公開鍵・Bobの秘密鍵で復号します（AES-GCM形式）。

```cs
using CatSdk.Crypto;

const string message = "Hello Symol!";
var encryptedMessage = Crypto.Encode(alicePrivateKey, bobPublicKey, message);
Console.WriteLine(encryptedMessage);
```
```cs
> A6FC9E7A678768E8CBB38871FBA2D758A02B9904CC364DD5B19CD4289D4EBF8DF722025293980AD0
```

#### 復号
```cs
var decryptedMessage = Crypto.Decode(bobPrivateKey, alicePublicKey, "A6FC9E7A678768E8CBB38871FBA2D758A02B9904CC364DD5B19CD4289D4EBF8DF722025293980AD0");
Console.WriteLine(decryptedMessage);
```
```cs
> "Hello Symol!"
```

#### 署名

Aliceの秘密鍵でメッセージを署名し、Aliceの公開鍵と署名でメッセージを検証します。

```cs
const string message = "Hello Symol!";
var sig = aliceKeyPair.Sign(message);
Console.WriteLine($"Signature: {sig}");
```
```cs
> Signature: 07CDC49A2D132BFE24FFC11F1DCEF288356A2588B074DBCA4DC09853A5D5B6139381A12C8D161C5F00D10A6F38B8DC67377F27C9293B6665ED164DA54F41DC01
```

#### 検証
```cs
var verifier = new Verifier(alicePublicKey);
var verify = verifier.Verify(message, sig);
Console.WriteLine($"Verify: {verify}");
```
```cs
> Verify: True
```

ブロックチェーンを使用しない署名は何度も再利用される可能性があることにご注意ください。

### アカウントの保管

アカウントの管理方法について説明しておきます。  
秘密鍵はそのままで保存しないようにしてください。Aes方式による暗号化を利用して秘密鍵をパスフレーズで暗号化して保存する方法を紹介します。

#### 秘密鍵の暗号化

```cs
using CatSdk.Crypto;
var salt = Converter.BytesToHex(Crypto.RandomBytes(32));
var ciphertext = Crypto.EncryptString(alicePrivateKey.ToString(), "パスフレーズ", salt);
Console.WriteLine($"salt: {salt}");
Console.WriteLine($"ciphertext: {ciphertext}");
```
###### 出力例
```cs
> salt: 491258AA0CB6096CB9BCF31EDC8068066AC923D0EDDBAEE422BCEC7EECD4B126
> ciphertext: LAgaS3g8QueyJImgKbruALW/pi3k5Y6+rwFkwEdCwXtQxCDPUVK5LOWaprcXLdLcLDfx646l+PY5vS678kVwlFfX+w1i+3kd4YQM0pyK/fQ=
```
saltはこの例ではランダムな文字列ですが端末固有のキーなど固定値で無ければ良いと思います。
このsaltとパスフレーズがあれば復号が可能です。

#### 暗号化された秘密鍵の復号

```cs
var decryptString = Crypto.DecryptString("LAgaS3g8QueyJImgKbruALW/pi3k5Y6+rwFkwEdCwXtQxCDPUVK5LOWaprcXLdLcLDfx646l+PY5vS678kVwlFfX+w1i+3kd4YQM0pyK/fQ=", "パスフレーズ", "491258AA0CB6096CB9BCF31EDC8068066AC923D0EDDBAEE422BCEC7EECD4B126");
Console.WriteLine(decryptString);
```
###### 出力例
```cs
> 5DB8324E7EB83E7665D500B014283260EF312139034E86DFB7EE736503EA****
```

