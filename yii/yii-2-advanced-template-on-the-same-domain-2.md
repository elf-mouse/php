# Yii2的高级应用程序模板中设置隐藏 index.php 步骤

如果不想把前台和后台的域名区分开来的朋友可以看下下面这个教程

修改 `advanced/backend/config/main.php` 文件如下:

```php
return [
    'homeUrl' => '/admin',
    'components' => [
        'request' => [
            'baseUrl' => '/admin',
        ],
        'urlManager' => [
            'enablePrettyUrl' => true,
            'showScriptName' => false,
        ],
    ],
];
```

同样修改 `advanced/frontend/config/main.php` 文件:

```php
return [
    'homeUrl' => '/',
    'components' => [
        'request' => [
            'baseUrl' => '',
        ],
        'urlManager' => [
            'enablePrettyUrl' => true,
            'showScriptName' => false,
        ],
    ],
];
```

接着设置服务器, 这里先以 `apache` 为例.

首先设置一下虚拟主机:

```
<VirtualHost *:80>
    ServerName advanced.loc
    ServerAlias www.advanced.loc

    DocumentRoot "/path/to/advanced"
    <Directory "/path/to/advanced">
        AllowOverride All
    </Directory>
</VirtualHost>
```

然后在站点根目录下创建 `.htaccess` 文件为:

```
# prevent directory listings
Options -Indexes
# follow symbolic links
Options FollowSymlinks
RewriteEngine on

RewriteCond %{REQUEST_URI} ^/admin/$
RewriteRule ^(admin)/$ /$1 [R=301,L]
RewriteCond %{REQUEST_URI} ^/admin
RewriteRule ^admin(/.+)?$ /backend/web/$1 [L,PT]

RewriteCond %{REQUEST_URI} ^.*$
RewriteRule ^(.*)$ /frontend/web/$1
```

然后在 `advanced/backend/web` 目录中创建 `.htaccess` 文件, 内容如下:

```
# use mod_rewrite for pretty URL support
RewriteEngine on
# if a directory or a file exists, use the request directly
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
# otherwise forward the request to index.php
RewriteRule . index.php
```

然后在 `advanced/frontend/web` 目录中复制一份上面的 `.htaccess` 文件

__Nginx__ 下的环境配置

Nginx 下的配置可能稍微复杂一些, 这里直接贴出配置, 大家请根据自己的需要进行相应的修改:

```
server {
    charset      utf-8;
    client_max_body_size  200M;

    listen       80; ## listen for ipv4
    #listen       [::]:80 default_server ipv6only=on; ## listen for ipv6

    server_name  advanced.loc;
    root         /path/to/advanced;

    access_log   /path/to/logs/advanced.access.log main buffer=50k;
    error_log    /path/to/logs/advanced.error.log warn;

    location / {
        root  /path/to/advanced/frontend/web;

        try_files  $uri /frontend/web/index.php?$args;

        # avoiding processing of calls to non-existing static files by Yii
        location ~ \.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar)$ {
            access_log  off;
            expires  360d;

            try_files  $uri =404;
        }
    }

    location /admin {
        alias  /path/to/advanced/backend/web;

        rewrite  ^(/admin)/$ $1 permanent;
        try_files  $uri /backend/web/index.php?$args;
    }

    # avoiding processing of calls to non-existing static files by Yii
    location ~ ^/admin/(.+\.(js|css|png|jpg|gif|swf|ico|pdf|mov|fla|zip|rar))$ {
        access_log  off;
        expires  360d;

        rewrite  ^/admin/(.+)$ /backend/web/$1 break;
        rewrite  ^/admin/(.+)/(.+)$ /backend/web/$1/$2 break;
        try_files  $uri =404;
    }

    location ~ \.php$ {
        include  fastcgi_params;
        # check your /etc/php5/fpm/pool.d/www.conf to see if PHP-FPM is listening on a socket or port
        fastcgi_pass  unix:/var/run/php5-fpm.sock; ## listen for socket
        #fastcgi_pass  127.0.0.1:9000; ## listen for port
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        try_files  $uri =404;
    }
    #error_page  404 /404.html;

    location = /requirements.php {
        deny all;
    }

    location ~ \.(ht|svn|git) {
        deny all;
    }
}
```
