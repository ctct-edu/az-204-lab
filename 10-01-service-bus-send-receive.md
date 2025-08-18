# Azure Service Bus からメッセージを送受信する



この演習では、Azure Service Bus リソースを作成して構成し、**Azure.Messaging.ServiceBus** SDK を使用してメッセージを送受信する .NET アプリを構築します。Service Bus 名前空間とキューをプロビジョニングし、アクセス許可を割り当て、プログラムでメッセージを操作する方法について説明します。

この演習で実行されるタスク:

- Azure Service Bus リソースを作成する
- Microsoft Entra ユーザー名にロールを割り当てる
- メッセージを送受信するための .NET コンソール アプリを作成する
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
   namespaceName=svcbusnsXXXXXXXX
   ```

   

5. 名前空間に名前を割り当てる必要があるのは、この演習の後半で説明します。次のコマンドを実行し、出力を記録します。

   ```
   echo $namespaceName
   ```

   

### Azure Service Bus 名前空間とキューを作成する



1. Service Bus メッセージング名前空間を作成します。次のコマンドは、前に作成した変数を使用して名前空間を作成します。操作が完了するまでに数分かかります。

   ```
   az servicebus namespace create \
       --resource-group $resourceGroup \
       --name $namespaceName \
       --location $location
   ```

   

2. 名前空間が作成されたので、メッセージを保持するキューを作成する必要があります。次のコマンドを実行して、**myqueue** という名前のキューを作成します。

   ```
   az servicebus queue create --resource-group $resourceGroup \
       --namespace-name $namespaceName \
       --name myqueue
   ```

   

### Microsoft Entra ユーザー名にロールを割り当てる



アプリでメッセージを送受信できるようにするには、Microsoft Entra ユーザーを Service Bus 名前空間レベルで **Azure Service Bus データ所有者**ロールに割り当てます。これにより、ユーザー アカウントに、Azure RBAC を使用してキューとトピックを管理およびアクセスするアクセス許可が与えられます。クラウドシェルで次の手順を実行します。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。これは、ロールが割り当てられるユーザーを表します。

   ```
   userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
       --headers 'Content-Type=application/json' \
       --query userPrincipalName --output tsv)
   ```

   

2. 次のコマンドを実行して、Service Bus 名前空間のリソース ID を取得します。リソース ID は、ロール割り当てのスコープを特定の名前空間に設定します。

   ```
   resourceID=$(az servicebus namespace show --name $namespaceName \
       --resource-group $resourceGroup \
       --query id --output tsv)
   ```

   

3. 次のコマンドを実行して、**Azure Service Bus データ所有者**ロールを作成して割り当てます。

   ```
   az role assignment create --assignee $userPrincipal \
       --role "Azure Service Bus Data Owner" \
       --scope $resourceID
   ```

   

## メッセージを送受信するための .NET コンソール アプリを作成する



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

> **手記：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir svcbus
   cd svcbus
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、**Azure.Messaging.ServiceBus** パッケージと **Azure.Identity** パッケージをプロジェクトに追加します。

   ```
   dotnet add package Azure.Messaging.ServiceBus
   dotnet add package Azure.Identity
   ```

   

### プロジェクトのスターターコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコンテンツを次のコードに置き換えます。コード内のコメントを確認し、前に記録した Service Bus 名前空間(svcbusnsXXXXXXXX)に置き換えてください。

   ```
   using Azure.Messaging.ServiceBus;
   using Azure.Identity;
   using System.Timers;
   
   
   // TODO: Replace <YOUR-NAMESPACE> with your Service Bus namespace
   string svcbusNameSpace = "<YOUR-NAMESPACE>.servicebus.windows.net";
   string queueName = "myQueue";
   
   
   // ADD CODE TO CREATE A SERVICE BUS CLIENT
   
   
   
   // ADD CODE TO SEND MESSAGES TO THE QUEUE
   
   
   
   // ADD CODE TO PROCESS MESSAGES FROM THE QUEUE
   
   
   
   // Dispose client after use
   await client.DisposeAsync();
   ```

   

3. **ctrl+s** を押して変更を保存します。



### キューにメッセージを送信するコードを追加する



次に、Service Bus クライアントを作成し、メッセージのバッチをキューに送信するコードを追加します。

1. **ADD CODE TO CREATE A SERVICE BUS CLIENT** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
   DefaultAzureCredentialOptions options = new()
   {
       ExcludeEnvironmentCredential = true,
       ExcludeManagedIdentityCredential = true
   };
   
   // Create a Service Bus client using the namespace and DefaultAzureCredential
   // The DefaultAzureCredential will use the Azure CLI credentials, so ensure you are logged in
   ServiceBusClient client = new(svcbusNameSpace, new DefaultAzureCredential(options));
   ```

   

2. **ADD CODE TO SEND MESSAGES** TO THE QUEUE コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Create a sender for the specified queue
   ServiceBusSender sender = client.CreateSender(queueName);
   
   // create a batch 
   using ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync();
   
   // number of messages to be sent to the queue
   const int numOfMessages = 3;
   
   for (int i = 1; i <= numOfMessages; i++)
   {
       // try adding a message to the batch
       if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
       {
           // if it is too large for the batch
           throw new Exception($"The message {i} is too large to fit in the batch.");
       }
   }
   
   try
   {
       // Use the producer client to send the batch of messages to the Service Bus queue
       await sender.SendMessagesAsync(messageBatch);
       Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
   }
   finally
   {
       // Calling DisposeAsync on client types is required to ensure that network
       // resources and other unmanaged objects are properly cleaned up.
       await sender.DisposeAsync();
   }
   
   Console.WriteLine("Press any key to continue");
   Console.ReadKey();
   ```

   

3. **ctrl+s** キーを押してファイルを保存し、演習を続行します。



### キュー内のメッセージを処理するコードを追加する



1. **ADD CODE TO PROCESS MESSAGES FROM THE QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Create a processor that we can use to process the messages in the queue
   ServiceBusProcessor processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions());
   
   // Idle timeout in milliseconds, the idle timer will stop the processor if there are no more 
   // messages in the queue to process
   const int idleTimeoutMs = 3000;
   System.Timers.Timer idleTimer = new(idleTimeoutMs);
   idleTimer.Elapsed += async (s, e) =>
   {
       Console.WriteLine($"No messages received for {idleTimeoutMs / 1000} seconds. Stopping processor...");
       await processor.StopProcessingAsync();
   };
   
   try
   {
       // add handler to process messages
       processor.ProcessMessageAsync += MessageHandler;
   
       // add handler to process any errors
       processor.ProcessErrorAsync += ErrorHandler;
   
       // start processing 
       idleTimer.Start();
       await processor.StartProcessingAsync();
   
       Console.WriteLine($"Processor started. Will stop after {idleTimeoutMs / 1000} seconds of inactivity.");
       // Wait for the processor to stop
       while (processor.IsProcessing)
       {
           await Task.Delay(500);
       }
       idleTimer.Stop();
       Console.WriteLine("Stopped receiving messages");
   }
   finally
   {
       // Dispose processor after use
       await processor.DisposeAsync();
   }
   
   // handle received messages
   async Task MessageHandler(ProcessMessageEventArgs args)
   {
       string body = args.Message.Body.ToString();
       Console.WriteLine($"Received: {body}");
   
       // Reset the idle timer on each message
       idleTimer.Stop();
       idleTimer.Start();
   
       // complete the message. message is deleted from the queue. 
       await args.CompleteMessageAsync(args.Message);
   }
   
   // handle any errors when receiving messages
   Task ErrorHandler(ProcessErrorEventArgs args)
   {
       Console.WriteLine(args.Exception.ToString());
       return Task.CompletedTask;
   }
   ```

   

2. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。



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

3. 次のコマンドを実行して、コンソール アプリを起動します。アプリはさまざまな段階を一時停止し、続行するにはキーを押すように求めます。これにより、Azure portal でメッセージを表示する機会が得られます。

   ```
   dotnet run
   ```

   

4. Azure portal で、作成した Service Bus 名前空間に移動します。

5. [**概要**] ウィンドウの下部にある **[myqueue**] を選択します。

6. 左側のナビゲーション ウィンドウで **[Service Bus エクスプローラー**] を選択します。

7. [**最初からクイック表示**] を選択すると、数秒後に 3 つのメッセージが表示されます。

8. クラウド シェルで、任意のキーを押して続行すると、アプリケーションは 3 つのメッセージを処理します。

9. アプリケーションがメッセージの処理を完了したら、ポータルに戻ります。もう一度 [**最初からクイック表示**] を選択すると、キューにメッセージがないことを確認します。



 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

2. ツール バーで、**リソース グループの削除** を選択します。

3. リソース グループ名を入力し、削除することを確認します。

   

   **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。

