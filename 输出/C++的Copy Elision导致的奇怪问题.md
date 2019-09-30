最近写设计模式作业的时候, 有一个作业是实现[装饰器模式(Decorator Pattern)](https://en.wikipedia.org/wiki/Decorator_pattern), 由于我不会Java, 所以只能用C++来实现(笑)

在这个背景下, 会有简单(表意)的几个类, 如下:

```cpp
class Base
{
  public:
    virtual ~Base() = 0;
    virtual int getData() = 0;
};

class DerivedA final : public Base
{
  public:
    DerivedA(int data) : data_(data) {}
    int getData override()
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
    int getData override()
    {
        return pre_.getData() + 1;
    }

  private:
    const Base &pre_;
};
```

简单来写就是上面这样, DerivedB类型的对象可以接收以Base类作为基类的对象引用并且绑定到成员pre_上, 在调用getData方法时会调用pre_绑定的对象的getData方法, 并在其结果的基础上运算后返回

而这样的设计会导致一种很直觉的使用方法, 如下:

```cpp
cout << DerivedB(DerivedB(DerivedA(10))).getData() << endl;
```

也即嵌套对象, 实现getData的多次复用
