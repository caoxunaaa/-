#Docker+Django+DRF+uwsgi+Vue+nginx+celery项目记录
##一.项目环境
- PC：Win7 64位
- 服务器：阿里云ubuntu16.04.7 公网IP: 47.115.52.186
- Django: 3.0.10
- djangorestframework: 3.11.1
- celery: 4.4.7
- Vue: 2.9.6
- redis: 3.5.3
- mysql
- 部署：uwsgi nginx
- 工具： WinSCP
- 前后端工程项目：[SuperxonWebSite](https://github.com/caoxunaaa/SuperxonWork/tree/master/SuperxonWebSite)
##二.Docker部署前后端(参考[Docker 部署 Django+Uwsgi+Nginx+Vue](https://testerhome.com/topics/25440))
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
```
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
```
location /handle/ {
            # uwsgi使用http
            # proxy_pass http://47.115.52.186:8001;

            # uwsgi使用socket
            uwsgi_pass 47.115.52.186:8001;
            include uwsgi_params;
        }
```

-注：还有个注意的点就是前端vue代码中跨域访问设置config/index.js
```
proxyTable: {
  '/api': {  //使用"/api"来代替"http://f.apiplus.c"
    target: 'http://47.115.52.186:8001/', //源地址
    changeOrigin: true, //改变源
    pathRewrite: {
      '^/api': '' //路径重写
    }
  }
},
```
###4.开始部署docker--前端
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
##三.Celery异步任务（项目中需要定时处理任务）
###1.linux安装redis服务器)
```
sudo apt-get install redis-server
```
修改redis配置文件 /etc/redis/redis.conf
```
#bind 127.0.0.1  这里是只能本地使用
bind 0.0.0.0  #这样是允许任意IP访问
```
启动redis
```
redis-server /etc/redis/redis.conf
```
###2.安装celery, redis
```
pip install django-redis
pip install celery
```
###3.配置相应的文件(工程项目为SuperxonWebSite,其下的根为SuperxonWebSite，其它应用为EquipmentManagement, user)
####- settings.py
```python
redis_url = 'redis://127.0.0.1:6379/0'
# 这两项必须设置，否则不能正常启动celery beat
CELERY_ENABLE_UTC = True
CELERY_TIMEZONE = TIME_ZONE
# 任务队列配置
CELERY_BROKER_URL = 'redis://localhost'   # Broker配置，使用Redis作为消息中间件

CELERY_RESULT_BACKEND = 'redis://localhost'   # BACKEND配置，这里使用redis

CELERY_RESULT_SERIALIZER = 'json'   # 结果序列化方案
```
####- 在settings.py同级目录下创建celery.py
```python
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
#SuperxonWebSite是工程应用目录
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'SuperxonWebSite.settings')
redis_url = 'redis://localhost'
app = Celery('SuperxonWebSite', broker=redis_url, backend=redis_url)

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()

#配置定时任务
app.conf.beat_schedule = {
    'add-every-30-seconds': {
        'task': 'EquipmentManagement.tasks.test', #任务函数
        'schedule': 10.0,  #每10秒钟触发一次
        'args': (3, )   #任务函数参数
    },
}
```
####- 在APP应用目录EquipmentManagement下新建tasks.py
```python
from __future__ import absolute_import, unicode_literals
from celery import shared_task


@app.task
def test(arg):
    #这是上面说的定时任务函数
    print('定时任务')


@app.task
def add(x, y):
    return x + y


@app.task
def mul(x, y):
    return x * y
```
####- 在APP应用目录EquipmentManagement中views.py视图中异步调用任务
```
from EquipmentManagement import tasks

@action(detail=False, methods=['get'])
def device_list(self, request):
    res = tasks.mul.delay(1, 3)
    print(res.task_id)
    device = request.GET.get('device', None)
    if device:
        device = Equipment.objects.filter(Q(sn=device) | Q(assets=device) | Q(internal_coding=device))
        serializer = EquipmentSerializer(device, many=True)
        return Response(serializer.data)
    else:
        return Response({'status': 'fail'}, status=status.HTTP_404_NOT_FOUND)
```
res = tasks.mul.delay(1, 3)     #delay方法中添加相应的参数
print(res.task_id)              #获取任务id，在redis中保存为celery-task-meta-[id],[id]类似fcb4328d-fc31-4ae5-9fe6-457f5529f870
###4.celery启动命令
####- 在工程目录下使用命令celery -A SuperxonWebSite beat -l info ,这是开启定时发送任务到celery任务队列中
得到打印信息
```
celery beat v4.4.7 (cliffs) is starting.
__    -    ... __   -        _
LocalTime -> 2020-10-29 15:53:10
Configuration ->
    . broker -> redis://localhost:6379//
    . loader -> celery.loaders.app.AppLoader
    . scheduler -> celery.beat.PersistentScheduler
    . db -> celerybeat-schedule
    . logfile -> [stderr]@%INFO
    . maxinterval -> 5.00 minutes (300s)
[2020-10-29 15:53:10,070: INFO/MainProcess] beat: Starting...
[2020-10-29 15:53:10,090: INFO/MainProcess] Scheduler: Sending due task add-every-30-seconds (EquipmentManagement.tasks.test)
[2020-10-29 15:53:40,095: INFO/MainProcess] Scheduler: Sending due task add-every-30-seconds (EquipmentManagement.tasks.test)
```

####- 在工程目录下使用命令celery -A SuperxonWebSite worker -l info
得到打印信息
```
[tasks]
  . EquipmentManagement.tasks.add
  . EquipmentManagement.tasks.mul
  . EquipmentManagement.tasks.test

[2020-10-29 15:45:20,387: INFO/MainProcess] Connected to redis://localhost:6379//
[2020-10-29 15:45:20,403: INFO/MainProcess] mingle: searching for neighbors
[2020-10-29 15:45:21,435: INFO/MainProcess] mingle: all alone
[2020-10-29 15:45:21,446: WARNING/MainProcess] /usr/local/lib/python3.7/site-packages/celery/fixups/django.py:206: UserWarning: Using settings.DEBUG leads to a memory
            leak, never use this setting in production environments!
  leak, never use this setting in production environments!''')
[2020-10-29 15:45:21,447: INFO/MainProcess] celery@dc6b86273c3a ready.
[2020-10-29 15:45:35,819: INFO/MainProcess] Received task: EquipmentManagement.tasks.mul[ebf93a95-574d-4270-9c35-5a1f8ae981ca]  
[2020-10-29 15:45:35,832: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.mul[ebf93a95-574d-4270-9c35-5a1f8ae981ca] succeeded in 0.01080656610429287s: 3
[2020-10-29 15:53:10,104: INFO/MainProcess] Received task: EquipmentManagement.tasks.test[09f713f5-b80f-411d-90c7-294e5f038f89]  
[2020-10-29 15:53:10,106: WARNING/ForkPoolWorker-1] 定时任务
[2020-10-29 15:53:10,108: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.test[09f713f5-b80f-411d-90c7-294e5f038f89] succeeded in 0.0019641800317913294s: None
[2020-10-29 15:53:40,098: INFO/MainProcess] Received task: EquipmentManagement.tasks.test[9e2fc38c-3fa7-4dad-b5bf-68e2c46b3008]  
[2020-10-29 15:53:40,100: WARNING/ForkPoolWorker-1] 定时任务
[2020-10-29 15:53:40,102: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.test[9e2fc38c-3fa7-4dad-b5bf-68e2c46b3008] succeeded in 0.002117861993610859s: None
[2020-10-29 15:54:10,119: INFO/MainProcess] Received task: EquipmentManagement.tasks.test[3ef85925-931d-43a8-87ea-63986b7f9ccf]  
[2020-10-29 15:54:10,125: WARNING/ForkPoolWorker-1] 定时任务
[2020-10-29 15:54:10,127: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.test[3ef85925-931d-43a8-87ea-63986b7f9ccf] succeeded in 0.0027211259584873915s: None
[2020-10-29 15:54:40,129: INFO/MainProcess] Received task: EquipmentManagement.tasks.test[c82057e7-99ca-4017-ba44-4b27a743bb4c]  
[2020-10-29 15:54:40,131: WARNING/ForkPoolWorker-1] 定时任务
[2020-10-29 15:54:40,133: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.test[c82057e7-99ca-4017-ba44-4b27a743bb4c] succeeded in 0.0024393098428845406s: None
[2020-10-29 15:55:04,548: INFO/MainProcess] Received task: EquipmentManagement.tasks.mul[4bc76016-c250-46bc-affc-1b9609f9a252]  
[2020-10-29 15:55:04,551: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.mul[4bc76016-c250-46bc-affc-1b9609f9a252] succeeded in 0.0009407009929418564s: 3
[2020-10-29 15:55:05,340: INFO/MainProcess] Received task: EquipmentManagement.tasks.mul[40cfac38-6ca6-4d15-b9a5-471d6a217ed3]  
[2020-10-29 15:55:05,343: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.mul[40cfac38-6ca6-4d15-b9a5-471d6a217ed3] succeeded in 0.0009097040165215731s: 3
[2020-10-29 15:55:10,145: INFO/MainProcess] Received task: EquipmentManagement.tasks.test[c4753c7f-6aac-41bf-a67c-549774542810]  
[2020-10-29 15:55:10,147: WARNING/ForkPoolWorker-1] 定时任务
[2020-10-29 15:55:10,149: INFO/ForkPoolWorker-1] Task EquipmentManagement.tasks.test[c4753c7f-6aac-41bf-a67c-549774542810] succeeded in 0.002141116186976433s: None
```
*注：这里有个坑，在windows下,打开网页执行到这个任务时，得到任务状态一直为PENDING等待中，这里celery默认的并发模式为prefork,可能是window系统下prefork会堵塞*
*甚至在Windows下都不能定时触发任务*
```
-P POOL_CLS, --pool=POOL_CLS
                        Pool implementation: prefork (default), eventlet,
                        gevent, solo or threads.
```
windows下更改启动命令
```
pip install eventlet #安装eventlet
celery -A SuperxonWebSite worker --pool=eventlet -l info
```
##四.DRF JWT验证
###1.JWT原理：[参考]（https://www.cnblogs.com/fiona-zhong/p/9951054.html）
按我的理解简单描述就是，前端给后端发送用户密码，通过JWT内部算法得到一个Token验证码并返回给前端保存，此后前端在请求中都需要
将Token放到header中的Authorization发给后端，后端验证JWT有效性，通过之后再进行相应的操作。
###2.安装DRF JWT
```
pip install djangorestframework-jwt
```
###3.后端配置DRF JWT
`settings.py`
```
REST_FRAMEWORK = {
    #默认全局配置，需要验证Token
    'DEFAULT_AUTHENTICATION_CLASSES': (
        # 引入JWT认证机制，当客户端将jwt token传递给服务器之后
        # 此认证机制会自动校验jwt token的有效性，无效会直接返回401(未认证错误)
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ),
}

# JWT扩展配置
JWT_AUTH = {
    # 设置生成jwt token的有效时间
    'JWT_EXPIRATION_DELTA': datetime.timedelta(days=1),
}
```
局部不需要Token认证的视图类，比如登录、注册和登出等等，可以在视图类下加入authentication_classes = []，例如
```python
class UserViewset(viewsets.ModelViewSet):
    authentication_classes = []
```
`urls.py`
```python
from rest_framework_jwt.views import obtain_jwt_token
urlpatterns = [
    #前端登录时访问此路由： POST （username, password）,返回给前端一个 Token。
    path('authorizations/', obtain_jwt_token), 
]
```
###4.前端Vue对JWT的保存和使用
`UserLogin.vue`
```
that.$axios({
    method: 'post',
    url: '/authorizations/',
    data: {
      username: that.ruleForm['username'],
      password: that.ruleForm['password']
    }
}).then(function (response) {
    const res = response.data
    console.log(res)
    //将返回的Token保存在localStorage中
    localStorage.setItem('Token', res['token'])
})
```
`main.js`
```js
axios.interceptors.request.use((config) => {
  config.headers['X-Requested-With'] = 'XMLHttpRequest'
  let regex = /.*csrftoken=([^;.]*).*$/ // 用于从cookie中匹配 csrftoken值
  config.headers['X-CSRFToken'] = document.cookie.match(regex) === null ? null : document.cookie.match(regex)[1]
  //在axios异步请求中的headers中的Authorization加入'JWT [Token]'
  config.headers.Authorization = 'JWT ' + localStorage.getItem('Token')
  return config
})
```
注：在登出或者注销的时候，应该删除localStorage中的Token.

