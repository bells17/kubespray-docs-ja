# Kubesprayドキュメント(日本語訳)

- この翻訳は[Kubespary](https://github.com/kubernetes-sigs/kubespray)のドキュメントの非公式日本語訳です
- 個人の学習用に適当に翻訳しているだけなので、間違いなどが多々あるかもしれません
- 翻訳は[v2.13.2](https://github.com/kubernetes-sigs/kubespray/tree/v2.13.2)ベースです

---

# 本番環境に対応したKubernetesクラスタの構築ツール

![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)

もし質問があれば[bells17.io/kubespray-docs-ja/](https://bells17.io/kubespray-docs-ja/)をチェックして、[kubernetes slack](https://kubernetes.slack.com)の **\#kubespray**チャンネルに参加してください。
[ここ](http://slack.k8s.io/)から招待を受け取れます

- **GCE、Azure、OpenStack、AWS、vSphere、Packet(ベアメタル)、Oracle Cloud Infrastructure(実験段階)またはベアメタル**上にKubernetesクラスターを構築
- **高可用性**クラスターを構築
- **Kubernetesクラスターの構成変更**(ネットワークプラグインを選択可能など)
- 最も人気な**Linuxディストリビューション**をサポート
- **継続的インテグレーションテストs**

## クイックスタート

クラスタをデプロイするには

### Ansible

#### 使い方

```ShellSession
# 依存パッケージを ``requirements.txt`` からインストール
sudo pip3 install -r requirements.txt

# ``inventory/sample`` を ``inventory/mycluster`` にコピー
cp -rfp inventory/sample inventory/mycluster

# inventoryビルダーを使ってAnsible inventoryファイルをアップデート
declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# ``inventory/mycluster/group_vars`` 内のパラメータの見直しと変更
cat inventory/mycluster/group_vars/all/all.yml
cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml

# Ansible Playbookを使ってKubesprayをデプロイする - rootでplaybookを実行する
# 例えば /etc/ にSSL鍵を書いたり、パッケージをインストールしたり、様々なsystemdデーモンと対話したりするためには、`--become`オプションが必要です。
# --become なしではプレイブックの実行に失敗します!
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

注意: Ansible が既にコントロールマシンにシステムパッケージを介してインストールされている場合、 `sudo pip install -r requirements.txt` でインストールされた他の python パッケージは Ansible とは異なるディレクトリツリー(例: Ubuntuの `/usr/local/lib/python2.7/dist-packages`)に移動します(例: Ubuntuの `/usr/lib/python2.7/dist-packages/ansible`)。
その場合`ansible-playbook` コマンドで以下のエラーが出て失敗します。

```raw
ERROR! no action detected in task. This often indicates a misspelled module name, or incorrect module path.
```

おそらくrequirements.txtにあるモジュール(例："unseal vault")に依存するタスクを指していると思われます。

これを解決する方法としては、Ansible パッケージをアンインストールしてから pip 経由でインストールすることが考えられますが、必ずしも可能とは限りません。
回避策としては、環境変数 `ANSIBLE_LIBRARY` と `ANSIBLE_MODULE_UTILS` をそれぞれ pip パッケージのインストール場所の `ansible/modules` と `ansible/module_utils` サブディレクトリに設定することです。これは `ansible-playbook` を実行する前に `pip show [package]` の出力の Location フィールドで見つけることができます。

### Vagrant

Vagrantの場合は、プロビジョニング作業のためにpythonの依存関係をインストールする必要があります。
Pythonとpipがインストールされているか確認します:

```ShellSession
python -V && pip -V
```

これでバージョンが返ってくれば問題ありません。そうでない場合は、ここからPythonをダウンロードしてインストールしてください <https://www.python.org/downloads/source/>。
必要な要件をインストールする

```ShellSession
sudo pip install -r requirements.txt
vagrant up
```

## ドキュメント

- [要件](#要件)
- [Kubespray vs ...](docs/comparisons.md)
- [はじめに](docs/getting-started.md)
- [Ansible inventoryとタグ](docs/ansible.md)
- [既存のAnsibleリポジトリと統合する](docs/integration.md)
- [データ変数のデプロイ](docs/vars.md)
- [DNSスタック](docs/dns-stack.md)
- [HAモード](docs/ha-mode.md)
- [ネットワークプラグイン](#network-plugins)
- [Vagrantのインストール](docs/vagrant.md)
- [CoreOSブートストラップ](docs/coreos.md)
- [Fedora CoreOSブートストラップ](docs/fcos.md)
- [Debian Jessieセットアップ](docs/debian.md)
- [openSUSEセットアップ](docs/opensuse.md)
- [ダウンロードした成果物](docs/downloads.md)
- [クラウドプロバイダー](docs/cloud.md)
- [OpenStack](docs/openstack.md)
- [AWS](docs/aws.md)
- [Azure](docs/azure.md)
- [vSphere](docs/vsphere.md)
- [Packet Host](docs/packet.md)
- [大規模デプロイ](docs/large-deployments.md)
- [ノードの追加/入れ替え](docs/nodes.md)
- [アップグレードの基本](docs/upgrades.md)
- [ロードマップ](docs/roadmap.md)

## サポートされるLinuxディストリビューション

- **Container Linux by CoreOS**
- **Debian** Buster, Jessie, Stretch, Wheezy
- **Ubuntu** 16.04, 18.04
- **CentOS/RHEL** 7, 8 (experimental: see [centos 8 notes](docs/centos8.md)
- **Fedora** 30, 31
- **Fedora CoreOS** (experimental: see [fcos Note](docs/fcos.md))
- **openSUSE** Leap 42.3/Tumbleweed
- **Oracle Linux** 7

Note: Upstart/SysV initベースのOSタイプはサポートしてません。

## Supported Components

- Core
  - [kubernetes](https://github.com/kubernetes/kubernetes) v1.17.7
  - [etcd](https://github.com/coreos/etcd) v3.3.12
  - [docker](https://www.docker.com/) v18.06 (see note)
  - [containerd](https://containerd.io/) v1.2.13
  - [cri-o](http://cri-o.io/) v1.17 (experimental: see [CRI-O Note](docs/cri-o.md). fedora, ubuntu and centosベースのOSのみ)
- Network Plugin
  - [cni-plugins](https://github.com/containernetworking/plugins) v0.8.6
  - [calico](https://github.com/projectcalico/calico) v3.13.2
  - [canal](https://github.com/projectcalico/canal) (given calico/flannel versions)
  - [cilium](https://github.com/cilium/cilium) v1.7.2
  - [contiv](https://github.com/contiv/install) v1.2.1
  - [flanneld](https://github.com/coreos/flannel) v0.12.0
  - [kube-router](https://github.com/cloudnativelabs/kube-router) v0.4.0
  - [multus](https://github.com/intel/multus-cni) v3.4.1
  - [weave](https://github.com/weaveworks/weave) v2.6.2
- Application
  - [cephfs-provisioner](https://github.com/kubernetes-incubator/external-storage) v2.1.0-k8s1.11
  - [rbd-provisioner](https://github.com/kubernetes-incubator/external-storage) v2.1.1-k8s1.11
  - [cert-manager](https://github.com/jetstack/cert-manager) v0.11.1
  - [coredns](https://github.com/coredns/coredns) v1.6.5
  - [ingress-nginx](https://github.com/kubernetes/ingress-nginx) v0.30.0

注意: 検証済みの[dockerのバージョン](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.16.md)のリストが1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09に更新されました。kubeadmはDocker 18.09.0以降を正しく認識するようになりましたが、18.06をデフォルトのサポートバージョンとして扱います。kubeletはdockerの非標準バージョンナンバリングで壊れてしまうことがありました(もうセマンティックバージョニングを使用していません)。
自動更新がクラスタを壊さないようにするには、yum versionlock pluginや apt pin などを利用してください。

## 要件

- **Kubernetesの最低バージョンはv1.15です**
- **Ansible v2.9+とJinja 2.11+、python-netaddrがAnsibleコマンドを実行するサーバーにインストールされている必要があります**
- 対象のサーバーはdockerイメージをpullするために**インターネットにアクセスできる**必要があります。それ以外の場合は追加の設定が必要です([オフライン環境を参照](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/offline-environment.md))
- 対象のサーバーには**IPv4フォワーディング**の許可設定が必要です
- **SSHキー**がインベントリのすべてのサーバにコピーされている必要があります
- **ファイアウォールは管理されていない**ので、自分自身で管理する必要があります。
    デプロイ時の問題を避けるためには、ファイアウォールを無効にする必要があります
- kubesprayをroot以外のユーザアカウントから実行する場合は、対象のサーバで正しい権限昇格方法を設定しておく必要があります。
    その際 `ansible_become` フラグまたはコマンドパラメータ `--become` または `-b` を指定する必要があります

ハードウェア:
これらの制限はKubesprayによって安全に守られています。実際に必要とされる作業量は異なる場合があります。サイジングガイドについては[大規模クラスターを構築する](https://kubernetes.io/docs/setup/cluster-large/#size-of-master-and-master-components)を参照してください。

- Master
  - Memory: 1500 MB
- Node
  - Memory: 1024 MB

## ネットワークプラグイン

10個のネットワークプラグインを選ぶことができます(デフォルト: `calico`、Vagrantは `flannel` を使用しています)

- [flannel](docs/flannel.md): gre/vxlan (layer 2) networking.

- [Calico](https://docs.projectcalico.org/latest/introduction/) is a networking and network policy provider. Calico supports a flexible set of networking options
    designed to give you the most efficient networking across a range of situations, including non-overlay
    and overlay networks, with or without BGP. Calico uses the same engine to enforce network policy for hosts,
    pods, and (if using Istio and Envoy) applications at the service mesh layer.

- [canal](https://github.com/projectcalico/canal): a composition of calico and flannel plugins.

- [cilium](http://docs.cilium.io/en/latest/): layer 3/4 networking (as well as layer 7 to protect and secure application protocols), supports dynamic insertion of BPF bytecode into the Linux kernel to implement security services, networking and visibility logic.

- [contiv](docs/contiv.md): supports vlan, vxlan, bgp and Cisco SDN networking. This plugin is able to
    apply firewall policies, segregate containers in multiple network and bridging pods onto physical networks.

- [weave](docs/weave.md): Weave is a lightweight container overlay network that doesn't require an external K/V database cluster.
    (Please refer to `weave` [troubleshooting documentation](https://www.weave.works/docs/net/latest/troubleshooting/)).

- [kube-ovn](docs/kube-ovn.md): Kube-OVN integrates the OVN-based Network Virtualization with Kubernetes. It offers an advanced Container Network Fabric for Enterprises.

- [kube-router](docs/kube-router.md): Kube-router is a L3 CNI for Kubernetes networking aiming to provide operational
    simplicity and high performance: it uses IPVS to provide Kube Services Proxy (if setup to replace kube-proxy),
    iptables for network policies, and BGP for ods L3 networking (with optionally BGP peering with out-of-cluster BGP peers).
    It can also optionally advertise routes to Kubernetes cluster Pods CIDRs, ClusterIPs, ExternalIPs and LoadBalancerIPs.

- [macvlan](docs/macvlan.md): Macvlan is a Linux network driver. Pods have their own unique Mac and Ip address, connected directly the physical (layer 2) network.

- [multus](docs/multus.md): Multus is a meta CNI plugin that provides multiple network interface support to pods. For each interface Multus delegates CNI calls to secondary CNI plugins such as Calico, macvlan, etc.

どのネットワークプラグインを選ぶかは変数 `kube_network_plugin` で定義されています。
代わりに組み込みのクラウドプロバイダのネットワークを利用するオプションもあります。
[Network checker](docs/netcheck.md)も参照してください。

## Community docs and resources

- [kubernetes.io/docs/setup/production-environment/tools/kubespray/](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/)
- [kubespray, monitoring and logging](https://github.com/gregbkr/kubernetes-kargo-logging-monitoring) by @gregbkr
- [Deploy Kubernetes w/ Ansible & Terraform](https://rsmitty.github.io/Terraform-Ansible-Kubernetes/) by @rsmitty
- [Deploy a Kubernetes Cluster with Kubespray (video)](https://www.youtube.com/watch?v=CJ5G4GpqDy0)

## Tools and projects on top of Kubespray

- [Digital Rebar Provision](https://github.com/digitalrebar/provision/blob/v4/doc/integrations/ansible.rst)
- [Terraform Contrib](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform)

## CI Tests

[![Build graphs](https://gitlab.com/kargo-ci/kubernetes-sigs-kubespray/badges/master/build.svg)](https://gitlab.com/kargo-ci/kubernetes-sigs-kubespray/pipelines)

CI/end-to-end tests sponsored by Google (GCE)
See the [test matrix](docs/test_cases.md) for details.

