# Kong系列-05-使用入门

Kong是一个API网关，其核心能力是代理客户端对上游服务的访问，下面我们演示一下如何配置Kong来进行代理服务。

Kong传统是通过Admin API进行管理的，对于Kong直接在操作系统如CentOS之上直接部署时，Kong的8001为管理端口，8000为Proxy端口；如果在Kubernetes集群部署，gateway-kong-admin服务提供管理接口,gateway-kong-proxy服务提供了代理接口。

一般来说，在学习Kong Ingress Controller配置方式前，建议先学习一下Admin API的基本使用。本文先简单介绍Admin API管理方式。

Kong的核心实体概念。

- Client：指下游客户端。
- Service：指上游服务。
- Route：定义匹配客户端请求的规则，每个路由都与一个服务相关联，而服务可能有多个与之相关联的路由。
- Plugin：它是在代理生命周期中运行的业务逻辑，插件可以生效于全局或者特定的路由和服务。
- Consumer：消费者表示服务的消费者或者使用者，可能是一个用户，也可能是一个应用。

本文中我们在[Kong系列-04-Helm安装Kong 1.3.0 with PostgreSQL and with Ingress Controller](04-kong-helm-postresql.md)环境下演示。注意无数据DB-Less部署方式是不支持Admin API管理的。

    kubectl get svc
    NAME                                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/gateway-kong-admin            NodePort    10.1.6.70      <none>        8444:32444/TCP               4m24s
    service/gateway-kong-proxy            NodePort    10.1.232.237   <none>        80:32080/TCP,443:32443/TCP   4m24s

我们先部署一个echo服务。该服务没有对Kubernetes外暴露端口。如果想从Kubernetes集群之外访问该服务就需要通过Kong网关代理。

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
      - port: 80
        name: low
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
      replicas: 1
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

为了让Kong感知k8s的echo服务，需要在Kong中创建一个Kong的服务echo-service，该Kong服务指向k8s的echo服务。

    curl -X POST \
      --url http://192.168.1.55:32444/services/ \
      --data 'name=echo-service' \
      --data 'url=http://echo:8080' \
      -s | python -m json.tool
    {
        "client_certificate": null,
        "connect_timeout": 60000,
        "created_at": 1576504925,
        "host": "echo",
        "id": "f15edf62-cb68-4279-a41f-7f5273ccd001",
        "name": "echo-service",
        "path": null,
        "port": 8080,
        "protocol": "http",
        "read_timeout": 60000,
        "retries": 5,
        "tags": null,
        "updated_at": 1576504925,
        "write_timeout": 60000
    }

查询服务services列表，可以看出刚才创建的服务echo-service。

    curl -X GET \
      --url http://192.168.1.55:32444/services/ \
      -s | python -m json.tool
    {
        "data": [
            {
                "client_certificate": null,
                "connect_timeout": 60000,
                "created_at": 1576504925,
                "host": "echo",
                "id": "f15edf62-cb68-4279-a41f-7f5273ccd001",
                "name": "echo-service",
                "path": null,
                "port": 8080,
                "protocol": "http",
                "read_timeout": 60000,
                "retries": 5,
                "tags": null,
                "updated_at": 1576504925,
                "write_timeout": 60000
            }
        ],
        "next": null
    }

为了让Kong知道客户端的哪些请求是访问echo服务的，需要为echo服务添加一条路由。每个服务是可以有多个路由与之对应。以下的路由规则为路径前缀为/foo的请求，将被转发到echo服务。

    curl -X POST \
      --url http://192.168.1.55:32444/services/echo-service/routes \
      --data 'name=echo-service-route' \
      --data 'paths[]=/foo' \
      -s | python -m json.tool
    {
        "created_at": 1576505187,
        "destinations": null,
        "headers": null,
        "hosts": null,
        "https_redirect_status_code": 426,
        "id": "d152f741-9ed1-47d2-98bc-f8c438075e2e",
        "methods": null,
        "name": "echo-service-route",
        "paths": [
            "/foo"
        ],
        "preserve_host": false,
        "protocols": [
            "http",
            "https"
        ],
        "regex_priority": 0,
        "service": {
            "id": "f15edf62-cb68-4279-a41f-7f5273ccd001"
        },
        "snis": null,
        "sources": null,
        "strip_path": true,
        "tags": null,
        "updated_at": 1576505187
    }

测试一下，可以看出根据路由规则能够访问echo服务。

    curl -iX GET --url http://192.168.1.55:32080/foo
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Mon, 16 Dec 2019 14:09:21 GMT
    Server: echoserver
    X-Kong-Upstream-Latency: 5
    X-Kong-Proxy-Latency: 65
    Via: kong/1.3.0
    
    
    Hostname: echo-75cf96d976-pwbg7
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-75cf96d976-pwbg7
            pod namespace:  default
            pod IP: 10.244.2.12
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.11
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo:8080/
    
    Request Headers:
            accept=*/*
            connection=keep-alive
            host=echo:8080
            user-agent=curl/7.29.0
            x-forwarded-for=10.244.0.0
            x-forwarded-host=192.168.1.55
            x-forwarded-port=8000
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
    
    Request Body:
            -no body in request-

Kong通过插件扩展其网关能力，Kong的一个很大的优势就是有功能丰富的开源插件，并且能方便地自定义插件。

key-auth插件是一个身份验证插件，在添加此插件之前，对服务的所有请求都会被直接转发到上游服务，如果添加并配置此插件后，只有正确密钥的请求才会被转发到上游服务，否则将被拒绝。以下API给echo-service应用了key-auth插件，意味所有对echo-service的请求都会被key-auth插件处理。

    curl -X POST \
      --url http://192.168.1.55:32444/services/echo-service/plugins \
      --data 'name=key-auth' \
      -s | python -m json.tool
    {
        "config": {
            "anonymous": null,
            "hide_credentials": false,
            "key_in_body": false,
            "key_names": [
                "apikey"
            ],
            "run_on_preflight": true
        },
        "consumer": null,
        "created_at": 1576674272,
        "enabled": true,
        "id": "0d8231dd-e820-45b4-bc78-f8b9758346fa",
        "name": "key-auth",
        "protocols": [
            "grpc",
            "grpcs",
            "http",
            "https"
        ],
        "route": null,
        "run_on": "first",
        "service": {
            "id": "f15edf62-cb68-4279-a41f-7f5273ccd001"
        },
        "tags": null
    }

此时请求echo-service，由于没有携带key，key-auth插件拒绝请求。

    curl -iX GET --url http://192.168.1.55:32080/foo
    HTTP/1.1 401 Unauthorized
    Date: Wed, 18 Dec 2019 13:08:18 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    WWW-Authenticate: Key realm="kong"
    Content-Length: 41
    Server: kong/1.3.0
    
    {"message":"No API key found in request"}

创建一个消费者consumer。

    curl -X POST \
      --url http://192.168.1.55:32444/consumers \
      --data 'username=twingao' \
      -s | python -m json.tool
    {
        "created_at": 1576674684,
        "custom_id": null,
        "id": "5b6be948-00bb-4479-a012-67d73824c2fe",
        "tags": null,
        "username": "twingao"
    }

为消费者twingao配置一个key。

    curl -X POST \
      --url http://192.168.1.55:32444/consumers/twingao/key-auth \
      --data 'key=123456' \
      -s | python -m json.tool
    {
        "consumer": {
            "id": "5b6be948-00bb-4479-a012-67d73824c2fe"
        },
        "created_at": 1576674825,
        "id": "0c815096-4cda-4056-824a-dff6eb799384",
        "key": "123456"
    }

此时访问echo-service，携带上请求头apikey: 123456，key-auth身份验证成功，会放行请求；如果携带的apikey不正确，则不会放行请求。

    curl -iX GET --url http://192.168.1.55:32080/foo --header 'apikey: 123456'
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Wed, 18 Dec 2019 13:19:22 GMT
    Server: echoserver
    X-Kong-Upstream-Latency: 7
    X-Kong-Proxy-Latency: 66
    Via: kong/1.3.0
    
    
    Hostname: echo-75cf96d976-pwbg7
    
    Pod Information:
            node name:      k8s-node2
            pod name:       echo-75cf96d976-pwbg7
            pod namespace:  default
            pod IP: 10.244.2.14
    
    Server values:
            server_version=nginx: 1.14.2 - lua: 10015
    
    Request Information:
            client_address=10.244.2.13
            method=GET
            real path=/
            query=
            request_version=1.1
            request_scheme=http
            request_uri=http://echo:8080/
    
    Request Headers:
            accept=*/*
            apikey=123456
            connection=keep-alive
            host=echo:8080
            user-agent=curl/7.29.0
            x-consumer-id=5b6be948-00bb-4479-a012-67d73824c2fe
            x-consumer-username=twingao
            x-forwarded-for=10.244.0.0
            x-forwarded-host=192.168.1.55
            x-forwarded-port=8000
            x-forwarded-proto=http
            x-real-ip=10.244.0.0
    
    Request Body:
            -no body in request-
    
    curl -iX GET --url http://192.168.1.55:32080/foo --header 'apikey: 123'
    HTTP/1.1 401 Unauthorized
    Date: Wed, 18 Dec 2019 13:19:57 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    {"message":"Invalid authentication credentials"}