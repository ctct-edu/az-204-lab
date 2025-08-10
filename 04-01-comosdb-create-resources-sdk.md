 # .NET を使用して Azure Cosmos DB for NoSQL でリソースを作成する

 

 この演習では、Azure Cosmos DB アカウントを作成し、Microsoft Azure Cosmos DB SDK を使用してデータベース、コンテナー、サンプル項目を作成する .NET コンソール アプリケーションを構築します。認証を構成し、データベース操作をプログラムで実行し、Azure portal で結果を確認する方法について説明します。

 この演習で実行されるタスク:

 - Azure Cosmos DB アカウントを作成する
 - データベース、コンテナー、およびアイテムを作成するコンソール アプリを作成する
 - コンソール アプリを実行し、結果を確認する

 この演習は完了するまでに約**30**分かかります。

 ## Azure Cosmos DB アカウントを作成する

 

 演習のこのセクションでは、リソース グループと Azure Cosmos DB アカウントを作成します。また、エンドポイントとアカウントのアクセス キーも記録します。

 1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

 2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。

 3. 「作業の開始」ウィンドウが表示された場合、以下のように操作します。

    ・「ストレージアカウントは不要です」を選択

    ・「ストレージ アカウントのサブスクリプション」で「AZ-204T00-A CSR1」を選択し「適用をクリック」

    クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

 4. クラウド シェル ツール バーの [**設定**] メニューで、[**クラシック バージョンに移動**] を選択します (これはコード エディターを使用するために必要です)。

 5. 多くのコマンドには一意の名前が必要で、同じパラメーターが使用されます。いくつかの変数を作成すると、リソースを作成するコマンドに必要な変更が減ります。次のコマンドを実行して、必要な変数を作成します。**myResourceGroup** を、この演習で使用している名前に置き換えます。

    ```
    resourceGroup=myResourceGrouplodXXXXXXXX
    accountName=cosmosexerciseXXXXXXXX
    ```

 

 6. 次のコマンドを実行して Azure Cosmos DB アカウントを作成します。各アカウント名は一意である必要があります。

    ```
    az cosmosdb create --name $accountName \
        --resource-group $resourceGroup
    ```

 

 7. 次のコマンドを実行して、Azure Cosmos DB アカウントの **documentEndpoint** を取得します。コマンドの結果からエンドポイントを記録します (演習の後半で必要になります)。

    ```
    az cosmosdb show --name $accountName \
        --resource-group $resourceGroup \
        --query "documentEndpoint" --output tsv
    ```

 

 8. 次のコマンドを使用して、アカウントの主キーを取得します。コマンドの結果から主キーを記録します。これは演習の後半で必要になります。

    ```
    az cosmosdb keys list --name $accountName \
        --resource-group $resourceGroup \
        --query "primaryMasterKey" --output tsv
    ```

 

 ## .NET コンソール アプリケーションを使用してデータ リソースと項目を作成する

 

 必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

 1. プロジェクトのフォルダを作成し、フォルダに変更します。

    ```
    mkdir cosmosdb
    cd cosmosdb
    ```

 

 2. .NET コンソール アプリを作成します。

    ```
    dotnet new console
    ```

 

 ### コンソール アプリケーションの構成

 

 1. 次のコマンドを実行して、**Microsoft.Azure.Cosmos**、**Newtonsoft.Json**、**dotenv.net** パッケージをプロジェクトに追加します。

    ```
    dotnet add package Microsoft.Azure.Cosmos --version 3.*
    dotnet add package Newtonsoft.Json --version 13.*
    dotnet add package dotenv.net
    ```

 

 2. 次のコマンドを実行して、シークレットを保持する **.env** ファイルを作成し、コード エディターで開きます。

    ```
    touch .env
    code .env
    ```

 

 3. 次のコードを **.env** ファイルに追加します。**YOUR_DOCUMENT_ENDPOINT**と**YOUR_ACCOUNT_KEY**を前に記録した値に置き換えます。

    ```
    DOCUMENT_ENDPOINT="YOUR_DOCUMENT_ENDPOINT"
    ACCOUNT_KEY="YOUR_ACCOUNT_KEY"
    ```

 

 4. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。

 次に、クラウド シェルのエディターを使用して、**Program.cs** ファイル内のテンプレート コードを置き換えます。

 ### プロジェクトの開始コードを追加する

 

 1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

    ```
    code Program.cs
    ```

 

 2. 既存のコードを次のコード スニペットに置き換えます。

    このコードは、アプリの全体的な構造を提供します。コード内のコメントを確認して、その仕組みを理解してください。アプリケーションを完成させるには、演習の後半で指定した領域にコードを追加します。

    ```
    using Microsoft.Azure.Cosmos;
    using dotenv.net;
    
    string databaseName = "myDatabase"; // Name of the database to create or use
    string containerName = "myContainer"; // Name of the container to create or use
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    string cosmosDbAccountUrl = envVars["DOCUMENT_ENDPOINT"];
    string accountKey = envVars["ACCOUNT_KEY"];
    
    if (string.IsNullOrEmpty(cosmosDbAccountUrl) || string.IsNullOrEmpty(accountKey))
    {
        Console.WriteLine("Please set the DOCUMENT_ENDPOINT and ACCOUNT_KEY environment variables.");
        return;
    }
    
    // CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY
    
    try
    {
        // CREATE A DATABASE IF IT DOESN'T ALREADY EXIST
    
    
        // CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY
    
    
        // DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER
    
    
        // ADD THE ITEM TO THE CONTAINER
    
    
    }
    catch (CosmosException ex)
    {
        // Handle Cosmos DB-specific exceptions
        // Log the status code and error message for debugging
        Console.WriteLine($"Cosmos DB Error: {ex.StatusCode} - {ex.Message}");
    }
    catch (Exception ex)
    {
        // Handle general exceptions
        // Log the error message for debugging
        Console.WriteLine($"Error: {ex.Message}");
    }
    
    // This class represents a product in the Cosmos DB container
    public class Product
    {
        public string? id { get; set; }
        public string? name { get; set; }
        public string? description { get; set; }
    }
    

 

 次に、プロジェクトの指定された領域にコードを追加して、クライアント、データベース、コンテナを作成し、コンテナにサンプル項目を追加します。

 

 ### クライアントを作成し、操作を実行するコードを追加します

 1. **CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY** コメントの後のスペースに次のコードを追加します。このコードは、Azure Cosmos DB アカウントへの接続に使用するクライアントを定義します。

    ```
    CosmosClient client = new(
        accountEndpoint: cosmosDbAccountUrl,
        authKeyOrResourceToken: accountKey
    );
    ```

 

     注: ベスト プラクティスは、*Azure ID* ライブラリの **DefaultAzureCredential** を使用することです。これには、サブスクリプションの設定方法に応じて、Azure で追加の構成要件が必要になる場合があります。

 2. **CREATE A DATABASE IF IT DOESN'T ALREADY EXIST** コメントの後のスペースに次のコードを追加します。

    ```
    Database database = await client.CreateDatabaseIfNotExistsAsync(databaseName);
    Console.WriteLine($"Created or retrieved database: {database.Id}");
    ```

 

 3. **CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY** コメントの後のスペースに次のコードを追加します。

    ```
    Container container = await database.CreateContainerIfNotExistsAsync(
        id: containerName,
        partitionKeyPath: "/id"
    );
    Console.WriteLine($"Created or retrieved container: {container.Id}");
    ```

 

 4. **DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER** コメントの後のスペースに次のコードを追加します。これにより、コンテナーに追加される項目が定義されます。

    ```
    Product newItem = new Product
    {
        id = Guid.NewGuid().ToString(), // Generate a unique ID for the product
        name = "Sample Item",
        description = "This is a sample item in my Azure Cosmos DB exercise."
    };
    ```

 

 5. **ADD THE ITEM TO THE CONTAINER** コメントの後のスペースに次のコードを追加します。

    ```
    ItemResponse<Product createResponse = await container.CreateItemAsync(
        item: newItem,
        partitionKey: new PartitionKey(newItem.id)
    );
    
    Console.WriteLine($"Created item with ID: {createResponse.Resource.id}");
    Console.WriteLine($"Request charge: {createResponse.RequestCharge} RUs");
    ```

 

 6. コードが完成したので、進行状況を保存し、**ctrl + s** を使用してファイルを保存し、**ctrl + q** を使用してエディターを終了します。

 7. クラウドシェルで次のコマンドを実行して、プロジェクト内のエラーをテストします。エラーが表示された場合は、エディターで*Program.cs*ファイルを開き、コードの欠落や貼り付けエラーがないか確認します。

    ```
    dotnet build
    ```

 

 プロジェクトが完了したので、次はアプリケーションを実行し、Azure portal で結果を確認します。

 

 ## アプリケーションを実行し、結果を確認する

 

 1. Cloud Shell の場合は `dotnet run` コマンドを実行します。出力は、次の例のようなものになります。

    ```
    Created or retrieved database: myDatabase
    Created or retrieved container: myContainer
    Created item: c549c3fa-054d-40db-a42b-c05deabbc4a6
    Request charge: 6.29 RUs
    ```

 

 2. Azure portal で、前に作成した Azure Cosmos DB リソースに移動します。左側のナビゲーションで **[データ エクスプローラー]** を選択します。**データ エクスプローラー**で、 [**myDatabase**] を選択し、[**myContainer]** を展開します。作成した項目は、[**項目]** を選択すると表示できます。

 ![](./media/cosmos-data-explorer.png)



