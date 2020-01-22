# Kong系列-15-自定义插件-Kubernetes

下面演示如何在Kong for Kubernetes加载自定义插件。

先准备自定义插件的Lua代码。

	mkdir myheader
	cd myheader
	
	vi handler.lua
	local MyHeader = {}
	
	MyHeader.PRIORITY = 1000
	
	function MyHeader:header_filter(conf)
	  -- do custom logic here
	  kong.response.set_header("myheader", conf.header_value)
	end
	
	return MyHeader
	
	vi schema.lua
	return {
	  name = "myheader",
	  fields = {
	    { config = {
	        type = "record",
	        fields = {
	          { header_value = { type = "string", default = "twingao", }, },
	        },
	    }, },
	  }
	}
	
	tree myheader/
	myheader/
	├── handler.lua
	└── schema.lua

创建Configmap或者Secret，将自定义插件myheader加载入Kubernetes。

	kubectl create configmap kong-plugin-myheader --from-file=myheader -n kong
	
	或者
	
	kubectl create secret generic kong-plugin-myheader --from-file=myheader
	
	kubectl get cm
	NAME                                        DATA   AGE
	kong-plugin-myheader                        2      25s
	
	kubectl get secret
	NAME                                                 TYPE                                  DATA   AGE
	kong-plugin-myheader                                 Opaque                                2      29s

对于按照[Kong系列-04-Helm安装Kong 1.3.0 with PostgreSQL and with Ingress Controller](04-kong-1.3.0-helm-postresql.md)使用helm安装的Kong的情况，要加载自定义插件到Kong中，需要修改helm的values.yaml文件的配置，但由于没有搞清楚如何升级应用的部分配置，只能先卸载gateway应用，并删除自动创建的pvc。

	helm uninstall gateway
	
	kubectl delete pvc data-gateway-postgresql-0

将kong直接下载下来，并修改values.yaml文件配置。并重新部署Kong。此时会自动加载自定义插件。

	helm pull stable/kong --version 0.26.1 --untar
	
	vi kong/values.yaml
	# 可以增加到最后面
	plugins:
	  configMaps:
	  - name: kong-plugin-myheader
	    pluginName: myheader
	
	或者
	
	plugins:
	  secrets:
	  - name: kong-plugin-myheader
	    pluginName: myheader
	
	helm install gateway kong \
	    --set admin.useTLS=false \
	    --set admin.nodePort=32444 \
	    --set proxy.http.nodePort=32080 \
	    --set proxy.tls.nodePort=32443 \
	    --set replicaCount=2 \
	    --set ingressController.enabled=true \
	    --set postgresql.persistence.storageClass="nfs-client" \
	    --set postgresql.persistence.size=1Gi

通过查看配置，可以看出确实加载自定义插件了。

	curl http://192.168.1.55:32444 -s | python -m json.tool
	{
	......
	        "loaded_plugins": {
	            "acl": true,
	            "aws-lambda": true,
	            "azure-functions": true,
	            "basic-auth": true,
	            "bot-detection": true,
	            "correlation-id": true,
	            "cors": true,
	            "datadog": true,
	            "file-log": true,
	            "hmac-auth": true,
	            "http-log": true,
	            "ip-restriction": true,
	            "jwt": true,
	            "key-auth": true,
	            "kubernetes-sidecar-injector": true,
	            "ldap-auth": true,
	            "loggly": true,
	
	            "myheader": true,   #注意此处
	
	            "oauth2": true,
	            "post-function": true,
	            "pre-function": true,
	            "prometheus": true,
	            "proxy-cache": true,
	            "rate-limiting": true,
	            "request-size-limiting": true,
	            "request-termination": true,
	            "request-transformer": true,
	            "response-ratelimiting": true,
	            "response-transformer": true,
	            "session": true,
	            "statsd": true,
	            "syslog": true,
	            "tcp-log": true,
	            "udp-log": true,
	            "zipkin": true
	        },
	......
	}

测试一下效果。

	vi myheader-plugin.yaml
	apiVersion: configuration.konghq.com/v1
	kind: KongPlugin
	metadata:
	  name: myheader-plugin
	plugin: myheader
	
	vi echo-server-service.yaml
	apiVersion: v1
	kind: Service
	metadata:
	  labels:
	    app: echo
	  name: echo
	spec:
	  ports:
	  - port: 8080
	    name: high
	    protocol: TCP
	    targetPort: 8080
	  selector:
	    app: echo
	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  labels:
	    app: echo
	  name: echo
	spec:
	  replicas: 2
	  selector:
	    matchLabels:
	      app: echo
	  strategy: {}
	  template:
	    metadata:
	      creationTimestamp: null
	      labels:
	        app: echo
	    spec:
	      containers:
	      - image: e2eteam/echoserver:2.2
	        name: echo
	        ports:
	        - containerPort: 8080
	        env:
	          - name: NODE_NAME
	            valueFrom:
	              fieldRef:
	                fieldPath: spec.nodeName
	          - name: POD_NAME
	            valueFrom:
	              fieldRef:
	                fieldPath: metadata.name
	          - name: POD_NAMESPACE
	            valueFrom:
	              fieldRef:
	                fieldPath: metadata.namespace
	          - name: POD_IP
	            valueFrom:
	              fieldRef:
	                fieldPath: status.podIP
	        resources: {}
	
	vi echo-server-ingress.yaml
	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	  name: echo-server-ingress
	  annotations:
	    plugins.konghq.com: myheader-plugin
	spec:
	  rules:
	  - http:
	      paths:
	      - path: /bar
	        backend:
	          serviceName: echo
	          servicePort: 80
	
	kubectl apply -f myheader-plugin.yaml
	kubectl apply -f echo-server-service.yaml
	kubectl apply -f echo-server-ingress.yaml
	
	curl -I http://192.168.1.55:32080/bar
	HTTP/1.1 200 OK
	Content-Type: text/plain; charset=UTF-8
	Connection: keep-alive
	Date: Thu, 26 Dec 2019 10:05:24 GMT
	Server: echoserver
	myheader: twingao       #增加了header
	X-Kong-Upstream-Latency: 2
	X-Kong-Proxy-Latency: 8
	Via: kong/1.3.0