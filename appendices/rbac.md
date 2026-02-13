---
layout: default
title: RBAC（ロールベースアクセス制御）
---

# [appendix：RBAC（ロールベースアクセス制御）](#rbac) {#rbac}

## [はじめに](#introduction) {#introduction}

[02-architecture](../../02-architecture/) では、API Server がクラスタへのすべての操作を受け付ける唯一の窓口であることを学びました

API Server はリクエストの<strong>認証</strong>（正当なユーザーやコンポーネントからのリクエストかを確認）と<strong>認可</strong>（そのリクエストを実行する権限があるかを確認）を行うと述べました

しかし、認可の具体的な仕組みは説明していませんでした

クラスタ内には、すべてのリソースを操作できる管理者もいれば、特定の Namespace の Pod だけを参照できる開発者もいます

Pod 内で動作するアプリケーションが API Server に問い合わせるケースもあります

それぞれに対して、どのような操作を許可し、どのような操作を拒否するのか

この補足資料では、Kubernetes の標準的な認可の仕組みである <strong>RBAC（Role-Based Access Control）</strong> を学びます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

- <strong>認証と認可の違い</strong>
  - 「誰であるか」を確認する認証と「何ができるか」を確認する認可の区別
- <strong>RBAC とは</strong>
  - 役割（ロール）に権限を割り当て、その役割をユーザーに割り当てるアクセス制御モデル
- <strong>4 つのリソース</strong>
  - Role、ClusterRole、RoleBinding、ClusterRoleBinding の役割と使い分け
- <strong>Role の定義</strong>
  - 動詞（操作）と対象リソースの組み合わせによる権限の表現
- <strong>ServiceAccount</strong>
  - Pod が API Server に認証される仕組み
- <strong>最小権限の原則</strong>
  - 必要最小限の権限のみを付与するセキュリティ原則

---

## [目次](#table-of-contents) {#table-of-contents}

1. [認証と認可の違い](#authentication-and-authorization-difference)
2. [RBAC とは](#what-is-rbac)
3. [4 つのリソース](#resource-types)
4. [Role の定義](#role-definition)
5. [ServiceAccount](#service-account)
6. [最小権限の原則](#least-privilege-principle)
7. [用語集](#glossary)
8. [参考資料](#references)

---

## [認証と認可の違い](#authentication-and-authorization-difference) {#authentication-and-authorization-difference}

### [認証（Authentication）](#authentication) {#authentication}

<strong>認証</strong>は、「<strong>誰であるか</strong>」を確認するプロセスです

API Server にリクエストが届いたとき、そのリクエストが誰から来たものかを特定します

認証の結果、リクエストの送信元が「管理者の Alice」なのか「開発者の Bob」なのか「Pod 内のアプリケーション」なのかが判明します

### [認可（Authorization）](#authorization) {#authorization}

<strong>認可</strong>は、「<strong>何ができるか</strong>」を確認するプロセスです

認証で「誰か」が特定された後、その人が「要求した操作を実行する権限があるか」を判断します

たとえば、「開発者の Bob が default Namespace の Pod を一覧取得しようとしている」というリクエストに対して、Bob にその権限があるかを確認します

### [2 つのプロセスの関係](#two-processes-relationship) {#two-processes-relationship}

```
リクエスト到着
  │
  ▼
認証（Authentication）
  「このリクエストは誰から？」→「開発者の Bob」
  │
  ▼
認可（Authorization）
  「Bob は default Namespace の Pod を一覧取得できるか？」→「許可 / 拒否」
  │
  ▼
リクエストの実行 または 拒否
```

認証と認可は別のプロセスです

認証に成功しても、認可で拒否されればリクエストは実行されません

RBAC は、この認可のステップで使われる仕組みです

---

## [RBAC とは](#what-is-rbac) {#what-is-rbac}

### [ロールベースアクセス制御](#role-based-access-control) {#role-based-access-control}

<strong>RBAC（Role-Based Access Control）</strong>は、<strong>役割（ロール）に権限を割り当て、その役割をユーザーに割り当てる</strong>アクセス制御モデルです

権限をユーザーに直接割り当てるのではなく、「ロール」という中間層を介して管理します

```
権限の定義        役割の割り当て
┌──────────┐     ┌────────────┐     ┌──────────┐
│ 権限     │ ──→ │ ロール     │ ──→ │ ユーザー │
│ Pod 参照 │     │ pod-reader │     │ Bob      │
│ Pod 一覧 │     │            │     │          │
└──────────┘     └────────────┘     └──────────┘
```

この設計には利点があります

複数のユーザーに同じ権限を与えたい場合、個別に権限を設定する代わりに、1 つのロールを定義してそれぞれのユーザーに割り当てるだけで済みます

ロールの権限を変更すれば、そのロールが割り当てられたすべてのユーザーの権限が一括で変更されます

### [許可のみの加算モデル](#additive-permission-model) {#additive-permission-model}

Kubernetes の RBAC は<strong>加算モデル</strong>で動作します

「この操作を拒否する」という規則は存在しません

権限はゼロから始まり、ロールによって必要な権限だけが追加されていきます

複数のロールが割り当てられた場合、すべてのロールの権限の和集合が適用されます

---

## [4 つのリソース](#resource-types) {#resource-types}

RBAC は 4 つの API リソースで構成されます

これらのリソースは「権限を定義するもの」と「権限を割り当てるもの」の 2 種類に分かれます

{: .labeled}
| 種類 | Namespace スコープ | クラスタスコープ |
| -------------- | ------------------ | ------------------ |
| 権限の定義 | Role | ClusterRole |
| 権限の割り当て | RoleBinding | ClusterRoleBinding |

### [Role（Namespace スコープの権限定義）](#role) {#role}

<strong>Role</strong> は、<strong>特定の Namespace 内で有効な権限のセット</strong>を定義します

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

この Role は、`default` Namespace 内の Pod に対する読み取り操作（取得、監視、一覧）を許可します

Role は 1 つの Namespace に閉じており、他の Namespace のリソースには影響しません

### [ClusterRole（クラスタスコープの権限定義）](#cluster-role) {#cluster-role}

<strong>ClusterRole</strong> は、<strong>クラスタ全体で有効な権限のセット</strong>を定義します

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "watch", "list"]
```

ClusterRole は Namespace を持ちません

Node のようなクラスタスコープのリソースに対する権限は、ClusterRole でのみ定義できます

ClusterRole は RoleBinding から参照することもでき、その場合は RoleBinding の Namespace 内に限定した権限として機能します

### [RoleBinding（Namespace スコープの権限割り当て）](#role-binding) {#role-binding}

<strong>RoleBinding</strong> は、<strong>Role または ClusterRole の権限を、特定の Namespace 内でユーザーに割り当てる</strong>リソースです

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

この RoleBinding は、ユーザー `bob` に `pod-reader` Role の権限を `default` Namespace 内で付与します

`bob` は `default` Namespace の Pod を読み取ることができますが、他の Namespace の Pod にはアクセスできません

### [ClusterRoleBinding（クラスタスコープの権限割り当て）](#cluster-role-binding) {#cluster-role-binding}

<strong>ClusterRoleBinding</strong> は、<strong>ClusterRole の権限を、クラスタ全体でユーザーに割り当てる</strong>リソースです

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

この ClusterRoleBinding は、ユーザー `alice` に `node-reader` ClusterRole の権限をクラスタ全体で付与します

### [4 つのリソースの関係](#resource-types-relationship) {#resource-types-relationship}

```
┌─────────────────────────────────────────────────────┐
│                    クラスタ全体                       │
│                                                     │
│  ClusterRole ── ClusterRoleBinding ──→ ユーザー     │
│  （権限定義）    （クラスタ全体で割り当て）            │
│                                                     │
│  ┌──────────────────────────────────┐                │
│  │     Namespace: default           │                │
│  │                                  │                │
│  │  Role ── RoleBinding ──→ ユーザー│                │
│  │  （権限定義） （この NS 内で割り当て）│              │
│  │                                  │                │
│  └──────────────────────────────────┘                │
│                                                     │
│  ┌──────────────────────────────────┐                │
│  │     Namespace: production        │                │
│  │                                  │                │
│  │  Role ── RoleBinding ──→ ユーザー│                │
│  │                                  │                │
│  └──────────────────────────────────┘                │
└─────────────────────────────────────────────────────┘
```

---

## [Role の定義](#role-definition) {#role-definition}

### [動詞と対象リソースの組み合わせ](#verb-and-resource-combination) {#verb-and-resource-combination}

Role（および ClusterRole）の権限は、<strong>動詞（verb）</strong>と<strong>対象リソース</strong>の組み合わせで表現されます

動詞は「何をするか」、リソースは「何に対して」を表します

### [主要な動詞](#major-verbs) {#major-verbs}

{: .labeled}
| 動詞 | 説明 |
| ------ | -------------------------------------- |
| get | 指定した名前のリソースを 1 つ取得する |
| list | リソースの一覧を取得する |
| watch | リソースの変更をリアルタイムに監視する |
| create | 新しいリソースを作成する |
| update | 既存のリソースを更新する |
| patch | 既存のリソースの一部を変更する |
| delete | リソースを削除する |

読み取り専用の権限を与えるには `get`、`list`、`watch` を指定します

書き込み権限も含める場合は `create`、`update`、`patch`、`delete` を追加します

### [apiGroups](#api-groups) {#api-groups}

Kubernetes のリソースは API グループに分類されています

{: .labeled}
| API グループ | 含まれるリソース |
| --------------------- | --------------------------------------------------------- |
| `""`（空文字列） | Pod、Service、Secret、ConfigMap など（コア API グループ） |
| `"apps"` | Deployment、ReplicaSet、DaemonSet など |
| `"autoscaling"` | HorizontalPodAutoscaler など |
| `"networking.k8s.io"` | NetworkPolicy など |

Role の `rules` で `apiGroups` を指定することで、その API グループに属するリソースに対する権限を定義します

### [複数の規則を持つ Role](#role-with-multiple-rules) {#role-with-multiple-rules}

1 つの Role に複数の `rules` を定義できます

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: app-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

この Role は、`production` Namespace 内で以下の権限を定義しています

- Deployment の読み取りと更新
- Pod の読み取り
- Pod のログの参照

---

## [ServiceAccount](#service-account) {#service-account}

### [Pod が API Server にアクセスする仕組み](#pod-api-server-access) {#pod-api-server-access}

[02-architecture](../../02-architecture/) で学んだように、kubelet や Controller Manager は API Server と通信します

Pod 内のアプリケーションも API Server にアクセスする必要がある場合があります

たとえば、アプリケーションが「自分と同じ Namespace にある ConfigMap を読み取る」といった操作を行う場合です

このとき、Pod はどうやって API Server に認証されるのでしょうか

### [ServiceAccount とは](#what-is-service-account) {#what-is-service-account}

<strong>ServiceAccount</strong> は、<strong>Pod が API Server に認証されるためのアイデンティティ（身元）</strong>です

ユーザーアカウントが人間のユーザーを表すのに対し、ServiceAccount は Pod 内で動作するプロセスを表します

すべての Namespace には `default` という名前の ServiceAccount が自動的に作成されます

Pod が明示的に ServiceAccount を指定しない場合、`default` ServiceAccount が使用されます

### [ServiceAccount と RBAC の組み合わせ](#service-account-and-rbac) {#service-account-and-rbac}

ServiceAccount は RBAC の<strong>subjects（対象）</strong>として使用できます

つまり、RoleBinding や ClusterRoleBinding で ServiceAccount にロールを割り当てることができます

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: my-app
    namespace: production
roleRef:
  kind: Role
  name: app-manager
  apiGroup: rbac.authorization.k8s.io
```

この構成では、`production` Namespace の `my-app` ServiceAccount に `app-manager` Role の権限が付与されます

この ServiceAccount を使用する Pod は、`app-manager` Role で定義された操作（Deployment の読み取りと更新、Pod の読み取りなど）を実行できます

### [subjects の種類](#subjects-types) {#subjects-types}

RoleBinding と ClusterRoleBinding で指定できる subjects は 3 種類あります

{: .labeled}
| 種類 | 説明 |
| -------------- | ------------------------ |
| User | 個人のユーザーアカウント |
| Group | ユーザーのグループ |
| ServiceAccount | Pod のアイデンティティ |

---

## [最小権限の原則](#least-privilege-principle) {#least-privilege-principle}

### [必要最小限の権限のみ付与する](#grant-minimum-necessary-privileges) {#grant-minimum-necessary-privileges}

<strong>最小権限の原則（Principle of Least Privilege）</strong>は、「ユーザーやプロセスには、その役割を果たすために必要な最小限の権限のみを付与する」というセキュリティの原則です

RBAC を設計する際には、この原則に従うことが重要です

### [最小権限の実践](#least-privilege-practice) {#least-privilege-practice}

{: .labeled}
| 実践 | 説明 |
| ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| 必要な動詞のみを許可する | 読み取りだけで十分なら `get` と `list` のみ。`delete` や `update` は不要なら付与しない |
| 必要なリソースのみを対象にする | すべてのリソースではなく、実際にアクセスが必要なリソースだけを指定する |
| Namespace スコープを優先する | ClusterRole と ClusterRoleBinding よりも、Role と RoleBinding を使い、影響範囲を Namespace 内に限定する |
| 専用の ServiceAccount を使う | `default` ServiceAccount を使い回さず、アプリケーションごとに専用の ServiceAccount を作成する |

### [default ServiceAccount のリスク](#default-service-account-risk) {#default-service-account-risk}

すべての Namespace に自動作成される `default` ServiceAccount は、デフォルトでは最低限の権限（基本的なクラスタ情報の参照のみ）しか持ちません

しかし、`default` ServiceAccount に広い権限を付与してしまうと、その Namespace 内のすべての Pod がその権限を持つことになります

アプリケーションごとに専用の ServiceAccount を作成し、必要な権限だけを付与することで、影響範囲を限定できます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| RBAC（Role-Based Access Control） | 役割（ロール）に権限を割り当て、その役割をユーザーに割り当てるアクセス制御モデル。Kubernetes の標準的な認可の仕組み |
| 認証（Authentication） | リクエストの送信元が「誰であるか」を確認するプロセス |
| 認可（Authorization） | 認証されたユーザーが「要求した操作を実行する権限があるか」を確認するプロセス |
| Role | 特定の Namespace 内で有効な権限のセットを定義するリソース |
| ClusterRole | クラスタ全体で有効な権限のセットを定義するリソース。Namespace を持たない |
| RoleBinding | Role または ClusterRole の権限を、特定の Namespace 内でユーザーに割り当てるリソース |
| ClusterRoleBinding | ClusterRole の権限を、クラスタ全体でユーザーに割り当てるリソース |
| ServiceAccount | Pod が API Server に認証されるためのアイデンティティ。ユーザーアカウントとは異なり、Pod 内のプロセスを表す |
| 動詞（Verb） | RBAC の権限定義で使用される操作の種類。get、list、watch、create、update、patch、delete など |
| apiGroups | Kubernetes のリソースが属する API グループ。Role の権限定義で対象リソースの API グループを指定する |
| subjects | RoleBinding や ClusterRoleBinding で権限を割り当てる対象。User、Group、ServiceAccount の 3 種類がある |
| 最小権限の原則（Principle of Least Privilege） | ユーザーやプロセスに、役割を果たすために必要な最小限の権限のみを付与するセキュリティ原則 |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Kubernetes 公式ドキュメント</strong>

- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/){:target="\_blank"}
  - RBAC の 4 つのリソース（Role、ClusterRole、RoleBinding、ClusterRoleBinding）の仕様と使用方法の公式ドキュメント

- [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/){:target="\_blank"}
  - ServiceAccount の管理と Pod の認証の公式ドキュメント
