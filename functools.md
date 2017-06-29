update_wrapper
===

```
# 要进行赋值的属性，qualname是函数或者方法的合法名(类似唯一名的意思)，__annotations__是包含参数注解的字典，key是参数，value是注解
WRAPPER_ASSIGNMENTS = ('__module__', '__name__', '__qualname__', '__doc__',
                       '__annotations__')
# 要进行更新的属性
WRAPPER_UPDATES = ('__dict__',)
def update_wrapper(wrapper, wrapped,
                   assigned = WRAPPER_ASSIGNMENTS,
                   updated = WRAPPER_UPDATES):
    '''更新新的wrapper函数 让其 和之前的函数基本信息一致
    wrapper是要更新信息的函数
    wrapped是原始函数
    assigned是一个元组，里面包含了所有直接赋值的属性
    updated是一个元组，里面是要从原始函数到现在函数要更新的属性
    '''
    for attr in assigned:
        try:
            value = getattr(wrapped, attr)
        except AttributeError:
            pass
        else:
            setattr(wrapper, attr, value)
    for attr in updated:
        getattr(wrapper, attr).update(getattr(wrapped, attr, {}))
    #
    wrapper.__wrapped__ = wrapped
    # 返回新的wrapper函数
    return wrapper
```

wraps
===

```
def wraps(wrapped,
          assigned = WRAPPER_ASSIGNMENTS,
          updated = WRAPPER_UPDATES):
    return partial(update_wrapper, wrapped=wrapped,
                  assigned=assigned, updated=updated)
```
