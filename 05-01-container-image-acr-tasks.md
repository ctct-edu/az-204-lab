 # Azure Container Registry タスクを使用してコンテナー イメージをビルドして実行する

 

 この演習では、アプリケーション コードからコンテナー イメージをビルドし、Azure CLI を使用して Azure Container Registry にプッシュします。コンテナー化用にアプリを準備し、ACR インスタンスを作成し、コンテナー イメージを Azure に格納する方法について説明します。

 この演習で実行されるタスク:

 - Azure Container Registry リソースを作成する
 - Dockerfile からイメージをビルドしてプッシュする
 - 結果を確認する
 - Azure Container Registry でイメージを実行する

 この演習は完了するまでに約 **15** 分かかります。

 ## Azure Container Registry リソースを作成する

 1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

 2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。「作業の開始」ウィンドウが表示された場合、以下のように操作します。

     ・「ストレージアカウントは不要です」を選択

     ・「サブスクリプション」をドロップダウンにて選択し「適用をクリック」

     クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

 3. 次のコマンドを実行して、基本的なコンテナーレジストリを作成します。レジストリ名は Azure 内で一意で、5 文字から 50 文字の英数字を含める必要があります。**myContainerRegistry** を一意の値に置き換えます。

    ※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

    ```
    az acr create --resource-group myResourceGroup \
        --name mycontainerregistryXXXXXXXX --sku Basic
    ```

    

     **手記：** このコマンドは、Azure Container Registry について学習する開発者向けのコスト最適化オプションである *Basic* レジストリを作成します。

 ## Dockerfile からイメージをビルドしてプッシュする

 

 次に、Dockerfile に基づいてイメージをビルドしてプッシュします。

 1. 次のコマンドを実行して、Dockerfile を作成します。Dockerfile には、Microsoft Container Registry でホストされている *hello-world* イメージを参照する 1 行が含まれています。

    ```
    echo FROM mcr.microsoft.com/hello-world > Dockerfile
    ```

    

 2. 次の **az acr build** コマンドを実行して、イメージをビルドし、イメージが正常にビルドされたら、レジストリにプッシュします。**myContainerRegistry** を前に作成した名前に置き換えます。

    ※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

    ```
    az acr build --image sample/hello-world:v1  \
        --registry mycontainerregistryXXXXXXXX \
        --file Dockerfile .
    ```

    

    以下は、前のコマンドからの出力の短縮サンプルであり、最終結果の最後の数行を示しています。*リポジトリ* フィールドに、*sample/hello-word* イメージが一覧表示されます。

    ```
    - image:
        registry: myContainerRegistry.azurecr.io
        repository: sample/hello-world
        tag: v1
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      runtime-dependency:
        registry: mcr.microsoft.com
        repository: hello-world
        tag: latest
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      git: {}
    
    
    Run ID: cf1 was successful after 11s
    ```

    

 ## 結果を確認する

 

 1. 次のコマンドを実行して、レジストリ内のリポジトリを一覧表示します。

    ※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

    ```
    az acr repository list --name mycontainerregistryXXXXXXXX --output table
    ```

    

    アウトプット：

    ```
    Result
    ----------------
    sample/hello-world
    ```

    

 2. 次のコマンドを実行して、**sample/hello-world** リポジトリのタグを一覧表示します。

    ※ XXXXXXXXにはLabUser-XXXXXXXXと同じ8桁の数字を入力します。

    ```
    az acr repository show-tags --name mycontainerregistryXXXXXXXX \
        --repository sample/hello-world --output table
    ```

    

    アウトプット：

    ```
    Result
    --------
    v1
    ```

    

 ## ACR でイメージを実行する

 

 1. **az acr run** コマンドを使用して、コンテナー レジストリから *sample/hello-world:v1* コンテナー イメージを実行します。次の例では、**$Registry** を使用して、コマンドを実行するレジストリを指定します。**myContainerRegistry** を前に使用した名前に置き換えます。

    ```
    az acr run --registry mycontainerregistryXXXXXXXX \
        --cmd '$Registry/sample/hello-world:v1' /dev/null
    ```

    

    この例の **cmd** パラメーターは、コンテナーを既定の構成で実行しますが、**cmd** は他の **docker run** パラメーターや他の **docker** コマンドもサポートします。

    次の出力例は短縮されています。

    ```
    Packing source code into tar to upload...
    Uploading archived source code from '/tmp/run_archive_ebf74da7fcb04683867b129e2ccad5e1.tar.gz'...
    Sending context (1.855 KiB) to registry: mycontainerre...
    Queued a run with ID: cab
    Waiting for an agent...
    2019/03/19 19:01:53 Using acb_vol_60e9a538-b466-475f-9565-80c5b93eaa15 as the home volume
    2019/03/19 19:01:53 Creating Docker network: acb_default_network, driver: 'bridge'
    2019/03/19 19:01:53 Successfully set up Docker network: acb_default_network
    2019/03/19 19:01:53 Setting up Docker configuration...
    2019/03/19 19:01:54 Successfully set up Docker configuration
    2019/03/19 19:01:54 Logging in to registry: mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Successfully logged into mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Executing step ID: acb_step_0. Working directory: '', Network: 'acb_default_network'
    2019/03/19 19:01:55 Launching container with name: acb_step_0
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    2019/03/19 19:01:56 Successfully executed container: acb_step_0
    2019/03/19 19:01:56 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.843801)
    
    Run ID: cab was successful after 6s
    ```

    

 ## リソースをクリーンアップする

 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

 1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。
 2. ツール バーで、**リソース グループの削除** を選択します。
 3. リソース グループ名を入力し、削除することを確認します。

  **注意：**リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
