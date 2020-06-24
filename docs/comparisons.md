# Comparison

## Kubespray vs [Kops](https://github.com/kubernetes/kops)

Kubesprayはベアメタルとほとんどのクラウド上で動作し、Ansibleを基板にしてプロビジョニングとオーケストレーションを行います。
Kopsはプロビジョニングとオーケストレーション自体を実行するため、デプロイメントプラットフォームの柔軟性が低くなります。Ansibleに慣れている人、既存のAnsibleデプロイメント、または複数のプラットフォームでKubernetesクラスターを実行したいと考えている人には、Kubesprayは良い選択です。
しかし、Kopsは、サポートしているクラウドのユニークな機能とより緊密に統合されているので、当面は1つのプラットフォームしか使用しないことがわかっている場合には、より良い選択となるかもしれません。

## Kubespray vs [Kubeadm](https://github.com/kubernetes/kubeadm)

Kubeadmは、自己ホスト型レイアウトやダイナミックディスカバリーサービスなど、Kubernetesクラスターのライフサイクル管理に関するドメイン知識を提供します。
新しい[オペレーターの世界]（https://coreos.com/blog/introducing-operators.html）に属していた場合、「Kubernetes cluster operator」と呼ばれた可能性があります。
しかしKubesprayは「OSオペレーター」のAnsibleな世界からの一般的な構成管理タスクに加えて、いくつかのk8sクラスターの初期化処理（ネットワークプラグインを含む）とコントロールプレーンブートストラップを行います。

Kubesprayは、v2.3以降のクラスター作成に `kubeadm` を利用します(v2.8からは非推奨のnon-kubeadmデプロイもサポートしています)。
kubeadmを利用することにより、ライフサイクル管理に関するドメイン知識をkubeadmに委託し、汎用OS構成の負荷を軽減します。
これにより、双方に利益がもたらされると期待しています。
