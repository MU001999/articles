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

毕竟如果要每个定义相同的 lambda 的类型相同是不现实而而且没有必要的, 何况还有闭包捕获之类的复杂性需要考虑, 但是众所周知, lambda 返回的是一个对象, 也即它的类型重载了 `operator()`, 而 `operator()` 又是一个函数, 那么我们不久可以通过推导其重载的 `operator()` 函数类型拿到 lambda 的类型了吗(
