---
layout: default
title: read.orchestration
---

# [read.orchestration](#read-orchestration) {#read-orchestration}

<strong>「コンテナ群はどう協調させるか」</strong>を学びます

---

## [このリポジトリは何のためにあるのか](#repository-purpose) {#repository-purpose}

前のシリーズでは、コンテナの仕組みを学び、Compose で複数のコンテナをまとめて定義・管理する方法を学びました

しかし、Compose は1台のマシンでしか動きません

利用者が増えて1台では処理しきれなくなったとき、コンテナを複数のマシンに分散させる仕組みが Compose にはありません

コンテナが異常終了しても、単純な再起動しかできません

複数のマシンにまたがるコンテナ群を、どうやって管理するのでしょうか？

コンテナが障害で停止したとき、どうやって自動的に復旧するのでしょうか？

「あるべき状態」を宣言するだけで、システムがその状態に向かって自律的に動く仕組みとは何でしょうか？

このリポジトリでは、その<strong>「コンテナオーケストレーションの仕組み」</strong>を学びます

---

## [このリポジトリの特徴](#repository-characteristics) {#repository-characteristics}

### [<strong>「なぜ必要か」の原理を理解する</strong>](#why-necessary-principle) {#why-necessary-principle}

Kubernetes の操作方法（kubectl コマンドや YAML の書き方）ではなく、オーケストレーションが<strong>なぜ必要で、どう機能するか</strong>の原理を学びます

Compose の限界から出発し、複数マシンの管理、セルフヒーリング、スケーリングといった課題がどう解決されるかを仕組みから理解します

### [<strong>「あるべき状態」の考え方を軸に学ぶ</strong>](#desired-state-learning) {#desired-state-learning}

オーケストレーションの中心にある考え方は、<strong>「あるべき状態（Desired State）」を宣言し、システムがその状態を維持し続ける</strong>ことです

この考え方を軸に、スケジューリング、サービスディスカバリ、セルフヒーリング、スケーリングを一貫して理解します

### [<strong>各トピック独立で読める構成</strong>](#independent-topics) {#independent-topics}

7つのトピックはそれぞれ独立して読めるように書かれています

興味のあるトピックから読み始めても、必要な知識はそのトピック内で説明されています

---

## [なぜこれを学ぶのか](#why-learn-this) {#why-learn-this}

### [<strong>1. オーケストレーションの仕組みを理解する</strong>](#understand-orchestration) {#understand-orchestration}

「Pod がどのノードに配置されるのか」「コンテナが停止したとき何が起きるのか」を、スケジューラの選択ロジックからコントローラの自動復旧まで、仕組みから説明できるようになります

Kubernetes を主例として使いますが、焦点はオーケストレーションの原理であり、Kubernetes 固有の操作ではありません

### [<strong>2. シリーズ全体の知識を統合する</strong>](#integrate-series-knowledge) {#integrate-series-knowledge}

オーケストレーションは、カーネル空間の namespace / cgroup、ネットワークの DNS と IP、コンテナの仕組み、セキュリティの原理といった、これまで学んできた知識を統合する分野です

各レイヤの知識がどう組み合わさってコンテナ群の協調を実現するかを理解します

### [<strong>3. アプリケーション開発への橋渡し</strong>](#bridge-to-application-development) {#bridge-to-application-development}

オーケストレーションの知識は、アプリケーション開発に直接つながります

デプロイ戦略、サービス間通信、宣言的なインフラ管理など、アプリケーションを本番環境で動かすために必要な基盤をこのリポジトリで学びます

---

## [このリポジトリで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

{: .labeled}
| 順番 | トピック | 学ぶこと |
| ---- | -------------------------------------------------------- | ------------------------------------------------------ |
| 01 | [orchestration](./01-orchestration/) | なぜ Compose では足りないか、複数マシン管理の課題 |
| 02 | [architecture](./02-architecture/) | コントロールプレーンとノード、「あるべき状態」の仕組み |
| 03 | [scheduling](./03-scheduling/) | Pod の配置、リソース要求と制限、ノード選択の仕組み |
| 04 | [service-discovery](./04-service-discovery/) | コンテナ間の発見、Service と DNS による名前解決 |
| 05 | [self-healing](./05-self-healing/) | あるべき状態と実際の状態の差分検知、自動復旧 |
| 06 | [scaling](./06-scaling/) | 水平スケーリング、メトリクスに基づく自動スケール |
| 07 | [declarative-management](./07-declarative-management/) | マニフェスト、宣言的 vs 命令的、全体の統合 |

---

## [前提知識](#prerequisites) {#prerequisites}

このリポジトリを始める前に、以下の学習を推奨します（必須ではありません）

- <strong>コンテナの学習</strong>
  - コンテナの仕組みと Compose の知識が、オーケストレーションの出発点の理解に直接つながります
  - ただし、このリポジトリでもオーケストレーションの文脈で改めて説明するため、未読でも学習できます
- <strong>ネットワークの学習</strong>
  - DNS や IP アドレスの知識が、サービスディスカバリの理解に役立ちます
  - ただし、このリポジトリでもサービスディスカバリの文脈で改めて説明するため、未読でも学習できます
- <strong>セキュリティの学習</strong>
  - サプライチェーンセキュリティの知識が、宣言的構成管理の理解に役立ちます
  - ただし、このリポジトリでも宣言的構成管理の文脈で改めて説明するため、未読でも学習できます

---

## [既存リポジトリとの関係](#existing-repository-relationship) {#existing-repository-relationship}

このリポジトリは、これまでのシリーズで学んだ知識を「コンテナオーケストレーション」として統合します

{: .labeled}
| 概念 | 前のシリーズでの扱い | このリポジトリでの扱い |
| ------------------ | -------------------------- | ---------------------------------------------------------- |
| Compose | 複数コンテナの定義と管理 | Compose の限界を出発点にオーケストレーションの必要性を理解 |
| namespace / cgroup | プロセス隔離とリソース制限 | Pod の隔離とリソース管理の基盤 |
| DNS / IP | 名前解決とパケット配送 | サービスディスカバリとクラスタネットワークの基盤 |
| OS スケジューラ | CPU 時間の配分 | オーケストレータのスケジューリングとの対比 |
| サプライチェーン | 依存関係とビルドの信頼 | パイプラインの信頼、宣言的管理 |

また、このリポジトリで学ぶ知識は、アプリケーション開発に直接つながります

{: .labeled}
| 概念 | このリポジトリでの扱い | 後続の学習での扱い |
| -------------------- | -------------------------------- | ----------------------------------------------- |
| 宣言的管理 | マニフェストによるインフラの定義 | アプリケーション開発での Infrastructure as Code |
| サービスディスカバリ | クラスタ内のコンテナ間通信 | アプリケーション開発でのマイクロサービス間通信 |
| スケーリング | メトリクスに基づく自動スケール | アプリケーション開発での負荷設計とデプロイ戦略 |

---

## [参考資料](#references) {#references}

このリポジトリの内容は、以下のソースに基づいています

各トピックで使用する個別のドキュメントは、各トピックのドキュメントに記載しています

<strong>Kubernetes</strong>

- [Kubernetes Documentation](https://kubernetes.io/docs/){:target="\_blank"}
  - Kubernetes の公式ドキュメント
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/){:target="\_blank"}
  - Kubernetes API の公式リファレンス
- [Kubernetes Design Proposals Archive](https://github.com/kubernetes/design-proposals-archive){:target="\_blank"}
  - Kubernetes の設計提案アーカイブ

<strong>学術論文</strong>

- "Large-scale cluster management at Google with Borg" (Verma et al., EuroSys 2015)
  - Google のクラスタ管理システム Borg の論文
- "Borg, Omega, and Kubernetes" (Burns et al., ACM Queue 2016)
  - Borg から Kubernetes への設計思想の変遷

<strong>分散合意</strong>

- [etcd Documentation](https://etcd.io/docs/){:target="\_blank"}
  - 分散キーバリューストア etcd の公式ドキュメント
- "In Search of an Understandable Consensus Algorithm" (Ongaro & Ousterhout, 2014)
  - Raft 合意アルゴリズムの論文

<strong>ネットワークとサービスディスカバリ</strong>

- [CoreDNS Documentation](https://coredns.io/manual/toc/){:target="\_blank"}
  - CoreDNS の公式ドキュメント
- [Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/){:target="\_blank"}
  - Kubernetes のネットワークモデル
- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/){:target="\_blank"}
  - Kubernetes Service の仕様

<strong>スケーリング</strong>

- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/){:target="\_blank"}
  - 水平 Pod オートスケーラーの仕様

<strong>コンテナランタイム</strong>

- [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/){:target="\_blank"}
  - コンテナランタイムインターフェースの仕様
- [containerd Documentation](https://containerd.io/docs/){:target="\_blank"}
  - containerd の公式ドキュメント
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec){:target="\_blank"}
  - コンテナランタイムの標準仕様
