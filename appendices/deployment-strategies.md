---
layout: default
title: デプロイ戦略の比較
---

# [appendix：デプロイ戦略の比較](#deployment-strategies) {#deployment-strategies}

## [はじめに](#introduction) {#introduction}

[07-declarative-management](../../07-declarative-management/) では、Deployment によるローリングアップデートの仕組みを学びました

古い Pod を 1 つずつ新しい Pod に置き換え、ダウンタイムなしでアプリケーションを更新できることを確認しました

しかし、ローリングアップデートはデプロイ戦略の 1 つに過ぎません

アプリケーションの性質や要件によっては、ローリングアップデートよりも適した戦略が存在します

この補足資料では、ローリングアップデート以外のデプロイ戦略を含めた<strong>4 つの戦略</strong>を比較し、それぞれの特徴と適したケースを整理します

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

- <strong>なぜ複数の戦略があるか</strong>
  - すべてのアプリケーションに最適な単一のデプロイ戦略は存在しないこと
- <strong>Recreate（再作成）</strong>
  - 全 Pod を停止してから全 Pod を起動する、最もシンプルな戦略
- <strong>Rolling Update（ローリングアップデート）</strong>
  - 宣言的構成管理で学んだ、段階的な置き換え戦略の振り返り
- <strong>Blue-Green デプロイ</strong>
  - 2 つの環境を用意し、トラフィックを一度に切り替える戦略
- <strong>Canary デプロイ</strong>
  - 少量のトラフィックで新バージョンを検証してから全展開する戦略
- <strong>各戦略の比較</strong>
  - ダウンタイム、ロールバック速度、リソースコスト、リスクの観点での整理

---

## [目次](#table-of-contents) {#table-of-contents}

1. [なぜ複数の戦略があるか](#why-multiple-strategies)
2. [Recreate（再作成）](#recreate)
3. [Rolling Update（ローリングアップデート）](#rolling-update-strategy)
4. [Blue-Green デプロイ](#blue-green-deployment)
5. [Canary デプロイ](#canary-deployment)
6. [各戦略の比較](#strategies-comparison)
7. [用語集](#glossary)
8. [参考資料](#references)

---

## [なぜ複数の戦略があるか](#why-multiple-strategies) {#why-multiple-strategies}

すべてのアプリケーションに最適な単一のデプロイ戦略は存在しません

その理由は、アプリケーションごとにデプロイに対する要件が異なるためです

### [要件のトレードオフ](#requirements-tradeoff) {#requirements-tradeoff}

デプロイ戦略を選択する際に考慮すべき要件には、以下のようなものがあります

{: .labeled}
| 要件 | 説明 |
| -------------------- | -------------------------------------------------------------- |
| ダウンタイムの許容度 | サービスが一時的に停止することを許容できるかどうか |
| ロールバックの速度 | 問題が発覚した際に、どれだけ速く以前のバージョンに戻せるか |
| リソースコスト | デプロイ中に追加で必要となるコンピュータリソースの量 |
| リスクの許容度 | 新バージョンの問題がユーザーに与える影響をどこまで許容できるか |

これらの要件は互いにトレードオフの関係にあります

たとえば、ロールバックを即時に行えるようにするには、旧バージョンの環境を維持する必要があり、リソースコストが増加します

リスクを最小限に抑えるには、段階的にトラフィックを移行する仕組みが必要であり、デプロイの複雑さが増します

### [4 つの戦略](#four-strategies) {#four-strategies}

この appendix では、以下の 4 つのデプロイ戦略を取り上げます

{: .labeled}
| 戦略 | 概要 |
| -------------- | -------------------------------------------------- |
| Recreate | 全 Pod を停止してから、全 Pod を起動する |
| Rolling Update | 古い Pod を 1 つずつ新しい Pod に置き換える |
| Blue-Green | 2 つの環境を用意し、トラフィックを一度に切り替える |
| Canary | 少数の Pod で新バージョンを検証してから全展開する |

それぞれの戦略には、異なるトレードオフがあります

以降のセクションで、各戦略の仕組みと特徴を見ていきます

---

## [Recreate（再作成）](#recreate) {#recreate}

### [仕組み](#recreate-mechanism) {#recreate-mechanism}

<strong>Recreate</strong> は、最もシンプルなデプロイ戦略です

全ての既存 Pod を停止してから、全ての新しい Pod を起動します

```
更新前
  Pod v1, Pod v1, Pod v1（全 Pod が v1）

更新開始
  │
  ▼
全ての v1 Pod を停止
  （Pod なし ← この間ダウンタイムが発生）
  │
  ▼
全ての v2 Pod を起動
  Pod v2, Pod v2, Pod v2（全 Pod が v2）

更新完了
```

### [特徴](#recreate-characteristics) {#recreate-characteristics}

<strong>ダウンタイムが発生する</strong>

旧 Pod が全て停止してから新 Pod の起動が完了するまでの間、トラフィックを処理する Pod が存在しません

この期間がダウンタイムになります

<strong>旧バージョンと新バージョンが同時に存在しない</strong>

Recreate では、旧 Pod が全て停止した後に新 Pod が起動するため、旧バージョンと新バージョンが同時に動作する期間がありません

これは、旧バージョンと新バージョンの互換性を気にする必要がないことを意味します

<strong>リソースコストが低い</strong>

旧 Pod と新 Pod が同時に存在しないため、追加のリソースを必要としません

### [適したケース](#recreate-suitable-cases) {#recreate-suitable-cases}

Recreate が適しているのは、旧バージョンと新バージョンが<strong>同時に動作できない</strong>場合です

たとえば、データベースのスキーマ変更を伴うデプロイでは、旧バージョンのアプリケーションが新しいスキーマに対応できない可能性があります

このような場合、旧バージョンを完全に停止してからスキーマを変更し、新バージョンを起動する必要があります

### [Kubernetes での設定](#configuring-kubernetes) {#configuring-kubernetes}

Kubernetes の Deployment では、`strategy.type` を `Recreate` に設定することでこの戦略を使用できます

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  strategy:
    type: Recreate
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
          image: web-app:v2
```

---

## [Rolling Update（ローリングアップデート）](#rolling-update-strategy) {#rolling-update-strategy}

### [振り返り](#rolling-update-review) {#rolling-update-review}

<strong>Rolling Update</strong> は、[07-declarative-management](../../07-declarative-management/) で学んだデプロイ戦略です

古い Pod を 1 つずつ新しい Pod に置き換え、常に一定数の Pod がトラフィックを処理し続けます

```
更新前
  ReplicaSet（v1）→ Pod v1, Pod v1, Pod v1

更新開始
  │
  ▼
新しい ReplicaSet（v2）を作成し、Pod v2 を 1 つ起動
  │
  ▼
Pod v2 が Ready になったら、v1 の Pod を 1 つ削除
  │
  ▼
これを繰り返す
  │
  ▼
更新完了
  ReplicaSet（v2）→ Pod v2, Pod v2, Pod v2
  ReplicaSet（v1）→ （Pod なし、履歴として保持）
```

### [特徴の整理](#rolling-update-characteristics-summary) {#rolling-update-characteristics-summary}

{: .labeled}
| 観点 | 内容 |
| -------------- | --------------------------------------------------------------------- |
| ダウンタイム | なし（常に一定数の Pod がトラフィックを処理） |
| ロールバック | Deployment のリビジョン管理により、以前の ReplicaSet に段階的に戻せる |
| リソースコスト | 中程度（maxSurge の分だけ一時的に追加の Pod が存在する） |
| バージョン共存 | あり（更新中、旧バージョンと新バージョンが一時的に共存する） |

### [バージョン共存への注意](#version-coexistence-caution) {#version-coexistence-caution}

Rolling Update では、更新中に旧バージョンと新バージョンの Pod が<strong>一時的に共存</strong>します

つまり、同じ Service に対するリクエストが、v1 の Pod に届くこともあれば v2 の Pod に届くこともあります

これは、旧バージョンと新バージョンの間で<strong>互換性が必要</strong>であることを意味します

たとえば、API のレスポンス形式を変更する場合、v1 と v2 のどちらのレスポンスを受け取ってもクライアントが正しく動作する必要があります

### [制御パラメータ](#control-parameters) {#control-parameters}

宣言的構成管理で学んだ通り、Rolling Update の速度は 2 つのパラメータで制御します

{: .labeled}
| パラメータ | 説明 | デフォルト |
| -------------- | --------------------------------------------- | ---------- |
| maxSurge | あるべきレプリカ数を超えて追加できる Pod の数 | 25% |
| maxUnavailable | 更新中に利用不可になってよい Pod の数 | 25% |

Rolling Update は Kubernetes の Deployment の<strong>デフォルト戦略</strong>です

`strategy.type` を明示的に指定しない場合、Rolling Update が使用されます

---

## [Blue-Green デプロイ](#blue-green-deployment) {#blue-green-deployment}

### [仕組み](#blue-green-mechanism) {#blue-green-mechanism}

<strong>Blue-Green デプロイ</strong>は、旧バージョン（<strong>Blue</strong>）と新バージョン（<strong>Green</strong>）の 2 つの環境を同時に用意し、準備が整ったらトラフィックを一度に切り替える戦略です

```
準備段階
  Service → Blue（v1）の Pod 群にトラフィックを転送
  Green（v2）の Pod 群を別途起動して準備

                 ┌─── Blue（v1） ← トラフィック
  Service ──────┤
                 └─── Green（v2）（準備中、トラフィックなし）

切り替え
  Service のラベルセレクタを変更
  → Green（v2）の Pod 群にトラフィックを転送

                 ┌─── Blue（v1）（トラフィックなし）
  Service ──────┤
                 └─── Green（v2） ← トラフィック

切り替え完了後
  Blue（v1）の Pod 群を削除（または待機）
```

### [Service のラベルセレクタによる実現](#service-label-selector-implementation) {#service-label-selector-implementation}

Blue-Green デプロイは、[04-service-discovery](../../04-service-discovery/) や [07-declarative-management](../../07-declarative-management/) で学んだ<strong>ラベルセレクタ</strong>の仕組みを活用して実現できます

Service はラベルセレクタで対象の Pod を選択します

このラベルセレクタを変更することで、トラフィックの転送先を切り替えられます

<strong>Blue 環境の Pod</strong>

```yaml
metadata:
  labels:
    app: web
    version: v1
```

<strong>Green 環境の Pod</strong>

```yaml
metadata:
  labels:
    app: web
    version: v2
```

<strong>Service（切り替え前）</strong>

```yaml
spec:
  selector:
    app: web
    version: v1
```

<strong>Service（切り替え後）</strong>

```yaml
spec:
  selector:
    app: web
    version: v2
```

Service のセレクタを `version: v1` から `version: v2` に変更するだけで、トラフィックの転送先が Blue から Green に切り替わります

### [特徴](#blue-green-characteristics) {#blue-green-characteristics}

<strong>ダウンタイムなし</strong>

Green 環境が完全に準備できた状態でトラフィックを切り替えるため、ダウンタイムは発生しません

<strong>即時ロールバックが可能</strong>

問題が発覚した場合、Service のセレクタを元に戻すだけで、Blue 環境にトラフィックを戻せます

Blue 環境の Pod はまだ残っているため、ロールバックは即時に完了します

<strong>バージョン共存がない</strong>

Rolling Update と異なり、トラフィックが一度に切り替わるため、旧バージョンと新バージョンが同じリクエストを処理する期間がありません

<strong>リソースコストが高い</strong>

Blue 環境と Green 環境の両方を同時に稼働させるため、通常の 2 倍のリソースが必要です

### [Kubernetes のネイティブ機能ではない](#not-kubernetes-native) {#not-kubernetes-native}

Blue-Green デプロイは、Kubernetes の Deployment リソースの `strategy.type` として直接サポートされているわけではありません

ただし、Service のラベルセレクタと複数の Deployment を組み合わせることで実現できます

---

## [Canary デプロイ](#canary-deployment) {#canary-deployment}

### [名前の由来](#canary-name-origin) {#canary-name-origin}

<strong>Canary デプロイ</strong>の名前は、<strong>炭鉱のカナリア</strong>に由来します

かつて炭鉱労働者は、坑道にカナリア（小鳥）を連れて入りました

カナリアは有毒ガスに敏感であるため、カナリアに異変があれば危険を早期に検知できました

Canary デプロイも同じ考え方です

新バージョンの Pod を少数だけ先にデプロイし、全体に展開する前に問題を早期に検知します

### [仕組み](#canary-mechanism) {#canary-mechanism}

Canary デプロイでは、新バージョンの Pod を少数だけデプロイし、全トラフィックの一部だけを新バージョンに流します

問題がなければ徐々に新バージョンの割合を増やし、最終的に全 Pod を新バージョンに置き換えます

```
段階 1：少数の v2 Pod をデプロイ
  v1 Pod × 9、v2 Pod × 1（トラフィックの約 10% が v2 に流れる）

段階 2：問題がなければ v2 の割合を増やす
  v1 Pod × 7、v2 Pod × 3（トラフィックの約 30% が v2 に流れる）

段階 3：さらに増やす
  v1 Pod × 5、v2 Pod × 5（トラフィックの約 50% が v2 に流れる）

段階 4：全展開
  v1 Pod × 0、v2 Pod × 10（全トラフィックが v2 に流れる）
```

### [Kubernetes での実現方法](#canary-kubernetes-implementation) {#canary-kubernetes-implementation}

Kubernetes では、2 つの Deployment（旧バージョンと新バージョン）を同じ Service のラベルセレクタで束ねることで、Canary デプロイを実現できます

<strong>旧バージョンの Deployment</strong>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-v1
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web
      version: v1
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      containers:
        - name: web
          image: web-app:v1
```

<strong>新バージョンの Deployment（Canary）</strong>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      version: v2
  template:
    metadata:
      labels:
        app: web
        version: v2
    spec:
      containers:
        - name: web
          image: web-app:v2
```

<strong>Service</strong>

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

Service のセレクタは `app: web` のみを指定しています

`version` ラベルは指定していないため、v1 と v2 の両方の Pod がこの Service の対象になります

Kubernetes の標準 Service では、EndpointSlice に登録された各 Pod にほぼ均等にトラフィックが分配されます

v1 が 9 Pod、v2 が 1 Pod であれば、v2 に流れるトラフィックはおおよそ全体の 10% 程度になります

ただし、正確な比率で制御したい場合は、Istio などのサービスメッシュによる重み付けルーティングが一般的に使われます

新バージョンに問題がなければ、v2 の Deployment のレプリカ数を増やし、v1 のレプリカ数を減らしていきます

### [特徴](#canary-characteristics) {#canary-characteristics}

<strong>段階的にリスクを検証できる</strong>

新バージョンの問題は、全体の一部のトラフィックにしか影響しません

全ユーザーに影響が及ぶ前に問題を検知し、対処できます

<strong>影響範囲が小さい</strong>

問題が発覚した場合、少数の Canary Pod をロールバックするだけで済みます

大部分のユーザーは旧バージョンでサービスを利用し続けています

<strong>バージョン共存あり（制御下）</strong>

Rolling Update と同様に旧バージョンと新バージョンが共存しますが、Canary デプロイではその割合を明示的に制御できます

---

## [各戦略の比較](#strategies-comparison) {#strategies-comparison}

### [比較表](#comparison-table) {#comparison-table}

{: .labeled}
| 観点 | Recreate | Rolling Update | Blue-Green | Canary |
| ---------------- | ------------------------ | ---------------- | ---------------- | ---------------- |
| ダウンタイム | あり | なし | なし | なし |
| ロールバック速度 | 遅い（再デプロイ） | 中程度（段階的） | 即時（切り替え） | 速い（少数のみ） |
| リソースコスト | 低い | 中程度 | 高い（2 倍） | 中程度 |
| バージョン共存 | なし | あり（一時的） | なし | あり（制御下） |
| リスク | 高い（全ユーザーに影響） | 中程度 | 低い | 低い |
| 適したケース | DB 移行等 | 一般的な更新 | 即時切替が必要 | リスク検証 |

### [各戦略が適したケース](#suitable-cases-per-strategy) {#suitable-cases-per-strategy}

<strong>Recreate</strong>

ダウンタイムが許容でき、旧バージョンと新バージョンが同時に動作できない場合に適しています

データベースのスキーマ変更を伴うデプロイや、開発環境・テスト環境での更新が典型的な使用例です

<strong>Rolling Update</strong>

ダウンタイムなしで更新したい一般的なケースに適しています

旧バージョンと新バージョンの互換性が確保されている場合、最もバランスの取れた戦略です

Kubernetes の Deployment のデフォルト戦略であり、追加の設定なしで利用できます

<strong>Blue-Green</strong>

即時にトラフィックを切り替えたい場合や、即時ロールバックが必要な場合に適しています

バージョンの共存期間を避けたいが、ダウンタイムも許容できない場合に有効です

ただし、2 倍のリソースが必要なため、コストを考慮する必要があります

<strong>Canary</strong>

新バージョンのリスクを段階的に検証したい場合に適しています

大規模なサービスや、新バージョンの動作に不確実性がある場合に特に有効です

少数のユーザーで問題を検知してから全展開するため、全体への影響を最小限に抑えられます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| デプロイ戦略（Deployment Strategy） | アプリケーションの新バージョンをリリースする際の方法<br>ダウンタイム、ロールバック速度、リソースコスト、リスクなどのトレードオフに基づいて選択する |
| Recreate | 全ての既存 Pod を停止してから全ての新しい Pod を起動するデプロイ戦略<br>ダウンタイムが発生するが、旧バージョンと新バージョンの共存を避けられる |
| Rolling Update | 古い Pod を 1 つずつ新しい Pod に置き換えるデプロイ戦略<br>ダウンタイムなしで更新できるが、旧バージョンと新バージョンが一時的に共存する |
| Blue-Green デプロイ | 旧バージョン（Blue）と新バージョン（Green）の 2 つの環境を同時に用意し、トラフィックを一度に切り替えるデプロイ戦略 |
| Canary デプロイ | 新バージョンの Pod を少数だけデプロイし、トラフィックの一部で検証してから段階的に全展開するデプロイ戦略 |
| Blue 環境 | Blue-Green デプロイにおける旧バージョンの環境 |
| Green 環境 | Blue-Green デプロイにおける新バージョンの環境 |
| ダウンタイム（Downtime） | サービスがリクエストを処理できない期間 |
| ロールバック（Rollback） | 新バージョンに問題があった場合に、以前のバージョンに戻す操作 |
| トラフィック分配 | Service がリクエストを複数の Pod に振り分ける仕組み<br>Canary デプロイでは Pod 数の比率でトラフィックのおおよその割合を制御する |
| ラベルセレクタ（Label Selector） | Service が対象の Pod を選択するために使用するラベルの条件<br>Blue-Green デプロイではセレクタの変更でトラフィックの転送先を切り替える |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Kubernetes 公式ドキュメント</strong>

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/){:target="\_blank"}
  - Deployment のデプロイ戦略（Recreate、RollingUpdate）の仕様と動作の公式ドキュメント

- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/){:target="\_blank"}
  - ラベルとラベルセレクタの仕組みの公式ドキュメントで、Blue-Green デプロイや Canary デプロイの実現に使用される

- [Services](https://kubernetes.io/docs/concepts/services-networking/service/){:target="\_blank"}
  - Service の仕様と、ラベルセレクタによる Pod の選択の公式ドキュメント
