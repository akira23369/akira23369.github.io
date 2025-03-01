---
title: csharp表达式体
date: 2025-02-28 18:17:19
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---

表达式体（Expression-bodied members）是 C# 6.0 及更高版本引入的特性，它允许用简洁的 `=>` 语法替代传统代码块，适用于方法、属性、构造函数等成员。

**注意**：
**{% post_link csharp的Lambda表达式 %}（核心是创建匿名函数）**
**表达式体成员（核心是简写方法体）**

# **综合示例**
```cs
public class Calculator {
    // 只读属性
    public string Model => "Scientific-Calculator-3000";

    // 方法
    public double Square(double x) => x * x;

    // 索引器
    private double[] _history = new double[10];
    public double this[int index] => _history[index];

    // 构造函数
    public Calculator(string model) => Model = model;

    // 运算符重载
    public static Calculator operator +(Calculator a, Calculator b) => 
        new Calculator($"{a.Model}+{b.Model}");
}
```



# **方法（Methods）**
用 `=>` 替代 `{}`，适用于单行返回值的方法。
```cs
// 传统写法
public int Add(int a, int b) {
    return a + b;
}

// 表达式体写法
public int Add(int a, int b) => a + b;
```

# **只读属性（Read-Only Properties）**
直接返回计算结果的属性（仅有 `get` 访问器）。
```cs
// 传统写法
public string FullName {
    get { return $"{FirstName} {LastName}"; }
}

// 表达式体写法
public string FullName => $"{FirstName} {LastName}";
```

# **构造函数/析构函数（C# 7.0+）**
单行初始化或清理逻辑。
```cs
// 构造函数
public class Person {
    public string Name { get; }
    public Person(string name) => Name = name; // 初始化
}

// 析构函数
public class Resource {
    ~Resource() => Console.WriteLine("资源已释放"); // 清理逻辑
}
```

# **索引器（Indexers）**
简化索引器的 `get` 访问器。
```cs
private string[] _data = new string[10];

// 传统写法
public string this[int index] {
    get { return _data[index]; }
}

// 表达式体写法
public string this[int index] => _data[index];
```

# **属性访问器（C# 7.0+）**
对 `get` 和 `set` 访问器分别使用表达式体。
```cs
private string _name;

// 传统写法
public string Name {
    get { return _name; }
    set { _name = value; }
}

// 表达式体写法
public string Name {
    get => _name;
    set => _name = value ?? throw new ArgumentNullException();
}
```

# **事件访问器（C# 7.0+）**
简化事件的 `add` 和 `remove` 逻辑。
```cs
private EventHandler _myEvent;

// 传统写法
public event EventHandler MyEvent {
    add { _myEvent += value; }
    remove { _myEvent -= value; }
}

// 表达式体写法
public event EventHandler MyEvent {
    add => _myEvent += value;
    remove => _myEvent -= value;
}
```

# **运算符重载（Operator Overloading）**
简化运算符的实现。
```cs
public class Vector {
    public int X { get; }
    public int Y { get; }

    public Vector(int x, int y) => (X, Y) = (x, y);

    // 传统运算符重载
    public static Vector operator +(Vector a, Vector b) {
        return new Vector(a.X + b.X, a.Y + b.Y);
    }

    // 表达式体写法
    public static Vector operator +(Vector a, Vector b) => new(a.X + b.X, a.Y + b.Y);
}
```

# **`throw` 表达式（C# 7.0+）**
直接在表达式中抛出异常。
```cs
// 参数校验
public string GetName(string input) => 
    input ?? throw new ArgumentNullException(nameof(input));

// 替代传统写法：
public string GetName(string input) {
    if (input == null) {
        throw new ArgumentNullException(nameof(input));
    }
    return input;
}
```


