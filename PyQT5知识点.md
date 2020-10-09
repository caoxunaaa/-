#PyQt5知识点记录
##一.控件的操作
####1.QcomboBox添加搜索
```python
list = [111, 222, 333]
self.comboBox.addItems(list)
self.completer = QCompleter(list)
self.comboBox.setCompleter(self.completer)
```
在下拉框中进行输入部分字符可以得到相关所有选项
####2.自定义信号QtCore.pyqtSignal
```python
class Main(object):
    def __init__(self):
        self.thread = Thread(func=do_something)
        self.thread.sinOut_sql_result.connect(self.func)
        self.thread.start()
    
    def func(self, sql_result_dict):
        pass

class Thread(QtCore.QThread):
    sinOut_sql_result = QtCore.pyqtSignal(dict)
    
    def __init__(self, **kwargs):
        self.kwargs = kwargs
        super().__init__()

    def run(self):
        sql_result_dict = dict()
        self.sinOut_sql_result.emit(sql_result_dict)
```
上面的例子是用在多线程中，自定义信号可以增加参数，通过emit方法进行触发
####3.定时器QtCore.QTimer()
```python
self.timer = QtCore.QTimer()
self.timer.timeout.connect(func)
self.timer.start(1000) #定时1s
```
定时1s，超时触发func方法， 可以使用self.timer.stop()停止定时器
##二.多线程
####1.互斥锁
```python
LOCK = QtCore.QMutex()

class Thread1(QtCore.QThread):
    def __init__(self, **kwargs):
        self.kwargs = kwargs
        super().__init__()

    def run(self):
        LOCK.lock()
        do_something()
        LOCK.unlock()

class Thread2(QtCore.QThread):
    def __init__(self, **kwargs):
        self.kwargs = kwargs
        super().__init__()

    def run(self):
        LOCK.lock()
        do_something()
        LOCK.unlock()
```
QMutex的目的是保护一个对象、数据结构或者代码段，所以同一时间只有一个线程可以访问它。如果QMutex被一个线程加了锁，另一个线程就会一直阻塞直到锁被释放。
