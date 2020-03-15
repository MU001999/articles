前两天给 Ice 加了 call/cc, 为此还重构了一波, 实现 call/cc 还是因为看了轮子哥的大专系列(

里边说提供 continuation 语言实现 Coroutine 起来很轻松, 后来又查了一些资料, 都说 continuation 表达能力很强, 就实现了一手, 调用方式完全等同 call/cc, 既然是看 Coroutine 才要实现 call/cc, 那实现了之后当然要用 call/cc 实现一手 Coroutine 了(

不过这个 Coroutine 比较简陋, 只提供 create, resume, yield, destroy

```
coroutine: (@() {
    @funcs: [];
    @conts: [];
    @deletes: [];
    @co_cont: none;

    @create(func) {
        if deletes.empty() {
            @id: funcs.size();
            funcs.push(func);
            conts.push(none);
            return id;
        } else {
            @id: deletes.pop();
            funcs[id]: func;
            conts[id]: none;
            return id;
        }
    }

    @resume(id) {
        if conts[id] = none {
            conts[id]: call_with_current_continuation(@(cont) {
                co_cont: cont;
                funcs[id]();
            });
        } else {
            conts[id]: call_with_current_continuation(@(cont) {
                co_cont: cont;
                conts[id](none);
            });
        }
    }

    @yield() {
        call_with_current_continuation(co_cont);
    }

    @destroy(id) {
        funcs[id]: conts[id]: none;
        deletes.push(id);
    }

    return @() {};
})();
```

测试一下:

```
@foo() {
    @i: 0;
    while i < 5 {
        println(i: i + 1);
        coroutine.yield();
    }
}

@id1: coroutine.create(foo);
@id2: coroutine.create(foo);

@i: 0;
while i < 5 {
    coroutine.resume(id1);
    coroutine.resume(id2);
    i: i + 1;
}

coroutine.destroy(id1);
coroutine.destroy(id2);
```

输出

```
1
1
2
2
3
3
4
4
5
5
```

里边唯一我觉得有点意思的地方是 resume 里, 直接在 lambda 里将当前的 continuation 绑定到 co_cont 上, 这样用户 create 的时候传入的函数就只需要调用 yield 了

可惜除了 vsc 都没有 Ice 的高亮(
