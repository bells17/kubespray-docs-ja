# Ansible変数

## Inventory

inventoryは3つのグループで構成されています:

* **kube-node** : ポッドを実行するkubernetesノードのリスト。
* **kube-master** : kubernetesのマスターコンポーネント(apiserver、scheduler、controller)を実行するサーバーのリスト。
* **etcd**: etcd サーバーを構成するサーバーのリスト。フェイルオーバーを目的とした場合、最低でも3台のサーバを用意する必要があります。

Note: do not modify the children of _k8s-cluster_, like putting
the _etcd_ group into the _k8s-cluster_, unless you are certain
to do that and you have it fully contained in the latter:

```ShellSession
k8s-cluster ⊂ etcd => kube-node ∩ etcd = etcd
```

kube-node_に_etcd_が含まれている場合は、Kubernetesのワークロードのスケジューリングが可能なようにetcdクラスタを定義します。
スタンドアロンにしたい場合は、これらのグループが交差しないようにしてください。
サーバをマスターとノードの両方として動作させたい場合は、_kube-master_と_kube-node_の両方のグループにサーバを定義する必要があります。
スタンドアロンでスケジュールされないマスターとして動作させたい場合は、_kube-master_のみで定義し、_kube-node_では定義しないようにしてください。

また、2つの特別なグループがあります:

* **calico-rr** : [Calicoのネットワーキング先進事例](/docs/calico.md)の解説
* **bastion** : ノードに直接アクセスできない場合には bastion ホストを設定します

以下は完全なinventoryの例です。:

```ini
## kubernetes サービスをデフォルトの iface とは異なる ip でバインドするために 'ip' 変数を設定します。
node1 ansible_host=95.54.0.12 ip=10.3.0.1
node2 ansible_host=95.54.0.13 ip=10.3.0.2
node3 ansible_host=95.54.0.14 ip=10.3.0.3
node4 ansible_host=95.54.0.15 ip=10.3.0.4
node5 ansible_host=95.54.0.16 ip=10.3.0.5
node6 ansible_host=95.54.0.17 ip=10.3.0.6

[kube-master]
node1
node2

[etcd]
node1
node2
node3

[kube-node]
node2
node3
node4
node5
node6

[k8s-cluster:children]
kube-node
kube-master
```

## グループ変数と変数の上書きの優先順位

メインのデプロイオプションを制御するグループ変数は ``inventory/sample/group_vars`` ディレクトリにあります。
オプション変数は `inventory/sample/group_vars/all.yml` にあります。
少なくとも一つのロール(またはノードグループ)に共通の必須となる変数は `inventory/sample/group_vars/k8s-cluster.yml` にあります。
dockerやkubernetesのプリインストール、マスターロール用のロール変数もあります。
[ansibleのドキュメント](https://docs.ansible.com/ansible/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)によると、これらはグループ変数からは上書きできません。
上書きするには、`-e` という実行時のフラグ(最も簡単な方法)や、ドキュメントで説明されている他のレイヤーを使うべきです。

Kubesprayは変数を上書きするために数層しか使いません（あるいはロールのために上書きされることを想定しています）。

Layer | Comment
------|--------
**role defaults** | Kubesprayの展開に最適なUXを提供しています
inventory vars | 未使用
**inventory group_vars** | ユーザーが ``all.yml``, ``k8s-cluster.yml`` などを使って上書きすることを想定しています
inventory host_vars | 未使用
playbook group_vars | 未使用
playbook host_vars | 未使用
**host facts** | Kubesprayが状態フラグのような内部ロールのロジックを上書きします
play vars | 未使用
play vars_prompt | 未使用
play vars_files | 未使用
registered vars | 未使用
set_facts | Kubesprayはいくつかの場所のためにset_factsを上書きします。
**role and include vars** | 変数の上書きするために悪いUXを提供します！extra varsを使用して強制します
block vars (only for tasks in block) | Kubesprayが内部ロールのロジックのために上書きします
task vars (only for the task) | ロールには使用されませんが、ヘルパースクリプトには使用されます
**extra vars** (always win precedence) | ``ansible-playbook -e @foo.yml``で上書きされます

## Ansibleのタグ

playbookでは以下のタグが定義されています。

|                 Tag name | Used for
|--------------------------|---------
|                     apps | k8sのアプリ定義
|                    azure | クラウドプロバイダー: Azure
|                  bastion | bastion用のsshの設定
|             bootstrap-os | ホストOSの設定に関するもの
|                   calico | ネットワークプラグイン: Calico
|                    canal | ネットワークプラグイン: Canal
|           cloud-provider | クラウドプロバイダー関連タスク
|                   docker | ホスト用のdocker設定
|                 download | デリゲートホストへのコンテナイメージのフェッチ
|                     etcd | etcdクラスタの設定
|         etcd-pre-upgrade | etcdクラスタのアップグレード
|             etcd-secrets | etcdの証明書/鍵の設定
|                 etchosts | ホストの/etc/hostsの設定
|                    facts | Gathering factsとmiscの結果確認
|                  flannel | ネットワークプラグイン: flannel
|                      gce | クラウドプロバイダー: GCP
|                hyperkube | K8sのhyperkubeイメージを使った操作
|          k8s-pre-upgrade | k8sクラスターのアップグレード
|              k8s-secrets | k8sの証明書/鍵の設定
|           kube-apiserver | 静的ポッドの設定: kube-apiserver
|  kube-controller-manager | 静的ポッドの設定: kube-controller-manager
|                  kubectl | kubectlとbash completionの設定
|                  kubelet | kubeletサービスの設定
|               kube-proxy | 静的ポッドの設定: kube-proxy
|           kube-scheduler | 静的ポッドの設定: kube-scheduler
|                localhost | localhost(ansible runner)のための特別なステップ
|                   master | k8のマスターノードの役割の設定
|               netchecker | netchecker k8sアプリのインストール
|                  network | k8sのためのネットワークプラグインの設定
|                    nginx | kube-apiserverインスタンスのためのLB設定
|                     node | k8のminion (compute)ノードロールの設定
|                openstack | クラウドプロバイダー: OpenStack
|               preinstall | 予備的な構成ステップ
|               resolvconf | hosts/appsへ/etc/resolv.confを設定
|                  upgrade | コンテナイメージ/バイナリなどのアップグレード
|                   upload | ホスト間でのイメージ/バイナリの配布
|                    weave | ネットワークプラグイン: Weave
|              ingress_alb | AWS ALB Ingress Controller

注意: ``bash scripts/gen_tags.sh``コマンドを使ってコードベースにあるすべてのタグのリストを生成してください。
新しいタグは空の"Used for"フィールドと共にリストアップされます。

## Example commands

DNS設定タスクのみをフィルタリングして適用し、ホストOSの設定やコンテナのイメージのダウンロードに関連する他のすべてをスキップするコマンド例:

```ShellSession
ansible-playbook -i inventory/sample/hosts.ini cluster.yml --tags preinstall,facts --skip-tags=download,bootstrap-os
```

そしてこれはhostsの/etc/resolv.confファイルからk8クラスタのDNSリゾルバのIPを削除するだけです:

```ShellSession
ansible-playbook -i inventory/sample/hosts.ini -e dns_mode='none' cluster.yml --tags resolvconf
```

And this prepares all container images locally (at the ansible runner node) without installing
or upgrading related stuff or trying to upload container to K8s cluster nodes:

そしてこれは関連するものをインストールしたりアップグレードしたり、k8sクラスタノードにコンテナをアップロードしようとすることなく、すべてのコンテナイメージをローカルに(ansible runnerノードで)準備します:

```ShellSession
ansible-playbook -i inventory/sample/hosts.ini cluster.yml \
    -e download_run_once=true -e download_localhost=true \
    --tags download --skip-tags upload,upgrade
```

注意: `--tags` と `--skip-tags` をうまく使い、何が起こるのか100%確信がある場合にのみ使用するようにしてください。

## 踏み台ホスト

ノードをパブリックにアクセスさせたくない場合（プライベートIPを持つノードのみ）、いわゆる *踏み台* ホストを使用してノードに接続できます。踏み台を指定して使用するには、inventoryに行を追加するだけです。以下のx.x.x.xを踏み台ホストのパブリックIPに置き換える必要があります。

```ShellSession
[bastion]
bastion ansible_host=x.x.x.x
```

Ansibleと踏み台インスタンスの詳細については
[SSH踏み台ホストを介したAnsibleの実行](https://blog.scottlowe.org/2015/12/24/running-ansible-through-ssh-bastion-host/)をご覧ください。

## Mitogen

[mitogen](/docs/mitogen.md)を使用して、kubesprayを高速化できます。
