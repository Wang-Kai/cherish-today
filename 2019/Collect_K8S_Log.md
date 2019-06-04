# UK8S 日志收集解决方案

在 K8S 集群中收集日志是一个较为复杂的工作。首先采集的目标多，集群中可能会有多个服务，每个服务又有多个 Pod 副本，每个 Pod 中又可能有多个容器。此外是日志输出方式不固定，比如有的容器会把日志输出到标准的 `stdout` 和 `stderr`，有的容器会把日志输出到固定日志文件。本文将详尽阐述 `DaemonSet` 和 `Sidecar` 两种日志收集方式，力求简洁有效的完成日志收集工作。

- [申请 UES 服务](#申请-ues-服务)
- [DaemonSet + Fluentd 方式](#daemonset--fluentd-方式)
- [Sidecar + Fluentd 方式](#sidecar--fluentd-方式)
- [DaemonSet + Filebeat 方式](#daemonset--filebeat-方式)
- [Sidecar + Filebeat 方式](#sidecar--filebeat-方式)

### 申请 UES 服务

日志存储是整个流程中的重要一环，[ElasticSearch](https://www.elastic.co/) 是一款优秀的文本搜索引擎，我们可以把收集来的日志存储到 ES 中，便于后续的日志查看和分析。在云计算时代，我们没必要自己再去搭建 ES 集群，可以直接申请 [UES](https://www.ucloud.cn/site/product/ues.html)，快速的得到一个完整的 ES 集群，并且会有专业的团队帮您运维。

![](http://wx1.sinaimg.cn/large/0060lm7Tly1g3dk4lq4svj31200u0jxa.jpg)

填写我们需要部署的可用区、节点个数、ES版本、节点类型等信息，点击创建按钮后就创建好了我们要求的 ES 集群了。

![](http://wx3.sinaimg.cn/large/0060lm7Tly1g3dkgxvfs5j31j00riwic.jpg) 可以在控制台查看到 ES 集群的节点信息。

### DaemonSet + Fluentd 方式

##### 1. 以 DaemonSet 方式部署 Fluentd agent

`DaemonSet Controller` 会在每一个 Node 上部署一个 Pod副本，Pod 的数量会随着 node 的添加或删除而变化，但总会确保每一个 node 上都会有且只有一个 Pod。

```yaml
apiVersion: v1
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
      host  10.9.33.98
      logstash_format true
      port 9200
    </match>
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: default

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    
---

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
  namespace: default
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""

---

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
      serviceAccountName: fluentd-es
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

修改上边 yaml 文件 host IP 为自己的 ES 集群 IP，然后将文件 Copy 到 K8S master 节点保存为文件 `fluentd-agent.yaml`，然后执行 `kubectl create -f fluentd-agent.yaml`。**此时，我们就完成了以 DaemonSet 方式通过 Fluentd 来收集 K8S 日志的所有工作。**

##### 2. 通过 kibana 查看日志

UES 为每一个 ES 集群提供了 kibana 日志可视化查看，点击右侧 `打开kibana` 按钮即可跳出 kibana 页面。

![](http://wx4.sinaimg.cn/large/0060lm7Tly1g3dnd4u28dj31to0jytc9.jpg)

通过 kibana，我们可以方便的查看我们的容器日志输出，此外日志中还补全了元数据信息，例如 K8S 命名空间、Pod 名称、容器名称等信息。

![](http://wx1.sinaimg.cn/large/0060lm7Tly1g3dnetu138j314k0u07bi.jpg)

##### 3. 总结

DaemonSet 方式做日志收集基于如下原理：K8S 集群中，所有容器中的标准输出都会以 `POD-NAME_NAMESPACE_CONTAINER-NAME_CONTAINER-ID` 的命名规则存储在宿主机 `/var/log/containers` 目录下。基于这样可循的规则，我们可以在每个 Worker 节点上部署 Fluentd agent 采集每个节点上的日志，并且能将日志对应到唯一的 Pod 以供分析查看。

### Sidecar + Fluentd 方式

##### 1. 部署 sidecar Pod

我们以一个简单的示例 `timer-app` 来模拟应用服务，该程序会每 5 秒输出一行日志到 `/data/test.log` 文件。然后我们在同一 Pod 中另起一个 `Fluentd` 容器，挂载 `/data` 目录并采集日志。

```yaml
# FLuentd 配置文件
apiVersion: v1
kind: ConfigMap
metadata:
  name: sidecar-fluentd-config
  namespace: default
data:
  collect-k8s-log: |
    <source>
      @type tail
      path /data/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag k8s.log
      time_format %Y-%m-%dT%H:%M:%S
      read_from_head false
      format json
    </source>

    <match k8s.**>
      @type elasticsearch
      host  10.9.33.98
      logstash_format true
      port 9200
    </match>

---

apiVersion: v1
kind: Pod
metadata:
  name: timer-log-sidecar
  labels:
    app: timer-log
spec:
  containers:
  - name: timer-app
    image: uhub.service.ucloud.cn/wangkai/timer-log:latest
    volumeMounts:
    - name: demo-dir
      mountPath: /data
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
    - name: demo-dir
      mountPath: /data
    - name: sidecar-fluentd-config
      mountPath: /home
  volumes:
  - name: demo-dir
    emptyDir: {}
  - name: sidecar-fluentd-config
    configMap:
      name: sidecar-fluentd-config
```

更换配置中 Host IP 为自己的集群 IP，然后 Copy 内容到 `sidecar-demo.yaml`，执行 `kubectl create -f sidecar-demo.yaml` **即可完成对 `timer-app` 服务的日志收集。**

##### 2. 通过 kibana 查看日志收集结果

![](http://wx1.sinaimg.cn/large/0060lm7Tly1g3eqz48wjij31a90u0n1w.jpg)

从 kibana 界面可以看出，我们每分钟可以采集到 12 条日志，也就是每 5 秒输出的日志都被成功采集到了。

##### 3. 总结

sidecar 日志采集方式可以采集程序输出到文件中的日志，并且灵活度较高。 Sidecar 方式采集日志是利用了 Pod 中各 Containers 可以共享存储的特性，主容器将日志输出到特定的 Volume，日志收集 Container 挂载该 Volume 并采集其中存储的日志。

### DaemonSet + Filebeat 方式

##### 1. 部署 filebeat DaemonSet

[filebeat](https://www.elastic.co/cn/products/beats/filebeat) 是一款轻量日志采集器。[官方有提供 K8S 集群日志收集方案](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html)，采用的是 DaemonSet 方式。采用官方提供的配置，我们只需要修改 ES 节点地址即可。

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: kube-system
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
data:
  filebeat.yml: |-
    filebeat.config:
      prospectors:
        # Mounted `filebeat-prospectors` configmap:
        path: ${path.config}/prospectors.d/*.yml
        # Reload prospectors configs as they change:
        reload.enabled: false
      modules:
        path: ${path.config}/modules.d/*.yml
        # Reload module configs as they change:
        reload.enabled: false

    processors:
      - add_cloud_metadata:

    cloud.id: ${ELASTIC_CLOUD_ID}
    cloud.auth: ${ELASTIC_CLOUD_AUTH}

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
      username: ${ELASTICSEARCH_USERNAME}
      password: ${ELASTICSEARCH_PASSWORD}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-prospectors
  namespace: kube-system
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
data:
  kubernetes.yml: |-
    - type: docker
      containers.ids:
      - "*"
      processors:
        - add_kubernetes_metadata:
            in_cluster: true
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: filebeat
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: uhub.service.ucloud.cn/wangkai/filebeat:6.2.4
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        env:
        - name: ELASTICSEARCH_HOST
          value: elasticsearch
        - name: ELASTICSEARCH_PORT
          value: "9200"
        - name: ELASTICSEARCH_USERNAME
          value: elastic
        - name: ELASTICSEARCH_PASSWORD
          value: changeme
        - name: ELASTIC_CLOUD_ID
          value:
        - name: ELASTIC_CLOUD_AUTH
          value:
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: prospectors
          mountPath: /usr/share/filebeat/prospectors.d
          readOnly: true
        - name: data
          mountPath: /usr/share/filebeat/data
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: prospectors
        configMap:
          defaultMode: 0600
          name: filebeat-prospectors
      - name: data
        emptyDir: {}
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""] # "" indicates the core API group
  resources:
  - namespaces
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
  namespace: kube-system
  labels:
    k8s-app: filebeat
---
```

将文件内容 Copy 到本地，命名为 `filebeat-daemonset.yaml`, 根据自己申请的 UES 集群信息修改如下参数：

```yaml
- name: ELASTICSEARCH_HOST
  value: elasticsearch
- name: ELASTICSEARCH_PORT
  value: "9200"
- name: ELASTICSEARCH_USERNAME
  value: elastic
- name: ELASTICSEARCH_PASSWORD
  value: changeme
```

ES 默认账号密码为 `elastic` 、`changeme` , 如果 ES 集群没有明确指定账号密码，则不需要修改这两个参数。**执行 `kubectl create -f filebeat-daemonset.yaml` 即可完成 filebeat 收集 K8S 日志的全流程。**

##### 2. 使用 kibana 查看日志

![](http://wx4.sinaimg.cn/large/0060lm7Tly1g3gt8vgeppj315q0u0q8c.jpg)

日志收集效果与 Fluentd DaemonSet 方式相似，同样也可以采集到 `namespace` 、`pod name` 等信息。

### Sidecar + Filebeat 方式

##### 1. 构建 filebeat Docker image

编写 filebeat 配置文件，以 JSON 格式采集 `/data/*.log` 日志。

```yaml
filebeat.inputs:
- type: log
  enable: true
  paths:
    - /data/*.log
  json.keys_under_root: true
  json.add_error_key: true

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

setup.template.settings:
  index.number_of_shards: 1

output.elasticsearch:
  hosts: ["10.9.33.98"]
```
将 yaml 保存到 `/root/filebeat.yml`，然后编写 filebeat 镜像 Dockerfile。**注意，基础镜像中并不一定有我们需要的目录，默认也没有 `root` 权限，最好手动创建我们需要的目录（这里是 `/data`）。**

```Dockerfile
FROM docker.elastic.co/beats/filebeat:7.1.0

COPY /root/filebeat.yml /usr/share/filebeat/filebeat.yml

USER root
RUN mkdir /data

RUN chown root:filebeat /usr/share/filebeat/filebeat.yml
```
在 Dockerfile 所在目录执行镜像构建 `docker build -t uhub.service.ucloud.cn/wangkai/filebeat-steve:v2 .`，并将该镜像推送到 [UHub](https://docs.ucloud.cn/compute/uhub/index) `docker push uhub.service.ucloud.cn/wangkai/filebeat-steve:v2` 。

##### 2. 部署应用服务

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: filebeat-sidecar-demo
  labels:
    app: filebeat-sidecar-demo
spec:
  containers:
  - name: timer-app
    image: uhub.service.ucloud.cn/wangkai/timer-log:latest
    volumeMounts:
    - name: demo-dir
      mountPath: /data
  - name: filebeat
    image: uhub.service.ucloud.cn/wangkai/filebeat-steve:v2
    volumeMounts:
    - name: demo-dir
      mountPath: /data
  volumes:
  - name: demo-dir
    emptyDir: {}
  - name: filebeat-config
    configMap:
      name: sidecar-filebeat-config
```

该示例中，我们使用 `timer-log` Container，其将会创建 `/data/test.log`, 然后定时向文件输出日志。保存 yaml 内容到 `filebeat-sidecar.yaml`, 然后执行 `kubectl create -f filebeat-sidecar.yaml` 。

##### 3. 使用 kibana 查看日志

![](http://wx2.sinaimg.cn/large/0060lm7Tly1g3h8a8y1xvj30uu0r6wir.jpg)

我们看到，日志已经成功被收集到了 ES 中。

##### 4. 总结

使用 filebeat 以 sidecar 方式收集日志的原理同样是基于 Pod 中各 Container 可以共享存储。自行构建 Filebeat 镜像，也是[官方推荐的方法之一](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-docker.html#_custom_image_configuration)。


