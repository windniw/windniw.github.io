---
title: Kubernetes Admission Controllers
date: 2019-07-15 10:00:15
tags:
- kubernetes
---

**Kubernetes Admission Controllers** 提供了一个资源创建，修改流程的切面，可以通过`kube-apiserver`启动参数进行配置。

<!--more-->

<p class="note note-primary">以下分析仅基于 Kubernetes:1.13 </p>

从来源而言，可分为静态钩子和动态钩子，前者指的是`kubernetes`提供的原生能力，代码直接编译在`kube-apiserver`二进制中，后者即两个`webhook`。从功能而言，可分为`validating`和`mutating`，前者只做校验，后者会修改或补充资源中的信息。

# 配置

**Admission Controller** 的开关分为以下三部分

## default

无需任何配置，默认打开

```bash
kube-apiserver -h | grep enable-admission-plugin
>> admission plugins that should be enabled in addition to default enabled ones (NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority, DefaultTolerationSeconds, DefaultStorageClass, PersistentVolumeClaimResize, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ResourceQuota). Comma-delimited list of admission plugins: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, DefaultStorageClass, DefaultTolerationSeconds, DenyEscalatingExec, DenyExecOnPrivileged, EnableIpv6, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, IngressAnnotations, Initializers, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeRestriction, OwnerReferencesPermissionEnforcement, PersistentVolumeAnnotations, PersistentVolumeClaimResize, PersistentVolumeLabel, PodNodeSelector, PodPreset, PodSecurityPolicy, PodTolerationRestriction, Priority, ResourceQuota, SecurityContextDeny, ServiceAccount, ServiceAnnotations, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionWebhook. The order of plugins in this flag does not matter.
```

## enable

在 **Admission Controller** 列表中但不在default里，手动开启

```bash
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger
```

## disable

在 **Admission Controller** 列表中，但不需要，手动关闭

```
kube-apiserver --disable-admission-plugins=NamespaceLifecycle,LimitRanger
```

以上两个配置，控制器的先后顺序没有区别

实际上的先后顺序是在编译时决定的，可参见以下文件内容

```go
// k8s.io/kubernetes/pkg/kubeapiserver/options/plugins.go

// AllOrderedPlugins is the list of all the plugins in order.
var AllOrderedPlugins = []string{
	admit.PluginName,                        // AlwaysAdmit
	autoprovision.PluginName,                // NamespaceAutoProvision
	lifecycle.PluginName,                    // NamespaceLifecycle
	exists.PluginName,                       // NamespaceExists
	scdeny.PluginName,                       // SecurityContextDeny
	antiaffinity.PluginName,                 // LimitPodHardAntiAffinityTopology
	podpreset.PluginName,                    // PodPreset
	limitranger.PluginName,                  // LimitRanger
	serviceaccount.PluginName,               // ServiceAccount
	noderestriction.PluginName,              // NodeRestriction
	nodetaint.PluginName,                    // TaintNodesByCondition
	alwayspullimages.PluginName,             // AlwaysPullImages
	enableipv6.PluginName,                   // EnableIpv6
	imagepolicy.PluginName,                  // ImagePolicyWebhook
	podsecuritypolicy.PluginName,            // PodSecurityPolicy
	podnodeselector.PluginName,              // PodNodeSelector
	podpriority.PluginName,                  // Priority
	defaulttolerationseconds.PluginName,     // DefaultTolerationSeconds
	podtolerationrestriction.PluginName,     // PodTolerationRestriction
	exec.DenyEscalatingExec,                 // DenyEscalatingExec
	exec.DenyExecOnPrivileged,               // DenyExecOnPrivileged
	eventratelimit.PluginName,               // EventRateLimit
	extendedresourcetoleration.PluginName,   // ExtendedResourceToleration
	label.PluginName,                        // PersistentVolumeLabel
	setdefault.PluginName,                   // DefaultStorageClass
	storageobjectinuseprotection.PluginName, // StorageObjectInUseProtection
	gc.PluginName,                           // OwnerReferencesPermissionEnforcement
	resize.PluginName,                       // PersistentVolumeClaimResize
	mutatingwebhook.PluginName,              // MutatingAdmissionWebhook
	initialization.PluginName,               // Initializers
	validatingwebhook.PluginName,            // ValidatingAdmissionWebhook
	resourcequota.PluginName,                // ResourceQuota
	deny.PluginName,                         // AlwaysDeny
	annotation.PluginName,                   // PersistentVolumeAnnotations
	serviceannotation.PluginName,            // ServiceAnnotations
	ingressannotation.PluginName,            // IngressAnnotations
}
```

# 支持列表

## 废弃

**AlwaysAdmit** [deprecated]

不检查，直接放行，等于没配。



**AlwaysDeny** [deprecated]

直接拒绝，没有使用场景。



**DenyExecOnPrivileged** [deprecated]

禁止了对特权容器的 `exec` 操作，功能雷同，建议使用 **DenyEscalatingExec** 。



**DenyEscalatingExec** [deprecated]

禁止了对特权容器的`exec`和`attach`操作，否则可以借此绕过操作系统验证，获取节点上的极大操作权限。建议改用 **PodSecurityPolicy** 做更精准的限制。



**PersistentVolumeLabel** [deprecated]

自动给`pv`和`pod`打`label`，防止其创建到不同`region`上去。

怎么看都觉得这种是各个云服务提供商应该操心的事，不应该在主线分支里提供这种东西，被废弃也理所当然吧。



## 可靠性

**TaintNodesByCondition**

依据`node.condition`给`node`自动打`taint`。



**DefaultTolerationSeconds**

为所有`pod`添加以下`toleration`，如果已有相同`key`值不会覆盖

```yaml
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
```

防止因为各种原因（网络/存储等），`kubelet`在`ready/notReady`间跳变，会瞬间将上面的`pod`全部驱逐。



**ExtendedResourceToleration**

为拥有扩展资源的节点(如`nvidia.com/GPU`)打上`taint`，并给申请该扩展资源的`pod`打`toleration`，防止拥有扩展资源的节点被其他普通请求的`pod`占满。



**PodTolerationRestriction**

检查`pod.toleration`和`namespace`是否有冲突，并合并命名空间的默认配置到`pod`上，命名空间的`toleration`的通过两个`annotation`(`scheduler.alpha.kubernetes.io/defaultTolerations` 和 `scheduler.alpha.kubernetes.io/tolerationsWhitelist` ) 进行标记。没有想到啥使用场景，可能用于对某些`namespace`自动打`toleration`？



**NamespaceAutoProvision**

自动创建资源中要求的但不存在的`namespace`。



**NamespaceExists**

强制校验资源中要求的`namespace`存在。



**NamespaceLifecycle**

保证删除`namespace`时不允许继续往其下面创建资源；不允许资源指定不存在的`namespace`；保证三个默认命名空间`default / kube-system / kub-plublic`存在，即禁止删除它们。



**EventRateLimit** [alpha]

```bash
kube-apiserver --enable-admission-plugins=EventRateLimit --admission-control-config-file=${ADMISSION_CONTROL_CONFIG_FILE}
```

简单而言，通过以上配置来控制事件的速率，防止由于事件的请求过大压垮整个集群。当前允许的控制级别有

- 集群
- 单个`namespace`
- 单个用户
- 特定来源

详见 [Admission control plugin: EventRateLimit](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/admission_control_event_rate_limit.md)



## 调度

**LimitPodHardAntiAffinityTopology**

对于所有`pod`的`requiredDuringSchedulingRequiredDuringExecution`策略，只允许`kubernetes.io/hostname`此`key`值，不允许反亲和其他标签。没啥想法，可能利于防止反亲和的策略过复杂对调度效率产生影响。



**NodeRestriction**

在`--authorization-mode=Node`之上，继续缩小`system:node`的权限。主要为

- 只允许修改此`system:node:<node-name>`管理的`node`及`pod`（但不限制读）
- `update`时禁止增加或修改`node`的`taint`
- 除以下系统定义的`label`域外，禁止修改或增加这三个头域的标签 `.*node-restriction.kubernetes.io/`，`.*kubernetes.io/`，`.*k8s.io/`
    - 值得注意的是，在`create`时，这个行为只会被`klog.Warning`后放行；但在`update`时，这个行为会被拒绝

```go
// white list label
// plugin/pkg/admission/noderestriction/admission.go#L445-L464
kubernetes.io/hostname
failure-domain.beta.kubernetes.io/zone
failure-domain.beta.kubernetes.io/region
beta.kubernetes.io/instance-type
beta.kubernetes.io/os
beta.kubernetes.io/arch

failure-domain.kubernetes.io/zone
failure-domain.kubernetes.io/region
kubernetes.io/instance-type
kubernetes.io/os
kubernetes.io/arch

kubelet.kubernetes.io/.*
node.kubernetes.io/.*
```



详见 [Using Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/)



**PodNodeSelector**

自动为`pod`补`nodeSelector`，拒绝不合法的`nodeSelector`。用于多租场景下，限制某个`namespace`下的资源只允许在某些`node`上，即节点的独立性，比如生产环境和测试环境隔离，或者价值高的节点限制普通用户的`pod`瞎调。

可参考 [Understanding and using the Kubernetes PodNodeSelector Admission Controller](https://www.mgasch.com/post/podnodesel/)



**Priority**

检查`pod`创建时指定的`priorityClassName`存在。



## 存储

**DefaultStorageClass**

为`pvc`配置默认的`storageclass`类型。



**StorageObjectInUseProtection**

给`pv / pvc`增加`annotation`，在其删除时增加`finalize`。



**PersistentVolumeClaimResize**

需要打开这个`--feature-gates=ExpandPersistentVolumes=true`，若`PersistentVolumeClaim`的`StorageClass`不允许扩容(`allowVolumeExpansion: fasle`)，阻止相关操作。



## 多租

**AlwaysPullImages**

强行更新所有`pod.container`的`imagePullPolicy`为`Always`。适用于多租场景，防止租户通过镜像名使用节点上已有的其他租户镜像。



**ImagePolicyWebhook**

外注册类型的钩子，需要一个外部的`webhook`，任何`annotation`为`*.image-policy.k8s.io/*`的`pod`都会确认一遍这个镜像是否允许被使用。没有联想到什么使用场景。



## 功能

**PodPreset** [alpha]

与`PodPreset`配套使用，按照预置信息填充`pod`。`PodPreset`是`kubernetes`的一种资源，可视为一个模板，通过`label`与`pod`相关联，用于自动补全`pod`的相关信息。

详见 [Inject Information into Pods Using a PodPreset](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)



## 资源限制

**LimitRanger**

与`LimitRange`配套使用，检查是否满足其资源限制。`LimitRange`是`kubernetes`中资源的一种，用于限制单个`Pod/Container/PersistentVolumeClaim`的资源请求，可用于以下场景

- 配置默认的`limit`或`request`
- 限制资源`limit`最大值
- 限制资源`limit`最小值
- 限制资源最大的 `limit/request` 比率



**ResourceQuota**

与`resourceQuota`配套使用，检查要求资源未超过限制。

详见 [Admission control plugin: ResourceQuota](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/admission_control_resource_quota.md)



## 权限限制

**OwnerReferencesPermissionEnforcement**

修改`metadata.ownerReferences`需要`delete`权限。



**PodSecurityPolicy**

与`PodSecurityPolicies`配套使用，对`pod`策略进行校验。`PodSecurityPolicies`是`kubernetes`的一种资源，用于限制容器中的特殊权限，必须关联到`Role/ClusterRole`上，并绑定到对应用户才能发挥功能。

```yaml
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs:     ['use']
```

详见 [Pod Security Policies](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)



**SecurityContextDeny**

拒绝任何`pod`的提权操作（指获取更高的权限），社区的建议是如果集群没有通过`PodSecurityPolicy`进行权限管理则应该开启。



**ServiceAccount**

根据`ServiceAccountName`补全相关信息



## 动态

**MutatingAdmissionWebhook**

串行调用，允许修改内容

**ValidatingAdmissionWebhook**

并行调用，不允许修改其内容

# 更多

[kubernetes-Using Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)