Contents
===

- [Flask中请求处理过程](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#flask中请求处理过程)
- [请求上下文](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#请求上下文)
- [Request类](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#request类)
  - [Flask Request](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#flask-request)
  - [Werkzeug Request](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#werkzeug-request)
  - [Werkzeug BaseRequest](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#werkzeug-baserequest)
- [总结](https://github.com/Microndgt/dive-in-Flask/blob/master/Request.md#总结)

Contents Created by [Toggle](https://github.com/Microndgt/toggle)

Request V0.1. Released at 2017/07/03.

对于WSGI server来说，请求又变成了文件流(套接字)，它要读取其中的内容，把 HTTP 请求包含的各种信息保存到一个字典中，调用 WSGI app;对于flask app来说，请求就是一个对象，当需要某些信息的时候，只需要读取该对象的属性或者方法就行了。那么flask app是如何做到从environ到request对象的转换呢，request对象实际在flask中是如何存储运作的呢。本文就是要解决这些问题。

Flask中请求处理过程
===

从上节flask的启动流程继续，当WSGI Server调用了app，并且传递进来`environ`和`start_response`。按
照WSGI协议，Flask应用要实现一个可调用对象，在Flask中就是`__call__()`方法：

```
def __call__(self, environ, start_response):
        """Shortcut for :attr:`wsgi_app`."""
        return self.wsgi_app(environ, start_response)
```

`__call__()`方法调用了`wsgi_app()`方法：

```
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)
    ctx.push()
    error = None
    try:
        try:
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)
```

在`wsgi_app`方法上，可以看到，首先通过传入的environ创建了一个请求上下文对象，然后将自己入栈，在此同时，如果没有应用上下文，也会将应用上下文入栈。结下来就是分发请求到视图函数上进行处理。所以跳到`self.full_dispatch_request`方法上。

```
def full_dispatch_request(self):
    '''分发请求，并且在这之前进行请求预处理和后处理并且处理HTTP异常和错误'''
    self.try_trigger_before_first_request_functions()
    try:
        request_started.send(self)
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
    except Exception as e:
        rv = self.handle_user_exception(e)
    return self.finalize_request(rv)
```

首先是处理整个app第一个请求，每个app启动时候只处理一次，在应用上就是使用`before_first_request`装饰器

```
def try_trigger_before_first_request_functions(self):
    '''在每个请求处理之前调用确保触发了before_first_request_funcs，对于每一个应用实例只执行一次'''
    # 表征是否接受到了第一个请求
    if self._got_first_request:
        return
    with self._before_request_lock:
        if self._got_first_request:
            return
        for func in self.before_first_request_funcs:
            func()
        self._got_first_request = True
```

这里在app处理第一个请求前，`self._got_first_request`为False，因此会执行所有被`before_first_request`装饰器装饰的函数，因为涉及到公用属性的修改，可能app有多个线程来处理请求，因此需要获得锁，这些函数只需要执行一次，执行完毕之后，`self._got_first_request`被置为True

执行完毕之后，`request_started`开始发送信号，表示请求处理开始，这些信号可以在实际开发应用中使用，进行信号订阅来执行相关动作。里面包含了许多在Flask应用状态改变的信号：

```
signals_available = False
try:
    from blinker import Namespace
    signals_available = True
except ImportError:
    class Namespace(object):
        def signal(self, name, doc=None):
            return _FakeSignal(name, doc)
    class _FakeSignal(object):
        """If blinker is unavailable, create a fake class with the same
        interface that allows sending of signals but will fail with an
        error on anything else.  Instead of doing anything on send, it
        will just ignore the arguments and do nothing instead.
        """

        def __init__(self, name, doc=None):
            self.name = name
            self.__doc__ = doc
        def _fail(self, *args, **kwargs):
            raise RuntimeError('signalling support is unavailable '
                               'because the blinker library is '
                               'not installed.')
        send = lambda *a, **kw: None
        connect = disconnect = has_receivers_for = receivers_for = \
            temporarily_connected_to = connected_to = _fail
        del _fail
_signals = Namespace()
template_rendered = _signals.signal('template-rendered')
before_render_template = _signals.signal('before-render-template')
request_started = _signals.signal('request-started')
request_finished = _signals.signal('request-finished')
request_tearing_down = _signals.signal('request-tearing-down')
got_request_exception = _signals.signal('got-request-exception')
appcontext_tearing_down = _signals.signal('appcontext-tearing-down')
appcontext_pushed = _signals.signal('appcontext-pushed')
appcontext_popped = _signals.signal('appcontext-popped')
message_flashed = _signals.signal('message-flashed')
```

这里需要从blinker获取支持，导入Namespace，从而创建信号，但是如果导入失败，那么就应该给一个友好的提示，这里就创建了一个Namespace类，使用了一个可以进行传送信号，但是会失败的类。使用Namespace的好处是唯一的信号名，并且将不同的信号源独立起来，这样方便编码和区分调试。关于Flask中信号的使用，见[Flask信号](http://skyrover.me/post/16/)

发送信号之后，调用`rv = self.preprocess_request()`进行请求预处理，这里处理的是`before_request`装饰的函数，也就是在每个请求前进行处理的东西。Flask的钩子函数就是这样的处理机制。

```
def preprocess_request(self):
    '''在真实请求分发前调用，会调用before_request装饰的函数，不传递参数。
    如果其中任何一个函数返回了值，那么就把这个值当成了视图函数的返回，后续的请求处理将会停止。
    这也会触发url_value_preprocessor函数，在before_request函数之前调用
    '''
    bp = _request_ctx_stack.top.request.blueprint
    funcs = self.url_value_preprocessors.get(None, ())
    # URL处理器
    if bp is not None and bp in self.url_value_preprocessors:
        funcs = chain(funcs, self.url_value_preprocessors[bp])
    for func in funcs:
        func(request.endpoint, request.view_args)
    # 请求前处理，如果有返回值，那么就返回，返回后就不会继续下一步的请求了
    funcs = self.before_request_funcs.get(None, ())
    if bp is not None and bp in self.before_request_funcs:
        funcs = chain(funcs, self.before_request_funcs[bp])
    for func in funcs:
        rv = func()
        if rv is not None:
            return rv
```

预处理请求部分主要完成的任务是，url处理器和`before_request`的处理内容。完成预处理请求部分之后，如果没有返回值，那么分发请求到真实的视图函数上。

```
def dispatch_request(self):
    '''请求分发，匹配URL并且返回视图函数或者错误处理器的值，不必是一个响应对象。可以调用make_response来将返回值加工成一个响应对象。
    '''
    req = _request_ctx_stack.top.request
    if req.routing_exception is not None:
        self.raise_routing_exception(req)
    rule = req.url_rule
    # 如果自动处理options请求
    if getattr(rule, 'provide_automatic_options', False) \
           and req.method == 'OPTIONS':
        return self.make_default_options_response()
    # 分发请求到路由函数上
    return self.view_functions[rule.endpoint](**req.view_args)
```

这里的请求分发就是路由系统做的事情，定位到响应的视图函数中执行处理，获取路由函数的返回值后，就调用`self.finalize_request(rv)`来包装返回值为response对象，自此正常的请求的处理基本完成。

请求上下文
===

查看`self.request_context()`方法：

```
def request_context(self, environ):
    '''从给定的环境中创建请求上下文，并且绑定当前上下文上。必须使用with语句因为with可以让其绑定到当前的上下文中。
    with app.request_context(environ):
        do_something_with(request)
    当然也可以不使用with语句
    ctx = app.request_context(environ)
        ctx.push()
    try:
        do_something_with(request)
    finally:
        ctx.pop()
    '''
    return RequestContext(self, environ)
```

下面是RequestContext的定义：

```
class RequestContext(object):
    '''请求上下文包含所有请求相关的信息，在请求一开始的时候创建，然后进入_request_ctx_stack栈中，在结束的时候移除。它会创建一个URL适配器和基于WSGI环境的请求对象。
    不要直接使用这个类，而是使用Flask.test_request_context和Flask.request_context

    当请求上下文出栈的时候，它会执行所有注册在Flask.teardown_request上面的函数

    请求上下文会自动出栈。在debug模式中，如果出现异常请求上下文会保留，这样交互式的debuggers才可以去检查数据。
    '''

    def __init__(self, app, environ, request=None):
        self.app = app
        if request is None:
            # 创建一个请求对象，保存请求的相关信息
            request = app.request_class(environ)
        self.request = request
        # 创建一个url适配器,这个是路由系统做的事情
        self.url_adapter = app.create_url_adapter(self.request)
        self.flashes = None
        # session也是请求上下文对象的一部分
        self.session = None

        # 请求上下文对象可能会被多次入栈，并且和其他上下文对象交错，所以只有等到其他的曾被pop之后才开始处理这个上下文。另外如果应用上下文缺失的话，就会隐式的为每一层请求上下文创建一个应用上下文
        self._implicit_app_ctx_stack = []
        # 指示上下文是否要保存，False 下一个上下文压栈则这个上下文就会弹出
        self.preserved = False

        self._preserved_exc = None
        self._after_request_functions = []
        # url适配器来匹配url
        self.match_request()

    def _get_g(self):
        return _app_ctx_stack.top.g

    def _set_g(self, value):
        _app_ctx_stack.top.g = value

    # 将g作为一个属性
    g = property(_get_g, _set_g)
    del _get_g, _set_g

    def copy(self):
        '''使用相同的请求对象创建一个请求上下文的拷贝，可以用来将
        一个请求上下文移动到不同的greenlet（协程)，因为实际的请求对象是一样的，所以不能用来移动到不同的线程中，除非加锁。'''

        return self.__class__(self.app,
            environ=self.request.environ,
            request=self.request
        )

    def match_request(self):
        '''url适配器的工作，url_adapter.match返回一个Rule对象url_rule```
        try:
            url_rule, self.request.view_args = \
                self.url_adapter.match(return_rule=True)
            self.request.url_rule = url_rule
        except HTTPException as e:
            self.request.routing_exception = e

    def push(self):
        '''绑定应用上下文到目前的上下文中'''
        top = _request_ctx_stack.top
        # 如果在debug模式有异常发生或者在出现异常上下文保存被激活，那么就会有一个上下文在栈中。
        # 其原理就是在debug模式想要获取上下文的信息。然而如果有人忘记将上下文出栈，那么我们要保证下一次入栈时候这个上下文就无效。否则就可能有内存泄漏的可能。
        if top is not None and top.preserved:
            top.pop(top._preserved_exc)

        # 在请求上下文入栈前需要保证将有一个应用上下文。

        app_ctx = _app_ctx_stack.top
        if app_ctx is None or app_ctx.app != self.app:
            app_ctx = self.app.app_context()
            app_ctx.push()
            self._implicit_app_ctx_stack.append(app_ctx)
        else:
            # 如果top是当前的应用，隐式的上下文栈进入None
            self._implicit_app_ctx_stack.append(None)

        if hasattr(sys, 'exc_clear'):
            sys.exc_clear()
        # 请求上下文入栈
        _request_ctx_stack.push(self)

        self.session = self.app.open_session(self.request)
        if self.session is None:
            self.session = self.app.make_null_session()

    def pop(self, exc=_sentinel):
        '''弹出请求上下文，并且解除绑定，并且会触发注册在teardown_request的函数'''
        app_ctx = self._implicit_app_ctx_stack.pop()
        try:
            clear_request = False
            # 如果app_ctx为False，表示没有创建新的app上下文
            if not self._implicit_app_ctx_stack:
                self.preserved = False
                self._preserved_exc = None
                if exc is _sentinel:
                    exc = sys.exc_info()[1]
                self.app.do_teardown_request(exc)
                if hasattr(sys, 'exc_clear'):
                    sys.exc_clear()
                request_close = getattr(self.request, 'close', None)
                if request_close is not None:
                    request_close()
                clear_request = True
        finally:
            rv = _request_ctx_stack.pop()
            if clear_request:
                rv.request.environ['werkzeug.request'] = None

            # Get rid of the app as well if necessary.
            if app_ctx is not None:
                app_ctx.pop(exc)
            assert rv is self, 'Popped wrong request context.  ' \
                '(%r instead of %r)' % (rv, self)
    def auto_pop(self, exc):
        if self.request.environ.get('flask._preserve_context') or \
           (exc is not None and self.app.preserve_context_on_exception):
            self.preserved = True
            self._preserved_exc = exc
        else:
            self.pop(exc)

    def __enter__(self):
        self.push()
        return self

    def __exit__(self, exc_type, exc_value, tb):
        # do not pop the request stack if we are in debug mode and an
        # exception happened.  This will allow the debugger to still
        # access the request object in the interactive shell.  Furthermore
        # the context can be force kept alive for the test client.
        # See flask.testing for how this works.
        self.auto_pop(exc_value)

        if BROKEN_PYPY_CTXMGR_EXIT and exc_type is not None:
            reraise(exc_type, exc_value, tb)

    def __repr__(self):
        return '<%s \'%s\' [%s] of %s>' % (
            self.__class__.__name__,
            self.request.url,
            self.request.method,
            self.app.name,
        )
```

请求上下文的栈是通过一个线程独立的类来实现的，所以保证了每次对于一个线程来处理请求，这个线程的应用上下文栈和请求上下文的栈都是独立于其他的线程的，这样视图函数就可以直接从栈中获取这些信息。在一个请求中，如果在一个线程中，应用上下文和请求上下文都是在栈中，所以可以取到相关的信息，如果进入了比如Celery的一个worker里面，就可能无法使用这些东西，必须将app入栈，才可以使用`current_app`等信息，也就是给当前处理线程创建一份app的上下文。

Request类
===

Flask Request
---

`from flask import request`点击request后可以看到

```
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)
```

也就是说request其实是请求上下文栈中的一个上下文对象里面的一个属性，这个属性其实是一个Request对象。这个request对象是这样定义的

```
if request is None:
  # app的request_class是Request类，这个下来详细讲解
  # 所以创建了一个请求的Request对象
  request = app.request_class(environ)
self.request = request
```

而`app.request_class`是`request_class = Request`，这个Request类定义在`flask.wrappers`，继承自RequestBase

可以看到request是定义在app中的一个属性，传入了environ参数所创建的，转入Flask类，可以看到一个类属性`request_class = Request`，因此请求上下文对象中的request属性是Request类的一个实例，继续向上溯源，可以得到Request类的源码：

```
class Request(RequestBase):
    '''Flask默认使用的request对象，保存了匹配的endpoint和参数，如果替换这个request对象，可以子类化Request，并且设置Flask里面的request_class为这个子类。
    这个Request类是werkzeug.wrappers.Request的子类，提供了werkzeug的所有定义的属性，以及Flask指定的属性'''
    # 内部的URL规则匹配请求，在通过before/after处理器检查哪种方法是被这个URL允许是比较有用的
    url_rule = None
    # 匹配这个请求的视图参数字典，如果在匹配过程中发生异常，None
    view_args = None
    # 如果匹配URL失败，这个异常将会被作为请求处理的一部分被raised，通常是werkzeug.exceptions.NotFound
    routing_exception = None
    # 被请求上下文切换直到1.0被弃用
    _is_old_module = False
    @property
    def max_content_length(self):
        """Read-only view of the ``MAX_CONTENT_LENGTH`` config key."""
        ctx = _request_ctx_stack.top
        if ctx is not None:
            ＃ 取得请求上下文绑定的应用的配置
            return ctx.app.config['MAX_CONTENT_LENGTH']
    @property
    def endpoint(self):
        '''匹配这个请求的endpoint，与view_args组合起来就可以用来重构相同的或者修改过的URL'''
        if self.url_rule is not None:
            return self.url_rule.endpoint
    @property
    def module(self):
        '''当前模块的名字，如果请求分发到了一个具体的模块上。这个已经被弃用了，而是使用蓝图'''
        from warnings import warn
        warn(DeprecationWarning('modules were deprecated in favor of '
                                'blueprints.  Use request.blueprint '
                                'instead.'), stacklevel=2)
        if self._is_old_module:
            return self.blueprint
    @property
    def blueprint(self):
        '''当前蓝图的名字'''
        if self.url_rule and '.' in self.url_rule.endpoint:
            # 只切分一次，第一个值就是蓝图的名字，比如auth.index
            return self.url_rule.endpoint.rsplit('.', 1)[0]
    @property
    def json(self):
        '''如果mimetype指定为application/json，将会包含一个json数据，其他情况就是None，
        应该使用get_json'''
        from warnings import warn
        warn(DeprecationWarning('json is deprecated.  '
                                'Use get_json() instead.'), stacklevel=2)
        return self.get_json()
    @property
    def is_json(self):
        '''用来判断一个请求是不是Json，默认一个请求应该包含Json数据如果mimetype是
        application/json或者application/*+json，仍然是一个属性'''
        mt = self.mimetype
        if mt == 'application/json':
            return True
        if mt.startswith('application/') and mt.endswith('+json'):
            return True
        return False
    def get_json(self, force=False, silent=False, cache=True):
        '''解析进来的Json请求并且返回Json数据，如果mimetype不是json默认会返回None，但是可以通过force参数来忽略mimetype并且强制解析json数据，如果解析失败，那么on_json_loading_failed方法将会被调用
        silent如果解析失败则不会提示
        cache是否缓存本次请求的json数据'''
        # _missing = object()
        rv = getattr(self, '_cached_json', _missing)
        # 读取缓存数据
        if cache and rv is not _missing:
            return rv
        # 如果force为True，强制解析json数据，如果为False，只有当mimetype为json时候才会解析
        if not (force or self.is_json):
            return None
        ＃ self.mimetype_params是werkzeug定义的，获取请求的字符集
        request_charset = self.mimetype_params.get('charset')
        try:
            data = _get_data(self, cache)
            if request_charset is not None:
                rv = json.loads(data, encoding=request_charset)
            else:
                rv = json.loads(data)
        except ValueError as e:
            if silent:
                rv = None
            else:
                rv = self.on_json_loading_failed(e)
        if cache:
            self._cached_json = rv
        return rv
    def on_json_loading_failed(self, e):
        '''如果json数据解析失败，则调用，主要是raise一个BadRequest异常'''
        ctx = _request_ctx_stack.top
        if ctx is not None and ctx.app.config.get('DEBUG', False):
            raise BadRequest('Failed to decode JSON object: {0}'.format(e))
        raise BadRequest()
    def _load_form_data(self):
        # 继承的方法，这个方法主要解决问题是替换files multidict专门的子类来为key error raise不同error
        RequestBase._load_form_data(self)
        if ctx is not None and ctx.app.debug and \
           self.mimetype != 'multipart/form-data' and not self.files:
            from .debughelpers import attach_enctype_error_multidict
            attach_enctype_error_multidict(self)

def _get_data(req, cache):
    # 取得get_data的属性，用来获取数据
    getter = getattr(req, 'get_data', None)
    if getter is not None:
        return getter(cache=cache)
    # 否则调用data属性
    return req.data

def attach_enctype_error_multidict(request):
    """如果reqeust监测到没有使用多表单数据，但是进行存取文件对象的话，raise DebugFilesKeyError
    """
    oldcls = request.files.__class__
    class newcls(oldcls):
        # 重写了旧类的__getitem__方法
        def __getitem__(self, key):
            try:
                return oldcls.__getitem__(self, key)
            except KeyError:
                if key not in request.form:
                    raise
                raise DebugFilesKeyError(request, key)
    newcls.__name__ = oldcls.__name__
    newcls.__module__ = oldcls.__module__
    # 直接将类替换
    request.files.__class__ = newcls
```

Werkzeug Request
---

这是Flask中对于Request进行的扩展，但是究竟从environ如何创建出一个Request对象呢，那么就必须继续向上探索，这将到werkzeug了，点击RequestBase，进入到了werkzeug/wrappers.py中的Request类

```
class Request(BaseRequest, AcceptMixin, ETagRequestMixin,
              UserAgentMixin, AuthorizationMixin,
              CommonRequestDescriptorsMixin):
    '''request对象的所有特性，通过下面这些mixins实现的
    AcceptMixin: 用来进行accept header的解析
    ETagRequestMixin: 用来etag和cache控制处理，把Last-Modified和ETags请求的http报头一起使用，这样可利用客户端（例如浏览器）的缓存。
    UserAgentMixin: user agent相关
    AuthorizationMixin: http auth处理
    CommonRequestDescriptorsMixin: 一般的headers处理
```

可以看到这个类是由许多mixin来组成的，mixin用来来扩展其他类的功能,这些类单独使用起来没有任何意义，事实上如果你去实例化任何一个类，除了产生异常外没任何作用。 它们是用来通过多继承来和其他对象混入使用的,用途同样是增强已存在的类的功能和一些可选特征。

对于混入类，有几点需要记住。首先是，混入类不能直接被实例化使用。 其次，混入类没有自己的状态信息，也就是说它们并没有定义 `__init__()` 方法，并且没有实例属性。

Werkzeug BaseRequest
---

首先研究BaseRequest这个类：

```
class BaseRequest(object):
    '''非常基础的request对象，没有实现一些高级的用法比如实体标签解析或者缓存控制。request对象通过将WSGI环境作为第一个参数传入创建，并且会将自己加入到WSGI环境中，名字是werkzeug.request,除非创建request时候populate_request设置为False
    同时也有一些可用的mixins来添加额外的功能，比如继承于BaseRequest的Request就有很多重要的mixins

    创建一个自己的子类并且添加缺失的功能，通过mixins或者直接的实现

    from werkzeug.wrappers import BaseRequest, ETagRequestMixin

    class Request(BaseRequest, ETagRequestMixin):
        pass

    Request对象是只读的，与低级解析函数不同，request对象将会使用不可变对象。

    request对象将默认假定所有的text数据是utf-8编码的。

    默认request对象将会被加入到WSGI环境中-werkzeug.request去支持测试系统，如果不需要这样，设置populate_request为False

    如果shallow是True的话，环境就会被初始化为shallow对象，每个操作如果修改environ，就会产生异常除非shallow显式的设置为False。这个对中间件很有用，如果你不想无意间消费了表单数据。shallow请求在WSGI对象中并不是很流行
    '''
    charset = 'utf-8'
    encoding_errors = 'replace'
    # 这个将会被转送到表单数据解析函数里,parse_form_data
    max_content_length = None
    # 最大的表单字段长度
    max_form_memory_size = None
    # 这个类是给args和form使用的，默认是这个不可变的字典支持一个key多个value，可选的可以使用ImmutableOrderedMultiDict，或者ImmutableDict更快但是只会记录最会一个key，不建议使用可变字典，以下的数据结构都将在werkzeug节解析
    parameter_storage_class = ImmutableMultiDict
    list_storage_class = ImmutableList
    dict_storage_class = ImmutableTypeConversionDict
    # 一个表单数据解析器
    form_data_parser_class = FormDataParser
    # 在这个请求中信任的主机列表，默认所有的主机都会被信任，意味着不论什么客户端发送数据，主机都会接受，推荐的做法是一个web服务器应该手动的设置唯一正确的主机，移除X-Forwarded-Host头信息如果没有使用的话
    trusted_hosts = None
    # 指示数据描述符是否应该被允许读取和缓存输入流，默认是不允许的
    disable_data_descriptor = False
    def __init__(self, environ, populate_request=True, shallow=False):
        self.environ = environ
        if populate_request and not shallow:
            self.environ['werkzeug.request'] = self
        self.shallow = shallow
    def __repr__(self):
        '''确保该方法能正常工作，即使request对象是使用非法WSGI环境创建的'''
        try:
            args.append("'%s'" % to_native(self.url, self.url_charset))
            args.append('[%s]' % self.method)
        except Exception:
            args.append('(invalid WSGI environ)')
        return '<%s %s>' % (
                self.__class__.__name__,
                ' '.join(args)
            )
    @property
    def url_charset(self):
        return self.charset
    @classmethod
    def from_values(cls, *args, **kwargs):
        '''基于给的值来创建一个新的请求对象'''
        from werkzeug.test import EnvironBuilder
        # 这块这么写貌似不太友好。。直接设置kwargs['charset'] = cls.charset就可以么
        charset = kwargs.pop('charset', cls.charset)
        kwargs['charset'] = charset
        builder = EnvironBuilder(*args, **kwargs)
        try:
            return builder.get_request(cls)
        finally:
            builder.close()
    @classmethod
    def application(cls, f):
        '''装饰了一个作为响应器的函数，接受一个request作为一个参数，就像responder一样工作，但是函数传递一个request对象作为第一个参数并且请求对象可以被自动关闭。
        @Request.application
        def my_wsgi_app(request):
            return Response("hello world")
        这个函数应该是request参数指的就是每次请求的参数吧，包含environ，start_response,request
        '''
        def application(*args):
            # 使用Request类传入environ实例一个request对象
            request = cls(args[-2])
            with request:
                return f(*args[:-2] + (request,))(*args[-2:])
        # 返回了已经更新函数信息的application函数
        return update_wrapper(application, f)
    def _get_file_stream(self, total_content_length, content_type, filename=None, content_length=None):
        '''调用去获取上传文件流
        必须提供一个文件对象类，有read(), readline(),seek()方法，并且是可读可写的
        如果总长度大于500k那么默认实现会返回一个临时文件，因为多数浏览器不提供文件的内容长度，所以只有总长度起作用。mimetype资源媒体类型
        '''
        # 下面这个将在werkzeug中的FormDataParser解析到
        return default_stream_factory(total_content_length, content_type,
                                      filename, content_length)
    @property
    def want_form_data_parsed(self):
        '''如果请求方法包含内容，则返回True，目前是有content_type则返回True'''
        return bool(self.environ.get('CONTENT_TYPE'))
    def make_form_data_parser(self):
        '''这个函数创建了一个表单数据解析器，form_data_parser_class其实是FormDataParser'''
        return self.form_data_parser_class(self._get_file_stream,
                                       self.charset,
                                       self.encoding_errors,
                                       self.max_form_memory_size,
                                       self.max_content_length,
                                       self.parameter_storage_class)
    def _load_form_data(self):
        '''内部使用这个方法来获取提交的数据。在调用这个函数之后，将会在request对象上的dict设置form和files这两个key，内容就是提交的数据。事实上，输入流在之后将会为空，可以调用这个方法强制去解析表单数据'''
        if 'form' in self.__dict__:
            return
        _assert_not_shallow(self)
        # 如果有数据内容的话
        if self.want_form_data_parsed:
            content_type = self.environ.get('CONTENT_TYPE', '')
            # 这个函数是werkzeug.wsgi中实现的
            content_length = get_content_length(self.environ)
            mimetype, options = parse_options_header(content_type)
            parser = self.make_form_data_parser()
            data = parser.parse(self._get_stream_for_parsing(), mimetype, content_length, options)
        else:
            # 默认数据都是ImmutableMultiDict
            data = (self.stream, self.parameter_storage_class(),
                    self.parameter_storage_class())
        d = self.__dict__
        d['stream'], d['form'], d['files'] = data
    def _get_stream_for_parsing(self):
        cached_data = getattr(self, '_cached_data', None)
        if cached_data is not None:
            return BytesIO(cached_data)
        return self.stream
    def close(self):
        '''关闭request对象的相关资源，显式的关闭所有的文件句柄'''
        # ImmutableMultiDict对象
        files = self.__dict__.get('files')
        for key, value in iter_multi_items(files or ()):
            value.close()
    def __enter__(self):
        return self
    def __exit__(self, exc_type, exc_value, tb):
        self.close()
    @cached_property
    def stream(self):
        '''读取传入的数据的流，这个流被合适的保护着，你不能意外的去读取过去输入的长度。werkzeug内部会经常使用这个数据流读取数据。
        '''
        assert_not_shallow(self)
        return get_input_stream(self.environ)
    input_stream = environ_property('wsgi.input', """
    The WSGI input stream.

    In general it's a bad idea to use this one because you can easily read past
    the boundary.  Use the :attr:`stream` instead.
    """)
    @cached_property
    def args(self):
        '''解析的url参数，默认是一个ImmutableMultiDict'''
        return url_decode(wsgi_get_bytes(self.environ.get('QUERY_STRING', '')),
                          self.url_charset, errors=self.encoding_errors,
                          cls=self.parameter_storage_class)
    @cached_property
    def data(self):
        if self.disable_data_descriptor:
            raise AttributeError('data descriptor is disabled')
        # 在第一次的时候触发表单数据解析意味着描述符将不会cache表单或者文件数据。
        return self.get_data(parse_form_data=True)
    def get_data(self, cache=True, as_text=False, parse_form_data=False):
        '''从客户端读取缓存的数据至bytestring。通常会被缓存下来。在没有检查数据长度的情况下不应该直接调用这个方法，因为可能会有大量的数据，导致服务器的存储问题。
        如果表单数据已经被解析过了，这个函数将不会返回任何数据如果没有缓存的话。隐式调用表单数据解析函数，设置parse_form_data为True。
        rv = getattr(self, '_cached_data', None)
        if rv is None:
            if parse_form_data:
                self._load_form_data()
            rv = self.stream.read()
            if cache:
                self._cached_data = rv
        if as_text:
            rv = rv.decode(self.charset, self.encoding_errors)
        return rv
    @cached_property
    def form(self):
        self._load_form_data()
        return self.form
    @cached_property
    def values(self):
        """Combined multi dict for :attr:`args` and :attr:`form`."""
        args = []
        for d in self.args, self.form:
            if not isinstance(d, MultiDict):
                d = MultiDict(d)
            args.append(d)
        return CombinedMultiDict(args)
    @cached_property
    def files(self):
        '''一个multidict对象，包含了所有的上传文件。每个files是一个FileStorage对象'''
        self._load_form_data()
        return self.files
    @cached_property
    def headers(self):
        '''WSGI环境形成的headers，不可变'''
        return EnvironHeaders(self.environ)
    @cached_property
    def cookies(self):
        """Read only access to the retrieved cookie values as dictionary."""
        return parse_cookie(self.environ, self.charset,
                            self.encoding_errors,
                            cls=self.dict_storage_class)
    @cached_property
    def path(self):
        '''请求的路径'''
        raw_path = wsgi_decoding_dance(self.environ.get('PATH_INFO') or '',
                                       self.charset, self.encoding_errors)
        return '/' + raw_path.lstrip('/')
    @cached_property
    def full_path(self):
        """请求路径，包含查询字符串"""
        return self.path + u'?' + to_unicode(self.query_string, self.url_charset)
    @cached_property
    def script_root(self):
        """没有尾斜线的脚本根路径"""
        raw_path = wsgi_decoding_dance(self.environ.get('SCRIPT_NAME') or '',
                                       self.charset, self.encoding_errors)
        return raw_path.rstrip('/')
    @cached_property
    def url(self):
        return get_current_url(self.environ,
                           trusted_hosts=self.trusted_hosts)
    @cached_property
    def base_url(self):
        return get_current_url(self.environ, strip_querystring=True,
                           trusted_hosts=self.trusted_hosts)
    @cached_property
    def url_root(self):
        return get_current_url(self.environ, True,
                           trusted_hosts=self.trusted_hosts)
    @cached_property
    def host_url(self):
        return get_current_url(self.environ, host_only=True,
                           trusted_hosts=self.trusted_hosts)
    @cached_property
    def host(self):
        return get_host(self.environ, trusted_hosts=self.trusted_hosts)
    query_string = environ_property(
        'QUERY_STRING', '', read_only=True,
        load_func=wsgi_get_bytes, doc='The URL parameters as raw bytestring.')
    method = environ_property(
        'REQUEST_METHOD', 'GET', read_only=True,
        load_func=lambda x: x.upper(),
        doc="The transmission method. (For example ``'GET'`` or ``'POST'``).")
    @property
    def remote_addr(self):
        """The remote address of the client."""
        return self.environ.get('REMOTE_ADDR')
    @cached_property
    def access_route(self):
        """If a forwarded header exists this is a list of all ip addresses
        from the client ip to the last proxy server.
        """
        if 'HTTP_X_FORWARDED_FOR' in self.environ:
            addr = self.environ['HTTP_X_FORWARDED_FOR'].split(',')
            return self.list_storage_class([x.strip() for x in addr])
        elif 'REMOTE_ADDR' in self.environ:
            return self.list_storage_class([self.environ['REMOTE_ADDR']])
        return self.list_storage_class()
    remote_user = environ_property('REMOTE_USER', doc='''
        If the server supports user authentication, and the script is
        protected, this attribute contains the username the user has
        authenticated as.''')

    scheme = environ_property('wsgi.url_scheme', doc='''
        URL scheme (http or https).

        .. versionadded:: 0.7''')

    is_xhr = property(lambda x: x.environ.get('HTTP_X_REQUESTED_WITH', '')
                      .lower() == 'xmlhttprequest', doc='''
        True if the request was triggered via a JavaScript XMLHttpRequest.
        This only works with libraries that support the `X-Requested-With`
        header and set it to "XMLHttpRequest".  Libraries that do that are
        prototype, jQuery and Mochikit and probably some more.''')
    is_secure = property(lambda x: x.environ['wsgi.url_scheme'] == 'https',
                         doc='`True` if the request is secure.')
    is_multithread = environ_property('wsgi.multithread', doc='''
        boolean that is `True` if the application is served by
        a multithreaded WSGI server.''')
    is_multiprocess = environ_property('wsgi.multiprocess', doc='''
        boolean that is `True` if the application is served by
        a WSGI server that spawns multiple processes.''')
    is_run_once = environ_property('wsgi.run_once', doc='''
        boolean that is `True` if the application will be executed only
        once in a process lifetime.  This is the case for CGI for example,
        but it's not guaranteed that the execution only happens one time.''')
```

这个类相当长，所包含内容也相当多，这里只对基本的内容做了解析，其他的内容暂时没有深入去解析。

总结
===

首先是从WSGI server传递过来的environ，使用RequestContext创建了一个请求上下文对象，在请求上下文对象中使用Request类来创建了一个请求对象。然后Flask开始处理请求，将其分发到响应的路由上。在路由所对应的视图函数上处理时，请求上下文入栈，这个栈是线程独立的，所以对于每个线程处理每个请求时，栈都是独立的。然后在视图函数上就可以使用到请求相关的信息，比如`from flask import request`
