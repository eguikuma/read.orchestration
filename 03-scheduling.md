---
layout: default
title: スケジューリング
---

# [03-scheduling：スケジューリング](#scheduling) {#scheduling}

## [はじめに](#introduction) {#introduction}

前のトピック [02-architecture](../02-architecture/) では、オーケストレーションのアーキテクチャを学びました

クラスタがコントロールプレーン（管理する側）とノード（実行する側）に分かれていること、API Server を中心にすべてのコンポーネントが連携していること、そして「あるべき状態（Desired State）」を Reconciliation Loop で維持する仕組みを確認しました

その中で、Scheduler というコンポーネントが Pod の配置先を決定する役割を持つことを学びました

しかし、いくつかの疑問が残っています

Scheduler は Pod の配置先をどのようなアルゴリズムで選択するのでしょうか？

Pod が必要とする CPU やメモリの量は、スケジューリングにどう影響するのでしょうか？

「この Pod は SSD を持つノードでしか動かさない」といった制約はどう実現されるのでしょうか？

このトピックでは、<strong>スケジューリング</strong>の仕組みを学びます

Scheduler がどのようなアルゴリズムで Pod の配置先を決定するか、リソース要求が配置にどう影響するか、そしてノード選択の制約をどう実現するかを見ていきます

---

## [日常の例え](#everyday-analogy) {#everyday-analogy}

スケジューリングの考え方を、日常の例えで見てみましょう

<strong>Scheduler = ホテルのフロントマネージャー</strong>

大きなホテルを想像してください

このホテルには複数の棟があり、それぞれの棟に異なる設備（オーシャンビュー、バリアフリー、スイートルームなど）があります

ゲストがチェックインすると、フロントマネージャーがどの棟のどの部屋に案内するかを決めます

フロントマネージャーは、2 つのステップで部屋を決めます

<strong>ステップ 1：候補の絞り込み</strong>

まず、ゲストを案内できない棟を除外します

満室の棟、改装中の棟、ゲストが必要とする設備がない棟は候補から外します

<strong>ステップ 2：最適な棟の選択</strong>

残った候補の中から、最も適した棟を選びます

空き部屋が多い棟、ロビーに近い棟、眺めが良い棟など、複数の基準で順位を付けます

<strong>リソース要求 = 部屋の条件</strong>

ゲストは「最低でもダブルベッドとバスルームが必要」という条件を出します

これがリソース要求（Requests）です

一方、部屋には物理的な限界があります

部屋の広さ、給湯設備の容量には上限があり、これがリソース制限（Limits）です

<strong>Taint = 「VIP 専用」の看板</strong>

ある棟の入口に「VIP 専用」という看板が掛かっています

一般のゲストはこの棟には案内されません

しかし、VIP バッジを持つゲストだけは案内されます

コンテナオーケストレーションでも同じです

ノードに Taint（忌避設定）を付け、特定の Toleration（許容設定）を持つ Pod だけを受け入れることができます

---

## [このページで学ぶこと](#what-you-will-learn) {#what-you-will-learn}

このページでは、以下の概念を学びます

<strong>スケジューリングアルゴリズム</strong>

- <strong>フィルタリングとスコアリング</strong>
  - 2 段階のアルゴリズムで Pod の配置先を決定する仕組み
- <strong>Scheduling Framework</strong>
  - プラグインベースの拡張可能なアーキテクチャ

<strong>リソース管理</strong>

- <strong>リソース要求（Requests）とリソース制限（Limits）</strong>
  - スケジューリング時の判断基準と実行時の上限
- <strong>CPU とメモリの計測単位</strong>
  - ミリコアとバイト単位の指定方法
- <strong>cgroup との関係</strong>
  - リソース制限がカーネルの仕組みを通じて適用される構造

<strong>ノード選択の仕組み</strong>

- <strong>ラベルと nodeSelector</strong>
  - 最もシンプルなノード選択制約
- <strong>Node Affinity（ノードアフィニティ）</strong>
  - 必須条件と推奨条件による柔軟なノード選択
- <strong>Taint と Toleration</strong>
  - ノード側からの Pod 受け入れ制御

<strong>OS スケジューラとの対比</strong>

- <strong>プロセススケジューラとの比較</strong>
  - 対象は異なるが原理は共通するスケジューリングの考え方

---

## [目次](#table-of-contents) {#table-of-contents}

1. [スケジューリングとは何か](#what-is-scheduling)
2. [フィルタリング（候補の絞り込み）](#filtering)
3. [スコアリング（最適なノードの選択）](#scoring)
4. [リソース要求と制限](#resource-requests-and-limits)
5. [ラベルと nodeSelector](#labels-and-node-selector)
6. [Node Affinity（ノードアフィニティ）](#node-affinity)
7. [Taint と Toleration](#taint-and-toleration)
8. [Scheduling Framework（拡張可能な設計）](#scheduling-framework)
9. [OS スケジューラとの対比](#os-scheduler-comparison)
10. [次のトピックへ](#next-topic)
11. [用語集](#glossary)
12. [参考資料](#references)

---

## [スケジューリングとは何か](#what-is-scheduling) {#what-is-scheduling}

前のトピックで、Scheduler は「まだノードに割り当てられていない Pod を見つけ、適切なノードを選んで割り当てるコンポーネント」であると学びました

ここでは、スケジューリングという行為そのものをもう少し掘り下げます

### [スケジューリングの定義](#scheduling-definition) {#scheduling-definition}

スケジューリングとは、<strong>Pod をクラスタ内のどのノードで実行するかを決定するプロセス</strong>です

コンテナの学習で学んだように、Pod は 1 つ以上のコンテナの集合であり、Kubernetes におけるスケジューリングの最小単位です

Scheduler は個々のコンテナではなく、Pod 単位で配置先を決定します

Pod 内の複数のコンテナは、必ず同じノード上で動作します

### [Scheduler の役割の境界](#scheduler-role-boundary) {#scheduler-role-boundary}

ここで重要なのは、<strong>Scheduler は Pod の配置先を決めるだけ</strong>ということです

Scheduler は「この Pod はノード 2 で動かすべきだ」と判断し、その結果を API Server 経由で etcd に記録します

実際にノード上で Pod を起動するのは、ノード上の kubelet の役割です

つまり、「どこで動かすか」を決める仕事と「実際に動かす」仕事は、明確に分離されています

### [スケジューリングとあるべき状態](#scheduling-and-desired-state) {#scheduling-and-desired-state}

スケジューリングは、<strong>あるべき状態を実現するための仕組み</strong>です

管理者がマニフェストで「Web サーバーの Pod を 3 つ動かす」と宣言すると、Controller Manager が Pod の不足を検知し、新しい Pod を作成します

作成された Pod はまだどのノードにも割り当てられていない状態です

Scheduler がこの未割り当ての Pod を検知し、適切なノードを選んで割り当てます

こうして、あるべき状態が実際のクラスタ上で実現されます

### [スケジューリングのタイミング](#scheduling-timing) {#scheduling-timing}

Scheduler は、Pod が作成されたタイミングで配置先を決定します

OS のプロセススケジューラのようにミリ秒単位で繰り返し実行されるのではなく、Pod が新しく作成されたときや、再スケジューリングが必要になったときに実行されます

Scheduler は API Server の Watch 機能を使い、未割り当ての Pod が作成されるのを常に監視しています

### [2 段階のアルゴリズム](#two-phase-algorithm) {#two-phase-algorithm}

Scheduler がノードを選ぶアルゴリズムは、大きく 2 つの段階に分かれます

```
全ノード ──→ [フィルタリング] ──→ 候補ノード ──→ [スコアリング] ──→ 最高スコアのノード
               硬い制約で                          柔らかい優先度で
               除外する                            順位を付ける
```

<strong>フィルタリング</strong>は、Pod を実行できないノードを候補から除外する段階です

<strong>スコアリング</strong>は、残った候補にスコアを付けて最適なノードを選ぶ段階です

以降のセクションで、それぞれの段階を詳しく見ていきます

---

## [フィルタリング（候補の絞り込み）](#filtering) {#filtering}

フィルタリングは、<strong>Pod を実行できないノードを候補から除外する段階</strong>です

ホテルの例えで言えば、満室の棟や改装中の棟を候補から外すステップです

### [硬い制約](#hard-constraints) {#hard-constraints}

フィルタリングで適用されるのは、<strong>硬い制約（ハードコンストレイント）</strong>です

硬い制約とは、「満たさなければ絶対に配置しない」という条件です

条件を満たすか満たさないかの二択であり、部分的に満たすという中間はありません

### [主なフィルタ条件](#main-filter-conditions) {#main-filter-conditions}

Scheduler は、以下のような条件でノードをフィルタリングします

{: .labeled}
| フィルタ条件 | 確認する内容 | ホテルの例え |
| --------------------- | ---------------------------------------------------- | ------------------------------- |
| リソースの空き | ノードの空き CPU / メモリが Pod の要求量を満たすか | 棟に空き部屋があるか |
| Taint の許容 | ノードの Taint を Pod が許容（Toleration）しているか | VIP 専用棟に VIP バッジがあるか |
| Node Affinity（必須） | ノードのラベルが Pod の必須条件を満たすか | 棟に必要な設備があるか |
| ノードの状態 | ノードが新しい Pod を受け入れ可能な状態か | 棟が営業中か（改装中でないか） |

各条件は独立して評価され、<strong>1 つでも満たさない条件があれば、そのノードは候補から除外されます</strong>

### [フィルタリングの結果](#filtering-result) {#filtering-result}

フィルタリングの結果、3 つのパターンがあります

<strong>複数のノードが候補に残る場合</strong>

次の段階であるスコアリングに進み、候補の中から最適なノードを選びます

<strong>1 つのノードだけが候補に残る場合</strong>

そのノードが配置先として選ばれます

スコアリングは行われますが、候補が 1 つしかないため結果は確定しています

<strong>候補が 0 の場合（どのノードもフィルタリングを通過しない）</strong>

Pod は<strong>Pending（保留）</strong>状態になります

ただし、Pod のあるべき状態は etcd に記録されたままです

新しいノードが追加されたり、既存のノードのリソースに空きが出たりすると、Scheduler が再びスケジューリングを試みます

このように、すぐに配置できなくても、あるべき状態は失われません

---

## [スコアリング（最適なノードの選択）](#scoring) {#scoring}

スコアリングは、<strong>フィルタリングを通過したノードにスコアを付けて、最適なノードを選ぶ段階</strong>です

ホテルの例えで言えば、空いている棟の中から、空き部屋の多さ・ロビーへの近さ・眺めの良さなどを総合的に評価するステップです

### [柔らかい優先度](#soft-priority) {#soft-priority}

スコアリングで適用されるのは、<strong>柔らかい優先度（ソフトプリファレンス）</strong>です

柔らかい優先度とは、「可能であれば優先したい」という条件です

フィルタリングの硬い制約とは異なり、満たさなくても配置は行われます

ただし、より多くの優先度を満たすノードが高いスコアを得ます

### [フィルタリングとスコアリングの対比](#filtering-and-scoring-comparison) {#filtering-and-scoring-comparison}

{: .labeled}
| 特性 | フィルタリング | スコアリング |
| ---------- | ------------------------------ | ---------------------------------- |
| 制約の性質 | 硬い制約（満たさなければ除外） | 柔らかい優先度（満たすほど高評価） |
| 結果 | 候補に残るか除外されるかの二択 | 0〜100 のスコア |
| 役割 | 配置できないノードを除外する | 配置に最適なノードを選ぶ |

### [主なスコアリング戦略](#main-scoring-strategies) {#main-scoring-strategies}

Scheduler は、複数のスコアリング戦略を組み合わせてノードにスコアを付けます

{: .labeled}
| スコアリング戦略 | 優先する条件 | 用途 |
| -------------------- | ---------------------------------------------------- | ---------------------------------- |
| LeastAllocated | リソースの空きが多いノード | 負荷をクラスタ全体に分散させる |
| MostAllocated | リソースの空きが少ないノード | ノードを集約してリソースを節約する |
| ImageLocality | Pod のコンテナイメージを既にキャッシュしているノード | コンテナの起動を高速化する |
| NodeAffinityPriority | Pod の推奨条件に合致するノード | 柔らかいノード選択の希望を反映する |

各戦略がノードにスコアを付け、すべてのスコアを合算した合計が最も高いノードが選ばれます

合計スコアが同点の場合は、ランダムに 1 つが選ばれます

### [スコアリングの例](#scoring-example) {#scoring-example}

具体例で見てみましょう

3 つのノードがフィルタリングを通過したとします

```
              LeastAllocated    ImageLocality    合計
ノード A          80                20            100
ノード B          50                80            130
ノード C          60                40            100
```

この場合、合計スコアが最も高いノード B が選ばれます

ノード A は空きリソースが最も多いですが、ノード B はコンテナイメージのキャッシュがあるため起動が速く、総合的に最も高いスコアとなりました

---

## [リソース要求と制限](#resource-requests-and-limits) {#resource-requests-and-limits}

Pod をどのノードに配置するかを決める際、Pod が必要とするリソースの量は重要な判断基準です

リソースには、主に <strong>CPU</strong> と<strong>メモリ</strong>の 2 種類があります

### [リソース要求（Requests）](#resource-requests) {#resource-requests}

<strong>リソース要求（Requests）</strong>は、Pod が正常に動作するために<strong>最低限必要なリソースの量</strong>です

ホテルの例えで言えば、ゲストが「最低でもダブルベッドとバスルームが必要」と伝える条件です

Scheduler はリソース要求を見て、ノードにその分の空きがあるかを確認します

あるノードの空き CPU が 200m で、Pod が 300m を要求している場合、そのノードはフィルタリングで除外されます

### [リソース制限（Limits）](#resource-limits) {#resource-limits}

<strong>リソース制限（Limits）</strong>は、Pod が使用できる<strong>リソースの上限</strong>です

ホテルの例えで言えば、部屋の物理的な限界（広さ、給湯設備の容量）です

Limits は Scheduler ではなく、ノード上の kubelet が実行時に適用します

Pod が Limits を超えてリソースを使おうとした場合、強制的に制限されます

### [Requests と Limits の違い](#requests-and-limits-difference) {#requests-and-limits-difference}

{: .labeled}
| 概念 | 意味 | いつ使われるか | 誰が使うか |
| -------- | -------------- | ------------------ | ---------------------- |
| Requests | 最低限必要な量 | スケジューリング時 | Scheduler |
| Limits | 使用可能な上限 | 実行時 | kubelet（cgroup 経由） |

この違いは非常に重要です

Requests は<strong>スケジューリングの判断基準</strong>であり、Limits は<strong>実行時の安全装置</strong>です

Pod は Requests 以上、Limits 以下のリソースを使用できます

Requests を超えてリソースを使うことは許可されますが、Limits を超えることは許可されません

### [CPU とメモリの単位](#cpu-and-memory-units) {#cpu-and-memory-units}

リソースの量は、以下の単位で指定します

<strong>CPU</strong>

CPU は<strong>ミリコア（millicores）</strong>という単位で指定します

1000m（ミリコア）が 1 つの CPU コアに相当します

たとえば、`250m` は CPU コアの 25% に相当し、`500m` は 50% に相当します

<strong>メモリ</strong>

メモリは<strong>バイト単位</strong>で指定します

一般的にはバイナリ接頭辞（Ki、Mi、Gi）を使います

`128Mi` は 128 メビバイト（約 134 メガバイト）、`1Gi` は 1 ギビバイト（約 1.07 ギガバイト）です

### [マニフェストでの指定](#manifest-specification) {#manifest-specification}

リソースの要求と制限は、Pod の仕様の中で以下のように記述します

```yaml
spec:
  containers:
    - name: web
      image: nginx
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

この例では、Pod は最低でも CPU 250m とメモリ 128Mi を必要とし、最大で CPU 500m とメモリ 256Mi まで使用できます

Scheduler はこの Requests（CPU 250m、メモリ 128Mi）を見て、十分な空きがあるノードを候補に残します

### [リソース制限と cgroup](#resource-limits-and-cgroup) {#resource-limits-and-cgroup}

Limits の実行時の適用は、Linux カーネルの<strong>cgroup（Control Group）</strong>を通じて行われます

cgroup とは、プロセスが使用できるリソース（CPU、メモリなど）を制限するカーネルの仕組みです

{: .labeled}
| リソース | Kubernetes の仕様 | cgroup のインターフェース | 制限を超えた場合 |
| -------- | ----------------- | ------------------------- | ------------------------------------------------ |
| CPU | limits.cpu | cpu.max | CPU 時間がスロットリングされる（処理が遅くなる） |
| メモリ | limits.memory | memory.max | OOM Kill（プロセスが強制終了される） |

CPU の Limits を超えた場合、Pod はすぐには停止されず、CPU の割り当て時間が制限されます

一方、メモリの Limits を超えた場合は、カーネルの OOM Killer によってプロセスが強制終了される可能性があります

前のシリーズでカーネル空間の仕組みを学んだ方は、ここで cgroup の知識がつながります

Kubernetes のリソース制限は、カーネルが提供する cgroup の上に構築されています

### [リソースとあるべき状態](#resources-and-desired-state) {#resources-and-desired-state}

リソース要求と制限は、Pod の<strong>あるべき状態の一部</strong>です

マニフェストに「この Pod は CPU 250m、メモリ 128Mi を必要とする」と宣言することで、Scheduler はその条件を満たすノードに配置します

あるべき状態にリソース情報を含めることで、クラスタ全体のリソース配分を宣言的に管理できます

---

## [ラベルと nodeSelector](#labels-and-node-selector) {#labels-and-node-selector}

ここまでは、リソースの空きに基づくノード選択を見てきました

しかし、リソースの量だけでは表現できない制約もあります

「SSD を持つノードに配置したい」「特定のゾーンのノードに配置したい」といった制約です

こうした制約を実現するのが、<strong>ラベル</strong>と<strong>nodeSelector</strong>です

### [ラベル（Label）](#label) {#label}

<strong>ラベル</strong>は、Kubernetes のオブジェクト（ノード、Pod など）に付与する<strong>キーとバリューの組み合わせ</strong>です

たとえば、SSD を持つノードには `disk: ssd` というラベルを付け、HDD を持つノードには `disk: hdd` というラベルを付けます

ラベルはオブジェクトの特徴を表すメタデータであり、オブジェクトの動作には影響しません

### [nodeSelector](#node-selector) {#node-selector}

<strong>nodeSelector</strong>は、Pod の仕様に記述する<strong>最もシンプルなノード選択制約</strong>です

Pod に nodeSelector を指定すると、Scheduler はそのラベルを持つノードにのみ Pod を配置します

```yaml
spec:
  nodeSelector:
    disk: ssd
```

この例では、Pod は `disk: ssd` というラベルを持つノードにのみ配置されます

これが、前のトピックの疑問「この Pod は SSD を持つノードでしか動かさない」への直接的な回答です

### [nodeSelector の特性](#node-selector-characteristics) {#node-selector-characteristics}

nodeSelector は<strong>フィルタリングの段階で適用される硬い制約</strong>です

指定したラベルを持たないノードは候補から除外されます

「できれば SSD のノードに配置したいが、なければ HDD でもよい」といった柔軟な指定はできません

こうした柔軟性が必要な場合には、次のセクションで学ぶ Node Affinity を使います

---

## [Node Affinity（ノードアフィニティ）](#node-affinity) {#node-affinity}

nodeSelector はシンプルですが、表現力に限界があります

ラベルの完全一致しか指定できず、「できれば〜に配置したい」という柔軟な希望を表現できません

<strong>Node Affinity（ノードアフィニティ）</strong>は、nodeSelector をより柔軟にしたノード選択の仕組みです

### [2 種類の条件](#two-condition-types) {#two-condition-types}

Node Affinity には、2 種類の条件があります

<strong>必須条件（requiredDuringSchedulingIgnoredDuringExecution）</strong>

「この条件を満たすノードにのみ配置する」という硬い制約です

フィルタリングの段階で適用されます

ホテルの例えで言えば、「オーシャンビューの棟でなければ泊まらない」という条件です

<strong>推奨条件（preferredDuringSchedulingIgnoredDuringExecution）</strong>

「可能であればこの条件を満たすノードに配置したい」という柔らかい優先度です

スコアリングの段階で適用されます

ホテルの例えで言えば、「オーシャンビューの棟が良いが、空いていなければ他の棟でもよい」という希望です

### [名前の意味](#name-meaning) {#name-meaning}

Node Affinity の名前は長いですが、2 つの部分に分解すると理解しやすくなります

<strong>〜DuringScheduling</strong>

スケジューリング時（Pod の配置先を決めるとき）にこの条件を適用するという意味です

<strong>IgnoredDuringExecution</strong>

Pod がノードに配置された後は、この条件を無視するという意味です

つまり、ノードのラベルが後から変更されても、既に実行中の Pod はそのノードに留まります

たとえば、`zone: tokyo` という条件で配置された後に、ノードの `zone` ラベルが変更されても、Pod は移動しません

### [Node Affinity の演算子](#node-affinity-operators) {#node-affinity-operators}

nodeSelector がラベルの完全一致しかサポートしないのに対し、Node Affinity は豊富な演算子を使えます

{: .labeled}
| 演算子 | 意味 | 例 |
| ------------ | -------------------------------- | ------------------------ |
| In | 値がリストのいずれかに一致する | zone In [tokyo, osaka] |
| NotIn | 値がリストのいずれにも一致しない | zone NotIn [staging] |
| Exists | キーが存在する（値は問わない） | gpu Exists |
| DoesNotExist | キーが存在しない | maintenance DoesNotExist |
| Gt | 値が指定した数値より大きい | cpu-cores Gt 4 |
| Lt | 値が指定した数値より小さい | cpu-cores Lt 16 |

### [nodeSelector と Node Affinity の比較](#node-selector-and-affinity-comparison) {#node-selector-and-affinity-comparison}

{: .labeled}
| 特性 | nodeSelector | Node Affinity |
| -------------- | ------------ | ------------------------------ |
| 一致条件 | 完全一致のみ | In、NotIn、Exists、Gt、Lt など |
| 硬い制約 | 対応 | 対応（required） |
| 柔らかい優先度 | 非対応 | 対応（preferred） |
| 表現力 | シンプル | 豊富 |

シンプルな条件には nodeSelector、柔軟な条件が必要な場合には Node Affinity を使います

---

## [Taint と Toleration](#taint-and-toleration) {#taint-and-toleration}

ここまで見てきた nodeSelector と Node Affinity は、<strong>Pod 側から</strong>「このノードに配置してほしい」と指定する仕組みです

<strong>Taint と Toleration</strong>は、逆方向の仕組みです

<strong>ノード側から</strong>「この Pod を受け入れない」と宣言します

### [方向の違い](#direction-difference) {#direction-difference}

{: .labeled}
| 仕組み | 誰が宣言するか | 方向 | ホテルの例え |
| ------------- | -------------- | ---------------------- | -------------------------------------------- |
| Node Affinity | Pod | Pod → Node（引き寄せ） | ゲスト：「オーシャンビューの棟に泊まりたい」 |
| Taint | Node | Node → Pod（拒否） | 棟の入口：「VIP 専用」の看板 |
| Toleration | Pod | Pod が Taint を許容 | ゲスト：VIP バッジを提示する |

Node Affinity が Pod の「希望」であるのに対し、Taint はノードの「拒否」です

### [Taint（テイント）](#taint) {#taint}

<strong>Taint</strong>は、ノードに付与する<strong>忌避設定</strong>です

Taint が付いたノードには、その Taint を許容する Toleration を持たない Pod は配置されません

Taint は `キー=バリュー:効果` の形式で指定します

たとえば、GPU を搭載したノードに `hardware=gpu:NoSchedule` という Taint を付けると、GPU ワークロード以外の Pod がそのノードに配置されるのを防げます

### [Toleration（トレレーション）](#toleration) {#toleration}

<strong>Toleration</strong>は、Pod に設定する<strong>Taint の許容宣言</strong>です

Pod に Toleration を設定すると、対応する Taint が付いたノードにも配置できるようになります

ただし、Toleration は「このノードに配置してほしい」という意味ではありません

Toleration は「このノードに配置されることを許容する」という意味です

Toleration を持つ Pod が、必ずしもそのノードに配置されるわけではありません

配置先は、フィルタリングとスコアリングの結果によって決まります

### [3 つの効果（Effect）](#three-effects) {#three-effects}

Taint には 3 種類の効果があります

{: .labeled}
| 効果 | 新しい Pod への影響 | 既存の Pod への影響 |
| ---------------- | -------------------------------------------------- | ------------------------------- |
| NoSchedule | Toleration がなければ配置しない | 影響なし（そのまま動く） |
| PreferNoSchedule | できれば配置しないが、他にノードがなければ配置する | 影響なし（そのまま動く） |
| NoExecute | Toleration がなければ配置しない | Toleration がなければ退去させる |

<strong>NoSchedule</strong>は最も一般的な効果で、Toleration を持たない新しい Pod の配置を拒否します

既に動いている Pod には影響しません

<strong>PreferNoSchedule</strong>は柔らかい拒否です

できれば配置を避けますが、他に適切なノードがなければ配置を許可します

<strong>NoExecute</strong>は最も強い効果です

新しい Pod の配置を拒否するだけでなく、既に動いている Pod も、Toleration を持たなければ退去させます

ノードのメンテナンスや障害時に使用されます

### [用途の例](#usage-example) {#usage-example}

{: .labeled}
| 用途 | Taint の設定例 | 説明 |
| ------------ | ---------------------------- | ----------------------------------------------------- |
| 専用ノード | `team=ml:NoSchedule` | 機械学習チーム専用のノードを確保する |
| GPU ノード | `hardware=gpu:NoSchedule` | GPU ワークロード以外の Pod が配置されるのを防ぐ |
| メンテナンス | `maintenance=true:NoExecute` | メンテナンス前に Pod を退去させ、新しい配置も拒否する |

---

## [Scheduling Framework（拡張可能な設計）](#scheduling-framework) {#scheduling-framework}

ここまで、フィルタリング、スコアリング、リソース要求、ラベル、Node Affinity、Taint と Toleration といった仕組みを個別に見てきました

では、これらの仕組みはどのように統合されているのでしょうか

### [プラグインベースのアーキテクチャ](#plugin-based-architecture) {#plugin-based-architecture}

Kubernetes の Scheduler は、<strong>Scheduling Framework</strong>と呼ばれるプラグインベースのアーキテクチャで設計されています

フィルタリングやスコアリングの各段階には<strong>拡張ポイント</strong>が定義されており、それぞれの仕組みが<strong>プラグイン</strong>として実装されています

```
Pod の作成（未割り当て）
  │
  ▼
┌─────────────────────────────────────────────────┐
│              スケジューリングサイクル              │
│                                                 │
│  PreFilter ──→ Filter ──→ PostFilter            │
│                  │                              │
│                  │  NodeResourcesFit（リソース） │
│                  │  TaintToleration（Taint）     │
│                  │  NodeAffinity（必須条件）     │
│                  │  NodeUnschedulable（状態）    │
│                                                 │
│  PreScore ──→ Score ──→ NormalizeScore           │
│                 │                               │
│                 │  LeastAllocated（負荷分散）    │
│                 │  NodeAffinityPriority（推奨）  │
│                 │  ImageLocality（キャッシュ）   │
│                                                 │
│  Reserve ──→ Permit                             │
└───────────────────────────────┬─────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────┐
│               バインディングサイクル               │
│                                                 │
│  PreBind ──→ Bind ──→ PostBind                  │
│                │                                │
│                │  Pod をノードに紐付ける          │
└─────────────────────────────────────────────────┘
```

### [拡張ポイント](#extension-points) {#extension-points}

主な拡張ポイントの役割は以下の通りです

{: .labeled}
| 拡張ポイント | 段階 | 役割 |
| -------------- | ---------------- | ------------------------------------------- |
| Filter | フィルタリング | Pod を配置できないノードを除外する |
| PostFilter | フィルタリング後 | フィルタリングで候補が 0 になった場合の対応 |
| Score | スコアリング | 候補ノードにスコアを付ける |
| NormalizeScore | スコアリング後 | スコアを正規化する（0〜100 の範囲に揃える） |
| Reserve | 予約 | 選ばれたノードのリソースを仮予約する |
| Permit | 許可 | バインディングを許可、拒否、または保留する |
| Bind | バインディング | Pod をノードに紐付ける |

### [プラグインとして実装されている仕組み](#plugin-implementation) {#plugin-implementation}

ここまで学んできた仕組みは、すべて Scheduling Framework のプラグインとして実装されています

{: .labeled}
| プラグイン | 拡張ポイント | 担当する仕組み |
| ----------------- | -------------- | ------------------------------------ |
| NodeResourcesFit | Filter | リソース要求に基づくフィルタリング |
| TaintToleration | Filter | Taint と Toleration のチェック |
| NodeAffinity | Filter / Score | Node Affinity の必須条件と推奨条件 |
| NodeUnschedulable | Filter | ノードが受け入れ可能かのチェック |
| LeastAllocated | Score | 空きリソースが多いノードを優先 |
| ImageLocality | Score | イメージキャッシュがあるノードを優先 |

### [拡張性の原則](#extensibility-principles) {#extensibility-principles}

Scheduling Framework の設計は、<strong>コアのアルゴリズムを変更せずに、プラグインの追加や入れ替えでスケジューリングの振る舞いをカスタマイズできる</strong>ようになっています

新しいスケジューリング戦略が必要になったとき、Scheduler 本体を書き換える必要はありません

対応する拡張ポイントに新しいプラグインを追加するだけです

この設計は、Kubernetes 全体に見られる「拡張性を重視した設計」の一例です

---

## [OS スケジューラとの対比](#os-scheduler-comparison) {#os-scheduler-comparison}

前のシリーズでカーネル空間を学んだ方は、OS のプロセススケジューラを思い出すかもしれません

プロセススケジューラは、CPU 上でどのプロセスを実行するかを決める仕組みです

オーケストレーションの Scheduler とは対象も規模も異なりますが、共通する原理があります

### [比較表](#comparison-table) {#comparison-table}

{: .labeled}
| 比較項目 | OS プロセススケジューラ | オーケストレーション Scheduler |
| ---------------- | ------------------------------ | --------------------------------------------- |
| 何を配置するか | プロセス / スレッド | Pod |
| どこに配置するか | CPU コア | ノード（マシン） |
| 判断基準 | 優先度、公平性、タイムスライス | リソース要求、制約条件、スコア |
| 実行頻度 | ミリ秒単位（CPU の切り替え） | Pod の作成時 |
| 硬い制約 | CPU アフィニティ | nodeSelector、Node Affinity（必須）、Taint |
| 柔らかい優先度 | nice 値 | Node Affinity（推奨）、スコアリングプラグイン |
| リソース制限 | cgroup | cgroup（同じ仕組み） |

### [共通する原理](#common-principles) {#common-principles}

対象と規模は大きく異なりますが、以下の原理は共通しています

<strong>「何を動かすか」と「どこで動かすか」の分離</strong>

OS スケジューラは「どのプロセスをどの CPU コアで実行するか」を決め、実際の実行は CPU が行います

オーケストレーション Scheduler は「どの Pod をどのノードで実行するか」を決め、実際の起動は kubelet が行います

どちらも、配置の決定と実行を分離しています

<strong>硬い制約と柔らかい優先度の組み合わせ</strong>

OS スケジューラでは CPU アフィニティ（硬い制約）と nice 値（柔らかい優先度）を組み合わせます

オーケストレーション Scheduler でもフィルタリング（硬い制約）とスコアリング（柔らかい優先度）を組み合わせます

<strong>cgroup という共通基盤</strong>

特に注目すべきは、どちらのスケジューラもリソース制限に<strong>同じカーネルの仕組み（cgroup）</strong>を使っている点です

OS がプロセスの CPU / メモリ使用量を cgroup で制限するのと同じメカニズムを、Kubernetes は Pod のリソース制限に使っています

カーネルの学習から、コンテナの学習、そしてオーケストレーションの学習まで、cgroup は一貫して登場する基盤技術です

---

## [次のトピックへ](#next-topic) {#next-topic}

このトピックでは、以下のことを学びました

- Scheduler が Pod の配置先を 2 段階のアルゴリズム（フィルタリングとスコアリング）で決定する仕組み
- リソース要求（Requests）がスケジューリング時の判断基準となり、リソース制限（Limits）が実行時に cgroup を通じて適用される仕組み
- ラベル、nodeSelector、Node Affinity による柔軟なノード選択
- Taint と Toleration による「ノード側からの拒否」の仕組み
- Scheduling Framework によるプラグインベースの拡張可能な設計

Pod がノードに配置され、実行されるようになりました

ここで、新たな疑問が生まれます

Pod がノードに配置された後、他の Pod はその Pod をどうやって見つけるのでしょうか？

Pod が再作成されて IP アドレスが変わった場合、通信先をどう維持するのでしょうか？

複数のノードに分散した Pod 群に対して、どうやって負荷を分散するのでしょうか？

次のトピック [04-service-discovery](../04-service-discovery/) では、<strong>サービスディスカバリ</strong>の仕組みを詳しく学びます

---

## [用語集](#glossary) {#glossary}

{: .labeled}
| 用語 | 説明 |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| スケジューリング（Scheduling） | Pod をクラスタ内のどのノードで実行するかを決定するプロセス |
| Scheduler（kube-scheduler） | まだノードに割り当てられていない Pod を見つけ、適切なノードを選んで割り当てるコンポーネント |
| フィルタリング（Filtering） | Scheduler がノード選択時に、Pod を実行できないノードを候補から除外する段階。硬い制約を適用する |
| スコアリング（Scoring） | Scheduler がフィルタリング後のノード候補にスコアを付け、最適なノードを選ぶ段階。柔らかい優先度を適用する |
| 硬い制約（ハードコンストレイント） | 満たさなければ配置しないという条件。フィルタリングで適用される |
| 柔らかい優先度（ソフトプリファレンス） | 可能であれば優先したいという条件。スコアリングで適用される |
| リソース要求（Requests） | Pod が正常に動作するために最低限必要な CPU やメモリの量。Scheduler がスケジューリング時の判断基準として使用する |
| リソース制限（Limits） | Pod が使用できる CPU やメモリの上限。kubelet が cgroup を通じて実行時に適用する |
| ミリコア（Millicores） | CPU リソースの計測単位。1000m が 1 つの CPU コアに相当する |
| Allocatable（割り当て可能） | ノードの全リソースからシステム予約分を差し引いた、Pod に割り当て可能なリソース量 |
| ラベル（Label） | Kubernetes オブジェクトに付与するキーとバリューの組み合わせ。ノード選択やグルーピングに使用する |
| nodeSelector | Pod の仕様に記述する最もシンプルなノード選択制約。指定したラベルを持つノードにのみ配置される |
| Node Affinity（ノードアフィニティ） | nodeSelector の拡張版。必須条件（required）と推奨条件（preferred）を柔軟に指定できるノード選択の仕組み |
| requiredDuringSchedulingIgnoredDuringExecution | Node Affinity の必須条件。フィルタリングで適用される硬い制約。スケジューリング時に必須、実行中は無視 |
| preferredDuringSchedulingIgnoredDuringExecution | Node Affinity の推奨条件。スコアリングで適用される柔らかい優先度。スケジューリング時に考慮、実行中は無視 |
| Taint（テイント） | ノードに付与する忌避設定。Toleration を持たない Pod の配置を拒否する |
| Toleration（トレレーション） | Pod に設定する Taint の許容宣言。Taint が付いたノードへの配置を可能にする |
| NoSchedule | Taint の効果の 1 つ。Toleration を持たない新しい Pod の配置を拒否する。既存の Pod には影響しない |
| PreferNoSchedule | Taint の効果の 1 つ。可能であれば配置を避けるが、他にノードがなければ許容する。既存の Pod には影響しない |
| NoExecute | Taint の効果の 1 つ。Toleration を持たない Pod の配置を拒否し、既に動いている Pod も退去させる |
| Scheduling Framework | Kubernetes の Scheduler のプラグインベースのアーキテクチャ。フィルタリングやスコアリングの各段階を拡張ポイントとして提供する |
| Pending（保留） | Pod が作成されたがまだノードに配置されていない状態。フィルタリングを通過するノードがない場合に発生する |
| cgroup（Control Group） | Linux カーネルの機能。プロセスが使用できるリソース（CPU、メモリなど）を制限する。Kubernetes では Limits の実行時の適用に使用される |
| OOM Kill（Out of Memory Kill） | メモリの Limits を超過した場合に、カーネルがプロセスを強制終了する仕組み |

---

## [参考資料](#references) {#references}

このページの内容は、以下のソースに基づいています

<strong>Kubernetes スケジューリング</strong>

- [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/){:target="\_blank"}
  - Scheduler の概要と 2 段階アルゴリズムの公式説明

- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/){:target="\_blank"}
  - nodeSelector と Node Affinity の公式ドキュメント

- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/){:target="\_blank"}
  - Taint と Toleration の公式ドキュメント

- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/){:target="\_blank"}
  - Scheduling Framework のプラグインアーキテクチャの公式説明

<strong>リソース管理</strong>

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/){:target="\_blank"}
  - リソース要求（Requests）とリソース制限（Limits）の公式ドキュメント

<strong>リソース制限の実行基盤</strong>

- [cgroups(7) - Linux manual page](https://man7.org/linux/man-pages/man7/cgroups.7.html){:target="\_blank"}
  - cgroup の Linux man ページ

<strong>起源</strong>

- "Large-scale cluster management at Google with Borg" (Verma et al., EuroSys 2015)
  - Google の大規模クラスタ管理システム Borg のスケジューリングアルゴリズム

- "Borg, Omega, and Kubernetes" (Burns et al., ACM Queue 2016)
  - Borg から Kubernetes への設計思想の変遷とスケジューリングの進化
