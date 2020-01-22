# Kong系列-06-代理路由与匹配

### 代理协议

Kong支持http/https、tcp/tls和grpc/grpcs协议的代理。

    http: methods, hosts, headers, paths (and snis, if https)
    tcp: sources, destinations (and snis, if tls)
    grpc: hosts, headers, paths (and snis, if grpcs)

路由匹配

- 请求必须包括所有配置属性（and）
- 请求中必须至少匹配一个属性值（or）

    {
        "hosts": ["example.com", "foo-service.com"],
        "paths": ["/foo", "/bar"],
        "methods": ["GET"]
    }

以下的请求能够匹配。

    GET /foo HTTP/1.1
    Host: example.com
    
    GET /bar HTTP/1.1
    Host: foo-service.com
    
    GET /foo/hello/world HTTP/1.1
    Host: example.com

以下的请求无法匹配。

    GET / HTTP/1.1
    Host: example.com
    
    POST /foo HTTP/1.1
    Host: example.com
    
    GET /foo HTTP/1.1
    Host: foo.com

举例说明。

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
    
    kubectl apply -f echo-server-service.yaml

创建service和routes。

    curl -X POST \
      --url http://192.168.1.55:32444/services/ \
      --data 'name=echo-service' \
      --data 'url=http://echo:8080' \
      -s | python -m json.tool
    {
        "client_certificate": null,
        "connect_timeout": 60000,
        "created_at": 1576925889,
        "host": "echo",
        "id": "6160de1d-0d86-4b1a-b317-66f422e02780",
        "name": "echo-service",
        "path": null,
        "port": 8080,
        "protocol": "http",
        "read_timeout": 60000,
        "retries": 5,
        "tags": null,
        "updated_at": 1576925889,
        "write_timeout": 60000
    }
    
    curl -X POST \
      --url http://192.168.1.55:32444/services/echo-service/routes \
      -H 'Content-Type: application/json' \
      --data \
        '{
            "name":"echo-service-route",
            "hosts":["example.com","foo-service.com"],
            "paths":["/foo", "/bar"],
            "methods":["GET"]
         }' \
      -s | python -m json.tool
    
    或
    
    curl -X POST \
      --url http://192.168.1.55:32444/services/echo-service/routes \
      --data 'name=echo-service-route' \
      --data 'hosts[]=example.com' \
      --data 'hosts[]=foo-service.com' \
      --data 'paths[]=/foo' \
      --data 'paths[]=/bar' \
      --data 'methods[]=GET' \
      -s | python -m json.tool
    
    {
        "created_at": 1576925902,
        "destinations": null,
        "headers": null,
        "hosts": [
            "example.com",
            "foo-service.com"
        ],
        "https_redirect_status_code": 426,
        "id": "f38e0dcf-8801-46c8-bfba-3f7efcd6c92c",
        "methods": [
            "GET"
        ],
        "name": "echo-service-route",
        "paths": [
            "/foo",
            "/bar"
        ],
        "preserve_host": false,
        "protocols": [
            "http",
            "https"
        ],
        "regex_priority": 0,
        "service": {
            "id": "6160de1d-0d86-4b1a-b317-66f422e02780"
        },
        "snis": null,
        "sources": null,
        "strip_path": true,
        "tags": null,
        "updated_at": 1576925902
    }

测试一下效果。

    curl -i -X GET \
      --url http://192.168.1.55:32080/foo \
      -H "Host: example.com"
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Sat, 21 Dec 2019 11:05:34 GMT
    Server: echoserver
    X-Kong-Upstream-Latency: 5
    X-Kong-Proxy-Latency: 86
    Via: kong/1.3.0
    
    curl -I -X GET \
      --url http://192.168.1.55:32080/bar/hello/world \
      -H "Host: foo-service.com"
    HTTP/1.1 200 OK
    Content-Type: text/plain; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Date: Sat, 21 Dec 2019 11:08:08 GMT
    Server: echoserver
    X-Kong-Upstream-Latency: 2
    X-Kong-Proxy-Latency: 2
    Via: kong/1.3.0
    
    curl -I -X GET \
      --url http://192.168.1.55:32080/ \
      -H "Host: foo-service.com"
    HTTP/1.1 404 Not Found
    Date: Sat, 21 Dec 2019 11:09:20 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    curl -I -X GET \
      --url http://192.168.1.55:32080/foo \
      -H "Host: foo.com"
    HTTP/1.1 404 Not Found
    Date: Sat, 21 Dec 2019 11:10:02 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0

### hosts

支持通配符，通配符只能在主机名的最左边或者最右边。

    {
        "hosts": ["*.example.com", "service*"]
    }

### preserve_host

Kong的默认行为是将上游请求host请求头设置为service的host中指定的主机名，preserve_host: true时将客户端请求时host上传给上游服务。

    {
        "hosts": ["service.com"],
        "preserve_host": true,
        "service": {
            "id": "..."
        }
    }

### paths

paths支持正则表达式。

regex_priority：正则表达式优先级。

strip_path：指定一个路径前缀来匹配某个路由，但不要将其包含在上游请求中。

    [
        {
            "paths": ["/status/\d+"],
            "regex_priority": 0
        },
        {
            "paths": ["/version/\d+/status/\d+"],
            "regex_priority": 6
        },
        {
            "paths": ["/version"],
        },
        {
            "paths": ["/version/any/"],
        }
    ]

### methods

可以有多个值，可以为空。

    {
        "methods": ["GET", "HEAD"],
        "service": {
            "id": "..."
        }
    }

### headers

除了host之外的其它header，可以有多个值，可以为空。

    {
        "headers": { "version": ["v1", "v2"] },
        "service": {
            "id": "..."
        }
    }

### sources和destinations

源sources属性和目的destinations属性仅适用于tcp/tls路由，可以通过源和目的IP和/或端口匹配路由。

    {
        "protocols": ["tcp", "tls"],
        "sources": [{"ip":"10.1.0.0/16", "port":1234}, {"ip":"10.2.2.2"}, {"port":9123}],
        "id": "...",
    }

### 后备路由

如果没有路由能否匹配，Kong网关将返回HTTP 404，可以通过配置后备路由，转发到统一的上游服务器中处理404错误。

    {
        "paths": ["/"],
        "service": {
            "id": "..."
        }
    }

### SSL路由

Kong提供一种根据每个连接动态提供SSL证书的方法，SSL证书由核心直接处理，通过管理API进行配置。

    curl -i -X POST http://localhost:8001/certificates \
      -F "cert=@/path/to/cert.pem" \
      -F "key=@/path/to/cert.key" \
      -F "snis=*.ssl-example.com,other-ssl-example.com"
    HTTP/1.1 201 Created
    ...

### WebSocket和TLS路由

### gRPC路由