# k8sのHAエンドポイント

以下のコンポーネントは、高可用性のエンドポイントを必要とします:

* etcd cluster,
* kube-apiserver service instances.

後者は同じ目標を達成するために、NginxやHAProxyのような3rdサイドのリバースプロキシに依存しています。

## Etcd

etcdクライアント(kube-api-master)にはすべてのetcdピアのリストが設定されています。
etcdクラスターに複数のインスタンスがある場合は、すでにHAで構成されています。

## Kube-apiserver

k8sコンポーネントは、リバースプロキシ経由でAPIサーバーにアクセスするためにロードバランサーを必要とします。
Kubesprayには、マスター以外の各Kubernetesノードにあるnginxベースのプロキシのサポートが含まれています。
これはlocalhostロードバランシングと呼ばれます。
Kubernetes apiserverで追加のヘルスチェックを作成するため、専用のロードバランサーよりも効率的ではありませんが、外部LBまたは仮想IP管理が不便なシナリオではより実用的です。
このオプションは`loadbalancer_apiserver_localhost` 変数によって設定されます（デフォルトは `True` です。また、外部の `loadbalancer_apiserver` が定義されている場合は `False` です）。
また `loadbalancer_apiserver_port`を変更して、ローカルの内部ロードバランサーが使用するポートを定義することもできます。
これはデフォルトで `kube_apiserver_port` の値になります。
Kubesprayがローカルマスターロードバランサーを使用するように非マスターノードでのみkubeletとkube-proxyを構成することに注意することも重要です。

ローカルの内部ロードバランサーを使用しない場合は、HAを実現するために独自のロードバランサーを構成する必要があります。
ロードバランサーのデプロイはユーザー次第であり、Kubesprayのansibleロールではカバーされないことに注意してください。
デフォルトでは、非HAエンドポイントのみを設定します。これは、 `kube-master` グループの最初のサーバーノードの `access_ip` またはIPアドレスを指します。
また、特定のロードバランサータイプのエンドポイントを使用するようにクライアントを構成することもできます。
次の図は、apiserverへのトラフィックがどのように送信されるかを示しています。

![Image](figures/loadbalancer_localhost.png?raw=true)

  注意: Kubernetesマスターノードは、マスターロールサービスでTLS authを使用している1.5.0未満のKubernetesにバグがあるため、安全ではないローカルホストアクセスを使用しています。
  これにより、バックエンドが暗号化されていないトラフィックを受信するようになり、異なるノードを相互接続する際にセキュリティ上の問題となる可能性がありますが、外部からのアクセスなしで隔離された管理ネットワークに属している場合は、問題にはならないしれません。

ユーザーは代わりに外部ロードバランサー（LB）を使用することを選択できます。
外部LBは外部クライアントへのアクセスを提供しますが、内部LBはローカルホストへのクライアント接続のみを受け入れます。
フロントエンドの `VIP` アドレスとバックエンドの `IP1, IP2` アドレスが与えられた場合に外部LBとして機能するHAProxyサービスの構成例を次に示します:

```raw
listen kubernetes-apiserver-https
  bind <VIP>:8383
  mode tcp
  option log-health-checks
  timeout client 3h
  timeout server 3h
  server master1 <IP1>:6443 check check-ssl verify none inter 10000
  server master2 <IP2>:6443 check check-ssl verify none inter 10000
  balance roundrobin
```

  注意：これはKubespray以外の場所で管理される構成の例です。

そして、Kubesprayで構成されたクラスターAPIアクセスモードを備えた「クラスター対応」外部LBに対応するグローバル変数の例:

```yml
apiserver_loadbalancer_domain_name: "my-apiserver-lb.example.com"
loadbalancer_apiserver:
  address: <VIP>
  port: 8383
```

  注意：デフォルトのkubernetes apiserver構成はすべてのインターフェースにバインドするため、APIがリッスンしているvipに別のポートを使用するか、APIが特定のインターフェースのみをリッスンするように `kube_apiserver_bind_address` を設定する必要があります（VIPアドレスのポートをバインドするhaproxyとの競合を回避するために）

このドメイン名、またはデフォルトの「lb-apiserver.kubernetes.local」は、`k8s-cluster` グループ内のすべてのサーバの `/etc/hosts` ファイルに追加され、生成された自己署名TLS/SSL証明書にも挿入されます。
HAProxyサービスはHAであるべきであり、VIP管理が必要であることに注意してください。

内部LBと外部設定された(Kubesprayによって構成したのではない)LBを同時に使用する場合には特別なケースがあります。
クラスターはそのような外部LBを認識していないことに注意してください。

  注意: 外部からアクセスされたAPIエンドポイントのTLS/SSL終端は、Kubespray によってカバーされるケースの対象外となります。
  外部LBがTLS/SSL終端機能を提供していることを確認してください。
  あるいは、`supplementary_addresses_in_ssl_keys` リストに外部からの負荷分散されたVIPを指定することもできます。
  指定することでkubesprayは生成されたクラスター証明書にもそれらを追加します。

この特定のケースは別として、`loadbalancer_apiserver` は `loadbalancer_apiserver_localhost` とは同時に設定することはできないと考えられます。

アクセスAPIのエンドポイントは以下のように自動的に評価されます:

| エンドポイントタイプ              | master           | master以外               | 外部                   |
|------------------------------|------------------|-------------------------|-----------------------|
| ローカルLB (デフォルト)          | `https://bip:sp` | `https://lc:nsp`        | `https://m[0].aip:sp` |
| ローカルLB + マネージドでないLB    | `https://bip:sp` | `https://lc:nsp`        | `https://ext`         |
| 外部LB(内部LBでない)            | `https://bip:sp` | `<https://lb:lp>`       | `https://lb:lp`       |
| 外部/内部LBでない               | `https://bip:sp` | `<https://m[0].aip:sp>` | `https://m[0].aip:sp` |

Where:

* `m[0]` - `kube-master` グループの最初のノード;
* `lb` - LBのFQDN, `apiserver_loadbalancer_domain_name`;
* `ext` - 外部で負荷分散されたVIP:ポートとFQDN、Kubesprayによって管理されない;
* `lc` - ローカルホスト;
* `bip` - デフォルトのバインドIP「0.0.0.0」のカスタムバインドIPまたはlocalhost;
* `nsp` -  nginxの安全なポート, `loadbalancer_apiserver_port`, `sp` に従う;
* `sp` - 安全なポート, `kube_apiserver_port`;
* `lp` - LBのポート, `loadbalancer_apiserver.port`, 安全なポートに従う;
* `ip` -　ノードIP, ansible IPに従う;
* `aip` - `access_ip`, ipに従う.

2番目と3番目の列は、内部クラスターアクセスモードを表します。
最後の列は、クラスターAPIに外部からアクセスするためのURIの例を示しています。
Kubesprayはそれとは何の関係もありません。これは情報提供のみを目的としています。

ご覧のように、マスターの内部APIエンドポイントは常にローカルバインドIP（ `https://bip:sp` ）を介して接続されます。

**注意** Kubesprayによってデプロイされたアプリケーションのヘルスチェックなどの場合、マスターのAPIは、ローカルの `kube_apiserver_insecure_bind_address` と ` kube_apiserver_insecure_port` で構成される安全でないエンドポイントを介してアクセスされることに注意してください。

## Optional configurations

### ETCDとLB

外部ロードバランシング（L4/TCPまたはSSLパススルーVIP付きのL7）を使用するには、group_varsで次の変数を上書きする必要があります

* `etcd_access_addresses`
* `etcd_client_url`
* `etcd_cert_alt_names`
* `etcd_cert_alt_ips`

#### VIPとFQDNの例

```yaml
etcd_access_addresses: https://etcd.example.com:2379
etcd_client_url: https://etcd.example.com:2379
etcd_cert_alt_names:
  - "etcd.kube-system.svc.{{ dns_domain }}"
  - "etcd.kube-system.svc"
  - "etcd.kube-system"
  - "etcd"
  - "etcd.example.com" # これはデフォルトのetcd_cert_alt_namesに追加する必要があります
```

#### VIPとFQDN (IPのみ)の例

```yaml
etcd_access_addresses: https://2.3.7.9:2379
etcd_client_url: https://2.3.7.9:2379
etcd_cert_alt_ips:
  - "2.3.7.9"
```
