+++
author = "TommyLike"
title = "Kubernetes nginx ingress cache 实践"
date = "2020-11-28"
description = "在kubernetes集群的nginx ingress中引入cache和实践"
tags = [
    "argocd",
    "nginx",
    "kubernetes",
]
categories = [
    "opensourceway",
    "kubernetes",
]
+++
最近项目在对部署在香港节点的网站服务做资源加载提速，本文主要记录了在kubernetes的原生nginx ingress中引入server cache的流程和方案，
里面涉及的都是最为常见的技术，并没有什么新的东西，算是一个总结和记录。
<!--more-->

## 背景和前提
当前已上线的网站都是基于`Nginx Ingress` + `Hugo/Vue Server`的模式部署的，发布采用 `ArgoCD(GitOps)` + `Jenkins` ,大致的流程如下:
![Website pipeline](/img/website-pipeline.png)
## 准备
Nginx做Server端的缓存目前有2种方式比较常见:
1. 基于外置的中间件做数据缓存，比如Nginx官方Redis Module给出的[部署介绍](https://www.nginx.com/resources/wiki/modules/redis/)，
这种方式的好处就是: 缓存集中，有利于命中和清理，同时不会增加nginx本身的业务复杂度，但他也有问题，就是会引入部署的成本。 
```shell script
server {
    location / {
        set $redis_key $uri;

        redis_pass     name:6379;
        default_type   text/html;
        error_page     404 = /fallback;
    }

    location = /fallback {
        proxy_pass backend;
    }
}
```
2. 基于Nginx本身的`proxy_cache`实现，这个是官方很早就引入的功能，直接利用本地磁盘做缓存存储，考虑到网站本身的规模和系统复杂度, 
我们选择了nginx自带的`proxy_cache`加kubernetes中的`memory emptyDir`做服务端缓存。
## 环境准备
Nginx本身已经包含了proxy_cache的模块，不过关于purge的功能却仅在商业版本中才包含，所以我们需要自己引入并制作镜像，下面是社区推荐的第三方
的purge模块:

    https://github.com/FRiCKLE/ngx_cache_purge
    
很遗憾他并不支持批量的缓存清除，从README里面的配置指导可以看出，我们需要对缓存的资源进行逐个清除:
```shell script
http {
    proxy_cache_path  /tmp/cache  keys_zone=tmpcache:10m;

    server {
        location / {
            proxy_pass         http://127.0.0.1:8000;
            proxy_cache        tmpcache;
            proxy_cache_key    $uri$is_args$args;
        }

        location ~ /purge(/.*) {
            allow              127.0.0.1;
            deny               all;
            proxy_cache_purge  tmpcache $1$is_args$args;
        }
    }
}
```
而且，这个仓库已经6年没人维护了，好在最近nginx社区自己fork并维护起来了[地址](https://github.com/nginx-modules/ngx_cache_purge)，
最关键的是作者还加入了我们正想要的`Partial Keys`功能 :) 。

Nginx ingress关于怎么制作自定义镜像的指导比较少，不过从Makefile里面能快速搜索到他们的[Base Image](https://github.com/kubernetes/ingress-nginx/tree/master/images/nginx)
而且制作镜像的流程非常清晰，基于现有的流程做扩展和第三方Module引入还是比较方便的。
```
现有引入的Module:
nginx-http-auth-digest
ngx_http_substitutions_filter_module
nginx-opentracing
opentracing-cpp
zipkin-cpp-opentracing
dd-opentracing-cpp
ModSecurity-nginx (only supported in x86_64)
brotli
geoip2
```
完整的改动已经放到了我们组织的fork[仓库](https://github.com/opensourceways/ingress-nginx/tree/feature/enable_cache_purge),
需要注意的一个点就是nginx ingress官方为了支持多架构和性能，在构建镜像时引入了**buildx**组件， 他跟我们原生的docker build有一点区别是他并不会默认导出制作好的镜像，需要我们在命令中具体指明，如:
```
#具体文档，请移步: https://github.com/docker/buildx
docker buildx build --output=type=docker .
```
有了基础镜像，在制作controller的镜像时，只需要将**BASE_IMAGE**替换为我们生成好的镜像即可, 如:
```
BASE_IMAGE=opensourceways/ingress-nginx-with-purge:0.0.1 make image
```
有了镜像，可以在镜像中通过`nginx -V`确认模块以加载:
```
➜  ~ docker run -it --rm  opensourceway/ingress-nginx-controller-with-purge-cache:v0.35.0.0 nginx -V
nginx version: nginx/1.19.2
built by gcc 9.3.0 (Alpine 9.3.0)
built with OpenSSL 1.1.1g  21 Apr 2020
TLS SNI support enabled
<content skipped>
--add-module=/tmp/build/ngx_devel_kit-0.3.1 
--add-module=/tmp/build/set-misc-nginx-module-0.32 
--add-module=/tmp/build/headers-more-nginx-module-0.33 
--add-module=/tmp/build/nginx-http-auth-digest-cd8641886c873cf543255aeda20d23e4cd603d05 
--add-module=/tmp/build/ngx_http_substitutions_filter_module-bc58cb11844bc42735bbaef7085ea86ace46d05b 
--add-module=/tmp/build/lua-nginx-module-0.10.17 --add-module=/tmp/build/stream-lua-nginx-module-0.0.8 
--add-module=/tmp/build/lua-upstream-nginx-module-0.07 
--add-module=/tmp/build/nginx-influxdb-module-5b09391cb7b9a889687c0aa67964c06a2d933e8b 
--add-dynamic-module=/tmp/build/nginx-opentracing-0.9.0/opentracing 
--add-dynamic-module=/tmp/build/ModSecurity-nginx-b55a5778c539529ae1aa10ca49413771d52bb62e 
--add-dynamic-module=/tmp/build/ngx_http_geoip2_module-3.3 
--add-module=/tmp/build/nginx_ajp_module-bf6cd93f2098b59260de8d494f0f4b1f11a84627 
--add-module=/tmp/build/ngx_brotli --add-module=/tmp/build/ngx_cache_purge-2.5.1
```
有了镜像，下一步就是配置了。
## 主要配置
1. `proxy_cache_path`:
```shell script
Syntax:	proxy_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [min_free=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];
Default:	—
Context:	http
```
这个是cache的全局配置，其中几个比较重要的字段介绍如下:

* `path`: 缓存的配置路径
* `keys_zone:<name:size>`: 缓存的key区域，当开启某个domain的访问缓存时，使用`proxy_cache`指定对应的key_zone, 注意缓存的key是存储在内存中的，所以需要指定存储的上限，根据官方文档，1mb基本可以存储8k个key。
* `use_temp_path`：是否启用temp目录用于上游返回内容的保存，nginx的cache在开启temp_path的时候会有2步，保存结果到temp目录，拷贝到cache目录，建议关闭。
* `inactive`: 缓存的保留时间`retention`, 如果超过时间资源没有被访问，缓存将被清除。
* `max_size`: 缓存数据的最大占用空间，超出后空间将被清理(基于最近最少使用原则)，
* `levels` + `proxy_cache_key`: 这2个字段配合起来，决定了我们缓存存储的路径，其中level决定了存储的目录层级以及目录名，cache_key决定了最终的文件名，举个例子，比如我的配置如下：
```shell script
proxy_cache_key $uri$is_args$args;
proxy_cache_path /tmp/statics_cache levels=1:2 keys_zone=statics_cache:50m use_temp_path=off max_size=500m inactive=1h;
```
当我们访问`http://somedomain.com/public/img/grafana_icon.svg`的时候，对`proxy_cache_key`(/img/grafana_icon.svg)进行MD5计算，得:
```
➜  ~ md5 -s "/public/img/grafana_icon.svg"
MD5 ("/public/img/grafana_icon.svg") = e07fe02651e785ebf74c2c5c0abae094
```
那我们最终的存储路径就是:/tmp/statics_cache/4/09/e07fe02651e785ebf74c2c5c0abae094, 其中目录信息反向截图。

2. `proxy_cache_lock`: cache锁，在多个请求访问同样资源，且开启cache的情况下，只有一个请求有权限去生成cache或访问cache。
3. `proxy_cache_valid`: cache的有效时间，超期的缓存将从上游重新获取， 我们可以基于返回结果做差异化配置，比如`proxy_cache_valid 404      1m;`设置404近保留1分钟。
4. `proxy_ignore_headers`：忽略响应中的某些header，比如: `proxy_ignore_headers Cache-Control`就能忽略上游的缓存策略，避免对我们的server cache造成影响。
5. `proxy_cache_methods`: 缓存的http方法，默认会包含GET和HEAD。
6. `proxy_cache_min_uses`: 设置资源访问多少次后再缓存，提高阈值能确保高频率的资源被缓存和使用。
基于上面的配置，我们就能在nginx-ingress中configmap中引入http-snippet， 修改deployment确保emptyDir挂载:
```yaml
apiVersion: v1
data:
  http-snippet: |
    proxy_cache_path /tmp/statics_cache levels=1:2 keys_zone=statics_cache:50m use_temp_path=off max_size=500m inactive=1h;
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
# other attribute in deployment are ignored.
volumes:
- name: nginx-proxy-cache
  emptyDir:
    medium: Memory
    sizeLimit: "500Mi"
```
## 启用cache
当需要网站的server cache时，需要在具体的Ingress中做如下配置:
```shell script
nginx.ingress.kubernetes.io/proxy-buffering: "on"
nginx.ingress.kubernetes.io/server-snippet: |
  proxy_cache statics_cache;
  proxy_cache_lock on;
  proxy_cache_key $server_name$uri$is_args$args;
  proxy_ignore_headers Cache-Control;
  proxy_ignore_headers Set-Cookie;
  proxy_cache_valid 60m;
  add_header X-Cache-Status $upstream_cache_status;
  location ~ /purge(/.*) {
      allow 127.0.0.1;
      allow 172.20.0.0/16;      
      deny all;
      proxy_cache_purge  statics_cache $server_name$1$is_args$args;
  }
```
说明如下：
1. `proxy_buffering`： 这个是开启缓存的前提，nginx ingress默认是关闭的。
2. `add_header`: 在返回的Header中添加cache status **(MISS,HIT,EXPIRED)**，用于查看缓存命中情况，如果我们在location server等多处有定义add_header, 需要注意的一点是:
```text
# from nginx document
There could be several add_header directives. These directives are inherited from the previous configuration level if and only if there are no add_header directives defined on the current level.
```
3. `proxy_cache_key`: 在nginx ingress中我们一般会服务多个domain，所以这里的key需要引入server_name避免相互干扰。

## 开启Purge
网站每次发布后，不能保证Assets中的资源都引入hash避免缓存干扰，所以我们需要支持通过API调用清理失效的缓存，purge模块支持我们配置单独的API用于网站缓存资源清理(前缀匹配):
```shell script
location ~ /purge(/.*) {
  allow 127.0.0.1
  allow 172.20.0.0/16;
  deny all;
  proxy_cache_purge  statics_cache $server_name$1$is_args$args;
}
```
上面的配置段需要同样添加到server-snippet中，且仅支持内网访问，**注意**：`proxy_cache_purge`的key需要跟`proxy_cache_key`保持一致。
我们通过API测试，结果是`some-domain`的缓存资源都会被清除:
```shell script
$: curl  -H "Host:some-domain" -k  https://<nginx-endpoint>/purge/*
```
```html
<html>
    <head><title>Successful purge</title></head>
    <body bgcolor="white"><center><h1>Successful purge</h1><p>Key : some-domain/*</p></center></body>
</html>
```
不过，我们还有一个问题: nginx ingress都是多实例的，清理单个实例还不够，当前原生的Kubernetes中并没有类似的概念可以支持我们一次性清理，只能考虑将purge的请求复制多份到endpoints分别处理, 大致的逻辑如下[Stackoverflow](https://stackoverflow.com/questions/49612412/kubenetes-is-it-possible-to-hit-multiple-pods-with-a-single-request-in-kubernet):
```shell script
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token);
curl https://kubernetes.default.svc/api/v1/namespaces/default/endpoints --silent \
--header "Authorization: Bearer $TOKEN" --insecure | jq -rM ".items[].subsets[].addresses[].ip" | xargs curl
```
我们搭建了一个Basic Auth的[Http Server](https://github.com/opensourceways/smart_tools/tree/main/nginx-purger)用于响应流水线中的清除缓存动作。
有了他我们就差最后一步，跟发布的流水线对接, 前文有提到我们的发布流程是基于`Jenkins + ArgoCD`实现的，
考虑到ArgoCD本身就有[Webhook机制](https://argoproj-labs.github.io/argocd-notifications/services/webhook/)，所以我们的应用的部署代码仓库需要新增如下资源:
## 触发Purge
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: website-nginx-purge
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  ttlSecondsAfterFinished: 600
  template:
    spec:
      containers:
        - name: purge-trigger
          image: curlimages/curl:7.72.0
          env:
            - name: USERNAME
              valueFrom:
                secretKeyRef:
                  key: username
                  name: purge-secrets
            - name: PASSWORD
              valueFrom:
                secretKeyRef:
                  key: password
                  name: purge-secrets
          command:
            - curl
            - -u
            - $(USERNAME):$(PASSWORD)
            - --fail
            - https://<api-endpoints>/nginx-purger/purge?hostname=<hostname-to-purge>
      restartPolicy: Never
  backoffLimit: 2

```
这样一旦我们更新网站，ArgoCD的Sync行为结束，nginx ingress中的cache就会被自动清除:
```
nginx cache purged for host: some-domain on ingress instance 172.20.0.103 
nginx cache purged for host: some-domain on ingress instance 172.20.0.144
nginx cache purged for host: some-domain on ingress instance 172.20.0.56
```
需要注意的是:
1. `argocd.argoproj.io/hook-delete-policy: BeforeHookCreation`: 能确保每次synced后Job资源删除后执行。
2. argo的application不能设置为自动Sync，这样会导致缓存被频繁清除。

全文完。













