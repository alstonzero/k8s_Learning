### helm search hub

- helm search hub mysql

添加repository

- helm add repo $URL

- ```
  helm repo add stable https://charts.helm.sh/stable
  ```

- helm repo list 

```
# helm repo list
NAME  	URL                          
stable	https://charts.helm.sh/stable
```

- helm search repo stable  查找stable repository当中可用的所有charts
- helm search repo stable/mysql：查找stable repository当中mysql chart

```
# helm search repo stable/mysql
NAME            	CHART VERSION	APP VERSION	DESCRIPTION                                       
stable/mysql    	1.6.9        	5.7.30     	DEPRECATED - Fast, reliable, scalable, and easy...
stable/mysqldump	2.6.2        	2.4.1      	DEPRECATED! - A Helm chart to help backup MySQL...

```

- helm install stable/mysql --generate-name 

install这个chart，并让helm给这个特定的install(即：release)生成一个name，(要么自己指定名称)

```
root@homegate:~# helm install stable/mysql --generate-name
WARNING: This chart is deprecated
NAME: mysql-1636444722
LAST DEPLOYED: Tue Nov  9 16:58:45 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mysql-1636444722.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql-1636444722 -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h mysql-1636444722 -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/mysql-1636444722 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}

```

这里还说明了获取root passwd的方法，和连接mysql的方法等

- helm install stable/airflow

```
Error: INSTALLATION FAILED: must either provide a name or specify --generate-name
```

- 这里给该release指定myairflow这个名称

```bash
#helm install myairflow stable/airflow
WARNING: This chart is deprecated
W1109 17:15:03.734534 3671278 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W1109 17:15:03.823059 3671278 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
NAME: myairflow
LAST DEPLOYED: Tue Nov  9 17:15:03 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
!WARNING! !WARNING! !WARNING! !WARNING! !WARNING!

This chart has MOVED to a new repository!

https://github.com/airflow-helm/charts/tree/main/charts/airflow

!WARNING! !WARNING! !WARNING! !WARNING! !WARNING!

--------------------

Congratulations. You have just deployed Apache Airflow!

1. Get the Airflow Service URL by running these commands:
   export POD_NAME=$(kubectl get pods --namespace default -l "component=web,app=airflow" -o jsonpath="{.items[0].metadata.name}")
   echo http://127.0.0.1:8080
   kubectl port-forward --namespace default $POD_NAME 8080:8080

2. Open Airflow in your web browser

```



## chart

- `helm create mychart`

  在当前目录下，将创建一个名为 mychart 的chart，将被放在一个名为 mychart 的文件夹中

```
# tree
.
└── mychart
    ├── charts
    ├── Chart.yaml
    ├── templates
    │   ├── deployment.yaml
    │   ├── _helpers.tpl
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── NOTES.txt
    │   ├── serviceaccount.yaml
    │   ├── service.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml

```

- values.yaml为templates下面的template文件提供value

```
mychart/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  values.yaml         # The default configuration values for this chart
  values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file
  charts/             # A directory containing any charts upon which this chart depends.
  crds/               # Custom Resource Definitions
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

- Chart.yaml：包含关于chart information的yaml文件
- values.yaml：默认配置的values



# chart

## template

在 mychart的template路径下面，重新创建一个configMap

`configmap.yaml`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Sample Config Map"
```

创建mychart的release，命名为helm-demo-configmap

`helm install helm-demo-configmap /path/to/mychart`

<details>

```bash
#kubectl describe cm mychart-configmap 
Name:         mychart-configmap
Namespace:    default
Labels:       app.kubernetes.io/managed-by=Helm
Annotations:  meta.helm.sh/release-name: helm-demo-configmap
              meta.helm.sh/release-namespace: default

Data
====
myvalue:
----
Sample Config Map
Events:  <none>

```

</details>

成功部署了configmap

- 删除该release

`helm uninstall helm-demo-configmap`



### value

`vim values.yaml`

添加下面内容

```
costCode: CC98112
```

`vim ./template/configmap.yaml`

创建名为costCode的key，value使用`values.yaml`中定义的costCode的value，`.Release.Name`是内置变量。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap 
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}

```

用`--dry-run`查看value是否成功应用于configMap

`helm install firstdryrun ./mychart --debug --dry-run `

<details>

```bash
root@homegate:~/helm-learning# helm install myfirst-dryrun mychart/ --debug --dry-run
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: myfirst-dryrun
LAST DEPLOYED: Wed Nov 10 11:00:00 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
costCode: CC98112

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myfirst-dryrun-configmap
data:
  myvalue: "Sample Config Map"
  costCode: CC98112

```

</details>

在这种情况下，我们对值进行了硬编码。并且在 YAML 文件中替换了相同的值。

接下来，正式deploy该chart

`helm install myfirstvalue mychart`

- 获取deploy的manifest——`helm get manifest ${RELEASE_NAME}`

```bash
# helm get manifest firstvalue
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: firstvalue-configmap
data:
  myvalue: "Sample Config Map"
  costCode: CC98112

```

### value-set

从command line设置value，拥有更高的优先级，将会替换`values.yaml`中定义的value

`helm install --dry-run --debug --set costCode=CC0000 valueseteg ./mychart`

<details>

```bash
# helm install valueseteg mychart --set costCode=CC000000 --debug --dry-run
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: valueseteg
LAST DEPLOYED: Wed Nov 10 11:36:26 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
costCode: CC000000

COMPUTED VALUES:
costCode: CC000000

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: valueseteg-configmap
data:
  myvalue: "Sample Config Map"
  costCode: CC000000

```

</details>

### Template-function

作为`values.yaml` 文件或`template yaml` 文件的一部分我们可以放入函数来计算value。

参考文档：

http://masterminds.github.io/sprig/strings.html

`vim values.yaml`

```
costCode: CC98112
projectCode: aaazzz
infra:
  zone: a,b,c
  region: us-east-1
```

`vim configmap.yaml`

- 在引用的value之前添加function

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode }}
```

- quote：wrap a string in double quotes
- upper：大写输出

`helm install valuefuntion mychart/ --dry-run --debug`

<details>

```bash
root@homegate:~/helm-learning# helm install valuefuntion mychart/ --dry-run --debug
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: valuefuntion
LAST DEPLOYED: Wed Nov 10 11:48:44 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: valuefuntion-configmap
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ

```

</details>

### pipeline

- 和Linux的pipline用法相同

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Sample Config Map"
  ProjectCode: {{ upper .Values.projectCode  }}
  pipeline: {{ .Values.projectCode | upper | quote }}
  now: {{ now | date "2006-01-02" | quote }}
  contant: {{ .Values.contact | default "1-800-123-0000" | quote}}
```

- default的用法，如果在values.yaml中没有定义contant的值，则使用default中定义的值。

`helm install --debug --dry-run pipeline-test mychart/`

<details>

```
root@homegate:~/helm-learning# helm install --debug --dry-run pipeline-test mychart/
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: pipeline-test
LAST DEPLOYED: Thu Nov 11 13:51:32 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pipeline-test-configmap
data:
  myvalue: "Sample Config Map"
  ProjectCode: AAAZZZ
  pipeline: "AAAZZZ"
  now: "2021-11-11"
  contant: "1-800-123-0000"

```

</details>

### flow control

```
{{ if PIPELINE }}
  # DO something
{{ else if OTHER PIPELINE }}
  # DO something else
{{ else }}
  # Default case
{{ end }}
```

- 添加一个if

`vim template/configmap.yaml`

如果满足 `Values.infra.region`等于 us-east-1则输出 true

注意：ha是key，必须和其他key对齐

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  {{ if eq .Values.infra.region "us-east-1" }} 
  ha: true 
  {{ end }}

```

`helm install --dry-run --debug controlif mychart/`

<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug controlif mychart/
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: controlif
LAST DEPLOYED: Thu Nov 11 16:53:28 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: controlif-configmap
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  ha: true

```

</details>

### with

Modifying scope using 'with'

在scope内，直接引用

```
{{ with PIPELINE }}
 # restricted scope
{{ end }}
```

`vim values.yaml`

```
costCode: CC98112
projectCode: aazzxxyy
infra:
  zone: a,b,c
  region: us-east-1
tags:
  machine: frontdrive
  rack: 4c
  drive: ssd
  vcard: 8g
```

`vim template/configmap.yaml`

`helm install --dry-run --debug withscope ./mychart`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
  {{- with .Values.tags }}
    first: {{ .machine }}
    second: {{ .rack }}
    third: {{ .drive }}
  {{- end }}
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  {{- with .Values.tags }}
  Machine Type: {{ .machine | default "NA" | quote }}
  Rack ID: {{ .rack | quote }}
  Storage Type: {{ .drive | upper | quote }}
  Video Card: {{ .vcard | quote }}
  {{- end }}
```



### range

Looping using 'range'

```
Lang Used: |-
  {{- range .Values.LangUsed }}
  - {{ . | title | quote }} # range的结果是".",转成title，并添加引号
  {{- end}} #"-" 消除空行
```

`-`：remove leading or trailing whitespace

`vim mychart/values.yaml`

```
costCode: CC98112
projectCode: aaazzz
infra:
  zone: a,b,c
  region: us-east-1
LangUsed:
  - Python
  - Ruby
  - Java
  - Scala

```

`vim mychart/templates/configmap.yaml`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  Lang Used: -|
  {{- range .Values.LangUsed }} 
   - {{ . | title | quote }}
  {{- end }}

```

- `.Values.LangUsed`获取值的集合，并用range迭代。

`helm install --dry-run --debug rangetest mychart/`

<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug rangetest mychart/
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: rangetest
LAST DEPLOYED: Thu Nov 11 18:36:21 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rangetest-configmap
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  Lang Used: -|
   - "Python"
   - "Ruby"
   - "Java"
   - "Scala"

```

</details>

### variables

```
$name.Variables are assigned with a special assignment operator :=
Eg: {{- $relname := .Release.Name -}}
```

把`.Release.Name`的值赋给`relname`变量中

在开头和结尾都添加了一个破折号，因为它不会print任何东西，如果不提供这个特定的破折号，它将添加一个new line character。

`vim mycart/templates/configmap.yaml`

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  Lang Used: -|
  {{- range $index , $topping := .Values.LangUsed }}
   - {{ $index }} {{ $topping }}
  {{- end }}
```

用 range 分别把key value赋值给index和topping这两个变量

在metadata中添加如下labels

```
labels:
  helm.sh/chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
  app.kubernetes.io/instance: "{{ $.Release.Name }}"
  app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
  app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
```

vim configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
   helm.sh/chart: "{{ $.Chart.Name }}-{{ $.Chart.Version }}"
   app.kubernetes.io/instance: "{{ $.Release.Name }}"
   app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
   app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  Lang Used: -|
  {{- range $index , $topping := .Values.LangUsed }}
   - {{ $index }} {{ $topping }}
  {{- end }}

```

- 使用 global variable

` helm install --dry-run --debug variabletest mychart/`

<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug variabletest mychart/
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: variabletest
LAST DEPLOYED: Thu Nov 11 20:41:29 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: variabletest-configmap
  labels:
   helm.sh/chart: "mychart-0.1.0"
   app.kubernetes.io/instance: "variabletest"
   app.kubernetes.io/version: "1.16.0"
   app.kubernetes.io/managed-by: "Helm"
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  Lang Used: -|
   - 0 Python
   - 1 Ruby
   - 2 Java
   - 3 Scala

```

</details>

### Template include another template

#### Named Templates 命名模板

`kubernetes manifests`文件保存在`templates/`路径下，除了`Notes.txt`

在一个template中包含另外一个template

- 定义一个名为mychart.systemlables的template
  - 定义template要注意缩进

```
Eg：
{{- define "mychart.systemlables" }}
   labels:
     drive: ssd
     machine: frontdrive
     rack: 4c
     vcard: 8g
 {{- end }}
```

- `vim mychart/templates/configmap.yaml`
  - 引用tempalte则不需要注意缩进

```yaml
{{- define "mychart.systemlables" }}
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
{{- end}}

apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.systemlables" }}
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  Lang Used: -|
  {{- range $index , $topping := .Values.LangUsed }}
   - {{ $index }} {{ $topping }}
  {{- end }}

```



<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug templatedemo ./mychart
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: templatedemo
LAST DEPLOYED: Thu Nov 11 21:12:08 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: templatedemo-configmap
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  Lang Used: -|
   - 0 Python
   - 1 Ruby
   - 2 Java
   - 3 Scala

```

</details>

### Template Include ——using Scope

- 从另外的文件中引入template，创建一个带有下划线前缀的`_helper`文件。所有的helper文件都必须带有前缀下划线

`vim mychart/templates/_helpers.tpl`

```
{{- define "mychart.systemlables" }}
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
    app.kubernetes.io/instance: "{{ $.Release.Name }}"
    app.kubernetes.io/version: "{{ $.Chart.AppVersion }}"
    app.kubernetes.io/managed-by: "{{ $.Release.Service }}"
{{- end }}

```

修改 comfigmap.yaml

- `{{- template "mychart.systemlables" $ }}`中 添加 `$`或 `.`，才能反应全局变量
  - And this particular template will have the builin objects so that it can pass and the values would get printed. 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.systemlables" $ }}
data:
  myvalue: "Sample Config Map"
  costCode: {{ .Values.costCode }}
  Zone: {{ quote .Values.infra.zone }}
  Region: {{ quote .Values.infra.region }}
  ProjectCode: {{ upper .Values.projectCode  }}
  Lang Used: -| 
  {{- range $index , $topping := .Values.LangUsed }}
   - {{ $index }} {{ $topping }}
  {{- end }}

```

- dry-run执行`helm install --dry-run --debug templatedemo ./mychart`

<details>

```
# helm install --dry-run --debug templatedemo ./mychart
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: templatedemo
LAST DEPLOYED: Thu Nov 11 22:23:41 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: templatedemo-configmap
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
    app.kubernetes.io/instance: ""
    app.kubernetes.io/version: ""
    app.kubernetes.io/managed-by: ""
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  Lang Used: -|
   - 0 Python
   - 1 Ruby
   - 2 Java
   - 3 Scala

```

</details>

### NOTES.txt

为chart添加注释

`NOTES.txt` 将包含文件和其他帮助程序template的注释。

`vim mychart/templates/NOTES.txt` 注意是全部大写

```
Thank you for support {{ .Chart.Name }}.
 
Your release is named {{ .Release.Name }}.
 
To learn more about the release, try:
 
 $ helm status {{ .Release.Name }}
 $ helm get all {{ .Release.Name }}
 $ helm uninstall {{ .Release.Name }}
```

`helm install notesdemo ./mychart`

NOTES:中输出

<details>

```
root@homegate:~/helm-learning# helm install notesdemo ./mychart --debug --dry-run
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: notesdemo
LAST DEPLOYED: Fri Nov 12 10:59:44 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: notesdemo-configmap
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
    app.kubernetes.io/instance: ""
    app.kubernetes.io/version: ""
    app.kubernetes.io/managed-by: ""
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  Lang Used: -|
   - 0 Python
   - 1 Ruby
   - 2 Java
   - 3 Scala

NOTES:
Thank you for support mychart.
 
Your release is named notesdemo.
 
To learn more about the release, try:
 
 $ helm status notesdemo
 $ helm get all notesdemo
 $ helm uninstall notesdemo
```

</details>

### sub chart

chart是一个单独的实体。chart收集template和value，并打包在一起，在repository内共享。也可以发送到external world

在mycart下面创建sub chart

```
cd mychart/charts
helm create mysubchart
rm -rf mysubchart/templates/*.*
```

在subchart的values.yaml中添加下面内容：

```
dbhostname: mysqlnode
```

定义subchart的configMap

`vim mychart/charts/mysubchart/templates/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-innerconfig
data:
  dbhost: {{ .Values.dbhostname }}
```

只install subchart

`helm install --dry-run --debug mysubchart mychart/charts/mysubchart`

<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug mysubchart mychart/charts/mysubchart
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart/charts/mysubchart

NAME: mysubchart
LAST DEPLOYED: Fri Nov 12 14:18:16 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
dbhostname: mysqlnode

HOOKS:
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysubchart-innerconfig
data:
  dbhost: mysqlnode

```
</details>

- 重载来自parent的values

在parent——mychart的values.yaml中重新定义dbhostname的value

```
mysubchart:
  dbhostname: prodmyqlnode
```

安装parent chart

`helm install --dry-run --debug subchartoverride mychart`

可以看到

<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug subchartoverride mychart
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: subchartoverride
LAST DEPLOYED: Fri Nov 12 14:22:13 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
infra:
  region: us-east-1
  zone: a,b,c
mysubchart:
  dbhostname: prodmyqlnode
  global: {}
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: subchartoverride-innerconfig
data:
  dbhost: prodmyqlnode
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: subchartoverride-configmap
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
    app.kubernetes.io/instance: "subchartoverride"
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: "Helm"
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  Lang Used: -|
   - 0 Python
   - 1 Ruby
   - 2 Java
   - 3 Scala

NOTES:
Thank you for support mychart.
 
Your release is named subchartoverride.
 
To learn more about the release, try:
 
 $ helm status subchartoverride
 $ helm get all subchartoverride
 $ helm uninstall subchartoverride

```

</details>

dbhost的value已经被parent chart(mychart)中定义的value所代替

```
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: subchartoverride-innerconfig
data:
  dbhost: prodmyqlnode
```

### Template SubChart Global

在parent chart中定义global variable

`vim mychart/values.yaml`

```
global:
  orgdomain: com.muthu4all
```

在subchart和mychart(parent chart)的template中添加如下内容

```
orgdomain: {{ .Values.global.orgdomain }}
```

- `helm install --dry-run --debug subchartoverride mychart`

输出结果显示，

```
global:
  orgdomain: com.muthu4all
```

被自动添加到了subchart的values中，因此在subchart的manifest中也被print出来

`helm install --dry-run --debug subchartoverride mychart`

<details>

```
root@homegate:~/helm-learning# helm install --dry-run --debug subchartoverride mychart
install.go:178: [debug] Original chart version: ""
install.go:199: [debug] CHART PATH: /root/helm-learning/mychart

NAME: subchartoverride
LAST DEPLOYED: Fri Nov 12 15:57:55 2021
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
LangUsed:
- Python
- Ruby
- Java
- Scala
costCode: CC98112
global:
  orgdomain: com.muthu4all
infra:
  region: us-east-1
  zone: a,b,c
mysubchart:
  dbhostname: prodmyqlnode
  global:
    orgdomain: com.muthu4all
projectCode: aaazzz

HOOKS:
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: subchartoverride-innerconfig
data:
  dbhost: prodmyqlnode
  orgdomain: com.muthu4all
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: subchartoverride-configmap
  labels:
    drive: ssd
    machine: frontdrive
    rack: 4c
    vcard: 8g
    app.kubernetes.io/instance: "subchartoverride"
    app.kubernetes.io/version: "1.16.0"
    app.kubernetes.io/managed-by: "Helm"
data:
  myvalue: "Sample Config Map"
  costCode: CC98112
  Zone: "a,b,c"
  Region: "us-east-1"
  ProjectCode: AAAZZZ
  orgdomain: com.muthu4all

NOTES:
Thank you for support mychart.
 
Your release is named subchartoverride.
 
To learn more about the release, try:
 
 $ helm status subchartoverride
 $ helm get all subchartoverride
 $ helm uninstall subchartoverride

```

</details>
