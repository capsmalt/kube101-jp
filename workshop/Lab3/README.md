# Lab 3: 構成ファイルを使用した宣言的な方法によるスケーリングおよびアップデート

このハンズオンでは，Lab1，2で使用してきた guestbookアプリケーションと同じものをデプロイします。
これまでとの違いは，`kubectl run`コマンドなどで直接Podを作成・開始するのではなく，構成ファイルを使用してアプリケーションのデプロイを行うことです。
構成ファイルを使用することで，Kubernetesクラスターにおけるあらゆるリソースに対して，よりきめ細やかな管理ができるようになります。

構成ファイル(yaml)を引数にして `kubectl` コマンドを何度も実行するので，構成ファイルを使用したDeploymentやServiceの作成について体感してください。

ではハンズオンを開始します。
まずはアプリケーションをGitHubから取得してください。

```
$ git clone https://github.com/capsmalt/guestbook.git
```

このリポジトリは複数バージョンのguestbookアプリケーションを含んでいます。構成ファイルを使用してアプリケーションをデプロイできるように各種yamlを準備しています。

git cloneが完了したら，そのディレクトリに移動してください。 

`$ cd guestbook` 

`v1` ディレクトリ配下にこのハンズオンで使用する全ての構成ファイルが配置されていますので移動します。

`$ cd v1`

# 1. 構成ファイルを使用したアプリケーションのスケーリング

Kubernetesは，アプリケーションを実行用のPodを個々にデプロイできますが，多数のリクエストに応じてスケールさせる必要があります。
Deploymentは，Pod群に似た集合を管理します。レプリカ数を指定して要求すると，Kubernetes Deployment Controllerは常にそのレプリカ数を維持しようとします。

作成する全てのKubernetesオブジェクトは，2つのネストされたオブジェクトフィールドを持ちます。`spec`と`status`です。
`spec`オブジェクトは，desired stateを定義します。`status`オブジェクトは，リソースの現在の状態のようにKubernetesシステムから提供される情報を含みます。

前述の通り，Kubernetesは，現在の状態をdesired stateにしようと試みます。

オブジェクトを作成する際には，`apiVersion`や`kind`，`metadata`，`name`，`labels`を作成することになります。またオプションで，オブジェクトが属する`namespace`を指定する場合もあります。

次のguestbookアプリケーションのdeploymentの構成を見てみましょう。

**guestbook-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-v1
  labels:
    app: guestbook
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
        version: "1.0"
    spec:
      containers:
      - name: guestbook
        image: ibmcom/guestbook:v1
        ports:
        - name: http-server
          containerPort: 3000
```

上記の構成ファイルは，`guestbook-v1`という名前のdeploymentオブジェクトを作成します。同時に`ibmcom/guestbook:v1`イメージを動作させる1つのコンテナを含むPodを作成します。この構成ファイルによって，`replicas: 3` と指定されるため，Kubernetesは少なくとも3つのアクティブなPodが動作するように試みます。

- guestbook deploymentの作成

   この構成ファイルを使用してDeploymentを作成する場合，以下のコマンドを実行します。

   ``` console
   $ kubectl create -f guestbook-deployment.yaml
   deployment "guestbook-v1" created
   ```

- labelが app=guestbook であるPod一覧を表示

  生成済の全てのPodの中から，labelが "app" で，その値が "guestbook" であるPodを一覧表示することが可能です。
  labelは，構成ファイル(yaml)内の `spec.template.metadata.labels` という項目の値を指します。

   ```console 
   $ kubectl get pods -l app=guestbook
   ```

構成ファイルのレプリカ数を変更した場合，Kubernetesはリクエストに合わせて，Podの追加/削除を行います。構成の変更は以下のコマンドで行えます。
(今回は閲覧のみで，値の変更はしません)

   ```console
   $ kubectl edit deployment guestbook-v1
   ```

上記の操作で，KubernetesサーバーからDeploymentの最新の構成情報を検索し，編集できます。使用していた元のyamlファイル(`guestbook-deployment.yaml`)に比べて多数のフィールドが含まれていることに気づくでしょう。これは我々が直接指定した値だけでなく，Kubernetesが知っているDeploymentに関する全てのプロパティを含むためです。

Deploymentを作成するために使ったDeploymentファイルを編集して，変更を加えることができます。手元で編集した後に以下のコマンドで，変更を反映させられます。

   ```console
   $ kubectl apply -f guestbook-deployment.yaml
   ```

この操作によって，変更を加えた我々のyamlと，現在の状態の構成との "diff" を取り，Kubernetesが変更を適用します。

次に，外部のクライアント向けにdeploymentを公開する Serviceオブジェクト を定義します。

**guestbook-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook
  labels:
    app: guestbook
spec:
  ports:
  - port: 3000
    targetPort: http-server
  selector:
    app: guestbook
  type: NodePort
```

上記の構成はguestbookという名前のServiceリソースを作成します。Serviceは，アプリケーションに対するトラフィックのためのネットワークパスを作成する際に使われます。今回は，クラスター上の3000番ポートからのルートをアプリケーションの "http-server" ポートに指定します。

- Deploymentを作成した時と同じコマンドを使って，guestbook-serviceを作成しましょう

  ` $ kubectl create -f guestbook-service.yaml `

- ブラウザ上で以下のURLからgurstbookアプリの動作をテストします
  `<your-cluster-ip>:<node-port>`

  `nodeport` と `public-ip` を取得する方法を思い出してください:

  `$ kubectl get service guestbook` でポート番号を確認
  および
  `$ ibmcloud cs workers <name-of-cluster>` でPublic IPを確認

# 2. バックエンドサービスに接続

ディレクトリ配下のguestbookのソースコードを見ると，多様なデータストアをサポートしていることがわかります。
デフォルトでは，メモリ上でguestbookエントリのログを保持する構成になっています。
これはテスト目的であれば問題ない構成ですが，アプリケーションをスケールさせるようなリアルな環境では上手く機能しないことでしょう。

この問題を解決するために，アプリケーションの全てのインスタンス間で同じデータストアを共有する必要があります。今回は，RedisデータベースをK8sクラスターにデプロイして使用します。Redisインスタンスは，guestbookと似たような構成で定義します。

**redis-master-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
      role: master
  template:
    metadata:
      labels:
        app: redis
        role: master
    spec:
      containers:
      - name: redis-master
        image: redis:2.8.23
        ports:
        - name: redis-server
          containerPort: 6379
```

このyamlは，'redis-master' という名前のDeploymentでRedisデータベースを作成します。
シングルインスタンスとして作成するので，レプリカ数を1にセットします。guestbookアプリケーションはRedisに接続しデータを永続化します。
コンテナイメージは，'redis:2.8.23' を使用し，デフォルトのRedisポート6379で公開します。

- RedisのDeploymentを作成します:

    ```console
    $ kubectl create -f redis-master-deployment.yaml
    ```

- RedisサーバーのPod動作を確認します:

    ```console
    $ kubectl get pods -l app=redis,role=master
    NAME                 READY     STATUS    RESTARTS   AGE
    redis-master-q9zg7   1/1       Running   0          2d
    ```

- スタンドアローン動作するRedisをテストします:

    ` $ kubectl exec -it redis-master-q9zg7 redis-cli `

    "kubectl exec" コマンドは，指定されたコンテナ内で，2つ目のプロセスを開始します。
    今回は，"redis-master-q9zg7"というコンテナ内で，"redis-cli" コマンドを実行しました。

    コンテナ内に入れば，"redis-cli" コマンドを使って，Redisデータベースが正常に動作しているか確認したり，必要に応じて構成したりできます。

    ```console
    redis-cli> ping
    PONG
    redis-cli> exit
    ```

次に，guestbookアプリケーションが `redis-master` Deploymentに接続できるように，Serviceを公開しましょう。

**redis-master-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    app: redis
    role: master
spec:
  ports:
  - port: 6379
    targetPort: redis-server
  selector:
    app: redis
    role: master
```

この構成ファイルは，'redis-master' Serviceを作成し，ポート番号6379で，かつ， "app=redis" と "role=master" が指定されたPodをターゲットとするように構成します。


- Redis masterにアクセスするサービスを作成します:

    ``` $ kubectl create -f redis-master-service.yaml ```

- データベースを使用するRedis serviceを発見できるようにguestbookを再起動します:

    ```console
    $ kubectl delete deploy guestbook-v1 
    $ kubectl create -f guestbook-deployment.yaml
    ```

- ブラウザ上で以下のURLからgurstbookアプリの動作をテストします:
  `<your-cluster-ip>:<node-port>`

複数のブラウザを開いてページを更新すると，一貫した状態を保持したguestbookの異なるコピーを確認できます。
全てのインスタンスは同一のバッキング・パーシスタンスストレージに書き込み，全てのインスタンスはguestbookエントリを表示するために同じストレージから読み出します。

つまり，データの永続化ができるようになりました。
従って，複数のコンテナが動作するようなトラフィック増に応じて，スケールするシンプルな3層アプリケーションができたことになります。

しかし，一般的に言われる主なボトルネックは，各リクエストを処理するデータベース・サーバーを一つしか持っていないことです。一つのシンプルな解決策は，読み・書き用に異なるデータベースを用いて分離することで，データ一貫性を達成することです。

![rw_to_master](../images/Master.png)

`redis-slave`という名前のDeploymentを作成し，データの読み(read)を管理するredisデータベースと対話できるようにします。
データを読む(read)用のいくつかのインスタンスを動作させて，データベースをスケールさせます。

Redis slaveのdeploymentは2つのレプリカを動作するように構成されます:

![w_to_master-r_to_slave](../images/Master-Slave.png)

**redis-slave-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
      role: slave
  template:
    metadata:
      labels:
        app: redis
        role: slave
    spec:
      containers:
      - name: redis-slave
        image: kubernetes/redis-slave:v2
        ports:
        - name: redis-server
          containerPort: 6379
```

- redis slave deploymentを作成します
 ``` $ kubectl create -f redis-slave-deployment.yaml ```

 - 全てのslaveレプリカが動作しているか確認します
 ```console
$ kubectl get pods -l app=redis,role=slave
NAME                READY     STATUS    RESTARTS   AGE
redis-slave-kd7vx   1/1       Running   0          2d
redis-slave-wwcxw   1/1       Running   0          2d
 ```

- redis slaveのいずれかのPod内コンテナに入り，データベースを正しく閲覧できるか確認します

 ```console
$ kubectl exec -it redis-slave-kd7vx  redis-cli
127.0.0.1:6379> keys *
1) "guestbook"
127.0.0.1:6379> lrange guestbook 0 10
1) "hello world"
2) "welcome to the Kube workshop"
127.0.0.1:6379> exit
```

次に，Redis slave serviceを公開します。
一度デプロイされたら，"読み(read)"操作は `redis-slave` podに，"書き(write)"操作は `redis-master` podに送信されるように構成されます。

**redis-slave-service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
  labels:
    app: redis
    role: slave
spec:
  ports:
  - port: 6379
    targetPort: redis-server
  selector:
    app: redis
    role: slave
```

- Redis slaveに接続するためのServiceを作成します
    ``` $ kubectl create -f redis-slave-service.yaml ```

- slave serviceを発見できるようにguestbookアプリケーションを再始動します
    ```console
    $ kubectl delete deploy guestbook-v1
    $ kubectl create -f guestbook-deployment.yaml
    ```
    
- ブラウザ上で以下のURLからgurstbookアプリの動作をテストします:
  `<your-cluster-ip>:<node-port>`

以上でLab3のハンズオンは完了です。以下のコマンドを使用して，作成したKubernetesリソースを削除しましょう。

```console
$ kubectl delete -f guestbook-deployment.yaml
$ kubectl delete -f guestbook-service.yaml
$ kubectl delete -f redis-slave-service.yaml
$ kubectl delete -f redis-slave-deployment.yaml 
$ kubectl delete -f redis-master-service.yaml 
$ kubectl delete -f redis-master-deployment.yaml
```
