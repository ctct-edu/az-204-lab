# Azure CLI を使用してコンテナーを Azure Container Apps にデプロイする



この演習では、Azure CLI を使用して、コンテナー化されたアプリケーションを Azure Container Apps にデプロイします。コンテナー アプリ環境を作成し、コンテナーをデプロイし、アプリケーションが Azure で実行されていることを確認する方法について説明します。

この演習で実行されるタスク:

- Azure でリソースを作成する
- Azure Container Apps 環境を作成する
- コンテナー アプリを環境にデプロイする

この演習は完了するまでに約 **15** 分かかります。



## リソース グループを作成し、Azure 環境を準備する



1. ブラウザーで Azure portal [https://portal.azure.com](https://portal.azure.com/) に移動します。プロンプトが表示されたら、Azure 資格情報を使用してサインインします。

2. ページ上部の検索バーの右側にある **[>_]** ボタンを使用して、Azure portal で新しいクラウド シェルを作成し、***Bash*** 環境を選択します。クラウド シェルは、Azure portal の下部にあるウィンドウにコマンド ライン インターフェイスを提供します。

   > **注**: *PowerShell* 環境を使用するクラウド シェルを以前に作成した場合は、***Bash*** に切り替えます。

3. この演習に必要なリソースのリソース グループを作成します。**myResourceGroup** をリソース グループに使用する名前に置き換えます。必要に応じて、**eastus** を近くのリージョンに置き換えることができます。使用するリソース・グループが既にある場合は、次のステップに進みます。

   ```azurecli
   az group create --location eastus --name myResourceGroup
   ```

   

4. 次のコマンドを実行して、CLI 用の最新バージョンの Azure Container Apps 拡張機能がインストールされていることを確認します。

   ```azurecli
   az extension add --name containerapp --upgrade
   ```

   

### 名前空間の登録



Azure Container Apps に登録する必要がある名前空間は 2 つあり、次の手順で登録されていることを確認します。各登録がサブスクリプションでまだ構成されていない場合は、完了するまでに数分かかる場合があります。

1. **Microsoft.App** 名前空間を登録します。

   ```
   az provider register --namespace Microsoft.App
   ```

   

2. Azure Monitor Log Analytics ワークスペースの **Microsoft.OperationalInsights** プロバイダーを以前に使用したことがない場合は、登録します。

   ```
   az provider register --namespace Microsoft.OperationalInsights
   ```

   

## Azure Container Apps 環境を作成する



Azure Container Apps の環境では、コンテナー アプリのグループを囲むセキュリティで保護された境界が作成されます。同じ環境にデプロイされた Container Apps は、同じ仮想ネットワークにデプロイされ、同じ Log Analytics ワークスペースにログが書き込まれます。

1. **az containerapp env create** コマンドを使用して環境を作成します。**myResourceGroup** と **myLocation** を前に使用した値に置き換えます。操作が完了するまでに数分かかります。

   ```
   az containerapp env create \
       --name my-container-env \
       --resource-group myResourceGroup \
       --location myLocation
   ```

   

## コンテナー アプリを環境にデプロイする



コンテナー アプリ環境のデプロイが完了したら、コンテナー イメージを環境にデプロイできます。

1. **containerapp create** コマンドを使用して、サンプルのアプリコンテナーイメージをデプロイします。**myResourceGroup** を前に使用した値に置き換えます。

   ```
   az containerapp create \
       --name my-container-app \
       --resource-group myResourceGroup \
       --environment my-container-env \
       --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
       --target-port 80 \
       --ingress 'external' \
       --query properties.configuration.ingress.fqdn
   ```

   

   **--ingress** を**外部**に設定すると、コンテナー アプリをパブリック要求で使用できるようになります。このコマンドは、アプリにアクセスするためのリンクを返します。

   ```
   Container app created. Access your app at <url>
   ```

   

デプロイを確認するには、**az containerapp create** コマンドによって返される URL を選択して、コンテナー アプリが実行されていることを確認します。



 ## リソースをクリーンアップする 

 演習が終了したので、不要なリソースの使用を避けるために、作成したクラウド リソースを削除する必要があります。

  1. 作成したリソース・グループに移動し、この演習で使用したリソースの内容を表示します。
  2. ツール バーで、**リソース グループの削除** を選択します。
  3. リソース グループ名を入力し、削除することを確認します。

  **注意：**リソース グループを削除すると、そのグループに含まれるすべてのリソースが削除されます。この演習で既存のリソース グループを選択した場合、この演習の範囲外の既存のリソースも削除されます。
