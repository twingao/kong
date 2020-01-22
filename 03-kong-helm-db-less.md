# Kong系列-03-Helm安装Kong 1.3.0 DB-less with Ingress Controller

系统环境。

    kubeadm version
    kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.4", GitCommit:"224be7bdce5a9dd0c2fd0d46b83865648e2fe0ba", GitTreeState:"clean", BuildDate:"2019-12-11T12:44:45Z", GoVersion:"go1.12.12", Compiler:"gc", Platform:"linux/amd64"}
    
    helm version
    version.BuildInfo{Version:"v3.0.1", GitCommit:"7c22ef9ce89e0ebeb7125ba2ebf7d421f3e82ffa", GitTreeState:"clean", GoVersion:"go1.13.4"}
    
    helm repo list
    NAME            URL
    stable          https://kubernetes-charts.storage.googleapis.com
    aliyuncs        https://apphub.aliyuncs.com
    bitnami         https://charts.bitnami.com/bitnami

查找Kong Chart，并安装。

    helm search repo kong
    NAME            CHART VERSION   APP VERSION     DESCRIPTION
    aliyuncs/kong   0.27.0          1.3             The Cloud-Native Ingress and Service Mesh for A...
    stable/kong     0.28.0          1.3             The Cloud-Native Ingress and Service Mesh for A...
    
    helm install gateway stable/kong --version 0.28.0 \
        --set admin.useTLS=false \
        --set admin.nodePort=32444 \
        --set proxy.http.nodePort=32080 \
        --set proxy.tls.nodePort=32443 \
        --set replicaCount=2
    NAME: gateway
    LAST DEPLOYED: Sun Dec 15 13:18:30 2019
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    NOTES:
    1. Kong Admin can be accessed inside the cluster using:
         DNS=gateway-kong-admin.default.svc.cluster.local
         PORT=8444
    
    To connect from outside the K8s cluster:
         HOST=$(kubectl get nodes --namespace default -o jsonpath='{.items[0].status                                                                                                     .addresses[0].address}')
         PORT=$(kubectl get svc --namespace default gateway-kong-admin -o jsonpath='                                                                                                     {.spec.ports[0].nodePort}')
    
    
    2. Kong Proxy can be accessed inside the cluster using:
         DNS=gateway-kong-proxy.default.svc.cluster.localPORT=443To connect from out                                                                                                     side the K8s cluster:
         HOST=$(kubectl get nodes --namespace default -o jsonpath='{.items[0].status                                                                                                     .addresses[0].address}')
         PORT=$(kubectl get svc --namespace default gateway-kong-proxy -o jsonpath='                                                                                                     {.spec.ports[0].nodePort}')

查看Kubernetes各资源状态。

    kubectl get all
    NAME                                READY   STATUS    RESTARTS   AGE
    pod/gateway-kong-67fb7768ff-c6wg6   2/2     Running   2          6m59s
    pod/gateway-kong-67fb7768ff-qczw5   2/2     Running   1          6m59s
    
    NAME                         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
    service/gateway-kong-admin   NodePort    10.1.31.119   <none>        8444:32444/TCP               7m
    service/gateway-kong-proxy   NodePort    10.1.244.45   <none>        80:32080/TCP,443:32443/TCP   7m
    service/kubernetes           ClusterIP   10.1.0.1      <none>        443/TCP                      15h
    
    NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/gateway-kong   2/2     2            2           7m
    
    NAME                                      DESIRED   CURRENT   READY   AGE
    replicaset.apps/gateway-kong-67fb7768ff   2         2         2       6m59s

在Kubernetes下部署Kong，Kong定义了多个自定义资源CRDs，Kong通过这些CRDs定义路由、插件等配置数据，这些配置数据通过Kubernetes API转发给Kong Ingress Controller，并转发给Kong数据平面，Kong继而控制客户端的请求流量。

CRDs数据应该存放在Kubernetes的etcd数据库中，这些数据通过Ingress Controller转换为Kong的配置数据，Kong数据平面缓存这些配置数据。从上面也可以看出Kong Pod中有两个容器：ingress-controller、kong。

![dbless deployment](images/dbless-deployment.png)

Kong的admin服务对外的nodePort为32444，proxy服务对外nodePort为32080(http)/32443(https)，如果都能访问，表示Kong安装成功。

    curl http://192.168.1.55:32444 -s | python -m json.tool
    {
        "configuration": {
            "admin_acc_logs": "/usr/local/kong/logs/admin_access.log",
            "admin_access_log": "/dev/stdout",
            "admin_error_log": "/dev/stderr",
            "admin_listen": [
                "0.0.0.0:8444"
            ],
            "admin_listeners": [
                {
                    "bind": false,
                    "deferred": false,
                    "http2": false,
                    "ip": "0.0.0.0",
                    "listener": "0.0.0.0:8444",
                    "port": 8444,
                    "proxy_protocol": false,
                    "reuseport": false,
                    "ssl": false,
                    "transparent": false
                }
            ],
            "admin_ssl_cert_default": "/usr/local/kong/ssl/admin-kong-default.crt",
            "admin_ssl_cert_key_default": "/usr/local/kong/ssl/admin-kong-default.key",
            "admin_ssl_enabled": false,
            "anonymous_reports": true,
            "cassandra_consistency": "ONE",
            "cassandra_contact_points": [
                "127.0.0.1"
            ],
            "cassandra_data_centers": [
                "dc1:2",
                "dc2:3"
            ],
            "cassandra_keyspace": "kong",
            "cassandra_lb_policy": "RequestRoundRobin",
            "cassandra_port": 9042,
            "cassandra_repl_factor": 1,
            "cassandra_repl_strategy": "SimpleStrategy",
            "cassandra_schema_consensus_timeout": 10000,
            "cassandra_ssl": false,
            "cassandra_ssl_verify": false,
            "cassandra_timeout": 5000,
            "cassandra_username": "kong",
            "client_body_buffer_size": "8k",
            "client_max_body_size": "0",
            "client_ssl": false,
            "client_ssl_cert_default": "/usr/local/kong/ssl/kong-default.crt",
            "client_ssl_cert_key_default": "/usr/local/kong/ssl/kong-default.key",
            "database": "off",
            "db_cache_ttl": 0,
            "db_cache_warmup_entities": [
                "services",
                "plugins"
            ],
            "db_resurrect_ttl": 30,
            "db_update_frequency": 5,
            "db_update_propagation": 0,
            "dns_error_ttl": 1,
            "dns_hostsfile": "/etc/hosts",
            "dns_no_sync": false,
            "dns_not_found_ttl": 30,
            "dns_order": [
                "LAST",
                "SRV",
                "A",
                "CNAME"
            ],
            "dns_resolver": {},
            "dns_stale_ttl": 4,
            "enabled_headers": {
                "Server": true,
                "Via": true,
                "X-Kong-Proxy-Latency": true,
                "X-Kong-Upstream-Latency": true,
                "X-Kong-Upstream-Status": false,
                "latency_tokens": true,
                "server_tokens": true
            },
            "error_default_type": "text/plain",
            "headers": [
                "server_tokens",
                "latency_tokens"
            ],
            "kong_env": "/usr/local/kong/.kong_env",
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
            "log_level": "notice",
            "lua_package_cpath": "",
            "lua_package_path": "/opt/?.lua;;",
            "lua_socket_pool_size": 30,
            "lua_ssl_verify_depth": 1,
            "mem_cache_size": "128m",
            "nginx_acc_logs": "/usr/local/kong/logs/access.log",
            "nginx_admin_directives": {},
            "nginx_conf": "/usr/local/kong/nginx.conf",
            "nginx_daemon": "off",
            "nginx_err_logs": "/usr/local/kong/logs/error.log",
            "nginx_http_directives": [
                {
                    "name": "ssl_protocols",
                    "value": "TLSv1.1 TLSv1.2 TLSv1.3"
                },
                {
                    "name": "include",
                    "value": "/kong/servers.conf"
                },
                {
                    "name": "lua_shared_dict",
                    "value": "prometheus_metrics 5m"
                }
            ],
            "nginx_http_include": "/kong/servers.conf",
            "nginx_http_ssl_protocols": "TLSv1.1 TLSv1.2 TLSv1.3",
            "nginx_http_upstream_directives": [
                {
                    "name": "keepalive_timeout",
                    "value": "60s"
                },
                {
                    "name": "keepalive_requests",
                    "value": "100"
                },
                {
                    "name": "keepalive",
                    "value": "60"
                }
            ],
            "nginx_http_upstream_keepalive": "60",
            "nginx_http_upstream_keepalive_requests": "100",
            "nginx_http_upstream_keepalive_timeout": "60s",
            "nginx_kong_conf": "/usr/local/kong/nginx-kong.conf",
            "nginx_kong_stream_conf": "/usr/local/kong/nginx-kong-stream.conf",
            "nginx_optimizations": true,
            "nginx_pid": "/usr/local/kong/pids/nginx.pid",
            "nginx_proxy_directives": {},
            "nginx_sproxy_directives": {},
            "nginx_stream_directives": {},
            "nginx_worker_processes": "1",
            "origins": {},
            "pg_database": "kong",
            "pg_host": "127.0.0.1",
            "pg_max_concurrent_queries": 0,
            "pg_port": 5432,
            "pg_semaphore_timeout": 60000,
            "pg_ssl": false,
            "pg_ssl_verify": false,
            "pg_timeout": 5000,
            "pg_user": "kong",
            "plugins": [
                "bundled"
            ],
            "prefix": "/usr/local/kong",
            "proxy_access_log": "/dev/stdout",
            "proxy_error_log": "/dev/stderr",
            "proxy_listen": [
                "0.0.0.0:8000",
                "0.0.0.0:8443 ssl"
            ],
            "proxy_listeners": [
                {
                    "bind": false,
                    "deferred": false,
                    "http2": false,
                    "ip": "0.0.0.0",
                    "listener": "0.0.0.0:8000",
                    "port": 8000,
                    "proxy_protocol": false,
                    "reuseport": false,
                    "ssl": false,
                    "transparent": false
                },
                {
                    "bind": false,
                    "deferred": false,
                    "http2": false,
                    "ip": "0.0.0.0",
                    "listener": "0.0.0.0:8443 ssl",
                    "port": 8443,
                    "proxy_protocol": false,
                    "reuseport": false,
                    "ssl": true,
                    "transparent": false
                }
            ],
            "proxy_ssl_enabled": true,
            "real_ip_header": "X-Real-IP",
            "real_ip_recursive": "off",
            "router_consistency": "strict",
            "ssl_cert": "/usr/local/kong/ssl/kong-default.crt",
            "ssl_cert_csr_default": "/usr/local/kong/ssl/kong-default.csr",
            "ssl_cert_default": "/usr/local/kong/ssl/kong-default.crt",
            "ssl_cert_key": "/usr/local/kong/ssl/kong-default.key",
            "ssl_cert_key_default": "/usr/local/kong/ssl/kong-default.key",
            "ssl_cipher_suite": "modern",
            "ssl_ciphers": "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256",
            "ssl_preread_enabled": true,
            "stream_listen": [
                "off"
            ],
            "stream_listeners": {},
            "trusted_ips": {},
            "upstream_keepalive": 60
        },
        "hostname": "gateway-kong-67fb7768ff-qczw5",
        "lua_version": "LuaJIT 2.1.0-beta3",
        "node_id": "fd1b5c61-fd0d-4f38-9450-4e25f6a18310",
        "plugins": {
            "available_on_server": {
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
            "enabled_in_cluster": []
        },
        "prng_seeds": {
            "pid: 1": 216228151213,
            "pid: 31": 112106812471
        },
        "tagline": "Welcome to kong",
        "timers": {
            "pending": 5,
            "running": 0
        },
        "version": "1.3.0"
    }
    
    curl -i http://192.168.1.55:32080
    HTTP/1.1 404 Not Found
    Date: Sun, 15 Dec 2019 08:33:53 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    {"message":"no Route matched with those values"}
    
    curl -i -k https://192.168.1.55:32443
    HTTP/1.1 404 Not Found
    Date: Sun, 15 Dec 2019 08:34:02 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    Server: kong/1.3.0
    
    {"message":"no Route matched with those values"}

安装完毕。