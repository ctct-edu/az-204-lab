# .NET クライアント ライブラリを使用して BLOB ストレージ リソースを作成する



この演習では、Azure Storage アカウントを作成し、Azure Storage BLOB クライアント ライブラリを使用して .NET コンソール アプリケーションを構築し、コンテナーの作成、BLOB ストレージへのファイルのアップロード、BLOB の一覧表示、ファイルのダウンロードを行います。Azure で認証し、BLOB ストレージ操作をプログラムで実行し、Azure portal で結果を確認する方法について説明します。

この演習で実行されるタスク:

- Azure リソースを準備する
- コンソールアプリを作成してデータを作成およびダウンロードする
- アプリを実行して結果を確認する
- リソースをクリーンアップする

この演習は完了するまでに約**30**分かかります。

## Azure Storage アカウントを作成する



演習のこのセクションでは、Azure CLI を使用して Azure に必要なリソースを作成します。

1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。「作業の開始」ウィンドウが表示された場合、以下のように操作します。

   ・「ストレージアカウントは不要です」を選択

   ・「サブスクリプション」をドロップダウンにて選択し「適用をクリック」

   クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

3. クラウド シェル ツール バーの [**設定**] メニューで、[**クラシック バージョンに移動**] を選択します (これはコード エディターを使用するために必要です)。

4. 多くのコマンドには一意の名前が必要で、同じパラメーターが使用されます。いくつかの変数を作成すると、リソースを作成するコマンドに必要な変更が減ります。次のコマンドを実行して、必要な変数を作成します。**myResourceGroup** を、この演習で使用している名前に置き換えます。

   ```
   resourceGroup=myResourceGrouplodXXXXXXXX
   location=eastus
   accountName=storageacct$RANDOM
   ```

   

5. 次のコマンドを実行して Azure Storage アカウントを作成します。各アカウント名は一意である必要があります。最初のコマンドは、ストレージ アカウントの一意の名前を持つ変数を作成します。**echo** コマンドの出力からアカウントの名前を記録します。

   ```
   az storage account create --name $accountName \
       --resource-group $resourceGroup \
       --location $location \
       --sku Standard_LRS 
   
   echo $accountName
   ```

   

### Microsoft Entra ユーザー名にロールを割り当てる

アプリでリソースと項目を作成できるようにするには、Microsoft Entra ユーザーを**ストレージ BLOB データ所有者**ロールに割り当てます。クラウドシェルで次の手順を実行します。

> **先端：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。これは、ロールが割り当てられるユーザーを表します。

   ```
   userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
       --headers 'Content-Type=application/json' \
       --query userPrincipalName --output tsv)
   ```

   

2. 次のコマンドを実行して、ストレージ アカウントのリソース ID を取得します。リソース ID は、ロール割り当てのスコープを特定の名前空間に設定します。

   ```
   resourceID=$(az storage account show --name $accountName \
       --resource-group $resourceGroup \
       --query id --output tsv)
   ```

   

3. 次のコマンドを実行して、**ストレージ BLOB データ所有者**ロールを作成して割り当てます。このロールは、コンテナーとアイテムを管理するためのアクセス許可を付与します。

   ```
   az role assignment create --assignee $userPrincipal \
       --role "Storage Blob Data Owner" \
       --scope $resourceID
   ```

   

## .NET コンソール アプリを作成してコンテナーと項目を作成する



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir azstor
   cd azstor
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、必要なパッケージをアプリケーションに追加します。

   ```
   dotnet add package Azure.Storage.Blobs
   dotnet add package Azure.Identity
   ```

   

4. 次のコマンドを実行して、プロジェクトに**データ** フォルダーを作成します。

   ```
   mkdir data
   ```

   

次に、プロジェクトのコードを追加します。

### プロジェクトのスターターコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコンテンツを次のコードに置き換えます。コード内のコメントを必ず確認してください。

   ```
   using Azure.Storage.Blobs;
   using Azure.Storage.Blobs.Models;
   using Azure.Identity;
   
   Console.WriteLine("Azure Blob Storage exercise\n");
   
   // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
   DefaultAzureCredentialOptions options = new()
   {
       ExcludeEnvironmentCredential = true,
       ExcludeManagedIdentityCredential = true
   };
   
   // Run the examples asynchronously, wait for the results before proceeding
   ProcessAsync().GetAwaiter().GetResult();
   
   Console.WriteLine("\nPress enter to exit the sample application.");
   Console.ReadLine();
   
   async Task ProcessAsync()
   {
       // CREATE A BLOB STORAGE CLIENT
       
   
   
       // CREATE A CONTAINER
       
   
   
       // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
       
   
   
       // UPLOAD THE FILE TO BLOB STORAGE
       
   
   
       // LIST BLOBS IN THE CONTAINER
       
   
   
       // DOWNLOAD THE BLOB TO A LOCAL FILE
       
   
   }
   ```

   

3. **ctrl+s** を押して変更を保存し、次の手順に進みます。

## プロジェクトを完了するためのコードを追加する



演習の残りの部分では、指定された領域にコードを追加して、完全なアプリケーションを作成します。

1. **CREATE A BLOB STORAGE CLIENT** コメントを見つけ、コメントのすぐ下に次のコードを追加します。**BlobServiceClient** は、ストレージ アカウント内のコンテナーと BLOB を管理するための主要なエントリ ポイントとして機能します。クライアントは、認証に *DefaultAzureCredential* を使用します。必ず**YOUR_ACCOUNT_NAME** を今回作成したストレージアカウントの名前に置き換えてください。

   ```
   // Create a credential using DefaultAzureCredential with configured options
   string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name
   
   // Use the DefaultAzureCredential with the options configured at the top of the program
   DefaultAzureCredential credential = new DefaultAzureCredential(options);
   
   // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
   string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
   BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
   ```

   

2. **ctrl+s** を押して変更を保存し、次の手順に進みます。

3. **CREATE A CONTAINER** コメントを見つけて、コメントのすぐ下に次のコードを追加します。コンテナーの作成には、**BlobServiceClient** クラスのインスタンスを作成し、**CreateBlobContainerAsync** メソッドを呼び出してストレージ アカウントにコンテナーを作成することが含まれます。GUID 値は、コンテナー名が一意であることを確認するために追加されます。**CreateBlobContainerAsync** メソッドは、コンテナーが既に存在する場合に失敗します。

   ```
   ///Create a unique name for the container
   string containerName = "wtblob" + Guid.NewGuid().ToString();
   
   // Create the container and return a container client object
   Console.WriteLine("Creating container: " + containerName);
   BlobContainerClient containerClient = 
       await blobServiceClient.CreateBlobContainerAsync(containerName);
   
   // Check if the container was created successfully
   if (containerClient != null)
   {
       Console.WriteLine("Container created successfully, press 'Enter' to continue.");
       Console.ReadLine();
   }
   else
   {
       Console.WriteLine("Failed to create the container, exiting program.");
       return;
   }
   ```

   

4. **ctrl+s** を押して変更を保存し、次の手順に進みます。

5. **CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE** コメントを見つけ、コメントのすぐ下に次のコードを追加します。これにより、コンテナーにアップロードされるファイルがデータディレクトリーに作成されます。

   ```
   // Create a local file in the ./data/ directory for uploading and downloading
   Console.WriteLine("Creating a local file for upload to Blob storage...");
   string localPath = "./data/";
   string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
   string localFilePath = Path.Combine(localPath, fileName);
   
   // Write text to the file
   await File.WriteAllTextAsync(localFilePath, "Hello, World!");
   Console.WriteLine("Local file created, press 'Enter' to continue.");
   Console.ReadLine();
   ```

   

6. **ctrl+s** を押して変更を保存し、次の手順に進みます。

7. **UPLOAD THE FILE TO BLOB STORAGE** コメントを見つけ、コメントのすぐ下に次のコードを追加します。このコードは、前のセクションで作成したコンテナーで **GetBlobClient** メソッドを呼び出すことで、**BlobClient** オブジェクトへの参照を取得します。次に、**UploadAsync** メソッドを使用して生成されたローカル ファイルをアップロードします。このメソッドは、BLOB がまだ存在しない場合は作成し、存在する場合は上書きします。

   ```
   // Get a reference to the blob and upload the file
   BlobClient blobClient = containerClient.GetBlobClient(fileName);
   
   Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);
   
   // Open the file and upload its data
   using (FileStream uploadFileStream = File.OpenRead(localFilePath))
   {
       await blobClient.UploadAsync(uploadFileStream);
       uploadFileStream.Close();
   }
   
   // Verify if the file was uploaded successfully
   bool blobExists = await blobClient.ExistsAsync();
   if (blobExists)
   {
       Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
       Console.ReadLine();
   }
   else
   {
       Console.WriteLine("File upload failed, exiting program..");
       return;
   }
   ```

   

8. **ctrl+s** を押して変更を保存し、次の手順に進みます。

9. **LIST BLOBS IN THE CONTAINER** コメントを見つけ、コメントのすぐ下に次のコードを追加します。コンテナー内の BLOB は、**GetBlobsAsync** メソッドを使用して一覧表示します。この場合、コンテナーに追加された BLOB は 1 つだけであるため、一覧表示操作はその 1 つの BLOB のみを返します。

   ```
   Console.WriteLine("Listing blobs in container...");
   await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
   {
       Console.WriteLine("\t" + blobItem.Name);
   }
   
   Console.WriteLine("Press 'Enter' to continue.");
   Console.ReadLine();
   ```

   

10. **ctrl+s** を押して変更を保存し、次の手順に進みます。

11. **DOWNLOAD THE BLOB TO A LOCAL FILE** コメントを見つけ、コメントのすぐ下に次のコードを追加します。このコードでは、**DownloadAsync** メソッドを使用して、以前に作成した BLOB をローカル ファイル システムにダウンロードします。このコード例では、BLOB 名に "DOWNLOADED" のサフィックスを追加して、両方のファイルをローカル ファイル システムで表示できるようにします。

    ```
    // Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
    // overwrite the original file
    
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
    
    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);
    
    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();
    
    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }
    
    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
    ```

    

12. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。

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

3. 次のコマンドを実行して、コンソール アプリを起動します。アプリは実行中に何度も一時停止し、続行するために任意のキーを押すのを待ちます。これにより、Azure portal でメッセージを表示する機会が得られます。

   ```
   dotnet run
   ```

   

4. Azure portal で、作成した Azure Storage アカウントに移動します。

5. 左側のナビゲーションで [**データ ストレージ] >**展開し、 **[コンテナー]** を選択します。

6. アプリケーションが作成したコンテナーを選択すると、アップロードされた BLOB を表示できます。

7. 以下の 2 つのコマンドを実行して**データ**ディレクトリに移動し、アップロードおよびダウンロードされたファイルを一覧表示します。

   ```
   cd data
   ls
   ```

   

 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

  1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。
  2. ツール バーで、**リソース グループの削除** を選択します。
  3. リソース グループ名を入力し、削除することを確認します。

  **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
