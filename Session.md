首先看一下session的用法：

```
@app.route('/login', methods=['GET', 'POST'])
def login():
    error = None
    if request.method == 'POST':
        if request.form['username'] != app.config['USERNAME']:
            error = 'Invalid username'
        elif request.form['password'] != app.config['PASSWORD']:
            error = 'Invalid password'
        else:
            session['logged_in'] = True
            flash('You were logged in')
            return redirect(url_for('show_entries'))
    return render_template('login.html', error=error)
```

点击session可以看到session是一个代理对象，`session = LocalProxy(partial(_lookup_req_object, 'session'))`，session是一个请求上下文对象中的属性，在请求上下文RequestContext中，当上下文进行push的时候，session就会产生，要么是一个`null_session`要么是一个真正的session：

```
def push(self):
    self.session = self.app.open_session(self.request)
    if self.session is None:
        self.session = self.app.make_null_session()
```

NullSession
===

首先先看看如果请求上下文中的session为None的时候，`make_null_session`是如何工作的：

```
def make_null_session(self):
    """创建一个缺失session的实例，我们建议替换session_interface而不是重写这个方法
    """
    return self.session_interface.make_null_session(self)
```

Flask默认是`SecureCookieSessionInterface`，这个类是继承于`SessionInterface`，`make_null_session`方法是定义在`SessionInterface`中的：

```
def make_null_session(self, app):
    """如果真正的session支持可能因为配置错误不能被载入的话，就会创建一个null session对象。这个主要是为了帮助用户体验，因为null session支持查询，但是如果要设置的话，就会出现错误
    """
    return self.null_session_class()
```

其中`null_session_class`是定义在类中的类变量，其值是`NullSession`

```
class NullSession(SecureCookieSession):
    """用来产生优雅的错误信息，如果sessions不可用的时候，可以允许只读，但是不能设置
    """

    def _fail(self, *args, **kwargs):
        raise RuntimeError('The session is unavailable because no secret '
                           'key was set.  Set the secret_key on the '
                           'application to something unique and secret.')
    __setitem__ = __delitem__ = clear = pop = popitem = \
        update = setdefault = _fail
    del _fail
```

在`self.session`为None的时候，将会创建一个NULLSession对象，但是什么时候`self.session`为None呢？

`open_session`
===

在请求上下文对象中首先调用的是Flask类中的`open_session`方法：

```
def open_session(self, request):
    """创建一个或者打开一个新的session，默认实现是在一个签名的cookie中存储了所有的session数据。这需要设置secret_key，我们推荐替换session_interface类，而不是重写这个方法
    """
    return self.session_interface.open_session(self, request)
```

同样，这个`session_interface`类默认是`SecureCookieSessionInterface`:

```
class SecureCookieSessionInterface(SessionInterface):
    """默认的session接口，在签名的cookies中存储session数据，通过itsdangerous模块
    """
    #: 盐应该在secret_key上面使用基于签名cookie的session
    salt = 'cookie-session'
    #: 生成签名的默认hash函数，默认是sha1
    digest_method = staticmethod(hashlib.sha1)
    #: itsdangerous支持的键值派生名字，默认是hmac
    key_derivation = 'hmac'
    #: 载荷的python的序列化工具，默认是紧凑的json派生序列化器，支持一些Python类型比如datetime对象和tuples
    serializer = session_json_serializer
    # 基于签名cookies的基础类
    session_class = SecureCookieSession

    def get_signing_serializer(self, app):
        # 如果没有app的secret_key，就会返回None
        if not app.secret_key:
            return None
        signer_kwargs = dict(
            key_derivation=self.key_derivation,
            digest_method=self.digest_method
        )
        return URLSafeTimedSerializer(app.secret_key, salt=self.salt,
                                      serializer=self.serializer,
                                      signer_kwargs=signer_kwargs)

    def open_session(self, app, request):
        s = self.get_signing_serializer(app)
        if s is None:
            return None
        # 在配置中使用SESSION_COOKIE_NAME来配置，默认是session
        # 从request.cookies（一个字典）获取 'session' 字段对应的值，这个值就是类似于id的作用
        # 如果没有值，则只会给一个空的session字典
        val = request.cookies.get(app.session_cookie_name)
        if not val:
            # 相当于只会给一个暂时的session，session其实是一个CallbackDict
            return self.session_class()
        # 如果有值，则会给定一个定义了id的session字典
        # 使用PERMANENT_SESSION_LIFETIME来配置，默认是timedelta(days=31)
        max_age = total_seconds(app.permanent_session_lifetime)
        try:
            data = s.loads(val, max_age=max_age)
            # 相当于用
            return self.session_class(data)
        except BadSignature:
            return self.session_class()

    def save_session(self, app, session, response):
        domain = self.get_cookie_domain(app)
        path = self.get_cookie_path(app)

        # Delete case.  If there is no session we bail early.
        # If the session was modified to be empty we remove the
        # whole cookie.
        if not session:
            if session.modified:
                response.delete_cookie(app.session_cookie_name,
                                       domain=domain, path=path)
            return

        # Modification case.  There are upsides and downsides to
        # emitting a set-cookie header each request.  The behavior
        # is controlled by the :meth:`should_set_cookie` method
        # which performs a quick check to figure out if the cookie
        # should be set or not.  This is controlled by the
        # SESSION_REFRESH_EACH_REQUEST config flag as well as
        # the permanent flag on the session itself.
        if not self.should_set_cookie(app, session):
            return

        httponly = self.get_cookie_httponly(app)
        secure = self.get_cookie_secure(app)
        expires = self.get_expiration_time(app, session)
        val = self.get_signing_serializer(app).dumps(dict(session))
        response.set_cookie(app.session_cookie_name, val,
                            expires=expires, httponly=httponly,
                            domain=domain, path=path, secure=secure)
```

可以看出来，要使用会话，你需要设置一个密钥，没有密钥就会产生一个NULLSession。在请求来的时候如果携带了cookie，那么解析session字段，将其解析出来，里面包含许多值，然后将其创建为一个session，否则就在创建一个新session。然后在处理返回的时候`save_session`，一个请求的产生和关闭过程就是一个session的过程。

在`save_session`的时候，会给response里面的头设置一个`Set-Cookie`，里面包含`key, value, max_age, expires, path, domain, secure, httponly, self.charset`,使用`URLSafeTimedSerializer`将session序列化

cookie 是由服务器发送到浏览器的变量。每当计算机通过浏览器请求一个页面，就会发送这个 cookie。然后服务器解析这个cookie，变成session，在服务器中处理完请求之后，操作session之后，再save session，将其置为response的headers，发送给客户端.

对于一个使用cookie和session的过程，客户端发送登陆请求，携带cookie，其中含有用户名，这样服务器根据cookie创建了一个session对象，即`open_session`，然后就可以获取用户名信息，根据这个用户名信息进行处理，然后比如还要加一些其他字段，加进去之后，在请求处理完发送响应的时候把session也发送过去，即`save_session`过程。

这个是默认的基于加密cookie实现的session
