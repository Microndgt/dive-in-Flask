flask中有应用上下文和请求上下文，关于请求上下文在上一节Request已经介绍过了。本节主要关注上下文栈是如何运作的，以及应用上下文的相关东西。

> 每一段程序都有很多外部变量。只有像Add这种简单的函数才是没有外部变量的。一旦你的一段程序有了外部变量，这段程序就不完整，不能独立运行。你为了使他们运行，就要给所有的外部变量一个一个写一些值进去。这些值的集合就叫上下文。 – vzch

在 flask 中，视图函数需要知道它执行情况的请求信息（请求的 url，参数，方法等）以及应用信息（应用中初始化的数据库以及一些参数等），才能够正确运行。

最直观地做法是把这些信息封装成一个对象，作为参数传递给视图函数。但是这样的话，所有的视图函数都需要添加对应的参数，即使该函数内部并没有使用到它。

那么另外一种思路就是类似全局变量的东西，需要的时候使用`from flask import request`获取，但是这个全局变量必须是线程独立的，也就是每个请求所在的线程得到的请求对象必须是不一样的，不会互相干扰。

Werkzeug中实现了类似于本地线程`threading.local`的类Local，通过这个Local实现了LocalStack和LocalProxy，而上下文栈则是使用LocalStack来建立起来的。

Local
===

多线程中有个非常类似的概念 threading.local，可以实现多线程访问某个变量的时候只看到自己的数据。这个对象有一个字典，保存了线程 id 对应的数据，读取该对象的时候，它动态地查询当前线程 id 对应的数据。下面的Local实现了一个类似线程独立的数据访问，代码如下：

```
try:
  from greenlet import getcurrent as get_ident
except ImportError:
  try:
    from thread import get_ident
  except ImportError:
    from _thread import get_ident
class Local(object):
  __slots__ = ('__storage__', '__ident_func__')

  def __init__(self):
    # 存储的key是每一个线程的id
    object.__setattr__(self, '__storage__', {})
    # 这个属性是一个函数对象
    object.__setattr__(self, '__ident_func__', get_ident)

  def __iter__(self):
    return iter(self.__storage__.items())

  def __call__(self, proxy):
    """Create a proxy for a name."""
    return LocalProxy(self, proxy)

  def __release_local__(self):
    # 将当前线程id代表的key-value pop掉
    self.__storage__.pop(self.__ident_func__(), None)

  def __getattr__(self, name):
    # 获取当前线程的数据
    try:
      return self.__storage__[self.__ident_func__()][name]
    except KeyError:
      raise AttributeError(name)

  def __setattr__(self, name, value):
    # 为当前线程设置属性
    ident = self.__ident_func__()
    storage = self.__storage__
    try:
      storage[ident][name] = value
    except KeyError:
      storage[ident] = {name: value}

  def __delattr__(self, name):
    try:
      del self.__storage__[self.__ident_func__()][name]
    except KeyError:
      raise AttributeError(name)
```

可以看到这个类暴露了两个属性`__storage__`和`__ident_func__`，其中`__storage__`保存了线程状态，其key为线程（或者协程）id，对应的值是一个包含线程内部信息的字典。而`__ident_func__`是根据不同情况的获取线程（或者协程）id的函数。实现了析构，`__getattr__`, `__setattr__`, `__delattr__`等操作，但是都是对于某一个线程id来操作的。其中`__getattr__`和`__getattribute__`都是访问属性的方法，区别在于，默认情况会调用`__getattribute__`，若没有找到这个属性则会调用`__getattr__`。

`__call__` 操作来创建一个 LocalProxy 对象，比如`local = Local();test_proxy = local('test');test_proxy.__repr__`，这样就把对test属性的访问做成了一个LocalProxy。LocalProxy的作用是可以动态的去产生相应的对象，而不是一开始定义好之后就不能变化了。

LocalProxy
===

LocalProxy 是一个Local对象的代理，负责把所有对自己的操作转发给内部的Local对象。LocalProxy 的构造函数接受一个callable的参数或者直接是一个Local对象，而这个callable 调用之后需要返回一个Local实例，后续所有的属性操作都会转发给 callable 返回的对象。

```
@implements_bool
class LocalProxy(object):
  # 私有对象在类中存储的名字为 _类名__属性名,这样的属性不会派生给子类
  __slots__ = ('__local', '__dict__', '__name__')

  def __init__(self, local, name=None):
    # 将获取local的可调用对象传入或者是local对象，并且设置要代理对象的名字，_LocalProxy__local表示的对于这个目前这个类的属性，不会继承
    object.__setattr__(self, '_LocalProxy__local', local)
    object.__setattr__(self, '__name__', name)

  def _get_current_object(self):
    """
    返回目前的对象，如果基于性能原因想要在代理之后真实的对象或者你想传递一个对象到不同的上下文中
    """
    # 获取当前代理的实际对象，如果没有__release_local__则说明是一个函数对象，直接返回函数返回的对象
    # 可以看出来，如果实例化传递的是一个函数，那么该方法会返回函数调用后产生的对象，也就是说代理到函数返回的对象上了
    # 如果是传递一个对象和属性名的话，就会返回该对象中该name的属性
    if not hasattr(self.__local, '__release_local__'):
      return self.__local()
    try:
      return getattr(self.__local, self.__name__)
    except AttributeError:
      raise RuntimeError('no object bound to %s' % self.__name__)

  # 后面的所有属性都代理到了self._get_current_object()上，也就是当前的local对象上
  @property
  def __dict__(self):
    try:
      return self._get_current_object().__dict__
    except RuntimeError:
      raise AttributeError('__dict__')

  def __repr__(self):
    try:
      obj = self._get_current_object()
    except RuntimeError:
      return '<%s unbound>' % self.__class__.__name__
    return repr(obj)

  def __bool__(self):
    try:
      return bool(self._get_current_object())
    except RuntimeError:
      return False

  def __unicode__(self):
    try:
      return unicode(self._get_current_object())  # noqa
    except RuntimeError:
      return repr(self)

  def __dir__(self):
    try:
      return dir(self._get_current_object())
    except RuntimeError:
      return []

  def __getattr__(self, name):
    if name == '__members__':
      return dir(self._get_current_object())
    return getattr(self._get_current_object(), name)

  def __setitem__(self, key, value):
    self._get_current_object()[key] = value

  def __delitem__(self, key):
    del self._get_current_object()[key]
```

LocalProxy可以接受一个函数，也可以接受一个对象和相应其中的属性名。对于接受函数对象来说，传入属性名就没有意义了，因为这时候就是对该函数返回对象的代理。另一种方式则就是对对象的属性的代理。比如`a = LocalProxy(get_a)`,`get_a`是一个函数对象，此时a对象，就是对`get_a`中返回的对象的代理，并且`a.b`则会取`get_a`返回的对象的b属性。

LocalProxy的好处就是可以动态的取到相应的值，而不是对象创建好了就不变了。

在Flask中`request = LocalProxy(partial(_look_req_object, 'request'))`中`_look_req_object(name)`会返回一个请求上下文相应的属性比如request，那么是如何实现线程独立的呢

```
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)
```

因为`_request_ctx_stack`，上下文栈是LocalStack()实现的，而LocalStack是基于Local实现的栈，所以其是线程独立的，所以在`_request_ctx_stack.top`就是基于本线程取出来的请求上下文。因此返回的request属性也是基于当前线程，当前请求的了。

LocalStack
===

LocalStack 是基于 Local 实现的栈结构。如果说 Local 提供了多线程或者多协程隔离的属性访问，那么 LocalStack 就提供了隔离的栈访问。下面是它的实现代码，可以看到它提供了 push、pop 和 top 方法。

`__release_local__` 可以用来清空当前线程或者协程的栈数据，`__call__` 方法返回当前线程或者协程栈顶元素的代理对象。

```
class LocalStack(object):
  def __init__(self):
    self._local = Local()

  def __release_local__(self):
    self._local.__release_local__()

  def _get__ident_func__(self):
    return self._local.__ident_func__

  def _set__ident_func__(self, value):
    # 设置相应的ident_func，即获取线程id的函数
    object.__setattr__(self._local, '__ident_func__', value)

  # 作为一个属性，get和set，stack.__indent_func__等
  __ident_func__ = property(_get__ident_func__, _set__ident_func__)

  # 删除函数对象，只是消除了LocalStack对象对于这些函数的引用
  # 防止直接去引用，现在必须使用属性来访问和写入
  del _get__ident_func__, _set__ident_func__

  # LocalStack的对象是可以调用的，返回一个代理对象，代理的是栈顶对象
  def __call__(self):
    def _lookup():
      rv = self.top
      if rv is None:
        raise RuntimeError('object unbound')
      return rv
    return LocalProxy(_lookup)

  def push(self, obj):
    """Pushes a new item to the stack"""
    rv = getattr(self._local, 'stack', None)
    # 设置local对象的栈
    if rv is None:
      # 线程独立的栈
      self._local.stack = rv = []
    rv.append(obj)
    return rv

  def pop(self):
    """Removes the topmost item from the stack, will return the
    old value or `None` if the stack was already empty.
    """
    stack = getattr(self._local, 'stack', None)
    if stack is None:
      return None
    elif len(stack) == 1:
      release_local(self._local)
      return stack[-1]
    else:
      return stack.pop()

  @property
  def top(self):
    """The topmost item on the stack.  If the stack is empty,
    `None` is returned.
    """
    try:
      return self._local.stack[-1]
    except (AttributeError, IndexError):
      return None
```

整个LocalStack就是基于线程的Local对象实现了一个栈，同时可以设置对应local对象的获取线程id的方法。可以看到应用上下文和请求上下文都是LocalStack()对象,`_request_ctx_stack = LocalStack()`, `_app_ctx_stack = LocalStack()`，这样访问就会自动定位到相应的线程中，取相应的数据。

上下文栈
===

在flask.globals中定义了请求上下文和应用上下文

```
from functools import partial
from werkzeug.local import LocalStack, LocalProxy


_request_ctx_err_msg = '''\
Working outside of request context.

This typically means that you attempted to use functionality that needed
an active HTTP request.  Consult the documentation on testing for
information about how to avoid this problem.\
'''
_app_ctx_err_msg = '''\
Working outside of application context.

This typically means that you attempted to use functionality that needed
to interface with the current application object in a way.  To solve
this set up an application context with app.app_context().  See the
documentation for more information.\
'''


def _lookup_req_object(name):
  # 获取请求上下文的栈顶对象
  top = _request_ctx_stack.top
  if top is None:
    raise RuntimeError(_request_ctx_err_msg)
  # 得到栈顶对象后，栈顶对象是一个请求对象，再获取请求对象的属性
  return getattr(top, name)


def _lookup_app_object(name):
  top = _app_ctx_stack.top
  if top is None:
    raise RuntimeError(_app_ctx_err_msg)
  return getattr(top, name)


def _find_app():
  # 获取应用上下文栈顶对象
  top = _app_ctx_stack.top
  if top is None:
    raise RuntimeError(_app_ctx_err_msg)
  return top.app


# context locals
# localStack对象
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
#
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```

request的每次调用每次都会调用 `_lookup_req_object`获取栈顶数据（线程独立）来获取保存在里面的requst context，从请求上下文中获取相应的请求对象。

请求上下文
---

见[请求上下文](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#请求上下文)

应用上下文
---

```
class AppContext(object):
    '''应用上下文隐式绑定到一个目前线程或者协程的应用对象，和请求上下文绑定到请求信息的方式类似。
    当一个请求上下文创建的时候应用上下文也会隐式的建立，但是应用不是在该独立应用上下文顶部。'''
    def __init__(self, app):
        self.app = app
        self.url_adapter = app.create_url_adapter(None)
        self.g = app.app_ctx_globals_class()
        # 和请求上下文一样，请求上下文可能被多次入栈，有一个基本的refcount足够去跟踪他们
        self._refcnt = 0

    def push(self):
        """Binds the app context to the current context."""
        self._refcnt += 1
        if hasattr(sys, 'exc_clear'):
            sys.exc_clear()
        _app_ctx_stack.push(self)
        # 发送应用上下文进栈的信号
        appcontext_pushed.send(self.app)

    def pop(self, exc=_sentinel):
        """Pops the app context."""
        try:
            self._refcnt -= 1
            if self._refcnt <= 0:
                if exc is _sentinel:
                    exc = sys.exc_info()[1]
                self.app.do_teardown_appcontext(exc)
        finally:
            rv = _app_ctx_stack.pop()
        assert rv is self, 'Popped wrong app context.  (%r instead of %r)' \
            % (rv, self)
        appcontext_popped.send(self.app)

    def __enter__(self):
        self.push()
        return self

    def __exit__(self, exc_type, exc_value, tb):
        self.pop(exc_value)

        if BROKEN_PYPY_CTXMGR_EXIT and exc_type is not None:
            reraise(exc_type, exc_value, tb)
```

可以使用Local或者LocalStack或者LocalProxy来基于这些原理来实现自己希望的东西。
