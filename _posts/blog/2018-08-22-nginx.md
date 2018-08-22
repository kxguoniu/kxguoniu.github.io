---
layout: post
title: Nginx防火墙
categories: Nginx
description: python实现压缩文件的体会
keywords: Nginx
---
<a href="#一"><h1>一、WAF的基本概念和原理</h1></a>
<a href="#二"><h1>二、Nginx的安装和基本配置</h1></a>
<a href="#2.1"><h2>&emsp;&emsp;**2.1安装nginx**</h2></a>
<a href="#2.2"><h2>&emsp;&emsp;**2.2编辑nginx启动脚本**</h2></a>
<a href="#2.2.1"><h3>&emsp;&emsp;&emsp;&emsp;**Vim支持nginx语法**</h3></a>
<a href="#2.3"><h2>&emsp;&emsp;**2.3测试**</h2></a>
<a href="#2.4"><h2>&emsp;&emsp;**2.4基本配置**</h2></a>
<a href="#三"><h1>三、naxsi与Nginx整合实现WAF</h1></a>
<a href="#3.1"><h2>&emsp;&emsp;**3.1安装naxsi**</h2></a>
<a href="#3.2"><h2>&emsp;&emsp;**3.2配置naxsi**</h2></a>
<a href="#3.3"><h2>&emsp;&emsp;**3.3naxsi动态变量**</h2></a>
<a href="#3.4"><h2>&emsp;&emsp;**3.4naxsi静态变量|Location 内部的指令**</h2></a>
<a href="#3.5"><h2>&emsp;&emsp;**3.5白名单**</h2></a>
<a href="#3.6"><h2>&emsp;&emsp;**3.6规则**</h2></a>
<a href="#3.7"><h2>&emsp;&emsp;**3.7checkrule**</h2></a>
<a href="#3.8"><h2>&emsp;&emsp;**3.8Matchzones (mz)**</h2></a>
<a href="#3.9"><h2>&emsp;&emsp;**3.9Naxsilogs**</h2></a>
<a href="#四"><h1>四、nxapi白规则生成算法</h1></a>
<a href="#4.1"><h2>&emsp;&emsp;**4.1安装nxtool.py工具**</h2></a>
<a href="#4.2"><h2>&emsp;&emsp;**4.2  nxapi生成白名单规则**</h2></a>
-----
<br \\>
<span id="一"><h1>一、WAF的基本概念和原理</h1></a>
&emsp;&emsp;网站安全防护(WAF)基于对http请求的分析，如果检测到请求是攻击行为，则会对请求进行阻断，不会让请求到业务的机器上去，提高业务的安全性，为web应用提供实时的防护  
&emsp;&emsp;Web防火墙，主要是对Web特有入侵方式的加强防护，如DDOS防护、SQL注入、XML注入、XSS等。由于是应用层而非网络层的入侵，从技术角度都应该称为Web IPS，而不是Web防火墙。这里之所以叫做Web防火墙，是因为大家比较好理解，业界流行的称呼而已。由于重点是防SQL注入，也有人称为SQL防火墙。

<span id="二"><h1>二、Nginx的安装和基本配置</h1></a>
<span id="2.1"><h2>2.1安装nginx</h2></a>
```bash
$ yum install zlib  zlib-devel  gcc  gcc-c++  openssl  openssl-devel
$ wget http://nginx.org/download/nginx-1.12.0.tar.gz
# wget https://ftp.pcre.org/pub/pcre/pcre-8.40.tar.gz
$ wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.40.zip
$ tar -xf nginx-1.12.0.tar.gz  -C  /usr/local/
$ tar -xf pcre-8.40.tar.gz  -C  /usr/local/ 
$ rm -rf nginx-1.12.0.tar.gz  pcre-8.40.tar.gz
$ groupadd -r nginx
$ useradd -r -g nginx -s /sbin/nologin nginx
$ mkdir -pv /var/tmp/nginx/client/
$ cd /usr/local/nginx-1.12.0/
$ ./configure \
	--prefix=/usr/local/nginx \
	--with-pcre=/usr/local/pcre-8.40 \
	--sbin-path=/usr/sbin/nginx \
	--conf-path=/etc/nginx/nginx.conf \
	--error-log-path=/var/log/nginx/error.log \
	--http-log-path=/var/log/nginx/access.log \
	--pid-path=/var/run/nginx/nginx.pid \
	--lock-path=/var/lock/nginx.lock \
	--user=nginx \
	--group=nginx \
	--with-debug \
	--with-http_ssl_module \
	--with-http_flv_module \
	--with-http_stub_status_module \
	--with-http_gzip_static_module \
	--http-client-body-temp-path=/var/tmp/nginx/client/ \
	--http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \
	--http-proxy-temp-path=/var/tmp/nginx/proxy/ \
	--http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
	--http-scgi-temp-path=/var/tmp/nginx/scgi
$ make
$ make install
```

<span id="2.2"><h2>2.2编辑nginx启动脚本</h2></a>
```bash
$ touch nginx
$ chmod 755 nginx
```
<br \\>
编辑文件，使文件内容如下：  
```bash
#!/bin/bash
#
# nginx        Startup script for the nginx Server
#
# chkconfig: - 85 15
# description:  Nginx is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server# processname: nginx
# config: /etc/nginx/nginx.conf
# config: /etc/sysconfig/nginx
# pidfile: /var/run/nginx/nginx.pid
#
# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration
. /etc/sysconfig/network

# Check that networking is up
[ "$NETWORKING" = "no" ] && exit 0

if [ -f /etc/sysconfig/nginx ]; then
    . /etc/sysconfig/nginx
fi

nginx=/usr/sbin/nginx
prog=$(basename $nginx)
pidfile=/var/run/nginx/nginx.pid
lockfile=/var/lock/nginx.lock
NGINX_CONF_FILE=/etc/nginx/nginx.conf
RETVAL=0

make_dirs() {
	# make required directories
	user=`nginx -V 2>&1 | grep "configure arguments:" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
	options=`$nginx -V 2>&1 | grep 'configure arguments:'`
	for opt in $options; do
        if [ `echo $opt | grep '.*-temp-path'` ]; then
            value=`echo $opt | cut -d "=" -f 2`
            if [ ! -d "value" ]; then
                # echo "creating" $value
                mkdir -p $value && chown -R $user $value
            fi
        fi
    done
}

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    if [ -f /var/run/nginx/nginx.pid ] && [ -f /var/lock/nginx.lock ]; then
        echo
    else
        daemon $nginx -c $NGINX_CONF_FILE
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && touch $lockfile
        return $RETVAL
    fi
}

stop() {
status -p $pidfile $nginx > /dev/null
if [[ $? = 0 ]]; then
    echo -n $"Stoping $prog: "
    killproc -p $pidfile $prog -QUIT
else
    echo -n $"Stoping $prog: "
    success
fi
RETVAL=$?
echo
[ $RETVAL -eq 0 ] && rm -f $lockfile $pidfile
return $RETVAL
}

reload() {
    $nginx  -t
    if [ $? -eq 0 ]; then
        echo -n $"Reloading $prog: "
        killproc -p $pidfile $nginx -HUP || echo
        RETVAL=$?
        if [ $RETVAL -eq 7 ]; then
            failure $"nginx shutdown"
        fi
        echo
    else
        echo
        failure
    fi
}

# See how we were called.
case "$1" in
start)
    start
    ;;
stop)
    stop
    ;;
status)
    status -p $pidfile $nginx
    RETVAL=$?
    ;;
restart)
    stop
    sleep 1
    start
    ;;
condrestart|try-restart)
    if status -p $pidfile $nginx >&/dev/null; then
        stop
        sleep 1
        start
    fi
    ;;
force-reload|reload)
    reload
    ;;
*)
    echo $"Usage: $nginx {start|stop|restart|condrestart|try-restart|force-reload|reload|status|help}"
    RETVAL=2
esac

exit $RETVAL
```
<br \\>
<span id="2.2.1"><h3>Vim支持nginx语法</h3></span>  
1.下载 nginx.vim  
&emsp;&emsp;http://www.vim.org/scripts/download_script.php?src_id=19394  
2.将 nginx.vim 复制到 vim/syntax 目录  
&emsp;&emsp;根据自身的需要和 vim 的目录来灵活操作，  
&emsp;&emsp;[root@localhost syntax]# pwd  
&emsp;&emsp;/usr/share/vim/vim70/syntax  
&emsp;&emsp;也可以复制到 ~/.vim/syntax/ 用户所在的目录  
3.配置 nginx.vim  
&emsp;&emsp;au BufRead,BufNewFile /etc/nginx/* set ft=nginx  
&emsp;&emsp;在 filetype.vim 文件中加入上面的代码，可以加 vim/filetype.vim 程序目录中，也可以是 ~/.vim/filetype.vim 用户目录中。以上目录或文件不存在的需要自行添加。其中 “/etc/nginx” 为 nginx 配置文件的目录。  
&emsp;&emsp;这样就可以把杂乱的 nginx 配置文件格式化为比较规范和漂亮的 nginx 配置文件了  
<span id="2.3"><h2>2.3测试</h2></a>
&emsp;&emsp;$ mv nginx  /etc/init.d/  
&emsp;&emsp;$ service  nginx  start  
&emsp;&emsp;通过浏览器访问  http://localhost/
<span id="2.4"><h2>2.4基本配置</h2></a>
>正常运行配置

&emsp;&emsp;daemon  on|off			是否让nginx运行与后台，默认on，调试时设置为off，所有信息输出在控制台  
&emsp;&emsp;master_process  on|off		是否以master/worker模式运行，默认on，调试时可以设置为off以方便追踪  
&emsp;&emsp;user userbane [ groupname ]	以那个用户身份运行  
&emsp;&emsp;pid  /path/to/pidfile		指定nginx的pid文件  
&emsp;&emsp;error_log  /path/to/file	level		错误日志存放位置和级别  
&emsp;&emsp;lock_file  /path/to/lock_file	锁文件  
&emsp;&emsp;worker_procrsses  #		work进程的个数  
&emsp;&emsp;worker_rlimit_nofile  #		一个worker进程能够打开的句柄数  
&emsp;&emsp;worker_priority  #			（-20~19）  
>事件相关

&emsp;&emsp;accept_mutex  [on|off]		是否打开nginx负载均衡锁，让多个worker轮流与新的客户端进行通信  
&emsp;&emsp;accept_mutex_delay  #ms	worker进程请求锁失败后等待#ms才能再一次请求锁  
&emsp;&emsp;multi_accept  on|off		是否允许一次地响应多个请求，默认off  
&emsp;&emsp;use [epoll|rtsig|select|poll]	指定使用那种模式  
&emsp;&emsp;worker_commections  #		每个worker能并发的最大请求数  
>HTTP

&emsp;&emsp;include	/path/to/file		包含文件  
&emsp;&emsp;default_type	 
&emsp;&emsp;log_format  main  \*\*\*		设置日志格式  
&emsp;&emsp;access_log  /path/to/logfile	访问日志文件路径  
>网络：

&emsp;&emsp;keepalive_timeout time		保持连接的超时时长，默认为65s  
&emsp;&emsp;keepalive_requests n		在一次长连接上允许承载的最大请求数  
&emsp;&emsp;keepalive_disable [msie6|safari |none]		对指定的浏览器禁止使用长连接  
&emsp;&emsp;tcp_nodelay on|off			对keepalive连接是否使用tcp_nodelay选项  
&emsp;&emsp;client_header_timeout time	读取http请求首部的超时时长  
&emsp;&emsp;client_body_timeout time	读取http请求包体的超时时间  
&emsp;&emsp;save_timeout time			发送响应的超时时长  
>文件操作：

&emsp;&emsp;sendfile  on|off			是否启用sendfile功能  
&emsp;&emsp;open_file_cache  max=N [incative=time]|off		是否打开文件缓存功能  
&emsp;&emsp;Max：用于缓存条目的最大值，允许打开缓存条目最大数  
&emsp;&emsp;Inactive：缓存条目指定时间没有访问自动删除，默认60s  
&emsp;&emsp;open_file_cache_errors  on|off	是否缓存文件找不到或者没有权限等相关信息  
&emsp;&emsp;open_file_cache_valid  time		多长时间检查一次缓存条目中是否超出非活动时长  
&emsp;&emsp;open_file_cache_min_use  #		在incative指定时间内访问超过此次数，则不会被删除  
>客户端请求：

&emsp;&emsp;ignore_invalid_headers on|off		是否忽略不合法的http首部，默认为on，只能用于server和http  
&emsp;&emsp;log_not_found on|off			用户访问的文件不存在时，是否将其记录到错误日志中  
&emsp;&emsp;resolver  address				指定nginx使用的dns服务器地址  
&emsp;&emsp;resolve  timeout				指定DNS解析超时时长，默认为30s  
&emsp;&emsp;server_tokens  on|o			是否在错误页面中显示nginx的版本号  
>SERVER：

&emsp;&emsp;server  { .... }					定义一个虚拟主机  
&emsp;&emsp;LISTEN  
&emsp;&emsp;&emsp;&emsp;listen  address[:port]  
&emsp;&emsp;&emsp;&emsp;listen  port  
&emsp;&emsp;&emsp;&emsp;listen  unix:socket  
&emsp;&emsp;default_server  定义此server为默认server，如果所有server都没有定义，第一个server即为默认>server

&emsp;&emsp;rcvbuf=SIZE		接收缓存大小  
&emsp;&emsp;sndbuf=SIZE		发送缓存大小  
&emsp;&emsp;ssl				必须以ssl连接  
&emsp;&emsp;server_name  [......]  
&emsp;&emsp;&emsp;&emsp;Server_name可以跟多个主机名，名称可以使用通配符和正则表达式，nginx收到请求比较server值的方式  
&emsp;&emsp;&emsp;&emsp;（1）先做精确匹配  
&emsp;&emsp;&emsp;&emsp;（2）左侧通配符匹配  
&emsp;&emsp;&emsp;&emsp;（3）右侧通配符匹配  
&emsp;&emsp;&emsp;&emsp;（4）正则表达式匹配  
&emsp;&emsp;location  [ =|~|~\*|^~ ]  uri  { ...... }  
&emsp;&emsp;&emsp;&emsp;允许根据用户请求的URI来匹配指定的各location已进行访问控制，匹配到时将被location块中的配置处理  
&emsp;&emsp;&emsp;&emsp;=	精确匹配  
&emsp;&emsp;&emsp;&emsp;~	正则匹配，区分大小写  
&emsp;&emsp;&emsp;&emsp;~*	正则匹配，不区分大小写  
&emsp;&emsp;&emsp;&emsp;^~	只需前半部分与uri匹配即可，不检查正则表达式  
&emsp;&emsp;&emsp;&emsp;匹配优先级  
&emsp;&emsp;&emsp;&emsp;字符字面量最精确匹配、正则表达式检索（由多个时，由第一个匹配到的所处理），按字符字面量  
>文件路径的定义：

&emsp;&emsp;root  /pathfile			设置web资源路径，用于指定请求的根文档目录  
&emsp;&emsp;alias  /pathfile			指定路径别名，只能用于location中，从最后一个/开始匹配  
&emsp;&emsp;index  file....			默认页面，可以有多个值，自左向右匹配  
&emsp;&emsp;error_page  code ......  Uri	当对某个请求发回错误时，如果匹配上了code 则重定向到新的uri  
&emsp;&emsp;try_files  path1  path2 .... Uri;	自左向右尝试读取有path所指定路径，在第一找到即停止并返回，如  			果所有path均不存在，则返回最后一个uri  
>访问控制access模块

&emsp;&emsp;Allow  
&emsp;&emsp;Deny  
&emsp;&emsp;自上而下一次认证，默认通过  
>URL rewrite

&emsp;&emsp;rewrite regex replacement [flag];  
&emsp;&emsp;rewrite  ^/imgages/(.*)$  /imgs/$1  
&emsp;&emsp;rewrite_log  on|off		是否将重写过程记录在错误日志中，默认为notice级别；默认为off  
  
>http核心模块的内置变量：  

&emsp;&emsp;$uri:当前请求的uri，不带参数  
&emsp;&emsp;$request_uri：请求的uri，带完整参数  
&emsp;&emsp;$host：http请求报文中host首部；如果请求中没有host首部，则以处理此请求的主机的主机名代替  
&emsp;&emsp;$hostname：nginx服务运行所在主机的主机名  
&emsp;&emsp;$remote_addr：客户端IP  
&emsp;&emsp;$remote_port: 客户端port  
&emsp;&emsp;$remote_user：使用用户认证时客户端用户输入的用户名  
&emsp;&emsp;$request_filename：用户请求中的URI经过本地root或alias转换后映射的本地的文件路径  
&emsp;&emsp;$request_method：请求方法  
&emsp;&emsp;$server_addr：服务器地址  
&emsp;&emsp;$server_name: 服务器名称  
&emsp;&emsp;$server_port：服务器端口  
&emsp;&emsp;$server_protocol：服务器向客户端发送响应时的协议，如http/1.1，http/1.0  
&emsp;&emsp;$scheme:在请求中使用的scheme 映射协议本身的协议  
&emsp;&emsp;$http_HEADER:匹配请求报文中指定的HEADER，$http_host匹配请求报文中的host首部  
&emsp;&emsp;$sent_http_HEADER:匹配响应报文中指定的HERDER，例如$http_content_type匹配相应报文中的content-type首部  
&emsp;&emsp;$document_root：当前请求映射到的root配置
<span id="三"><h1>三、naxsi与Nginx整合实现WAF</h1></a>
<span id="3.1"><h2>3.1安装naxsi</h2></a>
```bash
$  wget  https://github.com/nbs-system/naxsi/archive/master.zip
$  unzip master.zip
$  mv  naxsi-master  /usr/local/naxsi
$  cd  /usr/local/nginx-1.12.0/
$  ./configure \
	--add-module=/usr/local/naxsi/naxsi_src \
	--prefix=/usr/local/nginx \
	--with-pcre=/usr/local/pcre-8.40 \
	--with-pcre-jit \
	--sbin-path=/usr/sbin/nginx \
	--conf-path=/etc/nginx/nginx.conf \
	--error-log-path=/var/log/nginx/error.log \
	--http-log-path=/var/log/nginx/access.log \
	--pid-path=/var/run/nginx/nginx.pid \
	--lock-path=/var/lock/nginx.lock \
	--user=nginx \
	--group=nginx \
	--with-debug \
	--with-http_ssl_module \
	--with-http_flv_module \
	--with-http_stub_status_module \
	--with-http_gzip_static_module \
	--http-client-body-temp-path=/var/tmp/nginx/client/ \
	--http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \
	--http-proxy-temp-path=/var/tmp/nginx/proxy/ \
	--http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
	--http-scgi-temp-path=/var/tmp/nginx/scgi \
	--without-http_fastcgi_module \
	--without-http_uwsgi_module \
	--without-http_scgi_module \
$  make
$  make install
```
<span id="3.2"><h2>3.2配置naxsi</h2></a>
```bash
$  cp  /usr/local/naxsi/naxsi_config/naxsi_core.rules  /etc/nginx/			#核心规则
$  vim  /etc/nginx/my_naxsi.rules
	LearningMode; 			#学习模式，开启该模式，不会拦截任何请求
	SecRulesEnabled;			#启用naxsi
	#SecRulesDisabled;
	DeniedUrl "/RequestDenied";			#拦截后被重定向的地址

	## check rules
	CheckRule "$SQL >= 8" BLOCK;
	CheckRule "$RFI >= 8" BLOCK;
	CheckRule "$TRAVERSAL >= 4" BLOCK;
	CheckRule "$EVADE >= 4" BLOCK;
	CheckRule "$XSS >= 8" BLOCK;

$  vim  /etc/nginx/nginx.conf  
	user  nginx nginx;  
	worker_processes  1;  
	error_log   /var/log/nginx/error.log;  
	#error_log  logs/error.log;  
	#error_log  logs/error.log  notice;  
	#error_log  logs/error.log  info;  
	
	#pid        logs/nginx.pid;  
	
	
	events {  
	    worker_connections  1024;  
	}  
	
	
	http {
	    include       /etc/nginx/naxsi_core.rules;
	    include       mime.types;
	    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
	                      '$status $body_bytes_sent "$http_referer" '
	                      '"$http_user_agent" "$http_x_forwarded_for" "$request_body" "$http_content_type"';
	    default_type  application/octet-stream;
	    sendfile        on;
	    keepalive_timeout  65;
	    server {
	        listen       80;
	        server_name  directory1;
	        #proxy_set_header Proxy-Connection "";
	        access_log  /var/log/nginx/host.access.log  main;
	        error_log   /var/log/nginx/host.error.log info;
	        location / {
	            include  /etc/nginx/my_naxsi.rules;
	            root   html;
	            index  index.html index.htm;
	        }
	        location /RequestDenied {
	            return 418;
	        }
			error_page   500 502 503 504  /50x.html;
			location = /50x.html {
				root   html;
			}
		}
	}
```
**查看效果**  
&emsp;&emsp;$  service  nginx  start  
&emsp;&emsp;$  curl  ‘http://localhost:80/?a=<>’      #访问测试    结果为不通过，查看错误日志为  
&emsp;&emsp;2017/05/15 15:35:29 [error] 112874#0: *17 NAXSI_FMT:   ip=192.168.56.1&server=192.168.56.153&uri=/   &learning=0&vers=0.55.3&total_processed=4&total_blocked=4&block=1&cscore0=$XSS&score0=8&zone0=ARGS&id0=1302&var_name0=a, client: 192.168.56.1, server: directory1, request: "GET /?a=%3C%3E HTTP/1.1", host: "192.168.56.153"
<span id="3.3"><h2>3.3naxsi动态变量</h2></a>
```bash
Set  $naxsi_flag_learning  0|1  		如果存在，该变量将覆盖naxsi学习标志（“0”禁用学习，“1”启用它）。
Set  $naxsi_flag_post_action  0|1  		如果存在并设置为“0”，则该变量可用于在学习模式下禁用post_action。
Set  $naxsi_flag_enable  0|1  			如果存在，该变量将覆盖naxsi的“SecRulesEnabled”（“0”以禁用naxsi，“1”启用）。如果naxsi_flag_enable变量存在且设置为0，则在此请求中将禁用naxsi。这允许您在特定条件下部分禁用naxsi。要完全禁用“信任”用户的naxsi：
Set  $naxsi_extensive_log   0|1			如果存在（并设置为“1”），该变量将强制naxsi记录变量匹配规则的内容
Set  $naxsi_flag_libinjection_sql	0|1	可以在运行时设置的标志来启用或禁用libinjection的sql检测
Set  $naxsi_flag_libinjection_xss	0|1	可以在运行时设置的标志来启用或禁用libinjection的xss检测
##这些变量应该在location之前设置才能生效，在location之内设置不会有效
```
&emsp;&emsp;动态变量 $naxsi_flag_learning、 $naxsi_flag_enable 、$naxsi_extensive_log  必须在LearningMode、SecRulesEnabled存在的情况下才能生效，如果location内部没有设置学习模式和开启naxsi拦截，那么在location外部设置动态变量是没有效果的。  
&emsp;&emsp;Location内部设置静态学习、拦截模式，外部的动态变量的值就决定着学习和拦截模式的开关，  
&emsp;&emsp;Location内部没有设置静态学习、拦截模式，外部的变量无论何值都没用<br \\>
&emsp;&emsp;动态变量就在于可以根据条件决定学习、拦截模式是否开启  
&emsp;&emsp;例如：如果客户端IP是127.0.0.1  就禁用naxsi  
&emsp;&emsp;# Disable naxsi if client ip is 127.0.0.1  
&emsp;&emsp;&emsp;if ($remote_addr = "127.0.0.1") {  
&emsp;&emsp;&emsp;&emsp;set $naxsi_flag_enable 0;  
&emsp;&emsp;}

<span id="3.4"><h2>3.4naxsi静态变量|Location 内部的指令</h2></a>
```bash
DeniedUrl “/directory”;	naxsi内部重定向  阻止请求
LearningMode；		在该位置开启学习模式
SecRulesEnabled；	启用naxsi的强制关键字
SecRulesDisabled；	禁用naxsi的关键字
CheckRule “$SQL >=8 ” BLOCK;	规则，可以多个
BasicRule			用于声明规则或白名单指令
MainRule			用于声明规则或白名单指令  http内部指令
LibInjectionXss		一个指令，以使libinjection的XSS检测上所有的HTTP请求的一部分
LibInjectionSql		一个指令，以使libinjection的SQL检测上所有的HTTP请求的一部分
```
<span id="3.5"><h2>3.5白名单</h2></a>
![图片找不到啦！](../../images/baimingdan.png)  
&emsp;&emsp;Wl：0		列出所有规则  
&emsp;&emsp;Wl：42		白名单规则42  
&emsp;&emsp;Wl：41 42 43	白名单规则41、42、43  
&emsp;&emsp;Wl：-42		除了规则42外，列出所有规则(>=1000)  
&emsp;&emsp;Mz：		mz是匹配区  
<br \\>
白名单示例  
&emsp;&emsp;BasicRule wl:1100 "mz:$ARGS_VAR:redirect_to";  
&emsp;&emsp;BasicRule wl:1000 "mz:$BODY_VAR:save";  
&emsp;&emsp;BasicRule wl:1402 "mz:$HEADERS_VAR:content-type";  
&emsp;&emsp;BasicRule wl:1000 "mz:URL|$URL:/wp-admin/update.php";
<span id="3.6"><h2>3.6规则</h2></a>
![图片找不到啦！](../../images/guize.png)  
&emsp;&emsp;1>内部规则1-999 协议解析中的异常问题  
&emsp;&emsp;2>SQL注入规则1000-1099  
&emsp;&emsp;3>OBVIOUS RFI规则1100-1100  
&emsp;&emsp;4>Directory traversal规则1200-1299  
&emsp;&emsp;5>XSS规则1300-1399  
&emsp;&emsp;6>绕过规则1400-1500  
&emsp;&emsp;7>文件上传1500-1600  
<br \\>
&emsp;&emsp;Id：num			规则唯一的数字ID，将在NAXSI_FMT白名单中使用  
&emsp;&emsp;匹配模式  
&emsp;&emsp;s：$LABEL:SCORE	得分  
&emsp;&emsp;Mz：				mz是匹配区域  
&emsp;&emsp;Msg：			人类可读的信息  
<br \\>
规则示例  
&emsp;&emsp;MainRule "rx:select|union|update|delete|insert|table|from|ascii|hex|unhex|drop" "msg:sql keywords" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1000;  
&emsp;&emsp;MainRule "str:\"" "msg:double quote" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:8,$XSS:8" id:1001;  
&emsp;&emsp;MainRule negative "rx:multipart/form-data|application/x-www-form-urlencoded" "msg:Content is neither mulipart/x-www-form.." "mz:$HEADERS_VAR:Content-type" "s:$EVADE:4" id:1402;
<span id="3.7"><h2>3.7checkrule</h2></a>
![图片找不到啦！](../../images/checkrule.png)  
&emsp;&emsp;Checkrules指示naxsi采取动作  
&emsp;&emsp;CheckRule  “$SQL >= 8” BLOCK;
<span id="3.8"><h2>3.8Matchzones (mz)</h2></a>
![图片找不到啦！](../../images/matchzones.png)  
Mz粗匹配  
&emsp;&emsp;ARGS：GET args  
&emsp;&emsp;HEADERS：HTTP Headers  
&emsp;&emsp;BODY：POST args（and RAW_BODY）  
&emsp;&emsp;URL：The URL itself（before“？”）  
&emsp;&emsp;FILE_EXT：文件名  
&emsp;&emsp;RAW_BODY：对一个HTTP请求的主体解析表示  
Mz具体匹配  
&emsp;&emsp;$ARGS_VAR:string： GET 参数  
&emsp;&emsp;$HEADERS_VAR:string ： HTTP header  
&emsp;&emsp;$BODY_VAR:string： POST 参数  
Mz正则表达式  
&emsp;&emsp;$HEADERS_VAR_X:regex：正则表达式匹配一个名为HTTP头（> = 0.52）  
&emsp;&emsp;$ARGS_VAR_X:regex：正则表达式匹配GET参数的名称  
&emsp;&emsp;$BODY_VAR_X:regex：正则表达式匹配POST参数的名称  
&emsp;&emsp;一个matchzones可以使用正则表达式  
&emsp;&emsp;$URL：string		：限定此网址  
&emsp;&emsp;$URL_X：regex	：限定匹配的网址  
正则表达式和静态不能混用
<span id="3.9"><h2>3.9Naxsilogs</h2></a>
NAXSI_FMT  
&emsp;&emsp;ip ：客户端的ip  
&emsp;&emsp;server：请求的主机名（如http头文件Host所示）  
&emsp;&emsp;uri：请求的URI（没有参数，?分割）  
&emsp;&emsp;learning：告诉naxsi是否处于学习模式（0/1）  
&emsp;&emsp;vers：Naxsi版本  
&emsp;&emsp;total_processed：nginx的工作人员处理的请求总数  
&emsp;&emsp;total_blocked：（naxsi）nginx的worker阻塞的请求总数  
&emsp;&emsp;zoneN：发生匹配第N+1个区域  
&emsp;&emsp;idN：匹配到的第N+1个规则id  
&emsp;&emsp;var_nameN：发生匹配的第N+1个变量名称（可选）  
&emsp;&emsp;cscoreN：第N+1个命名得分标签  
&emsp;&emsp;scoreN：相关联的第N+1个命名得分值  
&emsp;&emsp;2017/05/15 15:35:29 [error] 112874#0: *17 NAXSI_FMT: ip=192.168.56.1&server=192.168.56.153&uri=/&learning=0&vers=0.55.3&total_processed=4&total_blocked=4&block=1&cscore0=$XSS&score0=8&zone0=ARGS&id0=1302&var_name0=a, client: 192.168.56.1, server: directory1, request: "GET /?a=%3C%3E HTTP/1.1", host: "192.168.56.153"
<span id="四"><h1>四、nxapi白规则生成算法</h1></a>
白名单生成方法（基于分析nginx日志，工具分析的是记录naxsi waf拦截事件的error日志）如下  
&emsp;&emsp;(1)  手动添加  
&emsp;&emsp;(2)  自动生成  

原理  
&emsp;&emsp;Nxapi将WAF事件（学习模式下产生的NAXSI_FMT或NAXSI_EXLOG日志文件）存储在elasticsearch中，然后将自定义模板（tpl文件）转化为检索条件使用elasticsearch进行检索，最后将检索出来的内容与评分条件相比较来生成白名单。  
&emsp;&emsp;亮点是elasticsearch的优秀检索能力，我们能轻易的按关键字查询出TOP N等统计数据，例如触发异常的server Top 10，URI Top 10，Zone（URI组件）Top 10， IP Top 10；  
评分条件如下  
&emsp;&emsp;rule_ip_count : &emsp;nb of peers hitting rule  
&emsp;&emsp;rule_uri_count : &emsp;nb of uri the rule hitted on  
&emsp;&emsp;template_ip_count : &emsp;nb of peers hitting template  
&emsp;&emsp;template_uri_count : &emsp;nb of uri the rule hitted on  
&emsp;&emsp;ip_ratio_template : &emsp;ratio of peers hitting the template vs peers hitting the rule  
&emsp;&emsp;uri_ratio_template : &emsp;ratio of uri hitting the template vs uri hitting the rule  
&emsp;&emsp;ip_ratio_global : &emsp;ratio of peers hitting the rule vs all peers  
&emsp;&emsp;uri_ratio_global : &emsp;ratio of uri hitting the rule vs all uri
<span id="4.1"><h2>4.1安装nxtool.py工具</h2></a>
```bash
$  cd  /usr/local/naxsi/nxapi/
$  python  setup.py  build
$  python  setup.py  install
$  vim  /usr/local/etc/nxapi.json		修改nxapi配置文件
	"elastic" : {
		 "host" : "192.168.56.160:9200",						ES的地址
		 "index" : "nxapi",
		 "doctype" : "events",
		 "default_ttl" : "7200",
		 "max_size" : "1000"
	},
	"naxsi" : {
		 "rules_path" : "/etc/nginx/naxsi_core.rules", 			#naxsi waf的配置路径
		 "template_path" : [ "tpl/"],
		 "geoipdb_path" : "nx_datas/country2coords.txt"
	},
```
&emsp;&emsp;$  nxtool.py  -c  /usr/local/etc/nxapi.json  -x  
可能报错：  
&emsp;&emsp;Traceback (most recent call last):  
&emsp;&emsp;&emsp;File "/usr/bin/nxtool.py", line 6, in <module>  
&emsp;&emsp;&emsp;&emsp;import elasticsearch  
&emsp;&emsp;ImportError: No module named elasticsearch  
解决：  
&emsp;&emsp;安装elasticsearch
```bash
$  wget  https://bootstrap.pypa.io/get-pip.py
$  python  get-pip.py
$  pip  install  elasticsearch
```
导入与查看数据：
```
$  nxtool.py  -c  /usr/local/etc/nxapi.json  --files=/var/log/nginx/error.log		导入WAF拦截的日志数据
$  nxtool.py  -c  /usr/local/etc/nxapi.json  -x			查看
	# size :1000
	# Whitelist(ing) ratio :
	# Top servers :
	# 192.168.56.158 60.36% (total:664/1100)
	# niu 39.64% (total:436/1100)
	# Top URI(s) :
	# /qiong/kai/xin/ 36.18% (total:398/1100)
	# /qiong/kai/xin/index.html 33.27% (total:366/1100)
	# /qiong/kai/xin/index.php 29.45% (total:324/1100)
	# / 1.09% (total:12/1100)
	# Top Zone(s) :
	# ARGS 100.0% (total:1100/1100)
	# Top Peer(s) :
	# 192.168.56.1 60.36% (total:664/1100)
	# 192.168.56.158 39.64% (total:436/1100)
```
<span id="4.2"><h2>4.2  nxapi生成白名单规则</h2></a>
```bash
$  nxtool.py -c nxapi.json -s www.x1.fr -f --filter 'uri /foo/bar/test' --slack
...
#msg: A generic whitelist, true for the whole uri
#Rule (1303) html close tag
#total hits 126
#content : 	lyiuqhfnp,+<a+href="http://preemptivelove.org/">Cialis+forum</a>,+KKSXJyE,+[url=http://preemptivelove.org/]Viagra+or+	cialis[/url],+XGRgnjn,+http
#content : 	4ThLQ6++<a+href="http://aoeymqcqbdby.com/">aoeymqcqbdby</a>,+[url=http://ndtofuvzhpgq.com/]ndtofuvzhpgq[/url],	+[link..
#peers : x.y.z.w
...
#uri : /faq/
#var_name : numcommande
#var_name : comment
...
# success : global_rule_ip_ratio is 58.82
# warnings : rule_ip is 10
BasicRule wl:1303 "mz:$URL:/foo/bar/test|BODY";
```