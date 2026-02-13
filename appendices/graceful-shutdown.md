<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# appendix：Pod の安全な終了

## はじめに

[05-self-healing](../05-self-healing.md) では、障害を自動で検知し、あるべき状態に向けて復旧するセルフヒーリングの仕組みを学びました

kubelet がコンテナの異常を検知して再起動し、ReplicaSet コントローラが Pod の不足を補い、Node コントローラがノード障害時に Pod を退避させる、3 層の自動復旧メカニズムを確認しました

[07-declarative-management](../07-declarative-management.md) では、ローリングアップデートの仕組みを学びました

古い Pod を 1 つずつ新しい Pod に置き換え、ダウンタイムなしでアプリケーションを更新できることを確認しました

しかし、ここで新たな疑問が生まれます

Pod が削除されるとき、その Pod が処理中のリクエストはどうなるのでしょうか？

ローリングアップデートで古い Pod が置き換えられるとき、そのリクエストは途中で切断されてしまうのでしょうか？

EndpointSlice から Pod が除外されるタイミングと、Pod が実際に終了するタイミングにずれがあったらどうなるのでしょうか？

この appendix では、Pod が安全に終了するための仕組みである<strong>Graceful Shutdown（グレースフルシャットダウン）</strong>を学びます

Pod の削除から実際の終了までのシーケンスと、処理中のリクエストを失わないための設計を見ていきます

---

## このページで学ぶこと

- <strong>Pod 削除時のシーケンス</strong>
  - API Server から kubelet を経てコンテナが終了するまでの全体の流れ
- <strong>Termination Grace Period</strong>
  - SIGTERM と SIGKILL の関係、猶予期間の仕組み
- <strong>EndpointSlice からの除外タイミング</strong>
  - Pod 削除時に発生するレース条件とその原因
- <strong>preStop フック</strong>
  - レース条件を緩和するためのライフサイクルフックの仕組み
- <strong>アプリケーション側の対応</strong>
  - SIGTERM ハンドリングとコネクションドレインの設計
- <strong>ローリングアップデートとの関係</strong>
  - Deployment による更新時に Graceful Shutdown がどう機能するか

---

## 目次

1. [Pod 削除時のシーケンス](#pod-削除時のシーケンス)
2. [Termination Grace Period](#termination-grace-period)
3. [EndpointSlice からの除外タイミング](#endpointslice-からの除外タイミング)
4. [preStop フック](#prestop-フック)
5. [アプリケーション側の対応](#アプリケーション側の対応)
6. [ローリングアップデートとの関係](#ローリングアップデートとの関係)
7. [用語集](#用語集)
8. [参考資料](#参考資料)

---

## Pod 削除時のシーケンス

### 削除の全体像

Pod が削除されるとき、Kubernetes の複数のコンポーネントが連携して終了処理を行います

削除の起点はさまざまです

管理者が手動で Pod を削除する場合、ローリングアップデートで古い Pod が置き換えられる場合、スケールダウンでレプリカ数が減る場合など、いずれの場合も同じ終了シーケンスが実行されます

### シーケンスの流れ

Pod の削除は、以下の流れで進みます

<strong>1. API Server が削除リクエストを受け付ける</strong>

管理者やコントローラが Pod の削除をリクエストすると、API Server がこれを受け付けます

API Server は Pod の状態を「Terminating」に更新し、etcd に保存します

<strong>2. 2 つの通知が並行して発行される</strong>

API Server の状態更新をきっかけに、以下の 2 つの処理が<strong>並行して</strong>開始されます

- kubelet への通知：Pod の終了処理を開始する
- EndpointSlice コントローラへの通知：Pod を EndpointSlice から除外する

この「並行して」という点が、後述するレース条件の原因になります

<strong>3. kubelet が終了処理を実行する</strong>

kubelet は Pod 内のコンテナに対して終了処理を実行します

具体的には、preStop フック（設定されている場合）を実行し、その後コンテナに SIGTERM を送信します

<strong>4. コンテナが終了する</strong>

コンテナ内のアプリケーションは SIGTERM を受け取り、終了処理を行います

猶予期間内にコンテナが終了しなければ、SIGKILL で強制終了されます

```
Pod の削除リクエスト
  │
  ▼
API Server が Pod を「Terminating」に更新
  │
  ├──→ kubelet への通知
  │      │
  │      ▼
  │    preStop フックを実行（設定されている場合）
  │      │
  │      ▼
  │    コンテナに SIGTERM を送信
  │      │
  │      ▼
  │    猶予期間内に終了しなければ SIGKILL
  │
  └──→ EndpointSlice コントローラへの通知
         │
         ▼
       Pod を EndpointSlice から除外
         │
         ▼
       kube-proxy がルールを更新
```

---

## Termination Grace Period

### SIGTERM と SIGKILL

前のシリーズでユーザー空間を学んだ方は、シグナルの仕組みを思い出すかもしれません

<strong>SIGTERM</strong>は、プロセスに「終了してください」と<strong>依頼</strong>するシグナルです

プロセスは SIGTERM を受け取ると、終了処理を行ってから自発的に終了できます

たとえば、処理中のリクエストを完了させたり、開いているファイルを閉じたり、データベースへの接続を切断したりする時間が与えられます

<strong>SIGKILL</strong>は、プロセスを<strong>強制的に終了</strong>するシグナルです

プロセスは SIGKILL を受け取ることができず、終了処理を行う機会もありません

即座にプロセスが停止します

### 猶予期間の仕組み

kubelet がコンテナに SIGTERM を送信してから SIGKILL を送信するまでの時間を、<strong>Termination Grace Period（終了猶予期間）</strong>と呼びます

この猶予期間は、Pod のマニフェストで <strong>terminationGracePeriodSeconds</strong> として設定されます

デフォルト値は<strong>30 秒</strong>です

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: web
      image: web-app:v1
```

上記の設定では、SIGTERM の送信後 60 秒以内にコンテナが終了しなければ、SIGKILL で強制終了されます

### 猶予期間のタイムライン

猶予期間は、preStop フックの実行時間も含みます

```
├─────────── terminationGracePeriodSeconds ──────────────┤

│  preStop フック  │     SIGTERM 後のアプリ終了処理      │ SIGKILL
│   の実行時間     │                                     │
```

preStop フックに 10 秒かかる場合、SIGTERM を受けたアプリケーションが使える時間は残りの 20 秒です（デフォルト 30 秒の場合）

preStop フックの実行時間が長すぎると、アプリケーションの終了処理に使える時間が短くなる点に注意が必要です

---

## EndpointSlice からの除外タイミング

### レース条件の発生

前述の通り、Pod が「Terminating」に更新されると、kubelet への通知と EndpointSlice コントローラへの通知が<strong>並行して</strong>発行されます

ここで問題になるのが、<strong>EndpointSlice の更新が実際のトラフィック転送に反映されるまでにタイムラグがある</strong>ことです

EndpointSlice から Pod が除外されても、その変更が kube-proxy に反映され、iptables や IPVS のルールが更新されるまでには時間がかかります

この間に、<strong>既に終了処理を開始した Pod にトラフィックが送られてしまう</strong>可能性があります

### レース条件の流れ

以下のタイムラインで、レース条件の発生を見てみましょう

```
時刻 0s：API Server が Pod を「Terminating」に更新
         ├──→ kubelet に通知
         └──→ EndpointSlice コントローラに通知

時刻 0.1s：kubelet が SIGTERM を送信
           アプリケーションが終了処理を開始

時刻 0.2s：EndpointSlice コントローラが Pod を除外

時刻 0.5s：kube-proxy が EndpointSlice の変更を検知

時刻 0.8s：kube-proxy が iptables ルールを更新

問題：時刻 0.1s ～ 0.8s の間、アプリケーションは終了処理中にもかかわらず、
      トラフィックが送られてくる可能性がある
```

kubelet が SIGTERM を送信するタイミング（時刻 0.1s）と、kube-proxy のルール更新が完了するタイミング（時刻 0.8s）の間にずれがあります

このずれの間に、終了処理を開始したアプリケーションに新しいリクエストが到着する可能性があります

### なぜこのレース条件が起きるか

[05-self-healing](../05-self-healing.md) で学んだように、Readiness Probe が失敗すると Pod は EndpointSlice から除外されます

しかし、Pod の削除時には Readiness Probe の結果を待つのではなく、EndpointSlice コントローラが直接 Pod を除外します

この除外と kubelet の終了処理は<strong>独立した非同期の操作</strong>であり、完了のタイミングに保証がありません

特に、EndpointSlice の変更が kube-proxy に反映されるまでの伝搬遅延が、レース条件の主な原因です

---

## preStop フック

### preStop フックとは

<strong>preStop フック</strong>は、コンテナの終了前に実行される<strong>ライフサイクルフック</strong>です

kubelet は、SIGTERM を送信する<strong>前に</strong> preStop フックを実行します

preStop フックが完了してから、SIGTERM がコンテナに送信されます

```
kubelet が終了処理を開始
  │
  ▼
preStop フックを実行
  │
  ▼
preStop フックが完了
  │
  ▼
コンテナに SIGTERM を送信
```

### レース条件の緩和

preStop フックに短い待機時間（sleep）を入れることで、EndpointSlice の更新が kube-proxy に反映される時間を確保できます

```yaml
spec:
  containers:
    - name: web
      image: web-app:v1
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]
```

この設定では、kubelet は SIGTERM を送信する前に 5 秒間待機します

この 5 秒の間に、EndpointSlice の更新が kube-proxy に反映され、iptables ルールが更新されます

ルールが更新された後は、新しいトラフィックがこの Pod に送られなくなります

### preStop フック付きのタイムライン

```
時刻 0s  ：API Server が Pod を「Terminating」に更新
           ├──→ kubelet に通知
           └──→ EndpointSlice コントローラに通知

時刻 0.1s：kubelet が preStop フック（sleep 5）を開始
時刻 0.2s：EndpointSlice コントローラが Pod を除外
時刻 0.5s：kube-proxy が EndpointSlice の変更を検知
時刻 0.8s：kube-proxy が iptables ルールを更新
           → ここ以降、新しいトラフィックはこの Pod に送られない

時刻 5.1s：preStop フック（sleep 5）が完了
時刻 5.1s：kubelet が SIGTERM を送信
           → アプリケーションが終了処理を開始
           → この時点では新しいトラフィックは来ない
```

preStop の sleep によって、SIGTERM の送信が遅延され、その間に kube-proxy のルール更新が完了します

SIGTERM を受け取った時点では、新しいトラフィックは送られてこないため、アプリケーションは処理中のリクエストだけを完了させれば安全に終了できます

### preStop フックの注意点

preStop フックの実行時間は、terminationGracePeriodSeconds に含まれます

preStop で 5 秒間 sleep し、terminationGracePeriodSeconds がデフォルトの 30 秒の場合、SIGTERM を受けたアプリケーションが終了処理に使える時間は 25 秒です

アプリケーションの終了処理に長い時間が必要な場合は、terminationGracePeriodSeconds を十分に大きく設定する必要があります

---

## アプリケーション側の対応

### SIGTERM ハンドリング

Pod の安全な終了は、Kubernetes 側の仕組みだけでは実現できません

アプリケーション自身が SIGTERM を適切にハンドリングする必要があります

アプリケーションが SIGTERM を受け取ったときの望ましい動作は、以下の通りです

<strong>1. 新しいリクエストの受付を停止する</strong>

SIGTERM を受け取ったら、新しいリクエストの受付を停止します

リスニングソケットを閉じるか、新しい接続を拒否することで実現します

<strong>2. 処理中のリクエストを完了させる（コネクションドレイン）</strong>

既に受け付けたリクエストは、最後まで処理を完了させます

この動作を<strong>コネクションドレイン（Connection Draining）</strong>と呼びます

ドレイン（Draining）とは「排水する」という意味であり、処理待ちのリクエストを「流しきる」イメージです

<strong>3. リソースのクリーンアップ</strong>

処理中のリクエストが完了したら、リソースを解放します

データベースへの接続を閉じる、一時ファイルを削除する、メモリを解放するなどの処理を行います

<strong>4. プロセスを終了する</strong>

すべてのクリーンアップが完了したら、プロセスを正常終了します

### SIGTERM を無視した場合

アプリケーションが SIGTERM をハンドリングせず、無視した場合はどうなるでしょうか

kubelet は terminationGracePeriodSeconds の猶予期間が経過した後、SIGKILL でコンテナを強制終了します

SIGKILL ではアプリケーションに終了処理の機会が与えられないため、処理中のリクエストは中断され、リソースのクリーンアップも行われません

これは、クライアントから見るとリクエストが突然失敗する状態です

### コネクションドレインの流れ

SIGTERM を適切にハンドリングするアプリケーションの動作を、タイムラインで見てみましょう

```
SIGTERM を受信
  │
  ▼
新しいリクエストの受付を停止
  │
  ▼
処理中のリクエスト A を完了 → レスポンスを返す
  │
  ▼
処理中のリクエスト B を完了 → レスポンスを返す
  │
  ▼
全リクエストの処理が完了
  │
  ▼
データベース接続を切断、一時ファイルを削除
  │
  ▼
プロセスを正常終了（終了コード 0）
```

---

## ローリングアップデートとの関係

### 各 Pod の削除に適用される終了シーケンス

[07-declarative-management](../07-declarative-management.md) で学んだローリングアップデートでは、古い Pod が 1 つずつ新しい Pod に置き換えられます

この「古い Pod の削除」のたびに、ここまで学んだ終了シーケンスが実行されます

```
ローリングアップデートの流れ

1. 新しい ReplicaSet（v2）が Pod v2 を 1 つ作成
2. Pod v2 が Ready になる
3. 古い ReplicaSet（v1）の Pod v1 を 1 つ削除
   │
   └──→ ここで Graceful Shutdown のシーケンスが実行される
        ├──→ preStop フック → SIGTERM → 終了処理
        └──→ EndpointSlice から除外 → kube-proxy のルール更新
4. 次の Pod v2 を作成 ...
```

古い Pod が安全に終了してから次の置き換えに進むため、ローリングアップデート中にリクエストが失われることを防げます

### maxUnavailable と Graceful Shutdown の関係

[07-declarative-management](../07-declarative-management.md) で学んだ maxUnavailable は、更新中に利用不可になってよい Pod の数です

maxUnavailable が 1 の場合、一度に 1 つの Pod だけが終了処理に入ります

残りの Pod はトラフィックを処理し続けるため、終了処理中の Pod に送られるはずだったリクエストは残りの Pod に振り分けられます

maxUnavailable の値が大きいほど、同時に終了処理に入る Pod の数が増えます

Graceful Shutdown の設定（terminationGracePeriodSeconds、preStop フック）と合わせて、処理中のリクエストを確実に完了させるための時間を確保することが重要です

### 安全なローリングアップデートの全体像

ここまで学んだ仕組みを組み合わせると、ローリングアップデート時のリクエストの流れは以下のようになります

<strong>preStop フックによるレース条件の緩和</strong>

EndpointSlice の更新が kube-proxy に反映されるまでの間、preStop の sleep が SIGTERM の送信を遅延させます

<strong>EndpointSlice の更新によるトラフィック停止</strong>

kube-proxy のルールが更新されると、終了処理中の Pod に新しいトラフィックが送られなくなります

<strong>SIGTERM ハンドリングによるコネクションドレイン</strong>

アプリケーションが SIGTERM を受け取ると、処理中のリクエストを完了させてから終了します

<strong>Readiness Probe の Ready 条件を満たした新しい Pod がトラフィックを引き継ぐ</strong>

新しい Pod が Readiness Probe に成功すると、EndpointSlice に追加され、トラフィックを受け始めます

これら 4 つの仕組みが連携することで、ローリングアップデート中のリクエスト損失を防ぎます

---

## 用語集

| 用語                                            | 説明                                                                                                                 |
| ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Graceful Shutdown（グレースフルシャットダウン） | 処理中のリクエストを完了させ、リソースをクリーンアップしてから終了する安全な終了方法                                 |
| SIGTERM                                         | プロセスに終了を依頼するシグナル。プロセスはこのシグナルを受け取って終了処理を行う機会が与えられる                   |
| SIGKILL                                         | プロセスを強制的に終了するシグナル。プロセスはこのシグナルをハンドリングできず、即座に停止する                       |
| Termination Grace Period（終了猶予期間）        | kubelet が SIGTERM を送信してから SIGKILL を送信するまでの猶予時間。terminationGracePeriodSeconds で設定する         |
| terminationGracePeriodSeconds                   | Pod のマニフェストで設定する終了猶予期間の秒数。デフォルトは 30 秒                                                   |
| preStop フック                                  | コンテナの終了前に実行されるライフサイクルフック。SIGTERM の送信前に実行される                                       |
| ライフサイクルフック（Lifecycle Hook）          | コンテナのライフサイクルの特定のタイミングで実行される処理。preStop はその一種                                       |
| コネクションドレイン（Connection Draining）     | 新しいリクエストの受付を停止し、処理中のリクエストを完了させてから終了する手法                                       |
| レース条件（Race Condition）                    | 複数の処理のタイミングによって異なる結果が生じる状態。Pod 削除時の SIGTERM 送信と EndpointSlice 更新の間で発生しうる |
| Terminating                                     | Pod が削除処理中であることを示す状態。API Server が Pod を Terminating に更新すると、終了シーケンスが開始される      |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>Pod のライフサイクル</strong>

- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
  - Pod のフェーズ、終了シーケンス、terminationGracePeriodSeconds の公式ドキュメント

<strong>Pod の終了</strong>

- [Termination of Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
  - Pod の終了シーケンス（SIGTERM、SIGKILL、猶予期間）の公式ドキュメント

<strong>コンテナのライフサイクルフック</strong>

- [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
  - preStop フックの仕組みと設定方法の公式ドキュメント

<strong>EndpointSlice</strong>

- [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
  - EndpointSlice の仕組みと更新タイミングの公式ドキュメント
