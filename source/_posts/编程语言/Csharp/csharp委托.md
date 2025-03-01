---
title: csharp委托
date: 2025-02-27 22:37:01
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---

# 委托基本概念
- **定义**：委托是**类型安全的函数指针**，通过 `delegate` 关键字声明。（ 委托就是一个用来装函数的类的类型）
- **作用**：允许将方法作为参数传递、动态调用多个方法。
- **核心特点**：
    - 类型安全（编译时检查参数和返回值）。
    - 支持多播（组合多个方法）。
    - 可用于异步编程。


# **委托的声明与使用**
写在哪里？
可以申明在namespace和class语句块中
更多的写在**namespace**中

 委托常用在：
1. 作为类的成员
2. 作为函数的参数

**声明委托类型**：
```cs
// 定义委托类型，指定方法签名
// MathOperation是一个委托类型，只能引用接受两个 int 参数并返回 string 的方法。
public delegate string MathOperation(int a, int b);
```

**实例化委托**：
eg1：
```cs
// 绑定到具体方法
MathOperation add = (a, b) => a + b;
MathOperation multiply = (a, b) => a * b;

// 调用委托
int result = add(3, 5); // 输出 8
```
eg2：
```cs
public delegate int Fun(int a);

static int MyFun1(int a) ...
static int MyFun2(int a) ...

Fun f = MyFun1;
int tmp = f.Invoke(666);
int tmp = f(666);
```

# **多播委托（Multicast Delegate）**
- **功能**：一个委托实例可绑定多个方法，按顺序执行。
- **操作符**：
    - `+=` 添加方法。
    - `-=` 移除方法。

```cs
MathOperation add = (a, b) => a + b;
MathOperation multiply = (a, b) => a * b;
MathOperation operations = add;
operations += multiply;

// 调用时会依次执行 add 和 multiply
int finalResult = operations(3, 5); // 返回 multiply 的结果 15（最后一个方法的返回值）
```
- **注意**：返回值通常只保留最后一个方法的返回值，中间结果可能被覆盖。
以下是一个示例，演示了如何获取多播委托每一个函数的返回值：
```csharp
using System;
public delegate int MyDelegate();
class Program
{
    static void Main()
    {
        MyDelegate myDelegate = Method1;
        myDelegate += Method2;
        myDelegate += Method3;
        // 获取每一个函数的返回值
        Delegate[] delegates = myDelegate.GetInvocationList();
        foreach (var del in delegates)
        {
            MyDelegate singleDelegate = (MyDelegate)del;
            int result = singleDelegate();
            Console.WriteLine($"Method returned: {result}");
        }
    }
    static int Method1()
    {
        Console.WriteLine("Method1");
        return 1;
    }
    static int Method2()
    {
        Console.WriteLine("Method2");
        return 2;
    }
    static int Method3()
    {
        Console.WriteLine("Method3");
        return 3;
    }
}
```


# **内置泛型委托**
**`Action`**：无返回值的方法，最多支持 16 个参数。
```cs
Action<string> log = message => Console.WriteLine(message);
log("Hello, Action!");
```

**`Func`**：有返回值的方法，最后一个类型参数为返回类型。
```cs
Func<int, int, string> formatSum = (a, b) => $"{a + b}";
Console.WriteLine(formatSum(3, 5)); // 输出 "8"
```

# **匿名方法**
**匿名函数的使用主要是配合委托和事件进行使用**
何时使用？
1. 函数中传递委托参数时
2. 委托或事件赋值时

缺点?
1. 不能删的具体只能无脑null

eg：
```csharp
 class Test
{
    public void Fun1(Action action)
    {
        Console.WriteLine("需要委托作为参数的函数,使用函数更加方便");
        action();
    }
    public Action GetFun()
    {
        return delegate ()
        {
            Console.WriteLine("匿名函数常用作返回值");
        };
    }
}
class Program
{
    static void Main(string[] args)
    {
	    // 匿名函数给委托赋值
        Action a = delegate ()
        {
            Console.WriteLine("匿名函数逻辑");
        };
        Test t = new Test();
        t.Fun1(a); 
        t.GetFun()();
        Func<int, int> b = delegate (int a)
        {
            Console.WriteLine("匿名函数的返回值直接返回就行");
            return a;
        };
    }
}
```

# {% post_link csharp事件 %}
# {% post_link csharp的Lambda表达式 %}

# {% post_link 泛型委托和泛型接口の协变和逆变 %}

