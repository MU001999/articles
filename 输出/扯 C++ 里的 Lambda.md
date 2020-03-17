之前写(抄) parsec 的时候, 在重载 `operator>>` 的时候, `operator>>` 需要接收一个 lambda, 之后返回一个 `Component<R>`, 其中 R 是接收 lambda 的返回值类型, 所以就要搞到 lambda 对应的函数类型

在一开始我是直接用 `std::function` 做的, 但是众所周知, 下面这样的写法是匹配不了的:

```cpp
template<typename R, typename ...Args>
ParsecComponent<R> operator>>(std::function<R(Args...)> callback) {
    ParsecComponent<R> component;
    ...
    return component;
}
```

因为 lambda 表达式到 std::function 要进行类型转换, 毕竟是两个类型, 所以指明 std::function 的模板实参的时候才能进行 lambda -> std::function 的隐式转换, 不过一开始为了偷懒, 而且我只需要拿到 lambda 的返回值类型, 就这样写了:

```cpp
template<typename Func>
auto operator>>(Func &&callback) {
    using NewResult = typename decltype(std::function(callback))::result_type;
    ParsecComponent<NewResult> component;
    ...
    return component;
}
```

所以说 `auto` 这种东西还真是好用啊, 类型还可以拖延到函数体里做(

这样做总归有种脱裤子放屁的感觉, 那么怎么不通过 std::function 就能拿到 lambda 表达式对应的函数类型呢?

众所周知, 每一个 lambda 表达式的类型都是不一样的, 比如:

```cpp
template<typename Func>
void print(Func &&f) {
    std::cout << typeid(Func).name() << std::endl;
}

int main() {
    print([](){});
    print([](){});
    return 0;
}
```

我这里输出的结果是

```plain
Z4mainE3$_0
Z4mainE3$_1
```

毕竟如果要每个定义相同的 lambda 的类型相同是不现实而而且没有必要的, 何况还有闭包捕获之类的复杂性需要考虑, 但是众所周知, lambda 返回的是一个对象, 也即它的类型重载了 `operator()`, 而 `operator()` 又是一个函数, 那么我们不久可以通过推导其重载的 `operator()` 函数类型拿到 lambda 对应的函数类型了吗(

所以这件事情很清晰明了了, 我们只需要拿到 lambda 表达式产生的匿名类型, 然后根据类成员函数寻址到它的 `operator()`, 然后推导出 `operator()` 的函数签名就可以了!

那就划分成三步吧

## 一, 得到 lambda 对应的匿名类类型

这一步是很简单的, 因为只要模板实参自动推导一下就出来了

```cpp
template<typename Func>
void foo(Func &&lambda);
```

那现在这个 Func 就是我们需要的类型

## 二, 推导 Func 重载的 `operator()` 函数签名

因为当前我们只有一个 Func 类型, 但是如果要获取 `operator()` 的签名的话, 我们还需要对 `operator()` 的类型做一个特化, 比如

```cpp
template<typename T> struct lambda_traits;
template<typename R, typename ClassType, typename ...Args>
struct lambda_traits<R(ClassType::*)(Args...) const> {
    using result_type = R;
    using args_type = std::tuple<Args...>;
    template<size_t index>
    using arg_type_at = std::tuple_element_t<index, args_type>;
    static constexpr size_t arity = sizeof...(Args);
};
```

很简单对吧, 只需要对类的成员函数的类型做一个特化就好了

其实这个时候已经可以用了, 比如

```cpp
template<typename Func,
    typename R = typename lambda_traits<decltype(&Func::operator())>::result_type>
void foo(Func &&lambda);
```

## 三, 包装一下

其实可以看到第二步的时候已经可以用了, 那么我们只需要把调用的过程包装一下, 但是由于我们拿到的是一个 Func, 所以需要把之前的 `lambda_traits` 变成基类, 然后另 Func 实例化的模板类继承它就可以了

```cpp
template<typename T> struct lambda_traits_base;
template<typename R, typename ClassType, typename ...Args>
struct lambda_traits_base<R(ClassType::*)(Args...) const> {
    using result_type = R;
    using args_type = std::tuple<Args...>;
    template<size_t index>
    using arg_type_at = std::tuple_element_t<index, args_type>;
    static constexpr size_t arity = sizeof...(Args);
};

template<typename Func>
struct lambda_traits : lambda_traits_base<decltype(&Func::operator())> {};
```

支持 C++17 的 lambda 现在是没有问题了, 那么有人要问了, C++20 的 lambda 是支持模板 operator() 的, 这个显然是不支持的啊, 是垃圾

那就来接轨一哈 20 吧, 反正接轨了之后是不影响 17 的 lambda 推导的

<hr>

支持 template operator() 的 lambda 顾名思义就是长这样子的啦:

```cpp
auto foo = []<typename T, typename ...Args>(Args...) -> T { return T(); };
foo.operator()<...>(...);
```

可以看出来(目前)如果要调用这个 lambda 的话, 我们需要显式地指明模板的实参, 所以在推导的时候, 模板实参的信息也是要提供的, 那么只需要简单地修改一下我们的 lambda_traits:

```cpp
template<typename Func, typename ...Args>
struct lambda_traits : lambda_traits_base<decltype(&Func::template operator()<Args...>)> {};

template<typename Func>
struct lambda_traits : lambda_traits_base<decltype(&Func::operator())> {};
```

这样一来就可以了, 不过还有一种特殊情况, 比如说我们有一个这样类似提供了 template operator() 的类:

```
struct Foo
{
    template<typename ...Args>
    void operator() {};
};
```

当我们在获取他无模板实参的 operator() 时, 我们只能通过 `&Foo::template operator()<>` 而不能写 `&Foo::operator()`, 当然这也是显而易见的, 不过如果我们的 lambda 支持的 template operator() 能够接收无实参的实例化的话, 就会导致前边的 lambda_traits 失效, 所以我们需要在模板实参 Args 为空的时候判断一下要取 `operator()` 还是 `template operator()<>`

```cpp
template<typename T, typename = std::void_t<>>
struct call_which {
    using type = decltype(&T::operator());
};
template<typename T>
struct call_which<T, std::void_t<decltype(&T::template operator()<>)>> {
    using type = decltype(&T::template operator()<>);
};
template<typename T>
using call_which_t = typename call_which<T>::type;

template<typename T, typename ...Args>
struct lambda_traits : lambda<decltype(&T::template operator()<Args...>)> {};
template<typename T>
struct lambda_traits<T> : lambda<call_which_t<T>> {};
```

通过 `void_t` 判断了一下是不是具有 `template operator()<>` 就可以了
