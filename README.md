[TOC]

## CRD、webhook的构建学习参考

> 主要根据【1.学习指引】来学习，2、3主要是相关补充

### 1. 学习指引

- kubebuilder官方文档：[英文版（推荐）](https://book.kubebuilder.io/)、[中文版（版本落后）](https://cloudnative.to/kubebuilder/)
- 一篇CRD、webhook从0到1构建的不错的博客：[kubebuilder实战](https://xinchen.blog.csdn.net/article/details/113035349)
  - 部分代码和操作可能因为kubebuilder版本问题，需要对着官网文档来操作
  - main里面要添加log的初始化函数，否则报错
  
	```go
		if err = (&controllers.ElasticWebReconciler{
		Client: mgr.GetClient(),
		Log:    ctrl.Log.WithName("controllers").WithName("ElasticWeb"), #添加这行
		Scheme: mgr.GetScheme(),
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "ElasticWeb")
		os.Exit(1)
	}
	```

  - 因为没有配置secret，所以使用镜像部署时注意要在仓库设置为公共镜像 
- 一个最终可用的工程：[https://github.com/butcoder/elasticweb](https://github.com/butcoder/elasticweb)

### 2. kind创建k8s集群并初始化
- 2.1 kind安装
  
	https://github.com/kubernetes-sigs/kind


- 2.2 kind创建k8s集群

	```shell
	kind create cluster
	```

- 2.3 登录kind创建的k8s集群

	```shell
	siriusxiong@SIRIUSXIONG-MB0 elasticweb % docker ps
	CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
	b3908295d8a8   kindest/node:v1.21.1   "/usr/local/bin/entr…"   4 minutes ago   Up 4 minutes   127.0.0.1:64022->6443/tcp   kind-control-plane
	```
	```shell
	siriusxiong@SIRIUSXIONG-MB0 elasticweb % docker exec -it b3908295d8a8 /bin/sh  
	```

- 2.4 安装常用工具

	```shell
	apt-get update
	apt-get upgrade
	apt-get install vim
	```

### 3. 检查k8s集群是否开启了动态准入控制、手动安装cert-manager

- 3.1 查看APIServer是否开启了MutatingAdmissionWebhook和ValidatingAdmissionWebhook

	```shell
	# 获取apiserver pod名字
	apiserver_pod_name=`kubectl get --no-headers=true po -n kube-system | grep kube-apiserver | awk '{ print $1 }'`
	# 查看api server的启动参数plugin
	kubectl get po $apiserver_pod_name -n kube-system -o yaml | grep plugin
	```
	如果输出如下，说明已经开启
	```shell
	- --enable-admission-plugins=NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
	```
	否则，需要修改启动参数，请不然直接修改Pod的参数，这样修改不会成功，请修改配置文件/etc/kubernetes/manifests/kube-apiserver.yaml，加上相应的插件参数后保存，APIServer的Pod会监控该文件的变化，然后重新启动。

- 3.2 修改配置文件

- 3.3 手动安装cert-manager

	```shell
	kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
	```

## kubebuilder 构建 webhook 开启 validate delete

> 在官放kubebuilder文档或者其他人的webhook demo中，都没有启用delete的，所以这里额外补充开启delete validate webhook的部分

- 1. 开启delete

	- 根据kubebuilder的注释修改即可
		```go
		// TODO(user): change verbs to "verbs=create;update;delete" if you want to enable deletion validation.
		//+kubebuilder:webhook:path=/validate-elasticweb-com-bolingcavalry-v1-elasticweb,mutating=false,failurePolicy=fail,sideEffects=None,groups=elasticweb.com.bolingcavalry,resources=elasticwebs,verbs=create;update;delete,versions=v1,name=velasticweb.kb.io,admissionReviewVersions=v1
		```
	- `make install`
		kubebuilder 根据注释自动修改相关的配置文件:config/webhook/manifests.yaml

- 2. 编写 delete validate 逻辑

	api/v1/elasticweb_webhook.go:

	```go
	func (r *ElasticWeb) ValidateDelete() error {
		//判断逻辑
	}
	```

	在`ValidateDelete()`函数中，不建议长时间阻塞，立即返回nil/error即可，相关资源的controller的协调器在涉及到删除操作时，会被webhook过滤并允许通过/拒绝

- 3. 验证 delete validate webhook 是否生效
  - 部署一个demo
	config/sample/02.yaml:
	```yaml
	apiVersion: v1
	kind: Namespace
	metadata:
	  name: dev2
	  labels:
	    name: dev2
	---
	apiVersion: elasticweb.com.bolingcavalry/v1
	kind: ElasticWeb
	metadata:
	namespace: dev2
	name: elasticweb-sample
	spec:
	# Add fields here
	image: ccr.ccs.tencentyun.com/butcoder/tomcat
	port: 30004
	singlePodQPS: 600
	totalQPS: 600
	```
	```shell
	kubectl apply -f config/sample/02.yaml
	```
  - 删除一个elasticweb-sample服务
	修改config/sample/02.yaml:
	```yaml
	#apiVersion: v1
	#kind: Namespace
	#metadata:
	#  name: dev2
	#  labels:
	#    name: dev2
	#---
	apiVersion: elasticweb.com.bolingcavalry/v1
	kind: ElasticWeb
	metadata:
	namespace: dev2
	name: elasticweb-sample
	spec:
	# Add fields here
	image: ccr.ccs.tencentyun.com/butcoder/tomcat
	port: 30004
	singlePodQPS: 600
	totalQPS: 600
	```
	```
	kubectl delete -f config/sample/02.yaml
	```
  - 验证
	- 日志查看
		```shell
		kubectl logs -f elasticweb-xxxxxxx \
		-c manager \
		-n elasticweb-system
		```
		能够发现webhook返回的FALSE
	- 查看pod
		```shell
		kubectl get pod -n dev2
		```
		发现pod没有被删除