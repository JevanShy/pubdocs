# 通过MinIO+JuiceFS实现K8S中OOM文件自动导出与消息通知

**版权声明**
本文为原创内容，著作权归[Jevanshy]所有。
未经授权**禁止转载**。如需引用或二次创作，请遵守以下规则：

1. **清晰标注**原文标题、作者及原始链接
2. **禁止商业用途**（包括但不限于付费专栏、培训材料）
3. **禁止洗稿**（修改关键代码/截图后伪装原创）
   违反者将追究责任。
   原文永久地址：[JuiceFS_DumpOOM_Files](https://github.com/JevanShy/pubdocs/blob/main/docs/JuiceFS_DumpOOM_Files.md)



## 问题背景



在传统的Kubernetes部署中，工作中面临以下几个典型问题：

- **存储空间管理复杂**：公司目前使用`hostPath`将日志或Dump文件直接挂载到宿主机磁盘，需要为每个节点配置定时清理任务，操作繁杂。当某个应用产生大量日志或文件时，极易占满节点磁盘，影响节点上所有服务的稳定性。
- **临时文件导出不便**：开发和测试人员需要获取容器内的文件（如Jacoco测试报告、Heap Dump文件）时，往往需要运维人员介入，流程繁琐且效率低下。
- **基础镜像臃肿**：为了方便调试，可能会在基础镜像中打包各种工具，导致镜像体积越来越大。

为了解决以上问题，引入了JuiceFS作为持久化存储的核心解决方案。



## 方案选型与架构



**JuiceFS**是一款专为云原生环境设计的高性能分布式文件系统。它的核心架构是将**“数据”和“元数据”分离存储**。数据块本身会`分片`存储在各种对象存储中（如S3、Webdav等），众所周知，`分片`存储对大文件而言效率提升是很客观的，而元数据则由独立的数据库（如Redis、MySQL、TiKV）管理。

选择JuiceFS的原因在于其强大的生态兼容性：

- **POSIX兼容**：通过FUSE，可以像本地磁盘一样直接挂载和使用。
- **Kubernetes CSI驱动**：无缝对接到K8S，管理员不需要因为后端存储介质更换而更换K8S的存储驱动。
- **S3网关**：提供S3兼容的访问接口，方便与其他S3工具链集成。

方案架构如下：

1. **数据存储**：使用MinIO作为对象存储后端，负责存储JuiceFS切分后的实际数据块。在本地或测试环境中，MinIO部署简单，成本低廉。
2. **元数据存储**：使用MySQL数据库存储文件的元数据（文件名、大小、权限等）。
3. **K8S集成**：通过JuiceFS CSI驱动，在K8S中创建`PersistentVolumeClaim` (PVC)，并将其挂载到应用Pod中。
4. **自动化流程**：
   - 配置Java应用的JVM参数，在OOM发生时，将Heap Dump文件输出到挂载的JuiceFS卷中。
   - 利用`-XX:OnOutOfMemoryError`参数触发OOM文件通知脚本。
   - 该脚本通过JuiceFS S3网关生成文件的预签名下载链接，并调用企业微信的Webhook接口发送通知。

![image-20250703141139905](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20250703141139905.png)



## 第一步：部署后端存储 (MinIO)



针对目前本地测试环境，使用Docker Compose快速拉起一个单节点的MinIO服务。

创建`docker-compose.yml`文件：

YAML

```
version: '3.8'
services:
  minio:
    image: "quay.io/minio/minio:RELEASE.2024-12-18T13-15-44Z"
    container_name: minio1
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - "/data/minio_storage/data:/data"
      - "/etc/localtime:/etc/localtime:ro"
      - "/etc/timezone:/etc/timezone:ro" 
      - "/usr/share/zoneinfo/Asia/Shanghai:/usr/share/zoneinfo/Asia/Shanghai:ro"
    command: server --console-address ":9001" /data
    environment:
      - MINIO_ROOT_USER=<Username>
      - MINIO_ROOT_PASSWORD=<Password>
      - TZ=Asia/Shanghai
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
```

启动服务：

Bash

```
docker-compose up -d
```

启动成功后，通过`http://<minio-server-ip>:9001`访问MinIO管理后台。创建一个Bucket（`devjfs`）及`Access Key`和`Secret Key`备用。



*注：本地环境已有Mysql实例，故MySQL的部署此处略过。*



## 第二步：在K8S中部署JuiceFS CSI驱动



官方更推荐使用Helm来部署和管理CSI驱动，因为它能更好地管理复杂的资源和配置。



### 1. 下载Helm Chart



由于网络原因，直接从GitHub的Helm仓库拉取可能会失败。先手动下载Chart包。

Bash

```
# 添加JuiceFS官方仓库
helm repo add juicefs https://juicedata.github.io/charts/
helm repo update

# 如果此步失败，可手动从以下地址下载
# https://github.com/juicedata/charts/releases
# 本次下载 juicefs-csi-driver-0.23.0.tgz
```



### 2. 准备配置文件



为了适应国内环境和私有化部署，我们需要创建一个自定义的values文件（`test-csi-values.yaml`）来覆盖默认配置。

关键配置项说明：

- **镜像地址**：将所有`image.repository`替换为私有镜像仓库或国内镜像加速地址。
- **`kubeletDir`**：若集群为修改过kubelet根目录则需使用对应配置，根目录通过任意一个非 Master 节点上执行`ps -ef | grep kubelet | grep root-dir`命令获得。
- **`pathPattern`**：设置为`${.pvc.name}`可以让JuiceFS在文件系统根目录下创建与PVC同名的文件夹，增强可读性，而不是一长串随机字符。
- **`defaultMountImage`**：为`Mount Pod`指定镜像地址，避免从默认的Docker Hub拉取。

YAML

```
# test-csi-values.yaml

# 1. 因阿里云已进行过镜像搬运，这里直接使用阿里云仓库地址
image:
  repository: registry.cn-hangzhou.aliyuncs.com/juicedata/juicefs-csi-driver
  tag: "v0.27.0"
  pullPolicy: ""
dashboardImage:
  repository: registry.cn-hangzhou.aliyuncs.com/juicedata/csi-dashboard
  tag: "v0.27.0"
  pullPolicy: ""
sidecars:
  livenessProbeImage:
    repository: registry.cn-hangzhou.aliyuncs.com/google_containers/livenessprobe
    tag: "v2.12.0"
    pullPolicy: ""
  nodeDriverRegistrarImage:
    repository: registry.cn-hangzhou.aliyuncs.com/google_containers/csi-node-driver-registrar
    tag: "v2.9.0"
    pullPolicy: ""
  csiProvisionerImage:
    repository: registry.cn-hangzhou.aliyuncs.com/google_containers/csi-provisioner
    tag: "v2.2.2"
    pullPolicy: ""
  csiResizerImage:
    repository: registry.cn-hangzhou.aliyuncs.com/google_containers/csi-resizer
    tag: "v1.9.0"
    pullPolicy: ""

# 2. 全局配置，主要配置了MountImage镜像地址，juicefs-delete-delay配置表示当没有任何Pod引用mountPod后juicefs等待5分钟再删除Pod，避免因调试等原因造成反复创建开销
globalConfig:
  enabled: true
  mountPodPatch:
    - resources:
        requests:
          cpu: 300m
          memory: 512Mi
        limits:
          cpu: 1
          memory: 2Gi
    - pvcSelector:
        matchLabels:
          custom-image: "true"
      ceMountImage: "registry.cn-hangzhou.aliyuncs.com/juicedata/juicedata-mount:ce-v1.2.3"
      mountOptions:
        - cache-size=2048
        - cache-dir=/data/k8s/jfsCache
    - annotations:
        juicefs-delete-delay: 5m

# 3. 设置Kubelet目录 (非常重要，请根据实际环境修改，若根目录为默认值`/var/lib/kubelet`则不需要单独配置)
kubeletDir: 

# 4. 设置默认的Mount Pod镜像
defaultMountImage:
  ce: "registry.cn-hangzhou.aliyuncs.com/juicedata/juicedata-mount:ce-v1.2.3"

# 5. 配置StorageClass以生成可读的PV目录
storageClasses:
  pathPattern: "${.pvc.name}"
  mountPod:
    resources:
      limits:
        cpu: 5000m
        memory: 5Gi
      requests:
        cpu: 1000m
        memory: 1Gi
```



### 3. 执行安装



使用以下命令对已下载的CSI驱动进行安装到`kube-ops`命名空间。

Bash

```
helm upgrade --install juicefs-csi-driver ./juicefs-csi-driver-helm.tgz \
  -f test-csi-values.yaml \
  -n kube-ops \
  --create-namespace
```



## 第三步：创建持久化存储卷 (PVC)



当相关资源安装完成后，创建一个`Secret`来保存MinIO和MySQL的连接信息，然后定义一个`StorageClass`来使用这些信息，最后再创建PVC资源供Pod挂载。

YAML

```
# juicefs-sc-pvc.yaml

apiVersion: v1
kind: Secret
metadata:
  name: juicefs-sc-secret
  namespace: kube-ops # 与CSI驱动部署在同一个命名空间
type: Opaque
stringData:
  name: "devjfs" # 文件系统名称
  # 替换为你的MySQL元数据数据库地址
  metaurl: "mysql://<Mysql User>:<Mysql Password>@(<Mysql Address>)/<DB Name>"
  storage: "minio" # 存储类型
  # 替换为你的MinIO地址和Bucket
  bucket: "http://<minio Address>/devjfs"
  # 替换为你的MinIO凭证
  access-key: "<minio ak>"
  secret-key: "<minio sk>"
  envs: "{TZ: Asia/Shanghai}"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jfsdata-juicefs-sc
provisioner: csi.juicefs.com
reclaimPolicy: Retain # 数据保留策略
volumeBindingMode: Immediate
parameters:
  # 引用上面创建的Secret
  csi.storage.k8s.io/node-publish-secret-name: juicefs-sc-secret
  csi.storage.k8s.io/node-publish-secret-namespace: kube-ops
  csi.storage.k8s.io/provisioner-secret-name: juicefs-sc-secret
  csi.storage.k8s.io/provisioner-secret-namespace: kube-ops
  # 使用在values.yaml中定义的pathPattern
  pathPattern: "${.pvc.name}"
allowVolumeExpansion: true # 支持扩容
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jfsdata
  namespace: dev # 你的应用所在的命名空间
  labels:
    # 对应刚刚创建CSI驱动时配置的自定义镜像仓库，若未配置也会使用之前的defaultMountImage配置
    custom-image: "true" 
spec:
  accessModes:
    - ReadWriteMany # 支持多节点读写
  resources:
    requests:
      storage: 50Gi # 请求的存储空间大小
  storageClassName: jfsdata-juicefs-sc
```

将以上内容保存为`juicefs-sc-pvc.yaml`并应用：

Bash

```
kubectl apply -f juicefs-sc-pvc.yaml
```



## 第四步：部署JuiceFS S3网关



因为JuiceFS会对文件进行切片存储，文件在Minio中无法被直接查看，需要部署S3网关对文件进行管理。

同样，我们使用Helm进行部署。



### 1. 准备S3网关的values文件



YAML

```
# sk-prod-s3-gateway.yaml

image:
  repository: registry.cn-hangzhou.aliyuncs.com/juicedata/juicedata-mount
  tag: "ce-v1.2.3"
  pullPolicy: IfNotPresent

# 定义S3网关要使用的后端存储和元数据信息，这里的ak和sk为S3网关的访问账号及密码，和minio的ak、sk没有关系
secret:
  enabled: true
  name: "jfsdata"
  # 确保这里的配置与之前CSI驱动的Secret一致
  metaurl: "mysql://<Mysql User>:<Mysql Password>@(<Mysql Address>)/<DB Name>"
  storage: "minio" # 存储类型
  accessKey: "<User Name For S3 Gateway>"
  secretKey: "<Password For S3 Gateway>"
  bucket: "http://<minio Address>/devjfs"

# 额外启动参数，--multi-buckets 支持多桶
options: "--multi-buckets --cache-size=2048"

resources:
  limits:
    cpu: 2000m
    memory: 2Gi
  requests:
    cpu: 500m
    memory: 512Mi
```



### 2. 安装S3网关并创建Ingress资源



Bash

```
# 假设你已下载S3网关的Chart包: juicefs-s3-gateway.tgz
helm upgrade --install s3-gateway ./juicefs-s3-gateway.tgz \
  -f sk-prod-s3-gateway.yaml \
  -n kube-ops
```





## 第五步：实现OOM自动导出与通知



### 1. 挂载PVC到应用Pod



修改应用`Deployment`，添加`volumes`和`volumeMounts`，将之前创建的`jfsdata` PVC挂载到容器的指定路径（例如`/data/jfsdata`）。

**关键点**：`mountPropagation`必须设置为`HostToContainer`或`Bidirectional`。这能确保在Mount Pod 重启后，宿主机上的挂载点被重新挂载，然后 CSI 驱动将会在容器挂载路径上重新执行一次 mount bind。

YAML

```
# 在Deployment的Pod template spec中添加

# ...
       volumes:
         - name: jfsdata
           persistentVolumeClaim:
             claimName: jfsdata # 引用在dev命名空间创建的PVC
       
       containers:
         - name: my-java-app
           # ...
           volumeMounts:
             - mountPath: /data/jfsdata # 挂载到容器内的路径
               name: jfsdata
               mountPropagation: HostToContainer
# ...
```



### 2. 准备通知脚本和工具



我们需要一个脚本来生成下载链接并发送通知。这个脚本将和`mc`（MinIO Client）工具一起，放在我们挂载的共享卷中，以便应用容器在OOM时可以调用。

首先，在共享卷中创建一个`tools`目录，并放入`mc`二进制文件。

然后，创建`mc`的别名配置文件 `.jfs_alias.yaml`：

YAML

```
# /data/jfsdata/tools/.jfs_alias.yaml
{
   "url" : "http://s3-gateway.kube-ops.svc.cluster.local", # S3网关的内部访问地址
   "accessKey": "***", # 使用JuiceFS的密钥
   "secretKey": "***",
   "api": "s3v4",
   "path": "auto"
}
```

最后，创建核心的通知脚本`dumpfile_share.sh`：

Bash

```
#!/bin/bash
# /data/jfsdata/tools/dumpfile_share.sh

# 环境变量由应用传入
DUMP_DIR="/data/jfsdata/dumpfiles"
DUMP_FILE_NAME="${project_env}_${pod_name}_heapDump.hprof"
DUMP_FILE="${DUMP_DIR}/${DUMP_FILE_NAME}"

MC_BIN="/data/jfsdata/tools/mc"
ALIAS_FILE="/data/jfsdata/tools/.jfs_alias.yaml"
MC_ALIAS="devjfs" # mc别名

# 企业微信机器人Webhook地址
WECHAT_WEBHOOK="https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=your_webhook_key"

# 1. 导入mc别名，指向我们的JuiceFS S3 Gateway
if ! "${MC_BIN}" alias import ${MC_ALIAS} "${ALIAS_FILE}" >/dev/null 2>&1; then
    echo "[ERROR] 无法配置MinIO别名"
    exit 2
fi

# 2. 生成有效期24小时的预签名下载链接
DOWNLOAD_URL=$(
    "${MC_BIN}" share download --expire 24h "${MC_ALIAS}/jfsdata/dumpfiles/${DUMP_FILE_NAME}" 2>/dev/null | awk 'END{print}' | awk '{print $NF}'
)

if [[ -z "${DOWNLOAD_URL}" ]]; then
    echo "[ERROR] 无法生成下载链接"
    exit 3
fi

# 3. 构造Markdown消息体
MARKDOWN_MSG=$(cat <<EOF
{
    "msgtype": "markdown",
    "markdown": {
        "content": "**OOM事件通知**\n
        >环境：<font color=\"comment\">${project_env}</font>
        >项目名称：<font color=\"warning\">${pkg}</font>
        >容器名称：<font color=\"warning\">${pod_name}</font>
        >下载链接：[点击下载](${DOWNLOAD_URL})
        >触发时间：<font color=\"comment\">$(date +"%Y-%m-%d %H:%M:%S")</font>
        >有效期：<font color=\"comment\">24小时</font>"
    }
}
EOF
)

# 4. 发送通知
HTTP_STATUS=$(
    curl -s -o /dev/null -w "%{http_code}" \
    -X POST "${WECHAT_WEBHOOK}" \
    -H "Content-Type: application/json" \
    -d "${MARKDOWN_MSG}"
)

if [[ "${HTTP_STATUS}" != "200" ]]; then
    echo "[ERROR] 消息发送失败，HTTP状态码: ${HTTP_STATUS}"
    exit 4
fi

echo "[INFO] 通知发送成功！"
exit 0
```



### 3. 修改Java启动参数



最后一步，修改Java应用的启动参数，添加两个关键的JVM参数：

- `-XX:+HeapDumpOnOutOfMemoryError`:当发生OOM时Dump内存。
- `-XX:HeapDumpPath`: 指定OOM时Heap Dump文件的输出路径，指向PVC中的一个子目录。
- `-XX:OnOutOfMemoryError`: 指定在OOM发生时要执行的命令或脚本。

Bash

```
# ...
java -jar \
    # ... 其他JVM参数 ...
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:HeapDumpPath=/data/jfsdata/dumpfiles/${project_env}_${pod_name}_heapDump.hprof \
    -XX:OnOutOfMemoryError="/bin/bash /data/jfsdata/tools/dumpfile_share.sh" \
    app.jar
# ...
```

确保`$project_env`和`$pod_name`等环境变量在容器中可用，以便脚本能生成带有上下文信息的文件名和通知。



### 4.定时清理数据CronJob

为避免Dump文件持续占用磁盘空间，创建一个CronJob任务定期对文件进行清理，为避免因目录下所有文件被删除导致`dumpfiles`目录被删除，创建一个.keep文件并在清理动作前写入`keepdir`到文件中

```shell
apiVersion: batch/v1
kind: CronJob
metadata:
  name: jfsdata-cleanup
  namespace: kube-ops
spec:
  schedule: "0 */6 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: jfsdata-cleaner
              image: mc:RELEASE.2025-02-21T16-00-46Z
              env:
                - name: ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: juicefs-sc-secret
                      key: access-key
                - name: SECRET_KEY
                  valueFrom:
                    secretKeyRef:
                      name: juicefs-sc-secret
                      key: secret-key
              command:
                - /bin/sh
                - -c
                - |
                  echo -n "keepdir" | mc pipe devjfs/jfsdata/dumpfiles/.keep
                  mc rm --recursive --force --older-than 2d devjfs/jfsdata/dumpfiles/
          restartPolicy: OnFailure
```





## 监控与告警



为了确保系统的健壮性，我们需要监控JuiceFS卷的使用情况。



### 1. 采集监控数据



集群中已部署了Prometheus监控，可以创建一个`PodMonitor`资源，自动发现并采集JuiceFS Mount Pod暴露的监控指标。

YAML

```
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: juicefs-mounts-metrics
  namespace: monitor # Prometheus Operator所在的命名空间
spec:
  namespaceSelector:
    matchNames:
      - kube-ops # JuiceFS CSI驱动所在的命名空间
  selector:
    matchLabels:
      app.kubernetes.io/name: juicefs-mount
  endpoints:
    - interval: '30s'
      path: '/metrics'
      port: metrics
      scheme: 'http'
```



### 2. 配置告警规则



我们可以使用`kubelet_volume_stats_*`系列指标来计算PVC的使用率并设置告警。例如，当`jfsdata`这个PVC在任何命名空间的使用率超过80%时触发告警。

代码段

```
# PromQL告警规则
round(
    (
        avg(kubelet_volume_stats_used_bytes{persistentvolumeclaim="jfsdata"}) by (persistentvolumeclaim, namespace) 
      / 
        avg(kubelet_volume_stats_capacity_bytes{persistentvolumeclaim="jfsdata"}) by (persistentvolumeclaim, namespace)
    ) * 100
, 0.01) > 80
```



### 3. Grafana仪表盘



JuiceFS官方提供了Grafana仪表盘模板（ID `20794`），可以直接导入，快速实现可视化监控。



