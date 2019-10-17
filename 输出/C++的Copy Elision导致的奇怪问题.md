最近写设计模式作业的时候, 有一个作业是实现[装饰器模式 (Decorator Pattern)](https://en.wikipedia.org/wiki/Decorator_pattern), 由于我不会 Java, 所以只能用 C++ 来实现 :)

在这个背景下, 会有简单(表意)的几个类, 如下:

```cpp
class Base
{
  public:
    virtual ~Base() = 0;
    virtual int getData() const = 0;
};

inline Base::~Base() {}

class DerivedA final : public Base
{
  public:
    DerivedA(int data) : data_(data) {}
    int getData () const override
    {
        return data_;
    }

  private:
    const int data_;
};

class DerivedB final : public Base
{
  public:
    DerivedB(const Base &pre) : pre_(pre) {}
    int getData () const override
    {
        return pre_.getData() + 1;
    }

  private:
    const Base &pre_;
};
```

简单来写就是上面这样, DerivedB 类型的对象可以接收以 Base 类作为基类的对象引用并且绑定到成员 pre_ 上, 在调用 getData 方法时会调用 pre_ 绑定的对象的 getData 方法, 并在其结果的基础上运算后返回

而这样的设计会导致一种很直觉的使用方法, 如下:

```cpp
cout << DerivedB(DerivedB(DerivedA(10))).getData() << endl;
```

也即嵌套对象, 实现 getData 的多次调用

但是这样的使用方式会造成与预期不符的结果出现, 如下:

```plaintext
~> g++ -std=c++17 test.cpp -o test
~> ./test
11
~> clang++ -std=c++17 test.cpp -o test
~> ./test
11
```

会发现结果表明, 只有一个 DerivedB 类型的对象被构造了出来, [cppreference 的 copy elision 章节](https://en.cppreference.com/w/cpp/language/copy_elision)中的解释部分有提到:

> Under the following circumstances, the compilers are required to omit the copy and move construction of class objects, **even if the copy/move constructor and the destructor have observable side-effects**. The objects are constructed directly into the storage where they would otherwise be copied/moved to. **The copy/move constructors need not be present or accessible**, as the language rules ensure that no copy/move operation takes place, even conceptually:

> * In a return statement, when the operand is a prvalue of the same class type (ignoring cv-qualification) as the function return type:
> ```cpp
> T f() {
>     return T();
> }
>
> f(); // only one call to default constructor of T
> ```

> * In the initialization of a variable, when the initializer expression is a prvalue of the same class type (ignoring cv-qualification) as the variable type:
> ```cpp
> T x = T(T(f())); // only one call to default constructor of T, to initialize x
> ```

上面这句被我标粗的文字可以看到, 即使拷贝/移动构造有副作用, 依然只构造一次, 甚至不需要有拷贝/移动构造函数

可以在类中添加如下定义:

```cpp
class DerivedB final : public Base
{
  public:
    // new additions
    DerivedB(const DerivedB &) = delete;
    DerivedB(DerivedB &&) = delete;

    DerivedB(const Base &pre) : pre_(pre) {}
    int getData () const override
    {
        return pre_.getData() + 1;
    }

  private:
    const Base &pre_;
};
```

会发现依然可以通过编译并且运行结果与之前相同 ( 因为在 C++17 中 Copy Elision 已经不再是可选项 ), 但是在 C++17 之前如果 delete 了这两个拷贝/移动构造函数, 会导致无法通过编译, 尽管有可以匹配 const Base & 类型的构造函数, 也依然不可以:

```plaintext
test.cpp:45:13: error: functional-style cast from 'DerivedB'
      to 'DerivedB' uses deleted function
    cout << DerivedB(DerivedB(DerivedA(10))).getData...
            ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test.cpp:31:5: note: candidate constructor has been
      explicitly deleted
    DerivedB(DerivedB &&) = delete;
    ^
test.cpp:30:5: note: candidate constructor has been
      explicitly deleted
    DerivedB(const DerivedB &) = delete;
    ^
test.cpp:33:5: note: candidate constructor
    DerivedB(const Base &pre) : pre_(pre) {}
    ^
1 error generated.
```

感谢 [禽牙](https://www.zhihu.com/people/lei-yu-10-27) 在评论中提出, 事实上如果我们在尚未 delete 那两个构造函数通过如下的方式调用, 也依然不可行:

```cpp
DerivedA a(10);
DerivedB b1(a);
DerivedB b2(b1);
cout << b2.getData() << endl;
```

结果依然是 11, 这是因为重载决议后其实我们调用的是匹配类型为 `const DerivedB &` 的构造函数, 同样如果我们约定在代码中都是用这样的方式编写程序, 我们就可以获得一种解决方式, 编写匹配类型的构造函数, 如下:

```cpp
...
DerivedB(const DerivedB &pre) : pre_(pre) {}
...
```

再通过声明中间变量的方式调用就可以获得正确结果, 但是通过纯右值的形式依然不行:

```cpp
...
DerivedA a(10);
DerivedB b1(a);
DerivedB b2(b1);
cout << b2.getData() << endl;
// print 12
...
cout << DerivedB(DerivedB(DerivedA(10))).getData() << endl;
// print 11
...
```

所以如何才能在这种设计下通过这种方式正常使用呢?

一种可以显著增加代码程度的方式是, 手动添加强制转换:

```cpp
cout << DerivedB(static_cast<const Base &>(DerivedB(DerivedA(10)))).getData() << endl;
cout << DerivedB((const Base &)(DerivedB(DerivedA(10)))).getData() << endl; // C-style
```

当然这种方式其实还是可以接受的

还有一种可以不会增加太多冗余代码的方式是在构造函数里增加一个冗余参数, 区分开就可以了, 比如:

```cpp
...
class DerivedB final : public Base
{
  public:
    DerivedB(const Base &pre, int) : pre_(pre) {}
    int getData () const override
    {
        return pre_.getData() + 1;
    }

  private:
    const Base &pre_;
};
...
cout << DerivedB(DerivedB(DerivedA(10), 0), 0).getData() << endl;
```

当然以上两种都是在不涉及模板的情况下完成的, 但是都会对调用的方便性产生影响

还有一种不会更改调用方式, 通过模板区分嵌套的相同类型的方式, 感谢 [91khr](https://www.zhihu.com/people/91khr) 提供:

首先先将 DerivedB 变成一个类模板, 之后添加[模板推导指引](https://en.cppreference.com/w/cpp/language/class_template_argument_deduction)来实现嵌套区分

```cpp
...
template <int Lv>
class DerivedB;
...
template <typename Ty> DerivedB(Ty) -> DerivedB<0>;
template <int Lv> DerivedB(DerivedB<Lv>) -> DerivedB<Lv + 1>;
...
cout << DerivedB(DerivedB(DerivedA(10))).getData() << endl;
```

在此条件之下, 结果与预期相同, 完美解决问题:

```plaintext
~> clang++ -std=c++17 test.cpp -o test
~> ./test
12
```
