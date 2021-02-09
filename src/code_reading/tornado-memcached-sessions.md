## 【源码剖析】tornado-memcached-sessions —— Tornado session 支持的实现

tornado 里面没有 session？不，当然有~ 只有官方没有，不存在 github 没有

我们看下下面这个项目，用 memcached 实现 tornado 的 session。

项目地址：[tornado-memcached-sessions](https://github.com/mmejia27/tornado-memcached-sessions)

### Demo

app.py 中：
首先可以注意到，这里定义了一个新的 Application 类，继承于 tornado.web.Application, 在该类的初始化方法中，设定了应用参数 settings, 之后初始化父类和 session_manager.

```python
class Application(tornado.web.Application):
    def __init__(self):
        settings = dict(
            # 设定 cookie_secret, 用于 secure_cookie
            cookie_secret = "e446976943b4e8442f099fed1f3fea28462d5832f483a0ed9a3d5d3859f==78d",
            # 设定 session_secret 用于生成 session_id
            session_secret = "3cdcb1f00803b6e78ab50b466a40b9977db396840c28307f428b25e2277f1bcc",
            # memcached 地址
            memcached_address = ["127.0.0.1:11211"],
            # session 过期时间
            session_timeout = 60,
            template_path = os.path.join(os.path.dirname(__file__), "templates"),
            static_path = os.path.join(os.path.dirname(__file__), "static"),
            xsrf_cookies = True,
            login_url = "/login",
        )

        handlers = [
            (r"/", MainHandler),
            (r"/login", LoginHandler)
        ]

        # 初始化父类 tornado.web.Application
        tornado.web.Application.__init__(self, handlers, **settings)
        # 初始化该类的 session_manager
        self.session_manager = session.SessionManager(settings["session_secret"], settings["memcached_address"], settings["session_timeout"])
```

在下面的 LoginHandler 中我们可以看到 session 的使用：

```python
class LoginHandler(BaseHandler):
    def get(self):
        self.render("login.html")

    def post(self):
        # 以字典的键值对形式存取
        self.session["user_name"] = self.get_argument("name")
        # 修改完要调用 session 的 save, 否则等于没有修改哦...
        self.session.save()
        self.redirect("/")
```

从使用来看，组件非常地简洁和清晰。那么，细心的你是不是发现现在的 handler 没有继承于 tornado.web.RequestHandler？我们看下 BaseHandler 的实现。

### BaseHandler

打开 base.py，可以看到 BaseHandler 的方法只是初始化，并重写了 get_current_user 的用于用户登录验证的方法。

```python
class BaseHandler(tornado.web.RequestHandler):
    def __init__(self, *argc, **argkw):
        super(BaseHandler, self).__init__(*argc, **argkw)
        # 定义 handler 的 session, 注意，根据 HTTP 特点，每次访问都会初始化一个 Session 实例哦，这对于你后面的理解很重要
        self.session = session.Session(self.application.session_manager, self)

    # 这是干嘛的？用于验证登录...请 google 关于 tornado.web.authenticated, 其实就是 tornado 提供的用户验证
    def get_current_user(self):
        return self.session.get("user_name")
```

可以看到，Handler 是实现主要是借助了 Session 类，下面来看下 session.py

### Session
首先看看 session 需要的库：
* pickle： 一个用于序列化反序列化的库（听不懂？你直接看成和 json 一样作用就行了...）
* hmac：和 hashlib 用于生成加密字符串
* uuid：用于生成一个唯一 id
* memcache ：Python 的 memcache 客户端

这里面有三个类，SessionData Session 和 SessionManager。先看最简单的 SessionData。

SessionData 用于以字典的结构存储 session 数据，继承于字典，其实只比字典多了两个成员变量：

```python
# 继承字典，因为 session 的存取类似于字典
class SessionData(dict):
    # 初始化时提供 session id 和 hmac_key
    def __init__(self, session_id, hmac_key):
        self.session_id = session_id
        self.hmac_key = hmac_key
```

然后就是真正的 Session 类了。Session 类继承于 SessionData, 注意，它还是十分像内置类型字典，只是重写了自己的初始化方法，并定义了 save 接口——用于保存修改后的 session 数据。

```python
# 继承 SessionData 类
class Session(SessionData):
    # 初始化，绑定 session_manager 和 tornado 的对应 handler
    def __init__(self, session_manager, request_handler):
        self.session_manager = session_manager
        self.request_handler = request_handler

        try:
            # 正常是获取该 session 的所有数据，以 SessionData 的形式保存
            current_session = session_manager.get(request_handler)
        except InvalidSessionException:
            # 如果是第一次访问会抛出异常，异常的时候是获取了一个空的 SessionData 对象，
            # 里面没有数据，但包含新生成的 session_id 和 hmac_key
            current_session = session_manager.get()

        # 取出 current_session 中的数据，以键值对的形式迭代存下
        for key, data in current_session.iteritems():
            self[key] = data

        # 保存下 session_id
        self.session_id = current_session.session_id
        # 以及对应的 hmac_key
        self.hmac_key = current_session.hmac_key

    # 定义 save 方法，用于 session 修改后的保存，实际调用 session_manager 的 set 方法
    def save(self):
        self.session_manager.set(self.request_handler, self)
```

__init__ 方法比较难理解，基本流程是定义自己的 session_manager 和 handler 处理对象。然后通过 session_manager 获得已有的 session 数据，用这些数据初始化一个访问的用户的 session, 如果用户是第一次访问，那么他拿到的是一个新的 SessionData 对象，因为有可能是新用户，所以这里要对 session_id 和 hmac_key 进行赋值。

而 save 方法是提供了对修改 session 数据后的保存接口，实际是调用 session_manager 的 set 方法，具体实现先不考虑。

看到这两个类，你就应该对 session 的工作有基本理解，可以从用户访问的流程来考虑。

注意 BaseHandler 这个入口，每个用户的访问都是一次 HTTP 请求。当用户第一次访问或者上一次的 session 过期了，这时用户访问时 tornado 建立了一个 handler 对象（该 handler 一定继承于 BaseHandler），并且在初始化时建立了一个 session 对象，因为是新访问，所以目前 session 里面没有数据，在之后采用 键/值 对的形式读写 session（不要忘了 Session 具有字典的所有操作），修改后通过 save 方法保存 session。

如果用户不是新访问，那么也是按照上述的流程，不过 session 初始化时把 之前的数据取出来保存在该实例中。当用户结束访问，HTTP 断开连接，handler 实例销毁，session 实例销毁（注意，是实例销毁，不是数据销毁）。

### SessionManager

最后，我们需要再看下 SessionManager 的实现。

首先是初始化，设置密钥， memcache 地址，session 超时时间。

```python
    # 初始化需要一个用于 session 加密的 secret, memcache 地址, session 的过期时间
    def __init__(self, secret, memcached_address, session_timeout):
        self.secret = secret
        self.memcached_address = memcached_address
        self.session_timeout = session_timeout
```

接着是 _fetch 方法，以 session_id  为键从 memcached 中取出数据，并用 pickle 反序列化解析数据：

```python
    # 该方法用 session_id 从 memcache 中取出数据
    def _fetch(self, session_id):
        try:
            # 连接 memcache 服务器
            mc = memcache.Client(self.memcached_address, debug=0)
            # 获取数据
            session_data = raw_data = mc.get(session_id)
            if raw_data != None:
                # 为了重新刷新 timeout
                mc.replace(session_id, raw_data, self.session_timeout, 0)
                # 反序列化
                session_data = pickle.loads(raw_data)
            # 如果拿到的数据是字典形式，才进行返回
            if type(session_data) == type({}):
                return session_data
            else:
                return {}
        except IOError:
            return {}
```

get 经过安全检查后，以 SessionData 的形式返回 memcached 的数据（调用了 _fetch）方法。


```python
    def get(self, request_handler = None):

        # 获取对应的 session_id 和 hmac_key
        if (request_handler == None):
            session_id = None
            hmac_key = None
        else:
            # session 的基础还是靠 cookie
            session_id = request_handler.get_secure_cookie("session_id")
            hmac_key = request_handler.get_secure_cookie("verification")

        # session_id 不存在的时候则生成一个新的 session_id 和 hmac_key
        if session_id == None:
            session_exists = False
            session_id = self._generate_id()
            hmac_key = self._generate_hmac(session_id)
        else:
            session_exists = True

        # 检查 hmac_key
        check_hmac = self._generate_hmac(session_id)
        # 不通过则抛出异常
        if hmac_key != check_hmac:
            raise InvalidSessionException()

        # 新建 SessionData 对象
        session = SessionData(session_id, hmac_key)

        if session_exists:
            # 通过 _fetch 方法获取 memcache 中该 session 的所有数据
            session_data = self._fetch(session_id)
            for key, data in session_data.iteritems():
                session[key] = data

        return session
```

至于 set 方法，是为了更新 memcached 的数据。

```python
    # 设置新的 session,需要设置 handler 的 cookie 和 memcache 客户端
    def set(self, request_handler, session):
        # 设置浏览器的 cookie
        request_handler.set_secure_cookie("session_id", session.session_id)
        request_handler.set_secure_cookie("verification", session.hmac_key)
        # 用 pickle 进行序列化
        session_data = pickle.dumps(dict(session.items()), pickle.HIGHEST_PROTOCOL)
        # 连接 memcache 服务器
        mc = memcache.Client(self.memcached_address, debug=0)
        # 写入 memcache
        mc.set(session.session_id, session_data, self.session_timeout, 0)
```

最后的两个函数，一个是生成 session_id，另一个用 session_id 与密钥加密后生成一个加密字符串，用于验证。

```python
    # 生成 session_id
    def _generate_id(self):
        new_id = hashlib.sha256(self.secret + str(uuid.uuid4()))
        return new_id.hexdigest()

    # 生成 hmac_key
    def _generate_hmac(self, session_id):
        return hmac.new(session_id, self.secret, hashlib.sha256).hexdigest()
```

回顾一下，我们在哪里初始化了 SessionManager 呢？可以回看开始的 demo, 我们在定义 Application 的时候就生成了一个 SessionManager，到此结束啦
