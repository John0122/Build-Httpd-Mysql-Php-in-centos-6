# 配置nginx+php+mysql

### 配置 nginx yum 源

```bash
sudo yum install epel-release -y
```

### 安装 nginx

```bash
sudo yum install nginx -y
```

### 启动nginx 设置开启启动

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl restart nginx
```

### 安装mariadb

```bash
sudo yum install mariadb-server mariadb -y
```

### 启动mariadb 设置开机启动

```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo systemctl restart mariadb
```

### 配置mariadb

```bash
1.打开数据库
mysql -uroot -p (会提示输入密码，直接回车就可以)
2.打开mysql库
use mysql;
3.修改密码
update user set password=password('密码') where user='root';
4.应用修改
flush privileges;
5.设置远程访问
grant all privileges on *.* to 'root'@'%' identified by '密码' with grant option;
6.修改端口
vi /etc/my.cnf
[mysqld]下面添加：port=33066
7.重启mysql
sudo systemctl restart mariadb
```

### 安装php 7

```bash
1.下载php7 yum源
sudo yum install php php-mysql php-fpm -y
2.安装php7 yum源
sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm -y
3.安装yum-conifg-manager
sodu yum install yum-utils -y
4.配置php7
sudo yum-config-manager --enable remi-php72
5.安装php7
sudo yum --enablerepo=remi,remi-php72 install php-fpm php-common -y --skip-broken
6.安装php7扩展包
sudo yum --enablerepo=remi,remi-php72 install php-opcache php-pecl-apcu php-cli php-pear php-pdo php-mysqlnd php-pgsql php-pecl-mongodb php-pecl-redis php-pecl-memcache php-pecl-memcached php-gd php-mbstring php-mcrypt php-xml -y
```

### 配置PHP

```bash
1.打开配置文件
sudo vi /etc/php.ini
2.修改
;cgi.fix_pathinfo=1 -> cgi.fix_pathinfo=0
3.保存修改并退出编辑
```

```hash
1.打开配置文件
sudo vi /etc/php-fpm.d/www.conf
2.修改
user = apache -> user = nginx
group = apache -> group = nginx
listen = 127.0.0.1:9000 -> listen = /var/run/php-fpm/php-fpm.sock
;listen.owner = nobody -> listen.owner = nginx
;listen.group = nobody -> listen.group = nginx
;listen.mode = 0660 -> listen.mode = 0660
3.保存修改并退出编辑
```

### 启动php 设置开机启动

```bash
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo systemctl restart php-fpm
```

### 安装CGI

```bash
这里我们使用[Fcgiwrap](https://github.com/gnosek/fcgiwrap)
在本地服务器下载Fcgiwrap源码，编译Fcgiwrap可执行文件

yum -y install fcgi-devel autoconf libtool automake
cd /tmp
git clone git://github.com/gnosek/fcgiwrap.git
cd fcgiwrap
autoreconf -i
./configure
make
make install

将编译好的可执行文件上传到nginx服务器上
scp /usr/local/sbin/fcgiwrap user@dest_server:/usr/local/sbin/
```

```bash
在nginx服务器上安装spawn-fcgi

yum -y install spawn-fcgi fcgi-devel

打开/创建配置文件
vi /etc/sysconfig/spawn-fcgi
添加一下内容
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
#SOCKET=/var/run/php-fcgi.sock
#OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"

FCGI_SOCKET=/var/run/fcgiwrap.socket
FCGI_PROGRAM=/usr/local/sbin/fcgiwrap
FCGI_USER=nginx
FCGI_GROUP=nginx
FCGI_EXTRA_OPTIONS="-M 0770"
OPTIONS="-u $FCGI_USER -g $FCGI_GROUP -s $FCGI_SOCKET -S $FCGI_EXTRA_OPTIONS -F 1 -P /var/run/spawn-fcgi.pid -- $FCGI_PROGRAM"
保存并退出编辑
```

### 启动cgi 设置开机启动

```bash
systemctl start spawn-fcgi
systemctl enable spawn-fcgi
```

### nginx配置文件模版

```bash
server {
    listen       8088;
    server_name  server_domain_name_or_IP;

    # note that these lines are originally from the "location /" block
    root   /home/skynet;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}

listen: 监听的端口
server_name: 服务器IP地址或者域名
root: 站点目录
index: 默认页面
```

### nginx配置php

```bash
在nginx配置文件的server中追加一下内容
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
```

### nginx配置cgi

```bash
在nginx配置文件的server中追加一下内容
    location /cgi-bin/ {
        try_files $uri =404;
        fastcgi_pass  unix:/var/run/fcgiwrap.socket;
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    }
```

### nginx vhost配置示例

```conf
server {
    listen       8090;
    server_name  test.skynetcloud.com;

    # note that these lines are originally from the "location /" block
    root   /home/skynet;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    
    location /cgi-bin/ {
        try_files $uri =404;
        fastcgi_pass  unix:/var/run/fcgiwrap.socket;
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
    }

}

```

### 测试PHP和WEB是否能访问

```bash
添加测试文件
echo '<?php phpinfo(); ?>' > /home/skynet/info.php
```

```bash
从浏览器中访问：
http://test.skynetcloud.com:8090/info.php
```



### 参考连接

[https://www.hostinger.com/tutorials/how-to-install-lemp-centos7#Step-3-Installing-PHP-v710](https://www.hostinger.com/tutorials/how-to-install-lemp-centos7#Step-3-Installing-PHP-v710)

[centos mysql 安装及配置](https://jingyan.baidu.com/article/fec7a1e5f8d3201190b4e782.html)

[查看和关闭SELinux](https://www.jianshu.com/p/01f7436998ae)

### 错误处理

403错误：

可能原因设置 selinux
