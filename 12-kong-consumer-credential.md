# Kong系列-12-KongConsumer和Credentials介绍

KongConsumer资源是用来身份验证的消费者。下面介绍一下如何使用KongConsumer资源，先将Kong初始化为空配置。

    curl -i http://192.168.1.55:32080
    HTTP/1.1 404 Not Found
    Date: Tue, 24 Dec 2019 14:03:40 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    {"message":"no Route matched with those values"}

创建httpbin服务。

    vi httpbin-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: httpbin
      labels:
        app: httpbin
    spec:
      ports:
      - name: http
        port: 80
        targetPort: 80
      selector:
        app: httpbin
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: httpbin
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: httpbin
      template:
        metadata:
          labels:
            app: httpbin
        spec:
          containers:
          - image: docker.io/kennethreitz/httpbin
            name: httpbin
            ports:
            - containerPort: 80
    
    kubectl apply -f httpbin-service.yaml

创建对应的Ingress。

    vi httpbin-ingress.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: httpbin-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /foo
            backend:
              serviceName: httpbin
              servicePort: 80
    
    kubectl apply -f httpbin-ingress.yaml

测试一下。

    curl -i http://192.168.1.55:32080/foo/status/200
    HTTP/1.1 200 OK
    Content-Type: text/html; charset=utf-8
    Content-Length: 0
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Date: Tue, 21 Jan 2020 12:41:34 GMT
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    X-Kong-Upstream-Latency: 7
    X-Kong-Proxy-Latency: 0
    Via: kong/1.3.0

创建Key Auth插件。

    vi key-auth-plugin.yaml
    ---
    apiVersion: configuration.konghq.com/v1
    kind: KongPlugin
    metadata:
      name: key-auth-plugin
    plugin: key-auth
    
    kubectl apply -f key-auth-plugin.yaml

修改Ingress，将插件应用于该Ingress。

    vi httpbin-ingress-2.yaml
    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: httpbin-ingress
      annotations:
        plugins.konghq.com: key-auth-plugin
    spec:
      rules:
      - http:
          paths:
          - path: /foo
            backend:
              serviceName: httpbin
              servicePort: 80
    
    kubectl apply -f httpbin-ingress-2.yaml

测试一下，需要API Key。

    curl -i http://192.168.1.55:32080/foo/status/200
    HTTP/1.1 401 Unauthorized
    Date: Tue, 21 Jan 2020 12:48:28 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    WWW-Authenticate: Key realm="kong"
    Content-Length: 41
    Server: kong/1.3.0
    
    {"message":"No API key found in request"}

需要提供一个KongConsumer。

    vi twingao-consumer.yaml
    ---
    apiVersion: configuration.konghq.com/v1
    kind: KongConsumer
    metadata:
      name: twingao-consumer
    username: twingao
    
    kubectl apply -f twingao-consumer.yaml

新建KongCredential，并和KongConsumer和KongPlugin关联。

    apiVersion: configuration.konghq.com/v1
    kind: KongCredential
    metadata:
      name: twingao-apikey
    consumerRef: twingao-consumer
    type: key-auth
    config:
      key: hello-api

    kubectl apply -f twingao-apikey.yaml

测试一下,访问httpbin服务需要提供API Key进行验证。不携带或者携带错误的API Key都验证不通过，返回401。

    curl -i -H 'apikey: hello-api' http://192.168.1.55:32080/foo/status/200
    HTTP/1.1 200 OK
    Content-Type: text/html; charset=utf-8
    Content-Length: 0
    Connection: keep-alive
    Server: gunicorn/19.9.0
    Date: Tue, 21 Jan 2020 14:15:40 GMT
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    X-Kong-Upstream-Latency: 8
    X-Kong-Proxy-Latency: 44
    Via: kong/1.3.0
    
    curl -i -H 'apikey: hello' http://192.168.1.55:32080/foo/status/200
    HTTP/1.1 401 Unauthorized
    Date: Tue, 21 Jan 2020 14:17:05 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    {"message":"Invalid authentication credentials"}

    curl -i http://192.168.1.55:32080/foo/status/200
    HTTP/1.1 401 Unauthorized
    Date: Tue, 21 Jan 2020 14:17:43 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    WWW-Authenticate: Key realm="kong"
    Content-Length: 41
    Server: kong/1.3.0
    
    {"message":"No API key found in request"}