## Ubuntu下快速部署安装 Nginx + PHP + MySQL 笔记

##### 先更新软件库
    sudo apt-get update

##### 安装 MySQL
    sudo apt-get install mysql-server

##### 安装 Nginx
    sudo apt-get install nginx

##### 安装 php-fpm
    sudo apt-get install php5-fpm

##### 配置 nginx 整合 php
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/app
sudo vi /etc/nginx/sites-available/app

编辑配置文件/etc/nginx/sites-available/app内容如下：
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    root /path/to/laravel/public;
    index index.php index.html index.htm;

    server_name laravel.app;

    location / {
        try_files $uri $uri/ /index.php$query_string;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }

    location ~ \.php$ {
        # With php5-fpm:
        try_files $uri =404;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    location ~ /\.ht {
        deny all;
    }
}
```

##### 接下来在/etc/nginx/sites-enabled目录下创建对应软链接：
    sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app

##### 然后检查配置文件正确性
    sudo service nginx configtest

##### 重新加载配置文件
    sudo service nginx reload

##### 安装 php 常用扩展
    sudo apt-get install php5-mysql php5-curl php5-gd php5-intl php-pear php5-imagick php5-imap php5-mcrypt php5-memcache php5-ming php5-ps php5-pspell php5-recode php5-snmp php5-sqlite php5-tidy php5-xmlrpc php5-xsl php5-xcache

##### 重启 php-fpm
    sudo service php5-fpm restart
