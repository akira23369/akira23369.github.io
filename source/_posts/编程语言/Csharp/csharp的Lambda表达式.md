---
title: csharp的Lambda表达式
date: 2025-02-28 10:38:33
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---

在C#中，**Lambda表达式**是一种简洁的{% post_link csharp委托#匿名方法 匿名函数%}，用于创建委托或表达式树类型。

# **Lambda表达式的基本形式**
Lambda表达式分为两种形式：

**表达式Lambda**  
仅包含单个表达式，无需大括号，自动返回结果。
```cs
(参数列表) => 表达式

Func<int, int> square = x => x * x;
Console.WriteLine(square(5)); // 输出 25
```

**语句块Lambda**  
包含多行语句，需用大括号包裹，显式使用`return`（若有返回值）。
```cs
(参数列表) => { 
    语句块;
    return 结果; 
}

Action<string> greet = name => {
    string message = $"Hello, {name}!";
    Console.WriteLine(message);
};
greet("Alice"); // 输出 "Hello, Alice!"
```

# Lambda表达式的简写
注意和{% post_link csharp表达式体 %}的区别

## **简化1：自动类型推断**
当委托类型明确时，参数类型可省略：  `(int x) => ...` → `x => ...`
```cs
Func<int, int> doubler = (x) => { 
    return x * 2; 
};
```

## **简化2：单参数可省略括号**
若只有一个参数，`()`可省略：  `(x) => ...` → `x => ...`
```cs
Func<int, int> doubler = x => { 
    return x * 2; 
};
```

## **简化3：单行表达式自动返回**
若主体是单行表达式，可省略`{}`和`return`：  `x => { return x*2; }` → `x => x*2`
```cs
Func<int, int> doubler = x => x * 2;
```

## **注意事项**
```cs
// 多参数必须保留括号：
Func<int, int, int> add = (a, b) => a + b; // 正确
Func<int, int, int> add = a, b => a + b;   // 错误！

 // 无参数时需空括号:
Action printHello = () => Console.WriteLine("Hello");

// 复杂逻辑仍需代码块：
Action log = () => {
    Console.WriteLine("Start");
    // 多行逻辑
    Console.WriteLine("End");
};
```

# **Lambda的常见用途**
委托实例化  和 事件处理
```cs
Func<int, int, int> sum = (a, b) => a + b;
Action<string> log = msg => Console.WriteLine(msg);
button.Click += (sender, e) => MessageBox.Show("Clicked!");
```

**LINQ查询**  
与LINQ方法结合，实现数据筛选、转换等操作：
```cs
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var evenNumbers = numbers.Where(n => n % 2 == 0); // 筛选偶数
var squares = numbers.Select(x => x * x);         // 计算平方
```

**表达式树（Expression Trees）**  
将Lambda编译为表达式树，供其他框架（如EF Core）解析：
```cs
Expression<Func<int, bool>> expr = x => x > 5;
```


# **闭包与变量捕获**
**当匿名函数捕获了外部变量时，C# 编译器会自动生成一个隐藏的类（称为“闭包类”），将捕获的变量“打包”到这个类的实例中。这个实例的生命周期会延长，使得闭包可以在后续继续访问这些变量。**

**示例 1：基本闭包**
```cs
Func<int> CreateCounter()
{
    int count = 0;
    return () => ++count; // 闭包捕获了外部变量 count
}

var counter = CreateCounter();
Console.WriteLine(counter()); // 输出 1
Console.WriteLine(counter()); // 输出 2（说明闭包修改并保留了 count 的状态）
```
- **现象**：`count` 本应在 `CreateCounter` 方法执行完毕后被销毁，但闭包保留了它的状态。
- **原理**：编译器生成一个类，将 `count` 作为该类的字段存储，闭包通过这个类的实例访问 `count`。

**示例 2：循环中的闭包陷阱**
```cs
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var action in actions)
{
    action(); // 输出 3, 3, 3（而非预期的 0, 1, 2）
}
```
- **问题原因**：所有闭包共享同一个变量 `i`（在循环结束后，`i` 的值为 3）。
- **解决方案**：在循环内部创建临时变量，让闭包捕获独立的值：

```cs
for (int i = 0; i < 3; i++)
{
    int current = i; // 每次循环新建一个临时变量
    actions.Add(() => Console.WriteLine(current)); // 输出 0, 1, 2
}
```

