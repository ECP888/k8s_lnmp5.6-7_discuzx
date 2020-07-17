#  kubernetes1.8.4 部署lnmp+discuz

在kubernetes中搭建LNMP环境，并安装Discuzx
本实验，需要已经搭建好kubernetes集群和harbor服务。
首先克隆本项目：git clone https://github.com/donxan/k8s_lnmp_discuzx.git

#  1.下载镜像
mysql5.7+php7用如下镜像：
docker pull mysql:5.7

docker pull richarvey/nginx-php-fpm

#  2.mysql5.6+php5.6用如下镜像：
基础镜像：
docker pull mysql:5.6

docker pull php:5.6-fpm-alpine

#  nginx+PHP二合一镜像构建：

#  3.mysql5.7+php7环境
#### 用dockerfile重建nginx-php-fpm镜像
cd k8s_discuz/dz_web_dockerfile/

docker build -t nginx-php .

#### 用dockerfile重建mysql5.7镜像
cd k8s_discuz/dz_mysql_dockerfile/

docker build -t mysql5.7 .

#### mysql5.6+php5.6环境
用dockerfile重建nginx-php-fpm镜像
cd k8s_discuz/nginx-php-fpm

docker build -t nginx-php-fpm:5.6 .

#### 用dockerfile重建mysql5.6镜像
cd k8s_discuz/dz_mysql_dockerfile/

docker build -t mysql:5.6 .

cd k8s_discuz/dz_web_dockerfile/

修改：该目录下面Dockerfile文件中FROM为如下内容：
FROM 192.168.168.219/lnmp/nginx-php-fpm:5.6

#### 根据实际项目需要修改重构nginx-php镜像：
docker build -t nginx-php .

#  4.将镜像push到harbor
#### 登录harbor，并push新的镜像
docker login -u admin -p Harbor12345 192.168.168.219  //输入正确的用户名和密码,如果之前已创建secret，就可以免输入密码

docker tag nginx-php  192.168.168.219/lnmp/nginx-php

docker push  192.168.168.219/lnmp/nginx-php

docker tag mysql:5.7  192.168.168.219/lnmp/mysql:5.7

docker push  192.168.168.219/lnmp/mysql:5.7

#  5.搭建NFS
#### 可以参考以下：
假设kubernetes集群网段为192.168.168.0/24，本机IP为192.168.168.219

#### 安装包
yum install nfs-utils

#### 编辑配置文件
vim /etc/exportfs  //内容如下

/data/k8s/ 192.168.168.0/24(sync,rw,no_root_squash)

#### 启动服务
systemctl start nfs

systemctl enable nfs

#### 创建目录
mkdir -p  /data/k8s/lnmp/{db,web}

注意db，web的属主，db999,web 100.

#  6.搭建MySQL服务
#### 创建secret (设定mysql的root密码)
kubectl create secret generic mysql-pass --from-literal=password=lnmp123

#### 创建pv
cd ../../k8s_discuz/mysql

kubectl apply -f mysql-pv.yaml

#### 创建pvc
kubectl apply -f mysql-pvc.yaml

#### 创建deployment
kubectl apply -f mysql-dp.yaml 

#### 创建service
kubectl apply -f mysql-svc.yaml


#  7.搭建Nginx+php-fpm服务
注意搭建步骤，在部署mysql时，不能deploy，svc一起执行，需要一步一步来操作。

#### 搭建pv
cd ../../k8s_discuz/nginx_php

kubectl apply -f web-pv.yaml

#### 创建pvc
kubectl apply -f web-pvc.yaml

#### 创建deployment
kubectl apply -f web-dp.yaml 

#### 创建service
kubectl apply -f web-svc.yaml

#  8.安装Discuz
下载dz代码 (到NFS服务器上)
cd /tmp/
git clone https://gitee.com/ComsenzDiscuz/DiscuzX.git

cd /data/k8s/lnmp/web/

mv /tmp/DiscuzX/upload/* .

chown -R 100 data uc_server/data/ uc_client/data/ config/

#  9.设置MySQL普通用户
kubectl get svc dz-mysql //查看service的cluster-ip，我的是10.68.122.120
mysql -uroot -h10.68.122.120 -pDzPasswd1  //这里的密码是在上面步骤中设置的那个密码
> create database dz;
> grant all on dz.* to 'dz'@'%' identified by 'dz-passwd-123';


#  10.访问首页
http://ip地址/+DiscuzX


#  11.部署traefik ingress
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
