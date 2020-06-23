# はじめに

## inventoryを構築する

Ansibleのインベントリは3つの形式で保存できます。YAML、JSON、INI的なものがあります。[ここ]((https://github.com/kubernetes-sigs/kubespray/blob/master/inventory/sample/inventory.ini))にインベントリの例があります。

Ansible inventoryを作成・修正するには、[inventory generator](https://github.com/kubernetes-sigs/kubespray/blob/master/contrib/inventory_builder/inventory.py) を使用します。
現在のところ、機能が制限されており、基本的なKubesprayクラスタインベントリの設定にしか使用されていませんが、大規模クラスタ用のインベントリファイルの作成もサポートしています。
これはサイズが特定のしきい値を超えた場合に、ETCDおよびKubernetesマスターのroleをノードのroleからの分離することができるというものです。
詳細については `python3 contrib/inventory_builder/inventory.py help` を実行してください。

inventory generatorの使用例:

```ShellSession
cp -r inventory/sample inventory/mycluster
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

次に `inventory/mycluster/hosts.yml` をinventoryファイルとして使用します。

## カスタムデプロイを開始する

inventoryができたら、デプロイのデータ変数をカスタマイズしてデプロイを開始します:

**重要**: my\_inventory/groups\_vars/\*.yamlを編集してデータ変数を上書きします:

```ShellSession
ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml -b -v \
  --private-key=~/.ssh/private_key
```

詳細は[ansibleガイド](docs/ansible.md)を参照してください。

### ノードの追加

既存のクラスタにワーカー、マスタ、etcd ノードを追加したいと思うかもしれません。これは `cluster.yml` playbookを再実行することで行うことができますし、ワーカーにkubeletをインストールしてマスターと対話するために必要な最低限のものをターゲットにすることもできます。これは、クラスタをオートスケーリングするようなことをするときに特に便利です。

- 新しいワーカーノードを適切なグループのインベントリに追加します(または[dynamic inventory](https://docs.ansible.com/ansible/intro_dynamic_inventory.html)を利用します)。
- `cluster.yml` の代わりに `scale.yml` をansible-playbookコマンドで実行します。

```ShellSession
ansible-playbook -i inventory/mycluster/hosts.yml scale.yml -b -v \
  --private-key=~/.ssh/private_key
```

### ノードの削除

既存のクラスタから**マスター**、**ワーカー**、または**etcd**ノードを削除したいと思うかもしれません。
これは `remove-node.yml` playbookを再実行することで行うことができます。
まず、指定されたすべてのノードがドレインされ、その後、一部のkubernetesのサービスと証明書を削除します。そして最後にkubectlコマンドを実行してこれらのノードを削除します。
これはノードの追加機能と組み合わせることができます。これは一般的にクラスタのオートスケーリングのようなことをするときに便利です。
もちろん、ノードが動作しない場合は、ノードを削除して再インストールすることができます。

削除したいノードを選択するには `--extra-vars "node=<nodename>,<nodename2>"` を使用します。

```ShellSession
ansible-playbook -i inventory/mycluster/hosts.yml remove-node.yml -b -v \
--private-key=~/.ssh/private_key \
--extra-vars "node=nodename,nodename2"
```

もしノードにsshで完全にアクセスできない場合は `--extra-vars reset_nodes=no` を追加して、ノードのリセットステップをスキップします。
1つのノードが利用できなくても、他のノードがSSHで接続できる場合は、インベントリのホスト変数として reset_nodes=no を設定することができます。

## Kubernetesに接続する

デフォルトでは、kubesprayは8080ポートを介してkube-apiserverに安全でないアクセスをしてkube-masterホストを設定しています。
この場合、kubectl は <http://localhost:8080> を使って接続するので、kubeconfig ファイルは必要ありません。
生成されたkubeconfigファイルは(kub-master上の)localhostを指し、kube-nodeホストはlocalhostのnginxプロキシか、設定されていればロードバランサに接続します。
この処理の詳細は [HA ガイド](docs/ha-mode.md) を参照してください。

Kubesprayはデフォルトでは6443ポートに対する任意のkube-masterホストの任意のIPでのクラスタへのリモート接続を許可しています。しかし、これには認証が必要です。
kube-masterホストからkubeconfigを取得するか(下記を参照してください](#accessing-kubernetes-api))、または[ユーザ名とパスワード]を指定して接続することができます(vars.md#user-accounts)。

kubeconfigやKubernetesクラスタへのアクセスについては、Kubernetes[ドキュメント](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)を参照してください。

## Kubernetes Dashboardへのアクセス

サポートされるバージョンはkubernetes-dashboard v2.0.x です。

- ログインオプションは以下の通りです: token/kubeconfig by default, basic can be enabled with `kube_basic_auth: true` inventory variable - not recommended because this requires ABAC api-server which is not tested by kubespray team
- デフォルトでは "kube-system" 名前空間にデプロイされていますが、インベントリの `dashboard_namespace: kubernetes-dashboard` でオーバーライドすることができます。
- httpsでのみ提供しています。

アクセス方法は[ダッシュボードのドキュメント](https://github.com/kubernetes/dashboard/tree/master/docs/user/accessing-dashboard)に記載されています。
KubesprayのデフォルトのDeploymentはkuberntes-dashboardではなく、kube-system名前空間にあります。

- プロキシURLは <http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#/login> です。
- kubectlコマンドには必ず"-n kube-system"オプションが必要です。

Ingress経由でのアクセスを強くお勧めします。プロキシでアクセスするには、プロキシが [localhost](https://github.com/kubernetes/dashboard/issues/692#issuecomment-220492484) をlistenしなければならないことに注意してください (`proxy --address="x.x.x.x.x"` は動作しません)。

トークン認証については、[ダッシュボードサンプルユーザ](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)のドキュメントにサービスアカウントの作成方法が記載されています。なお、デフォルトの名前空間には注意してください。

アクセスはまた、マスター上のsshトンネルで行うこともできます。

```bash
# localhost:8081 は master-1 の localhost:8081 に送信されます。
ssh -L8001:localhost:8001 user@master-1
sudo -i
kubectl proxy
```

## Kubernetes APIへのアクセス

Kubernetesのメインクライアントは `kubectl` です。
これは各kube-masterホストにインストールされており、オプションでansibleホストの設定の `kubectl_localhost: true` と `kubeconfig_localhost: true` で設定することができます。

- `kubectl_localhost` を有効にすると、`kubectl` が `/usr/local/bin/` にダウンロードされbash completionがセットアップされます。ヘルパースクリプト`inventory/mycluster/artifacts/kubectl.sh`もまた以下の`admin.conf`をセットアップするために作られます。
- `kubeconfig_localhost` を有効にすると、デプロイ後に `admin.conf` が `inventory/mycluster/artifacts/` ディレクトリに表示されます。
- これらのファイルをダウンロードする場所は `artifacts_dir` 変数で設定できます。

以下のコマンドを実行することで、ノードの一覧を見ることができます:

```ShellSession
cd inventory/mycluster/artifacts
./kubectl.sh get nodes
```

必要に応じて admin.conf を ~/.kube/config にコピーしてください。
