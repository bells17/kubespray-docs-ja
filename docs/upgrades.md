# KubesprayでのKubernetesのアップグレード

Kubesprayは、初期展開と同じ方法でアップグレードを処理します。
つまり、各コンポーネントは決まった順序で配置されます。

コンポーネントのバージョンを明示的に定義することで、コンポーネントのバージョンを個別に制御することもできます。
各コンポーネントのすべてのバージョン変数は次のとおりです:

* docker_version
* kube_version
* etcd_version
* calico_version
* calico_cni_version
* weave_version
* flannel_version
* kubedns_version

:warning: [古いリリースから最新のリリースに直接アップグレードすることはサポートされておらず、何かが壊れる可能性があります](https://github.com/kubernetes-sigs/kubespray/issues/3849#issuecomment-451386515) :warning:

古いリリースから最新のリリースにアップグレードする方法については、[複数のアップグレード](#複数のアップグレード)を参照してください

## 安全でないアップグレードの例

kube_versionのみをv1.4.3からv1.4.6にアップグレードする場合は、次の方法でデプロイできます:

```ShellSession
ansible-playbook cluster.yml -i inventory/sample/hosts.ini -e kube_version=v1.4.3 -e upgrade_cluster_setup=true
```

そしてkube_versionをv1.4.6にして再度実行します:

```ShellSession
ansible-playbook cluster.yml -i inventory/sample/hosts.ini -e kube_version=v1.4.6 -e upgrade_cluster_setup=true
```

クラスター内の kube-apiserver のようなデプロイをすぐに移行するためには ````-e upgrade_cluster_setup=true```` という変数を設定する必要がありますが、これは通常gracefulアップグレードでのみ行われます。([#4139](https://github.com/kubernetes-sigs/kubespray/issues/4139)と[#4736](https://github.com/kubernetes-sigs/kubespray/issues/4736)を参照してください)

## Gracefulアップグレード

Kubesprayはクラスターのアップグレードを実行する際にノードの紐付け、ドレイン、紐付け解除もサポートしています。
このために使用する別のplaybookがあります。
注意しなければならないのは、upgrade-cluster.yml は既存のクラスターのアップグレードにしか使えないということです。
つまり、少なくとも1つのkube-masterがすでにデプロイされている必要があります。

```ShellSession
ansible-playbook upgrade-cluster.yml -b -i inventory/sample/hosts.ini -e kube_version=v1.6.0
```

アップグレードが成功した後、サーバーのバージョンを更新する必要があります:

```ShellSession
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.0", GitCommit:"fff5156092b56e6bd60fff75aad4dc9de6b6ef37", GitTreeState:"clean", BuildDate:"2017-03-28T19:15:41Z", GoVersion:"go1.8", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"6", GitVersion:"v1.6.0+coreos.0", GitCommit:"8031716957d697332f9234ddf85febb07ac6c3e3", GitTreeState:"clean", BuildDate:"2017-03-29T04:33:09Z", GoVersion:"go1.7.5", Compiler:"gc", Platform:"linux/amd64"}
```

## 複数のアップグレード

:warning: [Do not skip releases when upgrading--upgrade by one tag at a time.](https://github.com/kubernetes-sigs/kubespray/issues/3849#issuecomment-451386515) :warning:

例えば、v2.6.0ならv2.7.0をチェックアウトして、アップグレードを実行して、次のタグをチェックアウトして、次のアップグレードを実行するなど。

k8s-cluster.ymlで明示的にkubernetesのバージョンを定義していないと仮定して、次のタグをチェックアウトして、upgrade-cluster.yml playbookを実行するだけです。

* inventoryにkubernetesのバージョンを定義している場合(例: group_vars/k8s-cluster.yml)、クラスターのアップグレードを実行する前に更新するか、このようにアップグレードするバージョンを指定してください:
`ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml -e kube_version=v1.11.3`

  行わなければ、アップグレードを行ってもクラスターのk8sバージョンはinventory変数で定義されたバージョンのままです。

以下の例では、v2.6.0 で設定したクラスターを v2.10.0 にアップグレードしています

```ShellSession
$ kubectl get node
NAME      STATUS    ROLES         AGE       VERSION
apollo    Ready     master,node   1h        v1.10.4
boomer    Ready     master,node   42m       v1.10.4
caprica   Ready     master,node   42m       v1.10.4

$ git describe --tags
v2.6.0

$ git tag
...
v2.6.0
v2.7.0
v2.8.0
v2.8.1
v2.8.2
...

$ git checkout v2.7.0
Previous HEAD position was 8b3ce6e4 bump upgrade tests to v2.5.0 commit (#3087)
HEAD is now at 05dabb7e Fix Bionic networking restart error #3430 (#3431)

# 注意: アップグレード時には、sudo pip3 install -r requirements.txt を実行する必要があるかもしれません

ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml

...

$ kubectl get node
NAME      STATUS    ROLES         AGE       VERSION
apollo    Ready     master,node   1h        v1.11.3
boomer    Ready     master,node   1h        v1.11.3
caprica   Ready     master,node   1h        v1.11.3

$ git checkout v2.8.0
Previous HEAD position was 05dabb7e Fix Bionic networking restart error #3430 (#3431)
HEAD is now at 9051aa52 Fix ubuntu-contiv test failed (#3808)
```

:information_source: 注意：バージョンをアップグレードする際には、サンプルinventoryとinventoryの間の変更を確認してください :information_source:

サンプルinventoryから始めた場合、2.7.0から2.8.0に直接アップグレードすることができないことを意味するバージョン間のいくつかの非推奨項目について。

この場合、"kubeadm_enabled"は2.9.0で非推奨となり削除されることを知っていて、クラスターのkubeadmへのコンバートを後回しにするために、"kubeadm_enabled "をfalseに設定しました。

```ShellSession
$ ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml
...
    "msg": "DEPRECATION: non-kubeadm deployment is deprecated from v2.9. Will be removed in next release."
...
Are you sure you want to deploy cluster using the deprecated non-kubeadm mode. (output is hidden):
yes
...

$ kubectl get node
NAME      STATUS   ROLES         AGE    VERSION
apollo    Ready    master,node   114m   v1.12.3
boomer    Ready    master,node   114m   v1.12.3
caprica   Ready    master,node   114m   v1.12.3

$ git checkout v2.8.1
Previous HEAD position was 9051aa52 Fix ubuntu-contiv test failed (#3808)
HEAD is now at 2ac1c756 More Feature/2.8 backports for 2.8.1 (#3911)

$ ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml
...
    "msg": "DEPRECATION: non-kubeadm deployment is deprecated from v2.9. Will be removed in next release."
...
Are you sure you want to deploy cluster using the deprecated non-kubeadm mode. (output is hidden):
yes
...

$ kubectl get node
NAME      STATUS   ROLES         AGE     VERSION
apollo    Ready    master,node   2h36m   v1.12.4
boomer    Ready    master,node   2h36m   v1.12.4
caprica   Ready    master,node   2h36m   v1.12.4

$ git checkout v2.8.2
Previous HEAD position was 2ac1c756 More Feature/2.8 backports for 2.8.1 (#3911)
HEAD is now at 4167807f Upgrade to 1.12.5 (#4066)

$ ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml
...
    "msg": "DEPRECATION: non-kubeadm deployment is deprecated from v2.9. Will be removed in next release."
...
Are you sure you want to deploy cluster using the deprecated non-kubeadm mode. (output is hidden):
yes
...

$ kubectl get node
NAME      STATUS   ROLES         AGE    VERSION
apollo    Ready    master,node   3h3m   v1.12.5
boomer    Ready    master,node   3h3m   v1.12.5
caprica   Ready    master,node   3h3m   v1.12.5

$ git checkout v2.8.3
Previous HEAD position was 4167807f Upgrade to 1.12.5 (#4066)
HEAD is now at ea41fc5e backport cve-2019-5736 to release-2.8 (#4234)

$ ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml
...
    "msg": "DEPRECATION: non-kubeadm deployment is deprecated from v2.9. Will be removed in next release."
...
Are you sure you want to deploy cluster using the deprecated non-kubeadm mode. (output is hidden):
yes
...

$ kubectl get node
NAME      STATUS   ROLES         AGE     VERSION
apollo    Ready    master,node   5h18m   v1.12.5
boomer    Ready    master,node   5h18m   v1.12.5
caprica   Ready    master,node   5h18m   v1.12.5

$ git checkout v2.8.4
Previous HEAD position was ea41fc5e backport cve-2019-5736 to release-2.8 (#4234)
HEAD is now at 3901480b go to k8s 1.12.7 (#4400)

$ ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml
...
    "msg": "DEPRECATION: non-kubeadm deployment is deprecated from v2.9. Will be removed in next release."
...
Are you sure you want to deploy cluster using the deprecated non-kubeadm mode. (output is hidden):
yes
...

$ kubectl get node
NAME      STATUS   ROLES         AGE     VERSION
apollo    Ready    master,node   5h37m   v1.12.7
boomer    Ready    master,node   5h37m   v1.12.7
caprica   Ready    master,node   5h37m   v1.12.7

$ git checkout v2.8.5
Previous HEAD position was 3901480b go to k8s 1.12.7 (#4400)
HEAD is now at 6f97687d Release 2.8 robust san handling (#4478)

$ ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml
...
    "msg": "DEPRECATION: non-kubeadm deployment is deprecated from v2.9. Will be removed in next release."
...
Are you sure you want to deploy cluster using the deprecated non-kubeadm mode. (output is hidden):
yes
...

$ kubectl get node
NAME      STATUS   ROLES         AGE     VERSION
apollo    Ready    master,node   5h45m   v1.12.7
boomer    Ready    master,node   5h45m   v1.12.7
caprica   Ready    master,node   5h45m   v1.12.7

$ git checkout v2.9.0
Previous HEAD position was 6f97687d Release 2.8 robust san handling (#4478)
HEAD is now at a4e65c7c Upgrade to Ansible >2.7.0 (#4471)
```

:warning: 重要: 2.8.5 と 2.9.0 の間で k8s-cluster.yml の一部の変数フォーマットが変更になりました :warning:

inventoryのコピーを最新の状態にしておかないと、**あなたのアップグレードは失敗し**、1台目のマスターは修正されて再実行されるまで機能しないままになります。

この時点では、非推奨の警告に従ってクラスターがnon-kubeadmからkubeadmにアップグレードされています。

```ShellSession
ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml

...

$ kubectl get node
NAME      STATUS   ROLES         AGE     VERSION
apollo    Ready    master,node   6h54m   v1.13.5
boomer    Ready    master,node   6h55m   v1.13.5
caprica   Ready    master,node   6h54m   v1.13.5

# Watch out: 2.10.0 is hiding between 2.1.2 and 2.2.0

$ git tag
...
v2.1.0
v2.1.1
v2.1.2
v2.10.0
v2.2.0
...

$ git checkout v2.10.0
Previous HEAD position was a4e65c7c Upgrade to Ansible >2.7.0 (#4471)
HEAD is now at dcd9c950 Add etcd role dependency on kube user to avoid etcd role failure when running scale.yml with a fresh node. (#3240) (#4479)

ansible-playbook -i inventory/mycluster/hosts.ini -b upgrade-cluster.yml

...

$ kubectl get node
NAME      STATUS   ROLES         AGE     VERSION
apollo    Ready    master,node   7h40m   v1.14.1
boomer    Ready    master,node   7h40m   v1.14.1
caprica   Ready    master,node   7h40m   v1.14.1


```

## アップグレードの順序

前述したように、コンポーネントはAnsible playbookにインストールした順番でアップグレードされます。
コンポーネントのインストール順は以下の通りです:

* Docker
* etcd
* kubelet and kube-proxy
* network_plugin (such as Calico or Weave)
* kube-apiserver, kube-scheduler, and kube-controller-manager
* Add-ons (such as KubeDNS)

## アップグレードの検討

KubesprayはetcdとKubernetesコンポーネントで使用される証明書のローテーションをサポートしていますが、いくつかの手動の手順が必要な場合があります。
サービストークンの使用を必要とするポッドがあり、`kube-system` 以外の名前空間にデプロイされている場合は、証明書をローテーションさせた後、影響を受けるポッドを手動で削除する必要があります。
これは、すべてのサービスアカウントのトークンが、それらを生成するために使用されるapiserverのトークンに依存しているからです。
証明書がローテーションすると、すべてのサービスアカウントのトークンもローテーションさせる必要があります。
kubernetes-apps/rotate_tokensロールであれば、kube-system 内のポッドのみが破壊され、再作成されます。
無効になった他のすべてのサービスアカウントトークンは自動的にクリーンアップされますが、他のポッドは、ユーザが展開したポッドへの影響を考慮して削除されません。

### コンポーネントベースのアップグレード

デプロイを行う人はリスクを最小化したり、時間を節約したりするために、特定のコンポーネントをアップグレードしたいと思うかもしれません。
この戦略は、この記事を書いている時点ではCIがカバーしていないので、動作が保証されているわけではありません。

これらのコマンドは、完全に配備された健全な既存のホストをアップグレードする場合にのみ有用です。
デプロイされていないホストや部分的にデプロイされているホストには絶対に使えません。

Upgrade docker:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=docker
```

Upgrade etcd:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=etcd
```

Upgrade vault:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=vault
```

Upgrade kubelet:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=node --skip-tags=k8s-gen-certs,k8s-gen-tokens
```

Upgrade Kubernetes master components:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=master
```

Upgrade network plugins:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=network
```

Upgrade all add-ons:

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=apps
```

Upgrade just helm (assuming `helm_enabled` is true):

```ShellSession
ansible-playbook -b -i inventory/sample/hosts.ini cluster.yml --tags=helm
```
