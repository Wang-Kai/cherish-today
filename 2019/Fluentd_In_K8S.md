## 实验目的

将 FLuentd 以 DaemonSet 的类型部署在 K8S 集群中，使运行在 K8S 中 Pod 的日志都能收集到指定的 ES 集群中

## 涉及到的技术

1. `Fluentd` 一款灵活的日志收集器
2. `Elasticsearch` 分布式查询引擎
3. `K8S` 容器编排工具

## 操作步骤

### 0. 编写 Fluentd 配置文件

```
# 指明 Input 为 in_tail 类型
# 日志来源为 /var/log/containers/*.log 文件
# 日志结构为 JSON
# 日志标签为 k8s.*，该标签相当于k8s.var.log.containers.${filename}.log
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag k8s.*
  time_format %Y-%m-%dT%H:%M:%S
  read_from_head true
  format json
</source>

# 指明使用 kubernetes_metadata 过滤器，其将在容器原有的日志上添加字段
# 包括：container_id、container_name、pod_name、namespace_name 从而更加明晰哪个 Pod 输出了哪些日志，便于筛选
<filter k8s.**>
  @type kubernetes_metadata
</filter>

# 指明 Output 类型为 elasticsearch
# 指明 ES 集群地址为 10.23.11.243
# 使用 logstash 格式来创建 ES Index
<match k8s.**>
    @type elasticsearch
    host 10.23.11.243
    logstash_format true
    port 9200
</match>
```

### 1. 测试 0 步骤中的 FLuentd 配置是否工作

##### 1.0 登录到一台 k8s worker 节点，安装 `td-agent`，以 Centos 为例：

```
curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent3.sh | sh
```

##### 1.1 安装 kubernetes_metadata 插件

```
td-agent-gem install fluent-plugin-kubernetes_metadata_filter
```

##### 1.2 加载 0 中的配置文件，启动 Fluentd

```
td-agent -c path/to/my_flentd.conf
```

持续制造日志，检查日志是否被正确的收集到了 ES 集群中。如有异常可在 fluentd 默认的日志文件中查看 `tail -f /var/log/td-agent/td-agent.log`

检查日志会发现，每条 Pod 被添加了几个 K8S 相关的参数，示例为 Nginx access 日志：

```json
{
    "log":"10.23.74.63 - - [15/May/2019:01:45:51 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
",
    "stream":"stdout",
    "docker":{
        "container_id":"d4a3c6fabe90ea52d58eeb2d99807acd016e51a2e8f99c3405887a96d3c7f7bf"
    },
    "kubernetes":{
        "container_name":"nginx",
        "namespace_name":"default",
        "pod_name":"nginx-log-789f5b8cdb-9l7d6"
    }
}
```



### 2. 在 K8S 中创建 ConfigMap，保存 Fluentd 配置

```yaml
data:
  collect-k8s-log: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag k8s.log.*
      time_format %Y-%m-%dT%H:%M:%S
      read_from_head false
      format json
    </source>

    <filter k8s.**>
      @type kubernetes_metadata
    </filter>

    <match k8s.**>
      @type elasticsearch
      host 10.23.11.243
      logstash_format true
      port 9200
    </match>
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: default
```

将 Fluentd 的配置文件保存在 ConfigMap 中，key 为 `collect-k8s-log`，将该文件保存为 Fluentd\_ConfigMap.yaml，然后在K8S 中创建该资源 `kubectl create -f Fluentd_ConfigMap.yaml`。

### 3. 配置一个对 Pod & NameSpace 有相关权限的 ServiceAccount

```yaml
# 授权 serviceAccountName: fluentd-es 可以对 namespace & pods 执行 get watch and list 权限

# 创建一个 ServiceAccount -- fluentd-es
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---

# 创建一个 ClusterRole -- fluentd-es，该集群角色可以
# get、watch、list namespaces & pods
# 其中 apiGroup "" 代表核心 API
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "namespaces"
  - "pods"
  verbs:
  - "get"
  - "watch"
  - "list"
---

# 为 fluentd-es ServiceAccount 绑定 fluentd-es 集群角色
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: fluentd-es
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""
```

将该配置文件命名为 `RBAC.yaml`，然后在 K8S 中创建这三个资源 `kubectl create -f RBAC.yaml`

### 4. 创建运行 Fluentd 的 DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-collector
spec:
  selector:
    matchLabels:
      k8s-app: fluentd
  template:
    metadata:
      labels:
        k8s-app: fluentd
    spec:
      # 绑定服务名称来获取相应的权限
      serviceAccountName: fluentd-es
      # 挂载需要的数据卷
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: config-volume
        configMap:
          name: fluentd-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      # Pod 中运行的 Docker Container
      containers:
      - name: fluentd-instance
        image: uhub.service.ucloud.cn/wangkai/fluentd-elasticsearch:v2.0.4
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        env:
        - name: FLUENTD_ARGS
          value: -c /home/collect-k8s-log -o /var/log/fluentd.log
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: config-volume
          mountPath: /home
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
```

该配置文件中挂载了宿主机的 /var/log & /var/lib/docker/containers 两个目录。原因解释如下：

```
[root@10-23-74-63 containers]# ls -alth /var/log/containers
总用量 8.0K
drwxr-xr-x   2 root root 4.0K 5月  15 14:49 .
lrwxrwxrwx   1 root root   73 5月  15 14:48 fluentd-collector-d9qnc_default_fluentd-instance-77c0b0b86561423325d80a66ce0e1bd64a660d353212575b732829d27952de82.log -> /var/log/pods/81d034a3-76dd-11e9-abac-525400933d09/fluentd-instance/0.log
```

`/var/log/containers` 下的文件为软链接文件，链接的文件地址为 `/var/log/pods`。

```
[root@10-23-74-63 containers]# ls -alth /var/log/pods/81d034a3-76dd-11e9-abac-525400933d09/fluentd-instance
总用量 0
drwxr-xr-x 2 root root  19 5月  15 14:48 .
lrwxrwxrwx 1 root root 165 5月  15 14:48 0.log -> /var/lib/docker/containers/77c0b0b86561423325d80a66ce0e1bd64a660d353212575b732829d27952de82/77c0b0b86561423325d80a66ce0e1bd64a660d353212575b732829d27952de82-json.log
drwxr-xr-x 3 root root  30 5月  15 14:48 ..
```

`/var/log/pods` 下的文件又是软链接到了 `/var/lib/docker/containers` 下，所以需要同时挂载  /var/log & /var/lib/docker/containers 两个目录。**假使自己修改了 Docker Root Dir 需要替换默认的 /var/lib/docker/containers**。具体 `Docker Root Dir` 可通过 `docker info` 查看。

YAML 配置文件中还将 ConfigMap 挂载到了 `/home` 目录，这样做的结果是在 Container 的 `/home` 目录下有一个 `collect-k8s-log` 文件，其内容为我们第 0 步写下的 Fluentd 配置文件。

还有一个关注点是 fluentd 容器启动参数 `-c /home/collect-k8s-log -o /var/log/fluentd.log` 分别指明了需要加载的 fluentd 配置文件地址和 fluentd 日志输出地址。对于容器的资源分配可根据实际情况调整，但必须指定。

将该文件命名为 `Fluentd-DaemonSet.yaml` 然后创建该资源 `kubectl create -f Fluentd-DaemonSet.yaml` 。

### 5. 检查 Kibana 的日志收集情况

注意点：

- 推测 `Fluentd elasticsearch` 插件背后向 ES 的插入逻辑是批处理，需要日志达到一定量或时间周期才会向 ES 存储。

## 结论

整个操作流程思路是十分清晰的，挂载日志目录到 Container，使用 Fluentd `tail -f ` 容器的日志文件，使用 `kubernetes_metadata` 插件向日志记录中填充 Pod、NameSpaces 等相关字段信息。最后使用 `elasticsearch` 插件把日志收集到指定的 ES 集群中。

文中的配置基本为可实现功能的最简化配置，后续可以添加一些优化项的配置，使功能更丰富稳定。

## 参考资料

1. [Fluentd 官方文档](https://docs.fluentd.org/v1.0/articles/quickstart)
2. [K8S 官方文档](https://kubernetes.io/docs/home/)
3. [Kubernetes metadata plugin 文档](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter)
4. [阳明的博客](https://www.qikqiak.com/)











