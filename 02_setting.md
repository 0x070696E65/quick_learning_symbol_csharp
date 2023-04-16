# 2.環境構築

本書の読み進め方について解説します。

## 2.1 使用言語

C#を使用します。

### SDK
symbol_cs_dual_sdk
https://github.com/0x070696E65/symbol_cs_dual_sdk/releases
 
### リファレンス
Symbol SDK for TypeScript and JavaScript  
https://0x070696e65.github.io/symbol_cs_dual_sdk_reference/

Catapult REST Endpoints (1.0.3)  
https://symbol.github.io/symbol-openapi/v1.0.3/

## 2.2 サンプルソースコード

### 出力値確認
Console.WriteLine()を変数の内容を出力します。状況（Unityの場合はDebug.Log等）に応じた出力関数に読み替えてお試しください。
出力内容は `>` 以下に記述しています。サンプルを実行する場合はこの部分を含まずに試してください。

### アカウント
#### Alice
本書では主にAliceアカウントを中心として解説します。  
3章で作成したAliceをその後の章でも引き続き使いますので、十分なXYMを送信した状態でお読みください。

#### Bob
Aliceとの送受信用のアカウントとして各章で必要に応じて作成します。その他、マルチシグの章などでCarolなどを使用します。

### 手数料
本書で紹介するトランザクションの手数料乗数は100でトランザクションを作成します。


## 2.3 事前準備
SDKをダウンロードしDllファイルを使用するプロジェクトの参照に追加してください。

ノード一覧より任意のノードを選択しノードURLを保存しておいてください。本書ではテストネットを前提として解説しています。

- テストネット
    - https://symbolnodes.org/nodes_testnet/
- メインネット
    - https://symbolnodes.org/nodes/

例）<br>
https://001-sai-dual.symboltest.net:3001

また、以下はほぼ全ての章で利用しますが、これ以降省略します。
Facadeの引数には適切なネットワークタイプを指定してください。（本書ではテストネットを使用します）

```cs
using CatSdk.Symbol;
using CatSdk.Facade;

var facade = new SymbolFacade(Network.TestNet);
const string node = "https://001-sai-dual.symboltest.net:3001";
```

これで準備完了です。  
