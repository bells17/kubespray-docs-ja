# Kubesprayで設定できるパラメータ

## Ansibleの一般的な変数

Ansible が自動的に収集したfactは[ここ](https://docs.ansible.com/ansible/playbooks_variables.html#information-discovered-from-systems-facts)で見ることができます。

注目すべき変数には以下のものがあります:

* *ansible_user*: SSH経由で接続するユーザー
* *ansible_default_ipv4.address*: Ansibleが自動的に選択するIPアドレス。
  ``ip -4 route get 8.8.8.8`` コマンドの出力に基づいて生成されます

## Kubesprayで使用される一般的な変数

* *calico_version* - 使用するCalicoのバージョンを指定します
* *calico_cni_version* - 使用するCalico CNIプラグインのバージョンを指定します
* *docker_version* - 使用するDockerのバージョンを指定します（引用符で囲んだ文字列である必要があります）。
`roles/container-engine/docker/vars/*.yml` の *docker_versioned_pkg* に定義されたキーの1つと一致する必要があります。
* *etcd_version* - 使用するETCDのバージョンを指定します
* *ipip* - デフォルトでCalico IPIPカプセル化を有効にします
* *kube_network_plugin* - k8sネットワークプラグインを設定します（デフォルトはCalico）
* *kube_proxy_mode* - k8sプロキシモードをiptablesモードに変更します
* *kube_version* - 特定のKubernetes hyperkubeバージョンを指定します
* *searchdomains* - ホスト名を検索するときに検索するDNSドメインの配列
* *nameservers* - DNSルックアップに使用するネームサーバーの配列
* *preinstall_selinux_state* - selinuxの状態を設定します。許可された値は許容され無効になります

## アドレス変数

* *ip* - サービスのバインドに使用するIP（ホスト変数）
* *access_ip* - 接続に使用する他のホストのIP。しばしば必要となる。
  OpenStackやGCEなどのクラウドからデプロイすると、パブリックIP/フローティングIPとプライベートIPを別々に持っている。
* *ansible_default_ipv4.address* - Kubespray固有ではありませんが、ipおよびaccess_ipが未定義の場合に使用されます
* *loadbalancer_apiserver* - If defined, all hosts will connect to this
  address instead of localhost for kube-masters and kube-master[0] for
  kube-nodes. See more details in the
  [HA guide](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md).
  定義されている場合、kube-masterやkube-nodeのkube-master[0]など、すべてのホストはlocalhostの代わりにこのアドレスに接続します。 詳細は[HAガイド](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md)をご覧ください
* *loadbalancer_apiserver_localhost* - すべてのホストがapiserverに接続する際に内部ロードバランサーのエンドポイントに接続するようにします。
  `loadbalancer_apiserver` と一緒に設定できません。 詳細については、[HAガイド](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ha-mode.md)をご覧ください。

## Cluster variables

Kubernetesがデプロイされるためには、いくつかのパラメータが必要です。
これらは以下のデフォルトのクラスタパラメータです:

* *cluster_name* - クラスタ名 (デフォルトは cluster.local)
* *container_manager* - ノードにインストールするコンテナランタイム(デフォルトはdocker)
* *dns_domain* - クラスタ DNS ドメイン名 (デフォルトは cluster.local)
* *kube_network_plugin* - コンテナネットワーキングに使うプラグイン
* *kube_service_addresses* - クラスタ IP のサブネット (デフォルトは 10.233.0.0/18)。
  kube_pods_subnet と重複してはいけません。
* *kube_pods_subnet* - ポッドIPのサブネット（デフォルトは10.233.64.0/18）。
  kube_service_addresses と重複してはいけません。
* *kube_network_node_prefix* - ノードごとにポッドIP用に割り当てられたサブネット。kube_pods_subnet の残りのビット数によって、クラスタ内のkube-nodeの数が決まります。これを 25 以上に設定すると、`kubelet_max_pods` 変数もそれに応じて調整されていないとplaybookでアサーションが発生します (アサーションは、これをハードリミットとして使用しない calico には適用されません、[Calico IP block sizes](https://docs.projectcalico.org/reference/resources/ippool#block-sizes)を見てください)。
* *skydns_server* - DNS のクラスタ IP (デフォルトは 10.233.0.3)
* *skydns_server_secondary* - coredns_dual展開で使用するCoreDNSのセカンダリクラスタIP(デフォルトは 10.233.0.4)
* *enable_coredns_k8s_external* - 有効にすると、CoreDNSサービスの[k8s_external plugin](https://coredns.io/plugins/k8s_external/)を設定します。
* *coredns_k8s_external_zone* - CoreDNS k8s_externalプラグインが有効な場合に使用されるゾーン
  (デフォルトは k8s_external.local)
* *enable_coredns_k8s_endpoint_pod_names* - 有効にすると、CoreDNSサービス上のkubernetesプラグインのendpoint_pod_namesオプションを設定します。
* *cloud_provider* - GCE または OpenStack 内で運用している場合、追加のKubeletオプションを有効にする (デフォルトは未設定)
* *kube_hostpath_dynamic_provisioner* - KubernetesでPetSets型を使用する場合に必要
* *kube_feature_gates* - alpha/experimental Kubernetes の機能のためのフィーチャーゲートを記述する key=value のペアのリストです。(デフォルトは `[]`) です)
* *authorization_modes* - クラスターが設定されるべき[認可モード](https://kubernetes.io/docs/admin/authorization/#using-flags-for-your-authorization-module)のリストです。
  デフォルトは `['Node', 'RBAC']` (ノードとRBACオーサライザー) です。
  注意: `Node` と `RBAC` はデフォルトで有効になっています。以前にデプロイされたクラスタは、RBACモードに変換することができます。ただし、Kubernetes APIに依存しているアプリケーションでは、サービスアカウントとクラスタのロールバインディングが必要になります。authorization_modesを`[]`に設定することで、この設定をオーバーライドすることができます。

クラウドプロバイダがインスタンスのプライベートアドレスのように ``10.233.0.0/16`` を使用している場合は、``kube_service_addresses`` や ``kube_pods_subnet`` に別の値を指定するようにしてください。例えば ``172.18.0.0/16`` のようにです。

## DNS変数

デフォルトでは、ホストはアップストリームのDNSサーバーとして8.8.8.8が設定され、既存の/etc/resolv.confからその他の設定はすべて失われます。
以下の変数を設定して、要件に合わせて設定します。

* *upstream_dns_servers* - KubesprayでデプロイされたDNSに追加する、ホスト上に構成されたアップストリームのDNSサーバーの配列
* *nameservers* - ホストが使用するように設定されたDNSサーバーの配列
* *searchdomains* - 最大4つの検索ドメインの配列

詳細については[DNSスタック](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/dns-stack.md)を参照してください。

## その他のサービスの変数

* *docker_options* - 一般的には ``--insecure-registry=myregistry.mydomain:5000`` を設定するのに使われます。
* *docker_plugins* - このリストを使って、インストールする[Docker plugin](https://docs.docker.com/engine/extend/)を定義することができます。
* *containerd_config* - containerdの設定ファイル(通常は/etc/containerd/config.toml)のパラメータを制御します。
  [デフォルト設定](https://github.com/kubernetes-sigs/kubespray/blob/master/roles/container-engine/containerd/defaults/main.yml)はinventory変数で上書きすることができます。
* *http_proxy/https_proxy/no_proxy* - プロキシの後ろにデプロイするためのプロキシ変数。no_proxy のデフォルトは、各ノードに対応するすべての内部クラスタIPとホスト名であることに注意してください。
* *kubelet_deployment_type* - どのプラットフォームにkubeletをデプロイするかを制御します。
  利用可能なオプションは ``host`` と ``docker`` です。
  新しいリリースでは ``docker`` モードは動作しません。
  Kubernetes v1.7 からはデフォルトが ``host`` になりました。
  v1.7以前はデフォルトがDockerでした。
  これはcgroup [issues](https://github.com/kubernetes/kubernetes/issues/43704)が原因です。
* *kubelet_load_modules* - いくつかのことについて、kubeletはカーネルモジュールをロードする必要があります。
  例えばコンテナに永続的なボリュームをマウントするためには動的なカーネルサービスが必要です。
  これらはプリインストールされたkubernetesプロセスではロードされない場合があります。
  例えばcephやrbdのバックアップされたボリュームなどです。
  この変数をtrueに設定すると、kubelet にカーネルモジュールをロードさせることができます。
* *kubelet_cgroup_driver* - Kubeletのcgroup-driverオプションを手動で上書きすることができます。
  デフォルトではDockerの設定に合わせた自動検出が使用されます。
* *kubelet_rotate_certificates* - 証明書の有効期限が近づいたときに、kube-apiserver に新しい証明書を要求することで、kubeletのクライアント証明書を自動的にローテーションさせます。
* *node_labels* - ラベルはkubeletの--node-labelsパラメータによってノードに適用されます。
  例えば、ラベルはinventory内の変数を変数で設定することができます。また、group_varsでより広範囲に設定することができます。
  *node_labels* はdictまたはカンマ区切りのラベル文字列として定義することができます:

```yml
node_labels:
  label1_name: label1_value
  label2_name: label2_value

node_labels: "label1_name=label1_value,label2_name=label2_value"
```

* *node_taints* - Taints applied to nodes via kubelet --register-with-taints parameter.
  For example, taints can be set in the inventory as variables or more widely in group_vars.
  *node_taints* has to be defined as a list of strings in format `key=value:effect`, e.g.:

  taintはkubeletの--register-with-taintsパラメータによってノードに適用されます。
  例えば、taintはinventory内の変数を変数で設定することができます。また、group_varsでより広範囲に設定することができます。
  *node_taints*は、`key=value:effect`形式の文字列のリストとして定義されなければなりません。
  例:

```yml
node_taints:
  - "node.example.com/external=true:NoSchedule"
```

* *podsecuritypolicy_enabled* - `true`に設定すると、PodSecurityPolicyアドミッションコントローラを有効にし、`privileged` (`kube-system` 名前空間とkubeletにあるすべてのリソースに適用) と `restricted` (他のすべての名前空間に適用) の2つのポリシーを定義します。
  kube-system 名前空間にデプロイされたアドオンに適用されます。
* *kubernetes_audit* - `true` に設定すると監査機能を有効にします。
  監査パラメータは、以下の変数(デフォルト値を以下に示す)で調整することができます:
  * `audit_log_path`: /var/log/audit/kube-apiserver-audit.log
  * `audit_log_maxage`: 30
  * `audit_log_maxbackups`: 1
  * `audit_log_maxsize`: 100
  * `audit_policy_file`: "{{ kube_config_dir }}/audit-policy/apiserver-audit-policy.yaml"

  デフォルトでは、`audit_policy_file` には [デフォルトルール](https://github.com/kubernetes-sigs/kubespray/blob/master/roles/kubernetes/master/templates/apiserver-audit-policy.yaml.j2) が含まれており、これは `audit_policy_custom_rules` 変数で上書きすることができます。

### Kubernetesコンポーネントのカスタムフラグ

すべてのKubernetesコンポーネントに対して、カスタムフラグを渡すことができます。これにより、すべてのデプロイメントに適用できないデフォルトのデプロイメントを変更する必要がある場合のエッジケースに対応することができます。

これらの変数は、kubeletのYAML設定ファイルに挿入される設定パラメータのキーバリューのペアの辞書形式で、kubeletの追加フラグを指定することができます。`kubelet_node_config_extra_args` はkubeletの設定をノードにのみ適用し、マスターには適用しません。例:

```yml
kubelet_config_extra_args:
  evictionHard:
    memory.available: "100Mi"
  evictionSoftGracePeriod:
    memory.available: "30s"
  evictionSoft:
    memory.available: "300Mi"
```

可能な変数は以下の通りです:

* *kubelet_config_extra_args*
* *kubelet_node_config_extra_args*

以前は、同じパラメータをフラグとしてkubeletバイナリに以下の変数を渡すことができました:

* *kubelet_custom_flags*
* *kubelet_node_custom_flags*

`kubelet_node_custom_flags` はマスターではなくノードにのみ適用されます。例:

```yml
kubelet_custom_flags:
  - "--eviction-hard=memory.available<100Mi"
  - "--eviction-soft-grace-period=memory.available=30s"
  - "--eviction-soft=memory.available<300Mi"
```

この代替案は非推奨であり、kubeletからフラグが完全に削除されるまで残ります。

APIサーバ、コントローラ、スケジューラコンポーネントの追加フラグは、以下の変数を使用してkubeadmのYAML設定ファイルに挿入される設定パラメータのキーバリューのペアの辞書形式で指定することができます:

* *kube_kubeadm_apiserver_extra_args*
* *kube_kubeadm_controller_extra_args*
* *kube_kubeadm_scheduler_extra_args*

## アプリケーション変数

* *helm_version* - デフォルトはv3.xで、Helm 2.xをインストールするにはv2のバージョン(例: `v2.16.1`)に設定します(Tillerもインストールされます！)。
  Tillerを実行している既存のクラスタにv3を選択すると、そのままになります。その場合、Tillerを手動で削除する必要があります。

## ユーザーアカウント

変数 `kube_basic_auth` はデフォルトでは false になっていますが、true に設定すると管理者権限を持つユーザが `kube` という名前で作成されます。
パスワードはデプロイ後に `{{ credentials_dir }}/kube_user.creds` というファイル(デフォルトでは `credentials_dir` に `{{ inventory_dir }}/credentials` が設定されています) を見ることで確認できます。
これにはランダムに生成されたパスワードが含まれています。
独自のパスワードを設定したい場合は、このファイルを自分で作成/変更するか、 `kube_api_pwd` 変数を変更してください。
