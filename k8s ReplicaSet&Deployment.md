# ReplicaSet&Deployment 

## replica

创建pod的副本

不会以`kubectl delete pod podName`的方式而被删除。必须通过replicaset相关命令进行操作。

### 创建replicaset模板文件

- apiVersion: apps/v1
- kind: ReplicaSet
- metadata: 
- spec:
  - selector
    - matchLables: :wq必须和template中metadata里面的label一样
  - replicas:确定replica的个数。（删除pod后自动添加新的pod。当pod数为replicas指定个数时，不能创建新pod）
  - template：里面是pod的定义(metadata和spec的部分)



### example

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
spec:
  selector:
    matchLabels:
      env: production
  replicas: 3
  template:
    metadata:
      name: nginx-2
      labels:
        env: production
    spec:
      containers:
        - name: nginx
          image: nginx

```

replicaset的selector的matchLabels必须和template的labels下面的内容一致。和replicaset自身labels无关。

### 常用指令

- 创建replicaset

  `kubectl create -f replicaset  definition.yaml` 

- 查看当前replicaset的个数（注意不是pod的个数）

  `kubectl get replicaset`

- 查看当前replicaset的详细信息(replicaSetName指的是具体的replicaset的名字，可通过`kubectl get replicaset`获得)

  `kubectl describe replicaset replicaSetName`

- get pods 查看具体pod信息

- 查看创建replicaset所需要的属性信息(replicaSetName指的是具体的replicaset的名字)

  `kubectl explain replicaset replicaSetName`

- 删除replicaset 

  `kubectl delete replicaset replicasetName`

  - `--all`：删除所有的replicaset

    `kubectl delete replicaset --all`

  

- 编辑指定的replicaset：默认以vim打开并修改replicaset的配置 

  `kubectl edit replicaset replicasetName`
  
  然后删除原有的旧pods `kubectl delete pods --all`，replicaset会自动创建新pods

- `scaled`命令修改当前replicaset的replica的个数

  `kubectl scale replicaset replicaSetName --replicas=2`





## Deployment

目的：为Pods和ReplicaSet提供声明式的更新

### 创建deployment模板

和replicaset模板很像，区别是kind为Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    tier: frontend
    app: nginx
spec:
  selector:
    matchLabels: #必须和template.metadata.lables一致
      app: myapp 
  replicas: 3
  template:
    metadata:
      name: nginx-2 #the name of pod
      labels:
        app: myapp
    spec:
      containers: #以list形式来定义pod里面的container
        - name: nginx #the name of container in the pod
          image: nginx
```



### command

- create deployment

  `kubectl create -f deployment.yaml`

- 查看当前deployment

  `kubectl get deployment`

- 查看某一个deployment的具体信息

  `kubectl describe deployment deploymentName`



### Update&Rollout 

#### Rollout 

追踪deployment的变化并能是我们能够回归到前一个版本

- 查看rollout的状态

  `kubectl rollout status deployment/myapp-deployment` (myapp-deployment是deployment的name)

- show reversion and history of our deployment

  `kubectl rollout history deployment/myapp-deployment`

#### Deployment策略：

#### Recreate 和 RollingUpdate (StrategyType)的区别

- Recreate策略：先scale down replicaset到0，然后再重新scale up replicaset到预定个数(not default)
- RollingUpdate(滚动更新)策略： 每次同时地scale up replicaset 和scale down replicaset(defalult)



#### Rollback (回退)

在新版本的pod里发现了问题，要返回到上一个版本

Run the `kubectl rollout undo `command followed by `the name of the deployment`

`kubectl rollout undo deployment/myapp-deployment`

### Summarize Commands

- Create deployment

`kubectl create -f deployment-definition.yml`

- Get 获取deployment的信息

  `kubectl get deployments`

- Update 单独升级image（会导致deployment的定义文件中有不同的image配置）(推荐kubectl edit deployment myapp-deployment)

  `kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1`

- 回滚的状态 `rollout status`

  `kubectl rollout status deployment/myapp-deployment`

- 回滚的history

  `kubectl rollout history deployment/myapp-deployment`

- Rollback

  `kubectl rollout undo deployment/myapp-deployment`



#### example

1,先create deployment

`kubectl create -f deployment-defination.yaml --record` 

2,获取deployment的名字

`kubectl get deployment`

```shell
root@master01 work]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
myapp-deployment   3/3     3            3           12s

```

3,使用 `kubectl edit deployment myapp-deployment`修改image版本为 `nginx:1.18`

`kubectl edit deployment  myapp-deployment --record`

`--record`：的意义是记录操作的历史记录

4,使用`kubectl rollout status deployment myapp-deployment`查看rollout状态

```shell
[root@master01 work]# kubectl rollout status deployment myapp-deployment
Waiting for deployment "myapp-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "myapp-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "myapp-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "myapp-deployment" successfully rolled out

```

5,使用`kubectl rollout history deployment myapp-deployment`

```shell
[root@master01 work]# kubectl rollout history deployment myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
1         kubectl create --filename=deployment-defination.yaml --record=true
2         kubectl edit deployment/myapp-deployment --record=true

```

6,使用`kubectl set image deployment deploymentName containerName=image`直接设置镜像

```shell
[root@master01 work]# kubectl set image deployment myapp-deployment nginx=nginx:1.18-perl --record
deployment.apps/myapp-deployment image updated

```

查看rollout history

```
[root@master01 work]# kubectl rollout history deployment myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
1         kubectl create --filename=deployment-defination.yaml --record=true
2         kubectl edit deployment/myapp-deployment --record=true
3         kubectl set image deployment myapp-deployment nginx=nginx:1.18-perl --record=true

```

当前在REVERSION3的位置

7，使用`kubectl rollout undo deployment myapp-deployment`  进行回退到上个版本,并查看history

```shell
[root@master01 work]# kubectl rollout undo deployment myapp-deployment 
deployment.apps/myapp-deployment rolled back
[root@master01 work]# kubectl rollout history deployment myapp-deployment
deployment.apps/myapp-deployment 
REVISION  CHANGE-CAUSE
1         kubectl create --filename=deployment-defination.yaml --record=true
3         kubectl set image deployment myapp-deployment nginx=nginx:1.18-perl --record=true
4         kubectl edit deployment/myapp-deployment --record=true

```

结果表明：`rollout undo`（把当前REVERSION3要回滚到的上一个位置）REVERSION2变成了最新的REVERSION4



8,修改image为一个不存在的version

使用`kubectl rollout status deployment myapp-deployment`查看：rollout卡住了

```
root@master01 work]# kubectl rollout status deployment myapp-deployment
Waiting for deployment "myapp-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
```

```
[root@master01 work]# kubectl get pods
NAME                                READY   STATUS             RESTARTS   AGE
myapp-deployment-7c8cf9794b-f5qn7   0/1     ImagePullBackOff   0          80s
myapp-deployment-ddbb47dd8-68l2m    1/1     Running            0          8m13s
myapp-deployment-ddbb47dd8-gv4k9    1/1     Running            0          8m22s
myapp-deployment-ddbb47dd8-rg98d    1/1     Running            0          8m17s
```

旧pod还在Running并没有被删除，因此application并没有被影响。

**rollout update是安全的**,rollout必须在新pod充分成功创建后，才会删除旧pod



