

# Tutorial Building CronJob

## Scaffolding Out Our Project

```bash
# create a project directory, and then run the init command.
mkdir project
cd project
# we'll use a domain of tutorial.kubebuilder.io,
# so all API groups will be <group>.tutorial.kubebuilder.io.
kubebuilder init --domain tutorial.kubebuilder.io --repo tutorial.kubebuilder.io/project
```

通过`--project-name=` 设置不同的project名称。

## main.go

`main.go`中包含的内容

#### import package

- 核心 [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime?tab=doc) 库
- 默认controller-runtime logging, Zap (more on that a bit later)

```go
package main

import (
    "flag"
    "fmt"
    "os"

    // Import all Kubernetes client auth plugins (e.g. Azure, GCP, OIDC, etc.)
    // to ensure that exec-entrypoint and run can make use of them.
    _ "k8s.io/client-go/plugin/pkg/client/auth"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/cache"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    // +kubebuilder:scaffold:imports
)
```

#### Scheme

每组控制器都需要一个 [*Scheme*](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#err-but-whats-that-scheme-thing)，它提供了Kinds与其对应的Go type之间的映射。

```go
var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))

    //+kubebuilder:scaffold:scheme
}
```

#### 设置flag和实例化manager

此时我们的main function还比较简单

- 为metrics(指标)设置了一些basic flags(基本标志)。
- 实例化一个 [*manager*](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager?tab=doc#Manager)，它跟踪运行所有的controller，以及设置共享缓存和客户端到 API server（注意：我们将我们的 Scheme 告诉了 manager）。
- 运行manager，它依次运行我们所有的controllers和webhooks。manager设置为运行，直到它收到graceful shutdown signal（正常关闭信号）。这样，当我们在 Kubernetes 上运行时，我们的行为会很好地终止 pod。

要记住`+kubebuilder:scaffold:builder`注释所在位置( 那里很快就会变得有趣。)

```go
func main() {
    var metricsAddr string
    var enableLeaderElection bool
    var probeAddr string
    flag.StringVar(&metricsAddr, "metrics-bind-address", ":8080", "The address the metric endpoint binds to.")
    flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081", "The address the probe endpoint binds to.")
    flag.BoolVar(&enableLeaderElection, "leader-elect", false,
        "Enable leader election for controller manager. "+
            "Enabling this will ensure there is only one active controller manager.")
    opts := zap.Options{
        Development: true,
    }
    opts.BindFlags(flag.CommandLine)
    flag.Parse()

    ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        MetricsBindAddress:     metricsAddr,
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
```

**注意：Manager 可以通过以下方式限制所有controller将要watch的资源的namespace**：

- 单个的namespace

```go
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        Namespace:              namespace,
        MetricsBindAddress:     metricsAddr,
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })
```

上面的例子将项目范围改成了单个的namespace。在这种情况下，还建议通过将默认的 `ClusterRole` 和` ClusterRoleBinding` 分别替换为 `Role` 和 `RoleBinding` 来限制对该命名空间提供的授权。

- watch**一组**特定的namespace

此外，可以使用 `MultiNamespacedCacheBuilder` 来watch**一组**特定的namespace：

```go
    var namespaces []string // List of Namespaces

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:                 scheme,
        NewCache:               cache.MultiNamespacedCacheBuilder(namespaces),
        MetricsBindAddress:     fmt.Sprintf("%s:%d", metricsHost, metricsPort),
        Port:                   9443,
        HealthProbeBindAddress: probeAddr,
        LeaderElection:         enableLeaderElection,
        LeaderElectionID:       "80807133.tutorial.kubebuilder.io",
    })
```

> 有关更多信息，请参阅[MultiNamespacedCacheBuilder](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache?tab=doc#MultiNamespacedCacheBuilder)

#### 运行manager

- 运行manager，它依次运行我们所有的controllers和webhooks。manager设置为运行，直到它收到graceful shutdown signal（正常关闭信号）。这样，当我们在 Kubernetes 上运行时，我们的行为会很好地终止 pod。

```go
	// +kubebuilder:scaffold:builder

    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up health check")
        os.Exit(1)
    }
    if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
        setupLog.Error(err, "unable to set up ready check")
        os.Exit(1)
    }

    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```

> 要记住`+kubebuilder:scaffold:builder`注释所在位置( 那里很快就会变得有趣)

## 1.3 GVK

https://book.kubebuilder.io/cronjob-tutorial/gvks.html

## 1.4 Adding a new API

使用`kubebuilder create api`来搭建一个新的 Kind和其相应的controller：（创建GVK为group为batch，version为v1和Kind为CronJob的api）

```bash
kubebuilder create api --group batch --version v1 --kind CronJob
```

> 按`y`“创建resource”和“创建controller”。

当第一次为每个Group Version调用这个命令时，它都将会为新的Group Version创建一个目录。

该command，创建了 [`api/v1/`](https://sigs.k8s.io/kubebuilder/docs/book/src/cronjob-tutorial/testdata/project/api/v1) 目录，对应于 `batch.tutorial.kubebuilder.io/v1`（domain之前设置为`tutorial.kubebuilder.io/v1`）。并在`api/v1`目录之下为`CronJob`Kind创建了`cronjob_types.go`。(每当为不同的Kind调用该command时，则都会添加相关的new file)

```
├── api
│   └── v1
│       ├── cronjob_types.go
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go

```

### cronjob_types.go内容

#### import的package

导入`meta/v1` API group，它通常不会自己公开，而是包含所有 Kubernetes Kind通用的metadata。

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

#### 定义Spec 和 Status的type

Kubernetes 通过reconcile期望状态 ( `Spec`) 与实际状态 (其他对象 `Status`) 和外部状态，然后记录它观察到的 ( `Status`) 来运行。因此，每个*功能*对象都包含规范和状态。（不过一些类型，比如 `ConfigMap`不遵循这种模式，因为它们不编码所需的状态，但大多数类型都这样做。）

```go
// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file
}

// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file
}
```

#### CronJob和CronJobList的定义

接下来，定义`CronJob`以及`CronJobList`。

- `CronJob`是我们的根类型(root type)，并描述了`CronJob`Kind。如同所有 Kubernetes 对象一样，它包含 `TypeMeta`（描述API version and Kind），也包含`ObjectMeta`，它包含name, namespace, and labels等内容。

- `CronJobList`是CronJob的List类型，能包含多个CronJob类型，是用于批量操作的 Kind。

一般来说，不直接修改`CronJob`以及`CronJobList`，而是去修改`Spec`或`Status`

注意：`+kubebuilder:object:root=true`注释被称为marker（标记）。它们充当额外的metadata的作用，提供给 [controller-tools](https://github.com/kubernetes-sigs/controller-tools) (code和YAML 的生成器)额外信息。`+kubebuilder:object:root=true`告诉`object` generator该type代表了一个Kind。接下来，`object`generator 将为我们生成 [runtime.Object](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Object)接口的实现。而`runtime.Object`接口则是所有代表了 Kinds 的type都必须实现的标准接口。

```go
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API
type CronJob struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CronJobSpec   `json:"spec,omitempty"`
    Status CronJobStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []CronJob `json:"items"`
}
```

#### 添加Go type到API Group

最后，我们将 Go type添加到 API group中。这允许我们将此 API Group中的type添加到任意[Scheme](https://pkg.go.dev/k8s.io/apimachinery/pkg/runtime?tab=doc#Scheme)中。

```go
func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

## 1.5 Designing an API

命名方式：所有的序列化字段必须使用`camelCase`，使用Json struct tags去指定。使用`omitempty`struct tag去标记一个field，当它为empty时应该从序列化中省略。

field可以使用大多数原始类型。而数字是例外，接受三种形式的数字：`int32` 和`int64`为整数，并且`resource.Quantity`为小数。还有另一种特殊类型：`metav1.Time`，它与`time.Time`相同，只是它具有固定的、可移植的序列化格式。

`vim project/api/v1/cronjob_types.go`

- 需要import的package

```go
package v1

import (
    batchv1beta1 "k8s.io/api/batch/v1beta1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

```

### CronJob对象

#### Spec

spec定义了desired state。

CronJob所需的基本部分:

- A schedule (the *cron* in CronJob)
- A template for the Job to run (the *job* in CronJob)

额外部分：

- starting jobs的deadline（如果我们错过了这个deadline，我们将等到下一个scheduled time）
- 如果同时运行multiple jobs怎么办（do we wait? stop the old one? run both?）
- 一种暂停 CronJob 运行的方法，以防它出现问题
- 对old job history的限制

> 注意：由于我们从不读取自己的status，因此我们需要通过其他方式来跟踪job是否已运行。我们可以使用至少一项old job来做到这一点。

##### CronJobSepc的具体定义

使用几个markers(`// +comment`)来指定额外的metadata。这些将在生成我们的 CRD manifest时由[controller-tools](https://github.com/kubernetes-sigs/controller-tools)使用。controller-tools还将使用 GoDoc 来形成字段的描述。

```go
// CronJobSpec defines the desired state of CronJob
type CronJobSpec struct {
    //+kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
    Schedule string `json:"schedule"`

    //+kubebuilder:validation:Minimum=0

    // Optional deadline in seconds for starting the job if it misses scheduled
    // time for any reason.  Missed jobs executions will be counted as failed ones.
    // +optional
    StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

    // Specifies how to treat concurrent executions of a Job.并发执行策略
    // Valid values are:
    // - "Allow" (default): allows CronJobs to run concurrently;
    // - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
    // - "Replace": cancels currently running job and replaces it with a new one
    // +optional
    ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

    // This flag tells the controller to suspend subsequent executions, it does
    // not apply to already started executions.  Defaults to false.
    // +optional
    Suspend *bool `json:"suspend,omitempty"`

    // Specifies the job that will be created when executing a CronJob.
    JobTemplate batchv1beta1.JobTemplateSpec `json:"jobTemplate"`

    //+kubebuilder:validation:Minimum=0

    // The number of successful finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

    //+kubebuilder:validation:Minimum=0

    // The number of failed finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```

我们定义了`ConcurrencyPolicy`这个自定义类型来保存并发策略。它实际上只是a string under the hood(`type ConcurrencyPolicy string`)，但是type提供了额外的文档，并允许我们在type上而不是field上添加验证，使validation更容易被重新利用。

```go
// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
    // AllowConcurrent allows CronJobs to run concurrently.
    AllowConcurrent ConcurrencyPolicy = "Allow"

    // ForbidConcurrent forbids concurrent runs, skipping next run if previous
    // hasn't finished yet.
    ForbidConcurrent ConcurrencyPolicy = "Forbid"

    // ReplaceConcurrent cancels currently running job and replaces it with a new one.
    ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```



#### Status

接下来定义status。它保存观察到的状态。它包含我们希望用户或其他控制者能够轻松获取的任何信息。

field的定义：

- a list of actively running jobs
- the last time that we successfully ran our job. 

注意，使用`metav1.Time`代替`time.Time`来获得稳定的序列化。

```go
// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // A list of pointers to currently running jobs.
    // +optional
    Active []corev1.ObjectReference `json:"active,omitempty"`

    // Information when was the last time the job was successfully scheduled.
    // +optional
    LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`
}
```

`//+kubebuilder:subresource:status`给type`CronJob`标记了一个status子资源，这样我们的行为就像内置的 kubernetes 类型

```go
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// CronJob is the Schema for the cronjobs API
type CronJob struct {
// Root Object Definitions
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   CronJobSpec   `json:"spec,omitempty"`
    Status CronJobStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []CronJob `json:"items"`
}

func init() {
    SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

### `api/v1`路径下其他文件

#### `groupversion_info.go`



#### `zz_generated.deepcopy.go`

`zz_generated.deepcopy.go`包含了`runtime.Object`接口的实现（自动生成的），它将我们所有的root type标记为representing Kinds。

`runtime.Object`接口的核心是一个deep-copy method， `DeepCopyObject`.

`controller-tools`的object generator还生成了其他两个方便的方法：`DeepCopy`和 `DeepCopyInto`，用于每个root type和它的所有子类型

## 1.6 Controller

Reconciling(协调)：确保对于任何给定的object，actual state(实际状态)与对象的desired state(所需状态)相匹配。(每个controller专注于一个 *root* Kind，但可以与其他 Kind 交互。)

### controller的基本结构

#### 导入的package

`vim cronjob_controller.go`

- import package。需要的package有 core controller-runtime，clinet package和API types的package。

```go
package controllers

import (
    "context"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```

#### reconciler struct 

kubebuilder 为我们搭建了一个基本的reconciler struct。几乎每个reconciler都需要记录日志，并且需要能够获取对象，所以这些都是开箱即用的。

```go
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}
```

大多数controller都会在cluster上运行，因此它们需要RBAC权限。使用controller-tools的RBAC markers指定，赋予运行所需的最低限度的权限。随着我们添加更多功能，我们需要重新审视这些。

```go
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
```

#### Reconcile方法：

传入参数：`Reconcile`实际上是为单一的named object执行reconciling。虽然传入的[Request](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile?tab=doc#Request)类型的参数只有一个名称，但我们可以使用client从缓存中获取该object。

> ```go
> type Request struct {
> 	// NamespacedName is the name and namespace of the object to reconcile.
> 	types.NamespacedName
> }
> ```

返回值：我们返回了一个empty result and no error(`ctrl.Result{}, nil`)，这表明我们已经成功地reconciled了该object并且不需要try again，直到有一些变化产生。

大多数controller需要一个日志句柄（logging handle）和一个上下文（context），所以我们在这里设置它们。

- 该[context](https://golang.org/pkg/context/)用于允许请求的取消，并有可能之类的东西跟踪。它是所有客户端方法(client methods)的第一个参数。该`Background`context是不具有任何extra data或timing restrictions的basic context。

- 日志句柄让我们记录。控制器运行时通过名为[logr](https://github.com/go-logr/logr)的库使用结构化日志记录。我们很快就会看到，日志记录的工作原理是将键值对附加到静态消息中。我们可以在我们的 reconcile 方法的顶部预先分配一些对，将这些对附加到这个 reconciler 中的所有日志行。

```go
func (r *CronJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}
```

#### 把reconciler添加到manager

最后将该reconciler添加到manager，以便在manager启动时该reconciler也会启动。（例子中是让manager监控CronJob类）

现在，我们只注意到这个reconciler operates 在`CronJob`s 上运行。稍后，我们也将使用它来标记我们关心的其他相关对象。

```go
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Complete(r)
}
```

接下来填写`CronJob`s的逻辑。

## 1.7 实现controller

CronJob controller的basic logic：

1. Load the named CronJob
2. List all active jobs, and update the status
3. Clean up old jobs according to the history limits
4. Check if we’re suspended (and don’t do anything else if we are)查看是否被暂停，如果被暂停了则什么也不做
5. Get the next scheduled run
6. Run a new job if it’s on schedule, not past the deadline, and not blocked by our concurrency policy
7. Requeue when we either see a running job (done automatically) or it’s time for the next scheduled run.

修改controller文件

`vim project/controllers/cronjob_controller.go`

### import package

```go
package controllers

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"

    batchv1 "tutorial.kubebuilder.io/project/api/v1"
)
```

### reconciler struct

在reconciler中添加一个Clock字段，allow us to fake timing in our tests.

```go
// CronJobReconciler reconciles a CronJob object
type CronJobReconciler struct {
    client.Client
    Scheme *runtime.Scheme
    Clock
}
```

我们将模拟时钟，以便在测试时更容易及时跳转，“真实”时钟只调用`time.Now`.

```go
type realClock struct{}

func (_ realClock) Now() time.Time { return time.Now() }

// clock knows how to get the current time.
// It can be used to fake out timing for testing.
type Clock interface {
    Now() time.Time
}
```

注意：我们需要更多的RBAC权限——因为我们现在正在创建和管理jobs，

```go
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/finalizers,verbs=update
//+kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get
```

接下来，实现controller的核心，Reconcile的逻辑

### Reconcile的逻辑（重点）

#### 1.Load the CronJob by name

使用client获取CronJob。所有的client method都将context（以允许取消）作为它们的第一个参数。Get有些特殊，它将NamespacedName作为中间参数。

```go
    var cronJob batchv1.CronJob
    if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
        log.Error(err, "unable to fetch CronJob")
        // we'll ignore not-found errors, since they can't be fixed by an immediate
        // requeue (we'll need to wait for a new notification), and we can get them
        // on deleted requests.
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
```

#### 2: List all active jobs, and update the status

为了完全更新我们的status，我们需要列出这个namespace中属于这个 CronJob 的所有child jobs。与 Get 类似，可以使用 List method列出child jobs。注意：我们使用可变参数(variadic)选项来设置namespace与field相匹配（这实际上是我们在下面设置的索引查找）。

```go
    var childJobs kbatch.JobList
    if err := r.List(ctx, &childJobs, client.InNamespace(req.Namespace), client.MatchingFields{jobOwnerKey: req.Name}); err != nil {
        log.Error(err, "unable to list child Jobs")
        return ctrl.Result{}, err
    }
```

> `ListOption`的设置目的：reconciler获取该状态的 cronjob 拥有的所有job。随着我们的 cronjobs 数量的增加，查找这些可能会变得非常缓慢，因为我们必须过滤所有这些job。为了更有效的查找，这些job将在controller的名称上进行本地索引。jobOwnerKey field被添加到缓存的job obeject中，该key作为index去引用owning controller and functions。在本文档的后面，我们将配置controller去实际上索引该filed。

一旦获取我们所拥有的所有job，我们会将job分成`active`、 `successful`和`failed`，并跟踪最近的job运行，以便可以将其记录在status中。请记住，status应该从the state of the world中重构，因此从root object中读取status并不是一个好主意。相反，应该在每次运行时重建它。这就是我们在这里要做的。

我们可以使用status conditions检查job是否“finished”以及它是successful还是failed。把这个逻辑放在一个helper function中，使代码更整洁。

```go
    // find the active list of jobs
    var activeJobs []*kbatch.Job
    var successfulJobs []*kbatch.Job
    var failedJobs []*kbatch.Job
    var mostRecentTime *time.Time // find the last run so we can update the status
```

```go
   //isJobFinished
   // getScheduledTimeForJob

    for i, job := range childJobs.Items {
        _, finishedType := isJobFinished(&job)
        switch finishedType {
        case "": // ongoing
            activeJobs = append(activeJobs, &childJobs.Items[i])
        case kbatch.JobFailed:
            failedJobs = append(failedJobs, &childJobs.Items[i])
        case kbatch.JobComplete:
            successfulJobs = append(successfulJobs, &childJobs.Items[i])
        }

        // We'll store the launch time in an annotation, so we'll reconstitute that from
        // the active jobs themselves.
        scheduledTimeForJob, err := getScheduledTimeForJob(&job)
        if err != nil {
            log.Error(err, "unable to parse schedule time for child job", "job", &job)
            continue
        }
        if scheduledTimeForJob != nil {
            if mostRecentTime == nil {
                mostRecentTime = scheduledTimeForJob
            } else if mostRecentTime.Before(*scheduledTimeForJob) {
                mostRecentTime = scheduledTimeForJob
            }
        }
    }

    if mostRecentTime != nil {
        cronJob.Status.LastScheduleTime = &metav1.Time{Time: *mostRecentTime}
    } else {
        cronJob.Status.LastScheduleTime = nil
    }
    cronJob.Status.Active = nil
    for _, activeJob := range activeJobs {
        jobRef, err := ref.GetReference(r.Scheme, activeJob)
        if err != nil {
            log.Error(err, "unable to make reference to active job", "job", activeJob)
            continue
        }
        cronJob.Status.Active = append(cronJob.Status.Active, *jobRef)
    }
```

在这里，我们使用稍高的日志级别，记录我们观察到的job数量，用于 debugging。请注意我们如何使用固定消息而不是使用格式字符串，并附加带有额外信息的键值对。这使得过滤和查询日志行变得更加容易。

```go
	log.V(1).Info("job count", "active jobs", len(activeJobs), "successful jobs", len(successfulJobs), "failed jobs", len(failedJobs))
```



更新status



#### 3: Clean up old jobs according to the history limit

首先，我们将尝试清理old jobs，以免留下太多闲置。

```go
  // NB: deleting these is "best effort" -- if we fail on a particular one,
    // we won't requeue just to finish the deleting.
    if cronJob.Spec.FailedJobsHistoryLimit != nil {
        sort.Slice(failedJobs, func(i, j int) bool {
            if failedJobs[i].Status.StartTime == nil {
                return failedJobs[j].Status.StartTime != nil
            }
            return failedJobs[i].Status.StartTime.Before(failedJobs[j].Status.StartTime)
        })
        for i, job := range failedJobs {
            if int32(i) >= int32(len(failedJobs))-*cronJob.Spec.FailedJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete old failed job", "job", job)
            } else {
                log.V(0).Info("deleted old failed job", "job", job)
            }
        }
    }

    if cronJob.Spec.SuccessfulJobsHistoryLimit != nil {
        sort.Slice(successfulJobs, func(i, j int) bool {
            if successfulJobs[i].Status.StartTime == nil {
                return successfulJobs[j].Status.StartTime != nil
            }
            return successfulJobs[i].Status.StartTime.Before(successfulJobs[j].Status.StartTime)
        })
        for i, job := range successfulJobs {
            if int32(i) >= int32(len(successfulJobs))-*cronJob.Spec.SuccessfulJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); (err) != nil {
                log.Error(err, "unable to delete old successful job", "job", job)
            } else {
                log.V(0).Info("deleted old successful job", "job", job)
            }
        }
    }
```

#### 4：检查我们是否被暂停

如果这个object被suspended（挂起），则表示我们不想运行任何jobs，所以现在停止。如果我们正在运行的job出现了问题并且想要暂停运行以进行调查或使用集群而不删除对象，这将非常有用。

```go
    if cronJob.Spec.Suspend != nil && *cronJob.Spec.Suspend {
        log.V(1).Info("cronjob suspended, skipping")
        return ctrl.Result{}, nil
    }
```

#### 5.获取下一次运行

如果我们没有暂停，我们将需要计算next scheduled run，以及是否还有一个尚未处理的运行。

使用 cron 库来计算next scheduled time。从上次运行时间开始计算，如果找不到最后一次运行时间(`cronJob.Status.LastScheduleTime.Time`)，则从 CronJob 的创建时间开始计算(`cronJob.ObjectMeta.CreationTimestamp.Time`)。

如果错过了太多的运行而且没有设置任何deadline，我们将bail(保释，放弃)，这样就不会导致controller restarts 或wedges出现问题。

否则，我们将只返回错过的运行（我们将只使用最新的运行）和下一次运行，以便我们可以知道什么时候再次进行reconcile。

```go
    getNextSchedule := func(cronJob *batchv1.CronJob, now time.Time) (lastMissed time.Time, next time.Time, err error) {
        sched, err := cron.ParseStandard(cronJob.Spec.Schedule)
        if err != nil {
            return time.Time{}, time.Time{}, fmt.Errorf("Unparseable schedule %q: %v", cronJob.Spec.Schedule, err)
        }

        // for optimization purposes, cheat a bit and start from our last observed run time
        // we could reconstitute this here, but there's not much point, since we've
        // just updated it.
        var earliestTime time.Time
        if cronJob.Status.LastScheduleTime != nil {
            earliestTime = cronJob.Status.LastScheduleTime.Time
        } else {
            earliestTime = cronJob.ObjectMeta.CreationTimestamp.Time
        }
        if cronJob.Spec.StartingDeadlineSeconds != nil {
            // controller is not going to schedule anything below this point
            schedulingDeadline := now.Add(-time.Second * time.Duration(*cronJob.Spec.StartingDeadlineSeconds))

            if schedulingDeadline.After(earliestTime) {
                earliestTime = schedulingDeadline
            }
        }
        if earliestTime.After(now) {
            return time.Time{}, sched.Next(now), nil
        }

        starts := 0
        for t := sched.Next(earliestTime); !t.After(now); t = sched.Next(t) {
            lastMissed = t
            // An object might miss several starts. For example, if
            // controller gets wedged on Friday at 5:01pm when everyone has
            // gone home, and someone comes in on Tuesday AM and discovers
            // the problem and restarts the controller, then all the hourly
            // jobs, more than 80 of them for one hourly scheduledJob, should
            // all start running with no further intervention (if the scheduledJob
            // allows concurrency and late starts).
            //
            // However, if there is a bug somewhere, or incorrect clock
            // on controller's server or apiservers (for setting creationTimestamp)
            // then there could be so many missed start times (it could be off
            // by decades or more), that it would eat up all the CPU and memory
            // of this controller. In that case, we want to not try to list
            // all the missed start times.
            starts++
            if starts > 100 {
                // We can't get the most recent times so just return an empty slice
                return time.Time{}, time.Time{}, fmt.Errorf("Too many missed start times (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.")
            }
        }
        return lastMissed, sched.Next(now), nil
    }
```

```go
    // figure out the next times that we need to create
    // jobs at (or anything we missed).
    missedRun, nextRun, err := getNextSchedule(&cronJob, r.Now())
    if err != nil {
        log.Error(err, "unable to figure out CronJob schedule")
        // we don't really care about requeuing until we get an update that
        // fixes the schedule, so don't return an error
        return ctrl.Result{}, nil
    }
```

我们将准备eventual request(最终请求)以重新排队直到下一个job，然后确定我们是否真的需要运行。

```go
    scheduledResult := ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())} // save this so we can re-use it elsewhere
    log = log.WithValues("now", r.Now(), "next run", nextRun)
```

#### 6：如果new job按计划运行，没有超过deadline，并且没有被我们的并发策略阻止，则运行new job

如果我们错过了一次运行，而我们仍在启动它的deadline内，我们将需要运行job。

```go
    if missedRun.IsZero() {
        log.V(1).Info("no upcoming scheduled times, sleeping until next")
        return scheduledResult, nil
    }

    // make sure we're not too late to start the run
    log = log.WithValues("current run", missedRun)
    tooLate := false
    if cronJob.Spec.StartingDeadlineSeconds != nil {
        tooLate = missedRun.Add(time.Duration(*cronJob.Spec.StartingDeadlineSeconds) * time.Second).Before(r.Now())
    }
    if tooLate {
        log.V(1).Info("missed starting deadline for last run, sleeping till next")
        // TODO(directxman12): events
        return scheduledResult, nil
    }
```

如果我们真的必须运行一个job，我们需要等待现有的job完成，替换现有的，或者只是添加新的。如果我们的信息由于缓存延迟而过时，我们将在获得最新信息时重新排队。

```go
    // figure out how to run this job -- concurrency policy might forbid us from running
    // multiple at the same time...
    if cronJob.Spec.ConcurrencyPolicy == batchv1.ForbidConcurrent && len(activeJobs) > 0 {
        log.V(1).Info("concurrency policy blocks concurrent runs, skipping", "num active", len(activeJobs))
        return scheduledResult, nil
    }

    // ...or instruct us to replace existing ones...
    if cronJob.Spec.ConcurrencyPolicy == batchv1.ReplaceConcurrent {
        for _, activeJob := range activeJobs {
            // we don't care if the job was already deleted
            if err := r.Delete(ctx, activeJob, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete active job", "job", activeJob)
                return ctrl.Result{}, err
            }
        }
    }
```

一旦弄清楚如何处理现有的job，我们将实际创建desired job

根据 CronJob 的template构建一个job。从template中复制spec并复制一些basic object meta。

然后，设置“`scheduled time`”annotation，以便可以在每次 reconcile时重新构建`LastScheduleTime`字段。

最后，需要设置一个owner reference（所有者引用）。这允许 Kubernetes 垃圾收集器在删除 CronJob 时去清理job，并允许controller-runtime确定在给定 job发生changes（added, deleted, completes等）时，需要reconcile哪个 cronjob。

```go
    constructJobForCronJob := func(cronJob *batchv1.CronJob, scheduledTime time.Time) (*kbatch.Job, error) {
        // We want job names for a given nominal start time to have a deterministic name to avoid the same job being created twice
        name := fmt.Sprintf("%s-%d", cronJob.Name, scheduledTime.Unix())

        job := &kbatch.Job{
            ObjectMeta: metav1.ObjectMeta{
                Labels:      make(map[string]string),
                Annotations: make(map[string]string),
                Name:        name,
                Namespace:   cronJob.Namespace,
            },
            Spec: *cronJob.Spec.JobTemplate.Spec.DeepCopy(),
        }
        for k, v := range cronJob.Spec.JobTemplate.Annotations {
            job.Annotations[k] = v
        }
        job.Annotations[scheduledTimeAnnotation] = scheduledTime.Format(time.RFC3339)
        for k, v := range cronJob.Spec.JobTemplate.Labels {
            job.Labels[k] = v
        }
        if err := ctrl.SetControllerReference(cronJob, job, r.Scheme); err != nil {
            return nil, err
        }

        return job, nil
    }
```

```go
    // actually make the job...
    job, err := constructJobForCronJob(&cronJob, missedRun)
    if err != nil {
        log.Error(err, "unable to construct job from template")
        // don't bother requeuing until we get a change to the spec
        return scheduledResult, nil
    }

    // ...and create it on the cluster
    if err := r.Create(ctx, job); err != nil {
        log.Error(err, "unable to create Job for CronJob", "job", job)
        return ctrl.Result{}, err
    }

    log.V(1).Info("created Job for CronJob run", "job", job)
```

#### 7：当我们看到正在运行的作业或下一次计划运行的时间时重新排队

最后，return result，即我们希望在需要进行下一次运行时重新排队。这被视为maximum deadline（最大截止日期）——如果中间有其他事情发生变化，比如job开始或完成，被修改等，我们可能会更快地再次进行reconcile。

```go
    // we'll requeue once we see the running job, and update our status
    return scheduledResult, nil
}
```

#### 设置

最后，update setup。

为了让Reconciler能够通过Job的owner从而快速地查找到Jobs，我们需要一个index。

首先声明一个index key索引键(`jobOwnerKey`)，将其作为pseudo-field name(伪字段名称)与客户端一起使用，然后描述如何从 Job object中提取indexed value的function(`func(rawObj client.Object) []string`)。indexer会自动为我们处理namespace，因此如果 Job 有 CronJob owner，则只需要提取其owner的Name。

此外，我们需要通知manager该controller拥有一些Jobs，以便能在Job发生更改、删除等时，自动调用底层 CronJob 上的 Reconcile。

```go
var (
    jobOwnerKey = ".metadata.controller"
    apiGVStr    = batchv1.GroupVersion.String()
)

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // set up a real clock, since we're not in a test
    if r.Clock == nil {
        r.Clock = realClock{}
    }

    if err := mgr.GetFieldIndexer().IndexField(context.Background(), &kbatch.Job{}, jobOwnerKey, func(rawObj client.Object) []string {
        // grab the job object, extract the owner... 获取job object，提取其owner
        job := rawObj.(*kbatch.Job)
        owner := metav1.GetControllerOf(job)
        if owner == nil {
            return nil
        }
        // ...make sure it's a CronJob...
        if owner.APIVersion != apiGVStr || owner.Kind != "CronJob" {
            return nil
        }

        // ...and if so, return it
        return []string{owner.Name}
    }); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&batchv1.CronJob{}).
        Owns(&kbatch.Job{}).
        Complete(r)
}
```

## 再次回到`main.go`

- 要注意的第一个区别是 kubebuilder 已将新的 API 组的包 ( `batchv1`) 添加到我们的方案中。这意味着我们可以在控制器中使用这些对象。如果我们要使用任何其他 CRD，我们必须以相同的方式添加他们的方案。内置类型（例如 Job）的方案由`clientgoscheme`.

  ```
  var (
      scheme   = runtime.NewScheme()
      setupLog = ctrl.Log.WithName("setup")
  )
  
  func init() {
      utilruntime.Must(clientgoscheme.AddToScheme(scheme))
  
      utilruntime.Must(batchv1.AddToScheme(scheme))
      //+kubebuilder:scaffold:scheme
  }
  ```

  

- kubebuilder 添加了一个调用 CronJob controller的`SetupWithManager`method的block。

  ```go
      if err = (&controllers.CronJobReconciler{
          Client: mgr.GetClient(),
          Scheme: mgr.GetScheme(),
      }).SetupWithManager(mgr); err != nil {
          setupLog.Error(err, "unable to create controller", "controller", "CronJob")
          os.Exit(1)
      }
  ```

  