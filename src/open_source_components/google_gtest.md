# google gtest 快速入门
gtest 提供了一套优秀的 C++ 单元测试解决方案，简单易用，功能完善，非常适合在项目中使用以保证代码质量。

## 安装
官方传送门：[googletest](https://github.com/google/googletest)
现在官方已经把 gtest 和 gmock 一起维护，所以这个 git 仓库还包含了 gmock。

这里建议安装 gtest 1.7 release 版本（该安装方法对 1.8 不适用）：

```shell
➜  ~ wget https://github.com/google/googletest/archive/release-1.7.0.tar.gz
➜  ~ tar xf release-1.7.0.tar.gz
➜  ~ cd googletest-release-1.7.0
➜  googletest-release-1.7.0 cmake -DBUILD_SHARED_LIBS=ON .
➜  googletest-release-1.7.0 make
➜  googletest-release-1.7.0 sudo cp -a include/gtest /usr/include
➜  googletest-release-1.7.0 sudo cp -a libgtest_main.so libgtest.so /usr/lib/
```

这样就 OK 了，可以用 `sudo ldconfig -v | grep gtest` 检查，看到下面就 OK 了：
```
libgtest.so -> libgtest.so
libgtest_main.so -> libgtest_main.so
```

## 使用
官方 WIKI：[Gtest](https://github.com/google/googletest/blob/master/googletest/docs/Primer.md)

### 断言
gtest 使用一系列断言的宏来检查值是否符合预期，主要分为两类：ASSERT 和 EXPECT。区别在于 ASSERT 不通过的时候会认为是一个 fatal 的错误，退出当前函数（只是函数）。而 EXPECT 失败的话会继续运行当前函数，所以对于函数内几个失败可以同时报告出来。通常我们用 EXPECT 级别的断言就好，除非你认为当前检查点失败后函数的后续检查没有意义。

#### 基础的断言
| **Fatal assertion** | **Nonfatal assertion** | **Verifies** |
|:--------------------|:-----------------------|:-------------|
| `ASSERT_TRUE(`_condition_`)`;  | `EXPECT_TRUE(`_condition_`)`;   | _condition_ is true |
| `ASSERT_FALSE(`_condition_`)`; | `EXPECT_FALSE(`_condition_`)`;  | _condition_ is false |

#### 数值比较
| **Fatal assertion** | **Nonfatal assertion** | **Verifies** |
|:--------------------|:-----------------------|:-------------|
|`ASSERT_EQ(`_val1_`, `_val2_`);`|`EXPECT_EQ(`_val1_`, `_val2_`);`| _val1_ `==` _val2_ |
|`ASSERT_NE(`_val1_`, `_val2_`);`|`EXPECT_NE(`_val1_`, `_val2_`);`| _val1_ `!=` _val2_ |
|`ASSERT_LT(`_val1_`, `_val2_`);`|`EXPECT_LT(`_val1_`, `_val2_`);`| _val1_ `<` _val2_ |
|`ASSERT_LE(`_val1_`, `_val2_`);`|`EXPECT_LE(`_val1_`, `_val2_`);`| _val1_ `<=` _val2_ |
|`ASSERT_GT(`_val1_`, `_val2_`);`|`EXPECT_GT(`_val1_`, `_val2_`);`| _val1_ `>` _val2_ |
|`ASSERT_GE(`_val1_`, `_val2_`);`|`EXPECT_GE(`_val1_`, `_val2_`);`| _val1_ `>=` _val2_ |

#### 字符串比较
| **Fatal assertion** | **Nonfatal assertion** | **Verifies** |
|:--------------------|:-----------------------|:-------------|
| `ASSERT_STREQ(`_str1_`, `_str2_`);`    | `EXPECT_STREQ(`_str1_`, `_str_2`);`     | the two C strings have the same content |
| `ASSERT_STRNE(`_str1_`, `_str2_`);`    | `EXPECT_STRNE(`_str1_`, `_str2_`);`     | the two C strings have different content |
| `ASSERT_STRCASEEQ(`_str1_`, `_str2_`);`| `EXPECT_STRCASEEQ(`_str1_`, `_str2_`);` | the two C strings have the same content, ignoring case |
| `ASSERT_STRCASENE(`_str1_`, `_str2_`);`| `EXPECT_STRCASENE(`_str1_`, `_str2_`);` | the two C strings have different content, ignoring case |

### Samples
我们安装时下载的代码就包含了 10 个例子，可以直接在根目录下执行 make 并运行。进入 samples 文件夹，阅读每份功能代码和对应的测试文件，可以帮助我们较快入门。我们下面看一下简单的例子。

#### sample1

```c++
#include <limits.h>
#include "sample1.h"
#include "gtest/gtest.h"

// Tests factorial of negative numbers.
TEST(FactorialTest, Negative) {
  // This test is named "Negative", and belongs to the "FactorialTest"
  // test case.
  EXPECT_EQ(1, Factorial(-5));
  EXPECT_EQ(1, Factorial(-1));
  EXPECT_GT(Factorial(-10), 0);
}

...

TEST(IsPrimeTest, Positive) {
  EXPECT_FALSE(IsPrime(4));
  EXPECT_TRUE(IsPrime(5));
  EXPECT_FALSE(IsPrime(6));
  EXPECT_TRUE(IsPrime(23));
}
```
sample1 演示了简单测试用例的编写，主要使用了 TEST() 宏。这个宏的使用类似于：
```
TEST(test_case_name, test_name) {
 ... test body ...
}
```
一个 test_case_name 对应一个函数的测试用例，test_name 可以对应这个测试用例下的某个场景的测试集。这两个名字可以任意取，但应该是有意义的，而且不能包含下划线 _ 。

sample1 运行结果如下：
![sample1](https://img-blog.csdnimg.cn/img_convert/7aa9a9fbed15e470797af2410c6b7a50.png)

如果出错的话会提醒我们哪个用例错误，哪个检查点不通过，以及对应代码位置，非常棒。
![test fail](https://img-blog.csdnimg.cn/img_convert/f8bfdc8ec9a9c78f431d527cce5bbcdb.png)

#### sample3
sample3 用来演示一个测试夹具的使用。前面我们每个测试用例每个测试集间都是完全独立的，使用的数据也互不干扰。但如果我们使用的测试集需要使用一些相似的数据呢？或者有些相似的检查方法？这时就需要用到测试夹具了。
```c++
#include "sample3-inl.h"
#include "gtest/gtest.h"

// To use a test fixture, derive a class from testing::Test.
class QueueTest : public testing::Test {
 protected:  // You should make the members protected s.t. they can be
             // accessed from sub-classes.

  // virtual void SetUp() will be called before each test is run.  You
  // should define it if you need to initialize the varaibles.
  // Otherwise, this can be skipped.
  virtual void SetUp() {
    q1_.Enqueue(1);
    q2_.Enqueue(2);
    q2_.Enqueue(3);
  }

  // virtual void TearDown() will be called after each test is run.
  // You should define it if there is cleanup work to do.  Otherwise,
  // you don't have to provide it.
  //
  // virtual void TearDown() {
  // }

  // A helper function that some test uses.
  static int Double(int n) {
    return 2*n;
  }

  // A helper function for testing Queue::Map().
  void MapTester(const Queue<int> * q) {
    // Creates a new queue, where each element is twice as big as the
    // corresponding one in q.
    const Queue<int> * const new_q = q->Map(Double);

    // Verifies that the new queue has the same size as q.
    ASSERT_EQ(q->Size(), new_q->Size());

    // Verifies the relationship between the elements of the two queues.
    for ( const QueueNode<int> * n1 = q->Head(), * n2 = new_q->Head();
          n1 != NULL; n1 = n1->next(), n2 = n2->next() ) {
      EXPECT_EQ(2 * n1->element(), n2->element());
    }

    delete new_q;
  }

  // Declares the variables your tests want to use.
  Queue<int> q0_;
  Queue<int> q1_;
  Queue<int> q2_;
};

// When you have a test fixture, you define a test using TEST_F
// instead of TEST.

// Tests the default c'tor.
TEST_F(QueueTest, DefaultConstructor) {
  // You can access data in the test fixture here.
  EXPECT_EQ(0u, q0_.Size());
}

// Tests Dequeue().
TEST_F(QueueTest, Dequeue) {
  int * n = q0_.Dequeue();
  EXPECT_TRUE(n == NULL);

  n = q1_.Dequeue();
  ASSERT_TRUE(n != NULL);
  EXPECT_EQ(1, *n);
  EXPECT_EQ(0u, q1_.Size());
  delete n;

  n = q2_.Dequeue();
  ASSERT_TRUE(n != NULL);
  EXPECT_EQ(2, *n);
  EXPECT_EQ(1u, q2_.Size());
  delete n;
}

// Tests the Queue::Map() function.
TEST_F(QueueTest, Map) {
  MapTester(&q0_);
  MapTester(&q1_);
  MapTester(&q2_);
}
```
可以看到我们首先需要从 testing::Test 来派生一个自己的测试类QueueTest，在这个类里你可以定义一些必要的成员变量或者辅助函数，还可以定义 SetUp 和 TearDown 两个虚函数，来指定每个测试集运行前和运行后应该做什么。后面测试用例的每个测试集应该使用 TEST_F 宏，第一个参数是我们定义的类名，第二个是测试集的名称。

对于每个 TEST_F 函数，对应的执行过程如下：
1. 创建测试夹具类（也就是说每个 TEST_F 都有一个运行时创建的夹具）。
2. 用 SetUp 函数初始化。
3. 运行测试集。
4. 调用 TearDown 进行清理。
5. delete 掉测试夹具。

#### 其他
gtest 还提供了其他更灵活也更复杂的测试方法，可以参考 sample5 之后的例子。这里限于篇幅就不介绍了，而且就我而言即使在生产环境也不需要用到这么复杂的测试方法。

### The End
最后的最后，希望大家把 gtest 用起来，单元测试对代码质量的保证作用真是非常大~
