# Azure Event Grid を使用してカスタム エンドポイントにイベントをルーティングする



この演習では、Azure Event Grid トピックと Web アプリ エンドポイントを作成し、カスタム イベントを Event Grid トピックに送信する .NET コンソール アプリケーションを構築します。イベント サブスクリプションを構成し、Event Grid で認証し、Web アプリでイベントを表示してイベントがエンドポイントに正常にルーティングされたことを確認する方法について説明します。

この演習で実行されるタスク:

- Azure Event Grid リソースを作成する
- Event Grid リソース プロバイダーを有効にする
- Event Grid でトピックを作成する
- メッセージエンドポイントを作成する
- トピックを購読する
- .NET コンソール アプリを使用してイベントを送信する
- リソースをクリーンアップする

この演習は完了するまでに約**25**分かかります。

## Azure Event Grid リソースを作成する



演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。「作業の開始」ウィンドウが表示された場合、以下のように操作します。

   ・「ストレージアカウントは不要です」を選択

   ・「サブスクリプション」をドロップダウンにて選択し「適用をクリック」

   クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

3. クラウド シェル ツール バーの [**設定**] メニューで、[**クラシック バージョンに移動**] を選択します (これはコード エディターを使用するために必要です)。

4. コマンドの多くは一意の名前を必要とし、同じパラメータを使用します。いくつかの変数を作成すると、リソースを作成するコマンドに必要な変更が減ります。次のコマンドを実行して、必要な変数を作成します。※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

   ```
   let rNum=XXXXXXXX
   resourceGroup=myResourceGroup
   location=eastus
   topicName="mytopic-evgtopic-${rNum}"
   siteName="evgsite-${rNum}"
   siteURL="https://${siteName}.azurewebsites.net"
   ```

   


### Event Grid リソース プロバイダーを有効にする



Azure リソース プロバイダーは、Azure 内の特定の種類のリソースを定義および管理するサービスです。これは、リソースをデプロイまたは管理するときに Azure がバックグラウンドで使用するものです。Event Grid リソース プロバイダーを **az provider register** コマンドで登録します。

```
az provider register --namespace Microsoft.EventGrid
```



登録が完了するまでに数分かかる場合があります。以下のコマンドで状態を確認できます。

```
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```



> **手記：**この手順は、以前に Event Grid を使用したことのないサブスクリプションでのみ必要です。



### Event Grid でトピックを作成する



**az eventgrid topic create** コマンドを使用してトピックを作成します。名前は DNS エントリの一部であるため、一意である必要があります。

```
az eventgrid topic create --name $topicName \
    --location $location \
    --resource-group $resourceGroup
```



### メッセージエンドポイントを作成する



カスタムトピックをサブスクライブする前に、イベントメッセージのエンドポイントを作成する必要があります。通常、エンドポイントはイベントデータに基づいてアクションを実行します。次のスクリプトでは、イベント メッセージを表示する事前構築済みの Web アプリを使用します。デプロイされたソリューションには、App Service プラン、App Service Web アプリ、GitHub のソース コードが含まれます。

1. 次のコマンドを実行して、メッセージエンドポイントを作成します。**echo** コマンドは、エンドポイントのサイト URL を表示します。

   ```
   az deployment group create \
       --resource-group $resourceGroup \
       --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
       --parameters siteName=$siteName hostingPlanName=viewerhost
   
   echo "Your web app URL: ${siteURL}"
   ```

   

   > **手記：**このコマンドは、完了するまでに数分かかる場合があります。

2. ブラウザーで新しいタブを開き、前のスクリプトの最後に生成された URL に移動して、Web アプリが実行されていることを確認します。現在メッセージが表示されていないサイトが表示されます。

   > **手記：**ブラウザを起動したままにしておくと、更新を表示するために使用されます。



### トピックを購読する



Event Grid トピックをサブスクライブして、追跡するイベントと、それらのイベントを送信する場所を Event Grid に指示します。

1. **az eventgrid event-subscription create** コマンドを使用してトピックをサブスクライブします。次のスクリプトは、アカウントからサブスクリプション ID を取得し、イベント サブスクリプションの作成に使用します。

   ```
   endpoint="${siteURL}/api/updates"
   topicId=$(az eventgrid topic show --resource-group $resourceGroup \
       --name $topicName --query "id" --output tsv)
   
   az eventgrid event-subscription create \
       --source-resource-id $topicId \
       --name TopicSubscription \
       --endpoint $endpoint
   ```

   

2. Web アプリをもう一度表示し、サブスクリプション検証イベントが送信されたことを確認します。目のアイコンを選択して、イベント データを展開します。Event Grid は、エンドポイントがイベント データを受信することを確認できるように、検証イベントを送信します。Web アプリには、サブスクリプションを検証するためのコードが含まれています。



## .NET コンソール アプリケーションを使用してイベントを送信する



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

> **手記：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir eventgrid
   cd eventgrid
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、**Azure.Messaging.EventGrid** パッケージと **dotenv.net** パッケージをプロジェクトに追加します。

   ```
   dotnet add package Azure.Messaging.EventGrid
   dotnet add package dotenv.net
   ```

   

### コンソール アプリケーションの構成



このセクションでは、トピックエンドポイントとアクセスキーを取得して、それらを **.env** ファイルに追加して、これらのシークレットを保持できるようにします。

1. 次のコマンドを実行して、前に作成したトピックの URL とアクセスキーを取得します。これらの値を必ず記録してください。

   ```
   az eventgrid topic show --name $topicName -g $resourceGroup --query "endpoint" --output tsv
   az eventgrid topic key list --name $topicName -g $resourceGroup --query "key1" --output tsv
   ```

   

2. 次のコマンドを実行して、シークレットを保持する **.env** ファイルを作成し、コード エディターで開きます。

   ```
   touch .env
   code .env
   ```

   

3. 次のコードを **.env** ファイルに追加します。**YOUR_TOPIC_ENDPOINT**と**YOUR_TOPIC_ACCESS_KEY**を前に記録した値に置き換えます。

   ```
   TOPIC_ENDPOINT="YOUR_TOPIC_ENDPOINT"
   TOPIC_ACCESS_KEY="YOUR_TOPIC_ACCESS_KEY"
   ```

   

4. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。

次に、クラウド シェルのエディターを使用して、**Program.cs** ファイル内のテンプレート コードを置き換えます。

### プロジェクトのコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコードを次のコードに置き換えます。コード内のコメントを必ず確認してください。

   ```
   using dotenv.net; 
   using Azure.Messaging.EventGrid; 
   
   // Load environment variables from .env file
   DotEnv.Load();
   var envVars = DotEnv.Read();
   
   // Start the asynchronous process to send an Event Grid event
   ProcessAsync().GetAwaiter().GetResult();
   
   async Task ProcessAsync()
   {
       // Retrieve Event Grid topic endpoint and access key from environment variables
       var topicEndpoint = envVars["TOPIC_ENDPOINT"];
       var topicKey = envVars["TOPIC_ACCESS_KEY"];
       
       // Check if the required environment variables are set
       if (string.IsNullOrEmpty(topicEndpoint) || string.IsNullOrEmpty(topicKey))
       {
           Console.WriteLine("Please set TOPIC_ENDPOINT and TOPIC_ACCESS_KEY in your .env file.");
           return;
       }
   
       // Create an EventGridPublisherClient to send events to the specified topic
       EventGridPublisherClient client = new EventGridPublisherClient
           (new Uri(topicEndpoint),
           new Azure.AzureKeyCredential(topicKey));
   
       // Create a new EventGridEvent with sample data
       var eventGridEvent = new EventGridEvent(
           subject: "ExampleSubject",
           eventType: "ExampleEventType",
           dataVersion: "1.0",
           data: new { Message = "Hello, Event Grid!" }
       );
   
       // Send the event to Azure Event Grid
       await client.SendEventAsync(eventGridEvent);
       Console.WriteLine("Event sent successfully.");
   }
   ```

   

3. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。

## Azure にサインインしてアプリを実行する



1. クラウド シェルのコマンド ライン ウィンドウで、次のコマンドを入力して Azure にサインインします。

   ```
   az login
   ```

   

   **クラウド シェル セッションが既に認証されている場合でも、Azure にサインインする必要があります。** 具体的には下記のメッセージが表示されたら、https:// ～ devicelogin のリンクをクリックし、「コード(毎回変わる)」を入力した後、portalのサインインで使用したアカウントを選択し、「続行」をクリックします。

   Cloud Shell is automatically authenticated under the initial account signed-in with. Run 'az login' only if you need to use a different account
   To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code 「コード(毎回変わる)」 to authenticate.

   > **注**: ほとんどのシナリオでは、*az login* を使用するだけで十分です。ただし、複数のテナントにサブスクリプションがある場合は、*--tenant* パラメーターを使用してテナントを指定する必要がある場合があります。詳細については、「[Azure CLI を使用して対話形式で Azure にサインインする」](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively)を参照してください。

2. Select a subscription and tenant (Type a number or Enter for no changes)　は何も入力せずEnterを押してください。

3. クラウドシェルで次のコマンドを実行して、コンソールアプリケーションを起動します。メッセージが**「イベントが正常に送信されました」**と表示されます。メッセージが送信されたとき。

   ```
   dotnet run
   ```

   

4. Web アプリを表示して、送信したばかりのイベントを確認します。目のアイコンを選択して、イベント データを展開します。

   

 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

2. ツール バーで、**リソース グループの削除** を選択します。

3. リソース グループ名を入力し、削除することを確認します。

   

   **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。

