title: php 容器化的几点建议
author: tusimo
tags: []
categories: []
cover: >-
  https://images.pexels.com/photos/1044990/pexels-photo-1044990.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=1
date: 2022-08-10 18:15:00
---
# 镜像编译

## 基础镜像编译

[github repo](https://github.com/tusimo/docker-php-fpm)

基础镜像编译使用环境变量替换配置.

```Dockerfile

FROM php:7.2-fpm-alpine

# replace repositories
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.cloud.tencent.com/g' /etc/apk/repositories

# add timezone
RUN apk add -u --no-cache tzdata \
 && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

RUN apk upgrade && \
    apk add --no-cache curl \
        boost-dev \
        git \
        ca-certificates \
        automake \
        libtool \
        file \
        linux-headers \
        re2c \
        pkgconf \
        openssl-dev \
        curl-dev \
        autoconf \
        openssl \
        gcc \
        make \
        g++ \
        zlib-dev \
        graphviz \
        libpng-dev \
        libpq \
        icu-dev \
        libffi-dev \
        freetype-dev \
        libxslt-dev \
        libjpeg-turbo-dev \
        libwebp-dev \
        libmemcached-dev \
        libmcrypt-dev \
        libzip-dev \
        librdkafka-dev && \
    docker-php-ext-configure gd \
      --with-gd \
      --with-freetype-dir=/usr/include/ \
      --with-png-dir=/usr/include/ \
      --with-jpeg-dir=/usr/include/ \
      --with-webp-dir=/usr/include/ && \
    docker-php-ext-install fileinfo pdo_mysql mysqli gd exif intl xsl soap zip opcache sockets bcmath pcntl && \
    docker-php-source delete

RUN pecl install redis-5.0.2 memcached-3.1.4 rdkafka yaf-3.0.8 yar-2.0.5 mcrypt hprose-1.6.8 \
    && docker-php-ext-enable redis memcached rdkafka yaf yar mcrypt hprose


RUN wget https://storage.googleapis.com/downloads.webmproject.org/releases/webp/libwebp-1.1.0.tar.gz -O /tmp/libwebp-1.1.0.tar.gz \
    && tar -C /tmp -zxvf /tmp/libwebp-1.1.0.tar.gz \
    && cd /tmp/libwebp-1.1.0 \
    && ./configure --prefix=/usr/local/libwebp --enable-everything \
    && make && make install

RUN apk del autoconf gcc make g++ \
    && rm -fr /var/cache/apk/* /tmp/* /usr/share/man

WORKDIR /var/www/html

# use env control php and php-fpm

ENV TZ=Asia/Shanghai
ENV APP_ENV=product
ENV PHP_DATE_TIMEZONE="Asia/Shanghai"
ENV PHP_ERROR_LOG="/proc/self/fd/2"
ENV PHP_LOG_LEVEL="notice"
ENV PHP_PROCESS_MAX=0
ENV PHP_RLIMIT_FILES=51200
ENV PHP_RLIMIT_CORE=0
ENV PHP_USER=www-data
ENV PHP_GROUP=www-data
ENV PHP_LISTEN=0.0.0.0:9000
ENV PHP_PM=static
ENV PHP_PM_MAX_CHILDREN=20
ENV PHP_PM_START_SERVERS=4
ENV PHP_PM_MIN_SPARE_SERVERS=2
ENV PHP_PM_MAX_SPARE_SERVERS=10
ENV PHP_PM_PROCESS_IDLE_TIMEOUT=10s
ENV PHP_PM_MAX_REQUESTS=10000
ENV PHP_SLOWLOG="/proc/self/fd/2"
ENV PHP_REQUEST_SLOWLOG_TIMEOUT="2s"
ENV PHP_REQUEST_TERMINATE_TIMEOUT="120s"
ENV PHP_MAX_EXECUTION_TIME=600
ENV PHP_MAX_INPUT_TIME=60
ENV PHP_MEMORY_LIMIT=384M
ENV PHP_ERROR_REPORTING="E_ALL & ~E_DEPRECATED & ~E_STRICT"
ENV PHP_DISPLAY_ERRORS="Off"
ENV PHP_DISPLAY_STARTUP_ERRORS="Off"
ENV PHP_POST_MAX_SIZE=100M
ENV PHP_UPLOAD_MAX_FILESIZE=50M
ENV PHP_MAX_FILE_UPLOADS=20
ENV PHP_ACCESS_LOG="/dev/null"
ENV PHP_TRACK_ERRORS=Off
ENV PHP_ACCESS_FORMAT="{ \"type\": \"access\", \"time\": \"%t\", \"environment\": \"%{APP_ENV}e\", \"method\": \"%m\", \"request_uri\": \"%r%Q%q\", \"status_code\": \"%s\", \"cost_time\": %{mili}d, \"cpu_usage\": { \"user\" : %{user}C, \"system\": %{system}C, \"total\": %{total}C }, \"memory_usage\": %{bytes}M, \"remote_ip\": \"%R\", \"module\": \"php-fpm\", \"log_type\": \"access-log\" }"


ENV PHP_YAF_USE_NAMESPACE=Off
ENV PHP_YAF_USE_SPL_AUTOLOAD=On
ENV PHP_YAR_CONNECT_TIMEOUT=1000
ENV PHP_YAR_TIMEOUT=5000
ENV PHP_YAR_DEBUG=off

ENV PHP_OPCACHE_ENABLE=1
ENV PHP_OPCACHE_ENABLE_CLI=1
ENV PHP_OPCACHE_MEMORY_CONSUMPTION=128
ENV PHP_OPCACHE_INTERNED_STRINGS_BUFFER=8
ENV PHP_OPCACHE_MAX_ACCELERATED_FILES=100000
ENV PHP_OPCACHE_MAX_WASTED_PERCENTAGE=5
ENV PHP_OPCACHE_USE_CWD=1
ENV PHP_OPCACHE_VALIDATE_TIMESTAMPS=0
ENV PHP_OPCACHE_REVALIDATE_FREQ=0
ENV PHP_OPCACHE_FAST_SHUTDOWN=1
ENV PHP_OPCACHE_CONSISTENCY_CHECKS=0
ENV PHP_OPCACHE_BLACKLIST_FILENAME=/var/www/html/.opcacheignore


COPY php-config/php.ini "$PHP_INI_DIR"
COPY php-config/conf.d/ "$PHP_INI_DIR"/conf.d/
COPY php-config/php-fpm.conf /usr/local/etc/
COPY php-config/www.conf /usr/local/etc/php-fpm.d/

EXPOSE 9000

RUN rm -fr /usr/local/etc/php-fpm.d/zz-docker.conf
COPY docker-entrypoint.sh /usr/local/bin/

ENTRYPOINT ["docker-entrypoint.sh"]

CMD ["php-fpm"]


```

通过环境变量控制程序的行为.替换 php 的配置为 `${XXX}`, 然后在`Dockerfile`中设置该环境变量.后续容器在运行时,可以通过改变环境变量来修改运行时的配置.

```ini
;;;;;;;;;;;;;;;;;;;;;
; FPM Configuration ;
;;;;;;;;;;;;;;;;;;;;;

; All relative paths in this configuration file are relative to PHP's install
; prefix (/usr/local). This prefix can be dynamically changed by using the
; '-p' argument from the command line.

;;;;;;;;;;;;;;;;;;
; Global Options ;
;;;;;;;;;;;;;;;;;;

[global]
; Pid file
; Note: the default prefix is /usr/local/var
; Default Value: none
;pid = run/php-fpm.pid

; Error log file
; If it's set to "syslog", log is sent to syslogd instead of being written
; into a local file.
; Note: the default prefix is /usr/local/var
; Default Value: log/php-fpm.log
error_log = ${PHP_ERROR_LOG}

; syslog_facility is used to specify what type of program is logging the
; message. This lets syslogd specify that messages from different facilities
; will be handled differently.
; See syslog(3) for possible values (ex daemon equiv LOG_DAEMON)
; Default Value: daemon
;syslog.facility = daemon

; syslog_ident is prepended to every message. If you have multiple FPM
; instances running on the same server, you can change the default value
; which must suit common needs.
; Default Value: php-fpm
;syslog.ident = php-fpm

; Log level
; Possible Values: alert, error, warning, notice, debug
; Default Value: notice
log_level = ${PHP_LOG_LEVEL}

; Log buffering specifies if the log line is buffered which means that the
; line is written in a single write operation. If the value is false, then the
; data is written directly into the file descriptor. It is an experimental
; option that can potentionaly improve logging performance and memory usage
; for some heavy logging scenarios. This option is ignored if logging to syslog
; as it has to be always buffered.
; Default value: yes
;log_buffering = no

; If this number of child processes exit with SIGSEGV or SIGBUS within the time
; interval set by emergency_restart_interval then FPM will restart. A value
; of '0' means 'Off'.
; Default Value: 0
;emergency_restart_threshold = 0

; Interval of time used by emergency_restart_interval to determine when
; a graceful restart will be initiated.  This can be useful to work around
; accidental corruptions in an accelerator's shared memory.
; Available Units: s(econds), m(inutes), h(ours), or d(ays)
; Default Unit: seconds
; Default Value: 0
;emergency_restart_interval = 0

; Time limit for child processes to wait for a reaction on signals from master.
; Available units: s(econds), m(inutes), h(ours), or d(ays)
; Default Unit: seconds
; Default Value: 0
;process_control_timeout = 0

; The maximum number of processes FPM will fork. This has been designed to control
; the global number of processes when using dynamic PM within a lot of pools.
; Use it with caution.
; Note: A value of 0 indicates no limit
; Default Value: 0
process.max = ${PHP_PROCESS_MAX}


; Specify the nice(2) priority to apply to the master process (only if set)
; The value can vary from -19 (highest priority) to 20 (lowest priority)
; Note: - It will only work if the FPM master process is launched as root
;       - The pool process will inherit the master process priority
;         unless specified otherwise
; Default Value: no set
; process.priority = -19

; Send FPM to background. Set to 'no' to keep FPM in foreground for debugging.
; Default Value: yes
daemonize = no

; Set open file descriptor rlimit for the master process.
; Default Value: system defined value
rlimit_files = ${PHP_RLIMIT_FILES}

; Set max core size rlimit for the master process.
; Possible Values: 'unlimited' or an integer greater or equal to 0
; Default Value: system defined value
;rlimit_core = 0

; Specify the event mechanism FPM will use. The following is available:
; - select     (any POSIX os)
; - poll       (any POSIX os)
; - epoll      (linux >= 2.5.44)
; - kqueue     (FreeBSD >= 4.1, OpenBSD >= 2.9, NetBSD >= 2.0)
; - /dev/poll  (Solaris >= 7)
; - port       (Solaris >= 10)
; Default Value: not set (auto detection)
events.mechanism = epoll

; When FPM is built with systemd integration, specify the interval,
; in seconds, between health report notification to systemd.
; Set to 0 to disable.
; Available Units: s(econds), m(inutes), h(ours)
; Default Unit: seconds
; Default value: 10
;systemd_interval = 10

;;;;;;;;;;;;;;;;;;;;
; Pool Definitions ;
;;;;;;;;;;;;;;;;;;;;

; Multiple pools of child processes may be started with different listening
; ports and different management options.  The name of the pool will be
; used in logs and stats. There is no limitation on the number of pools which
; FPM can handle. Your system will tell you anyway :)

; Include one or more files. If glob(3) exists, it is used to include a bunch of
; files from a glob(3) pattern. This directive can be used everywhere in the
; file.
; Relative path can also be used. They will be prefixed by:
;  - the global prefix if it's been set (-p argument)
;  - /usr/local otherwise
include=etc/php-fpm.d/*.conf
```

## 服务镜像编译

往往 `php-fpm` 一般和 `nginx`配合来提供服务,  由于 `php-fpm` 与 `nginx` 是通过 `cgi` 通信, 所以在编译镜像的时候就要考虑`nginx` 和 `php-fpm`是如何通信和部署的.在传统的方式下,一般通过在宿主机部署`nginx` 和 `php-fpm`两个程序并通过 `cgi`进行通信.

```conf

server {
    server_name _;

    listen      80 default_server;

    root        /var/www/html;

    index  index.php index.html index.htm;

	add_header 'Access-Control-Allow-Origin' '*';
	add_header 'Access-Control-Allow-Headers' '*';
	add_header 'Access-Control-Allow-Methods' '*';
	if ($request_method = 'OPTIONS') {
			return 204;
	}

	location /nginx-status {
        stub_status;

        access_log off;
        allow 127.0.0.1;
        deny all;
    }

	location / {
		try_files $uri $uri/ /index.php$is_args$args;
	}

	location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

	location ~ \.(jpg|jpeg|gif|png|css|js|ico|xml|swf)$ {
		etag           on;
		expires        max;
		access_log     off;
		log_not_found  off;
	}
	
	location ~ [^/]\.php(/|$) {
		try_files $uri =404;                      
		include        fastcgi_params;                                     
		fastcgi_index index.php;                                           
		fastcgi_split_path_info ^(.+\.php)(/.+)$;                          
		fastcgi_pass   unix:/dev/shm/php-cgi.sock;                         
		fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
	}

	location ~ /\.(?!well-known).* {
        deny all;
    }

	# return 404 for all other php files not matching the front controller
    # this prevents access to other php files you don't want to be accessible.
    location ~ \.php$ {
		return 404;
    }
}

```

以上是一个通过 `unix socket`进行通信的方式,该配置对应 `php-fpm` 的配置为: `listen: /dev/shm/php-cgi.sock`. 该方式需要`nginx`和 `php-fpm`在同一个机器上.

以上是一个通过 `unix socket`进行通信的方式,该配置对应 `php-fpm` 的配置为: `listen: /dev/shm/php-cgi.sock`. 该方式需要`nginx`和 `php-fpm`在同一个机器上.

我们也可以使用网络进行通信.设置 `fastcgi_pass` 为 `127.0.0.1:9000` 以及 `php-fpm` 的 `listen:0.0.0.0:9000`.通过网络的方式不需要`nginx`和 `php-fpm`在同一个机器上.

### php业务代码编译

:::tip 

- 使用分阶段构建,编译和运行分开,减少不必要的资源引入
- `composer`只在编译阶段引入,运行时无需开启
- 基础 `php-fpm`使用小体积镜像

:::

```Dockerfile

ARG BASE_TAG
# First stage: build the executable.
FROM composer AS builder

# Set the working directory outside $GOPATH to enable the support for modules.
WORKDIR /src

# Import the code from the context.
COPY ./composer.json ./composer.lock /src/

RUN composer install --no-dev --prefer-dist --no-autoloader

COPY ./ /src/

RUN composer dump-autoload -o && composer clear-cache

# Final stage: the running container.
FROM php-fpm  AS final

# Import the compiled executable from the first stage.
COPY --from=builder /src /var/www/html

WORKDIR /var/www/html

EXPOSE 9000

```

下面的图代表了两种打包方式:

### 分开打包

将`nginx`和`php-fpm`分别打包成两个镜像.

![upload successful](/images/pasted-5.png)


### 合并打包

将`nginx`和`php-fpm`打包到同一个镜像,使用`supervisor`,`pm2`等进程管理工具管理进程.

![upload successful](/images/pasted-6.png)



# 镜像部署

目前流行的部署方式一般都是部署到`kubernetes`中,所以我们以`kubernetes`部署为例来分析下几种不同部署方式的优缺点.

## nginx 和 php-fpm 单独部署


将`nginx`和`php-fpm`部署成两个`deployments`,使用`service`进行通信.

![upload successful](/images/pasted-7.png)


:::tip 优点

- `nginx`和`php-fpm`可以进行单独扩缩容
- `nginx`和`php-fpm`可以进行单独配置和更新,不相互影响
- `nginx`和`php-fpm`使用网络进行通信
- 发布代码简单,只需要更新 `php-fpm` 的镜像即可
- 比较灵活,节省资源

:::

:::warning 缺点

- 部署比较麻烦,需要同时暴露`80`,`9000`端口
- 容器的健康检查需要单独检查,无法针对业务接口设置健康检查

:::

## nginx 和 php-fpm 部署到同一个 pod

部署一个`deployments`并使用两个 `container`部署到同一个`pod`里面.
![upload successful](/images/pasted-8.png)


:::tip 优点

- 无需暴露`php-fpm`的端口,内部容器通过共享网络栈通信
- 部署相对简单,通过`pod`内部署多个`container`进行协作完成服务工作

:::

:::warning 缺点

- 扩缩容需要一起,无法单独扩缩容
- 相对浪费资源

:::


## nginx 和 php-fpm 共同打包部署到一个 pod
![upload successful](/images/pasted-9.png)

:::tip 优点

- `pod`内只有一个`container`,部署简单
- 可以检查服务的健康

:::

:::warning 缺点

- 扩缩容需要一起,无法单独扩缩容
- 浪费资源

:::


# 程序运行

## php-fpm pm 管理配置

在`kubernetes`中一般会使用`hpa`根据资源的消耗进行自动扩缩容.如果选用`ondemand`,`dynamic`的方式,内存使用会存在波动,如果针对内存使用进行`hpa`往往来不及等到调度
成功并运行新的`pod`进行服务.在内存充足的情况下使用`static`的方式固定内存使用.`hpa`可以只根据`cpu`进行 hpa会更好.

`PHP_PM_MAX_CHILDREN` 并不是越大越好.我们可以开启`php-fpm`自带的`metrics`来观察 php-fpm的使用情况.

```

; The URI to view the FPM status page. If this value is not set, no URI will be
; recognized as a status page. It shows the following informations:
;   pool                 - the name of the pool;
;   process manager      - static, dynamic or ondemand;
;   start time           - the date and time FPM has started;
;   start since          - number of seconds since FPM has started;
;   accepted conn        - the number of request accepted by the pool;
;   listen queue         - the number of request in the queue of pending
;                          connections (see backlog in listen(2));
;   max listen queue     - the maximum number of requests in the queue
;                          of pending connections since FPM has started;
;   listen queue len     - the size of the socket queue of pending connections;
;   idle processes       - the number of idle processes;
;   active processes     - the number of active processes;
;   total processes      - the number of idle + active processes;
;   max active processes - the maximum number of active processes since FPM
;                          has started;
;   max children reached - number of times, the process limit has been reached,
;                          when pm tries to start more children (works only for
;                          pm 'dynamic' and 'ondemand');
; Value are updated in real time.
; Example output:
;   pool:                 www
;   process manager:      static
;   start time:           01/Jul/2011:17:53:49 +0200
;   start since:          62636
;   accepted conn:        190460
;   listen queue:         0
;   max listen queue:     1
;   listen queue len:     42
;   idle processes:       4
;   active processes:     11
;   total processes:      15
;   max active processes: 12
;   max children reached: 0
;
; By default the status page output is formatted as text/plain. Passing either
; 'html', 'xml' or 'json' in the query string will return the corresponding
; output syntax. Example:
;   http://www.foo.bar/status
;   http://www.foo.bar/status?json
;   http://www.foo.bar/status?html
;   http://www.foo.bar/status?xml
;
; By default the status page only outputs short status. Passing 'full' in the
; query string will also return status for each pool process.
; Example:
;   http://www.foo.bar/status?full
;   http://www.foo.bar/status?json&full
;   http://www.foo.bar/status?html&full
;   http://www.foo.bar/status?xml&full
; The Full status returns for each process:
;   pid                  - the PID of the process;
;   state                - the state of the process (Idle, Running, ...);
;   start time           - the date and time the process has started;
;   start since          - the number of seconds since the process has started;
;   requests             - the number of requests the process has served;
;   request duration     - the duration in µs of the requests;
;   request method       - the request method (GET, POST, ...);
;   request URI          - the request URI with the query string;
;   content length       - the content length of the request (only with POST);
;   user                 - the user (PHP_AUTH_USER) (or '-' if not set);
;   script               - the main script called (or '-' if not set);
;   last request cpu     - the %cpu the last request consumed
;                          it's always 0 if the process is not in Idle state
;                          because CPU calculation is done when the request
;                          processing has terminated;
;   last request memory  - the max amount of memory the last request consumed
;                          it's always 0 if the process is not in Idle state
;                          because memory calculation is done when the request
;                          processing has terminated;
; If the process is in Idle state, then informations are related to the
; last request the process has served. Otherwise informations are related to
; the current request being served.
; Example output:
;   ************************
;   pid:                  31330
;   state:                Running
;   start time:           01/Jul/2011:17:53:49 +0200
;   start since:          63087
;   requests:             12808
;   request duration:     1250261
;   request method:       GET
;   request URI:          /test_mem.php?N=10000
;   content length:       0
;   user:                 -
;   script:               /home/fat/web/docs/php/test_mem.php
;   last request cpu:     0.00
;   last request memory:  0
;
; Note: There is a real-time FPM status monitoring sample web page available
;       It's available in: /usr/local/share/php/fpm/status.html
;
; Note: The value must start with a leading slash (/). The value can be
;       anything, but it may not be a good idea to use the .php extension or it
;       may conflict with a real PHP file.
; Default Value: not set
pm.status_path = /fpm-status

```

以上配置开启了监控.

使用[`php-fpm-exporter`](https://github.com/hipages/php-fpm_exporter)来暴露数据给`prometheus`拉取查看运行情况.

```shell

# 使用 unix 通信 启动 
/usr/local/bin/php-fpm-exporter server --phpfpm.scrape-uri "unix:///dev/shm/php-cgi.sock;/fpm-status" --phpfpm.fix-process-count true
# 使用 tcp 通信 启动
/usr/local/bin/php-fpm-exporter server --phpfpm.scrape-uri "tcp://127.0.0.1:9000/status" --phpfpm.fix-process-count true

```

查看 `Running` `Idle` 数据.可以看到服务一般同时运行的数量,一般同时运行的数量都是比较少的.不足 `10` 个.


![upload successful](/images/pasted-10.png)

如果响应比较慢就会发现同时运行的数量会攀升.部分慢接口会导致整个服务都会被拖垮.

![upload successful](/images/pasted-11.png)

一般接口都要设置服务端的超时和客户端的超时控制.

服务端可以通过`PHP_REQUEST_TERMINATE_TIMEOUT` 控制请求的执行时长.
客户端可以通过设置 `connect_timeout` `read_timeout` 等限制接口的执行时长.
避免慢接口堵塞请求导致服务大面积超时.

```ini
;;输出到控制台
ENV PHP_ERROR_LOG="/proc/self/fd/2" 
ENV PHP_RLIMIT_FILES=51200
ENV PHP_RLIMIT_CORE=0
ENV PHP_USER=www-data
ENV PHP_GROUP=www-data
ENV PHP_LISTEN=0.0.0.0:9000
ENV PHP_PM=static
ENV PHP_PM_MAX_CHILDREN=20
ENV PHP_PM_START_SERVERS=4
ENV PHP_PM_MIN_SPARE_SERVERS=2
ENV PHP_PM_MAX_SPARE_SERVERS=10
ENV PHP_PM_PROCESS_IDLE_TIMEOUT=10s
;;请求超时 10000 次重启 fpm 进程
ENV PHP_PM_MAX_REQUESTS=10000
ENV PHP_SLOWLOG="/proc/self/fd/2"
;;慢接口时长
ENV PHP_REQUEST_SLOWLOG_TIMEOUT="2s"
;;请求最大执行时长
ENV PHP_REQUEST_TERMINATE_TIMEOUT="120s"
;;cli 模式脚本最大执行时长
ENV PHP_MAX_EXECUTION_TIME=600
ENV PHP_MAX_INPUT_TIME=60
;; cli内存限制
ENV PHP_MEMORY_LIMIT=384M
ENV PHP_ERROR_REPORTING="E_ALL & ~E_DEPRECATED & ~E_STRICT"
;;不输出日志
ENV PHP_ACCESS_LOG="/dev/null"
```

## 高峰时期接口超时

1 查看`php-fpm`数量是否不够
2 `cpu`是否不够
3 查看 socket 连接是否不够

之前我们系统迁移`kubernetes`低峰时期运行良好.高峰时期会阶段性的连接超时.
经过排查,发现是 socket 连接数不够用了.我们服务大多是短连接,占用较多的连接数.大量连接数在`TIME_WAIT`.
导致无可用`socket`来建立连接.出现接口大量超时.由于我们没有关注`connect_timeout`.排查起来非常困难.
后来使用`pod`里面的`init_container`来动态修改内核参数.一般修改网络相关的参数.
参考如下:

有部分参数由于`腾讯云虚拟节点`的限制无法修改,所以只改了部分参数.
修改完之后我们连接超时的问题就消失了.

```shell

#!/bin/sh

echo "start optimize"

sysctl -w net.ipv4.tcp_max_syn_backlog=16384
sysctl -w net.ipv4.tcp_max_tw_buckets=32768
sysctl -w net.core.somaxconn=32768
sysctl -w net.ipv4.ip_local_port_range="10000 61000"

echo "optimize done"

```

## 关于日志收集

为了便于请求日式的收集,慢接口的日志收集.我们选择将所有的日志使用`json`的格式输出到控制台上.

> `php-fpm`的慢接口日志无法修改格式

`kubernetes`会将所有容器的控制台日志保存在文件里.我们通过`daemonsets`在每个节点部署`filebeat`收集日志并发送到`elasticsearch`.