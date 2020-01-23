# Kong系列-14-自定义插件

Kong开源了大量的开源插件，当这些开源插件不能满足我们的需求，就需要修改这些开源插件或者自定义插件。Kong提供了方便地自定义插件机制，用户可以开发自己的定制插件。自定义插件可以和Kong进行深层次的集成，如使用数据库表结构，或者扩展Admin API。如果插件实现了所有可选模块，则其目录结构如下所示：

    complete-plugin
    ├── api.lua
    ├── daos.lua
    ├── handler.lua
    ├── migrations
    │   ├── init.lua
    │   └── 000_base_complete_plugin.lua
    └── schema.lua

插件的每个模块对应不同的功能，其中handler.lua和schema.lua是两个必须模块。

| Module Name | Required | Description |
| :------| :------: | :------ |
| api.lua | No | 插件需要向Admin API暴露接口时使用。 |
| daos.lua | No | 数据层相关，当插件需要访问数据库是配置。 |
| handler.lua | Yes | 插件的主要逻辑，Kong在不同阶段执行其对应的handler。 |
| migrations/*.lua | No | 插件依赖的数据库表结构，启用了daos.lua时需要定义。 |
| schema.lua | Yes | 插件的配置参数定义，主要用于Kong参数验证。 |

可以参考官方开源插件[Key Authentication](https://github.com/Kong/kong/tree/master/kong/plugins/key-auth)的目录结构。

下面演示如何在Kong加载自定义插件。先准备自定义插件的Lua代码。

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

Kong已经预置了大量的开源插件，这些插件都放在/usr/local/share/lua/5.1/kong/plugins/目录下，我们自定义的插件也可以放到该目录下。

    cd /usr/local/share/lua/5.1/kong/plugins/
    
    ls
    acl              datadog                      ldap-auth        rate-limiting          syslog
    aws-lambda       file-log                     loggly           request-size-limiting  tcp-log
    azure-functions  hmac-auth                    log-serializers  request-termination    udp-log
    base_plugin.lua  http-log                     oauth2           request-transformer    zipkin
    basic-auth       ip-restriction               post-function    response-ratelimiting
    bot-detection    jwt                          pre-function     response-transformer
    correlation-id   key-auth                     prometheus       session
    cors             kubernetes-sidecar-injector  proxy-cache      statsd
    
    mv /root/kong/plugin/myheader/ .
    
    ls
    acl              datadog                      ldap-auth        proxy-cache            statsd
    aws-lambda       file-log                     loggly           rate-limiting          syslog
    azure-functions  hmac-auth                    log-serializers  request-size-limiting  tcp-log
    base_plugin.lua  http-log                     myheader         request-termination    udp-log
    basic-auth       ip-restriction               oauth2           request-transformer    zipkin
    bot-detection    jwt                          post-function    response-ratelimiting
    correlation-id   key-auth                     pre-function     response-transformer
    cors             kubernetes-sidecar-injector  prometheus       session

kong.conf定义了加载哪些插件，plugins配置项的缺省值是bundled，表示加载官方开源插件，如果要加载自定义插件，去掉注释并在后面跟上自定义插件名称。

    vi /etc/kong/kong.conf
    #plugins = bundled               # Comma-separated list of plugins this node
                                     # should load. By default, only plugins
                                     # bundled in official distributions are
                                     # loaded via the `bundled` keyword.
    
    改为
    
    plugins = bundled,myheader       # Comma-separated list of plugins this node
                                     # should load. By default, only plugins
                                     # bundled in official distributions are
                                     # loaded via the `bundled` keyword.

查看Kong配置。

    kong start
    
    curl http://localhost:8001 -s | python -m json.tool
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
    
                "myheader": true,
    
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

测试一下。

    curl -X POST \
      --url http://localhost:8001/services/ \
      --data 'name=example-service' \
      --data 'url=http://www.baidu.com' \
      -s | python -m json.tool
    {
        "client_certificate": null,
        "connect_timeout": 60000,
        "created_at": 1577356917,
        "host": "www.baidu.com",
        "id": "a658c31d-28f9-49aa-ab74-497beadd0921",
        "name": "example-service",
        "path": null,
        "port": 80,
        "protocol": "http",
        "read_timeout": 60000,
        "retries": 5,
        "tags": null,
        "updated_at": 1577356917,
        "write_timeout": 60000
    }
    
    curl -X POST \
      --url http://localhost:8001/services/example-service/routes \
      --data 'name=example-service-route' \
      --data 'paths[]=/foo' \
      -s | python -m json.tool
    {
        "created_at": 1577357100,
        "destinations": null,
        "headers": null,
        "hosts": null,
        "https_redirect_status_code": 426,
        "id": "b3190688-0412-4016-85e8-22c96ad26df3",
        "methods": null,
        "name": "example-service-route",
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
            "id": "a658c31d-28f9-49aa-ab74-497beadd0921"
        },
        "snis": null,
        "sources": null,
        "strip_path": true,
        "tags": null,
        "updated_at": 1577357100
    }
    
    curl -i http://localhost:8000/foo
    HTTP/1.1 200 OK
    Content-Type: text/html; charset=UTF-8
    Content-Length: 2381
    Connection: keep-alive
    Accept-Ranges: bytes
    Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
    Date: Thu, 26 Dec 2019 10:45:41 GMT
    Etag: "588604c4-94d"
    Last-Modified: Mon, 23 Jan 2017 13:27:32 GMT
    Pragma: no-cache
    Server: bfe/1.0.8.18
    Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
    X-Kong-Upstream-Latency: 13
    X-Kong-Proxy-Latency: 93
    Via: kong/1.4.2
    
    <!DOCTYPE html>
    <!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
    
    curl -X POST \
      --url http://localhost:8001/services/example-service/plugins \
      --data 'name=myheader' \
      -s | python -m json.tool
    {
        "config": {
            "header_value": "twingao"
        },
        "consumer": null,
        "created_at": 1577357447,
        "enabled": true,
        "id": "95f27854-ebc5-4797-9023-165d59089e98",
        "name": "myheader",
        "protocols": [
            "grpc",
            "grpcs",
            "http",
            "https"
        ],
        "route": null,
        "run_on": "first",
        "service": {
            "id": "a658c31d-28f9-49aa-ab74-497beadd0921"
        },
        "tags": null
    }
    
    curl -i http://localhost:8000/foo
    HTTP/1.1 200 OK
    Content-Type: text/html; charset=UTF-8
    Content-Length: 2381
    Connection: keep-alive
    Accept-Ranges: bytes
    Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
    Date: Thu, 26 Dec 2019 10:51:21 GMT
    Etag: "588604c4-94d"
    Last-Modified: Mon, 23 Jan 2017 13:27:32 GMT
    Pragma: no-cache
    Server: bfe/1.0.8.18
    Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
    myheader: twingao                  ##注意此处
    X-Kong-Upstream-Latency: 15
    X-Kong-Proxy-Latency: 5
    Via: kong/1.4.2
    
    <!DOCTYPE html>
    <!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>




