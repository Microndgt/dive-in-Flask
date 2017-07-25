Contents
===

- [使用蓝图](https://github.com/Microndgt/dive-in-Flask/blob/master/Blueprint.md#%E4%BD%BF%E7%94%A8%E8%93%9D%E5%9B%BE)
- [`flask.blueprints.Blueprint`](https://github.com/Microndgt/dive-in-Flask/blob/master/Blueprint.md#flaskblueprintsblueprint)
- [`register_blueprint`](https://github.com/Microndgt/dive-in-Flask/blob/master/Blueprint.md#register_blueprint)

Contents Created by [Toggle](https://github.com/Microndgt/toggle)

使用蓝图
===

蓝图的作用是更好的组织代码，部署蓝图以后，你会发现整个程序结构非常清晰，模块之间相互不影响。蓝图对restful api的最明显效果就是版本控制。

对于版本控制，可以创建一个蓝图为`api_v1`，将相关路由注册在`api_v1`下，这样如果以后由api的变化，可以将其修改后注册在`api_v2`里，这样两者不会互相影响，可以同时正常工作。具体做法如下：

```
from flask import Blueprint
# 前面是蓝图名字，后面是导入的名字
be_apiv1 = Blueprint('api_v1', __name__)
# 将视图函数导入进来，然后注册蓝图到应用上,导入python文件，其实就是运行导入的那个文件
from backend.api_v1 import calc, data
# 可以给这个蓝图增加默认url前缀
app.register_blueprint(be_apiv1, url_prefix='/api_v1')
```

另外要处理的是目录结构，处理之后就十分清晰了。那么Blueprint是如何工作的，为什么要导入视图函数的模块，如何注册路由到应用上呢？

`flask.blueprints.Blueprint`
===

实例化一个蓝图，使用Blueprint，其初始化代码如下：

```
class Blueprint(_PackageBoundObject):
    """表现一个蓝图，一个蓝图是记录将会之后被flask.blueprints.BlueprintSetupState类调用的函数，用来在主应用上注册函数或者其它东西

    .. versionadded:: 0.7
    """

    warn_on_modifications = False
    _got_registered_once = False

    def __init__(self, name, import_name, static_folder=None,
                 static_url_path=None, template_folder=None,
                 url_prefix=None, subdomain=None, url_defaults=None,
                 root_path=None):
        _PackageBoundObject.__init__(self, import_name, template_folder,
                                     root_path=root_path)
        self.name = name
        self.url_prefix = url_prefix
        self.subdomain = subdomain
        self.static_folder = static_folder
        self.static_url_path = static_url_path
        self.deferred_functions = []
        if url_defaults is None:
            url_defaults = {}
        self.url_values_defaults = url_defaults
```

可以看到初始化函数无非是绑定了一些参数，其中`_PackageBoundObject.__init__(self, import_name, template_folder, root_path=root_path)`是不太一样的，代码如下：

```
class _PackageBoundObject(object):

    def __init__(self, import_name, template_folder=None, root_path=None):
        #: The name of the package or module.  Do not change this once
        #: it was set by the constructor.
        self.import_name = import_name

        #: location of the templates.  ``None`` if templates should not be
        #: exposed.
        self.template_folder = template_folder

        if root_path is None:
            root_path = get_root_path(self.import_name)

        #: Where is the app root located?
        self.root_path = root_path

        self._static_folder = None
        self._static_url_path = None

    def _get_static_folder(self):
    if self._static_folder is not None:
        return os.path.join(self.root_path, self._static_folder)
    def _set_static_folder(self, value):
        self._static_folder = value
    static_folder = property(_get_static_folder, _set_static_folder, doc='''
        The absolute path to the configured static folder.
        ''')
    # 这里将static_folder声明为属性后，删除这两个函数，也就是说这两个方法对象在类中删除
    # 但是property仍然引用，因此还可以继续使用
    del _get_static_folder, _set_static_folder

    def _get_static_url_path(self):
        if self._static_url_path is not None:
            return self._static_url_path
        if self.static_folder is not None:
            return '/' + os.path.basename(self.static_folder)
    def _set_static_url_path(self, value):
        self._static_url_path = value
    static_url_path = property(_get_static_url_path, _set_static_url_path)
    del _get_static_url_path, _set_static_url_path

    @property
    def has_static_folder(self):
        """如果package bound对象有静态文件的文件夹则返回True

        .. versionadded:: 0.5
        """
        # 会调用_get_static_folder
        return self.static_folder is not None

    @locked_cached_property
    def jinja_loader(self):
        """这个package bound对象的jinja载入器

        .. versionadded:: 0.5
        """
        if self.template_folder is not None:
            return FileSystemLoader(os.path.join(self.root_path,
                                                 self.template_folder))    
```

这里有一个有意思的用法就是`@locked_cached_property`，查看其源代码：

```
class locked_cached_property(object):
    """一个将函数转换成惰性属性的装饰器，被装饰的函数第一次被调用的时候保存结果，这样计算的结果下一次就可以直接得到。类似与Werkzeug里面的cached_property，但是有锁来保证线程安全
    这也是类作为装饰器的一个例子，因为有赋值操作，因此加锁，__get__方法会越过getattribute方法直接运行，类似于 new_func = locked_cached_property(func), new_func里面有__get__方法，所以调用new_func的时候就会调用__get__，而不是getattribute，所以返回的是值而不是可调用对象
    """

    def __init__(self, func, name=None, doc=None):
        self.__name__ = name or func.__name__
        self.__module__ = func.__module__
        self.__doc__ = doc or func.__doc__
        self.func = func
        self.lock = RLock()

    def __get__(self, obj, type=None):
        if obj is None:
            return self
        with self.lock:
            value = obj.__dict__.get(self.__name__, _missing)
            if value is _missing:
                value = self.func(obj)
                obj.__dict__[self.__name__] = value
            return value
```

接下来，在处理每一个视图函数的时候，会注册路由，如下：

```
@api_v1.route('/')
def index():
    return sth
```

这里用到的就是蓝图的route方法了，而不是app的route方法，蓝图的route代码如下：

```
def route(self, rule, **options):
    """endpoint在蓝图中是要加上蓝图的名字作为前缀
    """
    def decorator(f):
        endpoint = options.pop("endpoint", f.__name__)
        self.add_url_rule(rule, endpoint, f, **options)
        return f
    return decorator

def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    if endpoint:
        assert '.' not in endpoint, "Blueprint endpoints should not contain dots"
    # 将这个匿名函数注册进去，将来调用时候，state将作为s参数传递进去
    self.record(lambda s: s.add_url_rule(rule, endpoint, view_func, **options))

def record(self, func):
    """当蓝图注册到应用时候，每次调用的时候注册一个函数，这个函数将会使用state参数被调用，state参数是通过make_setup_state来返回的
    """
    if self._got_registered_once and self.warn_on_modifications:
        from warnings import warn
        warn(Warning('The blueprint was already registered once '
                     'but is getting modified now.  These changes '
                     'will not show up.'))
    self.deferred_functions.append(func)
```

可以看到，在`api_v1.route`装饰视图函数后，就会将这些函数注册到`self.deferred_functions`里面，每一个函数是一个匿名函数`lambda s: s.add_url_rule(rule, endpoint, view_func, **options)`，这是blueprint特定的添加rule的方法，这个特定的方法主要是将蓝图的名字加入到endpoint作为前缀。

然后在注册蓝图的时候完成rule和视图函数的关联

`register_blueprint`
===

```
@setupmethod
    def register_blueprint(self, blueprint, **options):
        """在应用中注册蓝图

        .. versionadded:: 0.7
        """
        first_registration = False
        # self.blueprint是一个字典，name就是当时实例化时候的name
        if blueprint.name in self.blueprints:
            assert self.blueprints[blueprint.name] is blueprint, \
                'A blueprint\'s name collision occurred between %r and ' \
                '%r.  Both share the same name "%s".  Blueprints that ' \
                'are created on the fly need unique names.' % \
                (blueprint, self.blueprints[blueprint.name], blueprint.name)
        else:
            self.blueprints[blueprint.name] = blueprint
            self._blueprint_order.append(blueprint)
            first_registration = True
        # 调用blueprint实例的register方法
        blueprint.register(self, options, first_registration)
```

在app中的`self.blueprints`中添加了blueprint的name和其实例的对应关系，然后将其进入了blueprint的顺序队列，之后调用blueprint的注册方法，传递了一些关键字参数

```
def register(self, app, options, first_registration=False):
    """在app上注册一个蓝图，可能被覆盖来定制化注册行为
    """
    self._got_registered_once = True
    # 暂时的创建注册状态
    state = self.make_setup_state(app, options, first_registration)
    if self.has_static_folder:
        state.add_url_rule(self.static_url_path + '/<path:filename>',
                           view_func=self.send_static_file,
                           endpoint='static')

    for deferred in self.deferred_functions:
        deferred(state)
```

其中调用了`make_setup_state`方法，其作用是创建并且返回一个BlueprintSetupState实例，来看看BlueprintSetupState定义：

```
class BlueprintSetupState(object):
    """在应用上注册一个蓝图暂时的持有对象，这个类的实例是通过flask.Blueprint.make_setup_state方法来创建的，并且之后会传递给所有的注册回调函数
    """

    def __init__(self, blueprint, app, options, first_registration):
        #: a reference to the current application
        self.app = app

        #: a reference to the blueprint that created this setup state.
        self.blueprint = blueprint

        #: a dictionary with all options that were passed to the
        #: :meth:`~flask.Flask.register_blueprint` method.
        self.options = options

        #: as blueprints can be registered multiple times with the
        #: application and not everything wants to be registered
        #: multiple times on it, this attribute can be used to figure
        #: out if the blueprint was registered in the past already.
        self.first_registration = first_registration

        subdomain = self.options.get('subdomain')
        if subdomain is None:
            subdomain = self.blueprint.subdomain

        #: The subdomain that the blueprint should be active for, ``None``
        #: otherwise.
        self.subdomain = subdomain

        url_prefix = self.options.get('url_prefix')
        if url_prefix is None:
            url_prefix = self.blueprint.url_prefix

        #: The prefix that should be used for all URLs defined on the
        #: blueprint.
        self.url_prefix = url_prefix

        #: A dictionary with URL defaults that is added to each and every
        #: URL that was defined with the blueprint.
        self.url_defaults = dict(self.blueprint.url_values_defaults)
        self.url_defaults.update(self.options.get('url_defaults', ()))

    def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
        """一个辅助方法来注册一个路由规则，可选一个视图函数 到应用上，endpoint会自动的加上蓝图的名字作为前缀
        """
        # 蓝图可以加上url_prefix 上面主要的也就是url_prefix和subdomain了
        if self.url_prefix:
            rule = self.url_prefix + rule
        options.setdefault('subdomain', self.subdomain)
        if endpoint is None:
            # 默认是view_func的__name__
            endpoint = _endpoint_from_view_func(view_func)
        defaults = self.url_defaults
        if 'defaults' in options:
            defaults = dict(defaults, **options.pop('defaults'))
        # 默认在endpoint前面加上蓝图的名字，这样就实现了不同蓝图视图函数的区分
        # add_url_rule的具体过程参照上一节内容
        self.app.add_url_rule(rule, '%s.%s' % (self.blueprint.name, endpoint),
                              view_func, defaults=defaults, **options)
```

同时蓝图上也有很多和app类似的函数，比如`before_app_first_request`,`after_request`之类等等
