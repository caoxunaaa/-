#Django知识点记录
*文档可参考：*[Django官方文档](https://docs.djangoproject.com/en/3.1/)
##一.使用Django的准备工作和配置
####1.安装模块
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
- 设置使用mysql数据库
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
##二.项目中点滴知识记录
####1.Django Q F查询
- Q对象（django.db.models.Q）可以对关键字参数进行封装,组合使用&, |, ~操作符
```python
from django.db.models import Q
from .models import Equipment
queryset = Equipment.objects.filter(Q(sn=device) | ~Q(id=7))
queryset = Equipment.objects.filter(Q(sn=device) | Q(internal_coding=device)).extra(select={'device__name': 'name', 'device__sn': 'sn', 'device__id': 'id', 'device__assets': 'assets', 'device__internal_coding': 'internal_coding'}).values('device__name', 'device__sn', 'device__id', 'device__assets', 'device__internal_coding')
```
####2.Django extra得到自己想要的字段
- extra可以指定一个或者多个参数select, where
```python
Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})
Entry.objects.extra(where=['id IN (3, 4, 5, 20)'])

```
####3.Django执行原生sql语句
```python
from django.db import connection

with connection.cursor() as cursor:
    sql = '''select A.device, A.sn, A.internal_coding, A.assets, A.maintenance_item, MAX(A.maintenance_time) from EquipmentManagement_devicemaintenancerecord A group by A.device, A.maintenance_item'''
    cursor.execute(sql)
    row = cursor.fetchall()
```
####3.Django将查询数据导出csv文件
```python
file_path_abs = 'example.csv'
with open(file_path_abs, "w", newline='') as csv_file:
    writer = csv.writer(csv_file)
    # 表头
    writer.writerow(['设备', 'SN', '设备编号', '固资号', '保养项目', '保养时间'])
    with connection.cursor() as cursor:
        sql = '''select A.device, A.sn, A.internal_coding, A.assets, A.maintenance_item, MAX(A.maintenance_time) from EquipmentManagement_devicemaintenancerecord A group by A.device, A.maintenance_item'''
        cursor.execute(sql)
        row = cursor.fetchall()
        # 向CSV添加数据
        writer.writerows(row)
```
####4.数据库迁移（sqlite3 -> mysql）
- 从原来的数据库导出数据json文件
```
python manage.py dumpdata > superxon.json
python manage.py dumpdata --database default > superxon.json # 指定数据库
```
- 将json文件导出新数据库
```
python manage.py loaddata superxon.json
python manage.py loaddata --database slave superxon.json # 指定数据库
python manage.py dumpdata --exclude=contenttypes --exclude=auth.Permission default > superxon.json # 加入报错重复导入，可以执行exclude这个参数,忽略者两个表
```


