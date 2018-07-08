# 《STL源码剖析》学习半生记：第一章小结与反思

> 不学STL，无以立。——陈轶阳

从1.1节到1.8节大部分都是从各方面介绍STL，
包括历史之类的（大致上是这样，因为实在看不下去我就直接略到了1.9节（其实还有一点1.8.3的内容））。
第一章里比较实用（能用在自己代码当中）的部分应该就是1.9节*可能令你困惑的C++语法*这部分了。<br><br>

而1.9中又分为以下几个小节：

* 1.9.1 stl_config.h 中的各种组态（configurations）
* 1.9.2 临时对象的产生和运用
* 1.9.3 静态常量整数成员在class 内部直接初始化
* 1.9.4 increment/decrement/dereference 操作符
* 1.9.5 前闭后开区间表示法 [ )
* 1.9.6 function call 操作符（operator()）

### 1.9.1 stl_config.h 中的各种组态（configurations）
所谓组态（configurations）这个翻译一开始看得我是很晕……
后来看到configurations才大概知道了是配置、设定、设置的意思。
简而言之就是通过常量设定，
来取舍代码。
不过看了几个看得不是很懂，
所以打算后面的看完了再详细地看这一节（滑稽）。

### 1.9.2 临时对象的产生和运用
临时对象很常见，
比如值传递（copy by value）的时候就会copy出一个临时对象。
尽管大部分时候临时对象意味着效率会降低，
但有时候故意创造一些临时变量则会使程序变得干净清爽。
而刻意制造临时对象的方法，
则是利用仿函数（1.9.6节的内容）使得在构建一个对象后可将其通过类似函数调用的方式工作，
这时所构建出的对象相当于调用其构造函数(constructor)且没有指定对象名称。
<br>
```cpp
// 一个将临时对象用于for_each()的例子
#include <vector>
#include <algorithm>
#include <iostream>

using namespace std;

template <typename T>
class print
{
public:
    void operator()(const T& elem) // operator()重载，见1.9.6节
    {
        cout << elem << ' ';
    }
};

int main()
{
    int ia[6] = {0, 1, 2, 3, 4, 5};
    vector<int> iv{ia, ia+6};
    for_each(iv.begin(), iv.end(), print<int>());
    return 0;
}
```
> 这样使用时效果类似于使用模版函数，但是可以确保打印时的元素都是同一种类型。其中for_each用法为**for each**用法，更多用法请在网络上冲浪，搜索查阅。

### 1.9.3 静态常量整数成员在class 内部直接初始化
如果class中包含const static integral data member（静态常量整数成员），
那么根据C++标准就可以直接在class之内给予初值，
其中integral则泛指所有整数类型，
不单指int。
```cpp
// 一个测试class内部直接初始化静态常量整数成员的例子
#include <iostream>

using namespace std;

template <typename T>
class testClass
{
public:
    static const int _datai = 5;
    static const long _datal = 3L;
    static const char _datac = 'c';
};

int main()
{
    cout << testClass<int>::_datai << endl; // 5
    cout << testClass<int>::_datal << endl; // 3
    cout << testClass<int>::_datac << endl; // c
    return 0;
}
```

### 1.9.4 increment/decrement/dereference 操作符
即重载自增（decrement）与自减（increment）以及取值（dereference）操作符。
且由于编译器必须能够识别出前缀自增（减）与后缀自增（减），
人为规定了用一个 int 区分，
并没有实际的含义。
```cpp
// 一个示例定义
#include <iostream>

using namespace std;

class INT
{
friend ostream& operator<<(ostream& os, const INT& i);

public:
    INT(int i): m_i(i){};
    
    // prefix: increment and then fetch
    INT& operator++(){};
    
    // postfix: fetch and then increment
    const INT operator++(int){};
    
    // prefix: decrement and then fetch
    INT& operator--(){};
    
    // postfix: fetch and then decrement
    const INT operator--(int){};
    
    // dereference
    ElemType& operator*() const{};

private:
    int m_i;
};
```

### 1.9.5 前闭后开区间表示法 [ )
任何一个STL算法，
都需要获得由一对迭代器（泛型指针）所标示的区间，
而这一对迭代器所标示的即是个前闭后开区间，
以[first, last)表示，
first指向第一个元素的位置，
last则指向最后一个元素的下一个位置。

### 1.9.6 function call 操作符（operator()）
即重载函数调用操作符（一对小括号）。如果针对某个class进行operator()重载，它就成为一个仿函数。
```cpp
// 一个将operator()重载的例子
#include <iostream>

using namespace std;

template <class T>
struct plus
{
    T operator()(const T& x, const T& y) const
    {
        return x + y;
    }
};

int main()
{
    plus<int> plusobj; // 产生仿函数对象
    
    // 像一般函数一样使用仿函数对象
    cout << plusobj(3, 5) << endl; // 8
    
    // 直接产生临时仿函数对象并调用
    cout << plus<int>()(5, 3) << endl; // 8
}
```
> 要注意的是重载后的函数调用符是针对实例化对象而不是类