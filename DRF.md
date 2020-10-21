#Django Restful Framework知识点记录
*文档可参考：*[Django Restful Framework文档](https://www.django-rest-framework.org/api-guide/viewsets/)
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

####2.访问权限设置
```python
# 方式一
class EquipmentInfoViewset(viewsets.ModelViewSet):
    queryset = Equipment.objects.all()
    serializer_class = EquipmentSerializer
    permission_classes_by_action = {'default': [],
                                    'create': [permissions.IsAdminUser],
                                    'retrieve': [permissions.IsAuthenticated],
                                    'update': [permissions.IsAdminUser],
                                    'partial_update': [permissions.IsAdminUser],
                                    'destroy': [permissions.IsAdminUser],
                                    'list': [permissions.IsAuthenticated]
                                    }

    # 重写get_permissions
    def get_permissions(self):
        """
            A viewset that provides default `create()`, `retrieve()`, `update()`,
            `partial_update()`, `destroy()` and `list()` actions.
            """
        try:
            # return permission_classes depending on `action`
            return [permission() for permission in self.permission_classes_by_action[self.action]]
        except KeyError:
            # 没用明确权限的话使用默认权限
            # action is not set return default permission_classes
            return [permission() for permission in self.permission_classes_by_action['default']]

# 方式二
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdmin]
    return [permission() for permission in permission_classes]
```

####3.自定义视图集动作(参考https://www.django-rest-framework.org/api-guide/viewsets/)
```python
#urls.py 视图集路由设置
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet, basename='user')
urlpatterns = router.urls
```
```python
# 用于整个集合
@action(detail=False , methods=['get'])
def recent_users(self, request):
    ...
# 用于当个对象
@action(detail=True, methods=['post'], permission_classes=[IsAdminOrIsSelf])
def set_password(self, request, pk=None):
    ...
```
使用@action装饰器，以上两个操作的路由为/users/recent_users/和/users/{pk}/set_password/