# google glog 简单使用小结
glog 是 google 的一个 c++ 开源日志系统，轻巧灵活，入门简单，而且功能也比较完善。

### 安装
以下是官方的安装方法，一句命令：
```
➜  code git clone https://github.com/google/glog.git
➜  code cd glog
➜  glog git:(master) ✗ ./configure && make && make install
```

然而我出现了 N 个错误，以下是两个编译的错误：

#### 错误记录：
Issue1: 出现 `aclocal-1.14: command not found` 和 `recipe for target 'aclocal.m4' failed` 的错误提示。
解决方法：

```
➜  glog git:(master) ✗ sudo apt-get install automake
➜  glog git:(master) ✗ sudo autoreconf -ivf
```

Issue2：之后出现另一个错误：`error: expected initializer before 'Demangle'`。
解决方法：

```
➜  code cd glog
➜  glog git:(master) ✗ mkdir build && cd build
➜  build git:(master) ✗ export CXXFLAGS="-fPIC" && cmake .. && make VERBOSE=1
➜  build git:(master) ✗ make
➜  build git:(master) ✗ sudo make install
```

### 使用
#### 菜鸟级
从一个最简单的程序开始：

```c++
#include <glog/logging.h>

int main(int argc,char* argv[])
{
    google::InitGoogleLogging(argv[0]); //初始化 glog
    LOG(INFO) << "Hello,GOOGLE!";
}
```
编译：
```
➜  glog g++ glog_test2.cpp -lglog -lgflags -lpthread -o glog_test #编译
➜  glog ./glog_test

➜  glog ll /tmp  # 默认日志在 /tmp 下
total 28K
-rw-rw-r-- 1 angryrookie angryrookie  193  7月  3 22:04 glog_test.cheng.angryrookie.log.INFO.20160703-220405.6569
lrwxrwxrwx 1 angryrookie angryrookie   57  7月  3 22:04 glog_test.INFO -> glog_test.cheng.angryrookie.log.INFO.20160703-220405.6569
➜  glog cat /tmp/glog_test.INFO
Log file created at: 2016/07/03 22:04:05
Running on machine: cheng
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0703 22:04:05.242153  6569 glog_test2.cpp:7] Hello,GLOG!
```

这里 g++ 编译的时候注意链接的库是要按顺序的（我也是才知道）。gcc 和 g++ 中库的链接顺序是从右往左进行，所以要把最基础实现的库放在最后，这样左边的 lib 就可以调用右边的 lib 中的代码。

一开始没注意顺序出现各种找不到函数，用 `nm -C` 定位了好久。而且这里我必须再链接上 gflags 才能成功编译，看网上其他人都没有。如果有大神知道为什么麻烦带带我。

这里没有指定日志文件的目录，默认会放在 /tmp 下，文件名的格式为 `<program name>.<hostname>.<user name>.log.<severity level>.<date>.<time>.<pid>`。我们再看一下代码打印出的日志信息：
```
I0703 22:04:05.242153  6569 glog_test2.cpp:7] Hello,GLOG!
```
这一行信息对应的分别是：`I+日期 时：分：秒.微秒 线程号 源文件名：行数] 信息`，这些信息对定位问题，做统计都很有效，可以根据具体项目来做分析。

#### 新手级
日志文件的位置可以通过 gflags 命令行参数 log_dir 来指定，这时注意在 main 函数开始时初始化 gflags: `google::ParseCommandLineFlags(&argc, &argv, true);`，也可以直接在代码里 指定 `FLAGS_log_dir` 的值。这里还是推荐标准的做法，即用命令行参数。

glog 的日志会根据日志严重性分级记录日志，标准的分级为：INFO,  WARNING,  ERROR,  FATAL。不同级别的日志信息会输出到不同文件，同时高级别的日志也会写入到低级别中。

再看个例子：

```c++
#include <glog/logging.h>

int main(int argc,char* argv[])
{
    google::InitGoogleLogging(argv[0]);  // 初始化 glog
    google::ParseCommandLineFlags(&argc, &argv, true);  // 初始化 gflags
    LOG(INFO) << "Hello, GOOGLE!";  // INFO 级别的日志
    LOG(ERROR) << "ERROR, GOOGLE!";  // ERROR 级别的日志
    return 0;
}
```
再次编译运行：

```
➜  glog g++ glog_test2.cpp -lglog -lgflags -lpthread -o glog_test
➜  glog mkdir log # 建一个目录放日志文件
➜  glog ./glog_test -log_dir=./log
E0703 22:59:13.380125  6867 glog_test2.cpp:8] ERROR, GOOGLE!  # 这里 ERROR 级别的日志还作为 stderr 输出了到控制台

➜  glog ll log
total 16K
-rw-rw-r-- 1 angryrookie angryrookie 196  7月  3 22:59 glog_test.cheng.angryrookie.log.ERROR.20160703-225913.6867
-rw-rw-r-- 1 angryrookie angryrookie 257  7月  3 22:59 glog_test.cheng.angryrookie.log.INFO.20160703-225913.6867
-rw-rw-r-- 1 angryrookie angryrookie 196  7月  3 22:59 glog_test.cheng.angryrookie.log.WARNING.20160703-225913.6867
lrwxrwxrwx 1 angryrookie angryrookie  58  7月  3 22:59 glog_test.ERROR -> glog_test.cheng.angryrookie.log.ERROR.20160703-225913.6867
lrwxrwxrwx 1 angryrookie angryrookie  57  7月  3 22:59 glog_test.INFO -> glog_test.cheng.angryrookie.log.INFO.20160703-225913.6867
lrwxrwxrwx 1 angryrookie angryrookie  60  7月  3 22:59 glog_test.WARNING -> glog_test.cheng.angryrookie.log.WARNING.20160703-225913.6867
```

这里可以看到，我们通过 log_dir 指定了日志存放目录。而这里虽然我们没有输出 WARNING 级别的日志信息，但却有该级别的日志。看一下分别是什么：

```
➜  glog cd log
➜  log cat glog_test.ERROR
Log file created at: 2016/07/03 22:59:13
Running on machine: cheng
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
E0703 22:59:13.380125  6867 glog_test2.cpp:8] ERROR, GOOGLE!

➜  log cat glog_test.WARNING
Log file created at: 2016/07/03 22:59:13
Running on machine: cheng
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
E0703 22:59:13.380125  6867 glog_test2.cpp:8] ERROR, GOOGLE!

➜  log cat glog_test.INFO
Log file created at: 2016/07/03 22:59:13
Running on machine: cheng
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0703 22:59:13.379570  6867 glog_test2.cpp:7] Hello, GOOGLE!
E0703 22:59:13.380125  6867 glog_test2.cpp:8] ERROR, GOOGLE!
```
WARNING 级别的日志里放了 ERROR 的输出信息，这就是我们说的高级别的日志会写入到低级别中，而且是每个低级别。

还有一个常用的日志输出语句，即自定义日志类型 VLOG。VLOG 的用法也很简单，像这样 `VLOG(50) << "MY VLOG INFO";`。这里的 50 只是自己定义的一个级别，事实上你可以定义任何一个数字，但一般越小的数字代表这条输出日志越重要。通过命令行参数 -v 50 这样子来指定 VLOG 输出的级别，-v 50 意味着只有小于 50 以下的 VLOG 才会被输出（这里不会影响 `LOG(INFO) `这些）。一般项目里越底层的库使用越大数字的 VLOG 来打印调试信息，这样可以使得日志不会被一堆底层库的运行信息淹没。

再看个例子：

```c++
#include <glog/logging.h>

int main(int argc,char* argv[])
{
    google::InitGoogleLogging(argv[0]);
    google::ParseCommandLineFlags(&argc, &argv, true);
    LOG(INFO) << "Hello, GOOGLE!";
    VLOG(100) << "VLOG INFO 100";
    VLOG(50) << "VLOG INFO 50";
    VLOG(10) << "VLOG INFO 10";
    return 0;
}
```

这里我们有三条 VLOG 信息，分别是 100 50 10 三个级别。VLOG 的信息只会出现在 INFO 级别的日志中：
```
➜  glog g++ glog_test3.cpp -lglog -lgflags -lpthread -o glog_test
➜  glog ./glog_test -log_dir=./log -v 50

➜  glog cat log/glog_test.INFO
Log file created at: 2016/07/03 23:24:32
Running on machine: cheng
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I0703 23:24:32.225793  7046 glog_test3.cpp:7] Hello, GOOGLE!
I0703 23:24:32.226384  7046 glog_test3.cpp:9] VLOG INFO 50
I0703 23:24:32.226410  7046 glog_test3.cpp:10] VLOG INFO 10
```

我们可以看到，指定了 -v 50 后，只有小于等于 50 的 VLOG 及正常 LOG(INFO) 信息被打印出来。如果你问如果不指定会怎样，回答就是一条 VLOG 信息都不会出现。


#### 进阶版
如果你以为这里还有进阶版介绍的话，我只能向你表示抱歉。我的使用只是简单级别，目前项目也没有必要更复杂的 glog 接口。

当然，我们也可以像其他博客把 glog 的功能全列出来：

* 参数设置，以命令行参数的方式设置标志参数来控制日志记录行为
* 严重性分级，根据日志严重性分级记录日志
* 可有条件地记录日志信息
* 条件中止程序。丰富的条件判定宏，可预设程序终止条件
* 异常信号处理。程序异常情况，可自定义异常处理过程
* 支持debug功能。可只用于debug模式
* 自定义日志信息
* 线程安全日志记录方式
* 系统级日志记录
*  google perror风格日志信息
* 精简日志字符串信息

这是文档传送门：[文档](http://google-glog.googlecode.com/svn/trunk/doc/glog.html)
