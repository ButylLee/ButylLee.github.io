---
title: ScopeGuard原理介绍与实现
description: 介绍由RAII引申出的通用类ScopeGuard
date: 2021-05-13 23:37:00 +0800
categories: [C++]
tags: [ScopeGuard, RAII]
---

先贴个完整代码的 Github 地址：[ScopeGuard](https://github.com/ButylLee/ScopeGuard)，欢迎Star。

本来是想先彻底调查清楚各方面的资料再提笔的，但是这样估计就无限搁置了，索性先写了。

## 前言

如果你曾经写过跟各种系统API打交道的程序，或者是在程序里使用了大量的指针来管理堆内存，那么难免要写一大堆在资源使用结束后将其释放的语句（如释放内存、关闭句柄），程序顺序执行比较多的尚且能理清楚，但如果有各种条件分支，各种函数嵌套满天飞的话，各种资源的生存期就容易混乱。

典型的例子就是 C/C++ 里被人诟病最多的指针，一直被吐槽使用指针太过危险，程序员稍有疏忽便是各种内存泄漏，一句 delete 下去程序就炸了，事实上也确实如此，C 编程以及 C 样式的 C++ 编程中最常出现的 bug 就是内存泄漏，比如申请了内存没有跟指针绑定啦，绑定了却忘了释放啦，释放完了忘记置空指针啦，忘了已经释放过又释放了一次啦……最主要还是因为出现这类错误并不会让你的程序立即崩溃，而是会在你不知道的某个时候突然 crash，这就搞得程序很难调试了，往往找了半天还定位不出问题。除了内存之外还有如 Windows 下的句柄等程序资源需要我们管理，在这里我统称为资源，即 Resources。

资源管理向来都是 Native Language 的一个大问题，各种资源需要程序员手动申请、手动释放，程序一复杂就容易出 bug，所以才有了托管语言（如 Java、C#）里 Garbage Collection（简称GC）的出现，程序里随便 new 对象，虚拟机/运行时来帮你擦屁股，其设计意图不言而喻，”内存管理太重要啦，不能交给程序员来做“，但另一方面，用 Native Language 的人又说 “内存管理太重要啦，不能交给机器来做”。那既然不能舍弃 Native 开发，有没有什么办法来解决这类问题，减轻程序员的负担呢？

## 智能指针与RAII

在开始之前先稍微地列举一下使用裸指针的管理难处：

比如你最开始写了这么一段代码：

```c++
void* data = malloc(size);
if (xxx){
    // do something
}
else{
    // do other things
}

free(data);
```

很常见，对吧？但是如果逻辑一多，malloc 和 free 就会相隔很远，难以注意到它们的对应关系。真正麻烦的是如果在代码逻辑中途返回了（假设这段代码在某个函数里），那在 return 之前必须记得先 free 掉内存，你可能会说这只要多费点心思就行了嘛，那如果逻辑有几十上百行，如果条件判断、循环层层嵌套呢？你需要在每个 return 的位置前面写上 free。更不要提还有异常的存在，如果抛异常了，中途的 return、free 都不会被执行，这又得怎么处理呢？你当然可以写 try-catch，但是如果你调用的函数抛出了你不知道的异常又该如何呢？有些函数里调用别的函数层层嵌套完全是有可能的，这种时候你对可能抛出的异常无从知晓，就算搞清楚了，要撰写的 catch 也是一大令人头疼的问题。这样的代码写起来要非常小心，修改的时候也要非常小心，假如某段代码需要你很小心，那可以肯定你一定会犯错，写出不小心的代码来（墨菲定律直呼内行）。

有些老旧的 C/C++ 代码会用折衷的方法来解决，比如使用 `do-while(false)`：

```c++
void* data = malloc(size);
do {
    if (xxx) {
        break;
    }
    xxx // 正常操作
    if (xxx) {
        break;
    }

    xxx // 正常操作
} while(false);

free(data);
return 0;
```

甚至使用 goto：

```c++
void* data = malloc(size);
if (xxx) {
    goto fail;
}

if (xxx) {
    goto fail;
}

xxx // 正常操作
return 0;

fail:
    free(data);
    return 1;
```

这些写法可以缓解一部分问题，甚至 `do-while(false)` 在 C 语言里都成为了一种惯用法，但是不够通用，也不能解决 C++ 中的异常，你当然可以全局关闭异常，new 失败不抛异常，但都2021年了，除非你去 Google 上班，你还能不使用异常吗？

不满足于上课那一套的同学肯定知道在 C++ 里除了原生指针之外还有智能指针的存在，[智能指针](https://zh.cppreference.com/w/cpp/memory)（Smart Pointer）是标准库提供的用于智能管理内存的一套设施，什么意思呢？就是说像平常一样 new 一块内存之后把地址赋给智能指针，该怎么用还怎么用，用完之后就不用管它了，时候到了它自己会释放掉内存的。

如此神奇的效果是怎么做到的呢？这就要提到一个惯用法/范式/编程技术，叫做 Resource Acquisition Is Initialization，资源获取即是初始化，简称 RAII，是 C++ 它爹 Bjarne Stroustrup 提出来的。这一句话四个单词是什么意思呢？Resource 就是资源，跟前面的定义是一致的；Acquisition 是获取资源的行为，或者说申请，比如申请一块内存，获取相关的句柄；Initialization 是初始化，指类的初始化，即构造；Is 表明你获取了资源后要马上把它关联到一个类里。

我们知道在 C++ 中的局部变量是有[生存期](https://zh.cppreference.com/w/cpp/language/storage_duration)的，生存期内的代码范围称为变量的作用域，一个变量出了作用域就会解除分配，而如果这个变量的类型是类的话，我们知道它的析构函数就会被调用。那么巧思就来了，我在堆上手动申请手动释放的内存，和保存其地址的在栈上的局部变量的指针，如果他们能关联到一块，指针离开其作用域的时候自己执行析构函数把内存给释放了，岂不美哉？更重要的是，即使发生了异常，在抛出异常的时候也能保证释放语句得到执行，因为异常抛出时的[栈回溯](https://zh.cppreference.com/w/cpp/language/throw)保证具有[自动储存期](https://zh.cppreference.com/w/cpp/language/storage_duration)的对象得以析构，使得抛异常时资源也能被自动释放。这就是 RAII 的含义，这意味着我们应该用类来封装和管理资源，其他的资源也可以用这个方法做到自动释放，其本质就是将运行环境所给资源的生存周期与一个对象的生存期相绑定。

因为内存管理是最为常见的资源管理，所以 C++ 的标准库就内置了智能指针，在 C++11 之前是 `auto_ptr` ，从 C++11 开始变成了 `unique_ptr` 、`shared_ptr` 、`weak_ptr` 三种智能指针以区分所有权。基于 RAII 的设施还有用于多线程中给数据加锁的 `mutex`。

## 通用资源管理类——ScopeGuard

到此为止一切都很完美，不过，如果我有其他的资源要用，怎么办？Windows 下有各种句柄，绘图时可能有各种 Drawer，以及其他的各种系统资源要管理，难道我要为了一个 `CloseHandle()` 、`ReleaseXX()` 、`endDraw()` 等等而大张旗鼓地写出一个类来吗？其实不必，因为大部分的资源管理都很简单，通常只需要对应的释放语句就行了，所以我们可以写一个通用的管理类，每次传入相应的语句便可以达到效果。理想的使用范式应该是类似于下面这样的：

```c++
void foo()
{
    HANDLE handle = CreateFile(xxx);
    ON_SCOPE_EXIT{
        CloseHandle(handle);
    }

    xxx // use the file
}
```

在申请资源之后写上退出作用域时要执行的语句，使用完之后就不用管它了，无论是函数返回还是抛出异常，指定的语句都会被执行，就像使用智能指针一样。

实际上这样的设施早就有人提出来了，早在2000年，Andrei Alexandrescu 就在 DDJ 杂志上发表了[一篇文章](https://www.drdobbs.com/cpp/generic-change-the-way-you-write-excepti/184403758)，提出了这个叫做 ScopeGuard 的设施，按字面意思很好理解，就是在作用域处安排一个 Guard，越过作用域就帮你把资源给释放了。但是当时C++还没有太好的语言机制来支持这个设施，所以 Andrei 动用了你所能想到的各种奇技淫巧硬是造了一个出来，后来 Boost 也加入了 [ScopeExit 库](http://www.boost.org/libs/scope_exit/)，不过这些都是建立在 C++98 不完备的语言机制的情况下，所以其实现非常不必要的繁琐和不完美。

这里插几句题外话，资源管理始终是所有语言的一个共同问题，那么其他语言里有没有类似上面 `ON_SCOPE_EXIT` 的东西呢？答案是肯定的，在 Swift、Go、D 语言中内置的 `defer` 关键字可以指定在退出当前作用域时要执行的函数，在 C#、Java 中有 `try-finally{}` 块，finally 中的语句会被自动执行。

其实如果你仔细看 [cppreference](https://zh.cppreference.com/w/%E9%A6%96%E9%A1%B5) 的首页，你就会发现在技术规范的下面，标准库扩展 v3 里就有 [scope_exit](https://zh.cppreference.com/w/cpp/experimental/lib_extensions_3) 的类似设施：

![sg_in_ts](/assets/img/2021/sg_in_ts.png)

但是技术规范只是实验性质出版的，主流编译器根本就没有实现，那什么时候标准库里会有 ScopeGuard 设施呢？猴年马月就有了（逃

所以最好的办法就是自己来造这个轮子，自己写一个ScopeGuard。

## ScopeGuard原理

那么，ScopeGuard应该如何设计呢？C++11 发布之后，很多人都开始重新实现这个对于异常安全来说极其重要的设施，不过绝大多数人的实现受到了2000年 Andrei 的原始文章的影响，多多少少还是有不必要的复杂性。实际上我们只需要达到两个目的：能够储存一个函数以便在析构的时候执行，在函数里能访问含有资源信息的变量（如指针、HANDLE）。在 C++11前写会很别扭，但是在 C++11 中引入了 [lambda](https://zh.cppreference.com/w/cpp/language/lambda) 表达式和一个用于保存函数的标准库容器 std::function 之后，所有事情都立马变得简单起来了。

## 无脑实现ScopeGuard

将 C++11 中的 lambda 和 std::function 结合起来，这个设施可以简化到脑残的地步：

```c++
class ScopeGuard
{
public:
    explicit ScopeGuard(std::function<void()> callback)
        :m_callback(callback)
    {}
    ~ScopeGuard()
    {
        if (m_active)
            m_callback();
    }

    void dismiss()
    {
        m_active = false;
    }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
private:
    std::function<void()> m_callback;
    bool m_active = true;
};
```

这个类要怎么使用呢，其实很简单，你只要在构造时交给它一个无参无返回值的函数，它就能把函数隐式转换为 std::function<void()>，它负责在析构的时候调用。大部分时候你给它的函数就是 lambda，比如：

```c++
HANDLE handle = CreateFile(xxx);
ScopeGuard onExit([&] { CloseHandle(handle); });
// do something...
```

lambda 会以引用方式捕获 handle 变量，从而传递给 CloseHandle。至于 sg 为什么没有拷贝构造函数，这点是显而易见的。

然后那个 `dismiss()` 和 `m_active` 是什么东西呢？这个其实是 Andrei 的原始设计的一部分，其作用是为了支持 rollback 模式：

```c++
ScopeGuard onFailure([&] { /* rollback */ });
// do something that may fail
onFailure.dismiss();
```

在 “do something" 的过程中只要抛出了任何异常，rollback 的语句都会被执行。如果 “do something“ 成功了，就调用 `dismiss()`，使 `m_active` 为 false，rollback 就不会被执行了，这类似于 C#、Java 中的 `try-finally{}` 块。

观察第一个示例片段，可以看出资源的申请和释放是相邻的，这非常符合人们的编写思维，而且只需要写一次，永远不会忘记，可以说解决了资源管理的重大问题。

但是，应用到实际项目中的话，很快就会发现，如果我同时有很多个资源要管理，不仅要手动定义 ScopeGuard 的变量，还要自己起名字，能不能再简单点呢？甚至更进一步，做到像前面提到的其他语言里的 defer 关键字一样呢？利用宏就可以了。

## 完善ScopeGuard

**第一个问题**其实并不难解决：

```c++
#define SG_LINE_NAME_CAT(name,line) name##line
#define SG_LINE_NAME(name,line) SG_LINE_NAME_CAT(name,line)

#define SCOPEGUARD(callback) \
        ScopeGuard SG_LINE_NAME(SCOPEGUARD_,__LINE__) (callback)
```

SCOPEGUARD 宏会展开成定义变量的语句，并且由当前所在行数自动生成变量名，这样只要直接写 lambda 就行了：

```c++
HANDLE handle = CreateFile(xxx);
SCOPEGUARD([&] { CloseHandle(handle); });
```

简单解释一下吧：

- `\` 表示将下一行的内容合并到上一行来，因为宏定义是在行尾结束的，所以宏过长时可以用 `\` 分割成多行。

- `__LINE__` 是一个预定义的宏，它会展开成当前代码的行数。

- 宏 SG_LINE_NAME 的作用是使用 `##` 命令把两个参数 name 和 line 直接连在一起，相关知识可以看[这里](https://zh.cppreference.com/w/cpp/preprocessor/replace)。至于为什么要套两层，那是因为宏在第一遍展开时并不会展开参数的内容，如果只套一层的话就会展开成 `SCOPEGUARD___LINE__` ，不是我们想要的结果。（这应该也算一种惯用法）

**第二个问题**需要一点小心思，我们再次观察其他语言中的 defer：

```c++
{
    defer {
        // do thing2
    }
    // do thing1
}
```

结果是先执行 thing1 再执行 thing2。

我们要使 { } 里能填语句，并且当前位置会生成一个 sg，那么能在函数里写函数定义的就只有 lambda 了，所以 defer 后面的 { } 就是 lambda 的定义，但是要把 lambda 传给我们的类里面只能用函数啊，那就必须写在圆括号里了，似乎不得不在 { } 最后加上一个 `)`，但是不要忘记，C++ 里可以重载运算符！

于是乎，我们可以重载 `operator+`，达到上述的目的：

```c++
#define ON_SCOPE_EXIT \
        ScopeGuard SG_LINE_NAME(OnScopeExit_Block_,__LINE__) = \
        eOnScopeExit() + [&]() -> void

enum eOnScopeExit {};

ScopeGuard operator+(eOnScopeExit, std::function<void()> callback)
{
    return ScopeGuard(callback);
}

class ScopeGuard
{
    friend ScopeGuard operator+(eOnScopeExit, std::function<void()>);
    // ...
};
```

上面的枚举类型 `eOnScopeExit` 是辅助 `operator+` 的，由于我们用不到它，所以声明为无名形参。这样就可以在 `ON_SCOPE_EXIT` 宏后面写 { } 了，不过要带个尾巴 `;`。

```c++
{
    ON_SCOPE_EXIT {
        // do thing2
    };
    // do thing1
}
```

Neat！

至此，我们有三种使用 ScopeGuard 的方式：

- `ScopeGuard sg(callback);` 或 `ScopeGuard sg = callback;`

- `SCOPEGUARD(callback);`

- ```c++
    ON_SCOPE_EXIT {
        // releasing resources
    };
    ```

要注意到第一种方式依然是有必要的，因为我们需要知道变量的名字才能调用 `dismiss()`，但是大部分情况下都是使用后两种用法。

大致的实现如下，不过要注意还有很多细节我没有补充，完整实现请看 Github 上的[源码](https://github.com/ButylLee/ScopeGuard/blob/master/ScopeGuard/ScopeGuard.h#L169)的第170行。

```c++
#define SG_LINE_NAME_CAT(name,line) name##line
#define SG_LINE_NAME(name,line) SG_LINE_NAME_CAT(name,line)

#define SCOPEGUARD(callback) \
        ScopeGuard SG_LINE_NAME(SCOPEGUARD_,__LINE__) (callback)
#define ON_SCOPE_EXIT \
        ScopeGuard SG_LINE_NAME(OnScopeExit_Block_,__LINE__) = \
        eOnScopeExit() + [&]() -> void

enum eOnScopeExit {};

ScopeGuard operator+(eOnScopeExit, std::function<void()> callback)
{
    return ScopeGuard(callback);
}

class ScopeGuard
{
    friend ScopeGuard operator+(eOnScopeExit, std::function<void()>);
public:
    explicit ScopeGuard(std::function<void()> callback)
        :m_callback(callback)
    {}
    ~ScopeGuard()
    {
        if (m_active)
            m_callback();
    }

    void dismiss()
    {
        m_active = false;
    }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
private:
    std::function<void()> m_callback;
    bool m_active = true;
};
```

但是，这已经结束了吗？并没有！

## 优化ScopeGuard——使用模板实现

*注意：阅读本节需要对 C++ 模板有基本的了解*

在此之前，一切看起来都很美好，直到你 `sizeof` 了一下 ScopeGuard……

```c++
int main()
{
    // in MSVC
    std::cout << "the size of ScopeGuard is: " << sizeof(ScopeGuard) << "bytes";
}
```

然后……

```
the size of ScopeGuard is: 48bytes
```

发生了什么？感到不对劲的你在 x64 下又编译了一次：

```
the size of ScopeGuard is: 72bytes
```

坑爹呢这是！

“万恶的 MSVC！” 你这样想着，然后换了 gcc 编译，不过仍然有20字节的大小，在64位下则为40字节，占用还是挺大的。

其实，std::function 储存函数时是有小对象优化的，如果函数占用比较小就直接存在栈中，过大了才使用堆内存分配，通过阅读标准库头文件可得知，MSVC 预留了9个指针的大小，而 gcc 预留的空间只有3个指针的大小，所以显得 MSVC 里的占用很大。不过对于我们的 ScopeGuard 来说，就算是16字节的大小似乎也有点浪费了，因为其实储存函数只需要一个指针就够了（对原生函数来说），所以我们其实可以自己来储存函数。

### 可调用对象的种类

那么在储存函数之前，我们有必要搞清楚在 C++ 里函数都有哪几种，说得正式一点就是支持函数调用运算符 `operator()` 的对象，这里称呼为可调用对象。

那么从语言层面上讲，我们有四种可调用对象：（原生）函数、函数指针、重载了 `operaotr()` 的类、lambda 表达式。虽然 lambda 表达式产生的也是一个可调用的类，不过它比较特殊所以单独列出来。为什么要说在语言层面上呢？因为诸如 std::bind 和 std::function 这样的东西其实也属于上面四种之一，所以就不单独考虑了。

那么，该如何 ”保存“ 一个函数以便调用呢？众所周知，函数本身作为参数传递的话是会退化（decay）成对应的函数指针的，如 `int(int, bool)` 就会退化成 `int(*)(int, bool)`；而如果是函数指针的话，我们直接保存一个函数指针就行了；而对于类，我们既可以 copy 它也可以引用它。

### 使用typename T接收函数

搞清楚函数的种类之后，我们就可以开始了，先写好框架：

啪的一下，很快啊！

```c++
template<typename T>
class ScopeGuard
{
public:
    explicit ScopeGuard(T callback)
        :m_callback(callback)
    {}
    ~ScopeGuard()
    {
        if (m_active)
            m_callback();
    }

    void dismiss()
    {
        m_active = false;
    }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
private:
    T m_callback;
    bool m_active = true;
};
```

嗯，看起来不错，但是我们要怎么初始化一个 sg 呢？像 std::function 那样？

```c++
void foo()
{
    std::cout << "foo" << std::endl;
}
int main()
{
    ScopeGuard<void()> sg(foo);
}
```

然而这是编译不了的，提示无法获取函数类型，这时我们想到前面提到过的，函数会退化成函数指针的事情，于是我们这样写：

```c++
ScopeGuard<void(*)()> sg(foo);
```

这次可以通过编译了，更大的问题还在后面，先不说这样写繁不繁琐，如果要使用 lambda，马上就歇逼了，因为你无法写出 lambda 的类型！因为每一个 lambda 的类型都是独一无二的，所以它只能由编译器指定。如果你说 lambda 可以转换成函数指针啊，请不要忘记捕获了变量的 lambda 是无法转换成函数指针的。

等会，不是还有 decltype 嘛！嗯，你说得对，但是这样你就得先把 lambda 赋给一个具名变量，然后才可以推导类型，要稍微多打些不可避免的字。

更好的解决办法是，我们想到在函数模板里可以通过实参推导形参类型，那么我们可以通过一个辅助函数来初始化 ScopeGuard：

```c++
template<typename T>
class ScopeGuard; // 声明类
template<typename T>
ScopeGuard<T> MakeScopeGuard(T&& callback)
{
    return ScopeGuard<T>(std::forward<T>(callback));
}
```

这里我们使用了万能引用（T&&）以及完美转发（std::forward），具体细节就不累述了，请自行查阅相关资料。

接着我们添加友元，并且把构造函数设置为 private，因为加上前面的重载 `operator+` 的办法，我们就不用显式地声明变量类型和直接使用构造函数了：

```c++
template<typename T>
class ScopeGuard
{
    friend ScopeGuard<T> MakeScopeGuard<T>(T&&);
private:
    explicit ScopeGuard(T&& callback) // 仅限友元访问
        :m_callback(std::forward<T>(callback)) // 由于前面使用了完美转发，这里也一样使用
    {}
public:
    ScopeGuard(ScopeGuard&& other)
        : m_callback(std::move(other.m_callback))
        , m_active(std::move(other.m_active))
    {
        other.dismiss();
    }
    // ...
};
```

这里要解释两点：

- 友元的声明里要在函数名后面指定模板实参，即要写实例化后的函数声明，这样编译器才知道你指定哪个具体函数为友元。
- 这里还添加了移动构造函数，因为在 C++17 前，返回值优化 RVO（Return Value Optimization）是作为一个优化来对待的，因此需要存在复制或移动构造函数，如果两者都没有的话那么 MakeScopeGuard 是无法通过编译的。虽然 m_active 是个内建类型用不用 std::move 都一样，但是这里为了语义一致，就都使用 std::move。

然后可以这样用：

```c++
auto sg1 = MakeScopeGuard(foo);
auto sg2 = MakeScopeGuard([] {std::cout << "lambda" << std::endl;});
```

但是现在我们的 typename T 可以接受任何类型，接下来我们要对它的实参作出限制。

### 限制模板实参类型

要限制传入的类型模板实参为我们想要的函数类型（无参无返回值），就要先能判断实参是不是符合要求。我在实现 ScopeGuard 的时候参考了 Github 上 Star 最高的[实现](https://github.com/ricab/scope_guard/blob/master/scope_guard.hpp)，然而我发现其实它的实现过于繁琐，虽然声称用到了 C++17 的特性，但是在限制参数上却还是 C++98 的写法，竟然足足用了几十行，其实只要三行就够了：

```c++
#include <type_traits>
template<typename T>
constexpr bool is_proper_callback_v
    = std::is_same_v<void, decltype(std::declval<T>()())>;
```

这写的都是些什么玩意儿？别急，让我一一道来：

1. 第一行 include 了一个头文件 type_traits，这个头文件包含了一些可在编译期使用的关于[类型特性](https://zh.cppreference.com/w/cpp/types#.E7.B1.BB.E5.9E.8B.E7.89.B9.E6.80.A7.28C.2B.2B11_.E8.B5.B7.29)的元函数，我们会用到其中的一些。
2. 第二行声明了类型模板形参 T 作为输入
3. 在 C++14 中引入了变量模板，这使得我们不用写出繁琐的、借助类模板声明的编译期常量的代码；`constexpr` 说明符指定该变量为编译期常量，它的值会在编译期确定下来（在编译期求值），这种特性在后面会用到；`bool` 说明符指定变量 is_proper_callback_v 的类型为 bool，如果 T 符合要求则为 true，否则为 false。
4. 推断的基本思路是，先推断其能否被无参调用，再推断其返回值是否为 `void`。我们先看外层的 `std::is_same_v`，它的作用是如果两个模板形参的类型是一样的，那它就为 true，否则为 false。而里面的第二个位置的 `decltype` 是一个用于推断表达式类型的关键字，如果是 `decltype(T)` 那么就表示推断 T 的类型，如果写 `decltype(T())` 就表示推断 T 在被无参调用后的返回值。那么 std::declval 又是什么东西呢？如果传进的模板实参是一个重载了 `operator()` 的类的话，那么 `T()` 表示的是一个默认初始化的匿名对象，而不是调用 `operaotr()`，这需要括号的左边是一个变量而不是一个类型，std::declval 能把 T 转换为纯右值，使其在语义上能够直接使用类的成员函数，用法就是 `std::declval<T>()`，后面再加一对括号表示无参函数的调用。`std::is_same_v` 和 `std::declval` 的具体原理在此就不详述了。

现在我们使用 is_proper_callback_v 就可以确定 T 是不是我们想要的函数类型了，但是怎么应用到 ScopeGuard 上呢？

std::enable_if 是一个很实用的元函数，它的作用是根据传入的第一个模板实参的真假来决定是否启用当前模板的实例，这里同样不详述具体原理，请自行查阅相关资料。std::enable_if 有很多种用法，这里我们选择较简洁的一种：

```c++
// ScopeGuard的前置声明
template<typename T, typename = std::enable_if<is_proper_callback_v<T>>::type>
class ScopeGuard;
```

这里我们修改的是前面代码中 ScopeGuard 在 MakeScopeGuard 前的声明，添加的第二个模板形参是一个匿名的类型模板形参，它具有默认值，如果 T 是我们想要的类型的话，第二个模板形参就是后面的 `::type`，也就是 `void`，反之如果不符合 is_proper_callback_v 的要求，那么就没有 `::type` 类型，编译失败。

需要注意的是，原来的 ScopeGuard 有两部分，前一部分是声明，后一部分是定义，修改完声明后，按道理应该也修改定义部分的模板参数，但是其实可以不用，我们把声明的部分当成主模板，定义的部分当成偏特化后的模板，这样定义就变得简洁了：

```c++
template<typename T, typename = std::enable_if_t<is_proper_callback_v<T>>>
class ScopeGuard; // 无定义
template<typename T>
class ScopeGuard<T> // 偏特化的模板
{
    // ...
}
```

这样两个部分就不是声明和定义的关系了，而是主模板和偏特化模板的关系，偏特化的第二个参数就是默认参数得来的，而如果不满足 std::enable_if 的要求，那么就选择主模板，这里我们也没有必要定义，因为没有用到，就直接使用声明。

如果你不了解模板偏特化的话，请查阅相关资料。

std::enable_if_t 是 std::enable_if\<T\>::type 的别名模板，可以简化写法；而类似地，std::is_same_v 是 std::is_same\<T\>::value 的变量模板缩写，is_proper_callback_v 的命名也遵循此规范。不过由于 std::is_same_v 是 C++17 才加入的，所以在最终代码中我还是用 std::is_same。

还记得前面 is_proper_callback_v 的定义吗？如果没有用 constexpr 修饰的话，这里就不能通过编译了，因为它的值是运行期确定的，所以必须加上 constexpr。类似地，我们现在把能加上 constexpr 的都加上（函数也能被修饰），目前我们的成果是这样的：

```c++
#define SG_LINE_NAME_CAT(name,line) name##line
#define SG_LINE_NAME(name,line) SG_LINE_NAME_CAT(name,line)

#define SCOPEGUARD(callback) \
        auto SG_LINE_NAME(SCOPEGUARD_,__LINE__) = MakeScopeGuard(callback)
#define ON_SCOPE_EXIT \
        auto SG_LINE_NAME(OnScopeExit_Block_,__LINE__) = \
        eOnScopeExit() + [&]() noexcept ->void

template<typename T>
constexpr bool is_proper_callback_v
    = std::is_same<void, decltype(std::declval<T>()())>::value;

template<typename T, typename = std::enable_if_t<is_proper_callback_v<T>>>
class ScopeGuard;

enum class eOnScopeExit {};

template<typename T>
constexpr ScopeGuard<T> operator+(eOnScopeExit, T&& callback)
{
    return ScopeGuard<T>(std::forward<T>(callback));
}

template<typename T>
constexpr ScopeGuard<T> MakeScopeGuard(T&& callback)
{
    return ScopeGuard<T>(std::forward<T>(callback));
}
template<typename T>
class ScopeGuard<T>
{
    friend constexpr ScopeGuard<T> MakeScopeGuard<T>(T&&);
    friend constexpr ScopeGuard<T> operator+<T>(eOnScopeExit, T&&);
private:
    explicit ScopeGuard(T&& callback)
        :m_callback(std::forward<T>(callback))
    {}
public:
    ~ScopeGuard()
    {
        if (m_active)
            m_callback();
    }
    ScopeGuard(ScopeGuard&& other)
        : m_callback(std::move(other.m_callback))
        , m_active(std::move(other.m_active))
    {
        other.dismiss();
    }

    void dismiss()
    {
        m_active = false;
    }

    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
private:
    T m_callback;
    bool m_active = true;
};
```

注意我们现在只能通过辅助函数来初始化 ScopeGuard，因此辅助宏的定义里都改用 auto 申明变量。

这看起来很完美，不是么？

### 一个难以发现的问题——更加深入地思考

回想起最初的实现，我们使用到了 std::function，我们再来看一眼它的[参考](https://zh.cppreference.com/w/cpp/utility/functional/function)说明：

> 类模板 std::function 是通用多态函数封装器。std::function 的实例能存储、复制及调用任何可调用（Callable）目标——函数、lambda 表达式、bind 表达式或其他函数对象，还有指向成员函数指针和指向数据成员指针。

嗯......函数封装器......储存......复制......调用......等会，复制？看起来 std::function 的效果是将函数对象复制了一遍，而且我们也找不到其他的描述字眼。再看看我们的实现，马上就会意识到它在类型的推导过程中与 std::function 有着微妙的区别（事实上是很大的区别），我们先给出测试的结果：

```c++
using namespace std;
void func() // 原生函数
{
    cout << "func" << endl;
}
using func_type = void(*)();
func_type func_pointer = &func; // 函数指针
struct functor_struct
{
    void operator()()
    {
        cout << "functor" << endl;
    }
}functor; // 函数对象
auto lambda = [] { cout << "lambda" << endl; }; // lambda

SCOPEGUARD(func);          // T被推导为void (&)()，callback类型为void(*)()
SCOPEGUARD(func_pointer);  // T被推导为void (*&)()，callback类型为void (*&)()
SCOPEGUARD(functor);       // T被推导为functor_struct&，callback类型为functor_struct&
SCOPEGUARD(lambda);        // T被推导为void <lambda>&()，callback类型为void <lambda>&()
```

需要注意到的是，在传入原生函数的例子中，T 与 callback 的类型是不一致的，这里发生了 decay，因为储存原生函数只能使用指针。而剩下的三个例子中类型都为引用。

且慢！还有一些不容易想到的例子被我们忽略了，上面用于测试的后三个函数对象都是变量，这意味着传进 MakeScopeGuard 的时候它们都是左值，我们忘了测试右值的情况：

```c++
SCOPEGUARD(&func);                            // T被推导为void(*)()，callback类型为void(*)()
SCOPEGUARD(functor_struct());                 // T被推导为functor_struct，callback类型为functor_struct
SCOPEGUARD([] { cout << "lambda" << endl; }); // T被推导为void <lambda>()，callback类型为void <lambda>()
```

注意，对于原生函数来说是无所谓左右值的，所以我们的右值有三种情况（本来原生函数的测试应该放在这里，但是为了更符合思考的直觉就放在前面），这里的情况都是函数对象原来的类型。

好吧，那这能说明什么呢？你可能会这样问。我要说的是，这样的行为和我们一开始使用 std::function 来实现的行为是不一致的，那么，这里就存在一个问题，即它的使用者可以做出如下的暴行：

```c++
struct functor_struct
{
    void operator()()
    {
        cout << "release " << x << endl;
    }
    void setx(int newx)
    {
        x = newx;
    }
    int x = 0;
};
void func1()
{
    cout << "release A" << endl;
}
void func2()
{
    cout << "release B" << endl;
}
int main()
{
    func_type f = func1;
    functor_struct functor;
    SCOPEGUARD(f);
    SCOPEGUARD(functor);

    f = func2;       // 李在赣神魔？
    functor.setx(1); // 李在赣神魔？
}
```

由于我们储存函数的方式大部分是引用，这意味着交给 ScopeGuard 执行的函数可能会被纂改，或者其状态可能会被改变。在继续深入思考之前，我有必要指出这里还有另外一个问题：函数所储存的变量（资源）应该是值还是引用？

事实上关于这个问题我思考了很长一段时间，因为它决定着我们整个的设计应当是怎样的。我的结论是，对于函数对象内部的状态，即它所储存的变量的方式，应当为引用形式，因为这和我们用手动形式释放资源的形式是相同的；而 ScopeGuard 对函数对象的 ”捕获“ 应当为值形式，这意味着我们需要对推导出的类型 T 进行修改（说得高级点就是变换），并且在 ScopeGuard 里储存函数对象。

另外，如果你阅读过 std::unique_ptr 的源码的话，你会发现智能指针对删除器这一函数对象的处理方式与我们现在是一致的，但是问题在于，自定义智能指针的删除器需要你显式地给出模板实参，即手动指定删除器的类型，这样在类内部就是用户指定的类型，但是我们想让 ScopeGuard 的用法尽可能简洁，所以不能手动指定函数的类型，因此要对类型模板实参进行修改。

想清楚了之后我们就可以对实现进行修改了，事实上并没有想象中的那样需要大刀阔斧地更改：

```c++
template<typename T>
class ScopeGuard
{
    using Callback = std::decay_t<T>;

private:
    explicit ScopeGuard(Callback callback)
        : m_callback(std::move(callback))
    {}
    // ...
private:
    Callback m_callback;
    bool m_active = true;
};
```

没错，只需如此修改就足够了。

std::decay\<T\>的作用是先去除 T 的引用，然后若结果是（原生）函数类型的话，就加上一层指针，否则就去除 [cv限定](https://zh.cppreference.com/w/cpp/language/cv)。这样的结果就是我们想要的，函数对象原来的类型了。

那么我们现在能说实现和 std::function 的表现是相同的了吗？其实还有一点微小的区别，如果你还记得前面说过的，std::function 会优先在栈上储存函数，如果太大了才会使用堆内存来储存的描述的话，那么你马上就会发现，我们的 ScopeGuard 只会在栈上储存，而且它的大小是不固定的，这似乎不太妥当，但这其实恰恰是优点所在。

首先，我在前面说过函数里储存的变量应当是引用类型，这样的话每个变量的占用大小就始终是一个固定值（32位下为4个字节），这就避免了直接在函数里储存很大的变量（资源）；其次，虽然标准没有规定，但是据我实测，在 MSVC、GCC、Clang 下，默认捕获（=或者&）只会捕获你在 lambda 函数体里用到的变量，不过写出带变量名的显式捕获就始终会捕获了，所以推荐的写法就是只写一个&，这样 lambda 里只会有你用到的变量引用。

另外，如果你的资源释放函数真的需要 capture 很多变量，让 ScopeGuard 大到不可接受，那说明你应该改变下你的设计了。

综合考虑下来，在实际应用方面，直接在栈上储存函数是没有问题的。

我在前面提到，在实现 ScopeGuard 的时候参考了 Github 上 ricab 写的实现，但是他限制类型的方式过于繁琐，更重要的就是本节提到的问题，我一开始也是跟着他的思路走，到后面才发现 ricab 设计上的欠缺之处（当然，不排除故意的可能性），并且想了相当长一段时间才理清了头绪，很明显他没有想到这一层面，而且也只提供了一种使用方法。

### 完善细节——做到尽善尽美

为了更好地说明，先把最终的实现（主体部分）放在下面，基本就是从我的实现里直接复制过来了：

```c++
#define SG_BEGIN namespace sg {
#define SG_END   }
#define SG_LINE_NAME_CAT(name,line) name##line
#define SG_LINE_NAME(name,line) SG_LINE_NAME_CAT(name,line)

#define ON_SCOPE_EXIT \
        auto SG_LINE_NAME(OnScopeExit_Block_,__LINE__) = \
        sg::detail::eOnScopeExit() + [&]() noexcept ->void
#define SCOPEGUARD(callback) \
        auto SG_LINE_NAME(SCOPEGUARD_,__LINE__) = sg::MakeScopeGuard(callback)

SG_BEGIN
namespace detail {
    template<typename T>
    constexpr bool is_proper_callback_v
        = std::is_same<void, decltype(std::declval<T>()())>::value;

    template<typename TCallback, typename =
        std::enable_if_t<is_proper_callback_v<TCallback>>>
    class ScopeGuard;

    /* ----------------------------------------------------- */
    enum class eOnScopeExit {};

    template<typename TCallback>
    constexpr ScopeGuard<TCallback> operator+(eOnScopeExit, TCallback&& callback)
    {
        return ScopeGuard<TCallback>(std::forward<TCallback>(callback));
    }

    template<typename TCallback>
    constexpr ScopeGuard<TCallback> MakeScopeGuard(TCallback&& callback)
    {
        return ScopeGuard<TCallback>(std::forward<TCallback>(callback));
    }
    /* ----------------------------------------------------- */

    template<typename TCallback>
    class ScopeGuard<TCallback> final
    {
        using Callback = std::decay_t<TCallback>;

        friend constexpr ScopeGuard<TCallback> operator+<TCallback>(eOnScopeExit, TCallback&&);
        friend constexpr ScopeGuard<TCallback> MakeScopeGuard<TCallback>(TCallback&&);
    private:
        explicit ScopeGuard(Callback callback)
            : m_callback(std::move(callback))
        {}
    public:
        ~ScopeGuard() noexcept
        {
            if (m_active)
                m_callback();
        }

        ScopeGuard(ScopeGuard&& other)
            noexcept(std::is_nothrow_move_constructible_v<Callback>)
            : m_callback(std::move(other.m_callback))
            , m_active(std::move(other.m_active))
        {
            other.dismiss();
        }
        /* No move assign function */

        void dismiss() noexcept
        {
            m_active = false;
        }

        ScopeGuard(const ScopeGuard&) = delete;
        ScopeGuard& operator=(const ScopeGuard&) = delete;
    private:
        Callback m_callback;
        bool m_active = true;
    };
} // namespace detail

using detail::MakeScopeGuard;
SG_END
```

- 为了规范性，我们给整体加上了一层命名空间 sg，但是前两种用法都不用手动写出命名空间来。
- 第39行，我们给类加上 final 以防止被继承，注意前面的声明不用加，原因已经说过了，他们不是同一个模板实例。
- 第78行，为了隐藏细节，对于第三种用法，我们只暴露出 MakeScopeGuard 函数，其他的东西都放在内嵌命名空间 detail 里，而前两种用宏实现则没有必要。这里为什么不直接把函数放在 detail 外面呢，因为这样的话 MSVC 编译会有 bug，导致无法编译，故使用暴露的方法。
- 第63行，我们在前面添加了移动构造函数，而移动赋值函数是没有必要的，因为在同一作用域中把原来的 ScopeGuard 移动给另一个是没有意义的，我们只会在有必要且作用域变化时使用移动操作。
- 同样为了规范，我们给能加的函数都加上 noexcept 说明符，这里就不多解释了，不了解的请查阅相关资料。

这里不得不再吐槽一下 ricab 的实现，由于 C++17 里异常声明（noexcept）成为了函数类型的一部分，因此他在说明文档中说在 C++17 里可以开启强制要求（资源释放的）函数为 noexcept 的要求（在判断函数是否符合要求的那一部分），用意就是因为 ScopeGuard 执行函数的时机是在栈回溯的时期，而栈回溯中是不能抛出异常的（这也是为什么析构函数要求必须为 noexcept），所以可以让使用者必须传一个 noexcept 的函数进去。但是这要求其实并不合理，暂且不论每次写释放资源的函数都要加上 noexcept 麻不麻烦，你释放资源很多时候是要调用系统 API 的，但是大部分 API 存在失败的可能，而且大部分系统 API 使用的都是 C 语言，根本无从知晓是否 noexcept，故以我的观点，noexcept 的强制要求是完全没有必要的。

到目前为止，我们已经讲完了所有的细节，也有可能存在遗漏，无论如何，完整的[实现](https://github.com/ButylLee/ScopeGuard/blob/master/ScopeGuard/ScopeGuard.h)都在 Github 上。

### 与非模板实现的对比

说了那么多，最终用模板实现的版本有什么进步么？当然有了，最明显的一点是，由生成的汇编代码中可以看出，模板实现的版本在开了优化后已经没有了 ScopeGuard 的存在，换言之就是已经优化成语句的直接调用！这就相当于你写了一个 `ON_SCOPE_EXIT`，编译器就自动帮你在所有退出点前加上相应语句！而在非模板实现的版本中，std::function 无法被优化掉，使得在运行期还会有函数捕获的开销，但这不能归咎于 STL，有可能是专门不优化的，不过模板实现的行为正好符合我们的需求，这明显是一个优点。

这里以 MSVC 为例（gcc也是一样的结果）：

![cpp_code_in_MSVC](/assets/img/2021/cpp_code_in_MSVC.png)

![asm_code_in_MSVC](/assets/img/2021/asm_code_in_MSVC.png)

## 结语

至此，本篇文章就结束了。对于我来说，这是第一次尝试去正经地写一个库，第一次应用模板技术（虽然可以说没有），也了解到了新的思想，在这过程中我也体会到了写库的麻烦，不得不去了解很多冷知识，因为库的作者必须考虑各种事，要接触到语言里最边角的细节，从而确保写出来的库足够健壮。

话虽如此，但其实文章的很多东西来自前人的成果，我个人也不过是在前人的基础上进行完善、整合，添砖加瓦而已。写这篇文章的目的只是为了记录和分享知识，希望读完的你也学习到了新知识\^_\^。

---

**参考资料**

1. [ScopeGuard 介绍和实现 - 知乎](https://zhuanlan.zhihu.com/p/21303431)
2. [C++11（及现代C++风格）和快速迭代式开发 - 刘未鹏](http://mindhacks.cn/2012/08/27/modern-cpp-practices/)
3. [ricab/scope_guard - Github](https://github.com/ricab/scope_guard)
