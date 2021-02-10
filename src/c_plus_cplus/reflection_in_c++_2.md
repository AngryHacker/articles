# C++实现反射（二）
找了一些资料，参考了 [C++反射——开源中国](http://www.oschina.net/code/snippet_230828_9913) 这篇，做了一些修改和简化，成为了 Version3.

思路其实并不复杂，可以进行反推：
1. 反射是根据类名动态生成类，如果我们有一个全局的映射关系，可以从类名得到类的相关信息 ClassInfo，包括类的构造函数，那么我们便能实现这一点。所以我们需要维护一个 `map<std::string, ClassInfo>` 这样的结构作为全局映射，并提供 Register 接口作为 map 的注册操作，只要再定义一个 `CreateObject(const std::string class_name)`，每次在 map 中检索到对应的 ClassInfo，然后创建并返回对象，这样就有了反射的基础。
2. 那么 `CreateObject` 函数返回的类型是什么呢？因为反射得到的类各不相同，所以我们需要一个 Object 类作为所有反射类的基类，这样只要返回一个 Object* 指针就好了。
3. 而 ClassInfo 是作为保存类信息的结构，所以我们需要把类名和创建对象的函数指针保存下来。除此之外，还应提供一个 `CreateObject` 的成员函数供调用生成类。
4. 最后，ClassInfo 在哪里创建和保存呢？顾名思义，每个 ClassInfo 对象应该是每个类唯一的，所以我们可以每个类定义一个 static 的 ClassInfo 对象。还有，每个类要提供构造自身的接口，这样才可以把接口保存到 ClassInfo 中，供 `CreateObject` 调用。因为这两点每个反射类都是相似的，可以借助宏来简化。

这样分析完是不是挺简单的？当然更大的可能是你被我绕晕了。可以看看下面的代码，回过头再看一遍分析，应该就清楚了。：）

```C++
// base.h
#ifndef __BASE_H__
#define __BASE_H__
#include <string>
#include <map>


class Object;
class ClassInfo;

typedef Object* (*ObjectConstructorFn)(void);
bool Register(ClassInfo* ci);

class ClassInfo {
  public:
    ClassInfo(const std::string class_name, ObjectConstructorFn object_constructor):
      class_name_(class_name), object_constructor_(object_constructor) {
      Register(this);  // 构造的时候就进行注册
    }
    // 根据保存下来的函数指针构造对象
    Object* CreateObject() const {
      return object_constructor_ ? (*object_constructor_)() : nullptr;
    }
    ~ClassInfo() {}
  public:
    std::string class_name_;
    ObjectConstructorFn object_constructor_;
};

// 维护全局的映射关系
static std::map<std::string, ClassInfo*> *class_info_map = nullptr;

// 每个反射类的都需要一个 ClassInfo 和 CreateObject
#define DECLARE_CLASS(name) \
  protected: \
    static ClassInfo class_info_; \
  public: \
    static Object* CreateObject();

// class_info 用类名和类的 CreateObject 函数初始化
// 在每个反射类的 cpp 文件中使用该宏
#define IMPLEMENT_CLASS(name) \
  ClassInfo name::class_info_(#name, (ObjectConstructorFn) name::CreateObject);\
  Object* name::CreateObject() { \
    return new name; \
  }

// 所有反射类的基类
// 用来传递反射对象的指针
class Object {
    DECLARE_CLASS(Object);
  public:
    Object() {}
    virtual ~Object() {}
    // 提供全局可使用的 CreateObject 函数，根据类名生成对象
    static Object* CreateObject(std::string name);
};

IMPLEMENT_CLASS(Object)

Object* Object::CreateObject(std::string name)
{
    std::map<std::string, ClassInfo*>::const_iterator iter =
      class_info_map->find(name);
    if(class_info_map->end() != iter) {
      return iter->second->CreateObject();
    }
    return nullptr;
}

// map 的注册接口
bool Register(ClassInfo* ci) {
  if (!class_info_map) {
    class_info_map = new std::map<std::string, ClassInfo*>();
  }
  if (ci) {
    if (class_info_map->find(ci->class_name_) == class_info_map->end()) {
      class_info_map->insert(
          std::map<std::string, ClassInfo*>::value_type(ci->class_name_, ci));
    }
  }
}
#endif
```

使用的时候包含该头文件，反射类（例如 A）继承 Object，并在类中加入宏 `DECLARE_CLASS(A)`，在类的实现文件中加入宏 `IMPLEMENT_CLASS(A)`，最后创建对象时使用 `Object::CreateObject("A")` 就可以了。
```C++
// base_test.cpp
#include<iostream>
#include<cstring>
#include "base.h"

using namespace std;

class A : public Object
{
    DECLARE_CLASS(A)
  public :
    A(){ std::cout << "A constructor!" << std::endl; }
    virtual test() { std::cout << "I'm class A!" << std::endl; }
    ~A() { std::cout << "A destructor!" << std::endl; }
};
IMPLEMENT_CLASS(A)

class B : public Object
{
    DECLARE_CLASS(B)
  public :
    B(){ std::cout << "B constructor!" << std::endl; }
    virtual test() { std::cout << "I'm class B!" << std::endl; }
    ~B() { std::cout << "B destructor!" << std::endl; }
};
IMPLEMENT_CLASS(B)

int main()
{
    Object* p = Object::CreateObject("A");
    delete p;
    p = Object::CreateObject("B");
    delete p;
    p = Object::CreateObject("A");
    delete p;
    return 0;
}
```

到这一步我们就真正实现反射了，是不是其实并不难？但我还是觉得不够好：
1. 每个反射类都要继承于 Object，看起来总是有点奇怪，使用的时候要求使用方知道这个用 Object 指针来保存，不甚明朗。
2. 对于我来说，我只想实现根据类名创建对象，所以代码其实可以更简化一点，map 里直接保存类名和函数指针的映射就好了，这样整体会更简化些。

下一篇我们解决这两个问题，实现 Version 4.

