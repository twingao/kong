# Kong系列-02-CentOS 7下Kong 1.4.2安装

先安装PostgreSQL，请参见[CentOS下PostgreSQL 12 安装](../postgresql/postgresql-12-centos-installation.md)。然后为Kong初始化数据库。

    su - postges
    su: user postges does not exist
    [root@kong postgresql]# su - postgres
    上一次登录：日 12月 15 08:17:06 CST 2019pts/0 上
    -bash-4.2$ psql
    用户 postgres 的口令：
    psql (12.1)
    输入 "help" 来获取帮助信息.
    
    postgres=# CREATE USER kong WITH PASSWORD '1111';
    CREATE ROLE
    postgres=# CREATE DATABASE kong OWNER kong;
    CREATE DATABASE
    postgres=# GRANT ALL PRIVILEGES ON DATABASE kong TO kong;
    GRANT
    postgres=# \q
    -bash-4.2$ psql -U kong -d kong -h 127.0.0.1 -p 5432
    用户 kong 的口令：
    psql (12.1)
    输入 "help" 来获取帮助信息.
    
    kong=> \q
    -bash-4.2$ exit
    登出

下载rpm包，[https://bintray.com/kong/kong-rpm/centos/view/files/centos/7#files/centos/7](https://bintray.com/kong/kong-rpm/centos/view/files/centos/7#files/centos/7)。


安装所需工具。

    yum install -y gcc pcre-devel zlib-devel openssl-devel
    
    vi /etc/security/limits.conf
    *                soft    nofile          4096
    
    #重启生效
    reboot

安装Kong。

    mkdir kong
    cd kong
    
    #将下载的文件上传到此目录。
    
    yum install -y epel-release
    yum install -y kong-1.4.2.el7.amd64.rpm --nogpgcheck

配置Kong。

    cp /etc/kong/kong.conf.default /etc/kong/kong.conf
    
    vi /etc/kong/kong.conf
    ......
    admin_listen = 0.0.0.0:8001, 0.0.0.0:8444 ssl
    ......
    database = postgres              # Determines which of PostgreSQL or Cassandra
                                     # this node will use as its datastore.
                                     # Accepted values are `postgres`,
                                     # `cassandra`, and `off`.
    
    pg_host = 127.0.0.1              # Host of the Postgres server.
    pg_port = 5432                   # Port of the Postgres server.
    #pg_timeout = 5000               # Defines the timeout (in ms), for connecting,
                                     # reading and writing.
    
    pg_user = kong                   # Postgres user.
    pg_password = 1111               # Postgres user's password.
    pg_database = kong               # The database name to connect to.
    ......
    
    kong check
    configuration at /etc/kong/kong.conf is valid

初始化Kong，Kong会向数据库写入初始化数据。

    kong migrations bootstrap

启动Kong。

    kong start
    Kong started
    
    kong health
    nginx.......running
    
    Kong is healthy at /usr/local/kong

Kong的8001为管理端口，8000为Proxy端口，如果都能访问，表示Kong安装成功。

    curl http://localhost:8001 -s | python -m json.tool
    {
        "configuration": {
            "admin_acc_logs": "/usr/local/kong/logs/admin_access.log",
            "admin_access_log": "logs/admin_access.log",
            "admin_error_log": "logs/error.log",
            "admin_listen": [
                "0.0.0.0:8001",
                "0.0.0.0:8444 ssl"
            ],
            "admin_listeners": [
                {
                    "bind": false,
                    "deferred": false,
                    "http2": false,
                    "ip": "0.0.0.0",
                    "listener": "0.0.0.0:8001",
                    "port": 8001,
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
                    "listener": "0.0.0.0:8444 ssl",
                    "port": 8444,
                    "proxy_protocol": false,
                    "reuseport": false,
                    "ssl": true,
                    "transparent": false
                }
            ],
            "admin_ssl_cert": "/usr/local/kong/ssl/admin-kong-default.crt",
            "admin_ssl_cert_default": "/usr/local/kong/ssl/admin-kong-default.crt",
            "admin_ssl_cert_key": "/usr/local/kong/ssl/admin-kong-default.key",
            "admin_ssl_cert_key_default": "/usr/local/kong/ssl/admin-kong-default.key",
            "admin_ssl_enabled": true,
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
            "cassandra_refresh_frequency": 60,
            "cassandra_repl_factor": 1,
            "cassandra_repl_strategy": "SimpleStrategy",
            "cassandra_schema_consensus_timeout": 60000,
            "cassandra_ssl": false,
            "cassandra_ssl_verify": false,
            "cassandra_timeout": 60000,
            "cassandra_username": "kong",
            "client_body_buffer_size": "8k",
            "client_max_body_size": "0",
            "client_ssl": false,
            "client_ssl_cert_default": "/usr/local/kong/ssl/kong-default.crt",
            "client_ssl_cert_key_default": "/usr/local/kong/ssl/kong-default.key",
            "database": "postgres",
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
                "X-Kong-Admin-Latency": true,
                "X-Kong-Proxy-Latency": true,
                "X-Kong-Response-Latency": true,
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
            "lua_package_path": "./?.lua;./?/init.lua;",
            "lua_socket_pool_size": 30,
            "lua_ssl_verify_depth": 1,
            "mem_cache_size": "128m",
            "nginx_acc_logs": "/usr/local/kong/logs/access.log",
            "nginx_admin_directives": {},
            "nginx_conf": "/usr/local/kong/nginx.conf",
            "nginx_daemon": "on",
            "nginx_err_logs": "/usr/local/kong/logs/error.log",
            "nginx_http_directives": [
                {
                    "name": "ssl_protocols",
                    "value": "TLSv1.1 TLSv1.2 TLSv1.3"
                },
                {
                    "name": "lua_shared_dict",
                    "value": "prometheus_metrics 5m"
                }
            ],
            "nginx_http_ssl_protocols": "TLSv1.1 TLSv1.2 TLSv1.3",
            "nginx_http_status_directives": {},
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
            "nginx_worker_processes": "auto",
            "origins": {},
            "pg_database": "kong",
            "pg_host": "127.0.0.1",
            "pg_max_concurrent_queries": 0,
            "pg_password": "******",
            "pg_port": 5432,
            "pg_semaphore_timeout": 60000,
            "pg_ssl": false,
            "pg_ssl_verify": false,
            "pg_timeout": 60000,
            "pg_user": "kong",
            "plugins": [
                "bundled"
            ],
            "prefix": "/usr/local/kong",
            "proxy_access_log": "logs/access.log",
            "proxy_error_log": "logs/error.log",
            "proxy_listen": [
                "0.0.0.0:8000",
                "0.0.0.0:8443 http2 ssl"
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
                    "http2": true,
                    "ip": "0.0.0.0",
                    "listener": "0.0.0.0:8443 ssl http2",
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
            "router_update_frequency": 1,
            "service_mesh": false,
            "ssl_cert": "/usr/local/kong/ssl/kong-default.crt",
            "ssl_cert_csr_default": "/usr/local/kong/ssl/kong-default.csr",
            "ssl_cert_default": "/usr/local/kong/ssl/kong-default.crt",
            "ssl_cert_key": "/usr/local/kong/ssl/kong-default.key",
            "ssl_cert_key_default": "/usr/local/kong/ssl/kong-default.key",
            "ssl_cipher_suite": "modern",
            "ssl_ciphers": "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256",
            "ssl_preread_enabled": true,
            "status_access_log": "off",
            "status_error_log": "logs/status_error.log",
            "status_listen": [
                "off"
            ],
            "status_listeners": {},
            "stream_listen": [
                "off"
            ],
            "stream_listeners": {},
            "trusted_ips": {},
            "upstream_keepalive": 60
        },
        "hostname": "kong",
        "lua_version": "LuaJIT 2.1.0-beta3",
        "node_id": "ac7df796-7fb4-4e89-86d4-3f8ad5a93f6e",
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
            "pid: 1272": 115238374918,
            "pid: 1284": 559859622153
        },
        "tagline": "Welcome to kong",
        "timers": {
            "pending": 6,
            "running": 0
        },
        "version": "1.4.2"
    }
    
    curl -i http://localhost:8000
    HTTP/1.1 404 Not Found
    Date: Sun, 15 Dec 2019 04:33:54 GMT
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    X-Kong-Response-Latency: 2
    Server: kong/1.4.2
    
    {"message":"no Route matched with those values"}