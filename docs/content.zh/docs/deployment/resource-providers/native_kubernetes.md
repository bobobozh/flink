---
title: Native Kubernetes
weight: 3
type: docs
aliases:
  - /zh/deployment/resource-providers/native_kubernetes.html
  - /zh/ops/deployment/native_kubernetes.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# 原生 Kubernetes

本页介绍如何在[Kubernetes](https://kubernetes.io)上原生部署Flink。

## 开始使用

本入门指南部分将引导你完成在 Kubernetes 上搭建一个完整功能的 Flink 集群的全过程。

### 介绍

Kubernetes是一个流行的容器编排系统，可以为应用程序提供自动化的部署、缩放和管理。
Flink 的原生 Kubernetes 集成可以让你直接在运行的 Kubernetes 集群上部署 Flink。
此外，由于 Flink 可以直接与 Kubernetes 通信，因此它能够根据所需资源动态地分配和取消分配 TaskManagers。

### 准备

"开始" 部分假设你已经运行了一个满足以下要求的 Kubernetes 集群：

- Kubernetes >= 1.9。
- KubeConfig，能够列出、创建、删除 pods 和 services ，可通过 `~/.kube/config` 设置。你可以通过运行 `kubectl auth can-i <list|create|edit|delete> pods` 检查权限。
- 已启用 Kubernetes DNS。
- 具有 RBAC（#rbac）权限能够创建和删除 pods 的默认服务账户。

如果你在设置 Kubernetes 集群时遇到问题，请参考 [如何设置 Kubernetes 集群](https://kubernetes.io/docs/setup/)。

### 在Kubernetes上启动Flink会话

当你的Kubernetes集群运行起来，并且`kubectl`已配置指向它，你就可以通过[会话模式]({{< ref "docs/deployment/overview" >}}#session-mode)启动一个Flink集群。

```bash
# (1) 启动 Kubernetes 会话
$ ./bin/kubernetes-session.sh -Dkubernetes.cluster-id=my-first-flink-cluster

# (2) 提交示例 job
$ ./bin/flink run \
    --target kubernetes-session \
    -Dkubernetes.cluster-id=my-first-flink-cluster \
    ./examples/streaming/TopSpeedWindowing.jar

# (3) 通过删除集群部署来停止 Kubernetes 会话
$ kubectl delete deployment/my-first-flink-cluster

```

{{< hint info >}}
默认情况下，Flink 的 Web UI 和 REST 端点以 `ClusterIP` 服务的形式暴露。要访问该服务，请参阅[访问 Flink 的 Web UI](#accessing-flinks-web-ui) 获取说明。
{{< /hint >}}

恭喜！你已成功在 Kubernetes 上部署 Flink 并运行了一个 Flink 应用程序。

{{< top >}}

## 部署模式

对于生产使用，我们推荐在[应用模式]({{< ref "docs/deployment/overview" >}}#application-mode)下部署Flink应用程序，因为这种模式为应用程序提供了更好的隔离。

### 应用程序模式

[应用模式]({{< ref "docs/deployment/overview" >}}#application-mode)要求用户代码与Flink镜像一起打包，因为它会在集群上运行用户代码的`main()`方法。
应用模式确保在应用程序终止后清理所有Flink组件。
可以通过修改基础Flink Docker镜像或通过用户 Artifact 管理进行打包，这使得可以上传和下载本地不可用的制品。


#### 修改 Docker 镜像

Flink 社区提供了一个 [基础 Docker 镜像]({{< ref "docs/deployment/resource-providers/standalone/docker" >}}#docker-hub-flink-images)，可以用来打包用户代码：

```dockerfile
FROM flink
RUN mkdir -p $FLINK_HOME/usrlib
COPY /path/of/my-flink-job.jar $FLINK_HOME/usrlib/my-flink-job.jar
```

在创建并发布名为`custom-image-name`的Docker镜像后，你可以使用以下命令启动一个应用集群：

```bash
$ ./bin/flink run-application \
    --target kubernetes-application \
    -Dkubernetes.cluster-id=my-first-application-cluster \
    -Dkubernetes.container.image.ref=custom-image-name \
    local:///opt/flink/usrlib/my-flink-job.jar
```

#### 配置用户 Artifact 管理

如果你有一个本地可用的Flink作业JAR包，可以使用制品上传功能，以便在部署时让Flink将本地制品上传到DFS，并在部署的JobManager pod上获取它：

```bash
$ ./bin/flink run-application \
    --target kubernetes-application \
    -Dkubernetes.cluster-id=my-first-application-cluster \
    -Dkubernetes.container.image=custom-image-name \
    -Dkubernetes.artifacts.local-upload-enabled=true \
    -Dkubernetes.artifacts.local-upload-target=s3://my-bucket/ \
    local:///tmp/my-flink-job.jar
```

`kubernetes.artifacts.local-upload-enabled` 选项启用此功能，而 `kubernetes.artifacts.local-upload-target` 需要指向一个有效且权限设置正确的远程目标。

可以通过 `user.artifacts.artifact-list` 配置选项添加额外的制品，该选项可以包含本地和远程制品的组合：

```bash
$ ./bin/flink run-application \
    --target kubernetes-application \
    -Dkubernetes.cluster-id=my-first-application-cluster \
    -Dkubernetes.container.image=custom-image-name \
    -Dkubernetes.artifacts.local-upload-enabled=true \
    -Dkubernetes.artifacts.local-upload-target=s3://my-bucket/ \
    -Duser.artifacts.artifact-list=local:///tmp/my-flink-udf1.jar\;s3://my-bucket/my-flink-udf2.jar \
    local:///tmp/my-flink-job.jar
```

如果作业 JAR 包或任何其他附加制品已经通过DFS或HTTP(S)远程可用，Flink将简单地在部署的JobManager pod上获取它。

```bash
# FileSystem
$ ./bin/flink run-application \
    --target kubernetes-application \
    -Dkubernetes.cluster-id=my-first-application-cluster \
    -Dkubernetes.container.image=custom-image-name \
    s3://my-bucket/my-flink-job.jar

# HTTP(S)
$ ./bin/flink run-application \
    --target kubernetes-application \
    -Dkubernetes.cluster-id=my-first-application-cluster \
    -Dkubernetes.container.image=custom-image-name \
    https://ip:port/my-flink-job.jar
```

{{< hint warning >}}
请注意，本地上传时，已存在的制品将不会被覆盖！
{{< /hint >}}

{{< hint info >}}
JAR 的获取支持在应用模式下从 [文件系统]({{< ref "docs/deployment/filesystems/overview" >}}) 或 HTTP(S) 下载。
JAR 将被下载到镜像中的 [user.artifacts.base-dir]({{< ref "docs/deployment/config" >}}#user-artifacts-base-dir)/[kubernetes.namespace]({{< ref "docs/deployment/config" >}}#kubernetes-namespace)/[kubernetes.cluster-id]({{< ref "docs/deployment/config" >}}#kubernetes-cluster-id) 路径。
{{< /hint >}}

`kubernetes.cluster-id` 选项指定了集群的名称，必须是唯一的。
如果不指定此选项，Flink 将生成一个随机名称。

`kubernetes.container.image.ref` 选项指定了启动Pod所使用的镜像。

一旦应用程序集群部署完成，你就可以与其进行交互：

```bash
# List running job on the cluster
$ ./bin/flink list --target kubernetes-application -Dkubernetes.cluster-id=my-first-application-cluster
# Cancel running job
$ ./bin/flink cancel --target kubernetes-application -Dkubernetes.cluster-id=my-first-application-cluster <jobId>
```

你可以通过向 `bin/flink` 传递键值对 `-Dkey=value` 来覆盖 [Flink配置文件]{{< ref "docs/deployment/config#flink-配置文件" >}} 中的设置。

### Per-Job 集群模式

在 Kubernetes 上的 Flink 不支持 Per-Job 集群模式。

### Session 模式

你已经在本页面顶部的[入门](#getting-started)指南中看到了 Session 集群的部署。

Session模式可以在两种方式下执行：

* **分离模式**（默认）：`kubernetes-session.sh`会部署Flink集群到Kubernetes，然后退出。

* **附属模式** (`-Dexecution.attached=true`): `kubernetes-session.sh` 保持活动状态，允许输入命令来控制正在运行的 Flink 集群。
  例如，`stop` 命令可以停止运行中的 Session 集群。
  输入 `help` 列出所有支持的命令。

要重新连接到运行中的会话集群，其集群ID为`my-first-flink-cluster`，请使用以下命令：

```bash
$ ./bin/kubernetes-session.sh \
    -Dkubernetes.cluster-id=my-first-flink-cluster \
    -Dexecution.attached=true
```

你可以通过向 `bin/kubernetes-session.sh` 传递键值对 `-Dkey=value` 来覆盖 [Flink 配置文件]({{< ref "docs/zh/deployment/config#flink-配置文件" >}}) 中的设置。

#### 停止一个正在运行的 Session 集群

要停止具有集群ID `my-first-flink-cluster` 的运行中的Session 集群，你可以[删除Flink部署](#manual-resource-cleanup)或使用：

```bash
$ echo 'stop' | ./bin/kubernetes-session.sh \
    -Dkubernetes.cluster-id=my-first-flink-cluster \
    -Dexecution.attached=true
```

{{< top >}}

## Flink 在 Kubernetes 上的参考

### 在 Kubernetes 上配置 Flink

Kubernetes 特定的配置选项列在[配置页面]({{< ref "docs/deployment/config" >}}#kubernetes)上。

Flink 使用 [Fabric8 Kubernetes 客户端](https://github.com/fabric8io/kubernetes-client) 与 Kubernetes APIServer 通信，以创建/删除 Kubernetes 资源（如 Deployment、Pod、ConfigMap、Service 等），以及监视 Pods 和 ConfigMaps。
除了上述 Flink 配置选项外，还可以通过系统属性或环境变量配置 Fabric8 Kubernetes 客户端的一些[专家选项](https://github.com/fabric8io/kubernetes-client#configuring-the-client)。

例如，用户可以使用以下 Flink 配置选项设置最大并发请求数：

```yaml
containerized.master.env.KUBERNETES_MAX_CONCURRENT_REQUESTS: 200
env.java.opts.jobmanager: "-Dkubernetes.max.concurrent.requests=200"
```

### 访问 Flink 的 Web UI

可以通过 [kubernetes.rest-service.exposed.type]({{< ref "docs/deployment/config" >}}#kubernetes-rest-service-exposed-type) 配置选项以多种方式开放 Flink 的 Web UI 和 REST 端点。

- **ClusterIP**：在集群内部 IP 上公开服务。
  服务只能在集群内访问。
  如果要访问 JobManager UI 或向现有会话提交作业，需要启动本地代理。
  然后你可以使用 `localhost:8081` 向会话提交 Flink 作业或查看仪表板。

```bash
$ kubectl port-forward service/<ServiceName> 8081
```
- **NodePort**：在每个节点的 IP 上的静态端口（`NodePort`）上公开服务。可以通过
  `<NodeIP>:<NodePort>` 联系 JobManager 服务。

- **LoadBalancer**：使用云提供商的负载均衡器外部公开服务。
  由于云提供商和 Kubernetes 需要一些时间来准备负载均衡器，你可能会在客户端日志中看到一个 `NodePort` JobManager Web 界面。
  你可以使用 `kubectl get services/<cluster-id>-rest` 获取 EXTERNAL-IP，并手动构造负载均衡器 JobManager Web 界面 `http://<EXTERNAL-IP>:8081`。

更多信息，请参阅 Kubernetes 的官方文档 [在 Kubernetes 中发布服务](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)。

{{< hint warning >}}
根据你的环境，使用 `LoadBalancer` 类型的 REST 服务公开的 Flink 集群可能会使集群对外部公开（通常具有执行任意代码的能力）。
{{< /hint >}}

### 日志记录

Kubernetes 集成将 `conf/log4j-console.properties` 和 `conf/logback-console.xml` 作为 ConfigMap 映射到各个 Pod。
对这些文件的更改将在新启动的集群中可见。

#### 访问日志

默认情况下，JobManager 和 TaskManager 将日志同时输出到控制台和每个 Pod 中的 `/opt/flink/log`。
STDOUT 和 STDERR 输出只会重定向到控制台。
可以通过以下方式访问它们：

```bash
$ kubectl logs <pod-name>
```

如果 Pod 正在运行，也可以使用 `kubectl exec -it <pod-name> bash` 进行隧道连接并查看日志或调试进程。

```bash
$ kubectl logs <pod-name>
```
#### 访问 TaskManager 的日志

为了不浪费资源，Flink 会自动释放空闲的 TaskManagers。这可能使得访问相应 Pod 的日志变得更加困难。通过配置 [resourcemanager.taskmanager-timeout]({{< ref "docs/deployment/config" >}}#resourcemanager-taskmanager-timeout)，可以增加释放空闲 TaskManager 之前的时间，以便有更多时间检查日志文件。

#### 动态更改日志级别

如果已配置日志记录器以 [自动检测配置更改]({{< ref "docs/deployment/advanced/logging" >}})，则可以通过更改相应的 ConfigMap（假设集群 ID 为 `my-first-flink-cluster`）来动态调整日志级别：

```bash
$ kubectl edit cm flink-config-my-first-flink-cluster
```


#### Accessing the Logs of the TaskManagers

Flink will automatically de-allocate idling TaskManagers in order to not waste resources.
This behaviour can make it harder to access the logs of the respective pods.
You can increase the time before idling TaskManagers are released by configuring [resourcemanager.taskmanager-timeout]({{< ref "docs/deployment/config" >}}#resourcemanager-taskmanager-timeout) so that you have more time to inspect the log files.

#### Changing the Log Level Dynamically

If you have configured your logger to [detect configuration changes automatically]({{< ref "docs/deployment/advanced/logging" >}}), then you can dynamically adapt the log level by changing the respective ConfigMap (assuming that the cluster id is `my-first-flink-cluster`):

```bash
$ kubectl edit cm flink-config-my-first-flink-cluster
```

### 使用插件

要使用[插件]({{< ref "docs/deployment/filesystems/plugins" >}})，必须将它们复制到Flink JobManager/TaskManager pod的正确位置。
你可以在不挂载卷或构建自定义Docker镜像的情况下使用[内置插件]({{< ref "docs/deployment/resource-providers/standalone/docker" >}}#using-plugins)。
例如，要为你的Flink会话集群启用S3插件，请使用以下命令：

```bash
$ ./bin/kubernetes-session.sh \
    -Dcontainerized.master.env.ENABLE_BUILT_IN_PLUGINS=flink-s3-fs-hadoop-{{< version >}}.jar \ 
    -Dcontainerized.taskmanager.env.ENABLE_BUILT_IN_PLUGINS=flink-s3-fs-hadoop-{{< version >}}.jar
```

### 自定义Docker镜像

如果你想使用自定义Docker镜像，可以通过配置选项`kubernetes.container.image.ref`指定。Flink社区提供了一个丰富的[Flink Docker镜像]({{< ref "docs/deployment/resource-providers/standalone/docker" >}})，这可能是一个很好的起点。
有关如何启用插件、添加依赖项和其他选项的更多信息，请参阅[如何定制Flink的Docker镜像]({{< ref "docs/deployment/resource-providers/standalone/docker" >}}#customize-flink-image)。

### 使用Secrets

[Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) 是一个包含少量敏感数据（如密码、令牌或密钥）的对象。
这些信息可能会放在pod规范或镜像中。在Kubernetes上的Flink可以以两种方式使用Secrets：

* 从pod中作为文件使用Secrets；
  
* 作为环境变量使用Secrets。

#### 从Pod中作为文件使用Secrets

以下命令将在启动的pod中将秘密`mysecret`挂载到路径`/path/to/secret`下：

```bash
$ ./bin/kubernetes-session.sh -Dkubernetes.secrets=mysecret:/path/to/secret
```

然后，可以在文件`/path/to/secret/username`和`/path/to/secret/password`中找到秘密`mysecret`的用户名和密码。
更多详细信息，请参阅[官方Kubernetes文档](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。

#### 作为环境变量使用Secrets

以下命令将在启动的pod中将秘密`mysecret`作为环境变量暴露：

```bash
$ ./bin/kubernetes-session.sh -Dkubernetes.env.secretKeyRef=\\
    env:SECRET_USERNAME,secret:mysecret,key:username;\\
    env:SECRET_PASSWORD,secret:mysecret,key:password
```

环境变量`SECRET_USERNAME`包含`mysecret`秘密的用户名，而环境变量`SECRET_PASSWORD`包含`mysecret`秘密的密码。
更多详细信息，请参阅[官方Kubernetes文档](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)。

### 在Kubernetes上的高可用性

为了在Kubernetes上实现高可用性，你可以使用[现有的高可用性服务]({{< ref "docs/deployment/ha/overview" >}})。

将<a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-replicas">kubernetes.jobmanager.replicas</a>的值设置为大于1，以启动备用JobManagers。
这有助于实现更快的恢复。请注意，在启动备用JobManagers时应启用高可用性。

It will help to achieve faster recovery.
Notice that high availability should be enabled when starting standby JobManagers.

### Manual Resource Cleanup

Flink uses [Kubernetes OwnerReference's](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/) to clean up all cluster components.
All the Flink created resources, including `ConfigMap`, `Service`, and `Pod`, have the `OwnerReference` being set to `deployment/<cluster-id>`.
When the deployment is deleted, all related resources will be deleted automatically.

```bash
$ kubectl delete deployment/<cluster-id>
```

### Supported Kubernetes Versions

Currently, all Kubernetes versions `>= 1.9` are supported.

### Namespaces

[Namespaces in Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) divide cluster resources between multiple users via [resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).
Flink on Kubernetes can use namespaces to launch Flink clusters.
The namespace can be configured via [kubernetes.namespace]({{< ref "docs/deployment/config" >}}#kubernetes-namespace).

### RBAC

Role-based access control ([RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)) is a method of regulating access to compute or network resources based on the roles of individual users within an enterprise.
Users can configure RBAC roles and service accounts used by JobManager to access the Kubernetes API server within the Kubernetes cluster.

Every namespace has a default service account. However, the `default` service account may not have the permission to create or delete pods within the Kubernetes cluster.
Users may need to update the permission of the `default` service account or specify another service account that has the right role bound.

```bash
$ kubectl create clusterrolebinding flink-role-binding-default --clusterrole=edit --serviceaccount=default:default
```

If you do not want to use the `default` service account, use the following command to create a new `flink-service-account` service account and set the role binding.
Then use the config option `-Dkubernetes.service-account=flink-service-account` to configure the JobManager pod's service account used to create and delete TaskManager pods and leader ConfigMaps.
Also this will allow the TaskManager to watch leader ConfigMaps to retrieve the address of JobManager and ResourceManager.

```bash
$ kubectl create serviceaccount flink-service-account
$ kubectl create clusterrolebinding flink-role-binding-flink --clusterrole=edit --serviceaccount=default:flink-service-account
```

Please refer to the official Kubernetes documentation on [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) for more information.

### Pod Template

Flink allows users to define the JobManager and TaskManager pods via template files. This allows to support advanced features
that are not supported by Flink [Kubernetes config options]({{< ref "docs/deployment/config" >}}#kubernetes) directly.
Use [`kubernetes.pod-template-file.default`]({{< ref "docs/deployment/config" >}}#kubernetes-pod-template-file-default)
to specify a local file that contains the pod definition. It will be used to initialize the JobManager and TaskManager.
The main container should be defined with name `flink-main-container`.
Please refer to the [pod template example](#example-of-pod-template) for more information.

#### Fields Overwritten by Flink

Some fields of the pod template will be overwritten by Flink.
The mechanism for resolving effective field values can be categorized as follows:
* **Defined by Flink:** User cannot configure it.
* **Defined by the user:** User can freely specify this value. Flink framework won't set any additional values and the effective value derives from the config option and the template.

  Precedence order: First an explicit config option value is taken, then the value in pod template and at last the default value of a config option if nothing is specified.
* **Merged with Flink:** Flink will merge values for a setting with a user defined value (see precedence order for "Defined by the user"). Flink values have precedence in case of same name fields.

Refer to the following tables for the full list of pod fields that will be overwritten.
All the fields defined in the pod template that are not listed in the tables will be unaffected.

**Pod Metadata**
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 10%">Key</th>
            <th class="text-left" style="width: 20%">Category</th>
            <th class="text-left" style="width: 30%">Related Config Options</th>
            <th class="text-left" style="width: 40%">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>name</td>
            <td>Defined by Flink</td>
            <td></td>
            <td>The JobManager pod name will be overwritten with the deployment which is defined by <a href="{{< ref "docs/deployment/config" >}}#kubernetes-cluster-id">kubernetes.cluster-id</a>.
                The TaskManager pod names will be overwritten with the pattern <code>&lt;clusterID&gt;-&lt;attempt&gt;-&lt;index&gt;</code> which is generated by Flink ResourceManager.</td>
        </tr>
        <tr>
            <td>namespace</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-namespace">kubernetes.namespace</a></td>
            <td>Both the JobManager deployment and TaskManager pods will be created in the user specified namespace.</td>
        </tr>
        <tr>
            <td>ownerReferences</td>
            <td>Defined by Flink</td>
            <td></td>
            <td>The owner reference of JobManager and TaskManager pods will always be set to the JobManager deployment.
                Please use <a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-owner-reference">kubernetes.jobmanager.owner.reference</a> to control when the deployment is deleted.</td>
        </tr>
        <tr>
            <td>annotations</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-annotations">kubernetes.jobmanager.annotations</a>
                <a href="{{< ref "docs/deployment/config" >}}#kubernetes-taskmanager-annotations">kubernetes.taskmanager.annotations</a></td>
            <td>Flink will add additional annotations specified by the Flink configuration options.</td>
        </tr>
        <tr>
            <td>labels</td>
            <td>Merged with Flink</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-labels">kubernetes.jobmanager.labels</a>
                <a href="{{< ref "docs/deployment/config" >}}#kubernetes-taskmanager-labels">kubernetes.taskmanager.labels</a></td>
            <td>Flink will add some internal labels to the user defined values.</td>
        </tr>
    </tbody>
</table>

**Pod Spec**
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 10%">Key</th>
            <th class="text-left" style="width: 20%">Category</th>
            <th class="text-left" style="width: 30%">Related Config Options</th>
            <th class="text-left" style="width: 40%">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>imagePullSecrets</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-container-image-pull-secrets">kubernetes.container.image.pull-secrets</a></td>
            <td>Flink will add additional pull secrets specified by the Flink configuration options.</td>
        </tr>
        <tr>
            <td>nodeSelector</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-node-selector">kubernetes.jobmanager.node-selector</a>
                <a href="{{< ref "docs/deployment/config" >}}#kubernetes-taskmanager-node-selector">kubernetes.taskmanager.node-selector</a></td>
            <td>Flink will add additional node selectors specified by the Flink configuration options.</td>
        </tr>
        <tr>
            <td>tolerations</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-tolerations">kubernetes.jobmanager.tolerations</a>
                <a href="{{< ref "docs/deployment/config" >}}#kubernetes-taskmanager-tolerations">kubernetes.taskmanager.tolerations</a></td>
            <td>Flink will add additional tolerations specified by the Flink configuration options.</td>
        </tr>
        <tr>
            <td>restartPolicy</td>
            <td>Defined by Flink</td>
            <td></td>
            <td>"always" for JobManager pod and "never" for TaskManager pod.
                <br>
                The JobManager pod will always be restarted by deployment. And the TaskManager pod should not be restarted.</td>
        </tr>
        <tr>
            <td>serviceAccount</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-service-account">kubernetes.service-account</a></td>
            <td>The JobManager and TaskManager pods will be created with the user defined service account.</td>
        </tr>
        <tr>
            <td>volumes</td>
            <td>Merged with Flink</td>
            <td></td>
            <td>Flink will add some internal ConfigMap volumes(e.g. flink-config-volume, hadoop-config-volume) which is necessary for shipping the Flink configuration and hadoop configuration.</td>
        </tr>
    </tbody>
</table>

**Main Container Spec**
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 10%">Key</th>
            <th class="text-left" style="width: 20%">Category</th>
            <th class="text-left" style="width: 30%">Related Config Options</th>
            <th class="text-left" style="width: 40%">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>env</td>
            <td>Merged with Flink</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#forwarding-environment-variables">containerized.master.env.{ENV_NAME}</a>
                <a href="{{< ref "docs/deployment/config" >}}#forwarding-environment-variables">containerized.taskmanager.env.{ENV_NAME}</a></td>
            <td>Flink will add some internal environment variables to the user defined values.</td>
        </tr>
        <tr>
            <td>image</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-container-image-ref">kubernetes.container.image.ref</a></td>
            <td>The container image will be resolved with respect to the defined precedence order for user defined values.</td>
        </tr>
        <tr>
            <td>imagePullPolicy</td>
            <td>Defined by the user</td>
            <td><a href="{{< ref "docs/deployment/config" >}}#kubernetes-container-image-pull-policy">kubernetes.container.image.pull-policy</a></td>
            <td>The container image pull policy will be resolved with respect to the defined precedence order for user defined values.</td>
        </tr>
        <tr>
            <td>name</td>
            <td>Defined by Flink</td>
            <td></td>
            <td>The container name will be overwritten by Flink with "flink-main-container".</td>
        </tr>
        <tr>
            <td>resources</td>
            <td>Defined by the user</td>
            <td>Memory: <br>
                    <a href="{{< ref "docs/deployment/config" >}}#jobmanager-memory-process-size">jobmanager.memory.process.size</a>
                    <a href="{{< ref "docs/deployment/config" >}}#taskmanager-memory-process-size">taskmanager.memory.process.size</a>
                <br>
                CPU: <br>
                    <a href="{{< ref "docs/deployment/config" >}}#kubernetes-jobmanager-cpu">kubernetes.jobmanager.cpu</a>
                    <a href="{{< ref "docs/deployment/config" >}}#kubernetes-taskmanager-cpu">kubernetes.taskmanager.cpu</a></td>
            <td>The memory and cpu resources(including requests and limits) will be overwritten by Flink configuration options. All other resources(e.g. ephemeral-storage) will be retained.</td>
        </tr>
        <tr>
            <td>containerPorts</td>
            <td>Merged with Flink</td>
            <td></td>
            <td>Flink will add some internal container ports(e.g. rest, jobmanager-rpc, blob, taskmanager-rpc).</td>
        </tr>
        <tr>
            <td>volumeMounts</td>
            <td>Merged with Flink</td>
            <td></td>
            <td>Flink will add some internal volume mounts(e.g. flink-config-volume, hadoop-config-volume) which is necessary for shipping the Flink configuration and hadoop configuration.</td>
        </tr>
    </tbody>
</table>

#### Example of Pod Template
`pod-template.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jobmanager-pod-template
spec:
  initContainers:
    - name: artifacts-fetcher
      image: busybox:latest
      # Use wget or other tools to get user jars from remote storage
      command: [ 'wget', 'https://path/of/StateMachineExample.jar', '-O', '/flink-artifact/myjob.jar' ]
      volumeMounts:
        - mountPath: /flink-artifact
          name: flink-artifact
  containers:
    # Do not change the main container name
    - name: flink-main-container
      resources:
        requests:
          ephemeral-storage: 2048Mi
        limits:
          ephemeral-storage: 2048Mi
      volumeMounts:
        - mountPath: /opt/flink/volumes/hostpath
          name: flink-volume-hostpath
        - mountPath: /opt/flink/artifacts
          name: flink-artifact
        - mountPath: /opt/flink/log
          name: flink-logs
      # Use sidecar container to push logs to remote storage or do some other debugging things
    - name: sidecar-log-collector
      image: sidecar-log-collector:latest
      command: [ 'command-to-upload', '/remote/path/of/flink-logs/' ]
      volumeMounts:
        - mountPath: /flink-logs
          name: flink-logs
  volumes:
    - name: flink-volume-hostpath
      hostPath:
        path: /tmp
        type: Directory
    - name: flink-artifact
      emptyDir: { }
    - name: flink-logs
      emptyDir: { }
```

### User jars & Classpath

When deploying Flink natively on Kubernetes, the following jars will be recognized as user-jars and included into user classpath:
- Session Mode: The JAR file specified in startup command.
- Application Mode: The JAR file specified in startup command and all JAR files in Flink's `usrlib` folder.

Please refer to the [Debugging Classloading Docs]({{< ref "docs/ops/debugging/debugging_classloading" >}}#overview-of-classloading-in-flink) for details.

{{< top >}}
