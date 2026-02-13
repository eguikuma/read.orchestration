---
layout: default
title: 宣言的構成管理
---

# [07-declarative-management：宣言的構成管理](#declarative-management) {#declarative-management}

## [はじめに](#introduction) {#introduction}

前のトピック [06-scaling](../06-scaling/) では、負荷に応じてあるべき状態を動的に変更するスケーリングの仕組みを学びました

HPA がメトリクスを監視し、レプリカ数を自動で増減させることで、アクセスの急増にも自動で対応できることを確認しました

ここまでのトピックで、Kubernetes の個別の仕組みを学んできました

- スケジューリング：Pod をどのノードに配置するか
- サービスディスカバリ：Pod 同士がどう発見し合うか
- セルフヒーリング：障害からどう自動復旧するか
- スケーリング：負荷に応じてどう Pod を増減させるか

しかし、これらの仕組みはどうやって統合的に管理されるのでしょうか？

Kubernetes に対して「何を、どのような状態にしたいか」を、どうやって伝えるのでしょうか？

アプリケーションを更新するとき、ダウンタイムなしで新しいバージョンに切り替えるにはどうするのでしょうか？

このトピックでは、<strong>宣言的構成管理</strong>の仕組みを学びます

あるべき状態をマニフェストとして記述し、Kubernetes がその実現と維持を自動で行う仕組みを見ていきます

このリポジトリの最後のトピックとして、ここまで学んだ全メカニズムを統合する視点を提供します

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

宣言的構成管理の考え方を、日常の例えで見てみましょう

<strong>宣言的 vs 命令的 = エアコンの温度設定 vs 手動操作</strong>

部屋の温度を快適にしたいとします

<strong>命令的アプローチ</strong>は、「暑くなったらエアコンをつけて、寒くなったら消す」と都度指示する方法です

気温の変化を自分で監視し、そのたびに操作を行う必要があります

外出中は操作できないため、帰宅したら部屋が暑くなっています

<strong>宣言的アプローチ</strong>は、「25 度に設定する」とだけ伝える方法です

エアコンが現在の室温と設定温度の差を自動で検知し、冷房や暖房を調整して 25 度を維持します

外出中でも、窓を開けても、エアコンが自動で 25 度に戻します

Kubernetes の宣言的構成管理も同じ仕組みです

「Web サーバーの Pod を 3 つ、データベースの Pod を 1 つ動かす」とマニフェストに記述するだけで、Kubernetes が自動でその状態を実現し、維持します

<strong>マニフェスト = 建物の設計図</strong>

設計図があれば、何度でも同じ建物を再現できます

設計図を変更すれば、施工チームが差分を読み取り、必要な部分だけを改修します

マニフェストも同じです

マニフェストに記述された状態を変更すれば、Kubernetes が差分を検知し、必要な調整だけを行います

<strong>ローリングアップデート = 営業中の店舗改装</strong>

10 フロアのデパートを改装するとします

全フロアを一斉に閉鎖して改装すると、その間お客さんはどのフロアも使えません

代わりに、1 フロアずつ改装し、他のフロアは営業を続けます

すべてのフロアの改装が終わるまで、常にほとんどのフロアが営業中です

Kubernetes のローリングアップデートも同じ仕組みです

古い Pod を 1 つずつ新しい Pod に置き換え、常に一定数の Pod がトラフィックを処理し続けます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>宣言的管理の基本</strong>

- <strong>宣言的アプローチと命令的アプローチ</strong>
  - 2 つの管理スタイルの違いと、Kubernetes が宣言的を採用する理由
- <strong>マニフェスト</strong>
  - あるべき状態を YAML で記述する仕組み

<strong>アプリケーションのライフサイクル管理</strong>

- <strong>Deployment</strong>
  - ReplicaSet を管理し、アプリケーションの更新とロールバックを制御する上位リソース
- <strong>ローリングアップデート</strong>
  - ダウンタイムなしでアプリケーションを更新する仕組み
- <strong>ロールバック</strong>
  - 更新に問題があった場合に以前の状態に戻す仕組み

<strong>リソースの整理と変更管理</strong>

- <strong>ラベルとアノテーション</strong>
  - リソースの選択・分類・メタデータ付与の仕組み
- <strong>Namespace</strong>
  - リソースを論理的にグループ化する仕組み
- <strong>GitOps</strong>
  - マニフェストを Git リポジトリで管理し、変更を追跡可能にする手法

---

## [目次](#table-of-contents) {#table-of-contents}

1. [宣言的アプローチと命令的アプローチ](#declarative-and-imperative-approaches)
2. [マニフェスト（あるべき状態の記述）](#manifest)
3. [Deployment（アプリケーションのライフサイクル管理）](#deployment)
4. [ローリングアップデート](#rolling-update)
5. [ロールバック](#rollback)
6. [ラベルとアノテーション](#labels-and-annotations)
7. [Namespace によるリソース分離](#namespace-resource-isolation)
8. [GitOps（変更管理と継続的デプロイ）](#gitops)
9. [宣言的構成管理の全体像](#declarative-management-overview)
10. [まとめ：このリポジトリで学んだこと](#repository-learning-summary)
11. [用語集](#glossary)
12. [参考資料](#references)

---

## [宣言的アプローチと命令的アプローチ](#declarative-and-imperative-approaches) {#declarative-and-imperative-approaches}

### [命令的アプローチ](#imperative-approach) {#imperative-approach}

<strong>命令的アプローチ</strong>は、<strong>システムに対して「何をするか」を操作として指示する</strong>方法です

「Pod を 3 つ作れ」「Pod を 1 つ削除しろ」「コンテナイメージを更新しろ」のように、1 つ 1 つの操作を順番に指示します

命令的アプローチでは、管理者がシステムの現在の状態を把握し、目的の状態に至るまでの手順を考え、その手順を正しい順序で実行する必要があります

### [宣言的アプローチ](#declarative-approach) {#declarative-approach}

<strong>宣言的アプローチ</strong>は、<strong>システムに対して「どうあるべきか」を状態として宣言する</strong>方法です

「Pod は 3 つあるべき」「コンテナイメージは v2 であるべき」のように、最終的な状態だけを記述します

管理者は手順を考える必要がありません

Kubernetes が現在の状態と宣言された状態の差分を検知し、必要な操作を自動で実行します

### [2 つのアプローチの比較](#approaches-comparison) {#approaches-comparison}

{: .labeled}
| 観点 | 命令的アプローチ | 宣言的アプローチ |
| -------------- | --------------------------------------- | --------------------------------------- |
| 指示する内容 | 操作（「〜しろ」） | 状態（「〜であるべき」） |
| 手順の管理 | 管理者が考える | Kubernetes が自動で決定 |
| 再実行の安全性 | 同じ操作を 2 回実行すると問題が起きうる | 同じ宣言を何度適用しても同じ状態になる |
| 障害時の復旧 | 復旧手順を管理者が実行する | Kubernetes が自動で宣言された状態に戻す |

### [冪等性](#idempotency) {#idempotency}

宣言的アプローチの重要な特性が、<strong>冪等性（べきとうせい）</strong>です

冪等性とは、<strong>同じ操作を何度実行しても、結果が同じになる</strong>性質です

「Pod を 3 つあるべき」という宣言を 10 回適用しても、Pod の数は 3 つのままです

一方、命令的な「Pod を 3 つ作れ」を 10 回実行すると、Pod が 30 個になってしまいます

冪等性があることで、「今の状態が正しいか分からなければ、宣言をもう一度適用すればよい」という安心感が生まれます

### [Desired State との関係](#desired-state-relationship) {#desired-state-relationship}

[02-architecture](../02-architecture/) で導入した「あるべき状態（Desired State）」は、まさに宣言的アプローチの実現です

管理者があるべき状態を宣言し、Kubernetes が Reconciliation Loop を通じて実際の状態をあるべき状態に収束させます

ここまでの全トピックで見てきた仕組み（スケジューリング、サービスディスカバリ、セルフヒーリング、スケーリング）は、すべてこの宣言的アプローチの上に成り立っています

---

## [マニフェスト（あるべき状態の記述）](#manifest) {#manifest}

### [マニフェストとは](#what-is-manifest) {#what-is-manifest}

<strong>マニフェスト</strong>とは、<strong>あるべき状態を YAML 形式で記述したファイル</strong>です

マニフェストには「何を、どのような状態にしたいか」が書かれています

Kubernetes は、マニフェストを受け取ると、その内容をあるべき状態として etcd に保存し、Reconciliation Loop を通じて実現します

### [マニフェストの基本構造](#manifest-structure) {#manifest-structure}

マニフェストは、以下の 4 つのフィールドで構成されます

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: web-app:v1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 128Mi
```

{: .labeled}
| フィールド | 説明 |
| ------------ | ------------------------------------------------ |
| `apiVersion` | 使用する API のバージョン |
| `kind` | リソースの種類（Deployment、Service、Pod など） |
| `metadata` | リソースの名前やラベルなどの識別情報 |
| `spec` | あるべき状態の詳細（レプリカ数、コンテナの設定） |

### [「設定ファイル」ではなく「状態の宣言」](#state-declaration-not-config) {#state-declaration-not-config}

マニフェストは単なる設定ファイルではありません

「この設定でアプリケーションを起動してください」ではなく、「このリソースはこの状態であるべきです」という宣言です

この違いは重要です

設定ファイルは「起動時に読み込まれて反映される」ものですが、マニフェストは「常にあるべき状態として参照され、差分があれば自動で修正される」ものです

### [マニフェストの適用](#manifest-application) {#manifest-application}

マニフェストを API Server に送信すると、以下の流れで処理されます

```
マニフェストを API Server に送信
  │
  ▼
API Server がマニフェストを検証し、etcd に保存
  │
  ▼
コントローラがあるべき状態の変更を検知
  │
  ▼
Reconciliation Loop が実際の状態との差分を計算
  │
  ▼
差分を解消するための操作を実行（Pod の作成、更新、削除など）
```

マニフェストの内容を変更して再度適用すると、Kubernetes は前回の状態と新しい状態の差分を検知し、必要な部分だけを変更します

全体を作り直すのではなく、差分のみを反映する効率的な仕組みです

---

## [Deployment（アプリケーションのライフサイクル管理）](#deployment) {#deployment}

### [Deployment とは](#what-is-deployment) {#what-is-deployment}

<strong>Deployment</strong>は、<strong>ReplicaSet を管理し、アプリケーションの更新とロールバックを制御する</strong>上位リソースです

これまでのトピックでは、ReplicaSet が Pod のレプリカ数を維持する仕組みを学んできました

Deployment は、その ReplicaSet をさらに管理する階層です

### [Deployment → ReplicaSet → Pod の階層](#deployment-hierarchy) {#deployment-hierarchy}

Kubernetes のワークロード管理は、3 層の階層構造になっています

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

管理者が ReplicaSet を直接操作する必要はありません

### [なぜ ReplicaSet を直接使わないか](#why-not-use-replica-set-directly) {#why-not-use-replica-set-directly}

ReplicaSet は Pod のレプリカ数を維持する機能しか持ちません

アプリケーションの更新（コンテナイメージの変更など）を行うと、ReplicaSet はすべての Pod を一度に新しいバージョンに置き換えようとします

Deployment は、ReplicaSet を複数管理することで、段階的な更新（ローリングアップデート）やロールバックといった、より高度なライフサイクル管理を実現します

---

## [ローリングアップデート](#rolling-update) {#rolling-update}

### [Compose の限界](#compose-limitations) {#compose-limitations}

前のシリーズでコンテナ管理を学んだ方は、Compose でアプリケーションを更新するとき、コンテナを停止して新しいイメージで再作成したことを思い出すかもしれません

この方法では、停止から再作成までの間にサービスが利用できない時間（ダウンタイム）が発生します

Kubernetes の Deployment は、<strong>ローリングアップデート</strong>によってダウンタイムなしのアプリケーション更新を実現します

### [ローリングアップデートの仕組み](#rolling-update-mechanism) {#rolling-update-mechanism}

<strong>ローリングアップデート</strong>は、<strong>古い Pod を 1 つずつ新しい Pod に置き換える</strong>更新方法です

すべての Pod を一度に停止するのではなく、一部の Pod を新しいバージョンに置き換えながら、残りの Pod がトラフィックを処理し続けます

### [更新の流れ](#update-flow) {#update-flow}

アプリケーションのコンテナイメージを `v1` から `v2` に更新する場合を見てみましょう

```
更新前
  Deployment → ReplicaSet（v1）→ Pod v1, Pod v1, Pod v1

マニフェストでイメージを v2 に変更して適用
  │
  ▼
Deployment が新しい ReplicaSet（v2）を作成
  │
  ▼
ReplicaSet（v2）が Pod v2 を 1 つ作成
  │
  ▼
Pod v2 が Ready になったら、ReplicaSet（v1）の Pod を 1 つ削除
  │
  ▼
これを繰り返す
  │
  ▼
更新完了
  Deployment → ReplicaSet（v2）→ Pod v2, Pod v2, Pod v2
              ReplicaSet（v1）→ （Pod なし、履歴として保持）
```

このように、新しい ReplicaSet を段階的にスケールアップし、古い ReplicaSet を段階的にスケールダウンすることで、常に一定数の Pod がトラフィックを処理し続けます

### [maxSurge と maxUnavailable](#max-surge-and-max-unavailable) {#max-surge-and-max-unavailable}

ローリングアップデートの速度は、2 つのパラメータで制御します

{: .labeled}
| パラメータ | 説明 | デフォルト |
| -------------- | --------------------------------------------- | ---------- |
| maxSurge | あるべきレプリカ数を超えて追加できる Pod の数 | 25% |
| maxUnavailable | 更新中に利用不可になってよい Pod の数 | 25% |

たとえば、レプリカ数が 4 の Deployment で、デフォルト値（25%）の場合を考えます

- maxSurge = 1（4 × 25% = 1）：Pod は最大 5 つまで同時に存在できる
- maxUnavailable = 1（4 × 25% = 1）：利用可能な Pod は最低 3 つを維持する

これにより、常に 3 つ以上の Pod がトラフィックを処理しつつ、段階的に新しいバージョンに切り替わります

### [Service との連携](#service-integration) {#service-integration}

ローリングアップデートは、[04-service-discovery](../04-service-discovery/) で学んだ Service と連携して動作します

新しい Pod が起動し、Readiness Probe に成功すると、EndpointSlice に追加されてトラフィックを受け始めます

古い Pod が削除されると、EndpointSlice から除外されてトラフィックが転送されなくなります

この連携により、ユーザーから見てダウンタイムのない更新が実現されます

---

## [ロールバック](#rollback) {#rollback}

### [更新が失敗した場合](#update-failure) {#update-failure}

アプリケーションを `v1` から `v2` に更新したものの、`v2` にバグがあった場合はどうなるでしょうか

新しい Pod が起動しても、Liveness Probe に失敗してクラッシュを繰り返す可能性があります

このような場合、以前の正常な状態に<strong>ロールバック</strong>（巻き戻し）する必要があります

### [Deployment のリビジョン管理](#deployment-revision-management) {#deployment-revision-management}

Deployment は、過去の ReplicaSet を<strong>リビジョン（改訂履歴）</strong>として保持しています

```
Deployment
  ├── ReplicaSet（v1）← リビジョン 1（Pod なし、履歴として保持）
  └── ReplicaSet（v2）← リビジョン 2（現在アクティブ）
```

マニフェストを変更して適用するたびに新しいリビジョンが作成されますが、古い ReplicaSet は削除されずに残ります

### [ロールバックの仕組み](#rollback-mechanism) {#rollback-mechanism}

ロールバックは、<strong>あるべき状態を以前のリビジョンに戻す</strong>操作です

技術的には、古い ReplicaSet のレプリカ数を増やし、現在の ReplicaSet のレプリカ数を減らすことで実現されます

```
ロールバック実行
  │
  ▼
ReplicaSet（v1）のレプリカ数を増やす
  │
  ▼
ReplicaSet（v2）のレプリカ数を減らす
  │
  ▼
ローリングアップデートと同じ流れで段階的に切り替え
  │
  ▼
ロールバック完了
  Deployment → ReplicaSet（v1）→ Pod v1, Pod v1, Pod v1
              ReplicaSet（v2）→ （Pod なし、履歴として保持）
```

ロールバックもローリングアップデートと同じ段階的な切り替えで行われるため、ダウンタイムは発生しません

### [宣言的操作としてのロールバック](#rollback-as-declarative-operation) {#rollback-as-declarative-operation}

ロールバックは「v2 を停止して v1 を起動しろ」という命令的な操作ではありません

「あるべき状態を v1 の状態に戻す」という宣言的な操作です

Kubernetes は、新しいあるべき状態（v1）と現在の状態（v2）の差分を計算し、必要な調整を自動で行います

---

## [ラベルとアノテーション](#labels-and-annotations) {#labels-and-annotations}

### [ラベル](#label) {#label}

<strong>ラベル（Label）</strong>は、Kubernetes のリソースに付与する<strong>キーと値のペア</strong>です

ラベルは、リソースを選択・分類・フィルタリングするために使います

```yaml
metadata:
  labels:
    app: web
    environment: production
    version: v2
```

ラベルは Kubernetes の多くの仕組みで活用されています

{: .labeled}
| 仕組み | ラベルの役割 |
| ---------- | ------------------------------------------------ |
| Service | ラベルセレクタで対象の Pod を選択する |
| ReplicaSet | ラベルセレクタで管理する Pod を識別する |
| Scheduler | nodeSelector でラベルに基づくノード選択を行う |
| HPA | スケーリング対象の Deployment をラベルで特定する |

ここまでのトピックで学んだ Service のラベルセレクタも、[03-scheduling](../03-scheduling/) で学んだ nodeSelector も、すべてラベルの仕組みの上に成り立っています

### [アノテーション](#annotation) {#annotation}

<strong>アノテーション（Annotation）</strong>は、ラベルと同じくキーと値のペアですが、<strong>リソースの選択やフィルタリングには使われない</strong>点が異なります

アノテーションは、リソースに対するメタデータ（補足情報）を付与するために使います

```yaml
metadata:
  annotations:
    description: "メインの Web アプリケーション"
    managed-by: "team-platform"
```

ラベルが「このリソースを見つけるための情報」であるのに対し、アノテーションは「このリソースについての補足情報」です

---

## [Namespace によるリソース分離](#namespace-resource-isolation) {#namespace-resource-isolation}

### [Namespace とは](#what-is-namespace) {#what-is-namespace}

<strong>Namespace</strong>は、クラスタ内のリソースを<strong>論理的にグループ化する</strong>仕組みです

1 つのクラスタの中に、複数の仮想的な区画を作ることができます

たとえば、開発環境、ステージング環境、本番環境を同じクラスタ内で分離できます

```
クラスタ
  ├── Namespace: development
  │     ├── Deployment（web-app）
  │     └── Service（web-service）
  │
  ├── Namespace: staging
  │     ├── Deployment（web-app）
  │     └── Service（web-service）
  │
  └── Namespace: production
        ├── Deployment（web-app）
        └── Service（web-service）
```

同じ名前のリソース（`web-app`）が、異なる Namespace に存在できます

Namespace ごとに独立した空間が提供されるためです

### [リソースクォータ](#resource-quota) {#resource-quota}

Namespace には<strong>リソースクォータ</strong>を設定できます

リソースクォータは、その Namespace 内で使用できるリソースの上限を定義します

たとえば「この Namespace では CPU を最大 4 コアまで、メモリを最大 8 GiB まで使用可能」という制限を設けられます

これにより、1 つの Namespace がクラスタのリソースを独占することを防ぎます

---

## [GitOps（変更管理と継続的デプロイ）](#gitops) {#gitops}

### [マニフェストの変更管理](#manifest-change-management) {#manifest-change-management}

マニフェストは YAML ファイルとして記述されるため、<strong>Git リポジトリで管理</strong>できます

この手法を<strong>GitOps</strong>と呼びます

### [GitOps の考え方](#gitops-concept) {#gitops-concept}

GitOps は、<strong>Git リポジトリをシステムのあるべき状態の唯一の情報源（Single Source of Truth）として扱う</strong>手法です

マニフェストを Git リポジトリに保存し、Git の変更履歴がそのままシステムの変更履歴になります

```
Git リポジトリ（マニフェストの変更履歴）
  │
  │  コミット 1: web-app v1、レプリカ数 3
  │  コミット 2: web-app v2、レプリカ数 3（イメージ更新）
  │  コミット 3: web-app v2、レプリカ数 5（スケールアップ）
  │
  ▼
Kubernetes クラスタ（Git の最新状態を反映）
```

### [変更の追跡可能性](#change-traceability) {#change-traceability}

GitOps により、以下の情報が自動的に記録されます

{: .labeled}
| 情報 | 記録方法 |
| -------------- | ------------------------------ |
| 誰が変更したか | Git のコミットの著者 |
| いつ変更したか | Git のコミットのタイムスタンプ |
| 何を変更したか | Git の差分（diff） |
| なぜ変更したか | Git のコミットメッセージ |

これらの情報は、障害発生時の原因調査や、変更の監査に活用されます

### [サプライチェーンセキュリティとの接点](#supply-chain-security-connection) {#supply-chain-security-connection}

前のシリーズでセキュリティを学んだ方は、サプライチェーンの信頼性について学んだことを思い出すかもしれません

GitOps は、デプロイパイプラインの信頼性に貢献します

マニフェストが Git に保存されていれば、変更はコードレビューを経てからクラスタに反映されます

意図しない変更や不正な変更を、デプロイ前に検知できます

Git の変更履歴により、クラスタの状態変更の完全な追跡が可能になります

---

## [宣言的構成管理の全体像](#declarative-management-overview) {#declarative-management-overview}

### [すべてがマニフェストで統合される](#manifest-integration) {#manifest-integration}

ここまでのトピックで学んだ仕組みは、すべてマニフェストを通じて統合されます

1 つのアプリケーションに対して、Deployment、Service、HPA のマニフェストを定義すれば、Kubernetes が自動的に以下を実現します

<strong>Deployment のマニフェスト</strong>

Pod のあるべき状態（コンテナイメージ、レプリカ数、リソース要求）を宣言します

Scheduler が Pod を適切なノードに配置し、セルフヒーリングが状態を維持します

<strong>Service のマニフェスト</strong>

Pod への安定したアクセス先を宣言します

EndpointSlice が Pod を自動追跡し、kube-proxy がトラフィックを転送します

<strong>HPA のマニフェスト</strong>

スケーリングの条件を宣言します

メトリクスに基づいてレプリカ数が自動で調整されます

### [宣言から実現までの流れ](#declaration-to-realization-flow) {#declaration-to-realization-flow}

管理者がマニフェストを適用してから、アプリケーションが稼働するまでの全体の流れです

```
マニフェストを適用
  │
  ▼
API Server が etcd に保存（あるべき状態の登録）
  │
  ├──→ Deployment コントローラが ReplicaSet を作成
  │      │
  │      ▼
  │    ReplicaSet コントローラが Pod の作成を要求
  │      │
  │      ▼
  │    Scheduler が Pod を適切なノードに配置（03-scheduling）
  │      │
  │      ▼
  │    kubelet が Pod を起動
  │
  ├──→ Service のあるべき状態を登録
  │      │
  │      ▼
  │    EndpointSlice が Pod を追跡（04-service-discovery）
  │      │
  │      ▼
  │    kube-proxy がトラフィック転送ルールを設定
  │
  ├──→ HPA のあるべき状態を登録
  │      │
  │      ▼
  │    メトリクスを監視し、必要に応じてレプリカ数を変更（06-scaling）
  │
  └──→ セルフヒーリングが常に稼働
         │
         ▼
       障害を検知し、あるべき状態に自動復旧（05-self-healing）
```

管理者が行うのは、マニフェストの適用だけです

それ以降のすべての動作は、Kubernetes が自動で行います

### [Compose の限界からの完全な回答](#compose-limitations-complete-answer) {#compose-limitations-complete-answer}

[01-orchestration](../01-orchestration/) で挙げた Compose の限界が、すべて解決されたことを確認しましょう

{: .labeled}
| Compose の限界 | Kubernetes の解決策 | 関連トピック |
| -------------------------------------------- | -------------------------------------------------------------- | -------------------- |
| 単一マシンでの管理が前提 | 複数ノードへの自動配置 | 03-scheduling |
| 自動復旧がない（restart は単純な再起動のみ） | 3 層のセルフヒーリング（Probe、ReplicaSet、Node コントローラ） | 05-self-healing |
| 手動でのスケーリング | HPA によるメトリクスに基づく自動スケーリング | 06-scaling |
| DNS が Docker 内部に限定 | CoreDNS によるクラスタ全体のサービスディスカバリ | 04-service-discovery |
| ヘルスチェックが 1 種類のみ | Liveness / Readiness / Startup Probe の 3 種類 | 05-self-healing |
| ローリングアップデートがない | Deployment によるダウンタイムなしの段階的更新 | 07（本トピック） |

---

## [まとめ：このリポジトリで学んだこと](#repository-learning-summary) {#repository-learning-summary}

このリポジトリでは、<strong>コンテナオーケストレーション</strong>の仕組みを 7 つのトピックで学びました

### [全トピックの振り返り](#all-topics-review) {#all-topics-review}

<strong>01-orchestration：オーケストレーションとは何か</strong>

Compose の限界（単一マシン、手動スケーリング、セルフヒーリングの欠如）を出発点に、なぜオーケストレーションが必要かを学びました

オーケストレーションとは、複数マシンにまたがるコンテナ群のスケジューリング、管理、ネットワーキングを自動化する仕組みです

<strong>02-architecture：アーキテクチャ</strong>

Kubernetes のアーキテクチャとして、コントロールプレーンとノードの分離を学びました

あるべき状態（Desired State）と Reconciliation Loop という、以降の全トピックの基盤となる概念を導入しました

<strong>03-scheduling：スケジューリング</strong>

Pod をどのノードに配置するかを決める仕組みを学びました

フィルタリングとスコアリングの 2 段階アルゴリズム、リソース要求に基づく配置判断を確認しました

<strong>04-service-discovery：サービスディスカバリ</strong>

Pod 同士がどのように互いを発見し、安定した通信を実現するかを学びました

Service の ClusterIP、EndpointSlice、kube-proxy、CoreDNS が連携する仕組みを確認しました

<strong>05-self-healing：セルフヒーリング</strong>

障害を自動で検知し、あるべき状態に戻す仕組みを学びました

コンテナレベル、レプリカレベル、ノードレベルの 3 層構造で、さまざまな障害に自動で対応することを確認しました

<strong>06-scaling：スケーリング</strong>

負荷に応じてあるべき状態を動的に変更する仕組みを学びました

HPA がメトリクスに基づいてレプリカ数を自動で調整し、セルフヒーリングの「維持」を「動的な変更」に拡張することを確認しました

<strong>07-declarative-management：宣言的構成管理（本トピック）</strong>

ここまでの全メカニズムを、宣言的なマニフェストとして統合する仕組みを学びました

管理者はあるべき状態を宣言するだけで、Kubernetes が自動的にその実現と維持を行います

### [Desired State が貫く軸](#desired-state-axis) {#desired-state-axis}

全 7 トピックを貫く中心概念は、<strong>あるべき状態（Desired State）</strong>です

{: .labeled}
| あるべき状態の表現 | 関連する仕組み |
| ------------------------------- | ---------------------- |
| 「Pod を 3 つ動かす」 | セルフヒーリング |
| 「この Pod をこのノードに配置」 | スケジューリング |
| 「この名前でアクセスできる」 | サービスディスカバリ |
| 「CPU 50% を目標にスケール」 | スケーリング |
| 「イメージは v2 であるべき」 | ローリングアップデート |

管理者は「何をしたいか」を宣言し、Kubernetes は「どうやって実現するか」を自動で処理します

この分離が、Kubernetes のオーケストレーションの本質です

### [このリポジトリの位置づけ](#repository-positioning) {#repository-positioning}

このリポジトリは、カーネル空間からアプリケーション層に至るまでの学習シリーズの一部です

カーネル空間で学んだ namespace と cgroup が Pod の隔離とリソース制限の基盤を提供し、ユーザー空間で学んだプロセス管理がコンテナの実行モデルの基礎を提供しています

ネットワークの学習で理解した DNS や IP アドレスの仕組みが、サービスディスカバリの土台になっています

コンテナの学習で理解した単一マシンのコンテナ管理が、複数マシンのオーケストレーションへと拡張されました

セキュリティの学習で理解したサプライチェーンの信頼性が、GitOps の考え方につながっています

これらの知識が統合されることで、コンテナオーケストレーションの全体像が理解できるようになります

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| 宣言的アプローチ（Declarative Approach） | システムに対して「どうあるべきか」を状態として記述し、システムがその実現を自動で行う管理方法 |
| 命令的アプローチ（Imperative Approach） | システムに対して「何をするか」を操作として指示し、管理者が手順を制御する管理方法 |
| マニフェスト（Manifest） | あるべき状態を YAML 形式で記述したファイル<br>Kubernetes に適用することで、その状態の実現を指示する |
| 冪等性（Idempotency） | 同じ操作を何度実行しても結果が同じになる性質<br>宣言的アプローチの重要な特性 |
| Deployment | ReplicaSet を管理し、アプリケーションの更新やロールバックを制御する上位リソース |
| ローリングアップデート（Rolling Update） | 古い Pod を 1 つずつ新しい Pod に置き換え、ダウンタイムなしでアプリケーションを更新する方法 |
| ロールバック（Rollback） | 更新に問題があった場合に、あるべき状態を以前のリビジョンに戻す操作 |
| リビジョン（Revision） | Deployment の変更履歴<br>過去の ReplicaSet として保持され、ロールバック時に参照される |
| maxSurge | ローリングアップデート時に、あるべきレプリカ数を超えて追加できる Pod の数 |
| maxUnavailable | ローリングアップデート時に、利用不可になってよい Pod の数 |
| ラベル（Label） | リソースに付与するキーと値のペア<br>リソースの選択、分類、フィルタリングに使用する |
| アノテーション（Annotation） | リソースに付与するキーと値のペア<br>選択やフィルタリングには使われず、補足情報の付与に使用する |
| Namespace | クラスタ内のリソースを論理的にグループ化する仕組み<br>同じクラスタ内で複数の環境やチームを分離するために使用する |
| リソースクォータ（Resource Quota） | Namespace 内で使用できるリソースの上限を定義する仕組み |
| GitOps | Git リポジトリをシステムのあるべき状態の唯一の情報源として扱い、マニフェストの変更管理と継続的デプロイを実現する手法 |
| Single Source of Truth（唯一の情報源） | システムのあるべき状態を一箇所で管理し、それを正とする考え方<br>GitOps では Git リポジトリがこの役割を果たす |
| apiVersion | マニフェストで使用する Kubernetes API のバージョン |
| kind | マニフェストで指定するリソースの種類（Deployment、Service、Pod など） |
| metadata | マニフェストでリソースの名前、ラベル、アノテーションなどの識別情報を記述するフィールド |
| spec | マニフェストであるべき状態の詳細（レプリカ数、コンテナの設定など）を記述するフィールド |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>宣言的管理</strong>

- [Declarative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/){:target="\_blank"}
  - マニフェストによる宣言的なリソース管理の公式ドキュメント

- [Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/){:target="\_blank"}
  - 命令的アプローチと宣言的アプローチの比較の公式ドキュメント

<strong>Deployment</strong>

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/){:target="\_blank"}
  - Deployment の仕組み、ローリングアップデート、ロールバックの公式ドキュメント

<strong>リソース管理</strong>

- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/){:target="\_blank"}
  - ラベルとラベルセレクタの公式ドキュメント

- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/){:target="\_blank"}
  - アノテーションの公式ドキュメント

- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/){:target="\_blank"}
  - Namespace の公式ドキュメント

- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/){:target="\_blank"}
  - リソースクォータの公式ドキュメント
