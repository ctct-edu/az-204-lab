# Azure Event Hubs からイベントを送信および取得する



この演習では、Azure Event Hubs リソースを作成し、**Azure.Messaging.EventHubs** SDK を使用してイベントを送受信する .NET コンソール アプリを構築します。クラウド リソースをプロビジョニングし、Event Hubs と対話し、完了したら環境をクリーンアップする方法について説明します。

この演習で実行されるタスク:

- リソース グループを作成する
- Azure Event Hubs リソースを作成する
- イベントを送信および取得する .NET コンソール アプリを作成する
- リソースをクリーンアップする

この演習は完了するまでに約**20**分かかります。



## Azure Event Hubs リソースを作成する



演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。「作業の開始」ウィンドウが表示された場合、以下のように操作します。

   ・「ストレージアカウントは不要です」を選択

   ・「サブスクリプション」をドロップダウンにて選択し「適用をクリック」

   クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

3. クラウド シェル ツール バーの [**設定**] メニューで、[**クラシック バージョンに移動**] を選択します (これはコード エディターを使用するために必要です)。

4. コマンドの多くは一意の名前を必要とし、同じパラメータを使用します。いくつかの変数を作成すると、リソースを作成するコマンドに必要な変更が減ります。次のコマンドを実行して、必要な変数を作成します。※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

   ```
   resourceGroup=myResourceGrouplodXXXXXXXX
   location=eastus
   namespaceName=eventhubsnsXXXXXXXX
   ```

   

### Azure Event Hubs 名前空間とイベント ハブを作成する



Azure Event Hubs 名前空間は、Azure 内のイベント ハブ リソースの論理コンテナーです。これは、大量のイベント データの取り込み、処理、および格納に使用される 1 つ以上のイベント ハブを作成できる一意のスコープ コンテナーを提供します。次の手順は、クラウドシェルで実行されます。

1. 次のコマンドを実行して、Event Hubs 名前空間を作成します。

   ```
   az eventhubs namespace create --name $namespaceName --resource-group $resourceGroup -l $location
   ```

   

2. 次のコマンドを実行して、Event Hubs 名前空間に **myEventHub** という名前のイベント ハブを作成します。

   ```
   az eventhubs eventhub create --name myEventHub --resource-group $resourceGroup \
     --namespace-name $namespaceName
   ```

   

### Microsoft Entra ユーザー名にロールを割り当てる



アプリでメッセージを送受信できるようにするには、Event Hubs 名前空間レベルで Microsoft Entra ユーザーを **Azure Event Hubs データ所有者**ロールに割り当てます。これにより、ユーザー アカウントに、Azure RBAC を使用してキューとトピックを管理およびアクセスするアクセス許可が与えられます。クラウドシェルで次の手順を実行します。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。これは、ロールが割り当てられるユーザーを表します。

   ```
   userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
       --headers 'Content-Type=application/json' \
       --query userPrincipalName --output tsv)
   ```

   

2. 次のコマンドを実行して、Event Hubs 名前空間のリソース ID を取得します。リソース ID は、ロール割り当てのスコープを特定の名前空間に設定します。

   ```
   resourceID=$(az eventhubs namespace show --resource-group $resourceGroup \
       --name $namespaceName --query id --output tsv)
   ```

   

3. 次のコマンドを実行して、イベントを送信および取得するアクセス許可を与える **Azure Event Hubs データ所有者**ロールを作成して割り当てます。

   ```
   az role assignment create --assignee $userPrincipal \
       --role "Azure Event Hubs Data Owner" \
       --scope $resourceID
   ```

   

## .NET コンソール アプリケーションを使用したイベントの送信と取得



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

> **先端：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir eventhubs
   cd eventhubs
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、**Azure.Messaging.EventHubs** パッケージと **Azure.Identity** パッケージをプロジェクトに追加します。

   ```
   dotnet add package Azure.Messaging.EventHubs
   dotnet add package Azure.Identity
   ```

   

次に、クラウド シェルのエディターを使用して、**Program.cs** ファイル内のテンプレート コードを置き換えます。



### プロジェクトのスターターコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコンテンツを次のコードに置き換えます。コード内のコメントを確認し、**YOUR_EVENT_HUB_NAMESPACE**をご自身のイベント ハブ名前空間(eventhubsnsXXXXXXXX)に置き換えてください。

   ```
   using Azure.Messaging.EventHubs;
   using Azure.Messaging.EventHubs.Producer;
   using Azure.Messaging.EventHubs.Consumer;
   using Azure.Identity;
   using System.Text;
   
   // TO-DO: Replace YOUR_EVENT_HUB_NAMESPACE with your actual Event Hub namespace
   string namespaceURL = "YOUR_EVENT_HUB_NAMESPACE.servicebus.windows.net";
   string eventHubName = "myEventHub"; 
   
   // Create a DefaultAzureCredentialOptions object to exclude certain credentials
   DefaultAzureCredentialOptions options = new()
   {
       ExcludeEnvironmentCredential = true,
       ExcludeManagedIdentityCredential = true
   };
   
   // Number of events to be sent to the event hub
   int numOfEvents = 3;
   
   // CREATE A PRODUCER CLIENT AND SEND EVENTS
   
   
   
   // CREATE A CONSUMER CLIENT AND RECEIVE EVENTS
   ```

   

3. **ctrl+s** を押して変更を保存します。



### アプリケーションを完成させるためのコードを追加します



このセクションでは、イベントを送受信するプロデューサー クライアントとコンシューマー クライアントを作成するコードを追加します。

1. **「CREATE A PRODUCER CLIENT AND SEND EVENTS」**コメントを見つけ、コメントの直後に次のコードを追加します。コード内のコメントを必ず確認してください。

   ```
   // Create a producer client to send events to the event hub
   EventHubProducerClient producerClient = new EventHubProducerClient(
       namespaceURL,
       eventHubName,
       new DefaultAzureCredential(options));
   
   // Create a batch of events 
   using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
   
   
   // Adding a random number to the event body and sending the events. 
   var random = new Random();
   for (int i = 1; i <= numOfEvents; i++)
   {
       int randomNumber = random.Next(1, 101); // 1 to 100 inclusive
       string eventBody = $"Event {randomNumber}";
       if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes(eventBody))))
       {
           // if it is too large for the batch
           throw new Exception($"Event {i} is too large for the batch and cannot be sent.");
       }
   }
   
   try
   {
       // Use the producer client to send the batch of events to the event hub
       await producerClient.SendAsync(eventBatch);
   
       Console.WriteLine($"A batch of {numOfEvents} events has been published.");
       Console.WriteLine("Press Enter to retrieve and print the events...");
       Console.ReadLine();
   }
   finally
   {
       await producerClient.DisposeAsync();
   }
   ```

   

2. **ctrl+s** を押して変更を保存します。

3. **CREATE A CONSUMER CLIENT AND RETRIEVE EVENTS** コメントを見つけて、コメントの直後に次のコードを追加します。コード内のコメントを必ず確認してください。

   ```
   // Create an EventHubConsumerClient
   await using var consumerClient = new EventHubConsumerClient(
       EventHubConsumerClient.DefaultConsumerGroupName,
       namespaceURL,
       eventHubName,
       new DefaultAzureCredential(options));
   
   Console.Clear();
   Console.WriteLine("Retrieving all events from the hub...");
   
   // Get total number of events in the hub by summing (last - first + 1) for all partitions
   // This count is used to determine when to stop reading events
   long totalEventCount = 0;
   string[] partitionIds = await consumerClient.GetPartitionIdsAsync();
   foreach (var partitionId in partitionIds)
   {
       PartitionProperties properties = await consumerClient.GetPartitionPropertiesAsync(partitionId);
       if (!properties.IsEmpty && properties.LastEnqueuedSequenceNumber >= properties.BeginningSequenceNumber)
       {
           totalEventCount += (properties.LastEnqueuedSequenceNumber - properties.BeginningSequenceNumber + 1);
       }
   }
   
   // Start retrieving events from the event hub and print to the console
   int retrievedCount = 0;
   await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync(startReadingAtEarliestEvent: true))
   {
       if (partitionEvent.Data != null)
       {
           string body = Encoding.UTF8.GetString(partitionEvent.Data.Body.ToArray());
           Console.WriteLine($"Retrieved event: {body}");
           retrievedCount++;
           if (retrievedCount >= totalEventCount)
           {
               Console.WriteLine("Done retrieving events. Press Enter to exit...");
               Console.ReadLine();
               return;
           }
       }
   }
   ```

   

4. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。



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

3. 次のコマンドを実行して、アプリケーションを起動します。

   ```
   dotnet run
   ```

   

   数秒後、次の例のような出力が表示されます。

   ```
   A batch of 3 events has been published.
   Press Enter to retrieve and print the events...
   
   Retrieving all events from the hub...
   Retrieved event: Event 4
   Retrieved event: Event 96
   Retrieved event: Event 74
   Done retrieving events. Press Enter to exit...
   ```

   

アプリケーションは常に 3 つのイベントをハブに送信しますが、ハブ内のすべてのイベントを取得します。アプリケーションを複数回実行すると、取得されるイベントの数が増えます。イベントの作成に使用される乱数は、さまざまなイベントを識別するのに役立ちます。



 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

2. ツール バーで、**リソース グループの削除** を選択します。

3. リソース グループ名を入力し、削除することを確認します。

   

   **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
