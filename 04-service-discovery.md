---
layout: default
title: サービスディスカバリ
---

# [04-service-discovery：サービスディスカバリ](#service-discovery) {#service-discovery}

## [はじめに](#introduction) {#introduction}

前のトピック [03-scheduling](../03-scheduling/) では、Scheduler が Pod をどのノードに配置するかを決定する仕組みを学びました

フィルタリングとスコアリングの 2 段階アルゴリズム、リソース要求による配置判断、ラベルや Node Affinity、Taint と Toleration によるノード選択を確認しました

Pod がノードに配置され、実行されるようになりました

しかし、ここで新たな問題が生まれます

Pod がノードに配置された後、他の Pod はその Pod をどうやって見つけるのでしょうか？

Pod が再作成されて IP アドレスが変わった場合、通信先をどう維持するのでしょうか？

複数のノードに分散した Pod 群に対して、どうやって負荷を分散するのでしょうか？

このトピックでは、<strong>サービスディスカバリ</strong>の仕組みを学びます

Pod 同士がどのように互いを発見し、安定した通信を実現するかを見ていきます

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

サービスディスカバリの考え方を、日常の例えで見てみましょう

<strong>Pod の IP アドレス = 社員の個人携帯番号</strong>

会社の中で、経理部門に連絡を取りたいとします

経理部には 3 人の社員がいて、それぞれ個人の携帯番号を持っています

しかし、社員は異動や退職で入れ替わります

新しい社員が来れば新しい番号になり、社員が辞めれば番号は使えなくなります

もし個人の携帯番号を直接使って連絡していたら、社員が入れ替わるたびに番号を調べ直す必要があります

<strong>Service = 部門の代表電話番号</strong>

代わりに、経理部には「代表電話番号」があります

この番号は社員が入れ替わっても変わりません

代表電話に掛けると、その時点で在籍している社員の誰かにつながります

3 人の社員がいれば、電話はその 3 人に振り分けられます

Kubernetes の Service も同じ仕組みです

Pod が再作成されて IP アドレスが変わっても、Service を通じてアクセスすれば、常に正しい Pod に到達できます

<strong>DNS = 社内電話帳</strong>

代表電話番号を覚える必要もありません

社内電話帳に「経理部」と入力すれば、代表電話番号が自動で調べられます

Kubernetes では、DNS がこの電話帳の役割を果たします

Pod は Service の名前（たとえば `database`）を指定するだけで、DNS が自動的に Service の IP アドレスを返します

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>Pod の通信と問題</strong>

- <strong>Pod ごとの IP アドレス</strong>
  - すべての Pod が固有の IP アドレスを持つ Kubernetes のネットワークモデル
- <strong>IP アドレスの不安定性</strong>
  - Pod の再作成で IP アドレスが変わる問題

<strong>Service による安定したアクセス</strong>

- <strong>Service の仕組み</strong>
  - 安定した仮想 IP アドレス（ClusterIP）を提供する抽象化
- <strong>EndpointSlice</strong>
  - Service がどの Pod に通信を転送するかを追跡する仕組み
- <strong>kube-proxy</strong>
  - ノード上で Service へのトラフィックを実際の Pod に転送する仕組み

<strong>DNS によるサービスディスカバリ</strong>

- <strong>クラスタ内 DNS（CoreDNS）</strong>
  - Service の名前から IP アドレスを解決する仕組み
- <strong>DNS の命名規則</strong>
  - Service にアクセスするための名前の構造

<strong>Service の種類</strong>

- <strong>ClusterIP、NodePort、LoadBalancer</strong>
  - クラスタ内部アクセスから外部公開までの段階的な仕組み

---

## [目次](#table-of-contents) {#table-of-contents}

1. [Pod のネットワークモデル](#pod-network-model)
2. [IP アドレスの不安定性](#ip-address-instability)
3. [Service（安定したアクセス先）](#service)
4. [EndpointSlice（Pod の追跡）](#endpoint-slice)
5. [kube-proxy（トラフィックの転送）](#kube-proxy-traffic)
6. [DNS によるサービスディスカバリ](#dns-based-service-discovery)
7. [CoreDNS（クラスタの DNS サーバー）](#coredns)
8. [Service の種類](#service-types)
9. [サービスディスカバリの全体像](#service-discovery-overview)
10. [次のトピックへ](#next-topic)
11. [用語集](#glossary)
12. [参考資料](#references)

---

## [Pod のネットワークモデル](#pod-network-model) {#pod-network-model}

サービスディスカバリを理解するために、まず Kubernetes のネットワークモデルを確認します

### [Pod ごとの IP アドレス](#pod-ip-address) {#pod-ip-address}

Kubernetes では、<strong>すべての Pod が固有の IP アドレスを持ちます</strong>

これは Kubernetes のネットワークモデルの基本原則です

従来のコンテナネットワークでは、コンテナがホストマシンの IP アドレスとポートを共有し、ポートの競合を管理する必要がありました

Kubernetes はこの問題を、Pod ごとに独立した IP アドレスを割り当てることで解決しています

### [このモデルの利点](#pod-network-model-benefits) {#pod-network-model-benefits}

Pod ごとに IP アドレスを持つことで、以下の利点があります

{: .labeled}
| 利点 | 説明 |
| ------------------------ | -------------------------------------------------------------------------- |
| ポートの競合がない | 各 Pod が独立した IP を持つため、同じポート番号を使っても競合しない |
| アプリケーションの簡素化 | ポートの動的割り当てや管理をアプリケーション側で行う必要がない |
| 予測可能なアドレス体系 | Pod、Service、ノードがそれぞれ別の IP 範囲を持ち、アドレス体系が整理される |

前のシリーズでコンテナのネットワーキングを学んだ方は、bridge や veth といった仕組みを思い出すかもしれません

Kubernetes のネットワークは、これらのコンテナネットワーク技術を基盤として、クラスタ全体に拡張したものです

### [Kubernetes のネットワーク要件](#kubernetes-network-requirements) {#kubernetes-network-requirements}

Kubernetes のネットワークモデルには、以下の要件があります

- すべての Pod は、NAT（ネットワークアドレス変換）なしで、他のすべての Pod と通信できる
- すべてのノードは、NAT なしで、すべての Pod と通信できる
- Pod が自分自身の IP アドレスとして認識する IP は、他の Pod から見ても同じ IP である

これらの要件を満たすことで、Pod 同士は IP アドレスさえ知っていれば直接通信できます

しかし、ここに問題があります

---

## [IP アドレスの不安定性](#ip-address-instability) {#ip-address-instability}

Pod の IP アドレスには、大きな問題があります

<strong>Pod は再作成されるたびに新しい IP アドレスが割り当てられます</strong>

### [なぜ IP アドレスが変わるのか](#why-ip-changes) {#why-ip-changes}

Pod はさまざまな理由で再作成されます

{: .labeled}
| 理由 | 状況 |
| ---------------------- | ---------------------------------------------------------- |
| ノードの障害 | ノードがダウンし、Pod が別のノードで再作成される |
| スケーリング | Pod の数が増減し、新しい Pod が作成される |
| アプリケーションの更新 | ローリングアップデートで古い Pod が新しい Pod に置き換わる |
| ヘルスチェックの失敗 | Pod が異常と判断され、再起動または再作成される |

再作成された Pod には新しい IP アドレスが割り当てられます

元の IP アドレスは使われなくなり、新しい IP アドレスは以前とは異なります

### [直接通信の問題](#direct-communication-problems) {#direct-communication-problems}

もし Pod A が Pod B の IP アドレスを直接使って通信していたらどうなるでしょうか

Pod B が再作成されると IP アドレスが変わり、Pod A の通信は失敗します

Pod A は新しい IP アドレスを知る方法がありません

```
Pod A ──→ 10.1.2.3（Pod B の旧 IP）──→ 通信失敗
                                         Pod B は 10.1.2.7 に変わった
```

さらに、Pod が複数のレプリカで動いている場合、どの Pod に通信を送るかも問題になります

3 つの Web サーバー Pod があるとき、通信をどう振り分けるのでしょうか

これらの問題を解決するのが、<strong>Service</strong>です

---

## [Service（安定したアクセス先）](#service) {#service}

<strong>Service</strong>は、<strong>Pod の集合に対する安定したアクセス先を提供する抽象化</strong>です

日常の例えで言えば、部門の代表電話番号に相当します

### [ClusterIP](#cluster-ip) {#cluster-ip}

Service を作成すると、<strong>ClusterIP</strong>と呼ばれる仮想 IP アドレスが割り当てられます

この IP アドレスは、Service が存在する限り変わりません

Pod が再作成されて IP アドレスが変わっても、Service の ClusterIP は同じままです

```
Pod A ──→ 10.96.0.5（Service の ClusterIP）──→ Pod B（10.1.2.7）
                                                 Pod C（10.1.3.2）
                                                 Pod D（10.1.4.5）
```

Pod A は Service の ClusterIP に通信するだけで、背後にいる Pod B、C、D のいずれかに到達できます

Pod が再作成されても、Service の ClusterIP は変わらないため、Pod A は何も変更する必要がありません

### [ラベルセレクタによる Pod の選択](#label-selector-pod-selection) {#label-selector-pod-selection}

Service は、どの Pod に通信を転送するかを<strong>ラベルセレクタ</strong>で決めます

ラベルセレクタとは、Pod に付与されたラベルに基づいて Pod を選択する仕組みです

たとえば、以下のマニフェストでは `app: web` というラベルを持つすべての Pod を対象とする Service を定義しています

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

`app: web` というラベルを持つ Pod が 3 つあれば、Service はその 3 つの Pod にトラフィックを転送します

Pod が追加されれば自動的に転送先に加わり、Pod が削除されれば自動的に転送先から外れます

### [あるべき状態と Service](#desired-state-and-service) {#desired-state-and-service}

Service もまた、あるべき状態の一部です

「`app: web` ラベルを持つ Pod への安定したアクセス先を、ClusterIP 10.96.0.5 で提供する」という状態が宣言されています

この宣言に基づいて、Kubernetes は Pod の変動に関わらず、安定したアクセスを維持し続けます

---

## [EndpointSlice（Pod の追跡）](#endpoint-slice) {#endpoint-slice}

Service がどの Pod にトラフィックを転送するかを決めるには、「今どの Pod が動いているか」を常に把握する必要があります

この追跡を担うのが、<strong>EndpointSlice</strong>です

### [EndpointSlice の役割](#endpoint-slice-role) {#endpoint-slice-role}

<strong>EndpointSlice</strong>は、Service のラベルセレクタに一致する<strong>Pod の IP アドレスとポート番号のリスト</strong>です

Kubernetes のコントロールプレーンは、Pod の作成・削除・状態変化を監視し、EndpointSlice を自動的に更新します

```
Service（web-service）
  │
  │ ラベルセレクタ：app: web
  │
  ▼
EndpointSlice
  ├── 10.1.2.7:8080（Pod B、ノード 1）
  ├── 10.1.3.2:8080（Pod C、ノード 2）
  └── 10.1.4.5:8080（Pod D、ノード 3）
```

### [自動更新](#auto-update) {#auto-update}

EndpointSlice は、Pod の状態に応じて自動的に更新されます

<strong>Pod が追加された場合</strong>

新しい Pod が作成され、ラベルセレクタに一致すれば、その Pod の IP アドレスが EndpointSlice に追加されます

<strong>Pod が削除された場合</strong>

Pod が削除されると、その Pod の IP アドレスが EndpointSlice から削除されます

<strong>Pod が異常な場合</strong>

ヘルスチェック（Readiness Probe）に失敗した Pod は、EndpointSlice から一時的に除外されます

異常な Pod にはトラフィックが転送されません

Pod が回復すれば、再び EndpointSlice に追加されます

この自動追跡が、Service による安定したアクセスを支えています

---

## [kube-proxy（トラフィックの転送）](#kube-proxy-traffic) {#kube-proxy-traffic}

Service の ClusterIP は仮想的な IP アドレスであり、特定のネットワークインターフェースに紐付いていません

では、ClusterIP 宛てのトラフィックは、どうやって実際の Pod に届くのでしょうか

これを実現するのが、<strong>kube-proxy</strong>です

### [kube-proxy の役割](#kube-proxy-role) {#kube-proxy-role}

前のトピック [02-architecture](../02-architecture/) で、kube-proxy は各ノード上で動作し、ネットワークルールを管理するコンポーネントであると学びました

kube-proxy は、<strong>Service の ClusterIP 宛てのトラフィックを、EndpointSlice に記録されている実際の Pod に転送する</strong>役割を担います

### [トラフィック転送の仕組み](#traffic-forwarding-mechanism) {#traffic-forwarding-mechanism}

kube-proxy は、以下の流れでトラフィックを転送します

1. API Server を監視し、Service と EndpointSlice の変更を検知する
2. 各ノード上にネットワークルールを設定する
3. Service の ClusterIP 宛てのトラフィックを捕捉する
4. EndpointSlice に記録されている Pod の中から 1 つを選ぶ
5. トラフィックを選ばれた Pod に転送する

### [負荷分散](#load-balancing) {#load-balancing}

Service の背後に複数の Pod がある場合、kube-proxy は<strong>トラフィックを複数の Pod に振り分けます</strong>

これにより、1 つの Pod にトラフィックが集中することを避け、負荷を分散できます

これが、前のトピックの疑問「複数のノードに分散した Pod 群に対して、どうやって負荷を分散するのか」への回答です

Service と kube-proxy が連携して、トラフィックを自動的に振り分けます

---

## [DNS によるサービスディスカバリ](#dns-based-service-discovery) {#dns-based-service-discovery}

ここまでで、Service が安定した ClusterIP を提供し、kube-proxy がトラフィックを転送することを学びました

しかし、Pod が Service にアクセスするとき、ClusterIP の数値（たとえば `10.96.0.5`）を直接指定するのは不便です

IP アドレスは覚えにくく、Service が再作成されれば ClusterIP も変わる可能性があります

ここで登場するのが、<strong>DNS（Domain Name System）</strong>です

### [DNS による名前解決](#dns-name-resolution) {#dns-name-resolution}

Kubernetes クラスタ内には DNS サーバーが動作しており、すべての Service に対して<strong>DNS レコード</strong>が自動的に作成されます

Pod は Service の名前を指定するだけで、DNS が自動的に ClusterIP を返します

```
Pod A が「database」にアクセスしたい
  │
  ▼
DNS に問い合わせ：「database」の IP アドレスは？
  │
  ▼
DNS が回答：10.96.0.5（Service の ClusterIP）
  │
  ▼
Pod A が 10.96.0.5 に接続
  │
  ▼
kube-proxy が実際の Pod に転送
```

前のシリーズでネットワークの仕組みを学んだ方は、DNS による名前解決を思い出すかもしれません

インターネットでドメイン名から IP アドレスを解決するのと同じ仕組みが、クラスタ内でも使われています

### [DNS の命名規則](#dns-naming-convention) {#dns-naming-convention}

Service の DNS 名は、以下の規則で自動的に作成されます

```
<Service 名>.<Namespace 名>.svc.cluster.local
```

たとえば、`default` Namespace にある `database` という Service の完全な DNS 名は以下になります

```
database.default.svc.cluster.local
```

### [短い名前でのアクセス](#short-name-access) {#short-name-access}

Pod は通常、完全な DNS 名を指定する必要はありません

<strong>同じ Namespace 内の Service</strong>

Service 名だけでアクセスできます

```
database
```

<strong>異なる Namespace の Service</strong>

Service 名と Namespace 名を指定します

```
database.production
```

これは、Pod の DNS 設定にサーチリストが自動的に構成されているためです

Pod が `database` という名前を問い合わせると、DNS は自動的に `database.<Pod の Namespace>.svc.cluster.local` に展開して解決します

### [Headless Service](#headless-service) {#headless-service}

通常の Service は ClusterIP を持ち、DNS はその ClusterIP を返します

しかし、特定の用途では、背後の個々の Pod の IP アドレスを直接知りたい場合があります

<strong>Headless Service</strong>は、ClusterIP を持たない特殊な Service です

ClusterIP を `None` に設定することで作成します

Headless Service に対する DNS 問い合わせは、ClusterIP の代わりに、<strong>背後の Pod の IP アドレスのリスト</strong>を返します

これにより、クライアントが個々の Pod に直接接続できます

---

## [CoreDNS（クラスタの DNS サーバー）](#coredns) {#coredns}

Kubernetes クラスタ内で DNS の名前解決を担当するのが、<strong>CoreDNS</strong>です

### [CoreDNS とは](#what-is-coredns) {#what-is-coredns}

<strong>CoreDNS</strong>は、<strong>プラグインベースの DNS サーバー</strong>です

CoreDNS 自体はシンプルな基盤であり、ほぼすべての機能がプラグインとして実装されています

Kubernetes のサービスディスカバリに関しては、Kubernetes プラグインが担当しています

### [CoreDNS の動作](#coredns-operation) {#coredns-operation}

CoreDNS は、以下のように動作します

<strong>Kubernetes API の監視</strong>

CoreDNS は API Server を監視し、Service と Pod の作成・削除・変更を検知します

新しい Service が作成されると、CoreDNS はその Service に対応する DNS レコードを自動的に作成します

<strong>DNS 問い合わせの処理</strong>

Pod から DNS 問い合わせを受け取ると、CoreDNS はプラグインチェーン（複数のプラグインを順番に通す処理）を通じてレコードを解決し、結果を返します

### [kubelet による DNS 設定](#kubelet-dns-configuration) {#kubelet-dns-configuration}

Pod 内の DNS 設定は、ノード上の kubelet が自動的に構成します

kubelet は、Pod の `/etc/resolv.conf` に以下のような設定を書き込みます

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

`nameserver` は CoreDNS の IP アドレスを指し、`search` はサーチリストを定義しています

これにより、Pod は Service 名だけで DNS 問い合わせができるようになります

### [前のシリーズとの接続](#previous-series-connection) {#previous-series-connection}

前のシリーズでネットワークの仕組みを学んだ方は、DNS の名前解決プロセスを思い出すかもしれません

インターネットの DNS では、クライアントがリゾルバに問い合わせ、リゾルバがルートサーバーから順にたどって IP アドレスを解決します

Kubernetes の DNS も同じ原理ですが、解決の対象がインターネット上のドメインではなく、クラスタ内の Service と Pod です

CoreDNS はクラスタのローカルな DNS サーバーとして、クラスタ内の名前解決を完結させます

---

## [Service の種類](#service-types) {#service-types}

Service には、アクセス方法の異なる複数の種類があります

### [ClusterIP（クラスタ内部アクセス）](#cluster-ip-internal) {#cluster-ip-internal}

<strong>ClusterIP</strong>は、Service のデフォルトの種類です

クラスタ内部からのみアクセスできる仮想 IP アドレスを提供します

ここまで説明してきた Service は、すべて ClusterIP タイプです

クラスタ内の Pod 同士の通信に使われます

### [NodePort（ノード経由の外部アクセス）](#node-port) {#node-port}

<strong>NodePort</strong>は、ClusterIP に加えて、<strong>すべてのノードの特定のポートで Service を公開する</strong>種類です

クラスタ外部から `<ノードの IP>:<NodePort>` でアクセスできます

```
クラスタ外部 ──→ ノード IP:30080 ──→ Service（ClusterIP）──→ Pod
```

NodePort は 30000〜32767 の範囲から割り当てられます

### [LoadBalancer（外部ロードバランサー経由のアクセス）](#load-balancer) {#load-balancer}

<strong>LoadBalancer</strong>は、NodePort に加えて、<strong>外部ロードバランサーを通じて Service を公開する</strong>種類です

クラウド環境で使用され、外部ロードバランサーが自動的にプロビジョニングされます

```
クラスタ外部 ──→ 外部ロードバランサー ──→ ノード ──→ Service ──→ Pod
```

### [種類の比較](#service-types-comparison) {#service-types-comparison}

{: .labeled}
| 種類 | アクセス元 | 仕組み |
| ------------ | ---------------- | ------------------------------------ |
| ClusterIP | クラスタ内部のみ | 仮想 IP アドレスを割り当てる |
| NodePort | クラスタ外部 | 全ノードの特定ポートで公開する |
| LoadBalancer | クラスタ外部 | 外部ロードバランサーを通じて公開する |

これらの種類は階層的に構成されています

NodePort は内部的に ClusterIP を含み、LoadBalancer は内部的に NodePort を含みます

用途に応じて適切な種類を選択します

---

## [サービスディスカバリの全体像](#service-discovery-overview) {#service-discovery-overview}

ここまで学んだ仕組みを、1 つの流れとしてまとめます

Pod A が `database` という名前の Service にアクセスするシナリオです

```
1. Pod A がアプリケーションコードで「database」に接続する
   │
   ▼
2. Pod A の DNS リゾルバが CoreDNS に問い合わせる
   │  問い合わせ：database.default.svc.cluster.local
   ▼
3. CoreDNS が DNS レコードを返す
   │  回答：10.96.0.5（Service の ClusterIP）
   ▼
4. Pod A が 10.96.0.5 にトラフィックを送信する
   │
   ▼
5. kube-proxy がトラフィックを捕捉する
   │  EndpointSlice を参照し、転送先を選択する
   ▼
6. 実際の Pod に転送される
   │  10.1.2.7:5432（Pod B）に転送
   ▼
7. Pod B がリクエストを処理して応答する
```

### [各コンポーネントの役割](#component-roles) {#component-roles}

{: .labeled}
| コンポーネント | 役割 |
| -------------- | -------------------------------------------------------------- |
| DNS（CoreDNS） | Service 名を ClusterIP に変換する |
| Service | 安定した ClusterIP と、ラベルセレクタによる Pod 選択を提供する |
| EndpointSlice | Service に紐づく Pod の IP アドレスリストを維持する |
| kube-proxy | ClusterIP 宛てのトラフィックを実際の Pod に転送する |

### [あるべき状態との関係](#desired-state-relationship) {#desired-state-relationship}

サービスディスカバリは、あるべき状態を支える重要な仕組みです

Pod が再作成されて IP アドレスが変わっても、Service は安定したアクセスを維持します

これにより、Pod の入れ替わりがアプリケーションに影響しません

「あるべき状態」として「Web サーバーの Pod を 3 つ動かす」と宣言し、Pod の 1 つが障害で再作成されたとしても、Service を通じたアクセスは途切れません

EndpointSlice が自動的に更新され、新しい Pod への転送が始まるためです

スケジューリング（配置）とサービスディスカバリ（発見）が連携することで、あるべき状態の実現と維持が可能になります

---

## [次のトピックへ](#next-topic) {#next-topic}

このトピックでは、以下のことを学びました

- Kubernetes のネットワークモデルでは、すべての Pod が固有の IP アドレスを持つが、Pod の再作成で IP アドレスは変わる
- Service が安定した ClusterIP を提供し、Pod が入れ替わっても安定したアクセスを維持する
- EndpointSlice が Service に紐づく Pod の IP アドレスを自動追跡し、kube-proxy がトラフィックを実際の Pod に転送する
- DNS（CoreDNS）により、Pod は Service の名前だけで通信先を発見できる
- ClusterIP、NodePort、LoadBalancer の 3 種類の Service が、アクセスパターンに応じた公開方法を提供する

Pod の配置（スケジューリング）と発見（サービスディスカバリ）の仕組みが揃いました

ここで、さらなる疑問が生まれます

Pod やノードに障害が発生したとき、システムはどうやって自動的に復旧するのでしょうか？

あるべき状態と実際の状態にずれが生じたとき、それを検知する仕組みはどう動くのでしょうか？

Pod が「正常に動いている」とは、何をもって判断するのでしょうか？

次のトピック [05-self-healing](../05-self-healing/) では、<strong>セルフヒーリング</strong>の仕組みを詳しく学びます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| サービスディスカバリ（Service Discovery） | 分散システムにおいて、サービスの場所（IP アドレスやポート）を動的に発見する仕組み |
| Service | Pod の集合に対する安定したアクセス先を提供する Kubernetes の抽象化<br>ラベルセレクタで対象の Pod を選択する |
| ClusterIP | Service に割り当てられる仮想 IP アドレス<br>クラスタ内部からのみアクセスでき、Service が存在する限り変わらない |
| ラベルセレクタ（Label Selector） | ラベルに基づいて Kubernetes オブジェクト（Pod など）を選択する仕組み<br>Service が対象の Pod を決定するために使用する |
| EndpointSlice | Service のラベルセレクタに一致する Pod の IP アドレスとポート番号のリスト<br>Pod の作成・削除に応じて自動的に更新される |
| kube-proxy | 各ノード上で動作し、Service の ClusterIP 宛てのトラフィックを実際の Pod に転送するコンポーネント |
| DNS（Domain Name System） | 名前から IP アドレスを解決する仕組み<br>Kubernetes クラスタ内では CoreDNS がこの役割を担う |
| CoreDNS | Kubernetes クラスタ内で動作するプラグインベースの DNS サーバー<br>Service と Pod の DNS レコードを自動的に管理する |
| DNS レコード | DNS サーバーが保持する名前と IP アドレスの対応情報<br>Service が作成されると自動的に DNS レコードが作成される |
| サーチリスト（Search List） | DNS 問い合わせ時に、短い名前を完全な DNS 名に展開するためのサフィックスのリスト<br>kubelet が Pod に自動設定する |
| Namespace | Kubernetes のリソースを論理的にグループ化する仕組み<br>同じクラスタ内で複数の環境やチームを分離するために使用する |
| Headless Service | ClusterIP を持たない特殊な Service<br>DNS 問い合わせに対して個々の Pod の IP アドレスを返す |
| NodePort | すべてのノードの特定ポートで Service を公開する種類<br>クラスタ外部からノードの IP とポート番号でアクセスできる |
| LoadBalancer | 外部ロードバランサーを通じて Service を公開する種類<br>クラウド環境で外部ロードバランサーが自動的にプロビジョニングされる |
| NAT（Network Address Translation） | ネットワークアドレス変換<br>IP アドレスを別の IP アドレスに変換する仕組み<br>Kubernetes のネットワークモデルでは Pod 間通信に NAT を使わないことが要件 |
| CNI（Container Network Interface） | Kubernetes がネットワークプラグインと連携するための標準インターフェース<br>Pod への IP アドレス割り当てとネットワーク接続を担当する |
| Readiness Probe | Pod がトラフィックを受け入れる準備ができているかを確認するヘルスチェック<br>失敗した Pod は EndpointSlice から一時的に除外される |
| プラグインチェーン（Plugin Chain） | CoreDNS がDNS 問い合わせを処理する際に、複数のプラグインを順番に通す仕組み |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Kubernetes サービスとネットワーキング</strong>

- [Service](https://kubernetes.io/docs/concepts/services-networking/service/){:target="\_blank"}
  - Service の概念、ClusterIP、NodePort、LoadBalancer の公式ドキュメント

- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/){:target="\_blank"}
  - クラスタ内の DNS の命名規則と動作の公式ドキュメント

- [Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/){:target="\_blank"}
  - Kubernetes のネットワークモデルの公式ドキュメント

<strong>CoreDNS</strong>

- [CoreDNS Manual](https://coredns.io/manual/toc/){:target="\_blank"}
  - CoreDNS の公式マニュアル

<strong>EndpointSlice</strong>

- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/){:target="\_blank"}
  - EndpointSlice の仕組みの公式ドキュメント
