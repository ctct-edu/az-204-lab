# Azure App Configuration から構成設定を取得する



この演習では、Azure App Configuration リソースを作成し、Azure CLI を使用して構成設定を格納し、**ConfigurationBuilder** を使用して構成値を取得する .NET コンソール アプリケーションを構築します。階層キーを使用して設定を整理し、クラウドベースの構成データにアクセスするためにアプリケーションを認証する方法を学習します。

この演習で実行されるタスク:

- Azure App Configuration リソースを作成する
- 接続文字列の構成情報を格納する
- .NET コンソール アプリを作成して構成情報を取得する
- リソースをクリーンアップする

この演習は完了するまでに約 **15** 分かかります。

## Azure App Configuration リソースを作成し、構成情報を追加する



演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。「作業の開始」ウィンドウが表示された場合、以下のように操作します。

   ・「ストレージアカウントは不要です」を選択

   ・「サブスクリプション」をドロップダウンにて選択し「適用をクリック」

   クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

3. クラウド シェル ツール バーの [**設定**] メニューで、[**クラシック バージョンに移動**] を選択します (これはコード エディターを使用するために必要です)。

4. コマンドの多くは一意の名前を必要とし、同じパラメータを使用します。いくつかの変数を作成すると、リソースを作成するコマンドに必要な変更が減ります。次のコマンドを実行して、必要な変数を作成します。※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

   ```
   resourceGroup=myResourceGroup
   location=eastus
   appConfigName=appconfignameXXXXXXXX
   ```

   

6. 次のコマンドを実行して、App Configuration リソースの名前を取得します。演習の後半で必要になる名前を記録します。

   ```
   echo $appConfigName
   ```

   

7. 次のコマンドを実行して、**Microsoft.AppConfiguration** プロバイダーがサブスクリプションに登録されていることを確認します。

   ```
   az provider register --namespace Microsoft.AppConfiguration
   ```

   

8. 登録が完了するまでに数分かかる場合があります。以下のコマンドを実行して、登録の状態を確認します。結果が **Registered** を返したら、次の手順に進みます。

   ```
   az provider show --namespace Microsoft.AppConfiguration --query "registrationState"
   ```

   

9. 次のコマンドを実行して、Azure App Configuration リソースを作成します。この処理の実行には数分かかる場合があります。

   ```
   az appconfig create --location $location \
       --name $appConfigName \
       --resource-group $resourceGroup
   ```

   

### Microsoft Entra ユーザー名にロールを割り当てる



構成情報を取得するには、Microsoft Entra ユーザーを **App Configuration データ閲覧者**ロールに割り当てる必要があります。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。これは、ロールが割り当てられるユーザーを表します。

   ```
   userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
       --headers 'Content-Type=application/json' \
       --query userPrincipalName --output tsv)
   ```

   

2. 次のコマンドを実行して、App Configuration サービスのリソース ID を取得します。リソース ID は、ロール割り当てのスコープを設定します。

   ```
   resourceID=$(az appconfig show --resource-group $resourceGroup \
       --name $appConfigName --query id --output tsv)
   ```

   

3. 次のコマンドを実行して、**App Configuration Data Reader** ロールを作成して割り当てます。

   ```
   az role assignment create --assignee $userPrincipal \
       --role "App Configuration Data Reader" \
       --scope $resourceID
   ```

   

次に、プレースホルダー接続文字列を App Configuration に追加します。



### Azure CLI を使用して構成情報を追加する



Azure App Configuration では、**Dev:conStr** などのキーは階層キー、つまり名前空間キーです。コロン (:) は、論理階層を作成する区切り文字として機能します。

- **Dev** は、名前空間または環境プレフィックスを表します (この構成が開発環境用であることを示します)。
- **conStr** は、構成名を表します

この階層構造により、環境、機能、またはアプリケーション コンポーネントごとに構成設定を整理できるため、関連する設定の管理と取得が容易になります。

次のコマンドを実行して、プレースホルダー接続文字列を格納します。

```
az appconfig kv set --name $appConfigName \
    --key Dev:conStr \
    --value connectionString \
    --yes
```



このコマンドは、いくつかの JSON を返します。

以下のように、最後の行には、プレーンテキストの値が含まれています。

```
"value": "connectionString"
```



## 構成情報を取得するための .NET コンソール アプリを作成する



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

> **先端：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir appconfig
   cd appconfig
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、**Azure.Identity** パッケージと **Microsoft.Extensions.Configuration.AzureAppConfiguration** パッケージをプロジェクトに追加します。

   ```
   dotnet add package Azure.Identity
   dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
   ```

   

### プロジェクトのコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコンテンツを次のコードに置き換えます。必ず**YOUR_APP_CONFIGURATION_NAME**を前に記録した名前に置き換え、コード内のコメントに目を通してください。

   ```
   using Microsoft.Extensions.Configuration;
   using Microsoft.Extensions.Configuration.AzureAppConfiguration;
   using Azure.Identity;
   
   // Set the Azure App Configuration endpoint, replace YOUR_APP_CONFIGURATION_NAME
   // with the name of your actual App Configuration service
   
   string endpoint = "https://YOUR_APP_CONFIGURATION_NAME.azconfig.io"; 
   
   // Configure which authentication methods to use
   // DefaultAzureCredential tries multiple auth methods automatically
   DefaultAzureCredentialOptions credentialOptions = new()
   {
       ExcludeEnvironmentCredential = true,
       ExcludeManagedIdentityCredential = true
   };
   
   // Create a configuration builder to combine multiple config sources
   var builder = new ConfigurationBuilder();
   
   // Add Azure App Configuration as a source
   // This connects to Azure and loads configuration values
   builder.AddAzureAppConfiguration(options =>
   {
       
       options.Connect(new Uri(endpoint), new DefaultAzureCredential(credentialOptions));
   });
   
   // Build the final configuration object
   try
   {
       var config = builder.Build();
       
       // Retrieve a configuration value by key name
       Console.WriteLine(config["Dev:conStr"]);
   }
   catch (Exception ex)
   {
       Console.WriteLine($"Error connecting to Azure App Configuration: {ex.Message}");
   }
   ```

   

3. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。



## Azure にサインインしてアプリを実行する



1. クラウド シェルで、次のコマンドを入力して Azure にサインインします。

   ```
   az login
   ```

   

   **クラウド シェル セッションが既に認証されている場合でも、Azure にサインインする必要があります。** 具体的には下記のメッセージが表示されたら、https:// ～ devicelogin のリンクをクリックし、「コード(毎回変わる)」を入力した後、portalのサインインで使用したアカウントを選択し、「続行」をクリックします。

   Cloud Shell is automatically authenticated under the initial account signed-in with. Run 'az login' only if you need to use a different account
   To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code 「コード(毎回変わる)」 to authenticate.

   > **注**: ほとんどのシナリオでは、*az login* を使用するだけで十分です。ただし、複数のテナントにサブスクリプションがある場合は、*--tenant* パラメーターを使用してテナントを指定する必要がある場合があります。詳細については、「[Azure CLI を使用して対話形式で Azure にサインインする」](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)を参照してください。

2. Select a subscription and tenant (Type a number or Enter for no changes)　は何も入力せずEnterを押してください。

3. 次のコマンドを実行して、コンソール アプリを起動します。アプリには、演習の前半で **Dev:conStr** 設定に割り当てた **connectionString** 値が表示されます。

   ```
   dotnet run
   ```

   

   アプリには、演習の前半で **Dev:conStr** 設定に割り当てた **connectionString** 値が表示されます。

   

    ## リソースをクリーンアップする

    

    演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

   1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

   2. ツール バーで、**リソース グループの削除** を選択します。

   3. リソース グループ名を入力し、削除することを確認します。

      

        **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
