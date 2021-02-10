# C++ 实现反射（一）
反射，就是根据一个类名，即可根据类名获取类信息，创建新对象。反射在很多语言都天然支持，然而不包括 C++，但我们肯定会经常遇到这种根据类名生成对象的场景，这就需要我们自己动手来实现了。反正 C++ 这么强大，一定没有问题 :）
### version 1
我们略做思考，就可以想到一种最简单的方案：

```C++
if (class_name == "A") {
    return new A();
} else if (class_name == "B") {
    return new B();
} ...
```
当然，这个方案略做思考就可以否掉，这么暴力毫无美感，代码难以维护，每定义个新的类就需要添加一段新的 if，如果很多人共用和维护一定是场灾难。

### Version 2
然而其实我没有放弃，看起来需要添加的代码这么相似，好像用个 template 可以解决？

```C++
#include <string>

template<typename BaseClassName, typename SubClassName>
BaseClassName* CreateObject() {
  return new SubClassName();
}

#define CLASS_OBJECT_CREATOR(base_class_name, sub_class_name) \
  CreateObject<base_class_name, sub_class_name>()
```
每次调用的时候只要形如 `base* p = CLASS_OBJECT_CREATOR(base, A);` 这样就好了，也许我们不知道传入的是哪个派生类，但都可以用基类的指针保存下来，像下面：

```c++
class base
{
  public:
    base() {}
    virtual test() { std::cout << "I'm class base!" << std::endl; }
    virtual ~base() {}
};
// 基类这里再封装一个宏，就不用每次都传入基类名
#define CreateMyClass(class_name) \
  dynamic_cast<class_name*>(CLASS_OBJECT_CREATOR(base, class_name));

class A : public base
{
  public :
    A(){ std::cout << "A constructor!" << std::endl; }
    virtual test() { std::cout << "I'm class A!" << std::endl; }
    ~A() { std::cout << "A destructor!" << std::endl; }
};

class B : public base
{
  public :
    B(){ std::cout << "B constructor!" << std::endl; }
    virtual test() { std::cout << "I'm class B!" << std::endl; }
    ~B() { std::cout << "B destructor!" << std::endl; }
};

int main()
{
    A* p1 = CreateMyClass(A);
    delete p;
    B* p2 = CreateMyClass(B);
    delete p2;
    return 0;
}
```
看起来好像没什么问题，只要调用 `CreateMyClass(class_name)` 就可以创建一个新的对象了。但总觉得哪里不对....仔细一想，如果类名是个字符串呢？`CreateMyClass("A")` 压根就没办法处理呀！因为 C++ 本质上无法处理 `new "A"` 这种语法所以才说不支持反射，像上面这样处理的话也根本没有解决这个问题，只是把 new 的语法用宏包装起来而已...囧...

还是认真想想怎么解决这个问题吧，见下篇
