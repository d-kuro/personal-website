---
title: "Kubernetes Leader Election in Depth"
date: 2020-01-06T01:09:52+09:00
draft: false
aliases:
  - /posts/kubernetes-leader-election/
tags: [Kubernetes]
categories: [Kubernetes]
---

この記事では Kubernetes の Leader election について解説していきます。

<!--more-->

## Kubernetes における Leader election

Kubernetes における Leader election は Controller 間の競合を防ぐために使われます。

例として Deployment を watch して処理を行う Controller の Pod がある場合に Replica の Pod が 2 台だと 1 つの Deployment の操作に対して 2 つの Pod が処理を行うため, 場合によっては競合してしまいます。
そのような競合を防ぐために Kubernetes では clinet-go に Leader election の仕組みが用意されています。
Leader election を使用するとリーダーとなる Controller が reconciliation loop を実行している間, 他の Controller は待機します。
リーダーが辞任した場合待機していた Controller がリーダーに昇格し, すぐに処理を再開することができます。

可用性が必要な Controller 以外の場合は `replicas: 1` にすれば問題ない, と思うかもしれませんが Deployment の strategy を RollingUpdate にしている場合は一時的に 2 つの Pod が動作する状況が存在するため, 注意が必要です。


本ブログでは以下の 2 つの Leader election の実装について解説していきます。

* Leader-for-life
  * [github.com/operator-framework/operator-sdk/pkg/leader](https://godoc.org/github.com/operator-framework/operator-sdk/pkg/leader)
* Leader-with-lease
  * [github.com/kubernetes-sigs/controller-runtime/pkg/leaderelection](https://godoc.org/github.com/kubernetes-sigs/controller-runtime/pkg/leaderelection)

上記の名前は Operator SDK のドキュメントから持ってきています。

https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-leader-election.html

### 追記: Kubernetes Meetup Tokyo #27 で LT しました

<script async class="speakerdeck-embed" data-id="6aa22d2c4ff34536bc52461feacb2be5" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

## Leader-for-life

Leader-for-life は Operator SDK が提供している Leader election の仕組みです。

仕組みは単純で OwnerReference がリーダーの Pod である ConfigMap を作成し, ロックします。
Pod が削除されると Kubernetes のガベージコレクションの仕組みにより ConfigMap も自動的に削除されるため, 他の Pod がリーダーを獲得することができます。

[Garbage Collection - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/)

使用の際には以下の用に Pod の名前を環境変数として渡してあげる必要があります。

```yaml
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

作成される ConfigMap の例:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: "2019-12-27T04:26:18Z"
  name: operator-lock
  namespace: default
  ownerReferences:
  - apiVersion: v1
    kind: Pod
    name: operator-787d565ff9-xzf69
    uid: fb50cbf0-2860-11ea-8930-065483f65576
  resourceVersion: "27146109"
  selfLink: /api/v1/namespaces/default/configmaps/operator-lock
  uid: 07468a04-2861-11ea-8930-065483f65576
```

利用するための関数は次のようになっています。

```go
func Become(ctx context.Context, lockName string) error
```

https://godoc.org/github.com/operator-framework/operator-sdk/pkg/leader#Become

使用する際には次のように使用します。

```go
import (
  ...
  "github.com/operator-framework/operator-sdk/pkg/leader"
)

func main() {
  ...
  err = leader.Become(context.TODO(), "memcached-operator-lock")
  if err != nil {
    log.Error(err, "Failed to retry for leader lock")
    os.Exit(1)
  }
  ...
}
```

以下では詳しくコードを見ていきます。

### OwnerReference を作成する

環境変数 `POD_NAME` より自身の Pod の name を取得して Pod を取得後, その情報を用いて OwnerReference を作成します。

```go
owner, err := myOwnerRef(ctx, client, ns)
```

https://github.com/operator-framework/operator-sdk/blob/v0.13.0/pkg/leader/leader.go#L67

### ConfigMap を取得し, Owner を確認する

ここで一度 ConfigMap を取得します。
取得した結果, 自身が Owner であることが確認でした場合は `nil` を return し関数を終了します。

リーダーである Pod が再起動された場合はここのロジックでリーダーを継続することができます。

```go
// check for existing lock from this pod, in case we got restarted
existing := &corev1.ConfigMap{}
key := crclient.ObjectKey{Namespace: ns, Name: lockName}
err = client.Get(ctx, key, existing)

switch {
case err == nil:
    for _, existingOwner := range existing.GetOwnerReferences() {
        if existingOwner.Name == owner.Name {
            log.Info("Found existing lock with my name. I was likely restarted.")
            log.Info("Continuing as the leader.")
            return nil
        }
        log.Info("Found existing lock", "LockOwner", existingOwner.Name)
    }
case apierrors.IsNotFound(err):
    log.Info("No pre-existing lock was found.")
default:
    log.Error(err, "Unknown error trying to get ConfigMap")
    return err
}
```

https://github.com/operator-framework/operator-sdk/blob/v0.13.0/pkg/leader/leader.go#L72-L92

### ConfigMap の作成を試行する

for でループしながら ConfigMap の作成を試行します。
このループは ConfigMap が作成され, 自身がリーダーになるまで実行されます。

リーダーの Pod が evict された場合はその Pod の削除を行います。

```go
// try to create a lock
backoff := time.Second
for {
    err := client.Create(ctx, cm)
    switch {
    case err == nil:
        log.Info("Became the leader.")
        return nil
    case apierrors.IsAlreadyExists(err):
        existingOwners := existing.GetOwnerReferences()
        switch {
        case len(existingOwners) != 1:
            log.Info("Leader lock configmap must have exactly one owner reference.", "ConfigMap", existing)
        case existingOwners[0].Kind != "Pod":
            log.Info("Leader lock configmap owner reference must be a pod.", "OwnerReference", existingOwners[0])
        default:
            leaderPod := &corev1.Pod{}
            key = crclient.ObjectKey{Namespace: ns, Name: existingOwners[0].Name}
            err = client.Get(ctx, key, leaderPod)
            switch {
            case apierrors.IsNotFound(err):
                log.Info("Leader pod has been deleted, waiting for garbage collection do remove the lock.")
            case err != nil:
                return err
            case isPodEvicted(*leaderPod) && leaderPod.GetDeletionTimestamp() == nil:
                log.Info("Operator pod with leader lock has been evicted.", "leader", leaderPod.Name)
                log.Info("Deleting evicted leader.")
                // Pod may not delete immediately, continue with backoff
                err := client.Delete(ctx, leaderPod)
                if err != nil {
                    log.Error(err, "Leader pod could not be deleted.")
                }

            default:
                log.Info("Not the leader. Waiting.")
            }
        }

        select {
        case <-time.After(wait.Jitter(backoff, .2)):
            if backoff < maxBackoffInterval {
                backoff *= 2
            }
            continue
        case <-ctx.Done():
            return ctx.Err()
        }
    default:
        log.Error(err, "Unknown error creating ConfigMap")
        return err
    }
}
```

https://github.com/operator-framework/operator-sdk/blob/v0.13.0/pkg/leader/leader.go#L102-L153

## Leader-with-lease

Leader-with-lease は controller-runtime が提供している Leader election の仕組みです。

https://godoc.org/github.com/kubernetes-sigs/controller-runtime/pkg/leaderelection

中では client-go で提供されてる `leaderelection` package を使用しています。

https://godoc.org/k8s.io/client-go/tools/leaderelection

Leader-with-lease の仕組みは先程の Leader-for-life と同じように ConfigMap または Endpoints を用いて分散ロックを実現しています。
ロックを取得する Controller はリーダーになり, 取得できない Controller は待機します。
Leader-for-life と違う点はリース期間が設定されている点です。
リーダーは定期的にリースを更新しています。
リーダーが何らかの理由で死ぬとリースは期限切れになり, 待機していた Controller がリーダーを獲得しようとします。

[**この仕組みは現時点で Alpha であり将来的に大幅に変更または削除される可能性があることが記載されています。**](https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L50-L52)

使用する際には以下のように使用します。

```go
import (
  ...
  "sigs.k8s.io/controller-runtime/pkg/manager"
)

func main() {
  ...
  opts := manager.Options{
    ...
    LeaderElection: true,
    LeaderElectionID: "memcached-operator-lock"
  }
  mgr, err := manager.New(cfg, opts)
  ...
}
```

以下では詳しくコードを見ていきます。

### Manager の初期化

上記のコードで言うと以下の部分です。

```go
mgr, err := manager.New(cfg, opts)
```

`manager.New()` 関数の中で, Leader election に関する初期化を行っています。

```go
// Create the resource lock to enable leader election)
resourceLock, err := options.newResourceLock(config, recorderProvider, leaderelection.Options{
    LeaderElection:          options.LeaderElection,
    LeaderElectionID:        options.LeaderElectionID,
    LeaderElectionNamespace: options.LeaderElectionNamespace,
})
```

https://github.com/kubernetes-sigs/controller-runtime/blob/v0.4.0/pkg/manager/manager.go#L265-L270

`newResourceLock` は `setOptionsDefaults` という関数で初期化されています。

```go
// setOptionsDefaults set default values for Options fields
func setOptionsDefaults(options Options) Options {
    ...
    // Allow newResourceLock to be mocked
    if options.newResourceLock == nil {
        options.newResourceLock = leaderelection.NewResourceLock
    }
    ...
}
```

https://github.com/kubernetes-sigs/controller-runtime/blob/v0.4.0/pkg/manager/manager.go#L375-L378

`leaderelection.NewResourceLock()` 関数を見ていきます。
ここで client-go の `leaderelection` package を使用しているのがわかります。

```go
import (
    ...
    "k8s.io/client-go/tools/leaderelection/resourcelock"
    ...
)

// NewResourceLock creates a new config map resource lock for use in a leader
// election loop
func NewResourceLock(config *rest.Config, recorderProvider recorder.Provider, options Options) (resourcelock.Interface, error) {
    // TODO(JoelSpeed): switch to leaderelection object in 1.12
    return resourcelock.New(resourcelock.ConfigMapsResourceLock,
        options.LeaderElectionNamespace,
        options.LeaderElectionID,
        client.CoreV1(),
        client.CoordinationV1(),
        resourcelock.ResourceLockConfig{
            Identity:      id,
            EventRecorder: recorderProvider.GetEventRecorderFor(id),
        })
}
```

https://github.com/kubernetes-sigs/controller-runtime/blob/v0.4.0/pkg/leaderelection/leader_election.go#L82-L91

`resourcelock.New()` 関数の引数に `resourcelock.ConfigMapsResourceLock` という値を渡しており, controller-runtime では ConfigMap を用いる方法で分散ロックを実現していることがわかります。

### Leader election の実行

Manager は `Start()` という関数で実行されます。
Leader election の実行もこの中で行っています。

```go
func (cm *controllerManager) Start(stop <-chan struct{}) error {
    ...
    if cm.resourceLock != nil {
        err := cm.startLeaderElection()
        if err != nil {
            return err
        }
    } else {
        go cm.startLeaderElectionRunnables()
    }
    ...
```

https://github.com/kubernetes-sigs/controller-runtime/blob/v0.4.0/pkg/manager/internal.go#L424-L431

`cm.startLeaderElection()` 関数を見ていきます。
client-go の `leaderelection` package を使用して Leader election プロセスを開始してるのがわかります。

```go
import (
    ...
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
    ...
)

func (cm *controllerManager) startLeaderElection() (err error) {
    l, err := leaderelection.NewLeaderElector(leaderelection.LeaderElectionConfig{
        Lock:          cm.resourceLock,
        LeaseDuration: cm.leaseDuration,
        RenewDeadline: cm.renewDeadline,
        RetryPeriod:   cm.retryPeriod,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(_ context.Context) {
                cm.startLeaderElectionRunnables()
            },
            OnStoppedLeading: func() {
                // Most implementations of leader election log.Fatal() here.
                // Since Start is wrapped in log.Fatal when called, we can just return
                // an error here which will cause the program to exit.
                cm.errSignal.SignalError(fmt.Errorf("leader election lost"))
            },
        },
    })
    ...
    // Start the leader elector process
    go l.Run(ctx)
    ...
}
```

https://github.com/kubernetes-sigs/controller-runtime/blob/v0.4.0/pkg/manager/internal.go#L510-L544

### Leader election loop

1 つ前のセクションで見た下のコードの中を掘り下げていきます。

```go
// Start the leader elector process
go l.Run(ctx)
```

`Run()` 関数の中は以下のようになっています。

```go
// Run starts the leader election loop
func (le *LeaderElector) Run(ctx context.Context) {
	defer func() {
		runtime.HandleCrash()
		le.config.Callbacks.OnStoppedLeading()
	}()
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L189-L202

`le.acquire()` と `le.renew()` 関数はどちらも内部では `le.tryAcquireOrRenew()` 関数を呼び出しています。
`le.acquire()` では `le.tryAcquireOrRenew()` 関数が成功した場合は終了し, 失敗した場合は一定の周期でリトライされます。
`le.renew()` ではその逆で `le.tryAcquireOrRenew()` 関数が失敗した場合は終了し, 成功した場合は一定の周期でリトライされます。

わかりやすい言葉で説明すると, `le.acquire()` ではリーダーを獲得できるまでリトライを行い, `le.renew()` ではリーダー獲得できている間リトライを行います。
リーダーをロストした場合は defer で実行している `le.config.Callbacks.OnStoppedLeading()` というコールバック関数でエラーを返し, Controller を終了させます。
こうすることにより Kubernetes の Deployment 等の仕組みにより新しい Pod が起動し, また Leader election loop を実行することができます。

```go
Callbacks: leaderelection.LeaderCallbacks{
    OnStartedLeading: func(_ context.Context) {
        cm.startLeaderElectionRunnables()
    },
    OnStoppedLeading: func() {
        // Most implementations of leader election log.Fatal() here.
        // Since Start is wrapped in log.Fatal when called, we can just return
        // an error here which will cause the program to exit.
        cm.errSignal.SignalError(fmt.Errorf("leader election lost"))
    },
},
```

https://github.com/kubernetes-sigs/controller-runtime/blob/v0.4.0/pkg/manager/internal.go#L516-L525

<details>

<summary>le.acquire() Details</summary>

```go
// acquire loops calling tryAcquireOrRenew and returns true immediately when tryAcquireOrRenew succeeds.
// Returns false if ctx signals done.
func (le *LeaderElector) acquire(ctx context.Context) bool {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    succeeded := false
    desc := le.config.Lock.Describe()
    klog.Infof("attempting to acquire leader lease  %v...", desc)
    wait.JitterUntil(func() {
        succeeded = le.tryAcquireOrRenew()
        le.maybeReportTransition()
        if !succeeded {
            klog.V(4).Infof("failed to acquire lease %v", desc)
            return
        }
        le.config.Lock.RecordEvent("became leader")
        le.metrics.leaderOn(le.config.Name)
        klog.Infof("successfully acquired lease %v", desc)
        cancel()
    }, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
    return succeeded
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L228-L249

</details>

<details>

<summary>le.renew() Details</summary>

```go
// renew loops calling tryAcquireOrRenew and returns immediately when tryAcquireOrRenew fails or ctx signals done.
func (le *LeaderElector) renew(ctx context.Context) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    wait.Until(func() {
        timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
        defer timeoutCancel()
        err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
            done := make(chan bool, 1)
            go func() {
                defer close(done)
                done <- le.tryAcquireOrRenew()
            }()

            select {
            case <-timeoutCtx.Done():
                return false, fmt.Errorf("failed to tryAcquireOrRenew %s", timeoutCtx.Err())
            case result := <-done:
                return result, nil
            }
        }, timeoutCtx.Done())

        le.maybeReportTransition()
        desc := le.config.Lock.Describe()
        if err == nil {
            klog.V(5).Infof("successfully renewed lease %v", desc)
            return
        }
        le.config.Lock.RecordEvent("stopped leading")
        le.metrics.leaderOff(le.config.Name)
        klog.Infof("failed to renew lease %v: %v", desc, err)
        cancel()
    }, le.config.RetryPeriod, ctx.Done())

    // if we hold the lease, give it up
    if le.config.ReleaseOnCancel {
        le.release()
    }
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L251-L289

</details>

以下では, `le.tryAcquireOrRenew()` 関数の中を詳しく見ていきます。

### le.tryAcquireOrRenew()

<details>

<summary> le.tryAcquireOrRenew() Details</summary>

```go
// tryAcquireOrRenew tries to acquire a leader lease if it is not already acquired,
// else it tries to renew the lease if it has already been acquired. Returns true
// on success else returns false.
func (le *LeaderElector) tryAcquireOrRenew() bool {
    now := metav1.Now()
    leaderElectionRecord := rl.LeaderElectionRecord{
        HolderIdentity:       le.config.Lock.Identity(),
        LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
        RenewTime:            now,
        AcquireTime:          now,
    }

    // 1. obtain or create the ElectionRecord
    oldLeaderElectionRecord, err := le.config.Lock.Get()
    if err != nil {
        if !errors.IsNotFound(err) {
            klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
            return false
        }
        if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
            klog.Errorf("error initially creating leader election record: %v", err)
            return false
        }
        le.observedRecord = leaderElectionRecord
        le.observedTime = le.clock.Now()
        return true
    }

    // 2. Record obtained, check the Identity & Time
    if !reflect.DeepEqual(le.observedRecord, *oldLeaderElectionRecord) {
        le.observedRecord = *oldLeaderElectionRecord
        le.observedTime = le.clock.Now()
    }
    if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
        le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
        !le.IsLeader() {
        klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
        return false
    }

    // 3. We're going to try to update. The leaderElectionRecord is set to it's default
    // here. Let's correct it before updating.
    if le.IsLeader() {
        leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
        leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
    } else {
        leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
    }

    // update the lock itself
    if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
        klog.Errorf("Failed to update lock: %v", err)
        return false
    }
    le.observedRecord = leaderElectionRecord
    le.observedTime = le.clock.Now()
    return true
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L308-L365

</details>

#### 更新に使用する LeaderElectionRecord の作成

はじめに, 更新に使用する LeaderElectionRecord の作成を行います。

LeaderElectionRecord とは, ロックに用いる Object の `control-plane.alpha.kubernetes.io/leader` annotation に JSON で格納される struct です。
誰がロックを獲得しているかなどの情報が格納されており, この情報を元にロックの制御を行います。

```go
now := metav1.Now()
leaderElectionRecord := rl.LeaderElectionRecord{
    HolderIdentity:       le.config.Lock.Identity(),
    LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
    RenewTime:            now,
    AcquireTime:          now,
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L312-L318

<details>

<summary>LeaderElectionRecord Details</summary>

```go
// LeaderElectionRecord is the record that is stored in the leader election annotation.
// This information should be used for observational purposes only and could be replaced
// with a random string (e.g. UUID) with only slight modification of this code.
// TODO(mikedanese): this should potentially be versioned
type LeaderElectionRecord struct {
    // HolderIdentity is the ID that owns the lease. If empty, no one owns this lease and
    // all callers may acquire. Versions of this library prior to Kubernetes 1.14 will not
    // attempt to acquire leases with empty identities and will wait for the full lease
    // interval to expire before attempting to reacquire. This value is set to empty when
    HolderIdentity       string      `json:"holderIdentity"`
    LeaseDurationSeconds int         `json:"leaseDurationSeconds"`
    AcquireTime          metav1.Time `json:"acquireTime"`
    RenewTime            metav1.Time `json:"renewTime"`
    LeaderTransitions    int         `json:"leaderTransitions"`
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/resourcelock/interface.go#L35-L50

</details>

#### LeaderElectionRecord の取得

LeaderElectionRecord の取得 or 作成を行います。

```go
// 1. obtain or create the ElectionRecord
oldLeaderElectionRecord, err := le.config.Lock.Get()
if err != nil {
    if !errors.IsNotFound(err) {
        klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
        return false
    }
    if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
        klog.Errorf("error initially creating leader election record: %v", err)
        return false
    }
    le.observedRecord = leaderElectionRecord
    le.observedTime = le.clock.Now()
    return true
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L320-L334

#### ロックの所有者とリース時間の確認

取得した LeaderElectionRecord の情報を取り込み, ロックの所有者と時間の確認を行います。

現在のロックを保持しているのが自身ではなく, 最後に確認した時間がリース時間を超えていない場合は, 現在のロックは他人に確保されているとみなし, `false` を return します。
これより下のロジックは, 自身がリーダーである場合かリーダーの入れ替えを行う際に通過します。

```go
// 2. Record obtained, check the Identity & Time
if !reflect.DeepEqual(le.observedRecord, *oldLeaderElectionRecord) {
    le.observedRecord = *oldLeaderElectionRecord
    le.observedTime = le.clock.Now()
}
if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
    le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
    !le.IsLeader() {
    klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
    return false
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L336-L346

#### LeaderElectionRecord フィールドの更新

更新用の LeaderElectionRecord のフィールドを更新します。

自身がリーダーである場合は `AcquireTime` には古い値を使用します。
リーダーの入れ替え時には `LeaderTransitions` に 1 を加算します。

```go
// 3. We're going to try to update. The leaderElectionRecord is set to it's default
// here. Let's correct it before updating.
if le.IsLeader() {
    leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
    leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
} else {
    leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
}
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L348-L355

#### ロック情報の更新

ロックの情報を更新します。

ロックには Kubernetes の Object を用いているため Kubernetes API の Atomicity の仕組みを利用することができます。
Kubernetes の API Server は Object の 更新時に指定された `resourceVersion` と現在の `resourceVersion` が一致することを確認し, 更新操作の間に他の更新が行われていないことを確認します。

[Kubernetes API Concepts #Resource Versions - Kubernetes](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions)

```go
// update the lock itself
if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
    klog.Errorf("Failed to update lock: %v", err)
    return false
}
le.observedRecord = leaderElectionRecord
le.observedTime = le.clock.Now()
return true
```

https://github.com/kubernetes/client-go/blob/v12.0.0/tools/leaderelection/leaderelection.go#L357-L364

## Reference

* [Configuring leader election - Operator SDK | Applications | OpenShift Container Platform 4.1](https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-leader-election.html)
* [11.6. リーダー選択の設定 4.2 | Red Hat Customer Portal](https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.2/html/operators/osdk-leader-election)
* [kubernetes 自定义控制器的高可用](http://blog.fatedier.com/2019/04/17/k8s-custom-controller-high-available/)
