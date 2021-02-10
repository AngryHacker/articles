# Linux C++ 中文处理

## 背景
C++ 对于中文的处理是很蛋疼的事情，然而，不幸的我们接到命令，要在 Linux 下支持对文案进行文案超长截断处理。这样的话应该怎么做呢？

## UTF-8 介绍
首先，我们可以假定我们接受到的字符串是 UTF-8 编码的。如果在本地的话可以通过本地环境配置来保证。命令行下运行 `locale` 命令，LC_CTYPE 应该是 UTF-8 的。vim 打开文件敲下 `:set` 命令，应该有一行是 `fileencoding=utf-8`。这样我们就有了工作的基础。

UTF-8 是对 Unicode 字符集的实现，它是一种变长编码，对于一个Unicode 的字符编码成 1 至 4 个字节。我们可以认为，在 UTF-8 中，英文是 1 个字节，中文是 3 个字节。

UTF-8 的详细介绍可以看：
* [Unicode 和 UTF-8 有何区别？ — 知乎](https://www.zhihu.com/question/23374078)

* [UTF-8 — 维基百科](https://zh.wikipedia.org/wiki/UTF-8)

## 设计思路
既然知道 UTF-8 的中英文字符字节长度，那我们可能想用这样一个方案：遍历字符串，判断当前字节属于中文还是英文，如果英文则对长度加一并从下一个字节继续处理，如果是中文则对长度加一并跳到后面第三个字节继续处理。达到我们需要的文案长度时，break 跳出循环，返回当前遍历得到的子字符串。

但是这样的实现会感觉很 hack，有点暴力，程序容易写出问题。而且我们前面的假设毕竟是一般情况下（虽然概率很低），如果出现一个四字节的字符那程序会错得一塌糊涂。

如果有一种编码或数据类型，每个中英文字符都占据相同长度，那我们的处理就会简单多了。这时候我们想到了 C++ 的 wstring 类型，wstring 的 size() 函数返回的就是包含的中英文字符个数。wstring 与 string 一样都是基于  basic_string 类模板，不同的是 string 使用 char 为基本类型，而 wstring 是 wchat_t。wchar_t 可以支持 Unicode 字符的存储，在 Win 下是两个字节， Linux 的实现则是四个字节，可以直接用 `sizeof(wchar_t)` 查看类型长度。

到这里我们已经有了基本的思路：实现 string 和 wstring 的互相转换，并用 wstring 来判断字符个数，在超长时进行截断。

## string 与 wstring 的转换
###  转换版本一
如果你的 g++ 版本够高（5.0以上），那么可以采用下面的写法，这是最好的：
```
#include <codecvt>
#include <string>

std::wstring s2ws(const std::string& str)
{
    using convert_typeX = std::codecvt_utf8<wchar_t>;
    std::wstring_convert<convert_typeX, wchar_t> converterX;

    return converterX.from_bytes(str);
}

std::string ws2s(const std::wstring& wstr)
{
    using convert_typeX = std::codecvt_utf8<wchar_t>;
    std::wstring_convert<convert_typeX, wchar_t> converterX;

    return converterX.to_bytes(wstr);
}
```

std::wstring_convert 是 C++11 标准库提供的对 string 和 wstring 的转换，对 Unicode 进行了语言和库级别的支持。但这一特性在 gcc/g++ 5.0 以上才被支持。

参考资料：
* [How to convert wstring into string? — stackoverflow](http://stackoverflow.com/questions/4804298/how-to-convert-wstring-into-string)

* [std::wstring_convert — cppreference](http://en.cppreference.com/w/cpp/locale/wstring_convert)

* [std::wstring_convert — cplusplus](http://www.cplusplus.com/reference/locale/wstring_convert/)

### 转换版本二
如果你的 g++ 版本是支持部分 c++11 特性，那么第二个版本可以用 unique_ptr 来管理内存，这样可以避免直接操作指针的尴尬，程序更加安全。

```c++
#include <cstdlib>
#include <memory>
#include <string>

std::wstring s2ws(const std::string& str) {
  if (str.empty()) {
    return L"";
  }
  unsigned len = str.size() + 1;
  setlocale(LC_CTYPE, "en_US.UTF-8");
  std::unique_ptr<wchar_t[]> p(new wchar_t[len]);
  mbstowcs(p.get(), str.c_str(), len);
  std::wstring w_str(p.get());
  return w_str;
}

std::string ws2s(const std::wstring& w_str) {
    if (w_str.empty()) {
      return "";
    }
    unsigned len = w_str.size() * 4 + 1;
    setlocale(LC_CTYPE, "en_US.UTF-8");
    std::unique_ptr<char[]> p(new char[len]);
    wcstombs(p.get(), w_str.c_str(), len);
    std::string str(p.get());
    return str;
}
```

new 数组的长度要考虑到，因为 wchar_t 为 4 个字节，对于 s2ws， wstring 的长度肯定小于等于 string 的长度，而对 ws2s， string 的长度也肯定小于等于 wstring 4 倍的长度。+1 是预留给字符串的结束符 '\0'。

setlocale 函数用于运行时的语言环境，可以在命令行用 locale 查看当前系统的语言环境设置，LC_CTYPE 指语言符号及其分类 。网上很多版本使用 `setlocale(LC_CTYPE, "");` ， 这里第二个参数用空字符串，会使用系统当前默认的 locale 设置。但是这样有个问题，也许你写出来的程序在本机运行正确，但到服务器上就错了，因为服务器的 locale 不一定是 utf8，所以这里要强制设置为 en_US.UTF-8。

mbstowcs 和 wcstombs 是两个 C 语言中对多字节字符串和宽字符字符串的互相转换函数，依赖于当前 locale 中所指定的字符编码。

### 转换版本三
如果 g++ 连 unique_ptr 都不支持，那就只能使用下面的 new/delete 了。

```c++
#include <cstdlib>
#include <string>

std::wstring s2ws(const std::string& str) {
  if (str.empty()) {
    return L"";
  }
  unsigned len = str.size() + 1;
  setlocale(LC_CTYPE, "en_US.UTF-8");
  wchar_t *p = new wchar_t[len];
  mbstowcs(p, str.c_str(), len);
  std::wstring w_str(p);
  delete[] p;
  return w_str;
}

std::string ws2s(const std::wstring& w_str) {
    if (w_str.empty()) {
      return "";
    }
    unsigned len = w_str.size() * 4 + 1;
    setlocale(LC_CTYPE, "en_US.UTF-8");
    char *p = new char[len];
    wcstombs(p, w_str.c_str(), len);
    std::string str(p);
    delete[] p;
    return str;
}
```

## 最终实现
实现了 string 和 wstring 的转换后，接下来的处理就很简单了。实现处理函数 FormatText，然后加入 main 函数测试，完整代码如下：

```c++
#include <cassert>
#include <cstdlib>
#include <iostream>
#include <string>

static const int kTextSize = 10;

std::wstring s2ws(const std::string& str) {
  if (str.empty()) {
    return L"";
  }
  unsigned len = str.size() + 1;
  setlocale(LC_CTYPE, "");
  wchar_t *p = new wchar_t[len];
  mbstowcs(p, str.c_str(), len);
  std::wstring w_str(p);
  delete[] p;
  return w_str;
}

std::string ws2s(const std::wstring& w_str) {
    if (w_str.empty()) {
      return "";
    }
    unsigned len = w_str.size() * 4 + 1;
    setlocale(LC_CTYPE, "");
    char *p = new char[len];
    wcstombs(p, w_str.c_str(), len);
    std::string str(p);
    delete[] p;
    return str;
}


bool FormatText(std::string* txt) {
  if (NULL == txt) {
    return false;
  }
  std::cout << "before:" << *txt << std::endl;
  std::wstring w_txt = s2ws(*txt);
  std::cout << "wstring size:" << w_txt.size() << std::endl;
  std::cout << "string size:" << (*txt).size() << std::endl;
  if (w_txt.size() > kTextSize) {
    w_txt = w_txt.substr(0, kTextSize);
    *txt = ws2s(w_txt);
    *txt += "...";
  }
  std::cout << "after:" << *txt << std::endl;
  return true;
}

int main() {
  assert(L"" == s2ws(""));

  std::string txt = "龙之谷app好玩等你";
  assert(24 == txt.size());
  std::wstring w_txt = s2ws(txt);
  assert(10 == w_txt.size());

  assert("" == ws2s(L""));

  w_txt = L"龙之谷app好玩等你";
  assert(10 == w_txt.size());
  txt = ws2s(w_txt);
  assert(24 == txt.size());

  txt = "龙之谷app公测开启";
  std::string format_txt = txt;
  FormatText(&format_txt);
  assert(txt == format_txt);

  txt = "龙之谷app公测火爆开启";
  FormatText(&txt);
  format_txt = "龙之谷app公测火爆...";
  assert(format_txt == txt);

  return 0;
}

```