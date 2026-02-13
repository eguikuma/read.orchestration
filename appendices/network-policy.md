---
layout: default
title: Network Policy（Pod 間通信の制御）
---

# [appendix：Network Policy（Pod 間通信の制御）](#network-policy) {#network-policy}

## [はじめに](#introduction) {#introduction}

[04-service-discovery](../../04-service-discovery/) では、Kubernetes のネットワークモデルを学びました

すべての Pod が固有の IP アドレスを持ち、NAT なしで他のすべての Pod と通信できるという要件を確認しました

この「すべての Pod が互いに自由に通信できる」という性質は、Pod 間の連携を容易にします

しかし、裏を返せば、<strong>あらゆる Pod が他のあらゆる Pod にアクセスできる</strong>ということでもあります

たとえば、フロントエンドの Pod がデータベースの Pod に直接アクセスできてしまいます

本来、データベースにアクセスするのはバックエンドの Pod だけであるべきです

もし不正なコードがフロントエンドの Pod で実行された場合、データベースに直接アクセスされるリスクがあります

この補足資料では、Pod 間の通信を制御する仕組みである <strong>Network Policy</strong> を学びます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

- <strong>デフォルトの動作</strong>
  - Kubernetes ではすべての Pod 間通信がデフォルトで許可されていること
- <strong>Network Policy とは</strong>
  - ラベルセレクタに基づいて Pod の通信を制御するリソース
- <strong>Ingress と Egress</strong>
  - 受信（Ingress）と送信（Egress）の規則の仕組み
- <strong>Namespace セレクタ</strong>
  - Namespace をまたぐ通信の制御方法
- <strong>デフォルト拒否パターン</strong>
  - 全通信を拒否してから必要な通信だけを許可する設計
- <strong>CNI プラグインとの関係</strong>
  - Network Policy の実行には対応する CNI プラグインが必要であること

---

## [目次](#table-of-contents) {#table-of-contents}

1. [デフォルトの動作](#default-behavior)
2. [Network Policy とは](#what-is-network-policy)
3. [Ingress と Egress](#ingress-and-egress)
4. [Namespace セレクタ](#namespace-selector)
5. [デフォルト拒否パターン](#default-deny-pattern)
6. [CNI プラグインとの関係](#cni-plugin-relationship)
7. [用語集](#glossary)
8. [参考資料](#references)

---

## [デフォルトの動作](#default-behavior) {#default-behavior}

### [すべての Pod 間通信が許可される](#all-pod-communication-allowed) {#all-pod-communication-allowed}

Kubernetes のデフォルトの動作では、<strong>すべての Pod が他のすべての Pod と自由に通信できます</strong>

これは Namespace をまたいでも同様です

`default` Namespace の Pod が `production` Namespace の Pod に自由にアクセスできます

```
Namespace: default            Namespace: production
┌──────────────┐              ┌──────────────┐
│ frontend Pod │ ────────────→│ database Pod │
└──────────────┘   自由に通信  └──────────────┘
```

この動作は Kubernetes のネットワークモデルの基本原則に基づいています

[04-service-discovery](../../04-service-discovery/) で学んだ「すべての Pod は NAT なしで他のすべての Pod と通信できる」という要件がそのまま適用されています

### [なぜこれが問題になるか](#why-this-is-a-problem) {#why-this-is-a-problem}

すべての Pod が自由に通信できることは、開発環境や小規模なクラスタでは便利です

しかし、本番環境では以下のリスクがあります

{: .labeled}
| リスク | 説明 |
| ---------------- | --------------------------------------------------------------------------------------- |
| 不正アクセス | 本来アクセスすべきでない Pod からデータベースや内部サービスにアクセスされる可能性がある |
| 影響範囲の拡大 | 1 つの Pod が侵害された場合、クラスタ内のすべての Pod にアクセスできてしまう |
| 環境間の分離不足 | 開発環境の Pod が本番環境のデータベースに誤ってアクセスする可能性がある |

これらのリスクに対処するために、Pod 間の通信を明示的に制御する仕組みが必要です

---

## [Network Policy とは](#what-is-network-policy) {#what-is-network-policy}

### [ラベルセレクタによるトラフィック制御](#label-selector-traffic-control) {#label-selector-traffic-control}

<strong>Network Policy</strong> は、<strong>ラベルセレクタに基づいて Pod の通信を制御する Kubernetes リソース</strong>です

[04-service-discovery](../../04-service-discovery/) や [07-declarative-management](../../07-declarative-management/) で学んだラベルセレクタの仕組みを、ネットワークのアクセス制御に応用したものです

Network Policy は「どの Pod が」「どの方向の」「どの Pod からの（あるいはどの Pod への）」通信を許可するかを定義します

### [隔離の仕組み](#isolation-mechanism) {#isolation-mechanism}

Kubernetes では、<strong>Network Policy が選択していない Pod は「非隔離（non-isolated）」状態</strong>です

非隔離の Pod はすべての通信が許可されます

<strong>Network Policy に選択された Pod は「隔離（isolated）」状態</strong>になります

隔離された Pod は、Network Policy で明示的に許可された通信のみが可能になります

隔離は方向ごとに独立しています

{: .labeled}
| 方向 | 隔離条件 | 隔離時の動作 |
| --------------- | -------------------------------------------------------------- | -------------------------- |
| Ingress（受信） | `policyTypes` に `Ingress` を含む Network Policy が Pod を選択 | 許可された通信のみ受信可能 |
| Egress（送信） | `policyTypes` に `Egress` を含む Network Policy が Pod を選択 | 許可された通信のみ送信可能 |

### [基本的なマニフェスト](#basic-manifest) {#basic-manifest}

Network Policy のマニフェストの基本構造を見てみましょう

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 8080
```

この Network Policy は以下を宣言しています

- `podSelector`：`role: backend` ラベルを持つ Pod を対象とする
- `policyTypes`：Ingress（受信）方向を制御する
- `ingress`：`role: frontend` ラベルを持つ Pod からの TCP ポート 8080 への通信のみを許可する

つまり、バックエンドの Pod はフロントエンドの Pod からの通信だけを受け付け、それ以外の Pod からの通信は拒否されます

### [許可リスト方式](#allowlist-method) {#allowlist-method}

Network Policy は<strong>許可リスト方式（ホワイトリスト方式）</strong>で動作します

「この通信を拒否する」という規則は書けません

代わりに、Pod を隔離した上で「この通信を許可する」という規則だけを定義します

明示的に許可されていない通信はすべて拒否されます

複数の Network Policy が同じ Pod を選択する場合、すべての Policy の許可規則の<strong>和集合</strong>が適用されます

ある Policy が許可する通信を、別の Policy が拒否することはできません

---

## [Ingress と Egress](#ingress-and-egress) {#ingress-and-egress}

### [Ingress（受信規則）](#ingress) {#ingress}

<strong>Ingress</strong> は、Pod への<strong>受信トラフィック</strong>を制御する規則です

「この Pod は、どの Pod からの通信を受け付けるか」を定義します

```yaml
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: backend
      ports:
        - protocol: TCP
          port: 5432
```

この例では、`role: db` ラベルを持つデータベースの Pod は、`role: backend` ラベルを持つバックエンドの Pod からの TCP ポート 5432（PostgreSQL のデフォルトポート）への通信のみを受け付けます

フロントエンドの Pod やその他の Pod からのアクセスは拒否されます

### [Egress（送信規則）](#egress) {#egress}

<strong>Egress</strong> は、Pod からの<strong>送信トラフィック</strong>を制御する規則です

「この Pod は、どの宛先に通信を送ることができるか」を定義します

```yaml
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              role: db
      ports:
        - protocol: TCP
          port: 5432
```

この例では、`role: backend` ラベルを持つバックエンドの Pod は、`role: db` ラベルを持つデータベースの Pod の TCP ポート 5432 にのみ通信を送ることができます

### [Ingress と Egress の組み合わせ](#ingress-and-egress-combination) {#ingress-and-egress-combination}

通信が成功するためには、<strong>送信元の Egress 規則と宛先の Ingress 規則の両方で許可されている</strong>必要があります

バックエンドからデータベースへの通信を許可するには、以下の両方が必要です

- バックエンドの Pod の Egress 規則でデータベースへの送信を許可する
- データベースの Pod の Ingress 規則でバックエンドからの受信を許可する

```
backend Pod                           db Pod
┌───────────────┐                     ┌───────────────┐
│               │ ─── Egress 許可 ──→ │               │
│ Egress 規則   │                     │ Ingress 規則  │
│ 宛先: db Pod  │                     │ 送信元: backend│
└───────────────┘                     └───────────────┘
```

片方だけの規則では通信は成功しません

---

## [Namespace セレクタ](#namespace-selector) {#namespace-selector}

### [Namespace をまたぐ通信の制御](#cross-namespace-communication-control) {#cross-namespace-communication-control}

[07-declarative-management](../../07-declarative-management/) で学んだように、Namespace はリソースを論理的にグループ化する仕組みです

Network Policy では、<strong>namespaceSelector</strong> を使って Namespace をまたぐ通信を制御できます

### [namespaceSelector の使い方](#how-to-use-namespace-selector) {#how-to-use-namespace-selector}

```yaml
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              environment: production
          podSelector:
            matchLabels:
              role: backend
      ports:
        - protocol: TCP
          port: 5432
```

この例では、`namespaceSelector` と `podSelector` が<strong>同じブロック内</strong>に記述されています

この場合、2 つの条件は AND（かつ）で結合されます

`environment: production` ラベルを持つ Namespace 内の `role: backend` ラベルを持つ Pod からの通信のみが許可されます

### [OR と AND の違い](#or-and-and-difference) {#or-and-and-difference}

`from` リスト内での記述方法によって、条件の組み合わせが変わります

<strong>AND 条件（同じブロック内）</strong>

```yaml
from:
  - namespaceSelector:
      matchLabels:
        environment: production
    podSelector:
      matchLabels:
        role: backend
```

「`production` Namespace 内の `backend` Pod」からの通信のみ許可

<strong>OR 条件（別のブロック）</strong>

```yaml
from:
  - namespaceSelector:
      matchLabels:
        environment: production
  - podSelector:
      matchLabels:
        role: backend
```

「`production` Namespace 内のすべての Pod」または「同じ Namespace 内の `backend` Pod」からの通信を許可

この区別は Network Policy を書く上で特に重要です

同じブロック内に記述すれば AND 条件、別のブロックに分ければ OR 条件になります

---

## [デフォルト拒否パターン](#default-deny-pattern) {#default-deny-pattern}

### [全通信を拒否してから許可する](#deny-all-then-allow) {#deny-all-then-allow}

セキュリティの原則として、<strong>デフォルト拒否（Default Deny）</strong>パターンがあります

まずすべての通信を拒否し、必要な通信だけを明示的に許可する方法です

### [Ingress のデフォルト拒否](#ingress-default-deny) {#ingress-default-deny}

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

`podSelector: {}` は<strong>空のセレクタ</strong>であり、Namespace 内のすべての Pod を選択します

`ingress` フィールドを省略しているため、許可される受信通信はありません

つまり、この Namespace 内のすべての Pod に対する受信通信がデフォルトで拒否されます

### [Egress のデフォルト拒否](#egress-default-deny) {#egress-default-deny}

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

同様に、この Namespace 内のすべての Pod からの送信通信がデフォルトで拒否されます

### [デフォルト拒否 + 個別許可](#default-deny-plus-individual-allow) {#default-deny-plus-individual-allow}

デフォルト拒否の Network Policy を適用した上で、必要な通信だけを許可する Network Policy を追加します

```
1. default-deny-ingress（全受信拒否）
2. default-deny-egress（全送信拒否）
3. allow-frontend-to-backend（frontend → backend の通信を許可）
4. allow-backend-to-db（backend → db の通信を許可）
```

このパターンにより、意図した通信だけが許可され、予期しない通信経路が遮断されます

前のシリーズでアクセス制御を学んだ方は、「デフォルトで拒否し、必要なものだけを許可する」という原則を思い出すかもしれません

Network Policy はこの原則をネットワーク通信に適用したものです

---

## [CNI プラグインとの関係](#cni-plugin-relationship) {#cni-plugin-relationship}

### [Network Policy と CNI プラグイン](#network-policy-and-cni-plugin) {#network-policy-and-cni-plugin}

Network Policy は Kubernetes API のリソースとして定義されますが、Network Policy の<strong>実行（エンフォースメント）</strong>は Kubernetes 本体ではなく<strong>CNI プラグイン</strong>が行います

[CNI](../cni/) の appendix で学んだように、CNI プラグインは Pod のネットワーク構成を担当する仕組みです

すべての CNI プラグインが Network Policy をサポートしているわけではありません

<strong>Network Policy をサポートしない CNI プラグインを使っている場合、Network Policy リソースを作成しても通信は制御されません</strong>

Network Policy が定義されているが CNI プラグインがそれを実行しない場合、エラーは発生せず、すべての通信が引き続き許可されます

### [CNI プラグインの対応状況](#cni-plugin-support-status) {#cni-plugin-support-status}

{: .labeled}
| CNI プラグイン | Network Policy サポート |
| -------------- | ----------------------- |
| Calico | サポートあり |
| Cilium | サポートあり |
| Weave Net | サポートあり |
| Flannel | サポートなし |

[CNI](../cni/) の appendix で学んだように、CNI プラグインの選択はクラスタの運用要件に応じた判断です

Network Policy による通信制御が必要な場合は、Network Policy をサポートする CNI プラグインを選択する必要があります

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| Network Policy | ラベルセレクタに基づいて Pod の通信を制御する Kubernetes リソース<br>許可リスト方式で動作し、明示的に許可されていない通信は拒否される |
| 非隔離（Non-isolated） | Network Policy に選択されていない Pod の状態<br>すべての通信が許可される |
| 隔離（Isolated） | Network Policy に選択された Pod の状態<br>明示的に許可された通信のみが可能になる |
| Ingress | Pod への受信トラフィックの方向<br>Network Policy の Ingress 規則で制御する |
| Egress | Pod からの送信トラフィックの方向<br>Network Policy の Egress 規則で制御する |
| podSelector | Network Policy の対象となる Pod をラベルで選択する仕組み<br>空のセレクタは Namespace 内のすべての Pod を選択する |
| namespaceSelector | 通信を許可する Namespace をラベルで選択する仕組み<br>Namespace をまたぐ通信の制御に使用する |
| policyTypes | Network Policy が制御する方向（Ingress、Egress、またはその両方）を指定するフィールド |
| デフォルト拒否（Default Deny） | すべての通信をデフォルトで拒否し、必要な通信だけを明示的に許可するセキュリティパターン |
| 許可リスト方式（ホワイトリスト方式） | 許可する通信のみを定義し、それ以外を拒否する方式<br>Network Policy はこの方式で動作する |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Kubernetes 公式ドキュメント</strong>

- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/){:target="\_blank"}
  - Network Policy の概念、マニフェストの仕様、隔離の仕組みの公式ドキュメント
