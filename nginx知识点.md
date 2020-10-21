#nginx部署知识点记录
##一.使用前准备工作
####1.环境
- 阿里云ubuntu16.0.4 IP: //47.115.52.186
- docker里面执行django后端服务：在Docker知识点中讲解
- 前后端工程项目：[github.com](https://github.com/caoxunaaa/SuperxonWork/tree/master/SuperxonWebSite)

####2.安装方法
*可参考：*[Ubuntu安装nginx](https://blog.csdn.net/dong_ly/article/details/99686722/)

也可以使用sudo apt-get install nginx进行安装

##二.nginx部署前端页面（Vue）
####1.前端文件放置
Vue 使用nmp run build 命令对前端页面代码进行打包得到dist文件夹：里面有static文件夹和index.html文件

```把dist中的文件放在nginx/html中，如下所示
root@iZwz9bxltrtxtsix34kqlrZ:/usr/local/nginx# tree html
html
├── 50x.html # 原本就有
├── index_bak.html # 原本的index.thml重命名
├── index.html
└── static
    ├── css
    │   ├── app.338363c5bbe6c76e31f6b37288e5c74a.css
    │   └── app.338363c5bbe6c76e31f6b37288e5c74a.css.map
    ├── fonts
    │   ├── element-icons.535877f.woff
    │   └── element-icons.732389d.ttf
    └── js
        ├── app.e11bb3185721e279e79c.js
        ├── app.e11bb3185721e279e79c.js.map
        ├── manifest.2ae2e69a05c33dfc65f8.js
        ├── manifest.2ae2e69a05c33dfc65f8.js.map
        ├── vendor.e4b51d1dc556c8f9ded2.js
        └── vendor.e4b51d1dc556c8f9ded2.js.map
```
####2.修改nginx配置文件
```主要是对conf/nginx.conf进行修改以达到我们的使用目的
root@iZwz9bxltrtxtsix34kqlrZ:/usr/local/nginx# tree conf
conf
├── fastcgi.conf
├── fastcgi.conf.default
├── fastcgi_params
├── fastcgi_params.default
├── koi-utf
├── koi-win
├── mime.types
├── mime.types.default
├── nginx.conf # 就这个文件
├── nginx.conf.default
├── scgi_params
├── scgi_params.default
├── uwsgi_params
├── uwsgi_params.default
└── win-utf
```
```shell script
vim conf/nginx.conf
```
```
#user  nobody;
worker_processes 1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }
        
        #!!加入django反向代理，使前端axios能够通过匹配http://47.115.52.186:8001/handle/xxxxx获得相应的数据
        location /handle/ {
           proxy_pass http://0.0.0.0:8001;
        }


        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
        
```
主要增加下面语句
```editorconfig
#!!加入django反向代理，使前端axios能够通过匹配http://47.115.52.186:8001/handle/xxxxx获得相应的数据
location /handle/ {
   proxy_pass http://47.115.52.186:8001;
}
```
####3.nginx启动停止等等命令
- 查询nginx进程 
```
ps -ef | grep nginx
```
- 启动nginx进程 
```
root@iZwz9bxltrtxtsix34kqlrZ:/usr/local/nginx# ./sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
- 重启nginx进程 
```
service nginx -s  reload
```
- 停止nginx进程 
```
pkill -9 nginx
```

####4.启动之后，就能在外网浏览器中通过47.115.52.186/得到相应的前端
可能也会得到403错误响应，可能是由于index.html和static文件的权限不够，可以都给到777再进行页面的刷新，完成。





