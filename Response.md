Contents
===

- [从视图函数返回到Response](https://github.com/Microndgt/dive-in-Flask/blob/master/Response.md#%E4%BB%8E%E8%A7%86%E5%9B%BE%E5%87%BD%E6%95%B0%E8%BF%94%E5%9B%9E%E5%88%B0response)
- [BaseResponse](https://github.com/Microndgt/dive-in-Flask/blob/master/Response.md#baseresponse)
- [通过BaseResponse来生成响应](https://github.com/Microndgt/dive-in-Flask/blob/master/Response.md#%E9%80%9A%E8%BF%87baseresponse%E6%9D%A5%E7%94%9F%E6%88%90%E5%93%8D%E5%BA%94)
- [小结](https://github.com/Microndgt/dive-in-Flask/blob/master/Response.md#%E5%B0%8F%E7%BB%93)

Contents Created by [Toggle](https://github.com/Microndgt/toggle)

现在才发现解析源码和一个无底洞一样，总有其他相关的代码需要去深入理解。

现在终于到Response了，一个web框架从socket套接字编程开始，到接受请求，处理请求，WSGI协议的Server，然后再到中间件，最后到框架处理业务逻辑，之后再返回响应，这其中还要考虑到缓存，数据库，高并发等等相关的东西，将这些东西一一弄清楚，也许你就大成了。现在我们就到了框架解析的最后一步`Response`。

从视图函数返回到Response
===

首先还是从Flask类中的`wsgi_app`方法说起，当创建了请求上下文后，就开始分发请求到视图函数上，这一步骤已经在前面解析过了，下来就是视图函数返回，生成Response响应，返回给中间件或者Server。那么现在来看看视图函数生成响应之后发生了什么。在Request一节里提过，在完成路由函数的调用`rv = self.dispatch_request()`之后，rv就是路由函数返回的东西，最后一步要返回`self.finalize_request(rv)`调用的结果，下面看看这个方法。

```
def finalize_request(self, rv, from_error_handler=False):
    """传递视图函数的返回值，在这里结束请求处理，将其转换成response对象，并且调用postprocessing函数，这个方法不仅在正常时调用，也在错误时候调用。

    因为这意味这它可能因为失败而调用，因此需要使用 from_error_handler 置为True来启动特别安全模式。如果启动了，在response处理的错误将会被记录下来，否则将被忽略
    """
    response = self.make_response(rv)
    try:
        response = self.process_response(response)
        # 发送请求处理完成的信号
        request_finished.send(self, response=response)
    except Exception:
        if not from_error_handler:
            raise
        self.logger.exception('Request finalizing failed with an '
                              'error while handling an error')
    return response
```

可以看到，生成response的主要有两个方法，一个是`self.make_response(rv)`，这个直接使用视图函数的返回值来处理，返回一个response，一个是`self.process_response(response)`，使用上一个方法返回的response作为参数，生成最终的response，究竟有什么区别呢，请看下面分解：

```
def make_response(self, rv):
    """将视图函数返回的值转换成response_class的实例，一个真正的response对象，这个response_class默认是Response类，这个类继承Werkzeug的Response，这个后面再讲

    下面是rv被允许的类型

    .. tabularcolumns:: |p{3.5cm}|p{9.5cm}|

    ======================= ===========================================
    :attr:`response_class`  这个对象将直接被返回,flask的jsonify就是生成了这个对象，首先dumps，然后交给response_class实例化
    :class:`str`            将会使用string作为body创建一个response对象
    :class:`unicode`        将会将string编码成为utf-8作为body创建一个response对象
    a WSGI function         函数将会作为WSGI应用调用，并且作为response对象缓存
    :class:`tuple`          以(response, status, headers)格式的元组，或者(response, headers)的元组，其中response是以上任何类型，status是字符串或者数字，headers是一个列表或者包含header值的字典
    ======================= ===========================================

    :param rv: 视图函数的返回值
    """
    status_or_headers = headers = None
    if isinstance(rv, tuple):
        # 机智的解包操作
        rv, status_or_headers, headers = rv + (None,) * (3 - len(rv))

    if rv is None:
        raise ValueError('View function did not return a response')

    # 如果传入的是元组，并且status_or_headers是一个字典或者列表的化，那么应该是headers
    if isinstance(status_or_headers, (dict, list)):
        headers, status_or_headers = status_or_headers, None

    # 如果rv不是response对象
    if not isinstance(rv, self.response_class):
        # 当我们直接创建了一个response对象，我们让构造器设置headers和状态，这样做是因为在创建这些对象的时候可能会有一些而外的逻辑，比如默认的content_type选择
        if isinstance(rv, (text_type, bytes, bytearray)):
            rv = self.response_class(rv, headers=headers,
                                     status=status_or_headers)
            headers = status_or_headers = None
        else:
            # 处理可能是函数的类型
            rv = self.response_class.force_type(rv, request.environ)

    if status_or_headers is not None:
        if isinstance(status_or_headers, string_types):
            rv.status = status_or_headers
        else:
            rv.status_code = status_or_headers
    if headers:
        rv.headers.extend(headers)

    return rv
```

`self.make_response(rv)`返回了一个Response对象，当然这个response_class是可以自己定制的，下面看看`self.process_response(response)`

```
def process_response(self, response):
    """可以被重写用来在发送给WSGI服务器前修改response对象，默认这个会调用after_request装饰器的函数
    """
    ctx = _request_ctx_stack.top
    bp = ctx.request.blueprint
    funcs = ctx._after_request_functions
    if bp is not None and bp in self.after_request_funcs:
        funcs = chain(funcs, reversed(self.after_request_funcs[bp]))
    if None in self.after_request_funcs:
        funcs = chain(funcs, reversed(self.after_request_funcs[None]))
    for handler in funcs:
        response = handler(response)
    if not self.session_interface.is_null_session(ctx.session):
        self.save_session(ctx.session, response)
    return response
```

可以看到这个方法主要是处理被`after_request`装饰器装饰的函数，该函数处理完成后，返回response对象，之后在`wsgi_app`中会这么调用：`return response(environ, start_response)`,这个就表明Response对象有一个call方法，传入environ和start_response对象，下面就看看Response类吧！

BaseResponse
===

Response类和Request类类似，也是在Flask定义了一个继承于Werkzeug的Response，这个Response继承于BaseResponse，同时载入了许多Mixin，那么来看看BaseResponse类：

```
class BaseResponse(object):

    """
    基本的response类，关于response对象最重要的是它是一个常规的WSGI应用！它通过一组响应参数来初始化(headers, body, status code等)，通过environ和start_response来调用开始一个有效的WSGI响应。

    因为在真实的响应发送给服务器前，是WSGI应用自己结束处理的，这帮助debugging系统，因为它们可以在响应开始前捕捉所有的异常。

    这是一个小的示例WSGI应用，利用了response对象的优势

        rom werkzeug.wrappers import BaseResponse as Response

        def index():
            return Response('Index page')

        def application(environ, start_response):
            path = environ.get('PATH_INFO') or '/'
            if path == '/':
                response = index()
            else:
                response = Response('Not Found', status=404)
            return response(environ, start_response)

    像 BaseResponse 对象缺少许多的功能，这些功能都在mixins实现，这提供你对于response对象实际API更好的控制，因此你可以创建子类，并且添加定制化的功能。 一个完全的reponse对象 是实现了一组有用的mixins的Resposne

    将已存在的response转换成新的类型，可以使用 force_type 方法，在处理不同的response子类的对象的时候很用用，你想使用一个已知的接口来处理它。

    对于每一个request对象，都会假定text data是utf-8编码的

    response可能是任何种类的迭代类型或者字符串，如果是string的化，将会被视作是一个只有一个元素的迭代对象，那就是这个string。headers可能是一个列表或者元组或者是Headers对象

    对于mimetype或者content_type,对于大多数mimetype和content_type都是同样的，不同的只是text mimetype，如果mimetype传递的是text/开头的，response字符集参数将会添加到后面，相反的是content_type参数总是不变的添加到header里面。
    """

    charset = 'utf-8'

    default_status = 200

    default_mimetype = 'text/plain'

    # 如果设置为False response对象的属性将不会尝试消费response迭代器和将其转换成列表
    implicit_sequence_conversion = True

    # response对象应该将地址头纠正为RFC一致性吗？True为默认
    autocorrect_location_header = True

    # 如果可能的化response对象应该自动设置content-length吗？默认是True
    automatically_set_content_length = True

    def __init__(self, response=None, status=None, headers=None,
             mimetype=None, content_type=None, direct_passthrough=False):
        if isinstance(headers, Headers):
            self.headers = headers
        elif not headers:
            # what， Headers是一个dict like的接口，而且是排序的，并且可以在存储一个key多次
            self.headers = Headers()
        else:
            self.headers = Headers(headers)

        if content_type is None:
            if mimetype is None and 'content-type' not in self.headers:
                mimetype = self.default_mimetype
            if mimetype is not None:
                # 这里就设置了，charset 将其加在 mimetype后面
                mimetype = get_content_type(mimetype, self.charset)
            content_type = mimetype
        # 如果直接传递content_type，就不会有改变
        if content_type is not None:
            self.headers['Content-Type'] = content_type
        if status is None:
            status = self.default_status
        if isinstance(status, integer_types):
            self.status_code = status
        else:
            self.status = status

        self.direct_passthrough = direct_passthrough
        self._on_close = []

        # we set the response after the headers so that if a class changes
        # the charset attribute, the data is set in the correct charset.
        if response is None:
            self.response = []
        elif isinstance(response, (text_type, bytes, bytearray)):
            self.set_data(response)
        else:
            self.response = response

    def set_data(self, value):
        """将新的字符串作为response，要么是unicode要么是bytestring，如果unicode传递进来，那么会自动则将其编码为utf-8
        """
        # if an unicode string is set, it's encoded directly so that we
        # can set the content length
        if isinstance(value, text_type):
            value = value.encode(self.charset)
        else:
            value = bytes(value)
        self.response = [value]
        if self.automatically_set_content_length:
            self.headers['Content-Length'] = str(len(value))
```

这是Response类初始化部分，其接受一个str, unicode，如果是一个wsgi application会使用`force_type`来转换，下面来看看`force_type`这个方法

```
@classmethod
def force_type(cls, response, environ=None):
    '''
    强行将一个WSGI response转换成当前类型的对象。Werkzeug将会在内部很多场景使用这个BaseResponse比如异常，如果你在异常上调用get_response，你将会得到一个常规的BaseResponse对象，即使你在使用一个定制化的子类。

    这个方法会强制转换一个给定的response类型，并且会在给定environ时候转换任意的WSGI可调用对象至response对象

      转换一个Werkzeug response对象至MyResponseClass子类实例：
        response = MyResponseClass.force_type(response)

      转换任意的WSGI应用至一个response对象
      response = MyResponseClass.force_type(response, environ)

    如果你想在主分发器之后通过子类提供的功能处理response，这个方法将很有用

    它将会就地修改response对象
    '''
    if not isinstance(response, BaseResponse):
        if environ is None:
            raise TypeError('cannot convert WSGI application into '
                                'response objects without an environ')
        response = BaseResponse(*_run_wsgi_app(response, environ))
    response.__class__ = cls
    return response
```

可以看到其作用就是，如果是BaseResponse继承下来的类型，它只会将其对象的`__class__`属性改为当前类，如果是WSGI应用，则会先产生WSGI的response，然后通过这个response构建一个response，再将其`__class__`属性改成当前的类。

通过BaseResponse来生成响应
===

现在看看BaseResponse实例是如何是可调用的，调用之后发生了什么：

```
def __call__(self, environ, start_response):
    """作为WSGI 应用一样处理这个response
    """
    # 返回一个可迭代对象
    app_iter, status, headers = self.get_wsgi_response(environ)
    # 调用start_response函数，这个函数只是用来处理并且设置header的
    start_response(status, headers)
    return app_iter

def get_wsgi_response(self, environ):
    """以tuple形式返回最终的WSGI响应，元组的第一个元素应该是应用迭代器，第二个是状态，第三个是headers列表，返回的response是通过给定的环境变量来创建的，比如如果WSGI环境中请求方法是HEAD那么response将会是空的，只有头和状态码被返回
    """
    headers = self.get_wsgi_headers(environ)
    app_iter = self.get_app_iter(environ)
    return app_iter, self.status, headers.to_wsgi_list()
```

`start_response`来处理状态和headers,然后对每个生成的应用迭代结果进行迭代，然后发送给网络，当然会先发送`start_response`整理好的`headers_set`，headers发送完毕后就会设置`headers_sent`，状态和headers发送完毕后，就开始发送body数据，也就是`app_iter`中的每一项

下面是`get_app_iter`和`get_wsgi_headers`的源码：

```
def get_app_iter(self, environ):
    """通过给定的环境信息返回应用迭代数据，视请求方法和当前的状态码来决定返回值是空的还是其他

    如果请求方法是HEAD的化，或者状态码在HTTP指定的需要空的返回状态码之间，就会返回一个空的迭代对象
    """
    status = self.status_code
    if environ['REQUEST_METHOD'] == 'HEAD' or \
       100 <= status < 200 or status in (204, 304):
        iterable = ()
    # 如果这个选项为True，那么就不会调用iter_encoded
    elif self.direct_passthrough:
        if __debug__:
            _warn_if_string(self.response)
        return self.response
    else:
        iterable = self.iter_encoded()
    return ClosingIterator(iterable, self.close)

# self.iter_encoded本质是将每一个迭代项编码
def _iter_encoded(iterable, charset):
    for item in iterable:
        if isinstance(item, text_type):
            yield item.encode(charset)
        else:
            yield item
```

下面是`get_wsgi_headers`:

```
def get_wsgi_headers(self, environ):
    """这将会被自动调用，在response开始之前，并且会返回根据当前环境修改后的headers，返回response的一个修改后的副本

    比如location头将会和环境中的根URL组合起来，并且content_length将会在特定的状态码下置为0
    """
    headers = Headers(self.headers)
    location = None
    content_location = None
    content_length = None
    status = self.status_code

    # iterate over the headers to find all values in one go.  Because
    # get_wsgi_headers is used each response that gives us a tiny
    # speedup.
    for key, value in headers:
        ikey = key.lower()
        if ikey == u'location':
            location = value
        elif ikey == u'content-location':
            content_location = value
        elif ikey == u'content-length':
            content_length = value

    # make sure the location header is an absolute URL
    if location is not None:
        old_location = location
        if isinstance(location, text_type):
            # Safe conversion is necessary here as we might redirect
            # to a broken URI scheme (for instance itms-services).
            location = iri_to_uri(location, safe_conversion=True)

        if self.autocorrect_location_header:
            current_url = get_current_url(environ, root_only=True)
            if isinstance(current_url, text_type):
                current_url = iri_to_uri(current_url)
            location = url_join(current_url, location)
        if location != old_location:
            headers['Location'] = location

    # make sure the content location is a URL
    if content_location is not None and \
       isinstance(content_location, text_type):
        headers['Content-Location'] = iri_to_uri(content_location)

    # remove entity headers and set content length to zero if needed.
    # Also update content_length accordingly so that the automatic
    # content length detection does not trigger in the following
    # code.
    if 100 <= status < 200 or status == 204:
        headers['Content-Length'] = content_length = u'0'
    elif status == 304:
        remove_entity_headers(headers)

    # if we can determine the content length automatically, we
    # should try to do that.  But only if this does not involve
    # flattening the iterator or encoding of unicode strings in
    # the response.  We however should not do that if we have a 304
    # response.
    if self.automatically_set_content_length and \
       self.is_sequence and content_length is None and status != 304:
        try:
            content_length = sum(len(to_bytes(x, 'ascii'))
                                 for x in self.response)
        except UnicodeError:
            # aha, something non-bytestringy in there, too bad, we
            # can't safely figure out the length of the response.
            pass
        else:
            headers['Content-Length'] = str(content_length)

    return headers
```

这一段内容，看看就好，总之理解过程和原理比较重要，至于细节的东西，到要用要改的时候再详细看吧。

小结
===

到此完成了Flask中响应部分的解析，响应的过程大体如下：首先视图函数完成请求处理后，会返回一些东西，可能返回的是字符串，Response对象，tuple（包含了响应，status，headers）等，返回的这些东西都会通过`self.make_response`方法处理，处理后就成了标准的Response对象了，之后会处理被`after_request`装饰器装饰的函数，这些函数可能会修改返回的Response对象，当然不论如何，修改后都还是Response对象。这个Response对象还是可调用的，这样就可以这样调用`response(environ, start_response)`，来生成响应，这过程会生成headers, status和可迭代的应用数据，前两者会被`start_response`调用，准备发送headers和status数据，后者会在之后发送。

自此就完成了Flask中的处理。

后记
===

`make_response(*args)`
---

在看httpbin源码的时候看到用这个函数自定义了一个response对象，然后再返回，觉得很有用。于是看源码！

首先函数文档里是这么说的：

有时候在视图函数里添加额外的headers是很有必要的。因为视图函数没必要返回一个response对象，但是它可以返回一个值让Flask帮它转换成一个response对象。这样如果要添加headers就会变得棘手。调用这个函数你将得到一个response对象，这样你就可以往上面加headers了。

像这样的视图函数

```
def index():
    return render_template('index.html', foo=42)
```

你想在上面加一个新的header，可以这样做：

```
def index():
    response = make_response(render_template('index.html', foo=42))
    response.headers['X-Parachutes'] = 'parachutes are cool'
    return response
```

这个函数几乎接收你可以return的任何参数，这是一个产生404error的例子。`response = make_response(render_template('not_found.html'), 404)`

这个函数的其他用法是强制将一个视图函数的返回值变成一个response对象，使用视图的装饰器会很有用。

```
response = make_response(view_function())
response.headers['X-Parachutes'] = 'parachutes are cool'
```

本质上，这个函数做了以下事情：

- 如果什么参数也不传，那么它会创建一个新的response参数
- 如果值传递一个参数，将会调用`flask.Flask.make_response`
- 如果多于一个参数被传递进来，参数将会被当成元组传递给`flask.Flask.make_response`

下来看下源码，源码很简单，也就是上面的本质上说的事情。

```
if not args:
    # Response对象
    return current_app.response_class()
if len(args) == 1:
    # 从元组中取出来，然后当成一个参数传递给make_response
    args = args[0]
return current_app.make_response(args)
```

其中，`make_response`就是上面讲到的那个方法。

下面看一个简单的用法，取自httpbin：

```
@app.route('/robots.txt')
def view_robots_page():
    """Simple Html Page"""

    response = make_response()
    response.data = ROBOT_TXT
    response.content_type = "text/plain"
    return response
```