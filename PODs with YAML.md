## PODs with YAML

### YAML in Kubernetes

- k8s definition file always contains 4 top level fields.(apiVersion,kind,metadata,spec)

`pod-definition.yml`

```yaml
apiVersion:
kind:
metadata:

spec:
```

- apiVersion:

  | Kind       | Version |
  | ---------- | ------- |
  | POD        | v1      |
  | Service    | v1      |
  | ReplicaSet | apps/v1 |
  | Deployment | apps/v1 |

  deploy POD时使用v1

- kind：

  - 类型：Pod，Service，ReplicaSet，Deployment

- metadata：

  The metadata is data about the object like its **name，labels** etc.

  `name`: 用来指定pod的名字

  ```yaml
  metadata:
    name: myapp-pod
    labels:
      app: myapp
      type: front-end
  ```

  metadata里面是字典类型。

- spec:

  spec里面contianers下面是一个list,list里面有不同字段（是字典）

  ```
  spec:
    containers:
      - name: nginx-container
        image: nginx
  ```

### 启动pods

- `kubectl apply -f pod-definition.yml`

- `kubectl get pods`
- `kubectl describe pod myapp-pod`



### Example

- `pod.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: nginx
      # tier:层
      tier: frontend 
  spec:
    containers:
      - name: nginx
        image: nginx
      - name: busybox
        image: busybox
  ```

  

### 

- `pod-definition.yml` 

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: postgres
    labels:
      tier: db-tier
  spec:
    containers:
      - name: postgres
        image: postgres
        env:
          - name: POSTGRES_PASSWORD
            value: mysecretpassword  
  ```

### 启动pod

`kubectl apply -f pod.yaml`

### 使用vscode编辑yaml

- 在settting.json中加入

  ```json
  {
  	"yaml.schemas": {
          "kubernetes": "*.yaml"
      }
  }
  ```

### 删除pod

`kubectl delete pod myapp-pod`

### 编辑pod

`kubectl edit pod myapp-pod`

