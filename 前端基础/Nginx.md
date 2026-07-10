# Nginx 完整体系详解（分三大模块：基础、高性能、高级特性）
## 一、九、Nginx — 基础配置与服务（1-4）
### 1. 安装 + 配置文件体系
#### （1）主流安装方式
1. **yum/apt 官方源安装**（生产常用）
    CentOS：yum install nginx；Ubuntu：apt install nginx
    优点：自动注册systemd服务、目录标准化、升级简单
2. **源码编译安装**
    ./configure 自定义模块（如ngx_lua、http_sub、stream等）→ make && make install
    适用：需要定制第三方模块、指定安装路径、特殊编译参数
3. **Docker 容器部署**
    快速测试环境，生产需挂载配置、静态资源持久化

#### （2）核心配置目录结构
```
/etc/nginx/
├── nginx.conf        # 主配置入口
├── conf.d/           # 站点分配置（include 引入，规范拆分）
│   ├── site1.conf
│   └── site2.conf
├── mime.types        # 文件MIME类型映射（静态资源ContentType）
├── fastcgi_params    # PHP-FPM 通用参数
├── proxy_params      # 反向代理通用请求头
└── ssl/              # 证书存放目录
```
#### （3）服务管理命令（systemd）
```bash
systemctl start/stop/restart nginx   # 启停重启
systemctl enable nginx               # 开机自启
nginx -t                             # 校验配置语法（上线必执行）
nginx -s reload                      # 平滑重载（不中断业务）
nginx -s stop                        # 强制停止
nginx -s quit                        # 优雅停止（处理完现有连接再退出）
```

### 2. Web服务核心：http 全局块配置
`nginx.conf` 最外层 `http {}` 块，全局作用于所有虚拟主机server，定义Web通用参数。
#### 核心常用指令
1. **include mime.types;**
    绑定文件后缀与响应Content-Type，浏览器识别图片/视频/文本等资源
2. **default_type application/octet-stream;**
    未匹配mime的文件默认二进制下载
3. **log_format main 日志模板**
    自定义访问日志字段：客户端IP、请求时间、请求URI、状态码、响应耗时、UA、代理IP等
    ```nginx
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" "$http_user_agent"';
    access_log /var/log/nginx/access.log main buffer=16k flush=5s;
    ```
4. **sendfile on;**
    Linux零拷贝技术，静态文件不经过用户态，大幅提升文件传输性能
5. **tcp_nopush on; tcp_nodelay on;**
    网络发包优化：大包合并发送，小包实时响应
6. **keepalive_timeout 65;**
    HTTP长连接超时时间，复用TCP连接减少握手开销
7. **gzip 压缩系列**
    gzip on; gzip_min_length 1k; gzip_types text/css application/json;
    压缩文本类资源，减少带宽消耗，图片/视频不建议开启
8. **client_max_body_size 10M;**
    限制客户端上传文件最大体积，上传接口必备

### 3. Web服务：server 虚拟主机配置
`http` 内部可多个 `server {}`，代表独立网站（虚拟主机），通过**监听端口+域名**区分。
#### 核心指令
1. `listen 80; listen 443 ssl;` 监听端口，443开启SSL证书
2. `server_name www.xxx.com xxx.com;` 绑定域名，支持泛域名 *.xxx.com
3. **root / html/www; index index.html index.htm;**
    root：站点静态资源根目录；index：默认首页文件
4. **location 路由匹配（核心）**
    匹配优先级：精确匹配`=` > ^~前缀匹配 > 正则~ / ~* > 普通前缀匹配
    ```nginx
    # 精确匹配首页
    location = / { root /html; }
    # 静态资源缓存
    location ~* \.(jpg|png|css|js)$ {
        expires 7d; # 浏览器缓存7天
        add_header Cache-Control "public";
    }
    # 反向代理接口
    location /api/ { proxy_pass http://backend; }
    # 404兜底页面
    location / { try_files $uri $uri/ /404.html; }
    ```
5. 错误页面定制
    `error_page 404 /404.html; error_page 500 502 503 /5xx.html;`

#### 两种虚拟主机区分方式
1. 基于域名（最常用）：同端口不同server_name
2. 基于端口：同域名不同listen端口
3. 基于IP：服务器多网卡，listen绑定指定IP

### 4. Web服务安全配置
全部配置在 `http/server/location` 块，防爬虫、防攻击、隐藏敏感信息。
1. **隐藏Nginx版本号**
    `server_tokens off;` 响应头不返回nginx版本，避免针对性漏洞攻击
2. **安全响应头**
    ```nginx
    add_header X-Frame-Options DENY;          # 禁止iframe嵌套，防点击劫持
    add_header X-XSS-Protection "1; mode=block"; # 开启XSS过滤
    add_header X-Content-Type-Options nosniff;  # 禁止MIME类型嗅探
    add_header Referrer-Policy strict-origin-when-cross-origin;
    ```
3. **限制请求频率与并发**
    ```nginx
    # 定义限流规则
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    server {
        limit_req zone=req_limit burst=20 nodelay; # 每秒10请求，最多缓冲20
        limit_conn conn_limit 20; # 单IP最大并发连接20
    }
    ```
4. **防盗链**
    校验Referer，禁止其他站点引用本站图片资源
    ```nginx
    location ~* \.(png|jpg)$ {
        valid_referers none blocked server_names *.xxx.com;
        if ($invalid_referer) { return 403; }
    }
    ```
5. **IP黑白名单**
    deny 192.168.1.100; allow all; / allow 内网段，deny all
6. **SSL/TLS 加密安全（HTTPS）**
    禁用弱加密套件、关闭SSLv3/TLS1.0/1.1，强制TLS1.2+
    ```nginx
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ```
7. **禁止危险访问**
    屏蔽 .git、.env、.bak 配置备份文件
    `location ~ /\.(git|env|svn) { return 403; }`

---

## 二、Nginx 高性能核心（5-11）
### 5. upstream 后端服务集群定义
专门用于反向代理/负载均衡，在`http{}`块中定义一组后端服务池。
#### 基础语法
```nginx
upstream backend_api {
    server 127.0.0.1:8080 weight=3; # 权重3
    server 127.0.0.1:8081 weight=1;
    server 127.0.0.1:8082 backup;   # 备用节点，主节点全部挂掉才启用
    server 127.0.0.1:8083 down;     # 永久下线节点
}
```
#### 内置负载均衡策略（搭配第11点详解）
- 轮询（默认）：均匀分发请求
- weight权重：配置weight，机器性能好分配更多流量
- ip_hash：同一客户端IP固定打到同一后端（会话保持）
- least_conn：最少连接优先，缓解高负载节点压力

### 6. keepalive 长连接优化（前后端两层）
Nginx存在两层TCP连接：**客户端<->Nginx**、**Nginx<->后端upstream**，两层都要优化keepalive。
1. **客户端侧 keepalive_timeout**
    控制浏览器与Nginx的长连接复用，减少TCP三次握手
2. **后端 upstream keepalive（性能关键点）**
    默认Nginx反向代理每次请求新建TCP，大量短连接造成端口耗尽、TIME_WAIT堆积
    ```nginx
    upstream backend {
        server 127.0.0.1:8080;
        keepalive 128; # 每个worker进程维持128条长连接池
    }
    location /api {
        proxy_pass http://backend;
        # 开启后端长连接必备头
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
    ```
    作用：复用Nginx到Java/Go/PHP后端的TCP连接，大幅降低连接建立开销，提升QPS。

### 7. Nginx 配置层面性能优化
1. **worker进程参数**
    worker_processes auto; 自动匹配CPU核心数
    worker_cpu_affinity 绑定进程到指定CPU，减少上下文切换
2. **文件描述符上限**
    worker_rlimit_nofile 65535; 单进程最大打开文件数，高并发必须调大
3. **缓冲区优化**
    proxy_buffers 8 64k; proxy_buffer_size 128k;
    缓冲后端响应，避免小分包频繁发包
4. **静态资源缓存**
    expires 缓存静态文件，减少回源后端请求
5. **禁用不必要模块/日志缓冲**
    access_log buffer=32k flush=10s; 批量刷日志，降低磁盘IO

### 8. 反向代理 proxy（Nginx核心能力）
客户端请求nginx，nginx转发请求至后端业务服务，再将后端响应返回客户端。
#### 完整标准代理模板
```nginx
location /api/ {
    proxy_pass http://backend; # 指向upstream集群
    # 透传客户端真实信息（后端获取真实IP、域名）
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    # 超时控制（防止卡死连接）
    proxy_connect_timeout 5s;
    proxy_read_timeout 30s;
    proxy_send_timeout 30s;
    # 后端长连接配置
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```
#### 反向代理典型场景
1. 前后端分离：静态资源nginx本地，接口转发Java/Go服务
2. 跨域统一收口，所有请求走nginx统一处理CORS
3. 灰度发布、流量切分、限流统一收口

### 9. Nginx 进程模型（事件驱动，高性能根源）
Nginx采用**多进程+异步非阻塞IO(epoll)** 模型，区别于Apache多进程同步阻塞。
1. **Master 主进程**
    1个，负责读取配置、管理worker进程、接收信号（reload/stop）、监控worker异常拉起新进程
    reload平滑更新原理：master启动新worker处理新请求，旧worker处理完存量连接后自动退出，业务无中断
2. **Worker 工作进程**
    数量等于CPU核心数，完全独立，单线程异步处理所有连接
    核心：epoll多路复用，单个worker可同时处理上万并发连接，无线程切换开销
3. **Cache Loader / Cache Manager（缓存进程）**
    管理磁盘缓存文件清理、加载，静态资源磁盘缓存专用

### 10. 系统内核优化（配合Nginx发挥极限性能）
修改 `/etc/sysctl.conf`，全局Linux内核网络参数，高并发必备：
1. 文件句柄：`fs.file-max = 1048576` 系统全局最大打开文件数
2. TCP连接优化
    ```
    net.ipv4.tcp_syncookies = 1    # 防SYN洪水攻击
    net.ipv4.tcp_tw_reuse = 1      # 复用TIME_WAIT连接
    net.ipv4.tcp_fin_timeout = 30  # 缩短TIME_WAIT超时
    net.ipv4.tcp_keepalive_time = 60
    ```
3. 端口范围扩大
    `net.ipv4.ip_local_port_range = 1024 65535`
    解决反向代理大量短连接导致本地端口耗尽
4. 内核接收发送缓冲区调大，提升大流量吞吐

### 11. 负载均衡 upstream 全策略
基于upstream模块实现流量分发，4种原生策略+第三方扩展策略
1. **轮询（默认）**
    请求依次分发到各个后端，无状态，适合无会话业务
2. **weight 加权轮询**
    性能高的服务器配置更大weight，分配更多流量
3. **ip_hash IP绑定**
    客户端IP哈希固定节点，解决session会话共享问题；缺点：节点扩容会导致会话重新打散
4. **least_conn 最小连接**
    新请求分配给当前活跃连接最少的后端，均衡长连接业务压力
5. 第三方扩展策略（编译第三方模块）
    url_hash：按请求URL哈希，静态资源缓存场景
    fair：根据后端响应时间动态分配，响应快的机器承载更多流量

健康检查配套：`max_fails fail_timeout`，连续失败N次临时剔除故障节点，自动恢复

---

## 三、Nginx 高级特性（12-15）
### 12. Nginx 配置语法完整规范
#### 1）配置块层级（由外到内）
main全局块 → http块 → server虚拟主机块 → location路由块
指令作用域逐级继承，下层可覆盖上层参数

#### 2）变量体系（内置全局变量+自定义变量）
内置核心变量：
$remote_addr 客户端IP、$uri 当前请求路径、$args URL参数、$host 请求域名、$request_method 请求方法GET/POST、$status 响应码
自定义变量：`set $flag 0;` 用于if判断、分支逻辑

#### 3）流程控制指令
1. if 判断（仅支持简单匹配，不推荐复杂嵌套）
    ```nginx
    if ($request_method !~ ^(GET|POST)$ ) { return 405; }
    ```
2. return 直接返回状态码/页面，中断后续逻辑
3. rewrite URL重写，正则跳转
    `rewrite ^/old/(.*)$ /new/$1 permanent;` permanent=301永久重定向，redirect=302临时
4. try_files 按顺序查找文件，找不到跳转兜底地址

#### 4）正则匹配符号
`~` 区分大小写正则；`~*` 不区分大小写；`^`开头；`$`结尾；`()`捕获分组

### 13. Nginx 底层架构（事件驱动+模块化）
1. **模块化架构**
    Nginx核心极小，所有功能都是模块：http模块、stream四层代理模块、ssl模块、limit_req限流模块等
    分为：核心模块、标准HTTP模块、第三方扩展模块（ngx_lua、njs、nginx-rtmp）
    编译时按需加载模块，不加载无用模块节省内存

2. **事件驱动模型（epoll/kqueue/dev/poll）**
    Linux使用epoll，实现IO多路复用，单worker进程异步处理万级并发，无阻塞
    事件分为：读事件（接收客户端请求）、写事件（返回响应）、定时器事件（超时、缓存清理）

3. **内存与磁盘缓存架构**
    proxy_cache 磁盘缓存：热点接口/静态资源缓存到磁盘，减少后端请求；配套缓存内存元数据zone

4. **请求处理完整生命周期**
    客户端TCP连接建立 → 解析请求行/请求头 → 匹配server_name → 匹配location → 执行rewrite/if/限流/安全头 → 静态文件读取 or 反向代理转发后端 → 组装响应头体 → 返回客户端 → 记录access日志

### 14. ngx_lua（OpenResty 核心扩展，最常用高级开发）
原生Nginx配置能力有限，ngx_lua嵌入Lua脚本，在请求各个阶段执行自定义逻辑，实现复杂业务，OpenResty=Nginx+ngx_lua。
#### 1）Lua执行阶段（按执行顺序）
init_by_lua：Nginx启动初始化全局变量
rewrite_by_lua：URL重写阶段，可修改路由、鉴权
access_by_lua：访问控制阶段，接口鉴权、token校验、限流二次开发
content_by_lua：直接返回响应，不依赖后端服务
header_filter_by_lua：修改响应头
body_filter_by_lua：修改响应返回内容
log_by_lua：自定义日志上报

#### 2）典型业务场景
1. 自定义JWT鉴权，无需转发后端校验token
2. 分布式限流（基于Redis实现精准限流，原生limit_req单机限流）
3. 灰度发布、动态路由切分流量
4. 接口数据缓存（Redis热点缓存）
5. 动态修改响应内容、脱敏返回数据

#### 优势
Lua轻量、无GC卡顿、和Nginx事件循环无缝结合，性能接近原生C模块，替代复杂if/rewrite逻辑。

### 15. Nginx 模块体系（原生+第三方）
#### 一、原生内置模块（编译默认携带）
1. HTTP核心模块：ngx_http_core_module（server/location/root等基础指令）
2. 访问控制：ngx_http_access_module（allow/deny IP黑白名单）
3. 限流模块：ngx_http_limit_req / limit_conn
4. 压缩模块：ngx_http_gzip_module
5. 反向代理：ngx_http_proxy_module
6. 负载均衡：ngx_http_upstream_module
7. SSL加密：ngx_http_ssl_module
8. 静态缓存：ngx_http_proxy_cache_module
9. 重定向：ngx_http_rewrite_module

#### 二、常用第三方扩展模块（需源码编译安装）
1. ngx_lua：Lua脚本开发（OpenResty标配）
2. ngx_http_substitutions_filter_module：响应内容替换
3. nginx-stream：四层TCP/UDP代理（数据库、Redis代理）
4. nginx-rtmp：流媒体直播模块
5. ngx_brotli：brotli压缩（比gzip压缩率更高）
6. ngx_http_geoip2_module：根据客户端IP获取城市、国家地域信息
7. ngx_http_upstream_check_module：后端健康检查可视化面板

#### 模块使用要点
1. 源码编译时通过 `--add-module=/模块源码路径` 加载第三方模块
2. OpenResty发行版预编译集成大量常用第三方模块，企业开发首选，省去手动编译成本
3. 模块冲突：多个模块修改同一阶段逻辑时，需注意执行顺序，避免覆盖异常

## 整体总结
1. **基础层**：搭建Web基础服务，实现静态站点、HTTPS、基础安全防护，是Nginx入门必备；
2. **高性能层**：Nginx核心竞争力，依靠多进程epoll、长连接、负载均衡、内核调优支撑十万级并发，业务集群反向代理核心；
3. **高级特性层**：突破原生配置限制，依靠Lua扩展、自定义模块、底层架构理解，实现网关级复杂业务（鉴权、灰度、分布式限流、动态路由），适合网关、中间件开发场景。