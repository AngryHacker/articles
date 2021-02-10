# google gflags 库完全使用

### 简单介绍
gflags 是 google 开源的用于处理命令行参数的项目。

### 安装编译
项目主页：[gflags](https://github.com/gflags/gflags)

```
➜  ~  git clone https://github.com/gflags/gflags.git # 下载源码
➜  ~  cd gflags
➜  gflags git:(master) ✗ mkdir build && cd build # 建立文件夹
➜  build git:(master) ✗ cmake .. # 使用 cmake 编译生成 Makefile 文件
➜  build git:(master) ✗ make # make 编译
➜  build git:(master) ✗ sudo make install # 安装库
```

这时 gflags 库会默认安装在 `/usr/local/lib/` 下，头文件放在 `/usr/local/include/gflags/` 中。

### 基础使用
我们从一个简单的需求来看 gflags 的使用，只要一分钟。假如我们有个程序，需要知道服务器的 ip 和端口，我们在程序中有默认的指定参数，同时希望可以通过命令行来指定不同的值。

实现如下：

```c++
#include <iostream>

#include <gflags/gflags.h>

/**
 *  定义命令行参数变量
 *  默认的主机地址为 127.0.0.1，变量解释为 'the server host'
 *  默认的端口为 12306，变量解释为 'the server port'
 */
DEFINE_string(host, "127.0.0.1", "the server host");
DEFINE_int32(port, 12306, "the server port");

int main(int argc, char** argv) {
    // 解析命令行参数，一般都放在 main 函数中开始位置
    gflags::ParseCommandLineFlags(&argc, &argv, true);
    // 访问参数变量，加上 FLAGS_
    std::cout << "The server host is: " << FLAGS_host
        << ", the server port is: " << FLAGS_port << std::endl;
    return 0;
}
```

OK, 写完了让我们编译运行。

```shell
➜  test g++ gflags_test.cc -o gflags_test -lgflags -lpthread # -l 链接库进行编译

➜  test ./gflags_test #不带任何参数                                                       
The server host is: 127.0.0.1, the server port is: 12306

➜  test ./gflags_test -host 10.123.78.90 #只带 host 参数
The server host is: 10.123.78.90, the server port is: 12306

➜  test ./gflags_test -port 8008 # 只带 port 参数             
The server host is: 127.0.0.1, the server port is: 8008

➜  test ./gflags_test -host 10.123.78.90 -port 8008 # host 和 port 参数
The server host is: 10.123.78.90, the server port is: 8008

➜  test ./gflags_test --host 10.123.78.90 --port 8008 # 用 -- 指定
The server host is: 10.123.78.90, the server port is: 8008

➜  test ./gflags_test --host=10.123.78.90 --port=8008 # 用 = 连接参数值
The server host is: 10.123.78.90, the server port is: 8008

➜  test ./gflags_test --help # 用 help 查看可指定的参数及参数说明
gflags_test: Warning: SetUsageMessage() never called

  Flags from /home/rookie/code/gflags/src/gflags.cc:
    .... # 略

  Flags from /home/rookie/code/gflags/src/gflags_reporting.cc:
    ..... # 略

  Flags from gflags_test.cc: #这里是我们定义的参数说明和默认值
    -host (the server host) type: string default: "127.0.0.1"
    -port (the server port) type: int32 default: 12306

```

看，我们不仅快速完成了需求，而且似乎多了很多看起来不错的特性。在上面我们使用了两种类型的参数，string 和 int32，gflags 一共支持 5 种类型的命令行参数定义：

* DEFINE_bool: 布尔类型
* DEFINE_int32: 32 位整数
* DEFINE_int64: 64 位整数
* DEFINE_uint64: 无符号 64 位整数
* DEFINE_double: 浮点类型 double
* DEFINE_string: C++ string 类型

如果你希望支持更复杂的结构，比如 list，你需要通过自己做一定的定义和解析，比如字符串按某个分隔符分割得到一个列表。

每一种类型的定义和使用都跟上面我们的例子相似，有所不同的是 bool  参数，bool 参数在命令行可以不指定值也可以指定值，假如我们定义了一个 bool 参数 debug_switch，可以在命令行这样指定：

```shell
➜  test ./gflags_test -debug_switch  # 这样就是 true
➜  test ./gflags_test -debug_switch=true # 这样也是 true
➜  test ./gflags_test -debug_switch=1 # 这样也是 true
➜  test ./gflags_test -debug_switch=false # 0 也是 false
```
所有我们定义的 gflags 变量都可以通过 FLAGS_  前缀加参数名访问，gflags 变量也可以被自由修改：
 
```c++
if (FLAGS_consider_made_up_languages)
    FLAGS_languages += ",klingon";
if (FLAGS_languages.find("finnish") != string::npos)
    HandleFinnish();
```
 
### 进阶？同样 Easy

#### 定义规范
 
 如果你想要访问在另一个文件定义的 gflags 变量呢？使用 `DECLARE_`，它的作用就相当于用 extern 声明变量。为了方便的管理变量，我们推荐在 .cc 或者 .cpp 文件中 DEFINE 变量，然后只在对应 .h 中或者单元测试中 DECLARE 变量。例如，在 foo.cc 定义了一个 gflags 变量 `DEFINE_string(name, 'bob', '')`，假如你需要在其他文件中使用该变量，那么在 foo.h 中声明 `DECLARE_string(name)`，然后在使用该变量的文件中 `include "foo.h"` 就可以。当然，这只是为了更好地管理文件关联，如果你不想遵循也是可以的。

#### 参数检查
如果你定义的 gflags 参数很重要，希望检查其值是否符合预期，那么可以定义并注册参数的值的检查函数。如果采用 static 全局变量来确保检查函数会在 main 开始时被注册，可以保证注册会在 ParseCommandLineFlags 函数之前。如果默认值检查失败，那么 ParseCommandLineFlags 将会使程序退出。如果之后使用 SetCommandLineOption() 来改变参数的值，那么检查函数也会被调用，但是如果验证失败，只会返回 false，然后参数保持原来的值，程序不会结束。看下面的程序示例：

```c++
#include <stdint.h>
#include <stdio.h>
#include <iostream>

#include <gflags/gflags.h>

// 定义对 FLAGS_port 的检查函数
static bool ValidatePort(const char* name, int32_t value) {
    if (value > 0 && value < 32768) {
        return true;
    }
    printf("Invalid value for --%s: %d\n", name, (int)value);
    return false;
}

/**
 *  设置命令行参数变量
 *  默认的主机地址为 127.0.0.1，变量解释为 'the server host'
 *  默认的端口为 12306，变量解释为 'the server port'
 */
DEFINE_string(host, "127.0.0.1", "the server host");
DEFINE_int32(port, 12306, "the server port");

// 使用全局 static 变量来注册函数，static 变量会在 main 函数开始时就调用
static const bool port_dummy = gflags::RegisterFlagValidator(&FLAGS_port, &ValidatePort);

int main(int argc, char** argv) {
    // 解析命令行参数，一般都放在 main 函数中开始位置
    gflags::ParseCommandLineFlags(&argc, &argv, true);
    std::cout << "The server host is: " << FLAGS_host
        << ", the server port is: " << FLAGS_port << std::endl;

    // 使用 SetCommandLineOption 函数对参数进行设置才会调用检查函数
    gflags::SetCommandLineOption("port", "-2");
    std::cout << "The server host is: " << FLAGS_host
        << ", the server port is: " << FLAGS_port << std::endl;
    return 0;
}
```
让我们运行一下程序，看看怎么样：
```shell
#命令行指定非法值，程序解析参数时直接退出
➜  test ./gflags_test -port -2 
Invalid value for --port: -2
ERROR: failed validation of new value '-2' for flag 'port'
# 这里参数默认值合法，但是 SetCommandLineOption 指定的值不合法，程序不退出，参数保持原来的值 
➜  test ./gflags_test        
The server host is: 127.0.0.1, the server port is: 12306
Invalid value for --port: -2
The server host is: 127.0.0.1, the server port is: 12306
```

#### 使用 flagfile
如果我们定义了很多参数，那么每次启动时都在命令行指定对应的参数显然是不合理的。gflags 库已经很好的解决了这个问题。你可以把 flag 参数和对应的值写在文件中，然后运行时使用 -flagfile 来指定对应的 flag 文件就好。文件中的参数定义格式与通过命令行指定是一样的。

例如，我们可以定义这样一个文件，文件后缀名没有关系，为了方便管理可以使用 .flags：

```
--host=10.123.14.11
--port=23333
```

然后命令行指定：

```
➜  test ./gflags_test --flagfile server.flags 
The server host is: 10.123.14.11, the server port is: 23333
```

棒！以后再也不用担心参数太多了～^_^

看到这里，是不是觉得 gflags 对你的项目很有帮助？用起来吧，释放超能力 ：）
