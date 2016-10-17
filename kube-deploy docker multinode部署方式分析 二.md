# kube-deploy docker multinode部署方式分析 二


标签： kubernetes k8s kube-deploy 部署 容器

---

在 `kube-deploy docker multinode部署方式分析` 中已经进行了脚本启动 kubernetes 的初步分析, 其中 master.sh 脚本最终仅分析到 kubelet 容器的启动. 在 kubelet 启动后, 最终以若干 pod 的形式完成 apiserver, controller-manager, scheduler 等组件的启动,将在此处进一步分析

## master 部署分析

入口为 master.sh, 最终的启动容器命令为:
```bash
docker run -d \
    --net=host \
    --pid=host \
    --privileged \
    --restart=${RESTART_POLICY} \
    --name kube_kubelet_$(kube::helpers::small_sha) \
    ${KUBELET_MOUNTS} \
    ${IMAGE_HYPERKUBE}:${K8S_VERSION} \
    /hyperkube kubelet \
      --allow-privileged \
      --api-servers=http://localhost:8080 \
      --config=/etc/kubernetes/manifests-multi \
      --cluster-dns=10.0.0.10 \
      --cluster-domain=cluster.local \
      ${CNI_ARGS} \
      ${CONTAINERIZED_FLAG} \
      --hostname-override=${IP_ADDRESS} \
      --pod-infra-container-image="${IMAGE_PAUSE}" \
      --v=2
```

其中容器的启动命令为 `/hyperkube kubelet`, 代表实际启动的是 kubelet 组件.其中关注的两个配置为 `--api-servers=http://localhost:8080`, `--config=/etc/kubernetes/manifests-multi`.

其中 api-servers 指明了 kubelet 将pod信息通知到哪里. 此时 api-server 并没有启动, 但是仍然可以设置此值, 可以知道 kubelet 会容忍 api-server 暂时性的不在或掉线等情况,并在重新连接成功后通知 api-server 自己所管理的 pod 等信息.

另外 config 指明了 kubelet 监控的目录.因为 kubelet 除了根据 apiserver 启动 pod, 也能根据文件或 http endpoint 或作为 http server 三种方式来启动 pod. 此处即使用了根据文件来启动 pod. 监控的文件夹为镜像内的 `/etc/kubernetes/manifests-multi` 目录

此 manifests-multi 目录下存在 `master-multi.json`, `kube-proxy.json`, `addon-manager.json` 三个json文件.接下来分别分析这三个文件.

### master-multi.json

文件内容如下:
```json
{
"apiVersion": "v1",
"kind": "Pod",
"metadata": {
  "name": "k8s-master",
  "namespace": "kube-system"
},
"spec":{
  "hostNetwork": true,
  "containers":[
    {
      "name": "controller-manager",
      "image": "gcr.io/google_containers/hyperkube-amd64:v1.3.6",
      "command": [
              "/hyperkube",
              "controller-manager",
              "--master=127.0.0.1:8080",
              "--service-account-private-key-file=/srv/kubernetes/server.key",
              "--root-ca-file=/srv/kubernetes/ca.crt",
              "--min-resync-period=3m",
              "--v=2"
      ],
      "volumeMounts": [
        {
          "name": "data",
          "mountPath": "/srv/kubernetes"
        }
      ]
    },
    {
      "name": "apiserver",
      "image": "gcr.io/google_containers/hyperkube-amd64:v1.3.6",
      "command": [
              "/hyperkube",
              "apiserver",
              "--service-cluster-ip-range=10.0.0.1/24",
              "--insecure-bind-address=0.0.0.0",
              "--etcd-servers=http://127.0.0.1:4001",
              "--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota",
              "--client-ca-file=/srv/kubernetes/ca.crt",
              "--basic-auth-file=/srv/kubernetes/basic_auth.csv",
              "--min-request-timeout=300",
              "--tls-cert-file=/srv/kubernetes/server.cert",
              "--tls-private-key-file=/srv/kubernetes/server.key",
              "--token-auth-file=/srv/kubernetes/known_tokens.csv",
              "--allow-privileged=true",
              "--v=2"
      ],
      "volumeMounts": [
        {
          "name": "data",
          "mountPath": "/srv/kubernetes"
        }
      ]
    },
    {
      "name": "scheduler",
      "image": "gcr.io/google_containers/hyperkube-amd64:v1.3.6",
      "command": [
              "/hyperkube",
              "scheduler",
              "--master=127.0.0.1:8080",
              "--v=2"
        ]
    },
    {
      "name": "setup",
      "image": "gcr.io/google_containers/hyperkube-amd64:v1.3.6",
      "command": [
              "/setup-files.sh",
              "IP:10.0.0.1,DNS:kubernetes,DNS:kubernetes.default,DNS:kubernetes.default.svc,DNS:kubernetes.default.svc.cluster.local"
      ],
      "volumeMounts": [
        {
          "name": "data",
          "mountPath": "/data"
        }
      ]
    }
  ],
  "volumes": [
    {
      "name": "data",
      "emptyDir": {}
    }
  ]
 }
}

```

此处启动了一个 pod, 名称为 k8s-master, 使用 namespace 为 kube-system. pod 中共有4个容器, 分别运行 apiserver, controller-manager, scheduler 和 setup-files.sh. 前三个为 kubernetes 的集群组件, 最后的一个用于生成相关认证文件. 认证文件的交换利用了 pod 的 volumes emptyDir 特性, 可以将同一卷挂载到不同容器中并均可以读写 . pod 中有个名称为 data 的卷,在 setup 容器中挂载在 /data 下, 在 apiserver 容器中挂载在 /srv/kubernetes 下, 在 controller-manager 容器中挂载在 /srv/kubernetes 下.

setup-files.sh 主要做的事情包括:
1. 在 ca.crt 不存在时, 调用 make-ca-cert.sh, 通过使用 easy-rsa, 根据 ip 和 设置的 dns 名称,生成 ca.crt, server.cert, server.key, kubecfg.crt, kubecfg.key 这几个认证用的相关文件.
2. 在 basic_auth.csv 不存在时, 使用 `admin,admin,admin` 生成此文件.
3. 在 known_tokens.csv 不存在时, 使用 `随机值,admin,admin`, `随机值,kubelet,kubelet`, `随机值,kube_proxy,kube_proxy` 生成此文件.
4. 执行完毕后死循环 sleep.

### kube-proxy.json

kube-proxy.json文件内容为:
```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "k8s-proxy",
    "namespace": "kube-system"
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-proxy",
        "image": "gcr.io/google_containers/hyperkube-amd64:v1.3.6",
        "command": [
                "/hyperkube",
                "proxy",
                "--master=http://127.0.0.1:8080",
                "--v=2",
                "--resource-container=\"\""
        ],
        "securityContext": {
          "privileged": true
        }
      }
    ]
  }
}
```

此处启动了一个 pod, 名称为 k8s-proxy, namespace 为 kube-system. pod中运行一个容器 kube-proxy. 

### addon-manager.json
文件内容为:
```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "kube-addon-manager",
    "namespace": "kube-system",
    "version": "v1"
  },
  "spec": {
    "hostNetwork": true,
    "containers": [
      {
        "name": "kube-addon-manager",
        "image": "gcr.io/google-containers/kube-addon-manager-amd64:v4",
        "resources": {
          "requests": {
            "cpu": "5m",
            "memory": "50Mi"
          }
        },
        "volumeMounts": [
          {
            "name": "addons",
            "mountPath": "/etc/kubernetes/",
            "readOnly": true
          }
        ]
      },
      {
        "name": "kube-addon-manager-data",
        "image": "gcr.io/google_containers/hyperkube-amd64:v1.3.6",
        "command": [
          "/copy-addons.sh"
        ],
        "volumeMounts": [
          {
            "name": "addons",
            "mountPath": "/srv/kubernetes/",
            "readOnly": false
          }
        ]
      }
    ],
    "volumes":[
      {
        "name": "addons",
        "emptyDir": {}
      }
    ]
  }
}

```

此处启动了一个 pod, 名称为 kube-addon-manager, namespace 为 kube-system. pod中运行一个容器 kube-proxy. pod中运行了两个容器, kube-addon-manager 和 kube-addon-manager-data. 通过 addons 卷, 分别挂载在 /etc/kubernetes/ 和 /srv/kubernetes/ 下完成数据交换.

kube-addon-manager 使用镜像自带启动命令,并进行了 cpu 和内存的限制; kube-addon-manager-data 执行了 copy-addons.sh 脚本.

其中 kube-addon-manager 的启动命令为`/opt/kube-addons.sh`, copy-addons.sh 脚本执行的命令为 `cp -r /etc/kubernetes/* /srv/kubernetes/`.

kube-addons.sh 执行的内容如下:
1. 创建 add-ons 使用的 namespace, 使用 /opt/namespace.yaml 文件,创建的 namespace 名称为 kube-system.
2. 获取 kube-system 的 serviceaccount.
3. 查询 /etc/kubernetes/admission-controls 下的 yaml 或 json 文件,并进行添加.当前无此文件夹.
4. 执行死循环,使用 kube-addon-update.sh 脚本监控 /etc/kubernetes/addon 目录.
5. kube-addon-update.sh 中,扫描目录下的文件,根据文件中描述的 kubernetes 类型,进行对象的创建或删除或更新.
6. 目录下的文件现有dashboard-controller.yaml, dashboard-service.yaml, skydns-rc.yaml,skydns-svc.yaml.
7. 通过这几个文件,会启动 dashboard 和 dns 的 rc 和 service.

## worker 部署分析
入口为 worker.sh, 最终的启动容器命令为:
```bash
docker run -d \
    --net=host \
    --pid=host \
    --privileged \
    --restart=${RESTART_POLICY} \
    --name kube_kubelet_$(kube::helpers::small_sha) \
    ${KUBELET_MOUNTS} \
    ${IMAGE_HYPERKUBE}:${K8S_VERSION} \
    /hyperkube kubelet \
      --allow-privileged \
      --api-servers=http://${MASTER_IP}:8080 \
      --cluster-dns=10.0.0.10 \
      --cluster-domain=cluster.local \
      ${CNI_ARGS} \
      ${CONTAINERIZED_FLAG} \
      --hostname-override=${IP_ADDRESS} \
      --pod-infra-container-image="${IMAGE_PAUSE}" \
      --v=2
```
并根据需要启动proxy:
```bash
docker run -d \
    --net=host \
    --privileged \
    --name kube_proxy_$(kube::helpers::small_sha) \
    --restart=${RESTART_POLICY} \
    ${IMAGE_HYPERKUBE}:${K8S_VERSION} \
    /hyperkube proxy \
        --master=http://${MASTER_IP}:8080 \
        --v=2
```

