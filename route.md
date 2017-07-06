Contents
===

- [API文档](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#api文档)
  - [路由注册](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#路由注册)
  - [视图函数的属性](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#视图函数的属性)
- [Flask的路由系统](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#flask的路由系统)
  - [Flask.route](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#flaskroute)
  - [Flask.add_url_rule](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#flaskadd_url_rule)
  - [Werkzeug.routing.Rule](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#werkzeugroutingrule)
  - [Flask.create_url_adapter](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#flaskcreate_url_adapter)
  - [Werkzeug.routing.Map](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#werkzeugroutingmap)
  - [flask.ctx.RequestContex.match_request](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#flaskctxrequestcontexmatch_request)
- [总结](https://github.com/Microndgt/dive-in-Flask/blob/master/route.md#总结)

Contents Created by [Toggle](https://github.com/Microndgt/toggle)

一个 web 应用不同的路径会有不同的处理函数，路由就是根据请求的 URL 找到对应处理函数的过程。在执行查找之前，需要有一个规则列表，它存储了 url 和处理函数的对应关系。，key 是 url，value 是对应的处理函数。如果 url 都是静态的（url 路径都是实现确定的，没有变量和正则匹配），那么路由的过程就是从字典中通过 url 这个 key ，找到并返回对应的 value；对于动态路由，还需要更复杂的匹配逻辑。

本文将包含如何匹配请求中的URL到相应的视图函数以及蓝图的功能。

API文档
===

路由注册
---

[URL Route Registrations](http://flask.pocoo.org/docs/0.12/api/#url-route-registrations)里面讲到有三种方式来定义路由：

- 使用`flask.Flask.route()`装饰器
- 使用`flask.Flask.add_url_rule()`函数
- 使用底层Werkzeug路由系统，`flask.Flask.url_map`

路由中的变量部分可以通过尖括号来指定(`/user/<username>`)，URL中的变量部分默认接受任何没有斜线的字符串，并且可以指定不同的转换器使用`<converter:name>`

变量部分通过关键字参数传递给视图函数

下列的转换器是可用的：

- string
- int
- float
- path：接受斜线
- any
- uuid

可以使用`flask.Flask.url_map`自定义转换器

下面是几个例子：

```
@app.route('/')
def index():
  pass
@app.route('/<username>')
def show_user(username):
  pass
@app.route('/post/<int:post_id>')
def show_post(post_id):
  pass
```

一个重要的细节是Flask如何处理尾斜线：

- 如果rule以斜线结尾，那么请求的时候没有加尾斜线，那么会自动跳转到有尾斜线的URL上
- 如果rule没有以斜线结尾，那么请求的时候如果有尾斜线，那么就会报404错误

这个和web服务器处理静态文件的方式一致，这样可以更安全的使用相对链接目标

你可以为一个路由函数定义多个rule，但是应该是唯一的，默认也可以被指定，下面是一个接受可选页数的URL

```
@app.route('/users/', defaults={'page': 1})
@app.route('/users/page/<int:page>')
def show_users(page):
  pass
```

这个指定表明`/users/`将默认会是第一页，`/users/page/N`将会是第N页

下面是`route()`和`add_url_rule()`接收的参数，唯一的不同是通过route参数，视图函数是通过装饰器定义的而不是视图函数的参数

- rule
- endpoint: 默认是视图函数的名字
- `view_func`: endpoint产生请求，要调用的函数。也可以在`view_functions`字典中通过endpoint为键定义这个函数
- defaults: 这个rule的默认值
- subdomain: 为subdomain指定rule，如果没有指定，使用默认的subdomain
- `**options`: options会转发给底层的Rule对象，Werkzeug会处理methods选项，比如GET, POST，默认情况下只会监听GET，HEAD，OPTIONS，使用关键字参数指定

视图函数的属性
---

在内部使用中，视图函数可以有一些自定义属性，这些属性不是视图函数一般控制的。下列可选的属性或者是重写`add_url_rule()`的默认属性或者一般行为

- `__name__`: 函数名字默认被用作endpoint,如果endpoint显式提供的话这个值就被使用了，另外这个值将会使用blueprint的名字作为前缀，不能被函数自己自定义
- `methods`: 如果methods参数在url添加的时候没有指定，那么flask会查看视图函数对象本身是否有methods对象存在，如果存在，则会拉取这些信息
- `provide_automatic_options`: 这个属性被设置后，Flask会打开或者关闭HTTP的OPTIONS响应的自动完成过程。
- `required_methods`: 这个属性被设置之后，Flask会在注册URL的时候总是加上这些方法，不论这些方法是否在`route()`被显式的调用

```
def index():
  if request.method == 'OPTIONS':
    # custom options handling here
    ...
  return 'Hello World!'
index.provide_automatic_options = False
index.methods = ['GET', 'OPTIONS']

app.add_url_rule('/', view_func=index)
```

Flask的路由系统
===

Flask.route
---

route是一个装饰器函数，包装了`add_url_rule`方法，用来将url字符串和endpoint以及视图函数对象对应起来。默认参数中endpoint是None，则使用视图函数的名字作为endpoint。

```
def route(self, rule, **options):
    '''给定一个url规则注册一个视图函数，和add_url_rule一个作用
    @app.route('/')
    def index():
        return "Hello World"
    关键字参数中将传递给werkzeug.routing.Rule对象
    '''

    def decorator(f):
        endpoint = options.pop('endpoint', None)
        self.add_url_rule(rule, endpoint, f, **options)
        return f
    return decorator
```

Flask.add_url_rule
---

```
@setupmethod
def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    '''链接一个URL规则，和route装饰器一样。如果提供视图函数则将其和endpoint一起注册
    def index():
        pass
    app.add_url_rule('/', 'index', index)
    如果没有提供视图函数，那么应该链接endpoint到视图函数，app.view_functions['index'] = index
    url会对应于endpoint，然后endpoint会对应于视图函数。
    '''
    if endpoint is None:
        endpoint = _endpoint_from_view_func(view_func)
    # 默认为视图函数的名字
    options['endpoint'] = endpoint
    methods = options.pop('methods', None)
    if methods is None:
        methods = getattr(view_func, 'methods', None) or ('GET',)

    if isinstance(methods, string_types):
        raise TypeError('Allowed methods have to be iterables of strings, '
                            'for example: @app.route(..., methods=["POST"])')
    methods = set(item.upper() for item in methods)

    # Methods that should always be added
    required_methods = set(getattr(view_func, 'required_methods', ()))
    provide_automatic_options = getattr(view_func, 'provide_automatic_options', None)

    if provide_automatic_options is None:
        # 如果OPTIONS没有在要处理的方法之中，则将其OPTIONS置为自动处理
        if 'OPTIONS' not in methods:
            provide_automatic_options = True
            required_methods.add('OPTIONS')
        else:
            provide_automatic_options = False

    methods |= required_methods # 取并集

    # 构建Rule对象，传入url字符串和方法，以及相应endpoint等参数
    rule = self.url_rule_class(rule, methods=methods, **options)

    rule.provide_automatic_options = provide_automatic_options

    # url_map是一个Map()对象，Rule对象
    self.url_map.add(rule)

    if view_func is not None:
        # flask 中的view_functions字典中已经存储的endpoint
        old_func = self.view_functions.get(endpoint)
        if old_func is not None and old_func != view_func:
            raise AssertionError('View function mapping is overwriting an '
                                 'existing endpoint function: %s' % endpoint)
        # flask内部存储endpoint和视图函数之间的关系
        self.view_functions[endpoint] = view_func
```

可以看出来该函数对各种参数进行判断检查后，核心调了`self.url_rule_class`创建了一个Rule对象，需要传递参数是url字符串和请求方法以及相应的endpoint，Rule对象中存储的是url字符串和endpoint之间的关系。框架处理请求的时候会创建请求上下文，在这个时候创建一个url适配器(`create_url_adapter`)，然后匹配请求(`match_request`)，这样会匹配url到对应的endpoint，然后Flask内部保存了endpoint到视图函数的字典，由此所以即可得到相应的视图函数。

Werkzeug.routing.Rule
---

```
class RuleFactory(object):
    '''一旦你有了更多的复杂的URL处理器，使用rule工厂来避免重复的任务是很有用的。它们中一些是内置的，其它的可能是继承RuleFactory并且重写get_rules方法'''
    def get_rules(self, map):
        """子类实现并且返回可迭代的rules"""
        raise NotImplementedError()

@implements_to_string
class Rule(RuleFactory):
    '''一个Rule对象代表了一个URL类型，有一些选项可以改变Rule的工作方式，它们被传递给Rule的构造器。除了rule字符串，其它必须是关键字参数，为了使得其能正常运行当代码升级的时候。

    string: 只是普通的带有占位符的URL路径<converter(arguments):name>，转换器和参数都是可选的，默认的转换器是string
    URL以斜线结尾的都是分支URL，其它的都是叶子URL，如果strict_slashes被激活，那么以没有斜线结尾的URL匹配分支URL都会导致重定向。
    转换器在Map上定义。

    endpoint：rule的端点，可能是任何东西。函数引用，字符串，数字等。最好是用string因为端点是用来URL生成的。

    defaults: 可选的字典，为了匹配其它同样的端点但是不同的rules。如果你想有不同的URLs对应一个端点，下面是一个小技巧
    url_map = Map([
                Rule('/all/', defaults={'page': 1}, endpoint='all_entries'),
                Rule('/all/page/<int:page>', endpoint='all_entries')
            ])
    如果用户访问http://example.com/all/page/1则会重定向到http://example.com/all/

    subdomain: 当前rule的子域名，如果没有指定的话只会匹配map上的default_subdomain，如果map没有绑定到一个subdomain这个特性将会被禁用。如果你想使用user信息作为不同的subdomain，则所有的subdomain都会传递给你的应用
    url_map = Map([
                Rule('/', subdomain='<username>', endpoint='user/homepage'),
                Rule('/stats', subdomain='<username>', endpoint='user/stats')
            ])

    methods: 这个rule对应的http方法的序列。如果没有指定，所有的方法都允许。如果你想在POST和GET时有不同的端点，这将会很有用。如果方法已经被定义并且路径也匹配，但是方法没有在允许的方法内，或者是在同样端点的另一个rule方法定义内，这样就会引发MethodNotAllowed而不是NotFound。如果get方法出现但是HEAD没有出现，HEAD会自动添加进来。

    strict_slashes:

    build_only： 为True则rule不会进行匹配，而是创建一个URL对象，当你有一个不用WSGI应用处理但是在subdomain或者文件夹上的资源时

    redirect_to： 如果给定，那么应该是一个字符串或者一个可调用对象。如果是可调用对象，应该接收url适配器和URL的值（可变参数），返回一个重定向的目标，否则应该是定义URL的字符串

    def foo_with_slug(adapter, id):
                # ask the database for the slug for the old id.  this of
                # course has nothing to do with werkzeug.
                return 'foo/' + Foo.get_slug_for_id(id)

            url_map = Map([
                Rule('/foo/<slug>', endpoint='foo'),
                Rule('/some/old/url/<slug>', redirect_to='foo/<slug>'),
                Rule('/other/old/url/<int:id>', redirect_to=foo_with_slug)
            ])
    '''

    def __init__(self, string, defaults=None, subdomain=None, methods=None, build_only=False, endpoint=None, strict_slashes=None, redirect_to=None, alias=False, host=None):
        if not string.startswith('/'):
            raise ValueError('urls must start with a leading slash')
        self.rule = string
        # 不以/结尾的是叶子url
        self.is_leaf = not string.endswith('/')
        self.map = None
        self.strict_slashes = strict_slashes
        self.subdomain = subdomain  # string
        self.host = host  # string
        self.defaults = defaults  # defaults = {'page'： 1}
        self.build_only = build_only
        self.alias = alias
        if methods is None:
            self.methods = None
        else:
            self.methods = set([x.upper() for x in methods])
            if 'HEAD' not in self.methods and 'GET' in self.methods:
                self.methods.add('HEAD')
        self.endpoint = endpoint  # string, function ref or number
        self.redirect_to = redirect_to  # string or function

        if defaults:
            self.arguments = set(map(str, defaults))  # 键的集合
        else:
            self.arguments = set()
        self._trace = self._converters = self._regex = self._weights = None

    def empty(self):
        '''返回这个rule未绑定的副本，如果你想重用一个已经绑定RUL来生成一个新的map。使用get_empty_kwargs来决定提供哪些关键字参数'''
        # type(self) 或者当前实例的类型
        return type(self)(self.rule, **self.get_empty_kwargs())

    def get_empty_kwargs(self):
        '''为使用empty实例化一个副本提供关键字参数'''
        defaults = None
        if self.defaults:
            defaults = dict(self.defaults)
        return dict(defaults=defaults, subdomain=self.subdomain,
                    methods=self.methods, build_only=self.build_only,
                    endpoint=self.endpoint, strict_slashes=self.strict_slashes,
                    redirect_to=self.redirect_to, alias=self.alias,
                    host=self.host)

    def get_rules(self, map):
        '''返回当前的Rule实例(一个匹配规则)'''
        yield self

    def refresh(self):
        '''重新绑定并且刷新URL。如果你就地改变了rule对象，则调用'''
        self.bind(self.map, rebind=True)

    def bind(self, map, rebind=False):
        '''绑定url到一个map上，并且创建一个基于rule本身信息和map的默认值的正则表达式'''
        if self.map is not None and not rebind:
            raise RuntimeError('url rule %r already bound to map %r' %
                               (self, self.map))
        self.map = map
        # 所以Rule的strict_slashes会覆盖map的参数
        if self.strict_slashes is None:
            self.strict_slashes = map.strict_slashes
        if self.subdomain is None:
            self.subdomain = map.default_subdomain
        self.compile()

    def get_converter(self, variable_name, converter_name, args, kwargs):
        '''查询给定参数的转换器'''
        if converter_name not in self.map.converters:
            raise LookupError('the converter %r does not exist' % converter_name)
        return self.map.converters[converter_name](self.map, *args, **kwargs)

    def compile(self):
        '''编译正则表达式，并且存储之，为了以后实际请求url进行匹配'''
        assert self.map is not None, 'rule not bound'

        if self.map.host_matching:
            domain_rule = self.host or ''
        else:
            domain_rule = self.subdomain or ''

        self._trace = []
        self._converters = {}
        self._weights = []
        regex_parts = []

        def _build_regex(rule):
            for converter, arguments, variable in parse_rule(rule):
                # 如果有动态变量但是没有转换器，则默认为default
                # variable可能为静态url，如果converter为None
                if converter is None:
                    # 静态url, re.escape 对字符串中所有可能被解释为正则运算符的字符进行转义
                    regex_parts.append(re.escape(variable))
                    self._trace.append((False, variable))
                    for part in variable.split('/'):
                        if part:
                            self._weights.append((0, -len(part)))
                else:
                    if arguments:
                        c_args, c_kwargs = parse_converter_args(arguments)
                    else:
                        c_args = ()
                        c_kwargs = {}
                    convobj = self.get_converter(
                        variable, converter, c_args, c_kwargs)
                    regex_parts.append('(?P<%s>%s)' % (variable, convobj.regex))
                    self._converters[variable] = convobj
                    self._trace.append((True, variable))
                    self._weights.append((1, convobj.weight))
                    self.arguments.add(str(variable))

        _build_regex(domain_rule)
        regex_parts.append('\\|')
        self._trace.append((False, '|'))
        _build_regex(self.is_leaf and self.rule or self.rule.rstrip('/'))
        if not self.is_leaf:
            self._trace.append((False, '/'))

        if self.build_only:
            return
        regex = r'^%s%s$' % (
            u''.join(regex_parts),
            (not self.is_leaf or not self.strict_slashes) and
            '(?<!/)(?P<__suffix__>/?)' or ''
        )
        # (?<!...)之前的字符串内容需要不匹配表达式此啊能成功匹配，也就是说不能匹配/才可以匹配到后面的
        # re.U或者re.UNICODE把\w \W \s \S等这些元字符按照 Unicode 的标准来考虑
        self._regex = re.compile(regex, re.UNICODE)

# 返回正则表达式匹配对象，group方法，groups方法, groupdict()方法，start方法，end方法
def parse_rule(rule):
    '''解析一个rule并且作为生成器返回。每次迭代产生元组(converter,arguments,variable)的形式，
    如果转换器是None说明其是一个静态url部分，否则是动态的'''
    pos = 0
    end = len(rule)
    # 这个是一个编译后的正则表达式，后面讲到
    do_match = _rule_re.match
    used_names = set()
    # 每次匹配一个converter
    while pos < end:
        m = do_match(rule, pos)
        if m is None:
            break
        data = m.groupdict()
        if data['static']:
            yield None, None, data['static']
        variable = data['variable']
        converter = data['converter'] or 'default'
        if variable in used_names:
            raise ValueError('variable name %r used twice.' % variable)
        used_names.add(variable)
        yield converter, data['args'] or None, variable
        pos = m.end()
    # 为了处理没有转换器和动态参数的url，即static url
    if pos < end:
        # 剩余的url
        remaining = rule[pos:]
        if '>' in remaining or '<' in remaining:
            raise ValueError('malformed url rule: %r' % rule)
        yield None, None, remaining

def parse_converter_args(argstr):
    argstr += ','
    args = []
    kwargs = {}

    for item in _converter_args_re.finditer(argstr):
        value = item.group('stringval')
        if value is None:
            value = item.group('value')
        value = _pythonize(value)
        if not item.group('name'):
            args.append(value)
        else:
            name = item.group('name')
            kwargs[name] = value

    return tuple(args), kwargs
```

以下是对url的正则表达式匹配规则：

```
# ?表示匹配一个元字符0次或者1次
# ?P<name> 表示产生一个带命名的组, *表示匹配一个字符0或者无限次
#  (?P=name) 匹配前面已命名的组,使用.group(name)来访问匹配结果
# [^<]表示匹配在<出现前的字符，如果没有后面的*，则会取第一个字符，如果<之前没有字符，而且没有*则会匹配到None，有*就表示匹配0次，则会返回''
_rule_re = re.compile(r'''
    (?P<static>[^<]*)                           # 匹配在出现第一个<时候url前面的部分 *是表示匹配所有在<之前的字符
    <
    (?:   # 不分组版本，用于使用|或者后面接数量词 (?:abc){2}
        (?P<converter>[a-zA-Z_][a-zA-Z0-9_]*)   # 开头为字母或者下划线，不为数字
        (?:\((?P<args>.*?)\))?                  # converter的参数，匹配到才算匹配
        \:                                      # variable delimiter
    )?                                          # 匹配零个或者一个converter, string: 这种
    (?P<variable>[a-zA-Z_][a-zA-Z0-9_]*)        # 变量名
    >
''', re.VERBOSE)  # 表示给予更加灵活的格式将正则表达式写的更容易理解，把正则表达式写成多行，并且自动忽略空格
# 每次只匹配一个converter以及变量
```

url是通过正则表达式进行匹配的，所以大致的流程是，首先将代码中的路由函数定义的url规则编译成正则表达式对象，然后等到真正的请求来的时候，去匹配相应的url，然后导向到对应的视图函数。比如：`/`将会编译成如下的正则表达式,`'^\\|(?<!/)(?P<__suffix__>/?)$'`，下面进行匹配的时候，传递的path是`|/`，这样就会匹配到。

下面继续Route类的一些方法：

```
def match(self, path):
    '''检查给定一个path是否匹配到动态变量，path是以这种形式："subdomain|/path(method)"，并且是由map装配的，如果map在做host匹配，subdomain部分将是host。

    如果rule匹配到一个字典，转换后的值将会被返回，否则返回None
    '''
    if not self.build_only:
        # search匹配
        m = self._regex.search(path)
        if m is not None:
            groups = m.groupdict()
            # 如果一个url没有尾斜线但是strict_slashes激活了，那么引发异常去告诉map重定向到一个有尾斜线的url
            if self.strict_slashes and not self.is_leaf and not groups.pop('__suffix__'):
                raise RequestSlash()
            elif not self.strict_slashes:
                del groups['__suffix__']
            result = {}
            for name, value in iteritems(groups):
                try:
                    value = self._converters[name].to_python(value)
                except ValidationError:
                    return
                result[str(name)] = value
            if self.defaults:
                result.update(self.defaults)

            if self.alias and self.map.redirect_defaults:
                raise RequestAliasRedirect(result)

            return result
```

在匹配的时候，只有有转换器的才会有相应的匹配分组结果，然后返回匹配的result

Flask.create_url_adapter
---

```
def create_url_adapter(self, request):
    '''为每个请求创建URL适配器，在请求上下文还没有建立的时候就创建了，所以request是显式传递的。
    '''
    if request is not None:
        return self.url_map.bind_to_environ(request.environ,
                server_name=self.config['SERVER_NAME'])
    if self.config['SERVER_NAME'] is not None:
        return self.url_map.bind(
                self.config['SERVER_NAME'],
                script_name=self.config['APPLICATION_ROOT'] or '/',
                url_scheme=self.config['PREFERRED_URL_SCHEME'])
```

可以看到，其主要是调用了Map类的`bind_to_environ`和`bind方法`

Werkzeug.routing.Map
---

[官方文档](http://werkzeug.pocoo.org/docs/0.11/routing/)

Map中除了实现`bind_to_environ`和`bind方法`绑定到特定环境，还有`add`方法可以添加路由，添加好路由规则之后，使用MapAdapter对象的url.match就可以匹配路由了。正常情况下返回对应的 endpoint 名字和参数字典，可能报重定向或者 404 异常。

下面是Map的一些用法：

```
from werkzeug.routing import Map, Rule, NotFound, RequestRedirect

url_map = Map([
    Rule('/', endpoint='blog/index'),
    Rule('/<int:year>/', endpoint='blog/archive'),
    Rule('/<int:year>/<int:month>/', endpoint='blog/archive'),
    Rule('/<int:year>/<int:month>/<int:day>/', endpoint='blog/archive'),
    Rule('/<int:year>/<int:month>/<int:day>/<slug>',
         endpoint='blog/show_post'),
    Rule('/about', endpoint='blog/about_me'),
    Rule('/feeds/', endpoint='blog/feeds'),
    Rule('/feeds/<feed_name>.rss', endpoint='blog/show_feed')
])
urls = m.bind("example.com", "/")  # MapAdapter
urls.match("/", "GET")
# ('index', {})
urls.match("/downloads/42")
# ('downloads/show', {'id': 42})
```

flask.ctx.RequestContex.match_request
---

```
def match_request(self):
    """Can be overridden by a subclass to hook into the matching
    of the request.
    """
    try:
        # url_adapter是一个MapAdapter对象
        # match方法返回rule和参数
        url_rule, self.request.view_args = \
            self.url_adapter.match(return_rule=True)
        self.request.url_rule = url_rule
    except HTTPException as e:
        self.request.routing_exception = e
```

总结
===

路由系统解析大致流程总结一下：首先在处理wsgi调用的时候，创建了请求上下文对象，在创建这个的过程中，根据environ创建了url_adapter(MapAdapter对象)，绑定了相应环境。然后使用这个url_adapter来匹配，返回匹配的endpoint，和动态参数。而Flask中存储了endpoint到视图函数的字典`self.view_functions`，这样就可以处理请求时候调用相应的视图函数，进行处理了。

应用的视图函数和url rule的绑定过程主要是使用`add_url_rule`方法，来创建一个Rule对象，然后将其加入`self.map`，add方法，这样以后就可以进行匹配了。
