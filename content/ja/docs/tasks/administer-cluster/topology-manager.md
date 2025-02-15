---
title: ノードのトポロジー管理ポリシーを制御する

content_template: templates/task
min-kubernetes-server-version: v1.18
---

{{% capture overview %}}

{{< feature-state state="beta" for_k8s_version="v1.18" >}}

近年、CPUやハードウェア・アクセラレーターの組み合わせによって、レイテンシーが致命的となる実行や高いスループットを求められる並列計算をサポートするシステムが増えています。このようなシステムには、通信、科学技術計算、機械学習、金融サービス、データ分析などの分野のワークロードが含まれます。このようなハイブリッドシステムは、高い性能の環境で構成されます。

最高のパフォーマンスを引き出すために、CPUの分離やメモリーおよびデバイスの位置に関する最適化が求められます。しかしながら、Kubernetesでは、これらの最適化は分断されたコンポーネントによって処理されます。

_トポロジーマネージャー_ はKubeletコンポーネントの1つで最適化の役割を担い、コンポーネント群を調和して機能させます。

{{% /capture %}}

{{% capture prerequisites %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

{{% /capture %}}

{{% capture steps %}}

## トポロジーマネージャーはどのように機能するか

トポロジーマネージャー導入前は、KubernetesにおいてCPUマネージャーやデバイスマネージャーはそれぞれ独立してリソースの割り当てを決定します。
これは、マルチソケットのシステムでは望ましくない割り当てとなり、パフォーマンスやレイテンシーが求められるアプリケーションは、この望ましくない割り当てに悩まされます。
この場合の望ましくない例として、CPUやデバイスが異なるNUMAノードに割り当てられ、それによりレイテンシー悪化を招くことが挙げられます。

トポロジーマネージャーはKubeletコンポーネントであり、信頼できる情報源として振舞います。それによって、他のKubeletコンポーネントはトポロジーに沿ったリソース割り当ての選択を行うことができます。

トポロジーマネージャーは *Hint Providers* と呼ばれるコンポーネントのインターフェースを提供し、トポロジー情報を送受信します。トポロジーマネージャーは、ノード単位のポリシー群を保持します。ポリシーについて以下で説明します。

トポロジーマネージャーは *Hint Providers* からトポロジー情報を受け取ります。トポロジー情報は、利用可能なNUMAノードと優先割り当て表示を示すビットマスクです。トポロジーマネージャーのポリシーは、提供されたヒントに対して一連の操作を行い、ポリシーに沿ってヒントをまとめて最適な結果を得ます。もし、望ましくないヒントが保存された場合、ヒントの優先フィールドがfalseに設定されます。現在のポリシーでは、最も狭い優先マスクが優先されます。

選択されたヒントはトポロジーマネージャーの一部として保存されます。設定されたポリシーにしたがい、選択されたヒントに基づいてノードがPodを許可したり、拒否することができます。
トポロジーマネージャーに保存されたヒントは、*Hint Providers* が使用しリソース割り当てを決定します。

### トポロジーマネージャーの機能を有効にする

トポロジーマネージャーをサポートするには、`TopologyManager` [フィーチャーゲート](/ja/docs/reference/command-line-tools-reference/feature-gates/)を有効にする必要があります。Kubernetes 1.18ではデフォルトで有効です。

## トポロジーマネージャーのスコープとポリシー

トポロジーマネージャは現在:

 - 全てのQoAクラスのPodを調整する
 - Hint Providerによって提供されたトポロジーヒントから、要求されたリソースを調整する

これらの条件が合致した場合、トポロジーマネージャーは要求されたリソースを調整します。

この調整をどのように実行するかカスタマイズするために、トポロジーマネージャーは2つのノブを提供します: `スコープ` と`ポリシー`です。

`スコープ`はリソースの配置を行う粒度を定義します(例:`pod`や`container`)。そして、`ポリシー`は調整を実行するための実戦略を定義します(`best-effort`, `restricted`, `single-numa-node`等)。

現在利用可能な`スコープ`と`ポリシー`の値について詳細は以下の通りです。

{{< note >}}
PodのSpecにある他の要求リソースとCPUリソースを調整するために、CPUマネージャーを有効にし、適切なCPUマネージャーのポリシーがノードに設定されるべきです。[CPU管理ポリシー](/docs/tasks/administer-cluster/cpu-management-policies/)を参照してください。
{{< /note >}}

{{< note >}}
PodのSpecにある他の要求リソースとメモリー（およびhugepage）リソースを調整するために、メモリーマネージャーを有効にし、適切なメモリーマネージャーポリシーがノードに設定されるべきです。[メモリーマネージャー](/docs/tasks/administer-cluster/memory-manager/) のドキュメントを確認してください。
{{< /note >}}

### トポロジーマネージャーのスコープ

トポロジーマネージャーは、以下の複数の異なるスコープでリソースの調整を行う事が可能です:

* `container` (デフォルト)
* `pod`

いずれのオプションも、`--topology-manager-scope`フラグによって、kubelet起動時に選択できます。

### containerスコープ

`container`スコープはデフォルトで使用されます。

このスコープでは、トポロジーマネージャーは連続した複数のリソース調整を実行します。つまり、Pod内の各コンテナは、分離された配置計算がされます。言い換えると、このスコープでは、コンテナを特定のNUMAノードのセットにグループ化するという概念はありません。実際には、トポロジーマネージャーは各コンテナのNUMAノードへの配置を任意に実行します。

コンテナをグループ化するという概念は、以下のスコープで設定・実行されます。例えば、`pod`スコープが挙げられます。

### podスコープ

`pod`スコープを選択するには、コマンドラインで`--topology-manager-scope=pod`オプションを指定してkubeletを起動します。

このスコープでは、Pod内全てのコンテナを共通のNUMAノードのセットにグループ化することができます。トポロジーマネージャーはPodをまとめて1つとして扱い、ポッド全体（全てのコンテナ）を単一のNUMAノードまたはNUMAノードの共通セットのいずれかに割り当てようとします。以下の例は、さまざまな場面でトポロジーマネージャーが実行する調整を示します:

* 全てのコンテナは、単一のNUMAノードに割り当てられます。
* 全てのコンテナは、共有されたNUMAノードのセットに割り当てられます。

Pod全体に要求される特定のリソースの総量は[有効なリクエスト／リミット](/ja/docs/concepts/workloads/pods/init-containers/#resources)の式に従って計算されるため、この総量の値は以下の最大値となります。
* 全てのアプリケーションコンテナのリクエストの合計。
* リソースに対するinitコンテナのリクエストの最大値。

`pod`スコープと`single-numa-node`トポロジーマネージャーポリシーを併用することは、レイテンシーが重要なワークロードやIPCを行う高スループットのアプリケーションに対して特に有効です。両方のオプションを組み合わせることで、Pod内の全てのコンテナを単一のNUMAノードに配置できます。そのため、PodのNUMA間通信によるオーバーヘッドを排除することができます。

`single-numa-node`ポリシーの場合、可能な割り当ての中に適切なNUMAノードのセットが存在する場合にのみ、Podが許可されます。上の例をもう一度考えてみましょう:

* 1つのNUMAノードのみを含むセット - Podが許可されます。
* 2つ以上のNUMAノードを含むセット - Podが拒否されます(1つのNUMAノードの代わりに、割り当てを満たすために2つ以上のNUMAノードが必要となるため)。


要約すると、トポロジーマネージャーはまずNUMAノードのセットを計算し、それをトポロジーマネージャーのポリシーと照合し、Podの拒否または許可を検証します。

### トポロジーマネージャーのポリシー

トポロジーマネージャーは4つの調整ポリシーをサポートします。`--topology-manager-policy`というKubeletフラグを通してポリシーを設定できます。
4つのサポートされるポリシーがあります:

* `none` (デフォルト)
* `best-effort`
* `restricted`
* `single-numa-node`

{{< note >}}
トポロジーマネージャーが **pod** スコープで設定された場合、コンテナはポリシーによって、Pod全体の要求として反映します。
したがって、Podの各コンテナは **同じ** トポロジー調整と同じ結果となります。
{{< /note >}}

### none ポリシー {#policy-none}

これはデフォルトのポリシーで、トポロジーの調整を実行しません。

### best-effort ポリシー {#policy-best-effort}

Pod内の各コンテナに対して、`best-effort` トポロジー管理ポリシーが設定されたkubeletは、各Hint Providerを呼び出してそれらのリソースの可用性を検出します。
トポロジーマネージャーはこの情報を使用し、そのコンテナの推奨されるNUMAノードのアフィニティーを保存します。アフィニティーが優先されない場合、トポロジーマネージャーはこれを保存し、Podをノードに許可します。

*Hint Providers* はこの情報を使ってリソースの割り当てを決定します。

### restricted ポリシー {#policy-restricted}

Pod内の各コンテナに対して、`restricted` トポロジー管理ポリシーが設定されたkubeletは各Hint Providerを呼び出してそれらのリソースの可用性を検出します。
トポロジーマネージャーはこの情報を使用し、そのコンテナの推奨されるNUMAノードのアフィニティーを保存します。アフィニティーが優先されない場合、トポロジーマネージャーはPodをそのノードに割り当てることを拒否します。この結果、PodはPodの受付失敗となり`Terminated` 状態になります。

Podが一度`Terminated`状態になると、KubernetesスケジューラーはPodの再スケジューリングを試み **ません** 。Podの再デプロイをするためには、ReplicasetかDeploymenを使用してください。`Topology Affinity`エラーとなったpodを再デプロイするために、外部のコントロールループを実行することも可能です。

Podが許可されれば、 *Hint Providers* はこの情報を使ってリソースの割り当てを決定します。

### single-numa-node ポリシー {#policy-single-numa-node}

Pod内の各コンテナに対して、`single-numa-node`トポロジー管理ポリシーが設定されたkubeletは各Hint Prociderを呼び出してそれらのリソースの可用性を検出します。
トポロジーマネージャーはこの情報を使用し、単一のNUMAノードアフィニティが可能かどうか決定します。
可能な場合、トポロジーマネージャーは、この情報を保存し、*Hint Providers* はこの情報を使ってリソースの割り当てを決定します。
不可能な場合、トポロジーマネージャーは、Podをそのノードに割り当てることを拒否します。この結果、Pod は Pod の受付失敗となり`Terminated`状態になります。

Podが一度`Terminated`状態になると、KubernetesスケジューラーはPodの再スケジューリングを試み**ません**。Podの再デプロイをするためには、ReplicasetかDeploymentを使用してください。`Topology Affinity`エラーとなったpodを再デプロイするために、外部のコントロールループを実行することも可能です。

### Podとトポロジー管理ポリシーの関係

以下のようなpodのSpecで定義されるコンテナを考えます:

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
```

`requests`も`limits`も定義されていないため、このPodは`BestEffort`QoSクラスで実行します。

```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

requestsがlimitsより小さい値のため、このPodは`Burstable`QoSクラスで実行します。

選択されたポリシーが`none`以外の場合、トポロジーマネージャーは、これらのPodのSpecを考慮します。トポロジーマネージャーは、Hint Providersからトポロジーヒントを取得します。CPUマネージャーポリシーが`static`の場合、デフォルトのトポロジーヒントを返却します。これらのPodは明示的にCPUリソースを要求していないからです。


```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
        example.com/device: "1"
      requests:
        memory: "200Mi"
        cpu: "2"
        example.com/device: "1"
```

整数値でCPUリクエストを指定されたこのPodは、`requests`が`limits`が同じ値のため、`Guaranteed`QoSクラスで実行します。


```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "300m"
        example.com/device: "1"
      requests:
        memory: "200Mi"
        cpu: "300m"
        example.com/device: "1"
```

CPUの一部をリクエストで指定されたこのPodは、`requests`が`limits`が同じ値のため、`Guaranteed`QoSクラスで実行します。


```yaml
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        example.com/deviceA: "1"
        example.com/deviceB: "1"
      requests:
        example.com/deviceA: "1"
        example.com/deviceB: "1"
```
CPUもメモリもリクエスト値がないため、このPodは `BestEffort` QoSクラスで実行します。

トポロジーマネージャーは、上記Podを考慮します。トポロジーマネージャーは、Hint ProvidersとなるCPUマネージャーとデバイスマネージャーに問い合わせ、トポロジーヒントを取得します。

整数値でCPU要求を指定された`Guaranteed`QoSクラスのPodの場合、`static`が設定されたCPUマネージャーポリシーは、排他的なCPUに関するトポロジーヒントを返却し、デバイスマネージャーは要求されたデバイスのヒントを返します。

CPUの一部を要求を指定された`Guaranteed`QoSクラスのPodの場合、排他的ではないCPU要求のため`static`が設定されたCPUマネージャーポリシーはデフォルトのトポロジーヒントを返却します。デバイスマネージャーは要求されたデバイスのヒントを返します。

上記の`Guaranteed`QoSクラスのPodに関する2ケースでは、`none`で設定されたCPUマネージャーポリシーは、デフォルトのトポロジーヒントを返却します。

`BestEffort`QoSクラスのPodの場合、`static`が設定されたCPUマネージャーポリシーは、CPUの要求がないためデフォルトのトポロジーヒントを返却します。デバイスマネージャーは要求されたデバイスごとのヒントを返します。

トポロジーマネージャーはこの情報を使用してPodに最適なヒントを計算し保存します。保存されたヒントは Hint Providersが使用しリソースを割り当てます。 

### 既知の制限
1. トポロジーマネージャーが許容するNUMAノードの最大値は8です。8より多いNUMAノードでは、可能なNUMAアフィニティを列挙しヒントを生成する際に、生成する状態数が爆発的に増加します。

2. スケジューラーはトポロジーを意識しません。そのため、ノードにスケジュールされた後に実行に失敗する可能性があります。
