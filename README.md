kubernetes1.8.4 部署lnmp+discuz

###在kubernetes中搭建LNMP环境，并安装Discuzx
本实验，需要已经搭建好kubernetes集群和harbor服务。
首先克隆本项目：git clone https://github.com/donxan/k8s_lnmp_discuzx.git

1.下载镜像
mysql5.7+php7用如下镜像：
docker pull mysql:5.7
docker pull richarvey/nginx-php-fpm

2.mysql5.6+php5.6用如下镜像：
基础镜像：
docker pull mysql:5.6
docker pull php:5.6-fpm-alpine

nginx+PHP二合一镜像构建：
创建目录：
mkidr k8s_discuz/nginx-php-fpm

创建替换的配置文件：
cat >>default.conf<<"EOF"
server {
  listen 80;
  server_name localhost;

  root /var/www/html;
  index index.html index.htm index.nginx-debian.html;

  location / {
    try_files $uri $uri/ =404;
  }
  location ~ \.php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
  }
}
EOF

cat >>Dockerfile<<"EOF"
FROM  php:5.6-fpm-alpine
ADD repositories /etc/apk/repositories
ADD default.conf /
ADD index.html /
ADD run.sh /
ADD php.ini /usr/local/etc/php/
RUN apk update && apk add nginx && \
    apk add m4 autoconf make gcc g++ linux-headers && \
    docker-php-ext-install pdo_mysql opcache mysqli && \
    mkdir /run/nginx && \
    mv /default.conf /etc/nginx/conf.d && \
    mv /index.html /var/www/html && \
    touch /run/nginx/nginx.pid && \
    chmod 755 /run.sh && \
    apk del m4 autoconf make gcc g++ linux-headers

EXPOSE 80
EXPOSE 9000

ENTRYPOINT ["/run.sh"]
EOF

cat >>index.html<<"EOF" 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
EOF

cat >>php.ini<<"EOF" 
[PHP]
engine = On
short_open_tag = Off
precision = 14
output_buffering = 4096
zlib.output_compression = Off
implicit_flush = Off
unserialize_callback_func =
serialize_precision = -1
disable_functions =
disable_classes =
zend.enable_gc = On
expose_php = On
max_execution_time = 30
max_input_time = 60
memory_limit = 128M
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
log_errors_max_len = 1024
ignore_repeated_errors = Off
ignore_repeated_source = Off
report_memleaks = On
html_errors = On
variables_order = "GPCS"
request_order = "GP"
register_argc_argv = Off
auto_globals_jit = On
post_max_size = 8M
auto_prepend_file =
auto_append_file =
default_mimetype = "text/html"
default_charset = "UTF-8"
doc_root =
user_dir =
enable_dl = Off
cgi.fix_pathinfo=0
file_uploads = On
upload_max_filesize = 2M
max_file_uploads = 20
allow_url_fopen = On
allow_url_include = Off
default_socket_timeout = 60
[CLI Server]
cli_server.color = On
[Date]
[filter]
[iconv]
[imap]
[intl]
[sqlite3]
[Pcre]
[Pdo]
[Pdo_mysql]
pdo_mysql.default_socket=
[Phar]
[mail function]
SMTP = localhost
smtp_port = 25
mail.add_x_header = Off
[ODBC]
odbc.allow_persistent = On
odbc.check_persistent = On
odbc.max_persistent = -1
odbc.max_links = -1
odbc.defaultlrl = 4096
odbc.defaultbinmode = 1
[Interbase]
ibase.allow_persistent = 1
ibase.max_persistent = -1
ibase.max_links = -1
ibase.timestampformat = "%Y-%m-%d %H:%M:%S"
ibase.dateformat = "%Y-%m-%d"
ibase.timeformat = "%H:%M:%S"
[MySQLi]
mysqli.max_persistent = -1
mysqli.allow_persistent = On
mysqli.max_links = -1
mysqli.default_port = 3306
mysqli.default_socket =
mysqli.default_host =
mysqli.default_user =
mysqli.default_pw =
mysqli.reconnect = Off
[mysqlnd]
mysqlnd.collect_statistics = On
mysqlnd.collect_memory_statistics = Off
[OCI8]
[PostgreSQL]
pgsql.allow_persistent = On
pgsql.auto_reset_persistent = Off
pgsql.max_persistent = -1
pgsql.max_links = -1
pgsql.ignore_notice = 0
pgsql.log_notice = 0
[bcmath]
bcmath.scale = 0
[browscap]
[Session]
session.save_handler = files
session.use_strict_mode = 0
session.use_cookies = 1
session.use_only_cookies = 1
session.name = PHPSESSID
session.auto_start = 0
session.cookie_lifetime = 0
session.cookie_path = /
session.cookie_domain =
session.cookie_httponly =
session.cookie_samesite =
session.serialize_handler = php
session.gc_probability = 1
session.gc_divisor = 1000
session.gc_maxlifetime = 1440
session.referer_check =
session.cache_limiter = nocache
session.cache_expire = 180
session.use_trans_sid = 0
session.sid_length = 26
session.trans_sid_tags = "a=href,area=href,frame=src,form="
session.sid_bits_per_character = 5
[Assertion]
zend.assertions = -1
[COM]
[mbstring]
[gd]
[exif]
[Tidy]
tidy.clean_output = Off
[soap]
soap.wsdl_cache_enabled=1
soap.wsdl_cache_dir="/tmp"
soap.wsdl_cache_ttl=86400
soap.wsdl_cache_limit = 5
[sysvshm]
[ldap]
ldap.max_links = -1
[dba]
[opcache]
[curl]
[openssl]
EOF

cat >>repositories<<"EOF"
https://mirrors.aliyun.com/alpine/v3.11/main/
https://mirrors.aliyun.com/alpine/v3.11/community/
EOF

cat >>run.sh<<"EOF"
#!/bin/sh

# 后台启动
php-fpm -D
# 关闭后台启动，hold住进程
nginx -g 'daemon off;'
EOF

3.mysql5.7+php7环境
用dockerfile重建nginx-php-fpm镜像
cd k8s_discuz/dz_web_dockerfile/
docker build -t nginx-php .

用dockerfile重建mysql5.7镜像
cd k8s_discuz/dz_mysql_dockerfile/
docker build -t mysql5.7 .

mysql5.6+php5.6环境
用dockerfile重建nginx-php-fpm镜像
cd k8s_discuz/nginx-php-fpm
docker build -t nginx-php-fpm:5.6 .

用dockerfile重建mysql5.6镜像
cd k8s_discuz/dz_mysql_dockerfile/
docker build -t mysql:5.6 .

cd k8s_discuz/dz_web_dockerfile/
修改：该目录下面Dockerfile文件中FROM为如下内容：
FROM 192.168.168.219/lnmp/nginx-php-fpm:5.6

根据实际项目需要修改重构nginx-php镜像：
docker build -t nginx-php .

4.将镜像push到harbor
##登录harbor，并push新的镜像
docker login -u admin -p Harbor12345 192.168.168.219  //输入正确的用户名和密码,如果之前已创建secret，就可以免输入密码
docker tag nginx-php 192.168.168.219/lnmp/nginx-php
docker push 192.168.168.219/lnmp/nginx-php

docker tag mysql:5.7 192.168.168.219/lnmp/mysql:5.7
docker push 192.168.168.219/lnmp/mysql:5.7

5.搭建NFS
可以参考以下：
假设kubernetes集群网段为192.168.168.0/24，本机IP为192.168.168.219

安装包
yum install nfs-utils

编辑配置文件
vim /etc/exportfs  //内容如下
/data/k8s/ 192.168.168.0/24(sync,rw,no_root_squash)

启动服务
systemctl start nfs
systemctl enable nfs

创建目录
mkdir -p  /data/k8s/lnmp/{db,web}
注意db，web的属主，db999,web 100.

6.搭建MySQL服务
创建secret (设定mysql的root密码)
kubectl create secret generic mysql-pass --from-literal=password=lnmp123

创建pv
cd ../../k8s_discuz/mysql
kubectl apply -f mysql-pv.yaml

创建pvc
kubectl apply -f mysql-pvc.yaml

创建deployment
kubectl apply -f mysql-dp.yaml 

创建service
kubectl apply -f mysql-svc.yaml


7.搭建Nginx+php-fpm服务
注意搭建步骤，在部署mysql时，不能deploy，svc一起执行，需要一步一步来操作。

搭建pv
cd ../../k8s_discuz/nginx_php
kubectl apply -f web-pv.yaml

创建pvc
kubectl apply -f web-pvc.yaml

创建deployment
kubectl apply -f web-dp.yaml 

创建service
kubectl apply -f web-svc.yaml

8.安装Discuz
下载dz代码 (到NFS服务器上)
cd /tmp/
git clone https://gitee.com/ComsenzDiscuz/DiscuzX.git
cd /data/k8s/lnmp/web/
mv /tmp/DiscuzX/upload/* .
chown -R 100 data uc_server/data/ uc_client/data/ config/

9.设置MySQL普通用户
kubectl get svc dz-mysql //查看service的cluster-ip，我的是10.68.122.120
mysql -uroot -h10.68.122.120 -pDzPasswd1  //这里的密码是在上面步骤中设置的那个密码
> create database dz;
> grant all on dz.* to 'dz'@'%' identified by 'dz-passwd-123';


10.访问首页
http://ip地址/+DiscuzX


11.####  部署traefik ingress
cd ../../k8s_discuz/nginx_php
kubectl apply -f web-ingress.yaml
https://s1.51cto.com/images/blog/201901/20/48b397117a87913cfa8535effcd2136e.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)
如果没有部署ingress，可以使用安装nginx,配置nginx反向代理。

参考如下:
* 设置Nginx代理
注意：目前nginx服务是运行在kubernetes集群里，node节点以及master节点上是可以通过cluster-ip访问到，但是外部的客户端就不能访问了。
      所以，可以在任意一台node或者master上建一个nginx反向代理即可访问到集群内的nginx。

kubectl get svc dz-web //查看cluster-ip，我的ip是10.68.190.99
nginx代理配置文件内容如下：
server {
            listen 80;
            server_name dz.abcgogo.com;

            location / {
                proxy_pass      http://10.68.190.99:80;
                proxy_set_header Host   $host;
                proxy_set_header X-Real-IP      $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}

做完Nginx代理，就可以通过node的IP来访问discuz了。


#### 安装Discuz
按照提示配置即可。
