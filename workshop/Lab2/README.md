# Lab 2: Deployment のスケーリングとアップデート

Lab2では，Deploymentのインスタンス数を増減させるスケーリングと，安全にロールアウトさせながらアプリケーションをアップデートする方法を学びます。 

このLabを実施するには，`guestbook` アプリケーションのDeploymentが動作している必要があります。
前のハンズオン(Lab1)の最後に削除済の場合は，以下の手順で再度作成してください。

実行例:

```console
$ kubectl run guestbook --image=ibmcom/guestbook:v1
$ kubectl expose deployment guestbook --type="NodePort" --port=3000
```
    
# 1. レプリカ数(replica)の指定によるアプリケーションのスケーリング

レプリカ (*replica*) は，実行中のServiceを含むPodの複製です。複数のレプリカを用意することで，アプリケーションの負荷増大に対応できるリソースを保証できます。

1. `kubectl` は `scale` というサブコマンドを提供しています。既存のDeployment数を変更するために使用します。現在1インスタンスで動作する `guestbook` を 10インスタンスにキャパシティを増やしてみます。

    実行例:

   ``` console
   $ kubectl scale --replicas=10 deployment guestbook
   deployment "guestbook" scaled
   ```

   Kubernetesは，初期構成と同じ構成で 9つの新しいPodを作成することで，desired state (今回は10)を満たすように動作します。

2. ロールアウトされた変更内容を確認するために次のコマンドを実行します。 `kubectl rollout status deployment guestbook`.

    ※処理が早く完了した場合，以下のメッセージが表示されない場合があります

   ```console
   $ kubectl rollout status deployment guestbook
   Waiting for rollout to finish: 1 of 10 updated replicas are available...
   Waiting for rollout to finish: 2 of 10 updated replicas are available...
   Waiting for rollout to finish: 3 of 10 updated replicas are available...
   Waiting for rollout to finish: 4 of 10 updated replicas are available...
   Waiting for rollout to finish: 5 of 10 updated replicas are available...
   Waiting for rollout to finish: 6 of 10 updated replicas are available...
   Waiting for rollout to finish: 7 of 10 updated replicas are available...
   Waiting for rollout to finish: 8 of 10 updated replicas are available...
   Waiting for rollout to finish: 9 of 10 updated replicas are available...
   deployment "guestbook" successfully rolled out
   ```

3. ロールアウトが完了したら，Podの動作状況を次のコマンドで確認します。 `kubectl get pods`.

   以下のように，Deploymentのレプリカが10つ確認できるはずです。

   ```console
   $ kubectl get pods
   NAME                        READY     STATUS    RESTARTS   AGE
   guestbook-562211614-1tqm7   1/1       Running   0          1d
   guestbook-562211614-1zqn4   1/1       Running   0          2m
   guestbook-562211614-5htdz   1/1       Running   0          2m
   guestbook-562211614-6h04h   1/1       Running   0          2m
   guestbook-562211614-ds9hb   1/1       Running   0          2m
   guestbook-562211614-nb5qp   1/1       Running   0          2m
   guestbook-562211614-vtfp2   1/1       Running   0          2m
   guestbook-562211614-vz5qw   1/1       Running   0          2m
   guestbook-562211614-zksw3   1/1       Running   0          2m
   guestbook-562211614-zsp0j   1/1       Running   0          2m
   ```

**Tip:** 可用性を向上させるもう一つの方法
[add clusters and regions](https://console.bluemix.net/docs/containers/cs_planning.html#cs_planning_cluster_config)
クラスター自体，あるいはリージョンを追加することでも可用性を向上させられます。

![HA with more clusters and regions](../images/cluster_ha_roadmap.png)

# 2. アプリケーションのアップデートとロールバック

Kubernetesは，アプリケーションを新しいコンテナイメージにローリングアップデートする機能を提供します。これは動作中のコンテナイメージを容易にアップデート，または問題が起きた際には容易にロールバックできるようにします。

以前のハンズオン(Lab1)では `v1` タグが付与されたイメージを使用していました。アップグレードを実施する際には， `v2` タグが付与されたイメージを使用します。

アップデートおよびロールバック:
1. `kubectl` コマンドで， `v2` イメージを使用するように Deploymentをアップデートすることができます。`kubectl` コマンドと `set` サブコマンドを使用します。

    ```$ kubectl set image deployment/guestbook guestbook=ibmcom/guestbook:v2```

   ※1つのPodは，複数のコンテナで構成することが可能です。各々固有の名前を持っており，その名前を使用することによって個別にイメージを変更したり，一度に全てのコンテナイメージを変更させることも可能です。
   `guestbook` のDeploymentにおけるコンテナ名は，`guestbook` です。

    複数コンテナを同時にアップデート 
    ([More information](https://kubernetes.io/docs/user-guide/kubectl/kubectl_set_image/).)

2. `kubectl rollout status deployment/guestbook` を実行することで，ロールアウトのステータスを確認できます。
    
    ※処理が早く完了した場合，以下のメッセージが表示されない場合があります

   ```console
   $ kubectl rollout status deployment/guestbook
   Waiting for rollout to finish: 2 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 3 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 3 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 3 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 4 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 4 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 4 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 4 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 4 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 6 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 6 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 6 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 7 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for rollout to finish: 1 old replicas are pending termination...
   Waiting for rollout to finish: 1 old replicas are pending termination...
   Waiting for rollout to finish: 1 old replicas are pending termination...
   Waiting for rollout to finish: 9 of 10 updated replicas are available...
   Waiting for rollout to finish: 9 of 10 updated replicas are available...
   Waiting for rollout to finish: 9 of 10 updated replicas are available...
   deployment "guestbook" successfully rolled out
   ```

3. ブラウザ上で `<public-ip>:<nodeport>` を指定して，アプリケーション動作を確認します。

   "nodeport" および "public-ip" は以下の方法で取得できます。

   `$ kubectl describe service guestbook`
   および
   `$ ibmcloud cs workers <name-of-cluster>`

   guestbook アプリの "v2" が動作していることを確認してください。ページタイトルの文字列が `Guestbook - v2` に変更されているはずです。

4. 最新のロールアウトに戻したい場合は以下を実行します。

   ```console
   $ kubectl rollout undo deployment guestbook
   deployment "guestbook"
   ```

   実行状況を確認する場合は， `$ kubectl rollout status deployment/guestbook` で確認できます。
   
5. ロールアウト実行する際，*古い* レプリカと，*新しい* レプリカを確認できます。
   古いレプリカは，10のPodとしてデプロイされています。(Lab2で1から10に増加させました)
   新しいレプリカは，異なるイメージを使用して新たに生成されました。
   これらの全てのPodは，Deploymentが所有します。
   Deploymentは，これらの2セットのPod群を ReplicaSetと呼ばれるリソースを使用して管理しています。
   
   guestbookのReplicaSetは以下のようにして確認できます。
   
   ```console
   $ kubectl get replicasets -l run=guestbook
   NAME                   DESIRED   CURRENT   READY     AGE
   guestbook-5f5548d4f    10        10        10        21m
   guestbook-768cc55c78   0         0         0         3h
   ```

おめでとうございます。第2段階のバージョンのアプリケーションのデプロイが完了しました。
次のハンズオンはこちら [Lab3](../Lab3/README.md) です。

Lab3を開始する前に，Lab2で作成したK8sコンテンツを削除してください。以下のコマンドを実行します。

  1. Deploymentを削除する `$ kubectl delete deployment guestbook`.

  2. Serviceを削除する `$ kubectl delete service guestbook`.
  
