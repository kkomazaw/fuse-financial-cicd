# Fuse Integration Services 2.0を使った金融系アジャイル・インテグレーションデモ

シンプルなマイクロサービスを用いて、銀行業務の残高照会と送金のリクエストをバックエンドのシステムへ転送するデモです。
バックエンドには、MySQLデータベースに接続する従来型の銀行システムのマイクロサービスアプリケーションと、メッセージングブローカーを経由してモックアップのブロックチェーンに接続するビットコイン用のマイクロサービスアプリケーションがあります。

![alt text](images/outline.png "outline")

このデモのポイントは以下の通りです。
1. Source to Image (S2i) によるビルドとデプロイのプロセス
2. パイプラインによる自動化されたCI/CD
3. CamelによるREST APIの実装とSwaggerによるAPIドキュメントの生成
4. 3scale API managementによるAPIの管理
5. HystrixによるAPIの実装

まずは、アプリケーションをセットアップしましょう。

## DEVおよびUAT環境の設定
***Install OpenShift Container Platform 3.5 in [CDK 3.0](https://developers.redhat.com/products/cdk/overview/)***

このリポジトリをforkして、ローカルにダウンロードしてください。

```
git https://github.com/<あなたのリポジトリ>/fuse-financial-cicd.git
```

OpenShiftを起動してください。

```
minishift start --skip-registration
```

developerでログインし、このデモで必要なメッセージング用のテンプレートをインストールします。


```
oc login -u developer

# DEV/UATデプロイ用のプロジェクトを作成する
oc new-project fisdemo --display-name="Fuse Banking Demo - Dev and UAT" --description="Development and UAT environment for Agile Integration Banking Demo - Power by Red Hat Fuse"

# AMQイメージをインポートする
oc import-image amq62-openshift --from=registry.access.redhat.com/jboss-amq-6/amq62-openshift --confirm

cd support
# AMQのテンプレートを作成する
oc create -f projecttemplates/amq62-openshift.json

```

## MySqlデータベース、AMQブローカー、Jenkinsをセットアップする 

以下のコマンドラインを実行して、これらのセットアップを実行します。（OpenShiftコンソールから作成しても構いません）


```
oc create -f https://raw.githubusercontent.com/openshift/origin/master/examples/db-templates/mysql-ephemeral-template.json

oc new-app --template=mysql-ephemeral --param=MYSQL_PASSWORD=password --param=MYSQL_USER=dbuser --param=MYSQL_DATABASE=sampledb

oc new-app --template=amq62-basic --param=MQ_USERNAME=admin --param=MQ_PASSWORD=admin --param=IMAGE_STREAM_NAMESPACE=fisdemo

```

## アプリケーションをOpenShiftにプッシュする
これらの2つのマイクロサービスをアップロードするために、バイナリS2i(Source to Image)を使います。
 - 従来型銀行システム
 - ビットコインゲートウェイ

 まずは従来型銀行システムのプロジェクトフォルダーに移動して以下を実行します。

```
cd ..
cd fisdemoaccount
mvn fabric8:deploy -Dmysql-service-username=dbuser -Dmysql-service-password=password -DskipTests=true
```


次にビットコインゲートウェイのプロジェクトフォルダーに移動して以下を実行します。

```
cd ..
cd fisdemoblockchain
mvn fabric8:deploy
```

次にAPIゲートウェイサービスをデプロイします。今回はパイプラインを構築して、ステージングからUAT環境へCI/CDプロセスを自動化してみます。

```
cd ..
oc process -f support/projecttemplates/template-uat.yml | oc create -f -
oc start-build fisgateway-service
```

これで出来上がりです。それでは早速デモを実行してみましょう。ブラウザから以下のRESTサービスを呼び出してみてください。


```
http://fisgateway-service-fisdemo.<OPENSHIFT_HOST>/demos/sourcegateway/balance/234567?moneysource=bitcoin
http://fisgateway-service-fisdemo.<OPENSHIFT_HOST>/demos/sourcegateway/balance/234567
```

[![Installing UAT and DEV project video](images/video01-0.png)](https://vimeo.com/219952887 "Fuse Banking Agile Integration Demo - Installing UAT and DEV project")


## インターネットバンキング画面を実行する

もう少し派手にしたい場合は、GUIアプリケーションをインストールしてみましょう。

![alt text](images/bankinggui.png "Banking GUI")

```
cd fisdemogui
oc new-project fisdemogui --display-name="Fuse Banking Demo - GUI" --description="Web GUI for Banking demo, does transfer and balance enquiry"
oc new-build --image-stream=nodejs --binary=true --name=fisdemogui
oc start-build fisdemogui --from-dir=. --follow
oc new-app fisdemogui
oc expose svc fisdemogui
```

ブラウザで http://fisdemogui-fisdemogui.<OPENSHIFT_HOST>.nip.io を開きます。
画面上のAPI Hostに *fisgateway-service-fisdemo.<OPENSHIFT_HOST>* を入力して実行してください。

[![Installing Banking GUI](images/video02-0.png)](https://vimeo.com/219955921 "Fuse Banking Agile Integration Demo - Installing Banking GUI")

## 本番環境をセットアップする
それでは次に本番用のプロジェクトを作成します。
Add setup the environment including supporting microservices and configurations (deployment configs/service/route) in production

```
cd support
./setupprod.sh
```


## Hystrixを使ってAPIレジリエンスを高める

提供されているkubeflix.jsonテンプレートを使って、HystrixダッシュボードとTurbineサーバーを起動します。

```
oc process -f kubeflix.yml | oc create -f -
```

[![Installing PROD project video](images/video03-0.png)](https://vimeo.com/219957939 "Fuse Banking Agile Integration Demo - Installing PROD project")


## 3scale API Managementをセットアップする
3scaleのセットアップの２通りの方法があります。

1. **オプション1:** (推奨) 45日間のトライアルバージョン（オンライン版）に登録する
```
https://www.3scale.net/signup/
```
登録後、管理用ドメインを受け取ることができます。

   **オプション2:** ローカルに3scale環境をセットアップする

   **注意!!! CDKのために少なくとも16GBのメモリが必要です**

   A.  プロジェクトを作成する

	```
	oc new-project threescaleonprem
	```

   B.  永続化ボリュームをセットアップする（CDK V3/Minishift V1を使用している場合、この操作はオプションです)
	```
	oc new-app -f support/amptemplates/pv.yml
	```

   C.  3scaleを以下のコマンドでインストールする。WILDCARD_DOMAIN パラメータはあなたのOpenShiftにおけるドメインとなります。

	```
	oc new-app -f support/amptemplates/amp.yml --param WILDCARD_DOMAIN=<WILDCARD_DOMAIN>
	```

   詳細は公式のドキュメントをご参照ください。

2. アクセストークンを取得する

	**オプション1:**

	A. 管理者コンソールの上部右端にある、 *Personal Settings* を選択し、タブにある、Tokens をクリックします。さらに、 **Add Access Token** をクリックしてください。

	B. 以下の情報を設定してトークンを作成します。

	- **Name**: demomgmttoken
	- **Scopes**: Account Management API
	- **Permission**: Read & Write

	生成されたアクセストークンを安全な場所に記録します。


	**オプション2:**

	正常に3scaleをOpenShiftにインストールした際、コンソール上にてアクセストークンが出力されます。


3. 先ほど取得したアクセストークンを使用して以下のスクリプトを実行することで、3scaleの設定を行います。

	```
	cd threescalesetup
	mvn clean package
	mvn exec:java -Dexec.mainClass=threescalesetup.SetupApp -Dexec.args="<3SCALE_HOST_DOMAIN> <ACCESS_TOKEN> financeapidemo financeapidemo true productiondemo 'Finance API Demo for Agile Integration'"
	cd ..
	```
	![alt text](images/threescaleapiconfig.png "3scale configuration")

4. APIサービスにアクセスするアカウントをセットアップします。

	```
	cd threescalesetup
	mvn exec:java -Dexec.mainClass=threescalesetup.SetupAccount -Dexec.args=<3SCALE_HOST_DOMAIN> <ACCESS_TOKEN> <APPLICATION_PLAN_ID> financedemoapp 'The Finance Demo Application'
	cd ..
	```


5. あなたのアクセストークンとドメイン名を指定して、APICastをUATおよびPRODプロジェクトにインストールします。

	```
	oc project fisdemoprod
	oc secret new-basicauth apicast-configuration-url-secret --password=https://<ACCESS_TOKEN>@<DOMAIN>-admin.3scale.net

	oc new-app -f support/amptemplates/apicast.yml
	```

	![alt text](images/threescaleinstall.png "3scale install")

6. 3scaleのIntegration設定画面でエンドポイントのアドレスを更新します。

	これらの設定は手動で行う必要があります。3scaleの管理画面にログインし、 **API** タブを選択します。 *Fuse Financial Agile Integration Demo Service* をクリックし、左側のタブより **Integration** を選択します。 **edit Apicast Configuration** を削除します。

![alt text](images/threescaleapicastconfigmenu.png "3scale APICast Config")

Here is where we tell Apicast where to look for our APIs and how the APIs can be accessed.

- Private Base URL: **http://fisgateway-service-stable:8080**
- Public Basic URL: **http://apicast-fisdemoprod.<OPENSHIFT_HOST>**
- メトリクス設定のセクションで以下の3つを追加します。
	- GET /demos/sourcegateway/balance
	- GET /demos/sourcegateway/profile
	- POST /demos/sourcegateway/transfer

![alt text](images/threescaleapicastconfig.png "3scale APICast Config")

[![Setup 3scale API management video](images/video04-0.png)](https://vimeo.com/220360925 "Fuse Banking Agile Integration Demo - Setup 3scale API management")

## CI/CD をセットアップする

### 重要!!! 以下のCI/CD A/Bテストパイプラインを実行するためには、3scaleのアカウント設定が正常に設定されている必要があります。

![alt text](images/cicd.png "CI/CD pipelines")


パイプライン用のプロジェクトを作成します。

```
oc new-project fisdemocicd --display-name="Fuse Banking Pipeline" --description="All CI/CD Pipeline for Banking Demo"
```

cicdプロジェクトユーザーにUATとPROD環境を操作できるようアクセス権限を付与します。

```
oc policy add-role-to-group edit system:serviceaccounts:fisdemocicd -n fisdemo
oc policy add-role-to-group edit system:serviceaccounts:fisdemocicd -n fisdemoprod
```

3つのパイプラインをインストールします。

```
oc create -f support/pipelinetemplates/pipeline-uat.yml
oc create -f support/pipelinetemplates/pipeline-ab.yml
oc create -f support/pipelinetemplates/pipeline-allprod.yml

oc new-app pipeline-uat

oc new-app pipeline-ab \
--param=THREESCALE_URL=https://<3SCALE_HOST_DOMAIN>-admin.3scale.net \
--param=API_TOKEN=<ACCESS_TOKEN> \
--param=APP_PLAN_ID=<APPLICATION_PLAN_ID> \
--param=METRICS_ID=<METRICS_ID> \
--param=API_LIMITS=25 \
--param=OPENSHIFT_HOST=<OPENSHIFT_HOST>

oc new-app pipeline-allprod \
--param=THREESCALE_URL=https://<3SCALE_HOST_DOMAIN>-admin.3scale.net \
--param=API_TOKEN=<ACCESS_TOKEN> \
--param=APP_PLAN_ID=<APPLICATION_PLAN_ID> \
--param=METRICS_ID=<METRICS_ID> \
--param=API_LIMITS=50 \
--param=OPENSHIFT_HOST=<OPENSHIFT_HOST>
```

このデモパイプラインプロジェクトは、Fuseによる統合アプリケーションを構築するための3つのパイプラインが含まれています。

A. 事前構築済みUATパイプラインは、イメージをSCM (github)から構築し、テストインスタンスをデプロイします。そしてUAT移行前のテストをQAにより実施し、検証後、UAT環境に移行可能かどうかを選択します。（イメージにuatreadyタグをつけることによって） UAT環境に昇格すると、パイプラインはタグづけされたイメージをUATイメージとしてデプロイし、アクセス可能なアドレス（routeと呼びます）を付与します。

B. A/Bテストパイプラインは、UATプロジェクトにあるUATイメージを本番プロジェクトに移動させます。そしてトラフィックの30%を新しいバージョンに、残りの70%を既存の安定バージョンに振り分けます。さらにAPI管理層（3scale)のトラフィックを毎分25コールに更新します。

C. フルリリースの準備ができました。本番パイプラインは、ローリングアップデートを実施して、古いサービスを新しいサービスに置き換えていきます。新しいサービスは安定バージョンとなります。さらにAPI管理層（3scale)のトラフィックを毎分50コールに更新します。

![alt text](images/allpipelines.png "allpipelines")

[![Setup CI/CD pipelines video](images/video05-0.png)](https://vimeo.com/220360925 "Fuse Banking Agile Integration Demo - Setup CI/CD pipelines")

以上でデモは完了です。お疲れ様でした。
