---
title: Operator开发
author: falser101
date: 2023-10-07
category:
  - k8s
tag:
  - linux
---

> 基于Operatorframework sdk进行Operator的开发

# 使用OperatorSdk开发

## 环境要求
- [安装Operator SDK CLI](https://sdk.operatorframework.io/docs/installation/)
- git
- go，环境变量（export GO111MODULE=on）
- docker
- kubectl

## 开始
### 创建并初始化项目
```
mkdir memcached-operator
cd memcached-operator
operator-sdk init --domain example.com --repo github.com/example/memcached-operator
```

### 创建新的 API 和控制器
创建一个新的自定义资源定义 （CRD） API，其中包含组 cache 版本 v1alpha1 和内存缓存类型。出现提示时，输入yes以创建资源和控制器。
```
$ operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
Writing scaffold for you to edit...
api/v1alpha1/memcached_types.go
controllers/memcached_controller.go
...
```

### 定义API
```golang
// 添加 +kubebuilder:subresource:status 标记以将状态子资源添加到 CRD 清单，以便控制器可以在不更改 CR 对象的其余部分的情况下更新 CR 状态
//+kubebuilder:subresource:status
type MemcachedSpec struct {

  // 校验 可查阅https://book.kubebuilder.io/reference/markers/crd-validation.html
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=5
	// +kubebuilder:validation:ExclusiveMaximum=false

	// 实例数
	// +operator-sdk:csv:customresourcedefinitions:type=spec
	Size int32 `json:"size,omitempty"`

	// 容器的端口
	// +operator-sdk:csv:customresourcedefinitions:type=spec
	ContainerPort int32 `json:"containerPort,omitempty"`
}

// Memcached的状态
type MemcachedStatus struct {
  // 存储Memcached实例的状态条件
	// +operator-sdk:csv:customresourcedefinitions:type=status
	Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,1,rep,name=conditions"`
}
```

定义完成后，请运行以下命令以更新为该资源类型生成的代码

```
make generate
```
上面的 makefile 目标将调用`controller-gen`来更新 `api/v1alpha1/zz_generated.deepcopy.go` 文件，以确保我们的 API 的 Go 类型定义实现所有 Kind 类型必须实现的 runtime.Object 接口。

### 生成 CRD 清单
使用规范/状态字段和 CRD 验证标记定义 API 后，可以使用以下命令生成和更新 CRD 清单
```
make manifests
```
此makefile目标将调用`controller-gen`以生成`config/crd/bases/cache.example.com_memcacheds.yaml` CRD 清单。

## 实现Controller
### 协调循环
协调功能负责对系统的实际状态强制实施所需的 CR 状态。每当在监视的 CR 或资源上发生事件时，它都会运行，并且会根据这些状态是否匹配返回一些值

每个控制器都有一个协调器对象，其中包含一个 `Reconcile()` 实现协调循环的方法。协调循环传递参数， `Request` 该参数是用于从缓存中查找主要资源对象 `Memcached` 的命名空间/名称键

```
import (
	ctrl "sigs.k8s.io/controller-runtime"

	cachev1alpha1 "github.com/example/memcached-operator/api/v1alpha1"
	...
)

func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  // Lookup the Memcached instance for this reconcile request
  memcached := &cachev1alpha1.Memcached{}
  err := r.Get(ctx, req.NamespacedName, memcached)
  ...
}
```

以下是协调程序的一些可能的返回选项：

- With the error:
```
return ctrl.Result{}, err
```

- Without an error:
```
return ctrl.Result{Requeue: true}, nil
```

- Therefore, to stop the Reconcile, use:
```
return ctrl.Result{}, nil
```

- Reconcile again after X time:
```
return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
```

### 指定权限并生成 RBAC 清单
控制器需要某些 RBAC 权限才能与其管理的资源进行交互。这些标记通过如下所示的 RBAC 标记指定：
```
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=cache.example.com,resources=memcacheds/finalizers,verbs=update
//+kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch

func (r *MemcachedReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
  ...
}
```

ClusterRole 清单 at config/rbac/role.yaml 是使用以下命令通过控制器生成从上述标记生成的：
```
make manifests
```

### 构建镜像
可修改Makefil 的镜像名称
```
# IMG ?= controller:latest
IMG ?= $(IMAGE_TAG_BASE):$(VERSION)
```

修改后执行以下命令
```
make docker-build docker-push
```

## 运行operator
### 本地在集群外部执行
```
make install run
```

### 在群集内作为Deployment运行
```
make deploy
```

验证 memcached-operator是否已启动并正在运行
```
$ kubectl get deployment -n memcached-operator-system
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
memcached-operator-controller-manager   1/1     1            1           8m
```

## 创建Memcached CR资源
更新 config/samples/cache_v1alpha1_memcached.yaml，并定义 spec 如下：
```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  name: memcached-sample
spec:
  size: 3
  containerPort: 11211
```

创建CR:
```shell
kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
```

## 清除
```shell
kubectl delete -f config/samples/cache_v1alpha1_memcached.yaml
make undeploy
```