1. # Azure API Management を使用して API をインポートして構成する

    
   
    この演習では、Azure API Management インスタンスを作成し、OpenAPI 仕様バックエンド API をインポートし、Web サービス URL とサブスクリプション要件などの API 設定を構成し、API 操作をテストして正しく動作することを確認します。
    
    この演習で実行されるタスク:
    
    - Azure API Management (APIM) インスタンスを作成する
    - API のインポート
    - バックエンド設定を構成する
    - API をテストする
    
    この演習は完了するまでに約 **20** 分かかります。
    
    ## API Management インスタンスを作成する
    
    
    
    演習のこのセクションでは、リソース グループと Azure Storage アカウントを作成します。また、アカウントのエンドポイントとアクセス キーも記録します。
    
    1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。
    
    2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。
    
       > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成した場合は、***Bash*** に切り替えます。
    
    3. この演習に必要なリソースのリソース グループを作成します。**myResourceGroup** を、リソース グループに使用する名前に置き換えます。必要に応じて、**eastus2** を近くのリージョンに置き換えることができます。使用するリソース グループが既にある場合は、次の手順に進みます。
    
       ```azurecli
       az group create --location eastus2 --name myResourceGroup
       ```
    
       
    
    4. CLI コマンドで使用する変数をいくつか作成すると、入力の量が減ります。**myLocation** を前に選択した値に置き換えます。APIM 名はグローバルに一意の名前である必要があり、次のスクリプトはランダムな文字列を生成します。**myEmail** をアクセスできるメール アドレスに置き換えます。
    
       ```
       myApiName=import-apim-$RANDOM
       myLocation=myLocation
       myEmail=myEmail
       ```
    
       
    
    5. APIM インスタンスを作成します。**az apim create** コマンドを使用して、インスタンスを作成します。**myResourceGroup** を前に選択した値に置き換えます。
    
       ```
       az apim create -n $myApiName \
           --location $myLocation \
           --publisher-email $myEmail  \
           --resource-group myResourceGroup \
           --publisher-name Import-API-Exercise \
           --sku-name Consumption 
       ```
    
       
    
       > **手記：** 操作は約5分で完了するはずです。
    
    ## バックエンド API をインポートする
    
    
    
    このセクションでは、OpenAPI 仕様のバックエンド API をインポートして公開する方法について説明します。
    
    1. Azure portal で、**API Management サービス**を検索して選択します。
    
    2. [**API Management サービス**] 画面で、作成した API Management インスタンスを選択します。
    
    3. **API Management サービスの**ナビゲーション ウィンドウで、 [**> API]** を選択し、 [**API]** を選択します。
    
       [![Screenshot of the APIs section of the navigation pane.](https://github.com/MicrosoftLearning/mslearn-azure-developer/raw/main/instructions/azure-api-mgmt/media/select-apis-navigation-pane.png)](https://github.com/MicrosoftLearning/mslearn-azure-developer/blob/main/instructions/azure-api-mgmt/media/select-apis-navigation-pane.png)
    
    4. [**定義から作成]** セクションで **[OpenAPI**] を選択し、表示されるポップアップで **[基本/完全]** トグルを **[完全]** に設定します。
    
       [![Screenshot of the OpenAPI dialog box. Fields are detailed in the following table.](https://github.com/MicrosoftLearning/mslearn-azure-developer/raw/main/instructions/azure-api-mgmt/media/create-api.png)](https://github.com/MicrosoftLearning/mslearn-azure-developer/blob/main/instructions/azure-api-mgmt/media/create-api.png)
    
       次の表の値を使用してフォームに入力します。記載されていないフィールドはデフォルト値のままにしておくことができます。
    
       | 設定             | 価値                                       | 形容                                                         |
       | ---------------- | ------------------------------------------ | ------------------------------------------------------------ |
       | **OpenAPI 仕様** | `https://bigconference.azurewebsites.net/` | API を実装するサービスを参照し、リクエストはこのアドレスに転送されます。フォーム内の必要な情報のほとんどは、この値を入力すると自動的に入力されます。 |
       | **URL スキーム** | [**HTTPS]** を選択します。                 | API によって受け入れられる HTTP プロトコルのセキュリティ レベルを定義します。 |
    
    5. **[作成]** を選択します。
    
    
    
    ## API 設定を構成する
    
    
    
    *Big Conference API* が作成されます。次に、API 設定を構成します。
    
    1. メニューで**[設定**]を選択します。
    2. **[Web サービス URL**] フィールドに入力します。`https://bigconference.azurewebsites.net/`
    3. [**サブスクリプションが必要]** チェックボックスの選択を解除します。
    4. [**保存]** を選択します。
    
    
    
    ## API をテストする
    
    
    
    API がインポートされ、構成されたので、API をテストします。
    
    1. メニュー バーの [**テスト**] を選択します。これにより、API で使用可能なすべての操作が表示されます。
    
    2. **Speakers_Get**操作を検索して選択します。
    
    3. 送信 **を選択します。**HTTP 応答を表示するには、ページを下にスクロールする必要がある場合があります。
    
       バックエンドは **200 OK** といくつかのデータで応答します。



 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。

2. ツール バーで、**リソース グループの削除** を選択します。

3. リソース グループ名を入力し、削除することを確認します。

   

   **注意：** リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
