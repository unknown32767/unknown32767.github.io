---
layout: post
title: "yield! yield!"
date: 2026-04-17 23:00:00 +0800
categories: blog
math: true
---

之前在[[1]({% link _posts/2025-10-19-来做一个卡牌引擎吧.md %})]里提过，卡牌效果的执行经常需要随时中断，把控制权交回给玩家。宿主语言原本选的是 C#，一开始我打算基于 `IEnumerator` 稍微包一层来模拟，毕竟这里不只是“停一下”，还要把玩家输入再传回来，单靠 `MoveNext` 显然不够。

问题在于，玩家输入出现的时机并不稳定，比如效果处理中可能突然要询问一次“要不要代破”。一旦按这个方向写，`IEnumerator` 很快就会到处蔓延：向上传染到调度器，向下也会传染到那些原本只是做纯效果处理的原语。原生的 `async/await` 虽然能解决“把输入传回来”的问题，但“异步”这个性质本身一样会扩散。

所以最后还是绕回了 lua。不得不说，lua VM 的 coroutine 在这种场景下确实很顺手：不需要额外声明状态机，作为 stackful coroutine 可以在任意位置直接 `yield` 出来，而 `resume` 又能顺便把参数带回去。

# 可恢复执行

本质上，这里需要的是“可恢复执行”：发现需要用户输入时先停在原地，等输入回来之后再接着往下跑。

实现“可恢复执行”的方式其实很多，比如 C# 的 `async/await` 和 `IEnumerator`、C++20 的 coroutine、Unity 的 coroutine，以及 lua 的 coroutine。

## 编译期变换

C# 和 C++ 的方案，本质上都离不开编译期变换。无论是 `await`、`yield` 还是 `co_yield`，最终都是把相关代码改写成类似于状态机的形式，记录“从哪里挂起”，然后在恢复执行时根据参数跳回对应的位置。

Unity 则是自己做了一套调度器，调度器接收 `IEnumerator` 返回的 yield instruction，并据此决定后续怎么调度。

## 保存现场

lua coroutine 的实现就不太一样了。`coroutine.create` 本质上是通过 `lua_newthread` 创建了一个新的共享全局状态但拥有独立栈与调用链的 `lua_State`，所以 `yield` 时只要把这个 `lua_State` 的现场保留下来就行；而 `resume` 时额外参数正好压回栈上，恢复执行后就可以直接取出来。

但既然都说到“保存现场”了，自然就会想到更激进一点的东西……

# 可重入执行……

lisp/scheme 体系里有个强的东西：`call/cc`。它保存的是 `call/cc` 调用点的 continuation 和上下文。和 lua 那种“维护一个 `lua_State` 的现场”相比，continuation 更像是在保存“该如何回到这个 state，后续需要执行什么”，所以重入时会直接跳回那个 state。

## yinyang谜题

既然都聊到这了，那经典的 yinyang 谜题自然也绕不过去：

```scheme
(let* ((yin
         ((lambda (cc) (display #\@) cc) (call/cc (lambda (c) c))))
       (yang
         ((lambda (cc) (display #\*) cc) (call/cc (lambda (c) c)))))
    (yin yang))
```

大体上的执行过程可以这样理解：

- 输出 @，此时有 `yin` $\to C_{1}$。可以把 $C_{1}$ 理解成一个 continuation：它会在输出 @ 之后，把传入参数绑定给 `yin`，再继续执行。
- 接着输出 *，此时有 `yang` $\to C_{2}$。类似地，$C_{2}$ 也是一个 continuation，只不过它捕获到的环境里已经有 `yin` $\to C_{1}$。
- 执行 $(C_{1}\ C_{2})$ 时，程序会回到 $C_{1}$：再次输出 @，并把 $C_{2}$ 绑定给 `yin`。
- 然后会生成新的 `yang` $\to C_{3}$：此时 $C_{3}$ 捕获到的环境里，`yin` 绑定的是 $C_{2}$。接着执行 $(C_{2}\ C_{3})$，回到 $C_{2}$，输出 *，再把 $C_{3}$ 绑定给 `yang`。
- 之后流程就会在 $(C_{1}\ C_{3})$、$(C_{3}\ C_{4})$、$(C_{2}\ C_{4})$、$(C_{1}\ C_{4})$ 这样的结构里继续展开。

所以每次执行到 $(C_{1}\ C_{N})$ 时，都会生成一个新的 continuation，记作 $C_{N+1}$，其中 `yin` 会被重新绑定为 $C_{N}$。后面的过程就会以 $(C_{N}\ C_{N+1})$ 为起点，一路重新调回 $(C_{1}\ C_{N+1})$。

如果硬要用 lua 写一个“味道差不多”的版本，大概会是这样：
```lua
function make_yin()
    return function(cont)
        io.write("@")
        local yang = make_yang(cont)
        return yang(yang)
    end
end

function make_yang(yin)
    return function(cont)
        io.write("*")
        return yin(cont)
    end
end

local yin = make_yin()
yin(yin)
```

闭包里会嵌越来越多层 `upvalue`。每次执行 `yang(yang)`，都相当于拆开一层闭包；直到某一层里 `cont` 又绑定回 `yin`，`yin(cont)` 就开启了新一轮循环。某种意义上，这也挺符合 CPS 里“continuation 就是闭包”的说法。

## 尾递归

Scheme 和 lua 版本都不会爆调用栈，虽然看起来很奇怪，但两边都能做尾递归优化。

Scheme 最近没怎么碰，所以这里就看一下 lua 版本编译出来的 bytecode：

先把前面的两个 `make` 函数代换掉：

```lua
function yin(cont)
    io.write("@")

    local yang = function(cont2)
        io.write("*")
        return cont(cont2)
    end

    return yang(yang)
end

yin(yin)
```

`luac -l` 的结果如下：
```
main <test.lua:0,0> (6 instructions at 0x6425989036d0)
0+ params, 2 slots, 1 upvalue, 0 locals, 1 constant, 1 function
        1       [10]    CLOSURE         0 0     ; 0x642598903960
        2       [1]     SETTABUP        0 -1 0  ; _ENV "yin"
        3       [12]    GETTABUP        0 0 -1  ; _ENV "yin"
        4       [12]    GETTABUP        1 0 -1  ; _ENV "yin"
        5       [12]    CALL            0 2 1
        6       [12]    RETURN          0 1

function <test.lua:1,10> (10 instructions at 0x642598903960)
1 param, 4 slots, 1 upvalue, 2 locals, 3 constants, 1 function
        1       [2]     GETTABUP        1 0 -1  ; _ENV "io"
        2       [2]     GETTABLE        1 1 -2  ; "write"
        3       [2]     LOADK           2 -3    ; "@"
        4       [2]     CALL            1 2 1
        5       [7]     CLOSURE         1 0     ; 0x642598903eb0
        6       [9]     MOVE            2 1
        7       [9]     MOVE            3 1
        8       [9]     TAILCALL        2 2 0
        9       [9]     RETURN          2 0
        10      [10]    RETURN          0 1

function <test.lua:4,7> (9 instructions at 0x642598903eb0)
1 param, 3 slots, 2 upvalues, 1 local, 3 constants, 0 functions
        1       [5]     GETTABUP        1 0 -1  ; _ENV "io"
        2       [5]     GETTABLE        1 1 -2  ; "write"
        3       [5]     LOADK           2 -3    ; "*"
        4       [5]     CALL            1 2 1
        5       [6]     GETUPVAL        1 1     ; cont
        6       [6]     MOVE            2 0
        7       [6]     TAILCALL        1 2 0
        8       [6]     RETURN          1 0
        9       [7]     RETURN          0 1
```

这里只有 `yin(yin)` 和 `io.write` 是普通 `CALL`，而 `cont(cont2)` 和 `yang(yang)` 都已经变成了 `TAILCALL`。另外也能看出来，`yin` 每执行一次都会生成一个新的闭包，不过至少它只会消耗堆内存，而不是爆栈。
