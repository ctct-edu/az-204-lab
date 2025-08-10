# Azure CLI コマンドを使用してコンテナーを Azure Container Instances にデプロイする



この演習では、Azure CLI を使用して Azure Container Instances (ACI) にコンテナーをデプロイして実行します。コンテナーグループを作成し、コンテナー設定を指定し、コンテナー化されたアプリケーションがクラウドで実行されていることを確認する方法を学習します。

この演習で実行されるタスク:

- Azure で Azure Container Instance リソースを作成する
- コンテナーの作成とデプロイ
- コンテナーが実行されていることを確認する

この演習は完了するまでに約 **15** 分かかります。

## リソース グループを作成する



1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

   > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成した場合は、***Bash*** に切り替えます。

3. この演習に必要なリソースのリソース グループを作成します。**myResourceGroup** をリソース グループに使用する名前に置き換えます。必要に応じて、**eastus** を近くのリージョンに置き換えることができます。使用するリソース・グループが既にある場合は、次のステップに進みます。

   ```
   az group create --location eastus --name myResourceGroup
   ```

   

## コンテナーの作成とデプロイ



コンテナーを作成するには、名前、Docker イメージ、Azure リソース グループを **az container create** コマンドに指定します。コンテナーをインターネットに公開するには、DNS 名ラベルを指定します。

1. 次のコマンドを実行して、コンテナーをインターネットに公開するために使用される DNS 名を作成します。DNS 名は一意である必要がある場合は、Cloud Shell からこのコマンドを実行して、一意の名前を保持する変数を作成します。

   ```
   DNS_NAME_LABEL=aci-example-$RANDOM
   ```

   

2. 次のコマンドを実行して、コンテナインスタンスを作成します。**myResourceGroup** と **myLocation** を前に使用した値に置き換えます。操作が完了するまでに数分かかります。

   ```
   az container create --resource-group myResourceGroup \
       --name mycontainer \
       --image mcr.microsoft.com/azuredocs/aci-helloworld \
       --ports 80 \
       --dns-name-label $DNS_NAME_LABEL --location myLocation \
       --os-type Linux \
       --cpu 1 \
       --memory 1.5 
   ```

   

   前のコマンドでは、**$DNS_NAME_LABEL** が DNS 名を指定します。イメージ名 **mcr.microsoft.com/azuredocs/aci-helloworld** は、基本的な Node.js Web アプリケーションを実行する Docker イメージを参照します。

**az container create** コマンドが終了したら、次のセクションに進みます。

## コンテナーが実行されていることを確認する



コンテナーのビルド状態は、**az container show** コマンドで確認できます。

1. 次のコマンドを実行して、作成したコンテナーのプロビジョニング状態を確認します。**myResourceGroup** を前に使用した値に置き換えます。

   ```
   az container show --resource-group myResourceGroup \
       --name mycontainer \
       --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
       --out table 
   ```

   

   コンテナーの完全修飾ドメイン名 (FQDN) とそのプロビジョニング状態が表示されます。次に例を示します。

   ```
   FQDN                                    ProvisioningState
   --------------------------------------  -------------------
   aci-wt.eastus.azurecontainer.io         Succeeded
   ```

   

   > **手記：** コンテナーが [**作成中**] 状態の場合は、しばらく待ってから **[成功]** 状態が表示されるまでコマンドを再度実行します。

2. ブラウザーからコンテナーの FQDN に移動して、実行中であることを確認します。サイトが安全ではないという警告が表示される場合があります。

   

    ## リソースをクリーンアップする 

    演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

     1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。
     2. ツール バーで、**リソース グループの削除** を選択します。
     3. リソース グループ名を入力し、削除することを確認します。

     **注意：**リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
