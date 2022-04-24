## 导读

### 什么是operator ?

先说结论：**Operator是一个感知应用状态的控制器**

让我们先来回顾一下，我们平时最常使用的k8s内建资源类型主要有Service/Pod/Deployment。通过Controller Manager监听，对创建，删除和更新做出响应

但是这些资源类型并不能满足用户和企业的复杂业务需求，因此k8s1.7推出了**Custom Resource Definition**即自定义k8s的资源类型

创建crd时，k8s会通过Apiserver，在etcd中注册一种新的资源类型，Operator通过监听etcd的watch事件感知资源对象的变化，并根据业务逻辑执行对应的操作，以保证应用处于预期的状态

### 开发环境

#### 环境信息

-  centos7 
-  kubebuilder Version2.3.1 
-  go1.15.6 linux/amd64 

#### 工具

> 由于目前kubebuilder仅提供了Linux和Mac的版本,而笔者使用的win系统,因此需要通过远程开发的模式来调试代码,开发的IDE为: vscode

安装vscode完毕,可以到在线商店选择远程开发工具进行安装,详情可通过 [传送门](https://code.visualstudio.com/docs/remote/remote-overview) 了解

vscode提供了三种远程开发的扩展包的使用步骤和流程,你可以根据自身情况选择任一开发模式,笔者使用的是远程SSH的开发模式

- [Remote - SSH](https://code.visualstudio.com/docs/remote/ssh) - 使用SSH连接远程机器/虚拟机开发
- [Remote - Containers](https://code.visualstudio.com/docs/remote/containers) - 基于容器的应用程序开发
- [Remote - WSL](https://code.visualstudio.com/docs/remote/wsl) - 基于windows linux子系统下的开发

## 实战

### 为什么选择kubebuilder进行开发?

在开始开发之前,我想说说为什么选择kubebuilder作为我们的开发脚手架,截止笔者开发时,主流的k8s二次开发脚手架主要是 **[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)**和**[operator-sdk](https://github.com/operator-framework/operator-sdk)**

二者没有核心的区别,本质上都是使用官方的controller-tools 和 controller-runtime,只不过实现的细节稍有不同,比如 kubebuilder 有着更为完善的测试与部署以及代码生成的脚手架等；而 operator-sdk 对 ansible operator 这类上层操作的支持更好一些, kubebuilder出自k8s sig社区,并且两个框架的团队已经达成一致目标,未来将进行代码整合, kubebuilder 将会合并 operator-sdk 的功能具体 [详情](https://github.com/kubernetes-sigs/kubebuilder/projects/7)

### 项目初始化

```shell
# 命名elasticweb工程
go mod init elasticweb

# 创建operator工程
kubebuilder init --domain com.bolingcavalry

# 初始化 CRD
kubebuilder create api --group webapp --version v1 --kind Guestbook
```

### 项目骨架

```shell
.
├── api
│   └── v1
│       ├── elasticweb_types.go
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd
│   │   ├── bases
│   │   │   └── elasticweb.com.bolingcavalry_elasticwebs.yaml
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_elasticwebs.yaml
│   │       └── webhook_in_elasticwebs.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── elasticweb_editor_role.yaml
│   │   ├── elasticweb_viewer_role.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── role_binding.yaml
│   │   └── role.yaml
│   ├── samples
│   │   ├── elasticweb_v1_elasticweb.yaml
│   │   └── update_single_pod_qps.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── controllers
│   ├── elasticweb_controller.go
│   └── suite_test.go
├── Dockerfile
├── .gitignore
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT
```

### 权限

kubebuilder提供的权限对于[RBAC ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole)的生成,你的Controller需要通过 **标识代码**来进行授权,如下所示

我们来对 Depolyment 和 service 进行资源操作权限的授权,在kubebuilder中想要了解更多多相关标识可[前往](https://book.kubebuilder.io/reference/markers.html)

```
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
```

### 业务逻辑设计

![](https://img-blog.csdnimg.cn/20210217173845501.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JvbGluZ19jYXZhbHJ5,size_16,color_FFFFFF,t_70#crop=0&crop=0&crop=1&crop=1&id=ViTX8&originHeight=871&originWidth=731&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

### CRD设计

```go
// api/v1/elasticweb_types.go
package v1

import (
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"strconv"
)

// 期望状态
type ElasticWebSpec struct {
	// 业务服务对应的镜像，包括名称:tag
	Image string `json:"image"`
	// service占用的宿主机端口，外部请求通过此端口访问pod的服务
	Port *int32 `json:"port"`

	// 单个pod的QPS上限
	SinglePodQPS *int32 `json:"singlePodQPS"`
	// 当前整个业务的总QPS
	TotalQPS *int32 `json:"totalQPS"`
}

// 实际状态，该数据结构中的值都是业务代码计算出来的
type ElasticWebStatus struct {
	// 当前kubernetes中实际支持的总QPS
	RealQPS *int32 `json:"realQPS"`
}

// +kubebuilder:object:root=true

// ElasticWeb is the Schema for the elasticwebs API
type ElasticWeb struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   ElasticWebSpec   `json:"spec,omitempty"`
	Status ElasticWebStatus `json:"status,omitempty"`
}

func (in *ElasticWeb) String() string {
	var realQPS string

	if nil == in.Status.RealQPS {
		realQPS = "nil"
	} else {
		realQPS = strconv.Itoa(int(*(in.Status.RealQPS)))
	}

	return fmt.Sprintf("Image [%s], Port [%d], SinglePodQPS [%d], TotalQPS [%d], RealQPS [%s]",
		in.Spec.Image,
		*(in.Spec.Port),
		*(in.Spec.SinglePodQPS),
		*(in.Spec.TotalQPS),
		realQPS)
}

// +kubebuilder:object:root=true

// ElasticWebList contains a list of ElasticWeb
type ElasticWebList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []ElasticWeb `json:"items"`
}

func init() {
	SchemeBuilder.Register(&ElasticWeb{}, &ElasticWebList{})
}
```

### 编写Controller

该文件位于`controller/elasticweb_controller.go`

#### 定义常量

```go
const (
	// deployment中的APP标签名
	APP_NAME = "elastic-app"
	// tomcat容器的端口号
	CONTAINER_PORT = 8080
	// 单个POD的CPU资源申请
	CPU_REQUEST = "100m"
	// 单个POD的CPU资源上限
	CPU_LIMIT = "100m"
	// 单个POD的内存资源申请
	MEM_REQUEST = "512Mi"
	// 单个POD的内存资源上限
	MEM_LIMIT = "512Mi"
)
```

#### 新增RBAC权限

> kubebuilder提供了[rbac标记](https://cloudnative.to/kubebuilder/reference/markers/rbac.html)为controller赋予操作资源的权限

```go
// +kubebuilder:rbac:groups=elasticweb.com.bolingcavalry,resources=elasticwebs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=elasticweb.com.bolingcavalry,resources=elasticwebs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
func (r *ElasticWebReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	// your logic here
    ...
}
```

#### 计算pod数量

业务逻辑：

- 根据单个pod的QPS和总QPS，计算需要多少个pod并返回

编码实现：

```go
func getExpectReplicas(elasticWeb *elasticwebv1.ElasticWeb) int32 {
    // 获取单个pod的QPS
    singlePodQPS := *(elasticWeb.Spec.SinglePodQPS)
    // 期望的总QPS
    totalQPS := *(elasticWeb.Spec.TotalQPS)
    // 计算所需副本数
    replicas := totalQPS / singlePodQPS
    // 安全界限
    if totalQPS%singlePodQPS > 0 {
        replicas++
    }
    return replicas
}
```

#### 创建service

业务逻辑

- 查看service是否存在，不存在才创建
- 将service与CRD实例进行关联
- 利用client-go工具创建service

编码实现：

```go
func createServiceIfNotExists(ctx context.Context, r *ElasticWebReconciler, elasticWeb *elasticwebv1.ElasticWeb, req ctx.Request) error {
    log := r.Log.WithValues("func", "createService-创建服务")
    // 判断是否存在service
    service := &corev1.Service{}
    err := r.Get(ctx, req.NamespacedName, service)
    if err == nil {
        log.Info("service exists")
        return nil
    }
    if !errors.IsNotFound(err) {
        log.Error(err, "query service error")
        return err
    }
    
    //实例化准备
    service = &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Namespace: elasticWeb.Namespace,
            Name: elasticWeb.Name,
        },
        Spec: corev1.ServiceSpec{
            Ports: []corev1.ServicePort{{
                Name: "http",
                Port: 8080,
                NodePort: *elasticWeb.Spec.Port
            }},
            Selector: map[string]string{
                "app": APP_NAME,
            },
            Type: corev1.ServiceTypeNodePort,
        },
    }
    
    // 建立资源关联
    log.Info("set reference")
    if err := controllerutil.SetControllerReference(elasticWeb, service, r.Scheme); err != nil {
        log.Error(err, "SetControllerReference error")
        return err
    }
    
    // 创建service
    log.Info("start create service")
    if err := r.Create(ctx, service); err != nil {
        log.Error(err, "create service error")
        return err
    }
    
    log.Info("create service success")
    return nil
}
```

#### 创建deployment

业务逻辑：

- 通过getExpectReplicas方法得到需要创建pod的数量
- 设置每个pod所需的CPU和内存资源，作为deployment参数
- 将deployment与elasticweb建立关联
- 通过client-go创建deployment资源

编码实现：

```go
func createDeployment(ctx context.Context, r *ElasticWebReconciler, elasticWeb *elasticwebv1.ElasticWeb) error {
    log := r.Log.WithValues("func", "createDeployment")
    
    // 计算所需pod数量
    expectReplicas := getEcpectReplicas(elasticWeb)
   	// 实例化准备 
    deployment := &appsv1.Deployment{
        ObjectMeata: metav1.ObjectMeta{
            Namespace: elasticWeb.Namespace,
            Name: elasticWeb.Name,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: pointer.Int32Ptr(expectReplicas),
            Selector: &metav1.LabelSelector{
                MathLAbels: map[string]string{
                    "app": APP_NAME
                }
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app": APP_NAME
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name: APP_NAME,
                            Image: elasticWeb.Spec.Image,
                            ImagePullPolicy: "IfNotPresent",
                            Ports: []corev1.ContainerPort{
                                {
                                    Name: "http",
                                    Protocol: corev1.ProtocolSCTP,
                                    ContainerPort: CONTAINER_PORT,
                                },
                            },
                            Resources: corev1.ResourceRequirements{
                                Requests: corev1.ResourceList{
                                    "cpu": resource.MustParse(CPU_REQUEST),
                                    "memory": resource.MustParse(MEM_REQUEST),
                                },
                                Limits: corev1.ResourceList{
                                    "cpu": resource.MustParse(CPU_LIMIT),
                                    "memory": resource.MustParse(MEM_LIMIT),
                                },
                            },
                        },
                    },
                },
            },
        },
    },
    
    log.Inof("set reference")
    if err := controllerutil.SetControllerReference(elasticWeb, deployment, r.Scheme); err != nil {
        log.Error(err, "SetControllerReference error")
        return err
    }
    
    log.Info("start create deployment")
    if err := r.Create(ctx, deployment); err != nil {
        log.Error(err, "create deployment error")
        return error
    }
    
    log.Info("create deployment success")
    return nil
}
```

#### 更新状态

deployment的资源变更，都需要动态更新elasticWeb的状态

```go
func updateStatus(ctx context.Context, r *ElasticWebReconciler, elasticWeb *elasticwebv1.ElasticWeb) error {
    log := r.Log("func", "updateStatus - 状态更新")
    singlePodQPS := *(elasticWeb.Spec.SinglePodQPS)
    replicas := getExpectReplicas(elasticWeb)
    if nil == elasticWeb.Status.RealQPS {
        elasticWeb.Status.RealQPS = new(int32)
    }
    
    *(elasticWeb.Status.RealQPS) = singlePodQPS * replicas
    
    lof.Info(fmt.Sprintf("sinflePodQPS [%d], replicas [%d], realQPS [%d]", singlePodQPS, replicas, *(elasticWeb.Status.RealQPS)))
    
    if err := r.Update(ctx, elasticWeb); err != nil {
        log.Error(err, "update instance error")
        return err
    }
    
    return nil
}
```

#### 主流程

编码实现：

```go
func (r *ElasticWebReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) { 
    ctx := context.Background()
    log := r.Log.WithValues("elasticweb", req.NamespacedName)
    
    log.Info("1. start reconcile logic")
    
    // 获取实例
    instance := &elasticwebv1.ElasticWeb{}
    // 查询实例
    err := r.Get(ctx, req.NamespacedName, instance)
    
    // 实例不存在则终止Reconcile
    if err != nil {
        if errors.IsNotFound(err) {
            log.Info("2.1 instance not found, maybe removed")
            return reconcile.Result{}, nil
        }
    }
    
    log.Info("3. instaince : "+ instance.String())
    
    // 获取deployment
    deployment := &appsv1.Deployment{}
    // 查询deployment
    err = r.Get(ctx, req.NamespacedName, deployment)
    if err != nil {
        if errors.IsNotFound(err) {
            log.Info("4. deployment not exists")
            
            if *(instance.Spec.TotalQPS) < 1 {
                log.Info("5.1 not need deployment")
                return ctrl.Result{}, nil
            }
            
            if err = createServiceIfNotExists(ctx, r, instance, req); err != nil {
                log.Error(err, "5.2 create service error")
                return ctrl.Result{}, nil
            }
            
            if err = createDeployment(ctx, r, instance, req); err != nil {
                log.Error(err, "5.3 create deployment error")
                return ctrl.Result{}, nil
            }
            
            if err := updateStatus(ctx, r, instance); err != nil {
                log.Error(err, "5.4 update status error")
                return ctrl.Result{}, nil
            }
            
            return ctrl.Result{}, nil
        }
    } else {
        log.Error(err, "6. error")
        return ctrl.Result{}, err
    }
    
    expectReplicas := getExpectReplicas(instance)
    realReplicas := *deployment.Spec.Replicas
    
    log.Info(fmt.Sprintf("7. expectReplicas [%d], realReplicas [%d]", expectReplicas, realReplicas))
    if expectReplicas == realReplicas {
		log.Info("8. return now")
		return ctrl.Result{}, nil
	}
    
    *(deployment.Spec.Replicas) = expectReplicas
    
    if err = r.Update(ctx, deployment); err != nil {
		log.Error(err, "9. update deployment replicas error")
		// 返回错误信息给外部
		return ctrl.Result{}, err
	}
    
    if err = updateStatus(ctx, r, instance); err != nil {
		log.Error(err, "14. update status error")
		// 返回错误信息给外部
		return ctrl.Result{}, err
	}
}
```

### Webhook

#### 概述

拦截Kubernetes API Server听上去是一个相当常见的需求，事实上Kubernetes 有自己实现的一个[控制器列表](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)

用户可以根据自定义的逻辑去修改或者拒绝它们，但是问题在于这些控制器需要被编译进 kube-apiserver，并且只能在 apiserver 启动时启动

这种使用方式极大限制了用户，一种最佳的方式它应该是**动态的**，而不是和API Server耦合在一起

Admisson webhook就是通过动态配置的方法解决了这个限制问题

在Kubernetes Apiserver 内部，有两个特殊的准入控制器，它们分别是**MutatingAdmissionWebhook** 和  **ValidatingAdmissionWebhook**,如果启用了这两个控制器，k8s管理员可以在集群中创建和配置一个Admisson webhook

通过它们提供的协议，用户能够将自定义 webhook 集成到 admission  controller 控制流中。顾名思义，mutating admission webhook 可以拦截并修改请求资源，validating  admission webhook 只能拦截并校验请求资源，但不能修改它们。分成两类的一个好处是，后者可以被 apiserver  并发执行，只要任一失败，即可快速结束请求

Admission webhook在k8s请求生命周期出现的时机如下图所示：

![](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/k8s-api-request-lifecycle.png)

具体工作流程：

![](https://trstringer.com/images/kubernetes-validating-webhook1.png)

#### 先决条件

1、确保集群在k8sv1.16+以上，以便使用 `admissionregistration.k8s.io/v1` API或者v1.9以上使用`admissionregistration.k8s.io/v1beta1` API，如果你对集群中的是否启用了准入注册的API，可以通过

`kubectl api-versions |grep admission`查看

2、确保启用了这两个控制器，你可以通过`kube-apiserver -h | grep enable-admission-plugins`查询已经默认启用的插件，如果没有启用，可以通过`kubectl get pods kube-apiserver-ydzs-master -n kube-system -o yaml`指令修改enable-admission的配置，如下所示：

```sh
$ kubectl get pods kube-apiserver-ydzs-master -n kube-system -o yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver-ydzs-master
  namespace: kube-system
......
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.151.30.11
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
......
```

#### 业务逻辑

接下来让我们在原有基础上添加验证的功能，新增逻辑如下：

- **用户输入总QPS小于1300，webhook检测到自动设置为1300**
- **为单个pod设置QPS上限，超过1000将拒绝创建资源**

#### 准备工作

##### 创建webhook

```sh
kubebuilder create webhook \
--group elasticweb \
--version v1 \
--kind ElasticWeb \
--defaulting \
--programmatic-validation
```

执行该步骤后将对项目工程产生以下文件变化

```
.
├── api
│   └── v1
│       ├── elasticweb_types.go
│      +├── elasticweb_webhook.go // 实现webhook的接口 
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── config
│  +├── certmanager // 用于部署的HTTPS证书管理配置
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd
│   │   ├── bases
│   │   │   └── elasticweb.com.bolingcavalry_elasticwebs.yaml
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_elasticwebs.yaml
│   │      +└── webhook_in_elasticwebs.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│  +│   ├── manager_webhook_patch.yaml
│  +│   └── webhookcainjection_patch.yaml
│   ├── dev
│   │   ├── kustomization.yaml
│   │   └── webhook_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── elasticweb_editor_role.yaml
│   │   ├── elasticweb_viewer_role.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── role_binding.yaml
│   │   └── role.yaml
│   ├── samples
│   │   ├── elasticweb_v1_elasticweb.yaml
│   │   ├── update_single_pod_qps.yaml
│   │   └── update_total_qps.yaml
│  +└── webhook // webhook部署相关配置
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       ├── manifests.yaml
│       └── service.yaml
├── controllers
│   ├── elasticweb_controller.go
│   └── suite.go
├── cover.out
├── Dockerfile
├── .gitignore
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT
```

##### 修改配置

找到`api/config/default`下，启用 webhook 和 cert-manager 相关配置,如下:

```yaml
# Adds namespace to all resources.
namespace: elasticweb-system

# Value of this field is prepended to the
# names of all resources, e.g. a deployment named
# "wordpress" becomes "alices-wordpress".
# Note that it should also match with the prefix (text before '-') of the namespace
# field above.
namePrefix: elasticweb-

# Labels to add to all resources and selectors.
#commonLabels:
#  someName: someValue

bases:
- ../crd
- ../rbac
- ../manager
# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in 
# crd/kustomization.yaml
- ../webhook
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'. 'WEBHOOK' components are required.
- ../certmanager
# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'. 
#- ../prometheus

patchesStrategicMerge:
  # Protect the /metrics endpoint by putting it behind auth.
  # If you want your controller-manager to expose the /metrics
  # endpoint w/o any authn/z, please comment the following line.
- manager_auth_proxy_patch.yaml

# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in 
# crd/kustomization.yaml
- manager_webhook_patch.yaml

# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'.
# Uncomment 'CERTMANAGER' sections in crd/kustomization.yaml to enable the CA injection in the admission webhooks.
# 'CERTMANAGER' needs to be enabled to use ca injection
- webhookcainjection_patch.yaml

# the following config is for teaching kustomize how to do var substitution
vars:
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER' prefix.
- name: CERTIFICATE_NAMESPACE # namespace of the certificate CR
  objref:
    kind: Certificate
    group: cert-manager.io
    version: v1alpha2
    name: serving-cert # this name should match the one in certificate.yaml
  fieldref:
    fieldpath: metadata.namespace
- name: CERTIFICATE_NAME
  objref:
    kind: Certificate
    group: cert-manager.io
    version: v1alpha2
    name: serving-cert # this name should match the one in certificate.yaml
- name: SERVICE_NAMESPACE # namespace of the service
  objref:
    kind: Service
    version: v1
    name: webhook-service
  fieldref:
    fieldpath: metadata.namespace
- name: SERVICE_NAME
  objref:
    kind: Service
    version: v1
    name: webhook-service
```

##### 自签发证书

在项目工程下,新创建 config/cert 目录,使用openssl来生成证书,首先创建一个 csr.conf 文件

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = CN
ST = Guangzhou
L = Shenzhen
CN = host.docker.internal

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = host.docker.internal # 支持域名访问
IP.1 = 10.204.118.218  # 支持IP访问

[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
subjectAltName=@alt_names
```

生成CA证书并且签发本地证书

```
# 生成 CA 证书
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -subj "/CN=host.docker.internal" -days 10000 -out ca.crt

# 签发本地证书
openssl genrsa -out tls.key 2048
openssl req -new -SHA256 -newkey rsa:2048 -nodes -keyout tls.key -out tls.csr -subj "/C=CN/ST=Shanghai/L=Shanghai/O=/OU=/CN=host.docker.internal"
openssl req -new -key tls.key -out tls.csr -config csr.conf
openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 10000 -extensions v3_ext -extfile csr.conf
```

获取CA证书的Base64值

```
cat ca.crt | base64 | tr -d '\n'
```

快速测试

如果你的k8s服务是部署在本机的,这里有一个快速进行测试的做法,拿到apiserver的证书,存放在 /tmp/k8s-webhook-server/serving-certs,即可直接进行开发

```
.
└── serving-certs
    ├── tls.crt
    └── tls.key
```

##### kube-apiserver 是如何知晓服务的存在?

在开发之前,我想先跟大家探讨一个问题 kube-apiserver 是如何知晓服务的存在,答案是 MutatingWebhookConfiguration 和 ValidatingWebhookConfiguration 配置,通过往k8s集群写入该协议,最终 apiserver 会在其 MutatingAdmissionWebhook controller 和 ValidatingAdmissionWebhook controller 模块注册好我们的webhook, 需要注意以下几点

- apiserver 仅支持 HTTPS webhook 因此需要准备 TLS 证书,生产时官方内置了 cert-manager 证书管理工具, kubebuilder 也默认使用该证书管理工具; 本地开发时,需要自签发证书,后续后讲到
- clientConfig.caBundle 用于指定签发 TLS 证书的 CA 证书, 本地开发,自签发证书获取CA，base64 格式化再写入 clientConfig.caBundle 即可; 如果使用 cert-manager 签发证书，cert-manager ca-injector 组件会自动帮忙注入证书

区分配置

为了区分开发和生产,我们需要对原来的配置进行修改,由于kubebuilder才有kustomize进行k8s的配置管理,因此我们可以利用 patches 来做到这一点

创建 config/dev 目录,包含我们需要修改的配置文件

```
config/dev/
├── kustomization.yaml
└── webhook_patch.yaml
```

关于 kustomization.yaml 主要工作内容如下

- 继承default目录下的配置
- 分别为 MutatingWebhookConfiguration 和 ValidatingWebhookConfiguration 这两种准入控制的webhook添加 url 字段
- 引入 webhook_patch.yaml 更新配置

```
resources:
- ../default

patches:
- patch: |
    - op: "add"
      path: "/webhooks/0/clientConfig/url"
      value: "https://host.docker.internal:9443/mutate-nodes-lailin-xyz-v1-nodepool"
  target:
    kind: MutatingWebhookConfiguration
- patch: |
    - op: "add"
      path: "/webhooks/0/clientConfig/url"
      value: "https://host.docker.internal:9443/validate-nodes-lailin-xyz-v1-nodepool"
  target:
    kind: ValidatingWebhookConfiguration
- path: webhook_patch.yaml
  target:
    group: admissionregistration.k8s.io
```

关于 kustomization.yaml 主要工作内容如下

- 移除 cert-manager.io 的 annotation ,使本地调试时不在使用它进行证书注入
- 移除掉原来的 service 并添加CA证书的值

```
- op: "remove"
  path: "/metadata/annotations/cert-manager.io~1inject-ca-from"
- op: "remove"
  path: "/webhooks/0/clientConfig/service"
- op: "add"
  path: "/webhooks/0/clientConfig/caBundle"
  value: CA 证书 base64 后的值
```

在 makefile 中添加开发配置, 在开发阶段直接试用上述修改的配置开发

``` yaml
# ${IMG} 是你打包的工程镜像
dev: manifests kustomize
	cd config/manager && $(KUSTOMIZE) edit set image controller=${IMG}
	$(KUSTOMIZE) build config/dev | kubectl apply -f -
```

最后在工程内手动指定证书目录, 主要是修改 main.go 的代码

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:             scheme,
    MetricsBindAddress: metricsAddr,
    Port:               9443,
    LeaderElection:     enableLeaderElection,
    LeaderElectionID:   "65990fce.com.bolingcavalry",
    CertDir:            "config/cert", // 手动指定证书位置用于测试
})
```

#### 开发编码

##### 启用默认功能

```go
func (r *ElasticWeb) Default() {
	elasticweblog.Info("default", "name", r.Name)
	// TODO(user): fill in your defaulting logic.
	// 如果创建的时候没有输入总QPS，就设置个默认值
	if r.Spec.TotalQPS == nil {
		r.Spec.TotalQPS = new(int32)
		*r.Spec.TotalQPS = 1300
		elasticweblog.Info("a. TotalQPS is nil, set default value now", "TotalQPS", *r.Spec.TotalQPS)
	} else {
        // 如果创建的时候没有输入总QPS，就设置个默认值
        if *r.Spec.TotalQPS < 1300 {
			elasticweblog.Info("a. TotalQPS is less than 1000, set as 1300 ", "TotalQPS", *r.Spec.TotalQPS)
			*r.Spec.TotalQPS = 1300
		}
		elasticweblog.Info("b. TotalQPS exists", "TotalQPS", *r.Spec.TotalQPS)
	}
}
```

##### 封装验证功能

```go
func (r *ElasticWeb) validateElasticWeb() error {
	var allErrs field.ErrorList

	if *r.Spec.SinglePodQPS > 1000 {
		elasticweblog.Info("c. Invalid SinglePodQPS")

		err := field.Invalid(field.NewPath("spec").Child("singlePodQPS"),
			*r.Spec.SinglePodQPS,
			"d. must be less than 1000")

		allErrs = append(allErrs, err)

		return apierrors.NewInvalid(
			schema.GroupKind{Group: "elasticweb.com.bolingcavalry", Kind: "ElasticWeb"},
			r.Name,
			allErrs)
	} else {
		elasticweblog.Info("e. SinglePodQPS is valid")
		return nil
	}
}
```

## 部署

### 部署CRD

部署CRD到k8s集群

```shell
[root@master kubebuilder-demo]# make install
/var/go-project/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl apply -f -
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/elasticwebs.elasticweb.com.bolingcavalry created
```

查询CRD是否部署成功

```shell
[root@master kubebuilder-demo]# kubectl  api-versions | grep elasticweb
elasticweb.com.bolingcavalry/v1
```

查看CRD配置

```sh
 kubectl get crd elasticwebs.elasticweb.com.bolingcavalry -o yaml
```

### 构建与部署Controller

运行Controller

```shell
[root@master kubebuilder-demo]# make run
/var/go-project/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
/var/go-project/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go run ./main.go
2022-03-28T04:06:56.640-0400    INFO    controller-runtime.metrics      metrics server is starting to listen     {"addr": ":8080"}
2022-03-28T04:06:56.645-0400    INFO    setup   starting manager
2022-03-28T04:06:56.646-0400    INFO    controller-runtime.manager      starting metrics server {"path": "/metrics"}
2022-03-28T04:06:56.646-0400    INFO    controller-runtime.controller   Starting EventSource    {"controller": "elasticweb", "source": "kind source: /, Kind="}
2022-03-28T04:06:56.758-0400    INFO    controller-runtime.controller   Starting Controller     {"controller": "elasticweb"}
2022-03-28T04:06:56.758-0400    INFO    controller-runtime.controller   Starting workers        {"controller": "elasticweb", "worker count": 1}
```

构建docker镜像

前面简单的演示了下controller的运行效果，但在生产环境中，我们需要将controller以容器的方式部署到k8s集群内，因此我们先来构建我们的工程镜像

```bash
make docker-build docker-push IMG=[username]/elasticweb:001
```

让我为大家解释以下几个参数的含义

- docker-build - 执行镜像构建
- docker-push -对镜像进行推送
- IMG - 镜像的推送地址，默认是docker官网
- [username] - 你在docker官网注册的个人账户

部署Controller

```sh
make deploy IMG=[username]/elasticweb:001
```

打印日志：

```sh
[root@master kubebuilder-demo]# make deploy IMG=dockerdev0751/elasticweb:005
/var/go-project/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
cd config/manager && kustomize edit set image controller=dockerdev0751/elasticweb:005
kustomize build config/default | kubectl apply -f -
namespace/kubebuilder-demo-system created
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/elasticwebs.elasticweb.com.bolingcavalry configured
role.rbac.authorization.k8s.io/kubebuilder-demo-leader-election-role created
clusterrole.rbac.authorization.k8s.io/kubebuilder-demo-manager-role configured
clusterrole.rbac.authorization.k8s.io/kubebuilder-demo-proxy-role unchanged
Warning: rbac.authorization.k8s.io/v1beta1 ClusterRole is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 ClusterRole
clusterrole.rbac.authorization.k8s.io/kubebuilder-demo-metrics-reader unchanged
rolebinding.rbac.authorization.k8s.io/kubebuilder-demo-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/kubebuilder-demo-manager-rolebinding unchanged
clusterrolebinding.rbac.authorization.k8s.io/kubebuilder-demo-proxy-rolebinding unchanged
service/kubebuilder-demo-controller-manager-metrics-service created
service/kubebuilder-demo-webhook-service created
deployment.apps/kubebuilder-demo-controller-manager created
certificate.cert-manager.io/kubebuilder-demo-serving-cert created
issuer.cert-manager.io/kubebuilder-demo-selfsigned-issuer created
Warning: admissionregistration.k8s.io/v1beta1 MutatingWebhookConfiguration is deprecated in v1.16+, unavailable in v1.22+; use admissionregistration.k8s.io/v1 MutatingWebhookConfiguration
mutatingwebhookconfiguration.admissionregistration.k8s.io/kubebuilder-demo-mutating-webhook-configuration created
Warning: admissionregistration.k8s.io/v1beta1 ValidatingWebhookConfiguration is deprecated in v1.16+, unavailable in v1.22+; use admissionregistration.k8s.io/v1 ValidatingWebhookConfiguration
validatingwebhookconfiguration.admissionregistration.k8s.io/kubebuilder-demo-validating-webhook-configuration created
```

进入控制台查看打印日志

```sh
kubectl logs -f  kubebuilder-demo-controller-manager-5dddfccc88-hlgl4  -c manager  -n kubebuilder-demo-system
```

## 测试

#### 新建elasticweb资源对象

```yaml
apiVersion: elasticweb.com.bolingcavalry/v1
kind: ElasticWeb
metadata:
  namespace: dev
  name: elasticweb-sample
spec:
  # Add fields here
  image: tomcat:8.0.18-jre8
  port: 30003
  singlePodQPS: 500
  totalQPS: 600
```

#### 创建elasticweb实例

```shell
[root@master kubebuilder-demo]# kubectl apply -f config/samples/elasticweb_v1_elasticweb.yaml
elasticweb.elasticweb.com.bolingcavalry/elasticweb-sample created
```

#### 查看相关状态

```shell
kubectl get elasticweb -n dev
kubectl get service -n dev
kubectl get deployment -n dev
kubectl get pod -n dev
```

#### 修改单个pod的QPS

```shell
kubectl patch elasticweb elasticweb-sample \
-n dev \
--type merge \
--patch "$(cat config/samples/update_single_pod_qps.yaml)"
```

#### 修改总QPS

```shell
kubectl patch elasticweb elasticweb-sample \
-n dev \
--type merge \
--patch "$(cat config/samples/update_total_qps.yaml)"
```

#### 删除资源

```shell
# 清除实例资源
kubectl delete -f config/samples/elasticweb_v1_elasticweb.yaml
# 清除controller
kustomize build config/default | kubectl delete -f -
# 清除crd
make uninstall
```

进入控制台

```sh
kubectl logs -f  kubebuilder-demo-controller-manager-5dddfccc88-hlgl4  -c manager  -n kubebuilder-demo-system
```

webhook验证

```sh
kubectl patch elasticweb elasticweb-sample \
-n dev \
--type merge \
--patch "$(cat config/samples/update_single_pod_qps.yaml)"
```

#### 查看webhook相关的服务

### 查看服务

当本地服务启动时,我们可以来看看k8s集群内发生了什么变化

```
[root@node8 ~]# kubectl get svc -n elasticweb-system
NAME                                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
elasticweb-controller-manager-metrics-service   ClusterIP   10.43.52.92     <none>        8443/TCP        24h
elasticweb-webhook-service                      NodePort    10.43.211.145   <none>        443:30007/TCP   24h
```

可以看到这里分别运行了 controller 和 webhook 服务,但是本次的主角并不是他们,在生产环境中这两个服务将会接受 kube-apiserver 的请求调用,但是由于我们是本地开发,服务是跑在本地的,如果你没有修改启动端口的配置的话,默认在你的本地 8080 端口和 9443 端口会分别启动对应的服务

进阶着,我们来查看准入控制器的相关配置,这里 mutatingwebhookconfigurations 为例

```
 kubectl describe mutatingwebhookconfigurations elasticweb-mutating-webhook-configuration
Name:         elasticweb-mutating-webhook-configuration
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  admissionregistration.k8s.io/v1
Kind:         MutatingWebhookConfiguration
Metadata:
  Creation Timestamp:  2022-04-20T08:07:57Z
  Generation:          6
  Managed Fields:
    API Version:  admissionregistration.k8s.io/v1beta1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:webhooks:
        .:
        k:{"name":"melasticweb.kb.io"}:
          .:
          f:admissionReviewVersions:
          f:clientConfig:
            .:
            f:caBundle:
            f:url:
          f:failurePolicy:
          f:matchPolicy:
          f:name:
          f:namespaceSelector:
          f:objectSelector:
          f:reinvocationPolicy:
          f:rules:
          f:sideEffects:
          f:timeoutSeconds:
    Manager:         kubectl-client-side-apply
    Operation:       Update
    Time:            2022-04-20T08:07:57Z
  Resource Version:  26588753
  UID:               9646e033-8595-4b95-bde0-35a53195bf29
Webhooks:
  Admission Review Versions:
    v1beta1
  Client Config:
    Ca Bundle:     LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURFVENDQWZtZ0F3SUJBZ0lKQUxEeEt1L2lnRVBFTUEwR0NTcUdTSWIzRFFFQkN3VUFNQjh4SFRBYkJnTlYKQkFNTUZHaHZjM1F1Wkc5amEyVnlMbWx1ZEdWeWJtRnNNQjRYRFRJeU1EUXlNVEEyTXpFME9Wb1hEVFE1TURrdwpOakEyTXpFME9Wb3dIekVkTUJzR0ExVUVBd3dVYUc5emRDNWtiMk5yWlhJdWFXNTBaWEp1WVd3d2dnRWlNQTBHCkNTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCQVFDcWRtZGdxUFltN1pRQ0pIbEREZEVERUN4Z3FyYmkKTmlPU2hSMXZGVXcvUldMd3grTXoya0pNN2dXcGtHZnlJTkM0dFpBTytpOWJaL3hmUVBDVkJXazRKV3ZjR3BHVworckMrUFpEcHYrU0tYcXJDNGk0VENIUVl5QVhzNDhkUjhSOUJjcFpCaHJYeXA5WFJtYlVHRkY3V2RPUmRuaGtjCmJVYVU5WXhFY0VzYW0xbzZzN3VsckJydWd3bjBTNFVvSzM4YkZyK2hidWRNVmdFVkNMN0hZWHJGWE0zZmpkY1oKQzR6TVMzNGZDR3ppY0c0M3p6Yk5neVRhN0VtdXl0SURJUndQYWxXbFlMdFdYdWl1eFM0RFBLdVZObEp4cXBGQQpxRVZwd1J2em9RUkpTQlI5WTM4ektLdmlHOTBDd2VrdEFDVWs5QXFuelJNdStUYUhRREdNWnVmWEFnTUJBQUdqClVEQk9NQjBHQTFVZERnUVdCQlMzNkZmSm8yRmU3bEduRjlqd2wzcFRaSVN1SERBZkJnTlZIU01FR0RBV2dCUzMKNkZmSm8yRmU3bEduRjlqd2wzcFRaSVN1SERBTUJnTlZIUk1FQlRBREFRSC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQQpBNElCQVFDbjZBazdINWRKZG5MaVBKM3JLbnptWWZRVkNBQW9hQzcxdTJCaVArQ2Yrc1hHTXZ5eEpOZUNKZ1JTCkpyQkhpbjV4QXllMVRNU2dUK0VLdmZQSkxVV0xzQWZDZUozdXVCc2FRU0FLYU5SYnZXQ1oyYVFoY2lJcHZkRW4KQjJMUjVHd0NreDBTcWxVaFV0ZytJRHdHL0hyWjBZVTBwcUVLYzVQakg0UThGdlUwMkJTN1p1MmZOVXR4VUN5NQpyNDgyRFAzTDl5VytLTWxRNkNhRmt3SFcxK1lSRjdSa29pSHZVQ2RVS0ZMeXhhUnZJM1ZnOTBpbGpEbDU4bGkxCmUrcG9NcWlCYzRUMTBQSHlHbXVSYnFDUHUybGdwZHNKOTFYVW03Vks3V0x2QUZFeUlhWWlxcXBBREhDOHljTlkKOEVwMmFtMUNQb0JpQWY0bHJDSGEyTUFtUm9NcAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    URL:           https://10.204.118.218:9443/mutate-elasticweb-com-bolingcavalry-v1-elasticweb
  Failure Policy:  Fail
  Match Policy:    Exact
  Name:            melasticweb.kb.io
  Namespace Selector:
  Object Selector:
  Reinvocation Policy:  Never
  Rules:
    API Groups:
      elasticweb.com.bolingcavalry
    API Versions:
      v1
    Operations:
      CREATE
      UPDATE
    Resources:
      elasticwebs
    Scope:          *
  Side Effects:     Unknown
  Timeout Seconds:  30
Events:             <none>
```

## 其他

1. 开发环境的配置以及工程项目相关配置是非常折磨人的。在国内的网络环境下，使用梯子是基本条件
2. docker请设置多个国内加速镜像源，并设置好网络代理，说多都是泪
3. 注意kubebuilder的版本与当前k8s集群兼容。笔者使用的2.3.1版本似乎不能在k8s1.23+使用，webhook会出现部署出错

## 参考资料：

> 以下是我在开发过程中参考的资料,大家也可自行翻阅,笔者在开发过程中,发现kubebuilder又或者说k8s二次开发的中文资料非常匮乏,甚至外文资料也较少,大多数资料内容重复雷同又或是浅尝辄止

k8s官网admissionhook介绍 - https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/

kubebuilder官网 - https://book.kubebuilder.io/

kubebuilder中文官网翻译 - https://xuejipeng.github.io/kubebuilder-doc-cn/cronjob-tutorial/webhook-implementation.html

kubebuilder实战 - https://xinchen.blog.csdn.net/article/details/113922328

深入理解 Kubernetes Admission Webhook- https://www.qikqiak.com/post/k8s-admission-webhook/

阳明博客Kubernetes admission webhook server 开发教程 - https://www.zeng.dev/post/2021-denyenv-validating-admission-webhook/

webhook踩坑 - https://blog.csdn.net/zhangzhen02/article/details/114672609

kubebuilder2.0学习笔记 - https://segmentfault.com/a/1190000020338350

Create a Basic Kubernetes Validating Webhook - https://trstringer.com/kubernetes-validating-webhook/

CRD资源语法校验 - https://book.kubebuilder.io/reference/markers/crd-validation.html

最佳实践 - https://betterprogramming.pub/writing-custom-kubernetes-controller-and-webhooks-141230820e9

kubebuilder调试

- https://blog.csdn.net/ywq935/article/details/106311667
- https://blog.hdls.me/15708754600835.html
- https://lailin.xyz/post/operator-08-kubebuilder-webhook.html

原理解析 - https://blog.hdls.me/15564491070483.html

