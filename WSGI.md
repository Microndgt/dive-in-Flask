翻译自[https://www.python.org/dev/peps/pep-3333/](https://www.python.org/dev/peps/pep-3333/)

摘要
===

这篇文档指定了一个被推荐的在Python应用程序和web服务器之间的标准接口，旨在提高web应用程序在不同的web服务器之间的可移植性。

基本原理和目标
=======

Python现在支持多种web应用框架,比如Zope,Quixote,Webware和Twisted Web等,仅举几例。多种不同的选择可能对于新的Pythoner来说是个问题,因为一般来讲,web框架的选择就会限制他们对于web服务器的选择,反之亦然

与之对比是,尽管Java有许多web应用框架,但是Java的servlet API使得基于任何Java web应用框架的应用程序可以在任何支持servlet API的web服务器上运行。

基于这样API的web服务器的可用性和广泛使用将会分离框架与web服务器的选择,不管是用Python写的(Medusa),还是嵌入式Python(`mod_python`),或者是通过网关协议来调用Python(CGI, FastCGI),这样使得用户可以选择框架和服务器都适配这样的API,并且可以使得框架和服务器开发者集中精力于他们专业的领域。

这篇文章,提出了一个在web服务器和web应用或者框架之间简单通用的接口:Python Web Server Gateway Interface(WSGI)

但是仅仅是WSGI的存在并不能在现有的服务器和框架起作用,服务器和框架作者和维护者必须实际应用WSGI。

然而,既然现在没有服务器或者框架支持WSGI,那么就需要一些奖励给应用WSGI支持的作者了,因此,WSGI必须应用起来简单,这样作者在接口的开始投入才会相当低。

因此,在服务器和框架上接口应用复杂度是WSGI接口是否实用的关键因素,也是任何设计的标准原则。

然而,对框架作者应用起来简单并不能意味着对应用开发者就简单。WSGI表现出了绝对的不提供不必要的服务接口给框架开发者,因为像response对象和cookie这种华而不实的东西只会阻碍现有的框架对这些问题(应用复杂度)的解决。此外,WSGI的目标就是促进现有的服务器和应用或者框架之间的简单连接,而不是创建一个新的web框架。

同时注意到这个目标,在目前的Python部署版本里都满足WSGI的需求。因此,基于这个说明没有提出新的标准库模块需求,WSGI的唯一需求是Python版本大于2.2.2。在以后的Python版本里由标准库提供的web服务器里支持这个接口将是个好主意。

为了简化现有的和将来的框架和服务器的部署,创建请求预处理器,返回后处理器或者其他的基于WSGI的middleware组件应该都是容易的。这些组件对于存在的服务器类似于应用程序,对于现有的应用程序表现类似于服务器。

如果middleware可以既简单又稳健,并且WSGI在服务器和框架之间广泛应用,那么这将激发出全新的Python web框架:包含松耦合的WSGI middleware组件。事实上,现有框架作者甚至可以选择重构框架的现有服务来提供这些组件,就像使用WSGI的库一样,而不是一个庞大的框架。这样使得应用开发者可以自由选择基于特定功能的组件,而不是屈从于一个框架的好处和坏处。

同时,WSGI的一个短期目标是可以在任何框架和服务器上使用。

最后,现有的WSGI版本并没有规定部署一个使用web服务器或者服务网关的应用任何特定原理。目前,通过服务器或者网关来实现定义是必要的。在足够多的服务器和框架应用基于不同的部署需求使用了WSGI,创建另外一个PEP是有意义的,这个PEP将会用来描述WSGI服务器和应用框架的部署标准。

系统规范综述
======

WSGI接口有两部分,服务器或者网关部分,应用或者框架部分。服务器部分调用框架部分提供的可调用对象。对象如何提供的细节是由服务器或者网关决定的。假定一些服务器或者网关会需要应用部署者写一个比较短的脚本去创建服务器或者网关的实例,然后向应用对象提供。其他的服务器和网关可能使用配置文件或者其他机制来指定应用对象从哪里导入或者获得。

除了纯服务器/网关和应用/框架,在WSGI协议两边都可以创建middleware组件并应用。这样的组件在服务器方面表现的像应用,在应用方面表现的像服务器,并且可以提供可扩展的API,内容转换,导航和一些其他有用的函数。

贯穿这篇文章,我们将使用一个可调用对象来指一个函数,方法,类或者一个实例实现了`__call__`方法。这将由服务器,网关或者应用根据他们的需求实现可调用对象去选择合适的实现技术。相反的,一个服务器,网关或者应用在调用一个可调用对象时候不能对任何提供给它的可调用对象产生依赖。可调用对象就是为了调用的

字符串类型
-----

一般来讲,HTTP处理bytes数据,意味着这篇说明将主要是关于处理bytes的。

然而,这些bytes的内容通常含有文本解释,在Python中,字符串是处理文本最方便的方式了。

但是在许多Python版本和实现中,字符串是Unicode,而不是bytes。因此在一个可用的API和在bytes与HTTP环境的文本的正确翻译中需要一个谨慎的平衡,特别是在不同的Python实现,不同的str类型支持可移植代码上。

WSGI因此定义了两种字符串

- Native 字符串,总是使用str类型来实现,用作request/response头和元数据
- Bytestrings 在Python3中使用bytes实现,其他版本是用str实现,用作requests和responses的body信息,比如POST/PUT输入数据和HTML页面输出。

然而不要弄混了,即使Python的str类型在底层是Unicode,但是native字符串仍然需要通过Latin-1编码转换成bytes。

简而言之,当你在这篇文章中看到string,指的是native字符串,也就是str类型,不论它在内部实现是bytes还是unicode。当你看到bytestring,在Python3中就应该是bytes类型的对象,Python2中就是str类型。

因此,即使HTTP是仅仅传输bytes的,那也有很多方便的API使用Python的默认str类型。

应用/框架 端
-------

应用对象通常是一个简单的可调用对象接受两个参数。对象不应该被误解成需要一个对象实例,一个函数,方法,类和实例,只要实现了`__call__`方法都是可以接受为一个应用对象的。应用对象必须可以被多次调用,毕竟所有的服务器/网关(除了CGI)将会做一些重复的请求。

尽管我们把它说成一个应用对象,但是也不应该被解释成应用开发者使用WSGI作为web编程的API。应用开发者仍然会继续使用现有的,高层的框架服务来开发应用。WSGI是一个为框架和服务器开发者的工具,而不是直接服务应用开发者。

下面是两个应用对象的例子,一个是函数,另一个是类

```
HELLO_WORLD = b"Hello world!\n"
# 接受两个参数一个是环境信息environ,另一个是服务器提供的start_response函数对象
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    # start_response接受两个参数,一个是响应的状态,一个是响应头
    start_response(status, response_headers)
    return [HELLO_WORLD]
class AppClass:
    '''AppClass就是应用对象,调用它将返回一个AppClass的实例,然后会返回可迭代的值
    如果要使用AppClass的实例来作为应用对象,就要实现__call__方法'''
    def __init__(self, environ, start_response):
        self.environ = environ
        self.start = start_response
    def __iter__(self):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        self.start(status, response_headers)
        yield HELLO_WORLD
# 这样就创建了一个应用
a = AppClass(environ, start_response)
# 对于实例作为应用的话就需要这样
b = AppClass()
b(environ, start_response)
```

服务器/网关 端
--------

当从HTTP客户端接受到请求时服务器或者网关就会调用应用可调用对象,直接就定向到应用了。详细说明,这里有一个简单的CGI网关,实现为接受应用对象的一个函数。这个简单的例子只有有限的错误处理,因为默认没有捕捉的异常将会定向到sys.stderr并且服务器将会记录到日志。

```
import os, sys
enc, esc = sys.getfilesystemencoding(), 'surrogateescape'
def unicode_to_wsgi(u):
    # Convert an environment variable to a WSGI "bytes-as-unicode" string
    return u.encode(enc, esc).decode('iso-8859-1')
def wsgi_to_bytes(s):
    return s.encode('iso-8859-1')
def run_with_cgi(application):
    environ = {k: unicode_to_wsgi(v) for k,v in os.environ.items()}
    environ['wsgi.input']        = sys.stdin.buffer
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True
    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'
    headers_set = []
    headers_sent = []
    def write(data):
        out = sys.stdout.buffer

        if not headers_set:
             raise AssertionError("write() before start_response()")
        elif not headers_sent:
             # Before the first output, send the stored headers
             status, response_headers = headers_sent[:] = headers_set
             out.write(wsgi_to_bytes('Status: %s\r\n' % status))
             for header in response_headers:
                 # 转换成bytes数据
                 out.write(wsgi_to_bytes('%s: %s\r\n' % header))
             out.write(wsgi_to_bytes('\r\n'))
        # 写入body数据
        out.write(data)
        out.flush()
    def start_response(status, response_headers, exc_info=None):
        # start_response首先应该设置header
        if exc_info:
            try:
                if headers_sent:
                    raise exc_info[1].with_traceback(exc_info[2])
                finally:
                    exc_info = None
        elif headers_set:
            raise AssertionError("Headers already set!")
        # start_response主要的作用就是设置headers，传递给应用的时候带入了闭包的上下文环境
        headers_set[:] = [status, response_headers]
        return write
    # 将start_response传递给应用对象
    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                # 首先会设置从应用执行start_response后传出来的headers，并且发送
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```
