---
layout: default
title: コンテナオーケストレーションの歴史
---

# [appendix：コンテナオーケストレーションの歴史](#orchestration-history) {#orchestration-history}

## [はじめに](#introduction) {#introduction}

[01-orchestration](../../01-orchestration/) では、コンテナオーケストレーションの必要性と、Google の Borg から Kubernetes が生まれた背景を学びました

そのなかで、Kubernetes が事実上の標準であることには触れましたが、Kubernetes 以外のオーケストレーション技術については詳しく扱いませんでした

Docker Swarm、Apache Mesos、HashiCorp Nomad など、Kubernetes と並行して登場したオーケストレーション技術が存在します

前のシリーズでは、コンテナ技術そのものの歴史（chroot から Docker、OCI 標準化まで）を辿りました

この補足資料では、その上のレイヤである<strong>オーケストレーション技術の歴史</strong>を辿ります

各技術がどのような設計思想で作られ、なぜ Kubernetes が標準の地位を確立したのかを、比較の視点から理解することがこの資料の目的です

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

- コンテナオーケストレーション技術の年表と全体の流れ
- Google Borg と Omega の設計思想、および Kubernetes への影響
- Apache Mesos の二段階スケジューリングモデル
- Docker Swarm の設計哲学とその変遷
- HashiCorp Nomad の特徴と位置づけ
- 複数のオーケストレーション技術の比較を通じた、Kubernetes が標準となった理由の理解

---

## [目次](#table-of-contents) {#table-of-contents}

1. [年表](#timeline)
2. [Google Borg と Omega](#google-borg-and-omega)
3. [Apache Mesos](#apache-mesos)
4. [Docker Swarm](#docker-swarm)
5. [HashiCorp Nomad](#hashicorp-nomad)
6. [なぜ Kubernetes が標準となったか](#why-kubernetes-standard)
7. [オーケストレーション技術の進化の流れ](#orchestration-evolution)
8. [用語集](#glossary)
9. [参考資料](#references)

---

## [年表](#timeline) {#timeline}

{: .labeled}
| 年 | 技術 / プロジェクト | 概要 |
| ------- | ------------------- | ------------------------------------------------------------------------------------ |
| 2003 頃 | Google Borg | Google 社内の大規模クラスタ管理システム。数万台のマシンでサービスを運用 |
| 2009 | Apache Mesos | UC Berkeley で開発。二段階スケジューリングによるデータセンター規模のリソース共有基盤 |
| 2013 | Docker | コンテナプラットフォーム。コンテナの普及により、オーケストレーションの必要性が高まる |
| 2014 | Kubernetes | Google が Borg の経験を活かしてオープンソースとして公開（初期リリース） |
| 2014 | Docker Swarm | Docker 社によるスタンドアロン型のオーケストレーションツール（初期版） |
| 2015 | CNCF 設立 | Cloud Native Computing Foundation の設立。Kubernetes 1.0 が CNCF に移管される |
| 2015 | Marathon on Mesos | Mesos 上でコンテナの長時間稼働サービスを管理するフレームワーク |
| 2015 | HashiCorp Nomad | コンテナに限定しないワークロードスケジューラ。シンプルさを設計目標とする |
| 2016 | Docker Swarm mode | Docker Engine に Swarm 機能を統合（SwarmKit）。docker service create で操作可能に |
| 2017 | Docker の K8s 対応 | Docker Desktop が Kubernetes をサポート。Docker 社自身が Kubernetes を取り込む |
| 2018 | Kubernetes の標準化 | AWS（EKS）、GCP（GKE）、Azure（AKS）が Kubernetes のマネージドサービスを提供 |

---

## [Google Borg と Omega](#google-borg-and-omega) {#google-borg-and-omega}

[01-orchestration](../../01-orchestration/) では、Borg から Kubernetes への流れを概観しました

このセクションでは、01-orchestration では扱わなかった Borg の詳細と、Borg と Kubernetes の間に存在した研究プロジェクト <strong>Omega</strong> について掘り下げます

### [Borg の設計](#borg-design) {#borg-design}

<strong>Borg</strong> は、Google が社内で 10 年以上にわたり運用してきた大規模クラスタ管理システムです

Verma et al.（EuroSys 2015）の論文によると、Borg は数万台のマシンで構成されるクラスタ（<strong>セル</strong>と呼ばれる単位）を管理し、Google の検索エンジン、Gmail、YouTube などの主要サービスを動かしていました

Borg の設計には、いくつかの重要なアイデアが含まれています

<strong>宣言的な構成（Declarative Configuration）</strong>

管理者は「このジョブを N 個のインスタンスで動かしたい」と宣言します

Borg はその宣言に基づき、クラスタ内のマシンにインスタンスを自動的に配置します

<strong>調整ループ（Reconciliation Loop）</strong>

Borg は宣言された状態（あるべき状態）と実際の状態を継続的に比較します

インスタンスが障害で停止すれば、Borg が自動的に別のマシンで再作成します

<strong>API 中心の設計</strong>

すべての操作は Borg の API を通じて行われます

これにより、自動化ツールや監視システムとの連携が容易になります

<strong>優先度とプリエンプション</strong>

Borg はジョブに優先度を設定できます

リソースが不足した場合、低優先度のジョブを停止（プリエンプション）して、高優先度のジョブにリソースを割り当てます

### [Omega](#omega) {#omega}

<strong>Omega</strong> は、Google 社内で Borg の後に取り組まれた研究プロジェクトです

Borg のスケジューリングモデルを改良するための実験的なシステムとして開発されました

Borg では、1 つの中央スケジューラがクラスタ全体のスケジューリングを担当していました

この<strong>集中型スケジューラ</strong>は、大規模なクラスタではボトルネックになる可能性がありました

Omega は<strong>共有状態スケジューリング（Shared-state Scheduling）</strong>というモデルを採用しました

このモデルでは、複数のスケジューラが同時に動作し、クラスタの状態を共有します

各スケジューラはクラスタ全体の状態のコピーを保持し、独立してスケジューリング判断を行います

複数のスケジューラが同じリソースに対して競合した場合は、<strong>楽観的並行性制御（Optimistic Concurrency Control）</strong>によって解決します

先にコミットしたスケジューラの決定が採用され、競合した他のスケジューラはリトライします

### [Borg と Omega から Kubernetes への教訓](#borg-omega-kubernetes-lessons) {#borg-omega-kubernetes-lessons}

Burns et al.（ACM Queue 2016）では、Borg と Omega の運用から得られた教訓が Kubernetes の設計にどう活かされたかが解説されています

{: .labeled}
| Borg / Omega の教訓 | Kubernetes の設計への反映 |
| -------------------------------------------------- | ---------------------------------------------------------- |
| 宣言的な構成の方が大規模運用に適している | すべてのリソースを YAML / JSON の宣言で管理する |
| 調整ループが信頼性の鍵である | Controller がループで Desired State と実際の状態を比較する |
| API 中心の設計が自動化を容易にする | すべての操作を RESTful API（API Server）経由で行う |
| 集中型スケジューラにはスケーラビリティの限界がある | プラガブルなスケジューラ設計を採用 |
| ジョブではなくラベルで管理する方が柔軟である | Label と Selector によるリソースの柔軟なグルーピング |

Kubernetes は Borg をそのまま移植したものではなく、Borg と Omega の両方から得た教訓を反映して設計されたオープンソースプロジェクトです

---

## [Apache Mesos](#apache-mesos) {#apache-mesos}

### [背景と設計](#mesos-background) {#mesos-background}

<strong>Apache Mesos</strong> は、2009 年に UC Berkeley で開発されたクラスタ管理システムです

Hindman et al.（NSDI 2011）の論文 "Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center" で発表されました

Mesos の設計目標は、<strong>データセンター全体のリソースを複数のフレームワーク間で効率的に共有する</strong>ことでした

ここでいうフレームワークとは、Hadoop（バッチ処理）、Spark（データ分析）、Marathon（長時間稼働サービス）など、異なる種類のワークロードを管理するシステムのことです

### [二段階スケジューリング](#two-phase-scheduling) {#two-phase-scheduling}

Mesos の最も特徴的な設計は、<strong>二段階スケジューリング（Two-level Scheduling）</strong>モデルです

<strong>第一段階：Mesos がリソースを提示する</strong>

Mesos マスターは、クラスタ内のノードから利用可能なリソース（CPU、メモリなど）の情報を収集します

収集したリソース情報を<strong>リソースオファー（Resource Offer）</strong>として各フレームワークに提示します

<strong>第二段階：フレームワークがリソースを選択する</strong>

各フレームワークは、提示されたリソースオファーの中から、自分のタスクに必要なリソースを選択します

フレームワーク自身がスケジューリングの判断を行うため、ワークロードに特化した最適な配置が可能です

```
二段階スケジューリングの流れ：

ノード群
  │  リソース情報を報告
  ↓
Mesos マスター
  │  リソースオファーを提示
  ├────────────────┬────────────────┐
  ↓                ↓                ↓
Marathon          Spark             Hadoop
（コンテナ）     （データ分析）    （バッチ処理）
  │                │                │
  ↓                ↓                ↓
必要なリソースを   必要なリソースを   必要なリソースを
選択して使用       選択して使用       選択して使用
```

この設計により、Mesos はコンテナに限定されず、さまざまな種類のワークロードを 1 つのクラスタ上で効率的に動かすことができました

### [Marathon](#marathon) {#marathon}

<strong>Marathon</strong> は、Mesos 上でコンテナの長時間稼働サービスを管理するフレームワークです

Marathon は Mesos の二段階スケジューリングモデルにおけるフレームワークの 1 つとして動作し、コンテナのデプロイ、スケーリング、セルフヒーリングなどの機能を提供していました

Kubernetes における Deployment と似た役割を担っていたと言えます

### [Mesos の強みと課題](#mesos-strengths-and-challenges) {#mesos-strengths-and-challenges}

<strong>強み</strong>

- データセンター規模のリソース共有を目的として設計されており、コンテナだけでなく多様なワークロードに対応できる
- 二段階モデルにより、各フレームワークがワークロードに最適なスケジューリング判断を行える
- Twitter、Apple、Uber など大規模企業での採用実績がある

<strong>課題</strong>

- 二段階スケジューリングモデルは強力だが、理解と運用の複雑さが高い
- コンテナのオーケストレーションだけを行いたい場合、Mesos + Marathon の組み合わせは過剰な構成になりやすい
- Kubernetes がコンテナオーケストレーションに特化した統合的な体験を提供したことで、コンテナ用途では Kubernetes に移行する組織が増えた

---

## [Docker Swarm](#docker-swarm) {#docker-swarm}

### [背景と設計哲学](#swarm-background) {#swarm-background}

<strong>Docker Swarm</strong> は、Docker 社が開発したコンテナオーケストレーションツールです

Docker Swarm の設計哲学は<strong>シンプルさ</strong>でした

Docker CLI に慣れた開発者が、追加の学習コストを最小限に抑えてオーケストレーションを利用できることを目指していました

### [二つのフェーズ](#two-phases) {#two-phases}

Docker Swarm には、大きく分けて二つのフェーズがあります

<strong>スタンドアロン Swarm（2014 年）</strong>

初期の Docker Swarm は、Docker Engine とは別のツールとして提供されていました

複数の Docker ホストを束ねてクラスタを構成し、標準の Docker API を通じてコンテナを操作できるようにするものでした

<strong>Swarm mode / SwarmKit（2016 年）</strong>

2016 年の Docker Engine 1.12 で、Swarm の機能が Docker Engine 自体に統合されました

この統合版は <strong>SwarmKit</strong> をベースとしており、<strong>Swarm mode</strong> と呼ばれます

Swarm mode では、`docker swarm init` でクラスタを初期化し、`docker service create` でサービスを作成できます

### [Docker Swarm と Kubernetes の比較](#swarm-kubernetes-comparison) {#swarm-kubernetes-comparison}

{: .labeled}
| 観点 | Docker Swarm | Kubernetes |
| ------------ | ------------------------------------------------- | ----------------------------------------------------- |
| セットアップ | docker swarm init だけでクラスタを構築可能 | 複数のコンポーネントのセットアップが必要 |
| 学習コスト | Docker CLI の延長で操作できる | 独自の概念（Pod、Service、Deployment 等）の学習が必要 |
| サービス定義 | docker service create コマンドや Compose ファイル | YAML マニフェスト |
| 拡張性 | 限定的 | CRD や Operator パターンによる高い拡張性 |
| エコシステム | Docker 社のツールに限定 | CNCF を中心とした広大なエコシステム |

Docker Swarm の最大の強みは、Docker を使っている開発者にとっての親しみやすさでした

しかし、その親しみやすさは拡張性の制約と表裏一体でした

### [Docker 社自身の Kubernetes 採用](#docker-kubernetes-adoption) {#docker-kubernetes-adoption}

2017 年、Docker Desktop が Kubernetes のサポートを開始しました

Docker 社自身が Kubernetes を取り込んだことは、オーケストレーションの標準が Kubernetes に収束しつつあることを象徴する出来事でした

Docker Swarm は廃止されたわけではなく、シンプルなオーケストレーションが必要な場合に選択肢として残っています

しかし、エコシステムの規模やクラウドプロバイダのサポートという点では、Kubernetes との差は大きく開きました

---

## [HashiCorp Nomad](#hashicorp-nomad) {#hashicorp-nomad}

### [背景と設計](#nomad-background) {#nomad-background}

<strong>HashiCorp Nomad</strong> は、2015 年に HashiCorp 社がリリースしたワークロードスケジューラです

Nomad の最大の特徴は、<strong>コンテナに限定されない</strong>ことです

Docker コンテナ、Java アプリケーション、仮想マシン、バッチジョブなど、さまざまな種類のワークロードをスケジューリングできます

### [設計哲学](#nomad-design-philosophy) {#nomad-design-philosophy}

Nomad の設計哲学は<strong>シンプルさ</strong>です

<strong>単一バイナリ</strong>

Nomad は単一のバイナリファイルで動作します

Kubernetes のように etcd、API Server、Controller Manager、Scheduler など複数のコンポーネントを個別にセットアップする必要がありません

<strong>段階的な採用</strong>

小規模な環境から始めて、必要に応じて段階的にスケールアップできる設計です

最初はシンプルなジョブスケジューラとして使い、成長に合わせて機能を追加していくことができます

<strong>HashiCorp エコシステムとの統合</strong>

Nomad は、HashiCorp の他のツールと連携して使うことを想定しています

サービスディスカバリには Consul、シークレット管理には Vault を組み合わせる構成が一般的です

Kubernetes がこれらの機能を内蔵しているのに対し、Nomad は必要な機能を必要に応じて組み合わせるアプローチを取ります

### [Nomad と Kubernetes の比較](#nomad-kubernetes-comparison) {#nomad-kubernetes-comparison}

{: .labeled}
| 観点 | Nomad | Kubernetes |
| -------------------- | -------------------------------------- | ------------------------------------------------ |
| 対象ワークロード | コンテナ、VM、Java、バッチなど多種多様 | コンテナが中心 |
| アーキテクチャ | 単一バイナリ | 複数コンポーネント |
| セットアップ | シンプル（単一バイナリのデプロイ） | 複雑（複数コンポーネントの構成が必要） |
| サービスディスカバリ | Consul との連携 | 内蔵（CoreDNS、Service リソース） |
| シークレット管理 | Vault との連携 | 内蔵（Secret リソース） |
| エコシステム | HashiCorp ツール中心 | CNCF を中心とした広大なエコシステム |
| クラウドサポート | セルフマネージドが中心 | 主要クラウドプロバイダがマネージドサービスを提供 |

Nomad は、Kubernetes の複雑さを避けたい場合や、コンテナ以外のワークロードも統一的にスケジューリングしたい場合に選ばれることがあります

---

## [なぜ Kubernetes が標準となったか](#why-kubernetes-standard) {#why-kubernetes-standard}

[01-orchestration](../../01-orchestration/) でも触れましたが、ここではこれまで見てきた他のオーケストレーション技術との比較を踏まえて、Kubernetes が標準となった理由をより深く整理します

### [CNCF による中立的なガバナンス](#cncf-governance) {#cncf-governance}

2015 年に <strong>CNCF（Cloud Native Computing Foundation）</strong>が設立され、Kubernetes 1.0 が CNCF に移管されました

CNCF は特定の企業に依存しない中立的な組織です

Docker Swarm は Docker 社が、Nomad は HashiCorp 社が、それぞれ自社で開発・管理していました

一方、Kubernetes は CNCF のもとで、Google、Red Hat、Microsoft、IBM など多数の企業が共同で開発しています

この中立的なガバナンスにより、特定のベンダーに縛られるリスクが低減され、企業が安心して採用できる環境が整いました

### [拡張性の高い設計](#extensible-design) {#extensible-design}

Kubernetes は、コア機能を変更せずに機能を追加できる拡張ポイントを多数備えています

<strong>CRD（Custom Resource Definition）</strong>により、Kubernetes の API に独自のリソースタイプを追加できます

<strong>Operator パターン</strong>により、CRD と Controller を組み合わせて、特定のアプリケーション（データベース、メッセージキューなど）の運用知識を自動化できます

この拡張性は、Docker Swarm や Nomad にはない Kubernetes 固有の強みです

Mesos も拡張可能な設計でしたが、フレームワークモデルは Kubernetes の CRD / Operator モデルほど開発者コミュニティに浸透しませんでした

### [クラウドプロバイダによるマネージドサービス](#managed-service-by-cloud-providers) {#managed-service-by-cloud-providers}

2018 年までに、主要なクラウドプロバイダがすべて Kubernetes のマネージドサービスを提供しました

{: .labeled}
| クラウドプロバイダ | サービス名 |
| ------------------- | ---------- |
| Google Cloud | GKE |
| Amazon Web Services | EKS |
| Microsoft Azure | AKS |

マネージドサービスの提供により、Kubernetes のセットアップや運用の複雑さが大幅に軽減されました

これは、セットアップのシンプルさを売りにしていた Docker Swarm や Nomad のアドバンテージを相対的に小さくしました

### [エコシステムの規模](#ecosystem-scale) {#ecosystem-scale}

Kubernetes を中心に、監視（Prometheus）、サービスメッシュ（Istio）、CI/CD（Argo CD）など、膨大なエコシステムが形成されました

CNCF が管理するプロジェクトの多くが Kubernetes との連携を前提としています

このエコシステムの規模は、他のオーケストレーション技術が独自に構築することが困難なものでした

### [まとめ](#summary) {#summary}

{: .labeled}
| 要因 | 効果 |
| ----------------------------- | ------------------------------------------------------- |
| CNCF による中立的なガバナンス | ベンダーロックインの不安を解消 |
| CRD / Operator による拡張性 | あらゆるユースケースに対応できる柔軟性 |
| クラウドプロバイダのサポート | セットアップの複雑さを解消し、採用障壁を下げた |
| 大規模エコシステム | 周辺ツールが充実し、Kubernetes を選ぶ理由がさらに増えた |

これらの要因が相互に強化し合う好循環が生まれ、Kubernetes はコンテナオーケストレーションの事実上の標準となりました

---

## [オーケストレーション技術の進化の流れ](#orchestration-evolution) {#orchestration-evolution}

```
Google Borg（2003 頃〜）
  │  大規模クラスタ管理の経験
  │  宣言的構成、調整ループ、API 中心の設計
  │
  ├──→ Google Omega
  │      共有状態スケジューリングの実験
  │
  ↓
Apache Mesos（2009）
  │  データセンター規模のリソース共有
  │  二段階スケジューリング
  │
  ├──→ Marathon
  │      Mesos 上のコンテナオーケストレーション
  │
  ↓
Docker（2013）
  │  コンテナの普及
  │  オーケストレーションの需要が急増
  │
  ├──→ Docker Swarm（2014）
  │      Docker ネイティブのオーケストレーション
  │      → SwarmKit 統合（2016）
  │      → Docker 自身が K8s を採用（2017）
  │
  ├──→ Kubernetes（2014）
  │      Borg / Omega の教訓を反映
  │      → CNCF 設立（2015）
  │      → CRD / Operator パターン
  │      → 主要クラウドのマネージドサービス（2018）
  │      → 事実上の標準に
  │
  └──→ HashiCorp Nomad（2015）
         コンテナに限定しないスケジューラ
         単一バイナリのシンプルな設計
```

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Borg | Google が社内で運用していた大規模クラスタ管理システム。Kubernetes の設計に影響を与えた |
| Omega | Google 社内の研究プロジェクト。共有状態スケジューリングによって Borg のスケジューラのボトルネックを解消する実験的システム |
| Apache Mesos | UC Berkeley で開発されたクラスタ管理システム。二段階スケジューリングにより、データセンター全体のリソースを複数のフレームワーク間で共有する |
| Marathon | Mesos 上でコンテナの長時間稼働サービスを管理するフレームワーク。デプロイ、スケーリング、セルフヒーリングの機能を提供していた |
| Docker Swarm | Docker 社が開発したコンテナオーケストレーションツール。Docker CLI の延長として操作できるシンプルさが特徴 |
| SwarmKit | Docker Engine 1.12 で統合された Swarm mode の基盤。スタンドアロン Swarm から統合型に移行した際の内部実装 |
| Nomad | HashiCorp 社が開発したワークロードスケジューラ。コンテナだけでなく VM やバッチジョブなど多様なワークロードに対応する |
| 二段階スケジューリング（Two-level Scheduling） | Mesos が採用したスケジューリングモデル。第一段階で Mesos がリソースを提示し、第二段階でフレームワークが選択する |
| 共有状態スケジューリング（Shared-state Scheduling） | Omega が採用したモデル。複数のスケジューラがクラスタ状態のコピーを共有し、独立してスケジューリング判断を行う |
| CRD（Custom Resource Definition） | Kubernetes の API に独自のリソースタイプを追加する仕組み。Kubernetes の拡張性を支える重要な機能 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>学術論文</strong>

- "Large-scale cluster management at Google with Borg" (Verma et al., EuroSys 2015)
  - Google の大規模クラスタ管理システム Borg の設計と運用に関する論文

- "Borg, Omega, and Kubernetes" (Burns et al., ACM Queue 2016)
  - Borg、Omega、Kubernetes の設計思想の変遷と、各システムから得られた教訓を解説した論文

- "Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center" (Hindman et al., NSDI 2011)
  - Apache Mesos の二段階スケジューリングモデルとデータセンター規模のリソース共有の仕組みに関する論文

<strong>CNCF</strong>

- [Cloud Native Computing Foundation](https://www.cncf.io/){:target="\_blank"}
  - Kubernetes を含むクラウドネイティブ技術のオープンソースプロジェクトを管理する中立的な組織の公式サイト

<strong>Nomad</strong>

- [Nomad by HashiCorp](https://www.nomadproject.io/){:target="\_blank"}
  - HashiCorp Nomad の公式サイト
