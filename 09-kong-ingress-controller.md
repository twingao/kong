# Kong系列-09-Kong Ingress Controller介绍和入门

Kong之前都是使用Admin API来进行管理的，Kong主要暴露两个端口管理端口8001和代理端口8000，管理Kong主要的是为上游服务配置Service、Routes、Plugins、Consumer等实体资源，Kong按照这些配置规则进行对上游服务的请求进行路由分发和控制。在Kubernetes集群环境下，Admin API方式不是很适应Kubernetes声明式管理方式。所以Kong在Kubernetes集群环境下推出Kong Ingress Controller。Kong Ingress Controller定义了四个CRDs（CustomResourceDefinitions），基本上涵盖了原Admin API的各个方面。

- kongconsumers：Kong的用户，给不同的API用户提供不同的消费者身份。
- kongcredentials：Kong用户的认证凭证。
- kongingresses：定义代理行为规则，是对Ingress的补充配置。
- kongplugins：插件的配置。


    kubectl get crds
    NAME                                       CREATED AT
    kongconsumers.configuration.konghq.com     2019-12-15T08:02:29Z
    kongcredentials.configuration.konghq.com   2019-12-15T08:02:29Z
    kongingresses.configuration.konghq.com     2019-12-15T08:02:29Z
    kongplugins.configuration.konghq.com       2019-12-15T08:02:29Z

以下为[Kong系列-04-Helm安装Kong 1.3.0 with PostgreSQL and with Ingress Controller](04-kong-helm-postresql.md)的环境，可以看出Kong Pod其中有两个容器，一个为ingress-controller，一个为kong。Kong对外提供两个服务，gateway-kong-admin为管理服务，支持Admin API，gateway-kong-proxy为代理服务,这两个服务都由kong提供，而CRDs的API接口是ingress-controller容器提供的。

    kubectl get all -o wide
    NAME                                            READY   STATUS      RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
    pod/gateway-kong-79498b67b7-plmlm               2/2     Running     5          34d   10.244.1.13   k8s-node1   <none>           <none>
    pod/gateway-kong-79498b67b7-zcfh6               2/2     Running     5          34d   10.244.2.10   k8s-node2   <none>           <none>
    pod/gateway-kong-init-migrations-5qdxc          0/1     Completed   0          34d   10.244.1.10   k8s-node1   <none>           <none>
    pod/gateway-postgresql-0                        1/1     Running     1          34d   10.244.1.14   k8s-node1   <none>           <none>
    
    NAME                                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
    service/gateway-kong-admin            NodePort    10.1.6.70      <none>        8444:32444/TCP               34d   app=kong,component=app,release=gateway
    service/gateway-kong-proxy            NodePort    10.1.232.237   <none>        80:32080/TCP,443:32443/TCP   34d   app=kong,component=app,release=gateway
    service/gateway-postgresql            ClusterIP   10.1.161.34    <none>        5432/TCP                     34d   app=postgresql,release=gateway,role=master
    service/gateway-postgresql-headless   ClusterIP   None           <none>        5432/TCP                     34d   app=postgresql,release=gateway
    service/kubernetes                    ClusterIP   10.1.0.1       <none>        443/TCP                      34d   <none>
    
    NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS                IMAGES                                                                                        SELECTOR
    deployment.apps/gateway-kong                             2/2     2            2           34d   ingress-controller,kong   kong-docker-kubernetes-ingress-controller.bintray.io/kong-ingress-controller:0.6.0,kong:1.3   app=kong,component=app,release=gateway

其实在Kubernetes集群中也可以直接部署Kong和PostgreSQL，那样是不支持Kong Ingress Controller，直接使用Admin API管理即可。

下面介绍一下如何使用Kong Ingress Controller。先将Kong初始化为空配置。

    curl -i http://192.168.1.55:32080/
    HTTP/1.1 404 Not Found
    Date: Sun, 22 Dec 2019 11:12:00 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    {"message":"no Route matched with those values"}
    
创建一个echo服务。
    
    vi echo-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: echo
      name: echo
    spec:
      ports:
      - name: http
        port: 8080
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
    
    kubectl apply -f echo-service.yaml

创建Ingress，定义路由规则。

    vi echo-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: echo-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /foo
            backend:
              serviceName: echo
              servicePort: 80    

    kubectl apply -f echo-ingress.yaml

根据Ingress规则，访问echo服务。

    curl -i http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 11:34:02 GMT
    Server: echoserver
    X-Kong-Upstream-Latency: 6
    X-Kong-Proxy-Latency: 13
    Via: kong/1.3.0
    
    
    Hostname: echo-75cf96d976-4qvx4
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-4qvx4
            pod namespace:  default
            pod IP: 10.244.1.21
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.20
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://192.168.1.55:8080/
    
    Request Headers:
            accept=*/*
            connection=keep-alive
            host=192.168.1.55:32080
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=192.168.1.55
            x-forwarded-port=8000
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
    
    Request Body:
            -no body in request-

再演示一下插件的使用，插件可以在Ingress上启用。

先创建Correlation ID插件。Correlation ID可以在请求头中增加一个UUID，也可以在响应头中返回，可以用来追踪请求响应对。

    vi correlation-id-plugin.yaml
    ---
    apiVersion: configuration.konghq.com/v1
    kind: KongPlugin
    metadata:
      name: request-id
    config:
      header_name: my-request-id
      generator: uuid#counter
      echo_downstream: true
    plugin: correlation-id
    
    kubectl apply -f correlation-id-plugin.yaml

新建一个新的Ingress，并在Ingress应用新建的插件。注意Correlation ID插件没有应用到上一个Ingress上。

    vi echo-ingress-2.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: echo-ingress-2
      annotations:
        plugins.konghq.com: request-id
    spec:
      rules:
      - host: example.com
        http:
          paths:
          - path: /bar
            backend:
              serviceName: echo
              servicePort: 80
    
    kubectl apply -f echo-ingress-2.yaml

测试一下效果，访问/bar路径，可以发现插件已经启用，在请求和响应中都增加了头my-request-id: 6827852e-c165-4479-b5c9-a953ca3ff69b#1

    curl -i -H "Host: example.com" http://192.168.1.55:32080/bar/sample
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Sat, 18 Jan 2020 10:00:06 GMT
    Server: echoserver
    my-request-id: 6827852e-c165-4479-b5c9-a953ca3ff69b#1
    X-Kong-Upstream-Latency: 5
    X-Kong-Proxy-Latency: 166
    Via: kong/1.3.0
    
    
    Hostname: echo-75cf96d976-sl7xs
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-75cf96d976-sl7xs
            pod namespace:  default
            pod IP: 10.244.2.12
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.10
            method=GET
            real path=/sample
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://example.com:8080/sample
    
    Request Headers:
            accept=*/*
            connection=keep-alive
            host=example.com
            my-request-id=6827852e-c165-4479-b5c9-a953ca3ff69b#1
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=example.com
            x-forwarded-port=8000
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
    
    Request Body:
            -no body in request-

访问/foo路径，可以发现确实没有改请求头。

    curl -i http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Sat, 18 Jan 2020 09:59:10 GMT
    Server: echoserver
    X-Kong-Upstream-Latency: 3
    X-Kong-Proxy-Latency: 17
    Via: kong/1.3.0
    
    
    Hostname: echo-75cf96d976-g9db2
    
    Pod Information:
            node name:      k8s-node1
            pod name:       echo-75cf96d976-g9db2
            pod namespace:  default
            pod IP: 10.244.1.15
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.1.13
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://192.168.1.55:8080/
    
    Request Headers:
            accept=*/*
            connection=keep-alive
            host=192.168.1.55:32080
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=192.168.1.55
            x-forwarded-port=8000
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
    
    Request Body:
            -no body in request-

再演示一下插件在Service上启用。限速插件Rate Limiting可以配置一定时间内可以请求的次数，如下设置为限速5次/分钟。

    vi rate-limiting-plugin.yaml
    ---
    apiVersion: configuration.konghq.com/v1
    kind: KongPlugin
    metadata:
      name: rl-by-ip
    config:
      minute: 5
      limit_by: ip
      policy: local
    plugin: rate-limiting
    
    kubectl apply -f rate-limiting-plugin.yaml

将该插件在Service上启用。

    kubectl patch service echo \
      -p '{"metadata":{"annotations":{"plugins.konghq.com": "rl-by-ip\n"}}}'

第三方三发

    #HTTP requests with /foo -> Kong enforces rate-limit -> echo server
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:10 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 4
    X-Kong-Upstream-Latency: 3
    X-Kong-Proxy-Latency: 4
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:11 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 3
    X-Kong-Upstream-Latency: 2
    X-Kong-Proxy-Latency: 0
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:13 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 4
    X-Kong-Upstream-Latency: 2
    X-Kong-Proxy-Latency: 1
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:13 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 3
    X-Kong-Upstream-Latency: 2
    X-Kong-Proxy-Latency: 3
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:14 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 2
    X-Kong-Upstream-Latency: 1
    X-Kong-Proxy-Latency: 3
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:14 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 1
    X-Kong-Upstream-Latency: 1
    X-Kong-Proxy-Latency: 3
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:15 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 0
    X-Kong-Upstream-Latency: 1
    X-Kong-Proxy-Latency: 2
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 429 Too Many Requests
    Date: Sun, 22 Dec 2019 12:01:15 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 37
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 0
    Server: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:16 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 2
    X-Kong-Upstream-Latency: 3
    X-Kong-Proxy-Latency: 5
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:17 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 1
    X-Kong-Upstream-Latency: 4
    X-Kong-Proxy-Latency: 4
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:01:17 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 0
    X-Kong-Upstream-Latency: 4
    X-Kong-Proxy-Latency: 4
    Via: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 429 Too Many Requests
    Date: Sun, 22 Dec 2019 12:01:17 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 37
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 0
    Server: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 429 Too Many Requests
    Date: Sun, 22 Dec 2019 12:01:18 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 37
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 0
    Server: kong/1.3.0
    
    curl -I http://192.168.1.55:32080/foo
    HTTP/1.1 429 Too Many Requests
    Date: Sun, 22 Dec 2019 12:01:19 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 37
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 0
    Server: kong/1.3.0

访问/bar路径，可以发现两个插件同时启用了。

    #HTTP requests with /bar -> Kong enforces rate-limit +   -> echo-server
    #   on example.com          injects my-request-id header
    curl -I -H "Host: example.com" http://192.168.1.55:32080/bar/sample
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Connection: keep-alive
    Date: Sun, 22 Dec 2019 12:10:21 GMT
    Server: echoserver
    X-RateLimit-Limit-minute: 5
    X-RateLimit-Remaining-minute: 4
    my-request-id: 0eee7f62-7681-45d7-85b2-3b8f8ff63a0f#3
    X-Kong-Upstream-Latency: 11
    X-Kong-Proxy-Latency: 27
    Via: kong/1.3.0