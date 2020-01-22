# Kong系列-08-无数据库DB-less

### Kong无数据库介绍

Kong在1.1.0版本（2019-03-27）开始支持无数据库模式，只需要将kong.conf中的配置项database = off即可启用无数据库方式。

- 无数据库采用声明方式定义实体，所有的实体都配置在其中。一旦该文件加载到Kong中，它将替换整个配置。当需要增量更改时，将对声明性配置文件进行更改，然后将其全部重新加载。
- 由于没有中心数据库进行协调，多个Kong节点是完全彼此独立。
- 由于配置实体的唯一方式是通过声明性配置，管理API的CRUD操作只支持只读方法，即GET方法，不支持POST、DELETE等方法。

无数据库加载实体声明的方式：

- 在kong.conf配置文件中配置declarative_config = /etc/kong/kong.yaml
- 调用管理API，一次性加载声明配置。

由于没有数据库，有些Kong的插件将不再能够使用，有些插件能够部分使用。

- 兼容的插件


    bot-detection
    correlation-id
    cors
    datadog
    file-log
    http-log
    tcp-log
    udp-log
    syslog
    ip-restriction
    prometheus
    zipkin
    request-transformer
    response-transformer
    request-termination

- 部分兼容的插件


    acl
    basic-auth
    hmac-auth
    jwt
    key-auth
    rate-limiting
    response-ratelimiting
    key-auth

- 不兼容的插件


    oauth2

### Kong无数据库配置

将Kong改为无数据库模式。

    vi /etc/kong//kong.conf
    database = off                   # Determines which of PostgreSQL or Cassandra
                                     # this node will use as its datastore.
                                     # Accepted values are `postgres`,
                                     # `cassandra`, and `off`.

- 声明配置文件方式

前面说过，可以在声明性配置文件中定义配置数据，Kong会一次性加载到内存。我们先需要一个kong.yml文件，可以通过一个初始化命令生成。

    mkdir kong
    cd kong
    pwd
    /root/kong
    
    kong config -c kong.conf init
    
    ls
    kong.yml

kong.yml是一个模板文件。在其中定义一个service和route。

    vi kong.yml
    ......
    _format_version: "1.1"
    
    # Each Kong entity (core entity or custom entity introduced by a plugin)
    # can be listed in the top-level as an array of objects:
    
    services:
    - name: example-service
      url: http://www.baidu.com
      # Entities can store tags as metadata
      tags:
      - example
      # Entities that have a foreign-key relationship can be nested:
      routes:
      - name: example-route
        paths:
        - /foo
    ......

将声明配置文件加载到Kong。

    vi /etc/kong//kong.conf
    declarative_config = /root/kong/kong.yml    # The path to the declarative configuration
                                    # file which holds the specification of all
                                    # entities (Routes, Services, Consumers, etc.)
                                    # to be used when the `database` is set to
                                    # `off`.
    
    kong start
    
    curl http://localhost:8001 -s | python -m json.tool | grep database
            "database": "off",
    
    curl http://localhost:8001 -s | python -m json.tool | grep declarative_config
            "declarative_config": "/root/kong/kong.yml",

测试一下。

    curl -i http://192.168.1.40:8000/foo
    HTTP/1.1 200 OK
    Content-Type: text/html; charset=UTF-8
    Content-Length: 2381
    Connection: keep-alive
    Accept-Ranges: bytes
    Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
    Date: Sun, 22 Dec 2019 05:13:31 GMT
    Etag: "588604c4-94d"
    Last-Modified: Mon, 23 Jan 2017 13:27:32 GMT
    Pragma: no-cache
    Server: bfe/1.0.8.18
    Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
    X-Kong-Upstream-Latency: 17
    X-Kong-Proxy-Latency: 5
    Via: kong/1.4.2
    
    <!DOCTYPE html>
    <!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>

- 管理API加载方式

可以通过Admin API [http//kong_url:8001/conf]()加载申明配置数据。我们可以继续将声明数据存放到kong.yml中，通过httpie来调用。先安装httpie。

    yum install -y httpie

先注释掉声明配置文件方式，然后通过管理API加载。

    vi /etc/kong/kong.conf
    #declarative_config = /root/kong/kong.yml          # The path to the declarative configuration
                                    # file which holds the specification of all
                                    # entities (Routes, Services, Consumers, etc.)
                                    # to be used when the `database` is set to
                                    # `off`.
    
    kong stop
    kong start
    
    curl http://localhost:8001 -s | python -m json.tool | grep declarative_config
    
    http 192.168.1.40:8001/config config=@kong.yml
    HTTP/1.1 201 Created
    Access-Control-Allow-Origin: *
    Connection: keep-alive
    Content-Length: 683
    Content-Type: application/json; charset=utf-8
    Date: Sun, 22 Dec 2019 05:25:07 GMT
    Server: kong/1.4.2
    X-Kong-Admin-Latency: 323
    
    {
        "routes": {
            "eeb325b8-afd4-5ba2-a113-c34eeb969c1d": {
                "created_at": 1576992307,
                "https_redirect_status_code": 426,
                "id": "eeb325b8-afd4-5ba2-a113-c34eeb969c1d",
                "name": "example-route",
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
                    "id": "7a267194-6123-5ee0-a1c9-4c1559eb0d95"
                },
                "strip_path": true,
                "updated_at": 1576992307
            }
        },
        "services": {
            "7a267194-6123-5ee0-a1c9-4c1559eb0d95": {
                "connect_timeout": 60000,
                "created_at": 1576992307,
                "host": "www.baidu.com",
                "id": "7a267194-6123-5ee0-a1c9-4c1559eb0d95",
                "name": "example-service",
                "port": 80,
                "protocol": "http",
                "read_timeout": 60000,
                "retries": 5,
                "tags": [
                    "example"
                ],
                "updated_at": 1576992307,
                "write_timeout": 60000
            }
        }
    }

测试一下。

    curl -i http://192.168.1.40:8000/foo
    HTTP/1.1 200 OK
    Content-Type: text/html; charset=UTF-8
    Content-Length: 2381
    Connection: keep-alive
    Accept-Ranges: bytes
    Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
    Date: Sun, 22 Dec 2019 05:25:55 GMT
    Etag: "588604c4-94d"
    Last-Modified: Mon, 23 Jan 2017 13:27:32 GMT
    Pragma: no-cache
    Server: bfe/1.0.8.18
    Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
    X-Kong-Upstream-Latency: 13
    X-Kong-Proxy-Latency: 186
    Via: kong/1.4.2
    
    <!DOCTYPE html>
    <!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>