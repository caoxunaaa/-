#Python常用基础知识点记录
####1.检查文件不存在时便创建
```python
import os
if not os.path.exists(path + '\\filename'):
    os.makedirs(path + '\\filename')
```
####2. Python中*args, **kwargs
```python
def foo(*args, **kwargs):
    print('args = ', args)
    print('kwargs = ', kwargs)
    print('---------------------------------------')
if __name__ == '__main__':
    foo(1,2,3,4)
    foo(a=1,b=2,c=3)
    foo(1,2,3,4, a=1,b=2,c=3)
    foo('a', 1, None, a=1, b='2', c=3)
```
输出：
```
args =  (1, 2, 3, 4) 
kwargs =  {} 
--------------------------------------- 
args =  () 
kwargs =  {'a': 1, 'c': 3, 'b': 2} 
--------------------------------------- 
args =  (1, 2, 3, 4) 
kwargs =  {'a': 1, 'c': 3, 'b': 2} 
--------------------------------------- 
args =  ('a', 1, None) 
kwargs =  {'a': 1, 'c': 3, 'b': '2'} 
---------------------------------------
```
args 为一个元组参数，kwargs为一个字典参数
####3.datetime部分操作
```python
import datetime
now = datetime.datetime.now()
zero_today = now - datetime.timedelta(hours=now.hour, minutes=now.minute, seconds=now.second,
                                      microseconds=now.microsecond) # 时间运算
# last_today = zero_today + datetime.timedelta(hours=23, minutes=59, seconds=59)
zero_today = zero_today.strftime('%Y-%m-%d %H:%M:%S') # 字符串化
now = now.strftime('%Y-%m-%d %H:%M:%S') 
now_date = datetime.strptime(now, '%Y-%m-%d %H:%M:%S')
```
####4.map lambda 操作
map(__func, __iter)
```python
li = [1, 2, 3]
print(list(map(lambda x:x*2, li)))
```
输出
```
[2, 4, 6]
```
