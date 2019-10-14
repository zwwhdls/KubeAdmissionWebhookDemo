# KubeAdmissionWebhookDemo

这里使用一个简单的场景做一个演示，我们自定义一个名为 App 资源，当用户创建一个 App 实例时，我们根据用户的描述创建出一个 Deployment。

然后我们添加一个 `MutatingAdmissionWebhook`，当用户通过 App 创建 Deployment 时，自动添加一个 sidecar 容器到 Pod 中（这里使用 nginx 作为 sidecar）。

## 初始化 API 及 Controller

第一步是创建出 CRD 及其 Controller，几行命令就能搞定：

```bash
$ export GO111MODULE=on
﻿
$ mkdir $GOPATH/src/zww-app
$ cd $GOPATH/src/zww-app
$ kubebuilder init --domain o0w0o.cn --owner "zwwhdls"
﻿
$ kubebuilder create api --group app --version v1 --kind App
```

我这里做的比较简单，`AppSpec` 只定义了一个 deploy 属性（就是 `appsv1.DeploymentSpec`），Controller 中会根据 deploy 属性生成对应的 Deployment：

```go
type AppSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Deploy appsv1.DeploymentSpec `json:"deploy,omitempty"`
}
```

在完善了 `AppSpec` 和 Controller 的 `Reconcile` 函数后，使 Kubebuilder 重新生成代码，并将 `config/crd` 下的 CRD yaml 应用到当前集群：

```bash
make
make install
```

## 创建 Webhook Server

接下来就是用 Kubebuilder 来生成 Webhooks 了：

```bash
kubebuilder create webhook --group app --version v1 --kind App 
```

在路径 `api/v1` 下生成了一个名为 `app_webhook.go` 的文件。可以看到 Kubebuilder 已经帮你定义了两个变量：

```go
var _ webhook.Defaulter = &App{}
var _ webhook.Validator = &App{}
```

这两个变量分别表示 MutatingWebhookServer 和 ValidatingWebhookServer，在程序启动的时候，这两个 Server 会 run 起来。

对于 MutatingWebhookServer，Kubebuilder 预留了 `Default()` 函数，让用户来填写自己的逻辑：

```go
// Default implements webhook.Defaulter so a webhook will be registered for the type
func (r *App) Default() {
	applog.Info("default", "name", r.Name)

	// TODO(user): fill in your defaulting logic.
}
```

对于我们希望 Webhook 在资源发生什么样的变化时触发，可以通过这条注释修改：

```go
// +kubebuilder:webhook:path=/mutate-app-o0w0o-cn-v1-app,mutating=true,failurePolicy=fail,groups=app.o0w0o.cn,resources=apps,verbs=create;update,versions=v1,name=mapp.kb.io
```

对应的参数为：

- failurePolicy：表示 ApiServer 无法与 webhook server 通信时的失败策略，取值为 "ignore" 或 "fail"；
- groups：表示这个 webhook 在哪个 Api Group 下会收到请求；
- mutating：这个参数是个 bool 型，表示是否是 mutating 类型；
- name：webhook 的名字，需要与 configuration 中对应；
- path：webhook 的 path；
- resources：表示这个 webhook 在哪个资源发生变化时会收到请求；
- verbs：表示这个 webhook 在资源发生哪种变化时会收到请求，取值为 “create“, "update", "delete", "connect", 或 "*" (即所有)；
- versions：表示这个 webhook 在资源的哪个 version 发生变化时会收到请求；

对于 ValidatingWebhookServer，Kubebuilder 的处理与 MutatingWebhookServer 一致，这里不再赘述。

方便起见，我只定义了 MutatingWebhookServer 的 `Default` 函数，为每个 App 类型资源的 pod 注入一个 nginx sidecar 容器：

```go
func (r *App) Default() {
	applog.Info("default", "name", r.Name)
	var cns []core.Container
	cns = r.Spec.Deploy.Template.Spec.Containers

	container := core.Container{
		Name:  "sidecar-nginx",
		Image: "nginx:1.12.2",
	}

	cns = append(cns, container)
	r.Spec.Deploy.Template.Spec.Containers = cns
}
```

## 运行 Webhook Server

首先需要将 MutatingWebhookConfiguration 稍作修改，使得 ApiServer 能够与 Webhook Server 通信。具体方法如下：

### 配置 Server Path

本文仅分享本地开发测试的调试方案，线上部署方案请参考[官方文档](https://book.kubebuilder.io/cronjob-tutorial/running.html)。

第一步，配置 Server Path；将 service 去掉，换成 `url: https://<server_ip>:9443/mutate-app-o0w0o-cn-v1-app` ，其中 `server_ip` 是 Webhook Server 的 ip，如果运行在本地，就是本地的 ip。需要注意的是 url 中的 path 要与 `app_webhook.go` 中定义的保持一致。

### 配置证书

第二步，配置 caBundle；由于在 Kube 里，所有与 ApiServer 交互的组件都需要与 ApiServer 进行双向 TLS 认证，我们这里需要先手动签发自签名 CA 证书：

```bash
$ openssl genrsa -out ca.key 2048
$ openssl req -x509 -new -nodes -key ca.key -subj "/CN=<server_ip>" -days 10000 -out ca.crt
$ openssl genrsa -out server.key 2048
$ cat << EOF >csr.conf
> [ req ]
> default_bits = 2048
> prompt = no
> default_md = sha256
> req_extensions = req_ext
> distinguished_name = dn
> 
> [ dn ]
> C = <country>
> ST = <state>
> L = <city>
> O = <organization>
> OU = <organization unit>
> CN = <server_ip>
> 
> [ req_ext ]
> subjectAltName = @alt_names
> 
> [ alt_names ]
> IP.1 = <server_ip>
> 
> [ v3_ext ]
> authorityKeyIdentifier=keyid,issuer:always
> basicConstraints=CA:FALSE
> keyUsage=keyEncipherment,dataEncipherment
> extendedKeyUsage=serverAuth,clientAuth
> subjectAltName=@alt_names
> EOF
$ openssl req -new -key server.key -out server.csr -config csr.conf
$ openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000 -extensions v3_ext -extfile csr.conf
```

证书生成后将 `server.key` 和 `server.crt` 拷贝到 Kubebuilder 设置的 webhook server 的私钥和证书路径下：

webhook server 的私钥路径：`$(TMPDIR)/k8s-webhook-server/serving-certs/tls.key`
webhook server 的证书路径：`$(TMPDIR)/k8s-webhook-server/serving-certs/tls.crt`

*注：如果 $(TMPDIR) 为空，则默认路径为 "/tmp/k8s-webhook-server/..."，但 android 系统默认路径为 "/data/local/tmp/k8s-webhook-server/..."*

而 MutatingWebhookConfiguration 中的 caBundle 为 ca.crt 的 base64 编码结果。最终 yaml 结果为：

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  creationTimestamp: null
  name: mutating-webhook-configuration
webhooks:
- clientConfig:
  caBundle: LS0tLS1CRUdJTiBDRVJ...FLS0tLS0=
  url: https://<server_ip>:9443/mutate-app-o0w0o-cn-v1-app
  failurePolicy: Fail
  name: mapp.kb.io
  rules:
    ...
```

ValidatingWebhookConfiguration 的修改与 MutatingWebhookConfiguration 类似，只需要注意 server path 与 `app_webhook.go` 中一致即可。两个配置文件都修改好之后在集群中 apply 一下即可。

### 运行

最后直接在本地运行 CRD Controller 及 Webhook Server：

```bash
make run
```

### 验证

简单运行一个 app 试试：

```yaml
apiVersion: app.o0w0o.cn/v1
kind: App
metadata:
  name: app-sample
spec:
  deploy:
    selector:
      matchLabels:
        app: app-sample
    template:
      metadata:
        name: sample
        labels:
          app: app-sample
      spec:
        containers:
          - name: cn
            image: daocloud.io/library/redis:4.0.14-alpine
```

查看是否已经注入了 sidecar 容器：

```bash
$ kubectl apply -f config/samples/app_v1_app.yaml
$ kubectl get app
NAME         AGE
app-sample   43s
$ kubectl get deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
app-sample-deploy   0/1     1            0           43s
$ kubectl get po
NAME                                 READY   STATUS              RESTARTS   AGE
app-sample-deploy-5b5cfb9c9b-z8jk5   0/2     ContainerCreating   0          43s
```

