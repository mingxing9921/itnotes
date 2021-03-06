 部署lnmp环境

[TOC]

LNMP（linux,nginx,mariadb,php）部署，以下默认在root权限下操作，以centos７为例。

# 安装

- 安装nmp(nginx-mariadb-php)

`yum install nginx mariadb-server php php-fpm`

- 设置开机启动并立即启动服务：

`systemctl enable nginx mariadb php-fpm && systemctl start nginx mariadb php-fpm`

- 可安装phpmyadmin方便管理mariadb数据库：

`yum install phpmyadmin`

# 配置

## mariadb配置

`mysql_secure_installation`

回车>根据提示输入Y>输入2次密码(不建议无密码)>回车>根据提示一路输入Y>最后出现：Thanks for using MariaDB!

## php配置

- 修改php-fpm的执行用户为nginx组的nginx（默认为apache组的apache）

  编辑/**etc/php-fpm.d/www.conf**，修改用户名和组：

  ```shell
  user = nginx #修改用户为nginx
  group = nginx #修改组为nginx
  ```

- 将储存php会话(session)记录文件夹权限赋给nginx组的nginx（默认属于apache组的apache）：

  ```shell
  chown nginx:nginx /var/lib/php/session -R
  ```

提示：自定义session路径，可在`/etc/php.ini`中找到`session.save_path`行，去掉其注释，指定自定义路径值

## nginx配置

在[/etc/nginx/nginx.conf](nginx/nginx.conf)使用`include conf.d/*.conf;` ，而从`/tec/nginx/conf.d`中引入各个配置文件。

在`/etc/nginx/conf.d/`中新建一个.conf文件，如website.conf，内容如下(据情况修改)：
```nginx
server {
  listen 80;     #80是默认的端口
  server_name www.xxx.com;    #服务器名
  root /srv/web;    #ngnix默认的主目录，可根据具体情况修改
  index index.html index.php;    #默认主页
  charset utf-8,gbk;    #防止中文乱码可加上
}
```

### php解析

在server中添加[php解析](nginx/backend-parse/php)：

```nginx
location ~ \.php$ {
  fastcgi_pass 127.0.0.1:9000;
  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  include fastcgi_params;
}
```

### SSL和HTTP2

使用ssl和http2，需在listen后的端口号后面加上ssl/http2；填写ssl的证书路径和私钥路径。示例（仅示例server中ssl和http2相关配置部分）：

```nginx
server{
  listen  443 ssl http2;
  ssl_certificate  /etc/letsencrypt/live/xx.xxx/fullchain.pem;
  ssl_certificate_key  /etc/letsencrypt/live/xx.xx/privkey.pem;
  ssl_session_cache  shared:SSL:1m;
  ssl_session_timeout  10m;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;
}
```

http地址跳转到https地址可以新建一个server，示例：

```nginx
server{
  listen 80;
  server_name xxx;
  return 301 https://$server_name$request_uri;
}
```

### 禁止通过ip直接访问网站

[禁止使用ip访问](nginx/conf.d/donotvisitbyip.conf)以防止恶意解析，添加一个新的server：

```nginx
server{
  listen 80; 
  server_name ip;    #ip处写上ip地址
  return 444;
}
```

### 配置websocket

WebSocket协议的握手兼容于HTTP的，使用HTTP的`Upgrade`设置可以将连接从HTTP升级到WebSocket。配置示例（server内其他内容略）：

```nginx
location /wsapp/ {
  proxy_pass https://wsapp.xx.xxx;
  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
```

### 子域名访问对应的子目录

*如abc.xx.com访问xx.com/abc*

1. 确保在域名解析服务商设置了泛解析：使用A记录，主机记录填写`*`

2. 配置一个server，简单[示例](nginx/conf.d/donotvisitbyip.conf)：

   ```nginx
   server{
     listen 80;
     server_name ~^(?<subdomain>.+).xx.com$;
     root   /srv/web/$subdomain;
     index index.html;
   }
   ```

### 目录浏览

在server（或者指定的location中）添加（示例[autoindex](nginx/indexview/autoindex) ）：

```nginx
autoindex on;
autoindex_exact_size off;
autoindex_localtime on;
```

- [fancy插件](https://github.com/aperezdc/ngx-fancyindex) ：如果要修改目录浏览页面的样式需要使用

  1. 在server中添加[fancy配置](nginx/indexview/fancy)（使用fancy配置就不要再添加autoindex相关配置了）：

  ```nginx
  fancyindex on;
  fancyindex_exact_size off;
  fancyindex_localtime on;
  fancyindex_name_length 255;

  fancyindex_header "/fancyindex/header.html";
  fancyindex_footer "/fancyindex/footer.html";
  fancyindex_ignore "/fancyindex";
  ```
  2. 添加相应位置的header.html和footer.html页面（可以是空白页面）

     在header.html和footer.html进行目录浏览页面相关配置。


  3. 配置fancy后提示unknown directive "fancyindex" :

     在/etc/nginx/nginx.conf文件中加载fancy模块（例如该模块位于/usr/lib/nginx/modules下）：`load_module "/usr/lib/nginx/modules/ngx_http_fancyindex_module.so";` 。

- 目录浏览加密

  可以用htpasswd工具来生成密码，然后在要加密的目录的location中单独[配置](nginx/indexview/passlock)：

  ```nginx
  auth_basic "passwd";  #passwd是使用htpasswd生成的密码
  auth_basic_user_file /var/www/html/.htpasswd;  #密码文件路径
  ```

## phpmyadmin配置

将phpMyAdmin复制`/usr/share/phpMyAdmin`到web根目录`/srv/web`下，或者创建一个软链接：

```shell
ln -s /usr/share/phpMyAdmin /usr/share/nginx/html
```

提示：有的发行版中，通过包管理安装的phpmyadmin位于`/usr/share/webapps`目录下。

### 权限问题

如果出现“403forbiden”，可能是该目录下没有index规定的默认主页文件（如index.html）或者nginx的执行用户不具有读取该目录的权限。可以用以下方法解决：

- 确保正确的读取权限

  文件644（rw-r--r--），文件夹755（rwx-r-xr-x）。假如nginx的执行用户是nginx组的nginx，web主目录是/srv/http，可使用以下命令修改所有权限：

  ```shell
  chown -R nginx.nginx /srv/http/
  find /srv/web/ -type f -exec chmod 644 {} \;
  find /srv/web/ -type d -exec chmod 755 {} \;
  ```


- 给予该用户相应权限，如将执行用户（假如执行用户名为nginx）加入具有读取该目录的用户组（假如该用户组是users）中`useradd -aG users nginx` 。
- 换用具有权限的用户执行，如换用root用户，在`/etc/nginx/nginx.conf`中将user改为root。

## 测试

配置完后，测试前重启所有服务：

`systemctl restart nginx mariadb php-fpm`

- 测试nginx：

`nginx -t`  

成功则返回如下内容：
>nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
>nginx: configuration file /etc/nginx/nginx.conf test is successful

- 登录网站测试，在浏览器打开域名或IP。

- 测试php解析：
  添加phpinfo.php测试文件到根目录，其内容为：

```php
<?php
phpinfo();
?>
```
保存后，打开网站，例如网址是xxx.com，浏览xxx.com/info.php，就可以看到php详情页面。

- mariadb测试，以主目录下phpMyAdmin名字未更改为例，例如网址是xxx.com，浏览xxx.com/phpMyAdmin进入到mariadb的登录页面，用户名root，密码是mariadb配置时输入的密码。

