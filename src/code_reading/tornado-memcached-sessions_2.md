## 【源码剖析】tornado-memcached-sessions —— Tornado session 支持的实现（二）
客官您终于回头了！让我们本着探（zuo）索（si）精神把 session.py 看完吧...

首先看看需要的库：
* pickle： 一个用于序列化反序列化的库（听不懂？你直接看成和 json 一样作用就行了...）
* hmac：和 hashlib 用于生成加密字符串
* uuid：用于生成一个唯一 id
* memcache ：Python 的 memcache 客户端

这里面有三个类，SessionData Session 和 SessionManager。先看最简单的 SessionData。      SessionData 用于以字典的结构存储 session 数据，继承于字典，其实只比字典多了两个成员变量：

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

__init__ 方法比较难理解，基本流程是定义自己的 session_manager 和 handler 处理对象。然后通过 session_manager 获得已有的 session 数据，用这些数据初始化一个访问的用户的 session, 如果用户是第一次访问，那么他拿到的是一个新的 SessionData 对象，因为有可能是新用户，所以这里要对 session_id 和 hmac_key（什么鬼） 进行赋值。

而 save 方法是提供了对修改 session 数据后的保存接口，实际是调用 session_manager 的 set 方法，具体实现先不考虑。

看到这两个类，你就应该对 session 的工作有基本理解，可以从用户访问的流程来考虑。注意 BaseHandler 这个入口，每个用户的访问都是一次 HTTP 请求。当用户第一次访问或者上一次的 session 过期了，这时用户访问时 tornado 建立了一个 handler 对象（该 handler 一定继承于 BaseHandler），并且在初始化时建立了一个 session 对象，因为是新访问，所以目前 session 里面没有数据，在之后采用 键/值 对的形式读写 session（不要忘了 Session 具有字典的所有操作），修改后通过 save 方法保存 session。如果用户不是新访问，那么也是按照上述的流程，不过 session 初始化时把 之前的数据取出来保存在该实例中。当用户结束访问，HTTP 断开连接，handler 实例销毁，session 实例销毁（注意，是实例销毁，不是数据销毁）。

是不是感觉有点晕....嗯 我也晕了....理解完我们再看 SessionManager. 这个类大一点...喂...童鞋别走啊！真的只剩一点了....喂....
