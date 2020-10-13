#Django Restful Framework知识点记录
*文档可参考：*[Django Restful Framework中文文档](https://q1mi.github.io/Django-REST-framework-documentation/api-guide/serializers_zh/)
##一.使用Django Restful Framework的准备工作和配置
####1.需要安装模块
```
pip install djangorestframework
```

```python
# settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'EquipmentManagement',
    'rest_framework', #加入
    'corsheaders',
]

#urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include(router.urls)),
    path('api-auth', include('rest_framework.urls')),   # 加入
    path(r'index/', TemplateView.as_view(template_name="index.html")),
]
```
##二.DRF项目中常用知识
####1.serializerMethodField的作用和具体使用
```python
#models.py
class Equipment(models.Model):
    category = models.ForeignKey(EquipmentType, verbose_name='设备类型', on_delete=models.CASCADE)
    name = models.CharField(verbose_name='设备名称', max_length=100)
    sort = models.CharField(max_length=100, verbose_name='设备型号')
    owner = models.ForeignKey(Profile, verbose_name='归属者', on_delete=models.CASCADE)
    sn = models.CharField(max_length=30, unique=True)
    assets = models.CharField(max_length=30, unique=True)
    create_time = models.DateTimeField(verbose_name='入库时间', auto_now_add=True)

    def __str__(self):
        return self.name + '(' + self.sn + ')'

    class Meta:
        ordering = ['-create_time']

#serializers.py
class EquipmentSerializer(serializers.ModelSerializer):
    category_display = serializers.SerializerMethodField(read_only=True) # 设备类型名称
    owner_display = serializers.SerializerMethodField(read_only=True) # 归属者昵称
    category = serializers.PrimaryKeyRelatedField(queryset=EquipmentType.objects.exclude(parent_type=None)) 
    status = serializers.SerializerMethodField(read_only=True) # 设备状态

    class Meta:
        model = Equipment
        fields = '__all__'

    def get_category_display(self, obj):
        parent_type = EquipmentType.objects.get(name=obj.category.parent_type)
        return {'id': obj.category.id, 'category': obj.category.name, 'parent_category': parent_type.name}

    def get_owner_display(self, obj):
        return obj.owner.nickname

    def get_status(self, obj):
        if DeviceRepairManage.objects.filter(device=obj.id).exists():
            sn_queryset = DeviceRepairManage.objects.filter(device=obj.id).order_by('-record_time')
            print(sn_queryset)
            return sn_queryset.first().status
        else:
            return 0
```
- 作用：使用ModelSerializer序列化器进行序列化返回时，想得到其它模型中的字段数据，或者说根据自身模型中的数据返回别的数据时，就可以使用SerializerMethodField.
- 以本例进行解释：category_display， owner_display， status都是Equipment模型中没有的字段，通过定义serializers.SerializerMethodField，并写上相应的获取方法：get_category_display(self, obj), get_owner_display(self, obj), get_status(self, obj)返回想得到的值.
在方法中func(self, obj), obj 为 Equipment的对象实例.
- 前端进行查询的时候，后端返回的时候就会加入这些额外信息