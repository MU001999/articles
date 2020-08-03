---
title: C++ 实现 Parsec
date: 2020/03/15
---

前一段时间看到了梨梨喵聚聚写的[Parser Combinator 在 C++ 里的 DSL](https://zhuanlan.zhihu.com/p/25411428), 感觉好厉害, 正好毕设里要写一部分前端, 昨天又把这篇文章看了一遍, 想着我也要用这么酷炫的东西来参与一下毕设, 于是今天仿了一个, 不过由于电脑屏幕太小(理由), 看不懂梨梨喵聚聚的代码, 只好照着文章里的理念自己试着实现一下, 类的设计应该差不多, 不过具体的实现应该鶸了很多, 代码在[parsec](https://github.com/MU001999/parsec), 目前还不支持左递归和垃圾回收(

先上一个加减的小例子吧:

```cpp
Parsec<char> Decimal;
Parsec<string> Number;
Parsec<int> Primary, Additive, Additive_;

// Decimal := '0' | ... | '9'
Decimal = ('0'_T | '1'_T | '2'_T | '3'_T | '4'_T | '5'_T | '6'_T | '7'_T | '8'_T | '9'_T );

// Number := Decimal Number | Decimal
Number =
    (Decimal + Number >>
        [](char decimal, string number) {
            return decimal + number;
        }) |
    (Decimal >>
        [](char decimal) {
            return string() + decimal;
        });

// Primary := Number
Primary = Number >> [](string number) {
    return stoi(number);
};

// Additive := Primary Additive_ | Primary
// Additive_ := + Additive | - Additive
Additive =
    (Primary + Additive_ >>
        [](int primary, int additive) {
            return primary + additive;
        }) |
    (Primary >> [](int primary) {
        return primary;
    });
Additive_ = (('+'_T | '-'_T) + Additive >> [](char op, int additive) {
    return (op == '+' ? additive : -additive);
});

cout << Additive("1+2+3") << endl;
```

例子是改的梨梨瞄聚聚的, 因为不支持左递归, 只能手动提取公因子了(

和梨梨瞄聚聚重复的原理部分就不说了, 想知道的可以看一下梨梨瞄聚聚的那篇文章, 说一下不一样的地方吧, 我的实现里的类型笛卡尔乘积的结果使用 std::tuple<Ts...> 表示的, 类型的和的结果用 std::variant<Ts...> 表示, 当返回值的类型可能有多类型的时候就可以使形参的类型为 std::variant<Ts...>, 保证类型安全的同时还能接收多个类型的结果, 同样可以令组合子返回的是一个 tuple 类型, 可以继续和其它类型相乘或者相加, 后边就可以简化实现.

写了好几个小时, 虽然代码不多, 但是模板多起来密度太大真的眼花缭乱, 希望之后能支持一下左递归之类的操作吧, 毕竟还要拿来写毕设, 边用边改吧(
