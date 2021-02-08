## 【源码剖析】tornado-memcached-sessions —— Tornado session 支持的实现（一）

tornado 里面没有 session？不，当然有~
我知道 github 上肯定有人帮我写好了~ O(∩_∩)O~

于是乎，找到下面这个项目，用 memcached 实现 tornado 的 session。光会用可不行啊，让我们看看是怎么写的~

项目地址：[tornado-memcached-sessions](https://github.com/mmejia27/tornado-memcached-sessions)


让我们先从 demo 看起....

app.py 中：
首先可以注意到，这里定义了一个新的 Application 类，继承于 tornado.web.Application, 在该类的初始化方法中，设定了应用参数 settings, 之后初始化父类和 session_manager.（这是什么？暂时不管它...）

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

从使用来看是不是非常简洁和清晰？那么，细心的你是不是发现现在的 handler 没有继承于 tornado.web.RequestHandler？带着强烈的探（zuo）索（si）精神我们打开了 base.py。天啊，好短....（噢，你想到哪里去了...）

BaseHandler 的方法只是初始化，并重写了 get_current_user 的用于用户登录验证的方法。

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

看到这里，是不是心满意足？噢，我终于理解了！。。。喂，说好的探（zuo）索（si）精神呢？关键在于 session.py 啊！你一脸茫然地回过了头....

欲知后事如何，请听下回分解。（这种文风真的好吗....）
