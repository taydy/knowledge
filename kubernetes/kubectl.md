### kubectl create

```
kubectl create -f 我的配置文件
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

```

### kubectl get

通过 kubectl get pods 命令检查这个 Pods 运行起来的状态信息。

```shell
$ kubectl get pods -l app=nginx
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-67594d6bf6-9gdvr   1/1       Running   0          10m
nginx-deployment-67594d6bf6-v6j7w   1/1       Running   0          10m
```

通过 kubectl get deployments 命令来检查 deployment 创建后的状态信息。

```sh
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

在返回结果中，我们可以看到四个状态字段，它们的含义如下所示：

- **`DESIRED`**：用户期望的 Pod 副本个数（spec.replicas 的值）
- **`CURRENT`**：当前处于 Running 状态的 Pod 的个数
- **`UP-TO-DATE`**：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致
- **`AVAILABLE`**：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数

### kubectl scale

Deployment Controller 修改它所控制的 ReplicaSet 的 Pod 副本个数来实现 “水平扩展 / 收缩”。

```
$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```

### kubectl set image

kubectl set image 可以直接修改 nginx-deployment 所使用的镜像。

```shell
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.extensions/nginx-deployment image updated
```

### kubectl edit

使用 kubectl edit 指令编辑 Etcd 里的 API 对象。

```shell
$ kubectl edit deployment/nginx-deployment
... 
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1 # 1.7.9 -> 1.9.1
        ports:
        - containerPort: 80
...
deployment.extensions/nginx-deployment edited
```

这个 kubectl edit 指令，会帮你直接打开 nginx-deployment 的 API 对象。然后，你就可以修改这里的 Pod 模板部分了。比如，在这里，我将 nginx 镜像的版本升级到了 1.9.1。

### kubectl rollout status

实时查看 Deployment 对象的状态变化：

```shell
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

*在这个返回结果中，“2 out of 3 new replicas have been updated”意味着已经有 2 个 Pod 进入了 UP-TO-DATE 状态。*

### kubectl rollout history

使用 kubectl rollout history 命令，查看每次 Deployment 变更对应的版本

```shell
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```

通过 kubectl rollout history 指令，看到每个版本对应的 Deployment 的 API 对象的细节，具体命令如下所示：

```shell
$ kubectl rollout history deployment/nginx-deployment --revision=2
```

###kubectl rollout undo

kubectl rollout undo 命令，能把整个 Deployment 回滚到上一个版本

```shell
$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment
```

在 kubectl rollout undo 命令行最后，加上要回滚到的指定版本的版本号，就可以回滚到指定版本了

```shell
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment
```

### kubectl rollout pause

我们对 Deployment 进行的每一次更新操作，都会生成一个新的 ReplicaSet 对象，是不是有些多余，甚至浪费资源呢？

所以，Kubernetes 项目还提供了一个指令，使得我们对 Deployment 的多次更新操作，最后 只生成一个 ReplicaSet。

具体的做法是，在更新 Deployment 前，你要先执行一条 kubectl rollout pause 指令。它的用法如下所示：

```shell
$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused
```

这个 kubectl rollout pause 的作用，是让这个 Deployment 进入了一个“暂停”状态。

所以接下来，你就可以随意使用 kubectl edit 或者 kubectl set image 指令，修改这个 Deployment 的内容了。由于此时 Deployment 正处于“暂停”状态，所以我们对 Deployment 的所有修改，都不会触发新的“滚动更新”，也不会创建新的 ReplicaSet。

### kubectl rollout resume

而等到我们对 Deployment 修改操作都完成之后，只需要再执行一条 kubectl rollout resume 指令，就可以把这个 Deployment “恢复” 回来，如下所示：

```shell
$ kubectl rollout resume deploy/nginx-deployment
deployment.extensions/nginx-deployment resumed
```

### kubectl describe

使用 kubectl describe 命令，查看一个 API 对象的细节。

```
$ kubectl describe pod nginx-deployment-67594d6bf6-9gdvr
Name:               nginx-deployment-67594d6bf6-9gdvr
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               node-1/10.168.0.3
Start Time:         Thu, 16 Aug 2018 08:48:42 +0000
Labels:             app=nginx
                    pod-template-hash=2315082692
Annotations:        <none>
Status:             Running
IP:                 10.32.0.23
Controlled By:      ReplicaSet/nginx-deployment-67594d6bf6
...
Events:

  Type     Reason                  Age                From               Message

  ----     ------                  ----               ----               -------
  
  Normal   Scheduled               1m                 default-scheduler  Successfully assigned default/nginx-deployment-67594d6bf6-9gdvr to node-1
  Normal   Pulling                 25s                kubelet, node-1    pulling image "nginx:1.7.9"
  Normal   Pulled                  17s                kubelet, node-1    Successfully pulled image "nginx:1.7.9"
  Normal   Created                 17s                kubelet, node-1    Created container
  Normal   Started                 17s                kubelet, node-1    Started container

```

### kubectl replace

通过 kubectl replace 命令来升级服务。

```
$ kubectl replace -f nginx-deployment.yaml
```

### kubectl apply

**推荐使用 kubectl apply 命令，来统一进行 Kubernetes 对象的创建和更新操作**

```
$ kubectl apply -f nginx-deployment.yaml
```

### kubectl exec

使用 kubectl exec 指令，进入到这个 Pod 当中（即容器的 Namespace 中）

**如果一个pod容器中，有多个容器，需要使用-c选项指定容器**

```
$ kubectl exec -it nginx-deployment-5c678cfb6d-lg9lw -- /bin/bash
```

### kubectl delete

从 Kubernetes 集群中删除 deployment

```
$ kubectl delete -f nginx-deployment.yaml
```

对于无法删除的 pods，可以使用强制删除命令

```
kubectl delete pods xxxxxx --force --grace-period=0
```



### kubectl attach

使用 kubectl attach 命令，连接到 shell 容器的 tty 上

```
$ kubectl attach -it nginx -c shell
```

