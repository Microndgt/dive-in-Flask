Contents
===

- [摘要](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#摘要)
- [基本原理和目标](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#基本原理和目标)
- [系统规范综述](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#系统规范综述)
  - [字符串类型](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#字符串类型)
  - [应用/框架 端](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#应用框架-端)
  - [服务器/网关 端](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#服务器网关-端)
  - [middleware](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#middleware)
- [细节问题](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#细节问题)
  - [environ 变量](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#environ-变量)
  - [输入和错误流](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#输入和错误流)
  - [start_response()可调用对象](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#start_response可调用对象)
  - [处理`Content-Length`头](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#处理content-length头)
  - [缓存和流](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#缓存和流)
  - [处理块边界的中间件](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#处理块边界的中间件)
  - [write()对象](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#write对象)
  - [Unicode问题](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#unicode问题)
  - [错误处理](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#错误处理)
  - [HTTP 1.1 Expect/Continue](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#http-11-expectcontinue)
  - [线程支持](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#线程支持)
- [实现和应用小记](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#实现和应用小记)
  - [服务器扩展APIs](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#服务器扩展apis)
  - [应用配置](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#应用配置)
  - [重建URL](https://github.com/Microndgt/dive-in-Flask/blob/master/WSGI.md#重建url)

Created by [Toggle](https://github.com/Microndgt/toggle)

WSGI Translation V0.1. Released at 2017/07/03.

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

middleware
---

一个对象可以对于一些应用扮演服务器的角色，也可以对一些服务器扮演应用的角色。这样的middleware可以完成下面的功能：

- 在重写environ后，根据目标URL将一个请求分发到不同的应用对象
- 允许多个应用程序或框架同时在同一个进程中一起运行
- 在网络上转发请求和响应，进行负载均衡和远程处理
- 在内容上进行后加工，比如应用XSL样式

middleware的存在对于接口两端的服务器和应用都是透明的，应该不需要特别的支持。用户希望在应用中组合使用middleware只需要简单的向服务器提供middleware组件，就好像它是一个应用，然后配置middleware组件去调用应用，也就好像middleware是一个服务器。当然，这个middleware包装的应用也可能是另外一个包装了应用的中间件，以此类推，创建了一个中间件栈。

最重要的部分，中间件必须遵从WSGI两端服务器和应用的限制和需求。在一些情况，中间件的需求可能更严格相比较纯服务器和应用，这些要点将在说明中进一步阐述。

下面是一个中间件的例子，它将`text/plain`响应转换成pig Latin。一个真正的中间件可能会使用更鲁棒的方式检查Content-type，并且也应该检查内容编码。并且，这个简单的例子忽略了一个单词可能被一些边界元素分割。

```
from piglatin import piglatin
class LatinIter:
    '''转换一个可迭代输出到piglatin。另外因为第一次迭代会跳到非空的输出上，所以使用可变的bool标志来判断'''
    def __init__(self, result, transform_ok):
        # 可迭代对象是否有close属性
        if hasattr(result, 'close'):
            self.close = result.close
        # 获取可迭代对象的__next__方法
        self._next = iter(result).__next__
        # 迭代的变化标志
        self.transform_ok = transform_ok
    def __iter__(self):
        return self
    def __next__(self):
        if self.transform_ok:
            # 在此进行转换，只有当有数据的情况下
            return piglatin(self._next())   # call must be byte-safe on Py3
        else:
            return self._next

class Latinator:

    # by default, don't transform output
    transform = False
    # 中间件，接受一个应用对象
    def __init__(self, application):
        self.application = application
    # 中间件的实例对于服务器来说相当于一个应用对象
    # 对于应用，相当于一个服务器对象
    def __call__(self, environ, start_response):
        transform_ok = []
        # start_latin将会传入应用对象里（这里就相当于服务器)，包装了start_response
        # start_latin只是来处理headers的，并且包装write函数来发送body数据，然后headers的每一项
        # 转换成Latin是使用LatinIter来包装的
        def start_latin(status, response_headers, exc_info=None):
            # Reset ok flag, in case this is a repeat call
            del transform_ok[:]
            for name, value in response_headers:
                # 找到content-type然后将所有headers加入，没找到就结束了，不改变headers
                if name.lower() == 'content-type' and value == 'text/plain':
                    transform_ok.append(True)
                    # 因为要改变header对象的数据，所以长度项要去除，否则会报错
                    response_headers = [(name, value)
                        for name, value in response_headers
                            if name.lower() != 'content-length'
                    ]
                break
            # 包装在中间件内部的start_response，所以start_response返回的write函数，可以让中间件进行包装
            write = start_response(status, response_headers, exc_info)
            if transform_ok:
                def write_latin(data):
                    write(piglatin(data))   # call must be byte-safe on Py3
                return write_latin
            else:
                return write
        # 返回一个被中间件包装的可迭代对象，都是body数据，headers已经在start_latin设置好了
        return LatinIter(self.application(environ, start_latin), transform_ok)
# Run foo_app under a Latinator's control, using the example CGI gateway
from foo_app import foo_app
run_with_cgi(Latinator(foo_app))
```

细节问题
===

应用对象必须接受两个位置参数。为了说明，我们将其命名为environ和start_response，但是它们也可以取其他名字。一个服务器或者网关必须使用位置参数来调用应用对象，比如`result = application(environ, start_response)`

environ是一个字典对象，包含了CGI风格的环境变量。这个对象必须是一个Python内置的字典，并且允许应用在任何时候去修改这个字典。字典必须包含WSGI需要的变量，也可以包含特定的服务器扩展的变量，根据以下会说明的约定去命名。

start_response参数是一个可调用对象接受两个位置参数，一个可选参数。为了说明，我们命名这些参数分别为status,response_headers,exc_info，它们同样也可以取其他名字，应用必须使用位置参数来调用start_response这个可调用对象。`start_response(status, response_headers)`

status参数是一个状态字符串，格式为`999 message`这种形式，response_headers是一个列表元组，元组格式为`(header_name, header_value)`，主要是来描述HTTP的响应头。可选的exc_info参数将在下面的`The start_response() Callable`和`Error Handling`详细描述。只有当应用陷入错误并且尝试在浏览器显示一个错误信息时候用到。

start_response可调用对象必须返回一个`write(body_data)`接受一个位置参数的对象：一个将作为HTTP响应体一部分的bytestring写入。提供write对象主要是支持现有特定的框架一些必须的输出API。如果可以避免，应该避免被新的应用或者框架使用。

当被服务器调用，应用对象必须返回一个可迭代的产生零或者更多的bytestring。这可以通过很多方法实现，比如返回一个bytestring的列表，或者应用是一个生成器函数来生成bytestring，或者应用是一个类，其实例是可迭代的。不管怎么实现，应用对象必须返回一个可迭代的产生0或者更多的bytestring。

服务器或者网关必须以非缓存的形式将产生的bytestring传输到客户端(write函数),在另一个请求开始前，完成每个bytestring的传输。换句话说，应用必须实现自己的缓存，而不是依赖服务器。

服务器或者网关应该对待产生的bytestring为二进制字节的序列，特别的，应该保证行尾没有被改变。应用必须确保要写入的bytestring应该是以一种对客户端合适的格式。服务器或者网关可能应用HTTP传输编码，或者应用其他的传输来应用HTTP特性比如byte-range传输。

如果成功调用了`len(iterable)`，服务器可以依赖这个正确的结果。也就是说，如果应用返回的可迭代结果提供了一个`__len__()`方法，就必须返回一个正确的结果。

如果应用返回的可迭代对象有一个`close()`方法，服务器或者网关必须在当前请求结束完成调用这个方法，不管请求是正常完成还是因为在迭代结果或者浏览器提前断开连接时发生的应用错误提前中止了。`close()`方法需求是应用用来释放资源的。

应用返回的生成器或者其他自定义的迭代器不能假定所有的迭代器都被完全消费，可能被服务器提前关闭。

应用必须在可迭代对象生成第一个bytestring前调用start_response，这样服务器就可以在发送响应体内容前发送响应头。然而，这个调用可能会被迭代器的第一次迭代完成，所以服务器不能假定start_response在开始迭代迭代器前已经被调用。因此write函数前面才会有headers_sent的判断。

最后，服务器或者网关不能直接使用应用返回的迭代器的任何属性，除非是指定给服务器或者网关的类型的实例，比如`wsgi.file_wrapper`返回的file wrapper。一般来讲，只有指定的属性或者是迭代API允许的属性可以被接受。

environ 变量
---

environ字典需要包含在CGI说明中定义的CGI的环境变量，下列的变量必须出现，除非它们的值是空字符串，在这种情况下这个变量会被省略，除非另有说明。

- `REQUEST_METHOD` 不能为空，必须给定
- `SCRIPT_NAME` 对应应用对象的请求URL路径的初始部分，这样应用就知道它的虚拟路径。如果应用对应服务器的root，那么可能是空字符串
- `PATH_INFO` 请求URL路径，在应用中标明请求目标的虚拟路径，如果请求URL的目标是应用根路径并且没有尾斜线，那么可能是空字符串。
- `QUERY_STRING` 请求URL中?后面的部分，可能为空，也可能缺失
- `CONTENT_TYPE` HTTP请求的内容类型，可能为空，也可能缺失
- `CONTENT_LENGTH` HTTP请求中的内容长度，可能为空，也可能缺失
- `SERVER_NAME, SERVER_PORT` 和SCRIPT_NAME和PATH_INFO组合起来，这两个字符串可以用来完整的得到URL。如果HTTP_HOST出现的话，应该优先于SERVER_NAME来重建请求URL。这两个值不能为空，必须给定
- `SERVER_PROTOCOL` 客户端发送数据给服务器使用的协议版本，一般应该是`HTTP/1.0`或者是`HTTP/1.1`，会被应用用来决定如何处理HTTP请求头(这个变量应该被叫做`REQUEST_PROTOCOL`，因为它表示了在请求中使用的协议，并且没有必要服务器的响应也使用这个协议，然而，为了CGI的兼容性，现在必须使用这个名字)
- `HTTP_` 变量 对应客户端提供的请求头的变量，这些变量出现与否应该对应于请求报头中相应变量的出现与否。

一个服务器应该视图提供更多适用的其他CGI变量。另外，如果使用SSL，服务器也应该提供尽可能多的可使用的SSL环境变量，比如`HTTPS=on`和`SSL_PROTOCOL`。然而，对于一些不支持相关扩展的web服务器来说使用其他的CGI变量的应用可能是不兼容的(比如，web服务器不发布文件的话就不会提供`DOCUMENT_ROOT`或者`PATH_TRANSLATED`)。

服从WSGI的服务器应该有提供何种变量的文档以及相关详细信息。应用应该检查它们需要的参数是否存在，并且有一个如果变量不存在的时候的备用方案。

缺失的变量（比如当没有权限的时候`REMOTE_USER`缺失）应该被environ字典抛弃。并且CGI定义的变量如果出现的话必须是本地字符串。CGI变量值如果不是str的话，都是不遵守本协议的行为。

除了CGI定义的变量，environ字典也可能包含一些操作系统环境变量，并且必须包含下列的WSGI定义的变量。

- `wsgi.version`: 元组（1，0）表示WSGI版本1.0
- `wsgi.url_scheme`: 协议名，http或者https
- `wsgi.input` 读取http请求体数据的输入流（服务器可能按应用请求所需读取数据，或者可能预先读取数据并且缓存在内存或者磁盘里，或者使用其他技术来提供输入流）
- `wsgi.errors`: 输出错误流，为了以一种标准和集中的输出位置记录程序或者其他的错误。应该是文本模式流，也就是应用必须用`\n`来表示行的结尾，假定其可以被服务器转换成正确的行尾。（如果str类型是unicode，错误流应该接受并且不产生错误的记录任意的unicode。然而也被允许去替换这个字符如果不能被流的编码渲染）。对于大多数服务器，`wsgi.errors`将会是服务器的主要错误记录。可选的可能是`sys.stderr`，或者是一个日志文件等。服务器的文档应该包含如何去配置或者哪里去找到相应的记录。服务器可以根据需要可能给不同的应用提供不同的错误流。
- `wsgi.multithread` 如果可能在一个进程中其他线程中同时调用的应用对象的话，这个值应该是True，其他情况应该是False
- `wsgi.multiprocess` 如果可能在其他进程中同时调用这个app，这个值应该被置为True，否则为False
- `wsgi.run_once` 如果服务器期望这个应用只会在其进程中同时只有一个存在，这个值就应该是True，一般来讲，这个只会在CGI为基础的服务器上为True。

最后，environ字典可能包含服务器定义的变量，这些变量应该以小写单词，数字，点和下划线命名，并且应该以和定义服务器的名字相独立，比如`mod_python`可能定义一些变量名字是`mod_python.some_variable`

输入和错误流
---

服务器提供的输入和错误流必须支持下面的方法

|    方法    | 流          | 说明  |
| ------------- |:-------------:| :-----:|
|   read(size)   | 输入 |  1 |
|   readline()  | 输入 |  1,2 |
|   readlines(hint)   | 输入 |  1,3 |
|   `__iter__()`   | 输入 |  1 |
|   flush()   | 错误 |  4 |
|   write(str)   | 错误 |   |
|   writelines(seq)  | 错误 |  |

每个方法的含义都在Python标准库参考中记录着，除了下面的这些说明。

1. 服务器不需要读取客户端指定`content-length`之外的内容，应该在应用要读取超过那个点的时候模拟一个`end-of-file`的情况。应用不应该尝试读取超过`content-length`指定长度更多的内容。服务器应该允许read方法不传递参数的调用，返回客户端输入流的其余部分。服务器应该在读取一个空的输入流时候返回空的Bytestrings
2. 服务器应该支持readline()可选的size参数，但是在WSGI1.0，它们允许被忽略。在WSGI1.0，size参数没有被支持，因为可能实现起来太复杂，并且在实践中不是经常使用。但是cgi模块已经开始使用它了，所以实际中的服务器必须开始支持了！
3. hint参数对调用者和实现者来说都是可选的。应用可以选择是否提供，服务器也可以选择是否忽略。
4. 因为错误流可能不后退,服务器和网关可以自由提出立即写操作,没有缓冲。在这种情况下，flush操作可能是一个空操作。然而对于便携式应用不能假定输出是没有缓存的，或者flush操作是空操作。必须调用flush如果需要确定输出已经被写入。比如，为了最小化从不同的进程写入到相同的错误记录时数据的交织。

所有的遵从本说明的服务器都必须支持上面列出的方法。遵从本协议的应用必须不能使用input和errors对象的任意其他的方法。特别的，应用不能尝试关闭这些流，即使它们执行了close方法。

start_response()可调用对象
---

传递给应用对象的第二个参数是以下面这种形式的`start_response(status, response_headers, exc_info=None)`，对于所有的WSGI调用对象来讲，参数必须以位置参数的形式给定而不是关键字参数。这个函数用来开始HTTP响应，而且必须返回一个`write(body_data)`调用对象。

status参数是一个http状态比如"200 OK"。包含了一个状态码和信息短语，通过一个单个的空格分开，没有其他的空格和字符。该字符串不能含有其他控制字符，并且不能以换行，回车，或者它们的组合结尾。

响应头是一个列表元组，元组是以下列这种形式`(header_name, header_value)`，应该是Python列表。也就是说`type(response_headers)`是列表类型的，服务器可以根据其需要改变headers的内容。每个`header_name`都必须是一个有效的HTTP header字段名字，没有结尾冒号或者其他的标点符号。

每一个`header_value`不能包含任何的控制字符，包括回车和换行，不论是在值中间或者是在结尾。这一需求减少了服务器，或者中间的响应处理器需要检查或者修改响应头的解析的复杂度。

一般来说，服务器需要确保发送给客户端的响应头是正确的。如果应用省略了一个HTTP需要的头，服务器必须加上它。比如，`Date:`和`Server:`头必须由服务器提供。

提醒：HTTP头的名字是区分大小写的，所以确保在检查应用提供的头时候考虑到这一点。

应用或者中间件被拒绝使用`HTTP/1.1`的`hop-by-hop`特性或者头，或者`HTTP/1.0`的相关特性，或者其他可能会影响客户端到web服务器的连接的持久性的头。这些特性是真实的web服务器专属的，一个服务器应该考虑一个应用尝试传送这些header是一个严重的错误，如果提供给`start_response`应该引发错误。

服务器应该检查`start_response`调用是后出现在headers的错误，这样当应用还在运行的时候可以引发错误。

然而，`start_response`不能实际的传输响应头。相反的，它必须为服务器存储它们，只有当第一次迭代应用返回的非空值的时候传输。或者应用第一次调用write函数。也就是说，响应头必须知道它们真实的响应体数据准备好了之后才能传输。或者是在应用返回值失效的时候（唯一一种情况是响应头显式的包含了`Content-Length`为0

响应头延迟发送是为了确保缓存和异步的应用可以替换它们的最初输出为错误输出，一直到最后可能的时刻。比如，如果一个错误发生，但是响应体是在应用缓存中生成的，所以应用可能需要改变响应状态从"200 OK"到"500 Internal Error"。

`exc_info`参数，如果提供了，必须是一个`sys.exc_info()`的元组(type, value, traceback)，只有当`start_response`被一个错误处理器调用的时候，这个参数才会被应用提供。如果这个参数被提供，并且还没有HTTP的头被输出，`start_response`就应该改变现有的HTTP响应头，因此允许应用当错误发生的时候改变输出。

然后，如果`exc_info`被提供，但是HTTP headers已经被发送，`start_response`必须引发一个错误，然后使用`exc_info`重新引发一个错误:`raise exc_info[1].with_traceback(exc_info[2])`

这将重新引发应用捕获的异常，原则上应该中止应用（一旦HTTP头已经被发送应用就尝试给浏览器错误输出，这样是不安全的）。如果以`exc_info`调用`start_response`，那么应用不可以捕捉任何`start_response`引发的异常。相反的，应用应该传播这样的异常到服务器。

只有当`exc_info`参数提供的时候，应用可以调用`start_response`多次。更精确的说，在当前应用调用中，如果`start_response`已经被调用了，没有`exc_info`参数地再次调用`start_response`将是一个严重的错误。这包含了第一次调用`start_response`就出现错误的情况。

实现了`start_response`的服务器，或者中间件应该确保在函数运行期间没有到`exc_info`的引用。这是为了避免在traceback和框架之间的循环引用。最简单的方式如下：

```
def start_response(status, response_headers, exc_info=None):
    if exc_info:
        try:
        # do stuff w/exc_info here
        finally:
            exc_info = None    # Avoid circular ref.
```

处理`Content-Length`头
---

如果应用支持`Content-Length`头，那么服务器就不应该向客户端传递多于header所描述的长度的数据，应该在有足够数据已经发送的时候结束迭代，或者在应用可能超过这个点的时候引发错误。（当然，如果发送的数据不够的时候，服务器应该关闭链接并且记录或者报告错误）

如果应用不支持`Content-Length`头，服务器应该选择合适的方法处理它。最简单的方法是当响应完成的时候关闭客户端链接。

在某些情况下，服务器可能可以要么自己生成一个`Content-Length`头，以拒绝关闭客户端链接的需求。如果应用没有调用write，并且返回了一个长度是1的可迭代对象，那么服务器可以自动的使用可迭代对象生成的第一个bytestring的长度作为`Content-Length`。

并且，如果服务器和客户端都支持`HTTP/1.1` "chunked encoding"，服务器可以使用块编码，对于每一个write调用发送一个块或者可迭代对象生成的bytestring，因此为每一个块生成一个`Content-Length`。这样允许服务器保持客户端链接。服务器必须完全遵从RFC2616，否则在处理`Content-Length`缺失问题的时候使用其他策略之一。

缓存和流
---

一般来讲，应用会通过缓存它们的输出（适当的大小），然后将其全部输出会达到其最佳性能。这在现有的框架比如Zope是一种常见的实现。输出以StringIO或者类似的对象缓存起来，然后全部一起传输，和响应头一起。

对应在WSGI中的实现方法是简单的返回一个单元的迭代对象，比如一个列表，包含了响应体。这对于大量的渲染HTML页面的应用函数是一种推荐的方法，因为其文本适合在内存中。

对于大的文件，或者特别使用的HTTP流比如服务器推送，一个应用可能需要以更小的块来提供输出，以避免将大的文件全部载入到内存。也有可能是在一些响应可能非常耗时，但是如果可以提前发送已经处理的部分将非常有用。

在这种情况下，应用将通常返回一个迭代器，经常是一个生成器，来产生一块一块的输出。这些块可能被拆分的和多边界（服务器推送）一样，或者只是在耗时任务之前（比如在磁盘读取其他的block）

WSGI服务器和中间件不可以延迟任何块的传输，它们要么将所有块传输给客户端，要么保证它们即使在应用在产生下一个块也会持续传输。一个服务器或者中间件可能会提供下列三种之一的保证：

1. 在向应用返回控制权之前传输所有的块给操作系统（请求任何操作系统的缓存被清理）
2. 当应用在产生下一个块的时候使用另外一个线程确保块在持续传输。
3. （中间件）向上级服务器传递整个块

通过提供这些保证，WSGI允许应用确保传输在输出数据时候不会停止。这在服务器推送技术流中是至关重要的，在多个边界之间的数据会完全传输给客户端。（服务器推送技术中一个HTTP报文中可能包含多个数据项）

处理块边界的中间件
---

为了更好的支持异步应用和服务器，中间件不能在等待从应用的迭代对象产生的多值中将迭代分块。如果中间件在产生任何输出之前需要从应用积累更多的数据，那么它必须产生一个空的bytestring.

从其他角度考虑这个问题，一个中间件在底下的应用产生一个值的时候必须每次也产生至少一个值。如果中间件没法产生一个值，它必须产生一个空的bytestring.

这一需求确保了异步应用和服务器共同减少了需要共同运行给定数目的应用实例的线程的数量。

这一需求意味着只要底下的应用返回了一个迭代对象，那么中间件就必须返回一个可迭代的对象。并且拒绝中间件去使用write对象来传输由应用传递来的数据。中间件只能使用它们父服务器的write对象来传递应用使用中间件提供的write传递的数据。

write()对象
---

一些现存的框架以不同于WSGI的方式支持非缓存的输出。特别的，它们提供了write函数或者方法来写入一块非缓存的数据，或者是提供一个缓存的write函数和一个flush机制来清空缓存。

但是这样的API不能应用在WSGI的可调用对象返回值上面，除非使用线程或者其他特别的机制。

因此，允许这些框架继续使用这些API，WSGI包含了一个特别的write调用对象，由`start_response`对象返回。

现在如果可能，现在WSGI应用和框架不应该使用write对象。write是为了支持现有不可避免的流API。一般来说，应用应该通过返回可迭代对象产生输出，这使得web服务器可以在同一个Python线程中交错其他任务，潜在的提高了服务器整体的性能。

write对象是`start_response`返回的，它接受一个参数：要作为HTTP的响应体输出的bytestring，它必须保证传入的bytestring要么完整的传给客户端，要么在应用正在执行的时候缓存数据。

即使使用write来产生所有或者部分的响应数据，应用也必须返回一个可迭代对象。返回的可迭代对象可能是空的，但是如果它产生了一个非空bytestring，输出必须被服务器正常处理，也就是说必须发送或者进入发送队列。应用不能在它们返回的可迭代对象上使用write，因此任何从可调用对象上产生的bytestring都将在所有的bytestring传递给write之后，传输给客户端。

Unicode问题
---

HTTP没有直接的支持Unicode，同样它们的接口也没有支持。所有的编码/解码都必须由应用来处理，所有从服务器来的或者去的字符串都必须是str类型，或者bytes类型，绝对不是Unicode。使用一个unicode对象的结果是不可预知的。

所有作为状态或者响应头传递给`start_response`的字符串都必须遵从RFC2616来编码。也就是它们必须是iso-8859-1字符或者使用RFC 2047的MIME编码。

在Python平台上，使用的str或者StringType实际上都是一个基于Unicode的，所有本说明提到的strings都必须包含使用iso-8859-1编码的[代码点](http://www.cnblogs.com/runwulingsheng/p/5106078.html)表示。应用提供包含其他Unicode字符或者代码点的都是一个严重的错误。服务器不能支持包含其他Unicode字符的应用。

再次提到，本说明所有提到的strings都必须是str或者StringType，而不能是unicode或者UnicodeType。即使一个平台允许str的一个字符超过8位存储，对于本说明提到的作为string的所有值也只会使用低8位。

本文提到的所有bytestrings（从wsgi.input读取的，传递给write的或者应用产生的），值必须是Python3的bytes类型，在早期的Python中应该是str。

错误处理
---

一般来讲，应用应该试图捕获这些异常，内部错误，并且给浏览器显示有用的信息。

然而，为了显示这样的信息，应用必须不能已经传递给浏览器任何数据，否则会中止响应。WSGI因此提供了一个机制，要么允许应用发送错误数据，要么自动中止，传递`exec_info`给`start_response`。

```
try:
    # regular application code here
    status = "200 Froody"
    response_headers = [("content-type", "text/plain")]
    start_response(status, response_headers)
    return ["normal body goes here"]
except:
    # XXX should trap runtime issues like MemoryError, KeyboardInterrupt
    #     in a separate handler before this bare 'except:'...
    status = "500 Oops"
    response_headers = [("content-type", "text/plain")]
    start_response(status, response_headers, sys.exc_info())
    return ["error body goes here"]
```

如果当异常产生的时候没有输出被写入，那么`start_response`调用会正常返回，应用会返回一个错误体用来发送给浏览器。如果输出已经被发送给浏览器了，那么`start_response`将会重新引发提供的异常。异常不应该被应用捕获，这样应用将会中止，服务器然后可以捕捉这个异常并且中止响应。

服务器应该捕获并且记录任何中止应用或者返回值迭代的异常。如果应用发生错误的时候一部分响应已经被发送给浏览器了，服务器可能试图给输出添加一个错误信息，如果已经发送的headers包含了一个服务器知道如何优雅的修改的`test/*`内容类型。

一些中间件可能会提供一个额外的异常处理服务器，或者拦截和替换应用的错误信息。在这种情况下，中间件可能选择不去引发提供给`start_response`的`exc_info`，而是引发一个中间件的异常，或者在存储提供的参数后简单的返回。这将会导致应用返回它的错误体迭代对象，并且允许中间件去捕获和修改错误输出。只要应用作者如此做这项技术就可以工作

1. 开始错误响应的时候总是提供`exc_info`
2. `exc_info`提供的情况下不要捕获`start_response`引发的错误

HTTP 1.1 Expect/Continue
---

应用HTTP1.1的服务器必须提供HTTP1.1的"expect/continue"的显式支持。这可以通过以下的方式来完成：

1. 响应包含Expect的请求：立刻以"100 Continue"响应应答`100-continue`请求，并且正常处理
2. 正常处理请求，但是给应用提供一个`wsgi.input`流，在应用第一次尝试读取这个输入流的时候会发送"100 Continue"响应。读取请求接下来保持阻塞直到客户端响应。
3. 一直等待当客户端知晓服务器不支持expect/continue为止，然后向自己发送请求体。

这些行为严格上不会应用在HTTP1.0的请求上，或者没有直接指向应用对象的请求。

线程支持
---

线程支持，或者由于缺少，也是依赖服务器的。服务器可以并行运行多个请求，应该提供在单线程运行服务器的选项。这样应用或者框架不是线程安全的仍然可以使用这个服务器。

实现和应用小记
===

服务器扩展APIs
---

一些服务器作者希望暴露更多的高级API，这样应用和框架作者就可以为了特别的目的来使用。比如，基于`mod_python`的网关可能希望暴露部分Apache API作为WSGI扩展。

最简单的情况下，这只需要定义一个environ变量，比如`mod_python.some_api`。但是，大多数情况下，可能的中间件可能使得其变得很困难。比如，一个API可能返回了可以在environ找到的同样的HTTP头，所以这个API可能返回不同的数据，如果environ被中间件修改的话。

一般来讲，任何复制，替换或者忽视一些WSGI功能的一些部分会导致和中间件不兼容的风险。服务器开发者不应该假定没有人使用中间件，因为一些框架开发者特别的希望使用不同的中间件来组织他们的框架。

所以为了提供最大的兼容性，提供APIs的服务器要替换某些WSGI功能必须设计这些API，这样当被调用的时候使用他们替换的这些API。比如，一个可以读取HTTP请求头的扩展API必须需要应用传递到当前的environ，这样服务器可以验证通过API调用的HTTP头没有被中间件修改。如果扩展API不能保证其关于HTTP头的内容的environ保持一致，就必须停止给应用服务，比如引发错误，返回None而不是请求头的集合，或者对于API来将是合适的处理方法。

同样的，如果扩展API提供了可选的写入响应数据或者头的方式，应该在应用可以接受扩展服务之前，需要`start_response`传递进来。如果传递进来的对象和之前服务器提供给应用的对象不一致的话，就不能保证正确的操作所以必须拒绝提供扩展服务给应用。

这些原则也要应用在添加信息的中间件上，比如添加解析的cookie，表单变量，sessions和environ类似的对象。特别的，这样的中间件应该提供作为函数来操作environ的特性，而不是简单的将值填充到environ里。这帮助确保在任何中间件完成URL重写或者其他的environ修改之后，从environ获取的的信息已经被处理了。

这些安全扩展原则被服务器和中间件开发者遵从是很重要的，为了避免中间件开发者强制删除在从environ来的任何或者所有扩展APIs里的特性，来确保应用使用这些扩展的时候这些介入没有被忽略。

应用配置
---

这个说明没有定义服务器怎么选择应用去调用。这些或者其他的可选配置是高度服务器端的事情。服务器作者应该列出如何配置服务器去运行一个特定的应用对象，该选择何种选项等，比如线程选项。

框架作者，应该列出如何创建应用对象来包装框架的功能。用户，已经选择服务器和框架的，必须连接两者。然而，由于现在框架和服务器有同样的接口，对每一个新的服务器/框架对的选择，现在只不过是一个机械的事情，而不是一个有意义的技术工作。

最后，一些应用，框架和中间件可能希望使用environ字典来接收简单的字符串配置对象。服务器应该通过允许应用部署者在eenviron指定name-value对来支持这种方式。在最简单的情况下，这种支持可以包含操作系统提供的环境变量的副本，从`os.environ`到`environ`字典，因为部署者原则上可以在服务器外部配置这些，或者在CGI方面，可以通过在服务器配置文件来设置这些值。

应用应该尽量保持最小的变量需要，因为并不是所有服务器都可以通过简单的配置来得到。当然，最坏情况，可以通过写一个脚本来得到这些配置：

```
from the_app import application

def new_app(environ, start_response):
    environ['the_app.configval1'] = 'something'
    return application(environ, start_response)
```

但是，现在大多数存在的应用和框架可能只需要从environ来的简单的配置，来指示他们的应用或者特定的框架配置文件的位置（当然，应用应该缓存这样的配置，避免在每次调用时候重新读取)

重建URL
---

如果应用希望重建请求的完整URL，可能使用下列算法重建：

```
from urllib.parse import quote
url = environ['wsgi.url_scheme']+'://'
if environ.get('HTTP_HOST'):
    url += environ['HTTP_HOST']
else:
    url += environ['SERVER_NAME']
    if environ['wsgi.url_scheme'] == 'https':
        if environ['SERVER_PORT'] != '443':
           url += ':' + environ['SERVER_PORT']
    else:
        if environ['SERVER_PORT'] != '80':
           url += ':' + environ['SERVER_PORT']
# 转换成URL编码
url += quote(environ.get('SCRIPT_NAME', ''))
url += quote(environ.get('PATH_INFO', ''))
if environ.get('QUERY_STRING'):
    url += '?' + environ['QUERY_STRING']
```

这样的重建URL可能与客户端请求的不完全是同样的URI。服务器重写了规则，比如，可能修改了客户端原始的请求URL，将其转换成了标准形式。
