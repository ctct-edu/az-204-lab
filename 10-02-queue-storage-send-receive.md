# Azure Queue Storage からのメッセージの送受信



この演習では、Azure Queue Storage リソースを作成して構成し、**Azure.Storage.Queues** SDK を使用してメッセージを送受信する .NET アプリを構築します。ストレージ リソースのプロビジョニング、キュー メッセージの管理、および完了後の環境のクリーンアップの方法を学習します。

この演習で実行されるタスク:

- Azure Queue Storage リソースを作成する
- Microsoft Entra ユーザー名にロールを割り当てる
- メッセージを送受信するための .NET コンソール アプリを作成する
- リソースをクリーンアップする

この演習は完了するまでに約**30**分かかります。



## Azure Queue Storage リソースを作成する



演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。「作業の開始」ウィンドウが表示された場合、以下のように操作します。

   ・「ストレージアカウントは不要です」を選択

   ・「サブスクリプション」をドロップダウンにて選択し「適用をクリック」

   クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

3. クラウド シェル ツール バーの [**設定**] メニューで、[**クラシック バージョンに移動**] を選択します (これはコード エディターを使用するために必要です)。

5. コマンドの多くは一意の名前を必要とし、同じパラメータを使用します。いくつかの変数を作成すると、リソースを作成するコマンドに必要な変更が減ります。次のコマンドを実行して、必要な変数を作成します。※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

   ```
   resourceGroup=myResourceGrouplodXXXXXXXX
   location=eastus
   storAcctName=storactnameXXXXXXXX
   ```

   

6. この演習の後半で、ストレージ アカウントに割り当てられた名前が必要です。次のコマンドを実行し、出力を記録します。

   ```
   echo $storAcctName
   ```

   

7. 次のコマンドを実行して、前に作成した変数を使用してストレージ アカウントを作成します。操作が完了するまでに数分かかります。

   ```
   az storage account create --resource-group $resourceGroup \
       --name $storAcctName --location $location --sku Standard_LRS
   ```

   

### Microsoft Entra ユーザー名にロールを割り当てる



アプリでメッセージを送受信できるようにするには、Microsoft Entra ユーザーを**ストレージ キュー データ共同作成者**ロールに割り当てます。これにより、ユーザー アカウントに、キューを作成し、Azure RBAC を使用してメッセージを送受信するアクセス許可が与えられます。クラウドシェルで次の手順を実行します。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。これは、ロールが割り当てられるユーザーを表します。

   ```
   userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
       --headers 'Content-Type=application/json' \
       --query userPrincipalName --output tsv)
   ```

   

2. 次のコマンドを実行して、ストレージ アカウントのリソース ID を取得します。リソース ID は、ロール割り当てのスコープを特定の名前空間に設定します。

   ```
   resourceID=$(az storage account show --resource-group $resourceGroup \
       --name $storAcctName --query id --output tsv)
   ```

   

3. 次のコマンドを実行して、**ストレージ キュー データ共同作成者**ロールを作成して割り当てます。

   ```
   az role assignment create --assignee $userPrincipal \
       --role "Storage Queue Data Contributor" \
       --scope $resourceID
   ```

   

## メッセージを送受信するための .NET コンソール アプリを作成する



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

> **先端：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir queuestor
   cd queuestor
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、**Azure.Storage.Queues** パッケージと **Azure.Identity** パッケージをプロジェクトに追加します。

   ```
   dotnet add package Azure.Storage.Queues
   dotnet add package Azure.Identity
   ```

   

### プロジェクトのスターターコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコンテンツを次のコードに置き換えます。コード内のコメントを確認し、前に記録したストレージ アカウント名に置き換えてください。

   ```
   using Azure;
   using Azure.Identity;
   using Azure.Storage.Queues;
   using Azure.Storage.Queues.Models;
   using System;
   using System.Threading.Tasks;
   
   // Create a unique name for the queue
   // TODO: Replace the <YOUR-STORAGE-ACCT-NAME> placeholder 
   string queueName = "myqueue-" + Guid.NewGuid().ToString();
   string storageAccountName = "<YOUR-STORAGE-ACCT-NAME>";
   
   // ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE
   
   
   
   // ADD CODE TO SEND AND LIST MESSAGES
   
   
   
   // ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES
   
   
   
   // ADD CODE TO DELETE MESSAGES AND THE QUEUE
   ```

   

3. **ctrl+s** を押して変更を保存します。



### キュー クライアントを作成し、キューを作成するコードを追加します



次に、キュー ストレージ クライアントを作成し、キューを作成するコードを追加します。

1. **ADD CODE TO CREATE A QUEUE CLIENT と CREATE A QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Create a DefaultAzureCredentialOptions object to exclude certain credentials
   DefaultAzureCredentialOptions options = new()
   {
       ExcludeEnvironmentCredential = true,
       ExcludeManagedIdentityCredential = true
   };
   
   // Instantiate a QueueClient to create and interact with the queue
   QueueClient queueClient = new QueueClient(
       new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
       new DefaultAzureCredential(options));
   
   Console.WriteLine($"Creating queue: {queueName}");
   
   // Create the queue
   await queueClient.CreateAsync();
   
   Console.WriteLine("Queue created, press Enter to add messages to the queue...");
   Console.ReadLine();
   ```

   

2. **ctrl+s** キーを押してファイルを保存し、演習を続行します。



### キュー内のメッセージを送信および一覧表示するコードを追加する



1. **ADD CODE TO SEND AND LIST MESSAGES** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Send several messages to the queue with the SendMessageAsync method.
   await queueClient.SendMessageAsync("Message 1");
   await queueClient.SendMessageAsync("Message 2");
   
   // Send a message and save the receipt for later use
   SendReceipt receipt = await queueClient.SendMessageAsync("Message 3");
   
   Console.WriteLine("Messages added to the queue. Press Enter to peek at the messages...");
   Console.ReadLine();
   
   // Peeking messages lets you view the messages without removing them from the queue.
   
   foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
   {
       Console.WriteLine($"Message: {message.MessageText}");
   }
   
   Console.WriteLine("\nPress Enter to update a message in the queue...");
   Console.ReadLine();
   ```

   

2. **ctrl+s** キーを押してファイルを保存し、演習を続行します。



### メッセージを更新し、結果を一覧表示するコードを追加する



1. **ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Update a message with the UpdateMessageAsync method and the saved receipt
   await queueClient.UpdateMessageAsync(receipt.MessageId, receipt.PopReceipt, "Message 3 has been updated");
   
   Console.WriteLine("Message three updated. Press Enter to peek at the messages again...");
   Console.ReadLine();
   
   
   // Peek messages from the queue to compare updated content
   foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
   {
       Console.WriteLine($"Message: {message.MessageText}");
   }
   
   Console.WriteLine("\nPress Enter to delete messages from the queue...");
   Console.ReadLine();
   ```

   

2. **ctrl+s** キーを押してファイルを保存し、演習を続行します。



### メッセージとキューを削除するコードを追加する



1. **ADD CODE TO DELETE MESSAGES と THE QUEUE** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Delete messages from the queue with the DeleteMessagesAsync method.
   foreach (var message in (await queueClient.ReceiveMessagesAsync(maxMessages: 10)).Value)
   {
       // "Process" the message
       Console.WriteLine($"Deleting message: {message.MessageText}");
   
       // Let the service know we're finished with the message and it can be safely deleted.
       await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
   }
   Console.WriteLine("Messages deleted from the queue.");
   Console.WriteLine("\nPress Enter key to delete the queue...");
   Console.ReadLine();
   
   // Delete the queue with the DeleteAsync method.
   Console.WriteLine($"Deleting queue: {queueClient.Name}");
   await queueClient.DeleteAsync();
   
   Console.WriteLine("Done");
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

3. 次のコマンドを実行して、コンソール アプリを起動します。アプリは実行中に何度も一時停止し、任意のキーを押して続行するのを待ちます。これにより、Azure portal でメッセージを表示する機会が得られます。

   ```
   dotnet run
   ```

   

4. Azure portal で、作成した Azure Storage アカウントに移動します。

5. 左側>ナビゲーションで [**データ ストレージ**] を展開し、[**キュー]** を選択します。

6. アプリケーションが作成するキューを選択すると、送信されたメッセージを表示し、アプリケーションの動作を監視できます。



 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

2. ツール バーで、**リソース グループの削除** を選択します。

3. リソース グループ名を入力し、削除することを確認します。

   

   **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
