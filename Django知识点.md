#Django知识点记录
*文档可参考：*[Django官方文档](https://docs.djangoproject.com/en/3.1/)
##一.使用Django的准备工作和配置
####1.需要安装模块
```
pip install Django
pip install django-cors-headers //解决浏览器跨域的问题
pip install mysqlclient //MySql模块
pip install PyMySQL //MySql模块
pip install Pillow //图片模块
```
####2.基本命令
```
django-admin startproject [projectname] # 创建一个名为[projectname]的django工程
# 进入到工程目录中
python manage.py startapp [appname] # 创建一个名为[appname]的django应用
python manage.py makemigrations # 创建数据库迁移文件
python manage.py migrate # 进行数据库迁移
python manage.py createsuperuser # 创建超级用户
python manage.py runserver # 启动web服务,默认127.0.0.1：8000,可以在后面加上指定ip：port
```
####3.数据库设置
```python
'''默认数据库sqlite3
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
'''

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'tree_hole_db',
        'USER': 'caoxun',
        'PASSWORD': os.environ['DATABASE_PASSWORD'],
        'HOST': '47.115.52.186',
        'PORT': '3306',
    }
}
```
设置使用mysql数据库
####4.浏览器跨域问题解决
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'corsheaders', # 加入
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware', # 加入
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

CORS_ORIGIN_ALLOW_ALL = True # 加入
```