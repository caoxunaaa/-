#Docker+Django+DRF+uwsgi+Vue+nginx项目记录
##一.项目环境
- PC：Win7 64位
- 服务器：阿里云ubuntu16.04.7 公网IP: 47.115.52.186
- Django: 3.0.10
- djangorestframework: 3.11.1
- Vue: 2.9.6
- 部署：uwsgi nginx
- 工具： WinSCP
- 前后端工程项目：[github.com](https://github.com/caoxunaaa/SuperxonWork/tree/master/SuperxonWebSite)
##二.首先是Docker上运行django+uwsgi(参考[Docker 部署 Django+Uwsgi+Nginx+Vue](https://testerhome.com/topics/25440))
###1.通过WinSCP把PC上的Django工程转移到阿里云上
```
root@iZwz9bxltrtxtsix34kqlrZ:~/DockerProject/api_automation_test# ls
db.sqlite3  Dockerfile  EquipmentManagement  manage.py  mysql.txt  requirements.txt  start.sh  SuperxonWebSite  user  uwsgi.ini
```

####- Dockerfile:
```dockerfile
# 建立 python3.7 环境
FROM python:3.7
 # 设置 python 环境变量
 #ENV PYTHONUNBUFFERED 1
 # 在容器内/var/www/html/下创建 mysite1文件夹
RUN mkdir -p /var/www/html/api_automation_test
# 设置容器内工作目录
WORKDIR /var/www/html/api_automation_test
 # 将当前目录文件加入到容器工作目录中（. 表示当前宿主机目录）
ADD . /var/www/html/api_automation_test
#下载第三方包
RUN pip install -i https://pypi.doubanio.com/simple uwsgi
RUN pip install -i https://pypi.doubanio.com/simple/ -r requirements.txt
# Windows环境下编写的start.sh每行命令结尾有多余的\r字符，需移除。
RUN sed -i 's/\r//' ./start.sh
# 设置start.sh文件可执行权限
RUN chmod +x ./start.sh
```
####- uwsgi.ini:
```editorconfig
[uwsgi]
#api_automation_test是我代码的路径
project = SuperxonWebSite
uid = root
gid = root
chdir = /var/www/html/api_automation_test
module = SuperxonWebSite.wsgi:application
master = True
processes = 2
socket = 0.0.0.0:8001 #这里直接使用uwsgi做web服务器，使用http。如果使用nginx，需要使用socket沟通。
buffer-size = 65536
pidfile = /tmp/SuperxonWebSite-master.pid
vacuum = True
max-requests = 5000
daemonize = uwsgi.log
# 解决APSchedler任务不能执行
threads=4
enable-threads = true
preload = true
lazy-apps = true
#设置一个请求的超时时间(秒)，如果一个请求超过了这个时间，则请求被丢弃
harakiri = 60
#当一个请求被harakiri杀掉会，会输出一条日志
harakiri-verbose = true
```
####- start.sh:
```shell script
#!/bin/bash
# 从第一行到最后一行分别表示：
# 1. 生成数据库迁移文件
# 2. 根据数据库迁移文件来修改数据库
# 3. 用 uwsgi启动 django 服务, 不再使用python manage.py runserver
#python manage.py collectstatic --noinput&&
python manage.py makemigrations&&
python manage.py migrate&&
uwsgi --enable-threads /var/www/html/api_automation_test/uwsgi.ini
```
####- 生成requipment.txt:
```
pip freeze > requirements.txt
```
###2.开始部署docker--后端
- ####创建Docker镜像
```
docker build -t django_web .
```

- ####查看创建好的Docker镜像
```
docker images
```
- ###启动容器对应uwsgi.ini端口
```
docker run -it --name MyDjango -p 8001:8001 -d django_web
```
- ####查看启动的容器
```
docker ps
```
- ####进入容器执行start.sh
```
docker exec -it MyDjango /bin/bash
sh start.sh
```
- ####展示结果
能在网页http://47.115.52.186:8001/handle/中看到相应的drf api接口

###3.通过WinSCP把PC上的前端vue工程build生成的dist文件夹上传到阿里云上
```shell script
root@iZwz9bxltrtxtsix34kqlrZ:~/DockerProject/front_test# ls
dist  Dockerfile  nginx.conf
```
####- Dockerfile:
```dockerfile
# nginx镜像
FROM nginx:latest
# 删除原有配置文件，创建静态资源文件夹和ssl证书保存文件夹
RUN rm /etc/nginx/conf.d/default.conf \
&& mkdir -p /etc/nginx/html/static \
&& mkdir -p /etc/nginx/html/media \
&& mkdir -p /etc/nginx/ssl \
&& mkdir -p /etc/nginx/html
COPY dist/  /etc/nginx/html/
# 添加配置文件
COPY nginx.conf /etc/nginx/nginx.conf
# 构建镜像时执行的命令
RUN chmod -R 777 /etc/nginx/html/ 
RUN echo 'echo init ok!'
```
####- nginx.conf:
```editorconfig
#user  nobody;
worker_processes  1;

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

        location /handle/ {
            # uwsgi使用http
            # proxy_pass http://47.115.52.186:8001;

            # uwsgi使用socket
            uwsgi_pass 47.115.52.186:8001;
            include uwsgi_params;
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
        #}
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
主要修改就是加入下面语句,用于反向代理，匹配http://47.115.52.186:8001/handle/xxxxx获得相应的数据，在项目中主要作用axios异步通信
```editorconfig
location /handle/ {
            # uwsgi使用http
            # proxy_pass http://47.115.52.186:8001;

            # uwsgi使用socket
            uwsgi_pass 47.115.52.186:8001;
            include uwsgi_params;
        }
```

###3.开始部署docker--前端
- ####创建Docker镜像
```
docker build -t vue_web .
```
- ###启动容器对应uwsgi.ini端口
```
docker run -it --name Myfront -p 80:80 -d vue_web
```
- ####查看启动的容器
```
docker ps
```
- ####进入容器执行start.sh
```
docker exec -it MyDjango /bin/bash
sh start.sh
```
- ####展示结果
能在外网浏览器中输入http://47.115.52.186中看到相应的网页