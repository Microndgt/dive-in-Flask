cached_property
---

其是继承property的一个类，一个property即是一个描述符，所以`cached_property`也是一个描述符，实现了`__set__`和`__get__`方法，具体源码如下：

```
class cached_property(property):
    '''一个将函数转换成惰性属性的装饰器，函数第一次被调用去获取数据，然后下一次获取属性的时候就可以直接获取结果。这个类是用__dict__来使得属性工作的。
    '''
    def __init__(self, func, name=None, doc=None):
        self.__name__ = name or func.__name__
        self.__module__ = func.__module__
        self.__doc__ = doc or func.__doc__
        self.func = func

    def __set__(self, obj, value):
        # 每一个函数缓存一个值，存储在对象的__dict__属性中
        obj.__dict__[self.__name__] = value

    def __get__(self, obj, type=None):
        # obj对象是类的实例对象
        if obj is None:
            return self
        value = obj.__dict__.get(self.__name__, _missing)
        if value is _missing:
            value = self.func(obj)
             obj.__dict__[self.__name__] = value
        return value
```

environ_property
---
