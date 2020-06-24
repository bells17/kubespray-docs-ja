# ノードの追加/入れ替え

Modified from [comments in #3471](https://github.com/kubernetes-sigs/kubespray/issues/3471#issuecomment-530036084)

## 制限: 1台目のkube-masterとETCD-MASTERの削除

現在のところ、kube-masterの1台目のノードおよびetcd-masterのリストのを削除することはできません。
それでもこのノードを削除したい場合は、削除しなければなりません。

### 1) 現在のマスターの順番を変更する

1台目のホストを他の位置に移動することで、マスターリストの順序を変更します。
例えば以下の例の `node-1` を削除したい場合は:

```yaml
  children:
    kube-master:
      hosts:
        node-1:
        node-2:
        node-3:
    kube-node:
      hosts:
        node-1:
        node-2:
        node-3:
    etcd:
      hosts:
        node-1:
        node-2:
        node-3:
```

inventoryを以下のように入れ替えます:

```yaml
  children:
    kube-master:
      hosts:
        node-2:
        node-3:
        node-1:
    kube-node:
      hosts:
        node-2:
        node-3:
        node-1:
    etcd:
      hosts:
        node-2:
        node-3:
        node-1:
```

## 2) クラスターのアップグレード

`cluster-upgrade.yml` または `cluster.yml` を実行すれば、削除を実行できるようになります。

## ワーカーノードの追加/入れ替え

これが一番簡単なはずです。

### 1) inventoryに新しいノードを追加

### 2) `scale.yml` を実行

クラスター内の他のノードに影響を与えないように制限するには、`--limit=NODE_NAME` を使用してください。

制限をかけずに `--limit` を使う前に、`facts.yml` playbookを実行してすべてのノードのfactのキャッシュを更新します。

### 3) 古いノードをremove-node.ymlで削除する

古いノードがinventoryに残っている状態で `remove-node.yml` を実行します。
削除するノードの実行を制限するために、playbookに `-e node=NODE_NAME` を渡す必要があります。
  
削除したいノードがオンラインになっていない場合は、次の用に`reset_nodes=false`をextra-varsに追加してください:
`-e node=NODE_NAME reset_nodes=false`
このフラグは、masterやetcdノードのような他のタイプのノードを削除する場合にも使用します。

### 5) inventoryからノードを削除します。

それだけです。

## マスターノードの追加/入れ替え

### 1) `cluster.yml`を実行する

新しいホストをinventoryに追加して `cluster.yml` を実行します。これには `scale.yml` は使えません。

### 3) kube-system/nginx-proxy を再起動する

すべてのホストでnginx-proxyポッドを再起動します。
このポッドはapiserverのローカルプロキシです。
Kubesprayは静的設定を更新しますが、リロードするには再起動が必要です。

```sh
# このコマンドを全てのホストで実行します
docker ps | grep k8s_nginx-proxy_nginx-proxy | awk '{print $1}' | xargs docker restart
```

### 4) 古いマスターノードの削除

古いノードがinventoryに残っている状態で `remove-node.yml` を実行します。
削除するノードの実行を制限するために、playbookに `-e node=NODE_NAME` を渡す必要があります。
削除したいノードがオンラインになっていない場合は、`reset_nodes=false` を追加してください。

## etcdノードの追加

クラスター内に常に奇数のetcdノードが存在することを確認する必要があります。
このような方法では、これは常に入れ替えまたはスケールアップ操作になります。
新しいノードを2つ追加するか、古いノードを削除します。

### 1) cluster.ymlを実行している新しいノードを追加します。

inventoryを更新してから、 `--limit=etcd,kube-master -e ignore_assert_errors=yes` を指定して `cluster.yml` を実行します。
etcdノードとして追加したいノードが既にクラスター内のワーカーやマスターノードである場合は、まず `remove-node.yml` を使ってそのノードを削除してください。

`--limit=etcd,kube-master -e ignore_assert_errors=yes` を指定して `upgrade-cluster.yml` を実行します。
これはクラスター内のすべてのetcdの設定を更新するために必要です。  

この時点で、ノードの数は偶数になります。
すべてがまだ機能しているはずです。問題が発生するのは、ノードを削除する前にクラスターが新しいetcdリーダーを選出する場合のみです。
それでも、実行中のアプリケーションは引き続き利用できるはずです。

1回の実行で複数のectdノードを追加する場合は、 `-e etcd_retries=10` を追加して各ectdノードの結合間の再試行の回数を増やすことができます。
そうしないと、etcdクラスターが最初の結合を処理し続け、後続のノードで失敗する可能性があります。
`-e etcd_retries=10` は3つの新しいノードを結合するために機能する可能性があります。

## etcdノードの削除

### 1) 古いetcdノードを削除する

With the node still in the inventory, run `remove-node.yml` passing `-e node=NODE_NAME` as the name of the node that should be removed.
If the node you want to remove is not online, you should add `reset_nodes=false` to your extra-vars.

ノードがinventoryに残っている状態で、 `-e node=NODE_NAME` で削除するノードの名前を指定して `remove-node.yml` を実行します。
削除するノードがオンラインでない場合は、 `reset_nodes=false` をextra-varsに追加する必要があります。

### 2) 残りのノードのみがinventoryにあることを確認してください

inventoryファイルから `NODE_NAME` を削除します。

### 3) 有効なetcdメンバーのリストでkubernetesとネットワーク構成ファイルを更新します

`cluster.yml` を実行して、残りのすべてのノードで構成ファイルを再生成します。

### 4) 古いインスタンスをシャットダウンします

それだけです。
