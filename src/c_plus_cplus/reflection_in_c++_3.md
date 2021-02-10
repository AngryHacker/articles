# C++实现反射（三）
上一篇我们用一个 Object 类，让所有需要反射的类都继承这个对象，这样虽然解决了问题，但是用起来不太方便。Object 类的存在主要为了解决保存和返回时的类型问题，如果取消这个类，我们怎么对这些反射类做统一处理呢？答案当然是模板。

1. 实现一个模板类管理类名和类构造函数的映射关系，并提供构造对象的接口，每个基类需要初始化一个这样的管理对象。
2. 提供一个对应的 static 模板函数，用来保存和返回对应的管理对象。
3. 使用模板函数和 new 操作符作为每个类的构造函数。
4. 实现一个简单的 helper 模板类提供作为注册的简单封装，并封装宏实现注册。
5. 封装一个宏实现反射类的创建。


```C++
#ifndef __BASE_H__
#define __BASE_H__
#include <string>
#include <map>
#include <iostream>

// 使用模板，每个基类单独生成一个 ClassRegister
// 好处是需要反射的类不需要去继承 Object 对象
// ClassRegister 用来管理类名->类构造函数的映射，对外提供根据类名构造对象对函数
template<typename ClassName>
class ClassRegister {
  public:
    typedef ClassName* (*Constructor)(void);
  private:
    typedef std::map<std::string, Constructor> ClassMap;
    ClassMap constructor_map_;
  public:
    // 添加新类的构造函数
    void AddConstructor(const std::string class_name, Constructor constructor) {
      typename ClassMap::iterator it = constructor_map_.find(class_name);
      if (it != constructor_map_.end()) {
        std::cout << "error!";
        return;
      }
      constructor_map_[class_name] = constructor;
    }
    // 根据类名构造对象
    ClassName* CreateObject(const std::string class_name) const {
      typename ClassMap::const_iterator it = constructor_map_.find(class_name);
      if (it == constructor_map_.end()) {
        return nullptr;
      }
      return (*(it->second))();
    }
};

// 用来保存每个基类的 ClassRegister static 对象，用于全局调用
template <typename ClassName>
ClassRegister<ClassName>& GetRegister() {
  static ClassRegister<ClassName> class_register;
  return class_register;
}

// 每个类的构造函数，返回对应的base指针
template <typename BaseClassName, typename SubClassName>
BaseClassName* NewObject() {
  return new SubClassName();
}

// 为每个类反射提供一个 helper，构造时就完成反射函数对注册
template<typename BaseClassName>
class ClassRegisterHelper {
  public:
  ClassRegisterHelper(
      const std::string sub_class_name,
      typename ClassRegister<BaseClassName>::Constructor constructor) {
    GetRegister<BaseClassName>().AddConstructor(sub_class_name, constructor);
  }
  ~ClassRegisterHelper(){}
};

// 提供反射类的注册宏，使用时仅提供基类类名和派生类类名
#define RegisterClass(base_class_name, sub_class_name) \
  static ClassRegisterHelper<base_class_name> \
      sub_class_name##_register_helper( \
          #sub_class_name, NewObject<base_class_name, sub_class_name>);

// 创建对象的宏
#define CreateObject(base_class_name, sub_class_name_as_string) \
  GetRegister<base_class_name>().CreateObject(sub_class_name_as_string)

#endif
```

下面是使用的示例：
```
#include <iostream>
#include <memory>
#include <cstring>
#include "base3.h"
using namespace std;

class base
{
  public:
    base() {}
    virtual void test() { std::cout << "I'm base!" << std::endl; }
    virtual ~base() {}
};

class A : public base
{
  public:
    A() { cout << " A constructor!" << endl; }
    virtual void test() { std::cout << "I'm A!" <<std::endl; }
    ~A() { cout << " A destructor!" <<endl; }
};

// 注册反射类 A
RegisterClass(base, A);

class B : public base
{
  public :
    B() { cout << " B constructor!" << endl; }
    virtual void test() { std::cout << "I'm B!"; }
    ~B() { cout << " B destructor!" <<endl; }
};

// 注册反射类 B
RegisterClass(base, B);

class base2
{
  public:
    base2() {}
    virtual void test() { std::cout << "I'm base2!" << std::endl; }
    virtual ~base2() {}
};

class C : public base2
{
    public :
    C() { cout << " C constructor!" << endl; }
    virtual void test() { std::cout << "I'm C!" << std::endl; }
    ~C(){ cout << " C destructor!" << endl; }
};

// 注册反射类 C
RegisterClass(base2, C);


int main()
{
  // 创建的时候提供基类和反射类的字符串类名
  base* p1 = CreateObject(base, "A");
  p1->test();
  delete p1;
  p1 = CreateObject(base, "B");
  p1->test();
  delete p1;
  base2* p2 = CreateObject(base2, "C");
  p2->test();
  delete p2;
  return 0;
}

```
