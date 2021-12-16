[TOC]

## 1. kind创建k8s集群并初始化
- 1.1 kind安装
  
	[https://iwiki.woa.com/pages/viewpage.action?pageId=1284249742](https://iwiki.woa.com/pages/viewpage.action?pageId=1284249742)


- 1.2 kind创建k8s集群

	```
	kind create cluster
	```

- 1.3 登录kind创建的k8s集群

	```
	siriusxiong@SIRIUSXIONG-MB0 elasticweb % docker ps
	CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
	b3908295d8a8   kindest/node:v1.21.1   "/usr/local/bin/entr…"   4 minutes ago   Up 4 minutes   127.0.0.1:64022->6443/tcp   kind-control-plane
	```
	```
	siriusxiong@SIRIUSXIONG-MB0 elasticweb % docker exec -it b3908295d8a8 /bin/sh  
	```

- 1.4 安装常用工具

	```
	apt-get update
	apt-get upgrade
	apt-get install vim
	```

## 2. 检查k8s集群是否开启了动态准入控制、手动安装cert-manager

- 2.1 查看APIServer是否开启了MutatingAdmissionWebhook和ValidatingAdmissionWebhook

	```
	# 获取apiserver pod名字
	apiserver_pod_name=`kubectl get --no-headers=true po -n kube-system | grep kube-apiserver | awk '{ print $1 }'`
	# 查看api server的启动参数plugin
	kubectl get po $apiserver_pod_name -n kube-system -o yaml | grep plugin
	```
	如果输出如下，说明已经开启
	```
	- --enable-admission-plugins=NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
	```
	否则，需要修改启动参数，请不然直接修改Pod的参数，这样修改不会成功，请修改配置文件/etc/kubernetes/manifests/kube-apiserver.yaml，加上相应的插件参数后保存，APIServer的Pod会监控该文件的变化，然后重新启动。

- 2.2 修改配置文件

- 2.3 手动安装cert-manager

	```
	kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
	```

## 3. CRD、webhook的构建学习参考

- kubebuilder官方文档：[英文版（推荐）](https://book.kubebuilder.io/)、[中文版（版本落后）](https://cloudnative.to/kubebuilder/)
- 一篇CRD、webhook从0到1构建的不错的博客：[kubebuilder实战](https://xinchen.blog.csdn.net/article/details/113035349)
  - 部分代码和操作可能因为kubebuilder版本问题，需要对着官网文档来操作
  - main里面要添加log的初始化函数，否则报错
  
	```
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