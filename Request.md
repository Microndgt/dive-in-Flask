对于 WSGI server 来说，请求又变成了文件流(套接字)，它要读取其中的内容，把 HTTP 请求包含的各种信息保存到一个字典中，调用 WSGI app； 对于 flask app 来说，请求就是一个对象，当需要某些信息的时候，只需要读取该对象的属性或者方法就行了。

`from flask import request`点击request后可以看到，

```
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
def _lookup_req_object(name):
    top = _request_ctx_stack.top
    if top is None:
        raise RuntimeError(_request_ctx_err_msg)
    return getattr(top, name)
```

也就是说request其实是请求上下文栈中的一个上下文对象里面的一个属性，在上一节解析上下文中，研究过请求上下文对象，里面有一个属性是request,一个属性是session,其中request是这样定义的

```
if request is None:
  # app的request_class是Request类，这个下来详细讲解
  # 所以创建了一个请求的Request对象
  request = app.request_class(environ)
self.request = request
```

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
```
