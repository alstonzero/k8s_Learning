## cloud devops kubernetes——helm

## Building Manifest with helm

### helm chart的结构

```
demo
├── Chart.yaml
├── production-values.yaml
├── staging-values.yaml
├── templates
│ ├── deployment.yaml
│ └── service.yaml
└── values.yaml
```

### Chart.yaml

指定了chart name和version

```yaml
name: demo
sources:
  - https://github.com/cloudnativedevops/demo
version: 1.0.1
```

### values.yaml

包含user-modifiable设置。

```yaml
environment: development
container:
  name: demo
  port: 8888
  image: cloudnatived/demo
  tag: hello
replicas: 1
```

和Kubernetes YAML manifest不同点：`values.yaml`是完全free-from的YAML，没有预先定义的schema：由user去选择要定义的variables。

### Helm Templates

位于templates subdirectory之下。templates中包含占位符，Helm会用`values.yaml`中真实的值来进行替换。

`deployment.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: {{ .Values.container.name }}-{{ .Values.environment }}
sepc:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.container.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.container.name }}
        environment: {{ .Values.environment }}
    spec:
      containers:
        - name: {{ .Values.container.name }}
          image: {{ .Values.container.image }}:{{ .Values.container.tag }}
          ports:
            - containerPort: {{ .Values.container.port }}
          env:
            - name: ENVIRONMENT
              value: {{ .Values.environment }}
```

template使用了Go template语法。花括号`{{ }}`表示 Helm 应该替换的地方变量的值。

### Interpolating Variables（替换变量）

```
...
metadata:
  name: {{ .Values.container.name }}-{{ .Values.environment }}
```

整个文本部分，包括花括号`{{ }}`被替换为取自`values.yaml`中`container.name`和`environment`的值。生成的结果如下：

```
metadata:
  name: demo-development
```

在`service.yaml`template中container.name不止被引用一次。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.container.name }}-service-{{ .Values.environment }}
  labels:
    app: {{ .Values.container.name }}
spec:
  ports:
  - port: {{ .Values.container.port }}
    protocol: TCP
    targetPort: {{ .Values.container.port }}
  selector:
    app: {{ .Values.container.name }}
  type: ClusterIP
```

如果想改变container name的值只需在`values.yaml`中进行编辑，并reinstall该chart即可。该change将会应用到所有的templates。这大大提高了效率。

Go template format同时也支持 循环，表达式，条件，甚至调用函数。Helm chart能使用这些特性生成更加



### Specifying Dependencies（指定依赖）

指定依赖chart。例如，

`requirement.yaml`

```
dependencies:
  - name: redis
    version: 1.2.3
  - name: nginx
    version: 3.2.1
```



## Deploying Helm Charts

### Creating a environment variable（创建环境变量）

假设想要在staging环境中，deploy a version of application，application是根据一个名为ENVIRONMENT的环境变量的值改变其行为。如何创建environment变量？

在`deployment.yaml`template中，通过下面的内容将环境变量提供到container中

```
...
env:
 - name: ENVIRONMENT
   value: {{ .Values.environment }}
```

而environment的值则来自于`values.yaml`

```
environment: development
...
```

因此，使用默认的values来install chart则设置container的ENVIRONMENT变量为deployment。

假如想要将其变更为staging。需要修改values.yaml文件，或者在`k8s/demo/staing-values.yaml`路径下另外创建一个名为`staging-values.yaml`的文件，写入下面内容。

```
environment: staging
```

### Specifying Values in a Helm Release

在helm install安装chart时，使用`--values`flag指定values文件，此时container的ENVIRONMENT变量将会被设置为staging来代替deployment。即使用来自`staging-values.yaml`中的值来override来自default values file的值。

```
helm install --name demo-staging --values=./k8s/demo/staging-values.yaml
./k8s/demo ...
```

使用`helm insepect values <chart_name>`可用用来检查该chart中可供你来设置的value。

- 例如：`helm inspect values bitnami/postgresql`

### Updating an APP with Helm

`helm update`用来改变目前在application中已经运行的values的值。

假设你想要改变demo application的replica的数量。可以看到在values.yaml文件中这个value被定义为1

```
replicas: 1
```



```bash
helm upgrade demo-staging --values=./k8s/demo/staging-values.yaml ./k8s/demo
Release "demo-staging" has been upgraded. Happy Helming!
```



### 使用Sops管理Helm Chart Secrets

https://github.com/cloudnativedevops/demo/tree/main/hello-sops

```
cd hello-sops
tree
.
├── k8s
│ └── demo
│ ├── Chart.yaml
│ ├── production-secrets.yaml
│ ├── production-values.yaml
│ ├── staging-secrets.yaml
│ ├── staging-values.yaml
│ ├── templates
│ │ ├── deployment.yaml
│ │ └── secrets.yaml
│ └── values.yaml
└── temp.yaml
```

#### 安装helm secret plugin

```bash
# helm plugin install https://github.com/futuresimple/helm-secrets
# helm plugin list
NAME   	VERSION	DESCRIPTION                                                                  
secrets	2.0.2  	This plugin provides secrets values encryption for Helm charts secure storing

```

#### SOPS命令简介

helm-secrets plugin本身并没有任何encrypt与decrypt的能力，也没有提供任何只针对于 YAML 文件的 Key 值进行加密和解密的工作方式，而它所有的工作，都是通过调用 `sops` 命令来帮助它完成的。

SOPS 是由 Mozilla 开发的一款开源的文本编辑工具，它支持对 YAML, JSON, ENV, INI 和 BINARY 文本格式的文件进行编辑，并利用 AWS KMS, GCP KMS, Azure Key Vault 或 PGP 等加密方式对编辑的文件进行加密和解密。

 SOPS 命令是确保 helm-secrets 插件可以正常工作的必不可少的依赖，因此，我们必须要确保系统中正确安装了 SOPS 命令。

幸运的是，helm-secrets 插件会自动检测并安装 sops 命令到我们的系统中，因此在插件安装完成后，你还应该可以运行 sops 相关命令，如获取 sops 的版本信息：(截止到2021/11/23)

```bash
# sops -v
sops 3.7.1 (latest)
```

当然也可以手动安装sops命令，从这里下载二进制文件 https://github.com/mozilla/sops/releases。

#### PGP 与 GPG

helm-secrets 插件是通过调用 SOPS 命令来对 Values 文件进行加密和解密的，而 SOPS 本身又支持多种加密方式，如 AWS 云的 KMS，Google 云的 MKS，微软 Azure 云的 Key Vault，以及 PGP 等加密方式。本文将着重介绍如何利用 PGP 对我们的 Values 文件进行加密和解密。

在了解 PGP 加密之前，我们首先要区分好两个名词概念：PGP 与 GPG。

Pretty Good Privacy，也就是我们平时所说的 PGP，实际上通常指的是 OpenPGP。OpenPGP 只是一系列协议或标准，而并非某个特定的工具或命令，它规定了如何使用特定的方式和算法对文件或内容进行加密和解密，任何实现了 OpenPGP 协议的工具都拥有加密和解密的能力。而本文接下来所要使用的 `gpg` 命令，正是这样一个实现了 OpenPGP 协议的工具，或者说命令，利用 `gpg` 命令，我们可以实现 PGP 加密方式。为了避免混淆，后文将一律使用 GPG 代替。

#### 安装 GPG

目前绝大部分 Linux 发行版本都默认安装了 GPG

```bash
# gpg --version
gpg (GnuPG) 2.2.19
libgcrypt 1.8.5
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <https://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Home: /root/.gnupg
Supported algorithms:
Pubkey: RSA, ELG, DSA, ECDH, ECDSA, EDDSA
Cipher: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256, TWOFISH,
        CAMELLIA128, CAMELLIA192, CAMELLIA256
Hash: SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
Compression: Uncompressed, ZIP, ZLIB, BZIP2

```

如果没有安装则以下命令安装

```
# Ubuntu，Debian 用户
$ sudo apt install gnupg

# CentOS，Fedora，RHEL 用户
$ sudo yum install gnupg
```

#### 生成 GPG 密钥对

GPG 同样也使用了public key和secret key的概念实现了非对称加密算法。

<details>
    简单来说：公钥用于加密，拥有公钥的人可以且仅仅可以进行加密操作，它可以分发给任何组织或个人；而私钥则用于解密，且仅能用于解密那些由该私钥与之配对的公钥加密的信息，任何拥有私钥的人都可以进行解密操作，因此，确保私钥不被泄漏对安全性起着至关重要的作用。
</details>

使用`gpg --full-generate-key`命令来交互式地生成密钥对。之所以说交互式，是因为在生成密钥对时会要求用户交互式的输入一些相关信息，如如用户名、邮箱等等。

也可以使用下面命令为我们生成一对长度为 4096 且永不过期的 RSA 密钥对，注意，在执行命令之前，请注意修改下方 name、email 等字段信息为你自己相应的信息：

```bash
$ gpg --batch --generate-key <<EOF
%echo Generating a basic OpenPGP key for HELM Secret
Key-Type: RSA
Key-Length: 4096
Subkey-Type: RSA
Subkey-Length: 4096
Name-Real: HELM Secret
Name-Comment: Used for HELM Secret Plugin
Name-Email: helm-secret@email.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
%commit
%echo done
EOF
```

当生成 GPG 密钥对以后，我们就可通过`gpg` 的 `--list-keys` 和 `--list-secret-keys` 命令分别列出当前系统中的公钥和私钥信息了：

```bash
# gpg --list-keys
/root/.gnupg/pubring.kbx
------------------------
pub   rsa4096 2021-11-23 [SC]
      179D7100D05CD229405564A17694B92348945E38
uid           [ultimate] Helm Secret (Used for Helm Secret Plugin) <helm-secret@helm.com>
sub   rsa4096 2021-11-23 [E]

# gpg --list-secret-keys
/root/.gnupg/pubring.kbx
------------------------
sec   rsa4096 2021-11-23 [SC]
      179D7100D05CD229405564A17694B92348945E38
uid           [ultimate] Helm Secret (Used for Helm Secret Plugin) <helm-secret@helm.com>
ssb   rsa4096 2021-11-23 [E]
```

其中 `179D7100D05CD229405564A17694B92348945E38`  或者后十六位`7694B92348945E3`为公钥的 ID

也可以通过传递密钥的用户名、邮箱或 ID 来查看某个特定的密钥信息：

```bash
$ gpg --list-key "HELM Secret"
# 或
$ gpg --list-key "helm-secret@email.com"
# 或
$ gpg --list-key 7694B92348945E38
```

#### 使用sops进行加密

```bash
echo "password: secret123" > test.yaml
cat test.yaml
password: secret123
```

- 进行encrypt

`sops --encrypt --in-place --pgp 179D7100D05CD229405564A17694B92348945E38 test.yaml  cat test.yaml `

- `--encrypt, -e` 参数告诉 sops 进行加密操作；
- `--in-place, -i` 参数指定将加密后的内容直接替换掉原文件内容，若不指定该参数，则加密后的内容将会被输出到屏幕上；
- `--pgp, -p` 参数指定我们加密时所要使用的 PGP 公钥 ID。

<details>

```bash
# cat test.yaml 
password: ENC[AES256_GCM,data:9P4QUlNXJfFN,iv:sQ4+TRxiFM8gTZXhizuZ090Jp29J2HwRvtZeT316gS4=,tag:6b/h5kfC4nbgsnpNjVJNow==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2021-11-23T07:36:06Z"
    mac: ENC[AES256_GCM,data:JUM+D5fuKvbKu22jf9jg5A5TVpz7XdWYraZcJV/cq7pXAwmtotbNe53vVTKIyZpqRVMirKzYeb+1tCT4k/E2jE1dhxhGNHwMjVZGYK4W7+K2tFSgR3T9jvmEaxNehPdq14qbr/3nJl/wBthiqMg1SFbvW/KP1vGw6DrTZp8cq8I=,iv:0HYuMCyxrclZ9ULeh4ab8teWN9SigzqmvcZUgjTqCcE=,tag:TtmMztBvtWoRiqzVDiK67w==,type:str]
    pgp:
        - created_at: "2021-11-23T07:36:04Z"
          enc: |
            -----BEGIN PGP MESSAGE-----

            hQIMA/QAsyVsZMsrARAAmORzTOYSFG4SVdY/eNSplwgO898vav2qbQqZqvlpoNrV
            3Q2Khd8WryoNzUTxxTEusFS4yV5ClpL67on5pMKMTgRO4oPqpjFN72hKJHaaR4Ff
            7WiivRbwPvWeG6YyabSiJcwCQWCKa1QvbQHEo0p1fEKd3Mlb4Pe8l+2buXc5M6QL
            bLmnr1reQ4c8bdbd9TzT509cLrJACTNKuaxVYh7VNXWs4LwB4ZBvbG1LqzoAQvm4
            9EiXlFfrtO1EQvIkmJ/1FGj6p5EKD+62/YMZmFNxGhUJe2qMwsIqz/beZfocLrgR
            Mn4urwu6nqPBwtkzaC7DL5DngUs5xg+qkYgKxeHMAmp3H3oPNefVrNisgz8Xxlir
            jvHGTZcC/W71zdmlg5wjMhpuE4rpkXWVrbU7RjGdRwUwR3jWrPUsKI2Vkr8HFZXa
            LOG+VMzL7QezdgGMgmMPG7yCbZ7RHJAXWcn62VAVjl1WWV/7zqB7qqvMpFGeNB5o
            kVgcrd0KWsvd2zhHEUA3xplVqznEt/tOKNz8tEN78cBM+e0cRWhqtNk7o8goNPZT
            dOylww8+gPBL/AbQ2+gtJ53FuMoA0gtq859froAfoR0fqVh5Lpu2apnbUCveDnzv
            NjckXEuWvO6Xh9RqXhZb/FXVg8Z+cgUB4zC3IUcc77vD4plq8Vs9OPn20Wx+EXPS
            XgEf0rufhTa73Ue1JCwaPUFHsVI76kbFVNVgDlTjrXGwuqaqQRrxTfKSGdDwcGju
            TFhDb/0dGtMfi3T2ilTTp/r/22bXYc0lJip4yBnrnpUz8im3O8hBAzX9/Yi70dA=
            =UGWs
            -----END PGP MESSAGE-----
          fp: 179D7100D05CD229405564A17694B92348945E38
    unencrypted_suffix: _unencrypted
    version: 3.7.1

```

</details>

不难看出，SOPS 仅仅是对 YAML 文件中的value进行了加密，而保留了所有 Key 信息，这使得即使是加密后的文件仍然保留了很强的可读性。

此外，SOPS 还在文件中追加了一个新的 Key 值 `sops`，用于保存加密该文件时所使用的一些加密信息。

当然，我们还可以通过 sops 命令对加密后的文件进行解密操作：

```
# sops --decrypt test.yaml
password: secret123
```

`--decrypt, -d` 参数指定文件进行解密操作。当然，也可以通过 `-p` 参数将解密时的私钥 ID 传递给 `sops` 命令，但这通常是不必要的，因为与 `gpg` 一样，`sops` 在加密文件时会自动解密时需要的私钥 ID 记录在内。

> 如果报错将下面内容添加到`.bashrc`中去，并更新`.bashrc`
>
> ```
> GPG_TTY=$(tty)
> export GPG_TTY
> ```
>
> https://github.com/mozilla/sops/issues/304#issuecomment-377195341

#### 使用helm-secrets plugin加密 `secrets.yaml` 文件

helm-secrets plugin背后就是使用了`sops`命令进行加密

- 创建`secret.yaml`

```bash
# echo "password: secret123" > test-helm-secret.yaml
# cat test-helm-secret.yaml
password: secret123
```

- 直接运行会报错，需要通过设定环境变量和配置文件的方式来指定 GPG 密钥对。

  - **设定环境变量：**

    ```
    $ export SOPS_PGP_FP=179D7100D05CD229405564A17694B92348945E38
    ```

  - **SOPS 配置文件（推荐）：**

    在**当前目录下**创建名为 `.sops.yaml` 的文件作为它的配置文件，下面是一个最基本的 SOPS 配置文件

    ```
    creation_rules:
        - pgp: "179D7100D05CD229405564A17694B92348945E38"
    ```

    SOPS 的配置文件能做到的远远不止这些，我们可以在配置文件中设定多条规则，并为每条规则设定不同的密钥信息，以达到不同的文件使用不同的密钥进行加密和解密，更多关于 SOPS 配置信息，请参考 SOPS 文档。

- 接下来使用`helm-secrets` 插件的 `enc` 子命令对该文件进行加密：

```bash
# helm secrets enc test-helm-secret.yaml 
Encrypting test-helm-secret.yaml
[PGP]	 WARN[0000] Deprecation Warning: GPG key fetching from a keyserver within sops will be removed in a future version of sops. See https://github.com/mozilla/sops/issues/727 for more information. 
Encrypted test-helm-secret.yaml

```

加密结果：

<details>

```
# cat test-helm-secret.yaml 
password: ENC[AES256_GCM,data:BozXS1u7bZW+,iv:EtugCz+FyLdJ0jYhzu2LRVpKON//XLvDbqC0CEqYY4Q=,tag:1Bswsof7KQ2j+N0WVRzcjQ==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2021-11-23T07:47:54Z"
    mac: ENC[AES256_GCM,data:3wl/kobAZiYIqoEI9mIhFK1B1wGofBiMHJhng2q0T24J4xRJcHHCAlquiX8xAQRmBSddamdB+OmZseoTzXTZGrOh3u/gpWajNYjJkohPz6WmIOdQNdwbdfBYkwICNYU+uzvPVaaVxY3lyTqxDPRiuUEXubWcm2whM+WP7dphQlU=,iv:4IW8Sey5WZnDT2bWWCQN/q8WjqIv20wJ2jpyGTKhQqI=,tag:YqpVISRsKXyGsmG2eFU7UA==,type:str]
    pgp:
        - created_at: "2021-11-23T07:47:53Z"
          enc: |
            -----BEGIN PGP MESSAGE-----

            hQIMA/QAsyVsZMsrAQ/+MkL2qQ51b2/aJy2jHviXAzXYtO1FC5YBkdT/7F29cXhU
            Ljpq46ZIxo3tepSfgkl+aqN3B4hgUvMspJS/cyvxmDSuBoxmo2jJY5KmDKY/dxGx
            Zh8cK8jK6Sxcl98iUbw8Kkt7h4H5+NlAOHFI28+lUx4xLmfhqf4dm2pYst3TwZJg
            ewfxuI4slnXzNFFiIsgto7ulvAzmorRJRIA6v6yhD42QIdFbKKRKAb1zfnRyY+Ys
            JmLpU6EMV+D+ijtz5n6oO4qdMkSRxnCc+oUN0k0qAGQ9uwejs1/vr4W7/tUFif7N
            6h4Kg5r1/f0zkwbGud58tSPSzMgL8hZcJiuiJPONuogofry73zi6w77VC2s4WJ91
            z3lKj63+PGpExHwpxaqf8eqxZJ3ngSzMRRGUeLJCuTluEZr3ouL4MAw/8s8cI6ML
            +iKPAHIywsePW0wojrQ3QcgOoGkYnCq1d9EW5+KclmMXj+yYh6fuaGkzSPeKyxMD
            wkiNTzQ/lzU07ETtroNTPRA41Xbk0uzjOL33skw6k7+qXnMwE+knEJahgg+1msqF
            dfeJ6PqpCAQwa81I5Oq2SgMP6LaEHIodr+iawwdZCV5zX/WA+Pr5JLw8Xid8XXf4
            fCxlHKi9ZeeZclGavoEKZ7BHXOv7AqvtWQKii0fhdsG6ukustektfJL+2mXgih3S
            XgH3kkbQsZzD8z9U/2WHV5BPGYux6TMn14mxFKcyp16dW2NLebUd3aTYVjo8tynI
            /6irQcXGwfKw87jz2lcx8z0RDGd7XKXCy5EezmrYDKdtv4DiZ3gz5wBiqJbZ+Mc=
            =yLfm
            -----END PGP MESSAGE-----
          fp: 179D7100D05CD229405564A17694B92348945E38
    unencrypted_suffix: _unencrypted
    version: 3.7.1
```

</details>

加密后的 `secrets.yaml` 文件将被用于部署我们的 Chart，且该文件也可以被自由分发、存储到 Git 中去，因为我们只要确保 PGP 私钥不被泄漏，就无需担心该文件会被他人解密，从而导致敏感信息被泄漏的问题。

- 使用`helm secrets dec` 子命令进行decrypt：

  与 SOPS 一样，解密后的内容将被存储到新创建的 `test-helm-secret.yaml.dec` 文件中去

```bash
# helm secrets dec test-helm-secret.yaml 
Decrypting test-helm-secret.yaml
# cat test-helm-secret.yaml.dec 
password: secret123
```

- 用 `helm secrets view` 子命令直接查看解密后的文件内容：

```bash
# helm secrets view test-helm-secret.yaml
password: secret123
```



#### 公钥的导入与导出

通过前面对 PGP 的讲解我们了解到，公钥用于加密，私钥用于解密。因此，你若希望其他人或在其它工作机器上也有能够加密 Values 文件的能力，那么，你只需将你的 PGP 公钥导入到他的工作电脑上即可。

导入 PGP 公钥有两种方式：一种是手动导出导入公钥；另一种是上传你的公钥到公钥服务器中，其他人则可以直接从公钥服务器中导入你的公钥。下面就让我们分别看一下如何通过这两种方式导入与导出你的公钥。

**手动导入导出公钥**

首先将公钥导入到文件中：

```
$ gpg --export --armor "helm-secret@email.com" > helm.pub
```

该命令将公钥导出到了名为 `helm.pub` 的文件中，将导出的公钥传递给他人。

通过 `import` 命令导入到自己机器中即可：

```elixir
$ gpg --import helm.pub
```





## helmfile

https://github.com/cloudnativedevops/demo/tree/main/hello-helmfile

```
repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts

releases:
  - name: demo
    namespace: demo
    chart: ../hello-helm3/k8s/demo
    values:
      - "../hello-helm3/k8s/demo/production-values.yaml"

  - name: kube-state-metrics
    namespace: kube-state-metrics
    chart: prometheus-community/kube-state-metrics

  - name: prometheus
    namespace: prometheus
    chart: prometheus-community/prometheus
    set:
      - name: rbac.create
        value: true
```

- `repository`：定义了要引用的Helm chart的repository
- `releases`：我们想要deploy到cluster的应用程序。在每个release中指定了下面的metadata
  - `name`：想要deploy的Helm chart的名称
  - `namespace`：想要deploy到的命名空间
  - `chart`：chart本身的URL或者path
  - `values`：提供了`values.yaml`的所在路径
  - `set`：设置了`values.yaml`所定义之外的额外的values
- 在上述的example中定义了三个release：
  - demo app
  - kube-state-metrics
  - prometheus

### Chart Metadata

在上述的example中，我们指定的是demo chart和values文件的相对路径：

```
- name: demo
  namespace: demo
  chart: ../hello-helm3/k8s/demo
  values:
    - "../hello-helm3/k8s/demo/production-values.yaml"
```

因此，不需要Helmfile中的chart repository去管理它们。可以将它们全部保存在同一个源代码存储库中。

对于 prometheus chart来说，我们指定了`prometheus-community/prometheus`而它并不是一个filesystem路径，Helmfile会在`prometheus-community` repo中寻找`prometheus`的chart。而先前在repositories中已经定义了`prometheus-community` repo：

```
repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
```

所有的charts在它们各自的`values.yaml`都设置了默认值。在Helmfile中可以指定任何在install应用程序时想要覆盖的value。

例如，下面的prometheus release，把rbac.create的默认值从false改成了true。

```
- name: prometheus
  namespace: prometheus
  chart: prometheus-community/prometheus
  set:
  - name: rbac.create
    value: true
```

