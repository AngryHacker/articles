## 【源码剖析】tornado-memcached-sessions —— Tornado session 支持的实现（三
童鞋，我就知道你是个好学滴好孩子～来吧，让我们进行最后的探（zuo）索（si）！

上一次我们讲到哪里？哦。。。准备讲 SessionManager 是吧，来～一个一个函数看～

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

我们在哪里初始化了 SessionManager 呢？还记得第一篇里面的 Application 类吗？噢...快回去翻翻。

好了，童鞋们我们放学了，大家回家吧～对本篇博客有任何意见，拒绝吐槽！
