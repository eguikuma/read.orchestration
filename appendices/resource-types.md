---
layout: default
title: Kubernetes リソース一覧
---

# [appendix：Kubernetes リソース一覧](#kubernetes-resources) {#kubernetes-resources}

## [はじめに](#introduction) {#introduction}

このリポジトリでは、7 つのトピック（01-orchestration 〜 07-declarative-management）を通じて、多くの Kubernetes リソースが登場しました

Pod、ReplicaSet、Deployment、Service、HPA など、各リソースはそれぞれ異なるトピックで解説されています

しかし、全リソースを俯瞰して関係性を把握できる一覧はありませんでした

この補足資料では、このリポジトリで学んだ<strong>全リソースをカテゴリ別に整理</strong>し、リソース間の関係と各リソースの基本情報をまとめます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

- <strong>ワークロード管理のリソース</strong>
  - Pod、ReplicaSet、Deployment の階層構造
- <strong>サービスとネットワーキングのリソース</strong>
  - Service、EndpointSlice による安定した通信の仕組み
- <strong>スケーリングのリソース</strong>
  - HPA、VPA による自動スケーリング
- <strong>構成管理のリソース</strong>
  - Namespace、ResourceQuota、Label、Annotation によるリソースの整理
- <strong>ノード管理のリソース</strong>
  - Node、Lease によるノードの状態管理
- <strong>リソース間の関係</strong>
  - 各リソースがどのように連携してオーケストレーションを実現するか

---

## [目次](#table-of-contents) {#table-of-contents}

1. [ワークロード管理](#workload-management)
2. [サービスとネットワーキング](#service-and-networking)
3. [スケーリング](#scaling)
4. [構成管理](#configuration-management)
5. [ノード管理](#node-management)
6. [リソース間の関係](#resource-relationships)
7. [クイックリファレンス](#quick-reference)
8. [用語集](#glossary)
9. [参考資料](#references)

---

## [ワークロード管理](#workload-management) {#workload-management}

ワークロード管理のリソースは、コンテナの実行とライフサイクルを管理します

### [Pod](#pod) {#pod}

<strong>Pod</strong> は、Kubernetes における<strong>コンテナの実行単位</strong>です

1 つ以上のコンテナを含み、スケジューリングの最小単位として機能します

Pod 内のコンテナは、同じネットワーク空間とストレージを共有します

{: .labeled}
| 項目 | 内容 |
| -------------------- | ---------------------------------------- |
| 学んだトピック | [02-architecture](../../02-architecture/) |
| 役割 | コンテナを実行する最小単位 |
| 管理するコントローラ | ReplicaSet、Deployment（間接的） |

### [ReplicaSet](#replica-set) {#replica-set}

<strong>ReplicaSet</strong> は、<strong>指定された数の Pod レプリカを維持する</strong>リソースです

Pod の数があるべき状態より少なければ新しい Pod を作成し、多ければ余分な Pod を削除します

{: .labeled}
| 項目 | 内容 |
| -------------------- | ---------------------------------------------------------------------------------- |
| 学んだトピック | [02-architecture](../../02-architecture/)、[05-self-healing](../../05-self-healing/) |
| 役割 | Pod のレプリカ数を維持する |
| 管理するコントローラ | ReplicaSet コントローラ（Controller Manager 内） |

### [Deployment](#deployment) {#deployment}

<strong>Deployment</strong> は、<strong>ReplicaSet を管理し、アプリケーションの更新とロールバックを制御する</strong>上位リソースです

ローリングアップデートによるダウンタイムなしの更新と、リビジョン管理によるロールバックを実現します

{: .labeled}
| 項目 | 内容 |
| -------------------- | --------------------------------------------------------------- |
| 学んだトピック | [07-declarative-management](../../07-declarative-management/) |
| 役割 | ReplicaSet を管理し、アプリケーションのライフサイクルを制御する |
| 管理するコントローラ | Deployment コントローラ（Controller Manager 内） |

### [階層構造](#hierarchy) {#hierarchy}

```
Deployment
  │  アプリケーションの更新とロールバックを管理する
  │
  ▼
ReplicaSet
  │  指定されたレプリカ数の Pod を維持する
  │
  ▼
Pod
     コンテナを実行する最小単位
```

管理者は通常、Deployment を作成します

Deployment が ReplicaSet を自動的に作成し、ReplicaSet が Pod を自動的に作成します

---

## [サービスとネットワーキング](#service-and-networking) {#service-and-networking}

サービスとネットワーキングのリソースは、Pod 間の通信と外部からのアクセスを管理します

### [Service](#service) {#service}

<strong>Service</strong> は、<strong>Pod の集合に対する安定したアクセス先を提供する</strong>抽象化です

Pod が再作成されて IP アドレスが変わっても、Service の ClusterIP は変わりません

ラベルセレクタで対象の Pod を選択し、トラフィックを振り分けます

{: .labeled}
| 項目 | 内容 |
| -------------- | -------------------------------------------------- |
| 学んだトピック | [04-service-discovery](../../04-service-discovery/) |
| 種類 | ClusterIP、NodePort、LoadBalancer |
| 役割 | Pod への安定したアクセス先と負荷分散を提供する |

### [Headless Service](#headless-service) {#headless-service}

<strong>Headless Service</strong> は、ClusterIP を持たない特殊な Service です

DNS 問い合わせに対して、ClusterIP の代わりに背後の個々の Pod の IP アドレスを返します

{: .labeled}
| 項目 | 内容 |
| -------------- | --------------------------------------------------- |
| 学んだトピック | [04-service-discovery](../../04-service-discovery/) |
| 設定方法 | ClusterIP を `None` に設定 |
| 役割 | クライアントが個々の Pod に直接接続できるようにする |

### [EndpointSlice](#endpoint-slice) {#endpoint-slice}

<strong>EndpointSlice</strong> は、<strong>Service のラベルセレクタに一致する Pod の IP アドレスとポート番号のリスト</strong>です

Pod の作成・削除に応じて自動的に更新されます

{: .labeled}
| 項目 | 内容 |
| -------------- | --------------------------------------------------- |
| 学んだトピック | [04-service-discovery](../../04-service-discovery/) |
| 役割 | Service がトラフィックを転送する先の Pod を追跡する |
| 更新タイミング | Pod の作成、削除、ヘルスチェック結果の変化 |

---

## [スケーリング](#scaling) {#scaling}

スケーリングのリソースは、負荷に応じた Pod の自動増減を管理します

### [HPA（HorizontalPodAutoscaler）](#hpa) {#hpa}

<strong>HPA</strong> は、<strong>メトリクスに基づいて Pod のレプリカ数を自動で調整する</strong>コントローラです

CPU 使用率などのメトリクスを監視し、目標値との差に基づいてレプリカ数を計算します

{: .labeled}
| 項目 | 内容 |
| -------------- | -------------------------------------------- |
| 学んだトピック | [06-scaling](../../06-scaling/) |
| 役割 | メトリクスに基づいてレプリカ数を自動調整する |
| 対象 | Deployment（または ReplicaSet） |

### [VPA（VerticalPodAutoscaler）](#vpa) {#vpa}

<strong>VPA</strong> は、<strong>Pod のリソース要求（CPU やメモリ）を自動で調整する</strong>仕組みです

Kubernetes のコア機能ではなく、追加コンポーネントとして導入します

{: .labeled}
| 項目 | 内容 |
| -------------- | ------------------------------------------------ |
| 学んだトピック | [06-scaling](../../06-scaling/)（対比として紹介） |
| 役割 | Pod のリソース割り当てを自動調整する |
| 導入 | 追加コンポーネントとして別途インストールが必要 |

---

## [構成管理](#configuration-management) {#configuration-management}

構成管理のリソースは、クラスタ内のリソースの整理と分類を管理します

### [Namespace](#namespace) {#namespace}

<strong>Namespace</strong> は、<strong>クラスタ内のリソースを論理的にグループ化する</strong>仕組みです

同じ名前のリソースが異なる Namespace に存在でき、環境やチームごとにリソースを分離できます

{: .labeled}
| 項目 | 内容 |
| -------------- | ------------------------------------------------------------ |
| 学んだトピック | [07-declarative-management](../../07-declarative-management/) |
| 役割 | リソースの論理的なグループ化と分離 |
| 例 | development、staging、production |

### [ResourceQuota](#resource-quota) {#resource-quota}

<strong>ResourceQuota</strong> は、<strong>Namespace 内で使用できるリソースの上限を定義する</strong>仕組みです

1 つの Namespace がクラスタのリソースを独占することを防ぎます

{: .labeled}
| 項目 | 内容 |
| -------------- | ------------------------------------------------------------ |
| 学んだトピック | [07-declarative-management](../../07-declarative-management/) |
| 役割 | Namespace 内のリソース使用量に上限を設ける |
| 制限対象 | CPU、メモリ、Pod 数、Service 数など |

### [Label](#label) {#label}

<strong>Label</strong> は、<strong>リソースに付与するキーと値のペア</strong>です

リソースの選択・分類・フィルタリングに使用されます

{: .labeled}
| 項目 | 内容 |
| -------------- | ------------------------------------------------------------------- |
| 学んだトピック | [07-declarative-management](../../07-declarative-management/) |
| 用途 | Service のラベルセレクタ、ReplicaSet の Pod 識別、nodeSelector など |
| 特徴 | リソースの選択に使用できる |

### [Annotation](#annotation) {#annotation}

<strong>Annotation</strong> は、<strong>リソースに対する補足情報を付与するキーと値のペア</strong>です

Label とは異なり、リソースの選択やフィルタリングには使用されません

{: .labeled}
| 項目 | 内容 |
| -------------- | ------------------------------------------------------------ |
| 学んだトピック | [07-declarative-management](../../07-declarative-management/) |
| 用途 | 説明、管理者情報、外部ツールの設定など |
| 特徴 | リソースの選択には使用できない |

---

## [ノード管理](#node-management) {#node-management}

ノード管理のリソースは、クラスタを構成するマシンの状態を管理します

### [Node](#node) {#node}

<strong>Node</strong> は、<strong>クラスタを構成する個々のマシン</strong>を表すリソースです

Node リソースには、ノードの状態（CPU やメモリの空き、正常かどうか）が記録されています

{: .labeled}
| 項目 | 内容 |
| -------------------- | ---------------------------------------------------------------------------------- |
| 学んだトピック | [02-architecture](../../02-architecture/)、[05-self-healing](../../05-self-healing/) |
| 役割 | ノードの状態を表現する |
| 管理するコントローラ | Node コントローラ（Controller Manager 内） |

### [Lease](#lease) {#lease}

<strong>Lease</strong> は、<strong>ノードの可用性を効率的に確認するための軽量なリソース</strong>です

各ノードの kubelet が `kube-node-lease` Namespace に Lease オブジェクトを作成し、定期的に更新します

Lease の更新が途絶えると、Node コントローラがノードの異常を検知します

{: .labeled}
| 項目 | 内容 |
| -------------- | ---------------------------------------------- |
| 学んだトピック | [05-self-healing](../../05-self-healing/) |
| 役割 | ノードのハートビート（生存報告）として機能する |
| 格納先 | kube-node-lease Namespace |

---

## [リソース間の関係](#resource-relationships) {#resource-relationships}

### [ワークロードの管理チェーン](#workload-management-chain) {#workload-management-chain}

```
Deployment
  │
  │ Deployment コントローラが管理
  ▼
ReplicaSet
  │
  │ ReplicaSet コントローラが管理
  ▼
Pod ────────────────────────────── Node
  │    Scheduler が配置先を決定       │
  │    kubelet が Pod を起動          │
  │                                  │
  │                                Lease
  │                     kubelet がハートビートを送信
  │
  ▼
EndpointSlice ← Service
  │    ラベルセレクタで Pod を追跡
  │
  ▼
kube-proxy
     トラフィックを Pod に転送
```

### [スケーリングとセルフヒーリングの連携](#scaling-and-self-healing) {#scaling-and-self-healing}

```
HPA
  │
  │ メトリクスに基づいてレプリカ数を変更
  ▼
Deployment（レプリカ数の変更）
  │
  │ Reconciliation Loop
  ▼
ReplicaSet → Pod の増減
  │
  │ セルフヒーリング
  ▼
Pod の障害検知 → 自動復旧
```

HPA があるべき状態を動的に変更し、Reconciliation Loop がその状態を実現し、セルフヒーリングがその状態を維持します

### [通信の制御](#communication-control) {#communication-control}

```
Pod A ──→ Service ──→ EndpointSlice ──→ Pod B
  │                                       │
  │         Network Policy                │
  │    通信の許可 / 拒否を制御             │
  └───────────────────────────────────────┘
```

---

## [クイックリファレンス](#quick-reference) {#quick-reference}

### [各リソースの apiVersion と kind](#api-version-and-kind) {#api-version-and-kind}

{: .labeled}
| リソース | apiVersion | kind | スコープ |
| ----------------------- | ---------------------------- | ----------------------- | --------- |
| Pod | v1 | Pod | Namespace |
| ReplicaSet | apps/v1 | ReplicaSet | Namespace |
| Deployment | apps/v1 | Deployment | Namespace |
| Service | v1 | Service | Namespace |
| EndpointSlice | discovery.k8s.io/v1 | EndpointSlice | Namespace |
| HorizontalPodAutoscaler | autoscaling/v2 | HorizontalPodAutoscaler | Namespace |
| Namespace | v1 | Namespace | Cluster |
| ResourceQuota | v1 | ResourceQuota | Namespace |
| Node | v1 | Node | Cluster |
| Lease | coordination.k8s.io/v1 | Lease | Namespace |
| NetworkPolicy | networking.k8s.io/v1 | NetworkPolicy | Namespace |
| Role | rbac.authorization.k8s.io/v1 | Role | Namespace |
| ClusterRole | rbac.authorization.k8s.io/v1 | ClusterRole | Cluster |
| RoleBinding | rbac.authorization.k8s.io/v1 | RoleBinding | Namespace |
| ClusterRoleBinding | rbac.authorization.k8s.io/v1 | ClusterRoleBinding | Cluster |
| ServiceAccount | v1 | ServiceAccount | Namespace |

### [スコープの区別](#scope-distinction) {#scope-distinction}

<strong>Namespace スコープ</strong>のリソースは、特定の Namespace に属します

同じ名前のリソースが異なる Namespace に存在できます

<strong>Cluster スコープ</strong>のリソースは、クラスタ全体に 1 つだけ存在します

Namespace に属さず、クラスタ全体で一意の名前を持ちます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| -------------------------- | ------------------------------------------------------------------------------------------------------- |
| リソース（Resource） | Kubernetes が管理するオブジェクトの種類<br>Pod、Service、Deployment など、apiVersion と kind で識別される |
| apiVersion | マニフェストで使用する Kubernetes API のバージョン<br>リソースの種類によって異なる |
| kind | マニフェストで指定するリソースの種類 |
| Namespace スコープ | 特定の Namespace に属するリソースのスコープ<br>Namespace 内で名前が一意であればよい |
| Cluster スコープ | クラスタ全体に属するリソースのスコープ<br>クラスタ全体で名前が一意でなければならない |
| コントローラ（Controller） | 特定のリソースのあるべき状態を維持する責任を持つ制御ループ<br>Controller Manager に内包される |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Kubernetes 公式ドキュメント</strong>

- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/){:target="\_blank"}
  - 全リソースの apiVersion、kind、フィールド仕様の公式リファレンス

- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/){:target="\_blank"}
  - Kubernetes API の基本概念（リソース、Namespace スコープ、Cluster スコープ）の公式ドキュメント
