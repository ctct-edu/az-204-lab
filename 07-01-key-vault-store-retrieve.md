# Azure Key Vault からシークレットを作成して取得する



この演習では、Azure Key Vault を作成し、Azure CLI を使用してシークレットを格納し、キー コンテナーからシークレットを作成および取得できる .NET コンソール アプリケーションを構築します。認証の構成方法、プログラムによるシークレットの管理方法、完了時にリソースをクリーンアップする方法を学習します。

この演習で実行されるタスク:

- Azure Key Vault リソースを作成する
- Azure CLI を使用してキー コンテナーにシークレットを格納する
- シークレットを作成および取得するための .NET コンソール アプリを作成する
- リソースをクリーンアップする

この演習は完了するまでに約**30**分かかります。

## Azure Key Vault リソースを作成し、シークレットを追加する



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
   keyVaultName=mykeyvaultnameXXXXXXXX
   ```

   

5. 次のコマンドを実行して、キー コンテナーの名前を取得し、その名前を記録します。演習の後半で必要になります。

   ```
   echo $keyVaultName
   ```

   

6. 次のコマンドを実行して、Azure Key Vault リソースを作成します。この処理の実行には数分かかる場合があります。

   ```
   az keyvault create --name $keyVaultName \
       --resource-group $resourceGroup --location $location
   ```

   

### Microsoft Entra ユーザー名にロールを割り当てる



シークレットを作成して取得するには、Microsoft Entra ユーザーを **Key Vault シークレット担当者**ロールに割り当てます。これにより、ユーザー アカウントにシークレットを設定、削除、一覧表示するアクセス許可が与えられます。一般的なシナリオでは、**Key Vault シークレット担当者**を 1 つのグループに割り当て、**Key Vault シークレット ユーザー** (シークレットを取得および一覧表示できる) を別のグループに割り当てることで、作成/読み取りアクションを分離できます。

1. 次のコマンドを実行して、アカウントから **userPrincipalName** を取得します。これは、ロールが割り当てられるユーザーを表します。

   ```
   userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
       --headers 'Content-Type=application/json' \
       --query userPrincipalName --output tsv)
   ```

   

2. 次のコマンドを実行して、キー コンテナーのリソース ID を取得します。リソース ID は、ロール割り当てのスコープを特定のキー コンテナーに設定します。

   ```
   resourceID=$(az keyvault show --resource-group $resourceGroup \
       --name $keyVaultName --query id --output tsv)
   ```

   

3. 次のコマンドを実行して、**Key Vault Secrets Officer** ロールを作成して割り当てます。

   ```
   az role assignment create --assignee $userPrincipal \
       --role "Key Vault Secrets Officer" \
       --scope $resourceID
   ```

   

次に、作成したキー コンテナーにシークレットを追加します。



### Azure CLI を使用してシークレットを追加および取得する



1. 次のコマンドを実行して、シークレットを作成します。

   ```
   az keyvault secret set --vault-name $keyVaultName \
       --name "MySecret" --value "My secret value"
   ```

   

2. 次のコマンドを実行してシークレットを取得し、シークレットが設定されていることを確認します。

   ```
   az keyvault secret show --name "MySecret" --vault-name $keyVaultName
   ```

   

   このコマンドは、いくつかの JSON を返します。

   以下のように、最後の行にはパスワードがプレーンテキストで含まれています。
   
   ```
   "value": "My secret value"
   ```
   
   

## シークレットを格納および取得するための .NET コンソール アプリを作成する



必要なリソースが Azure にデプロイされたので、次の手順はコンソール アプリケーションを設定することです。次の手順は、クラウドシェルで実行されます。

> **先端：**クラウドシェルのサイズを変更して、上部の境界線をドラッグして、より多くの情報とコードを表示します。[最小化] ボタンと [最大化] ボタンを使用して、クラウド シェルとメイン ポータル インターフェイスを切り替えることもできます。

1. 次のコマンドを実行して、プロジェクトを含むディレクトリを作成し、プロジェクト ディレクトリに変更します。

   ```
   mkdir keyvault
   cd keyvault
   ```

   

2. .NET コンソール アプリケーションを作成します。

   ```
   dotnet new console
   ```

   

3. 次のコマンドを実行して、**Azure.Identity** パッケージと **Azure.Security.KeyVault.Secrets** パッケージをプロジェクトに追加します。

   ```
   dotnet add package Azure.Identity
   dotnet add package Azure.Security.KeyVault.Secrets
   ```

   

### プロジェクトのスターターコードを追加する



1. クラウドシェルで次のコマンドを実行して、アプリケーションの編集を開始します。

   ```
   code Program.cs
   ```

   

2. 既存のコンテンツを次のコードに置き換えます。**YOUR-KEYVAULT-NAME** を実際のキー コンテナー名に置き換えてください。

   ```
   using Azure.Identity;
   using Azure.Security.KeyVault.Secrets;
   
   // Replace YOUR-KEYVAULT-NAME with your actual Key Vault name
   string KeyVaultUrl = "https://YOUR-KEYVAULT-NAME.vault.azure.net/";
   
   
   // ADD CODE TO CREATE A CLIENT
   
   
   
   // ADD CODE TO CREATE A MENU SYSTEM
   
   
   
   // ADD CODE TO CREATE A SECRET
   
   
   
   // ADD CODE TO LIST SECRETS
   ```

   

3. **ctrl+s** を押して変更を保存します。

### アプリケーションを完成させるためのコードを追加します



次に、アプリケーションを完成させるためのコードを追加します。

1. **[ADD CODE TO CREATE A CLIENT]** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Configure authentication options for connecting to Azure Key Vault
   DefaultAzureCredentialOptions options = new()
   {
       ExcludeEnvironmentCredential = true,
       ExcludeManagedIdentityCredential = true
   };
   
   // Create the Key Vault client using the URL and authentication credentials
   var client = new SecretClient(new Uri(KeyVaultUrl), new DefaultAzureCredential(options));
   ```

   

2. **ADD CODE TO CREATE A MENU SYSTEM** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   // Main application loop - continues until user types 'quit'
   while (true)
   {
       // Display menu options to the user
       Console.Clear();
       Console.WriteLine("\nPlease select an option:");
       Console.WriteLine("1. Create a new secret");
       Console.WriteLine("2. List all secrets");
       Console.WriteLine("Type 'quit' to exit");
       Console.Write("Enter your choice: ");
   
       // Read user input and convert to lowercase for easier comparison
       string? input = Console.ReadLine()?.Trim().ToLower();
       
       // Check if user wants to exit the application
       if (input == "quit")
       {
           Console.WriteLine("Goodbye!");
           break;
       }
   
       // Process the user's menu selection
       switch (input)
       {
           case "1":
               // Call the method to create a new secret
               await CreateSecretAsync(client);
               break;
           case "2":
               // Call the method to list all existing secrets
               await ListSecretsAsync(client);
               break;
           default:
               // Handle invalid input
               Console.WriteLine("Invalid option. Please enter 1, 2, or 'quit'.");
               break;
       }
   }
   ```

   

3. **ADD CODE TO CREATE A SECRET** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   async Task CreateSecretAsync(SecretClient client)
   {
       try
       {
           Console.Clear();
           Console.WriteLine("\nCreating a new secret...");
           
           // Get the secret name from user input
           Console.Write("Enter secret name: ");
           string? secretName = Console.ReadLine()?.Trim();
   
           // Validate that the secret name is not empty
           if (string.IsNullOrEmpty(secretName))
           {
               Console.WriteLine("Secret name cannot be empty.");
               return;
           }
           
           // Get the secret value from user input
           Console.Write("Enter secret value: ");
           string? secretValue = Console.ReadLine()?.Trim();
   
           // Validate that the secret value is not empty
           if (string.IsNullOrEmpty(secretValue))
           {
               Console.WriteLine("Secret value cannot be empty.");
               return;
           }
   
           // Create a new KeyVaultSecret object with the provided name and value
           var secret = new KeyVaultSecret(secretName, secretValue);
           
           // Store the secret in Azure Key Vault
           await client.SetSecretAsync(secret);
   
           Console.WriteLine($"Secret '{secretName}' created successfully!");
           Console.WriteLine("Press Enter to continue...");
           Console.ReadLine();
       }
       catch (Exception ex)
       {
           // Handle any errors that occur during secret creation
           Console.WriteLine($"Error creating secret: {ex.Message}");
       }
   }
   ```

   

4. **ADD CODE TO LIST SECRETS** コメントを見つけて、コメントの直後に次のコードを追加します。コードとコメントを必ず確認してください。

   ```
   async Task ListSecretsAsync(SecretClient client)
   {
       try
       {
           Console.Clear();
           Console.WriteLine("Listing all secrets in the Key Vault...");
           Console.WriteLine("----------------------------------------");
   
           // Get an async enumerable of all secret properties in the Key Vault
           var secretProperties = client.GetPropertiesOfSecretsAsync();
           bool hasSecrets = false;
   
           // Iterate through each secret property to retrieve full secret details
           await foreach (var secretProperty in secretProperties)
           {
               hasSecrets = true;
               try
               {
                   // Retrieve the actual secret value and metadata using the secret name
                   var secret = await client.GetSecretAsync(secretProperty.Name);
                   
                   // Display the secret information to the console
                   Console.WriteLine($"Name: {secret.Value.Name}");
                   Console.WriteLine($"Value: {secret.Value.Value}");
                   Console.WriteLine($"Created: {secret.Value.Properties.CreatedOn}");
                   Console.WriteLine("----------------------------------------");
               }
               catch (Exception ex)
               {
                   // Handle errors for individual secrets (e.g., access denied, secret not found)
                   Console.WriteLine($"Error retrieving secret '{secretProperty.Name}': {ex.Message}");
                   Console.WriteLine("----------------------------------------");
               }
           }
   
           // Inform user if no secrets were found in the Key Vault
           if (!hasSecrets)
           {
               Console.WriteLine("No secrets found in the Key Vault.");
           }
       }
       catch (Exception ex)
       {
           // Handle general errors that occur during the listing operation
           Console.WriteLine($"Error listing secrets: {ex.Message}");
       
       }
       Console.WriteLine("Press Enter to continue...");
       Console.ReadLine();
   }
   ```

   

5. **ctrl+s を押して**ファイルを保存し、**次に ctrl+q** を押してエディターを終了します。

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

3. 次のコマンドを実行して、コンソール アプリを起動します。アプリには、アプリケーションのメニュー システムが表示されます。

   ```
   dotnet run
   ```

   

4. この演習の冒頭でシークレットを作成しましたが、**2** を入力して取得して表示します。

5. **1** を入力し、シークレット名と値を入力してｑ、新しいシークレットを作成します。

6. シークレットをもう一度リストして、新しい追加を表示します。

アプリケーションの使用が終了したら、**quit** と入力します。



 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

2. ツール バーで、**リソース グループの削除** を選択します。

3. リソース グループ名を入力し、削除することを確認します。

   

     **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。

