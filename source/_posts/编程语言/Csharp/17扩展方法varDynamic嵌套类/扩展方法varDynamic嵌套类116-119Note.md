---
title: 扩展方法varDynamic嵌套类116-119Note
date: 2026-03-22 20:18:24
toc: true
categories:
  - 编程语言
  - Csharp
  - 17扩展方法varDynamic嵌套类

tags:
  - 编程语言
  - Csharp
  - 17扩展方法varDynamic嵌套类

---



# 第18节：扩展方法与模式匹配

## 第一讲：扩展方法

### 引言
**扩展方法**是 C#的一项强大功能，它允许您在不创建新派生类型、重新编译或以其他方式修改原始类型源代码的情况下，"添加"新方法到现有类型。这对于扩展您不拥有的类型特别有用，比如来自.NET Framework 的类（例如 string、DateTime）或第三方库中的类。它们提供了实例方法的假象，但实际上是一种调用静态方法的语法糖。

### 工作原理：static 与 this 关键字
要创建扩展方法，必须遵循三条严格规则：
1. 该方法必须定义在**静态类**中。
2. 方法本身必须是**静态的**。
3. 方法的第一个参数必须用 `this` 关键字标记，后跟要扩展的类型名称。

`this` 参数是编译器用来将方法关联到类型的关键。调用该方法时，无需为这个首个参数提供实参，它会自动填充为调用该方法的实例。

### 何时使用及实际案例
扩展方法最著名的实际案例是 **LINQ（语言集成查询）**。你在集合上使用的每个 LINQ 方法——`.Where()`、`.Select()`、`.OrderBy()`、`.ToList()`——都是 `IEnumerable` 接口的扩展方法。

当需要创建与特定类型逻辑相关的可复用辅助函数或工具函数时，请使用扩展方法。

### 示例
假设我们经常需要统计字符串中的单词数量。与其在每个地方都单独编写工具函数，我们可以直接扩展字符串类型本身。

```csharp
// 1. 必须是静态类。
public static class StringExtensions
{
    // 2. 必须是静态方法。
    // 3. 'this string str' 表示此方法扩展了字符串类型。
    public static int WordCount(this string str)
    {
        if (string.IsNullOrEmpty(str))
        {
            return 0;
        }
        // 'str' 参数将是调用该方法的字符串实例。
        return str.Split(new char[] { ' ', '.', '?' }, StringSplitOptions.RemoveEmptyEntries).Length;
    }
}

// --- 在你的 Main 方法中 ---
string mySentence = "This is a sample sentence for testing.";

// 现在我们可以像调用字符串类的内置实例方法一样调用 WordCount 了！
int count = mySentence.WordCount();

Console.WriteLine($"句子内容是：'{mySentence}'");
Console.WriteLine($"单词数量是：{count}"); // 输出：8
```

### 面试视角
**问题**："扩展方法能否访问其扩展类型的私有成员？"

**理想答案**："不能。扩展方法本质上只是调用静态方法并将实例作为第一个参数传递的语法糖。它无法特殊访问类型的私有或受保护成员，只能通过被扩展类型的公共接口进行操作。"

## 第二讲：模式匹配

### 引言
**模式匹配**是现代 C#中一组强大的功能，它提供了一种更复杂且可读性更高的方式来检查变量是否符合特定的"形状"或"模式"，并在符合时从中提取信息。它显著增强了 if 语句的能力，特别是将 switch 语句和表达式转变为高度先进的控制流工具。

### 工作原理：常见模式
模式匹配引入了多种可用于条件逻辑的新模式。

#### 1. 类型模式（is Type 变量名）
该模式既检查对象的运行时类型，又会在检查成功时将该对象赋值给该特定类型的新变量。
```csharp
object myValue = "Hello World";

if (myValue is string str) // 检查 myValue 是否是 string 类型，如果是则赋值给 'str'
{
    Console.WriteLine($"It's a string with length: {str.Length}");
}
```

#### 2. 属性模式（{ 属性名: 模式 }）
该模式允许检查对象属性的值。在 switch 表达式中使用时功能最为强大。
```csharp
public class Car { public int PassengerCount { get; set; } }

Car myCar = new Car { PassengerCount = 4 };

string message = myCar switch
{
    { PassengerCount: 0 } => "It's empty.",
    { PassengerCount: 1 } => "It's a solo ride.",
    { PassengerCount: 2 } => "A couple's trip.",
    _ => "It's a group." // _ 是弃元模式，匹配所有其他情况
};
```

#### 3. 关系模式 (<, >, <=, >=)
这些模式允许 switch 语句处理范围值，而不仅仅是常量值。
```csharp
int score = 85;

string grade = score switch
{
    >= 90 => "A",
    >= 80 => "B",
    >= 70 => "C",
    _ => "F"
};
```

#### 4. 逻辑模式（and、or、not）
这些模式允许您组合其他模式以实现更复杂的条件。
```csharp
string GetWeatherReport(double temp, bool isRaining) => (temp, isRaining) switch
{
    (> 30.0, false) => "Hot and sunny.",
    (> 20.0, true) => "Warm and rainy.",
    (< 10.0, _ ) and (_, true) => "Cold and rainy.", // _ 是弃元模式
    _ => "Moderate weather."
};
```

### 面试视角
**问题**： "模式匹配如何改变了现代 C#中的 switch 语句？"

**理想答案**： "模式匹配将 switch 语句从一个仅能处理常量整数值的简单控制结构，转变为了一个高度表达性的工具。借助现代模式匹配，switch 可以根据对象的类型、属性值以及使用关系和逻辑模式的数值范围进行分支判断，这使其成为复杂 if-else if 链的更简洁替代方案。"

## 第三讲：隐式类型变量（var）

### 引言
C# 3.0 引入的 `var` 关键字允许你在不显式指定数据类型的情况下声明局部变量。编译器会根据你用于初始化变量的值来推断其类型。

### 工作原理：编译时类型推断
这是最需要理解的核心概念：`var` **不会创建动态类型变量**。C# 仍然是静态类型语言。`var` 关键字纯粹是一个**编译时**特性。编译器会查看赋值操作符右侧的表达式并确定其类型，然后在中间语言（IL）中将 `var` 关键字替换为该显式类型。

当你写下：`var name = "John";`
编译器生成的代码与你写下：`string name = "John";` 时相同

一旦类型被推断并编译后，就会被固定。之后你不能为该变量赋予不同类型的值。

### 规则与最佳实践
- **规则**： `var` 变量必须在声明时初始化，以便编译器能推断其类型。`var x;` 会导致编译错误。
- **规则**： `var` 仅可用于方法内部的局部变量，不能用于字段或方法参数。
- **最佳实践 - 何时使用 var**： 当变量类型在赋值语句右侧显而易见时使用。这可以减少代码混乱并提高可读性。
  ```csharp
  var user = new User();  // 良好：类型明确为 User
  var names = new List<string>();  // 良好：类型明确为 List<string>
  var name = GetName();  // 良好，前提是 GetName() 明确返回字符串类型
  ```
- **最佳实践 - 应避免使用 var 的情况**： 当类型不明确时避免使用，这会降低代码可读性。
  ```csharp
  var amount = 20;  // 这是 int、double 还是 decimal 类型？应明确声明：decimal amount = 20M;
  var result = ProcessData();  // ProcessData 返回什么类型？最好明确声明：ProcessResult result = ProcessData();
  ```

### 面试视角
**问题**："var 关键字是否让 C#变成了动态语言？"

**最佳答案**："不，绝对不是。var 是用于编译时类型推断的特性。变量仍然是强类型且静态类型的；编译器只是根据初始化值为你推断出类型。编译后，用 var 声明的变量与用显式类型名称声明的变量没有任何区别。"

## 第四讲：动态类型变量（dynamic）

### 引言
C# 4.0 引入的 `dynamic` 关键字允许创建在编译时绕过静态类型检查的变量。它从根本上改变了编译器对待变量的方式。

### 工作原理：运行时解析
当你将变量声明为 `dynamic` 时，实际上是在告诉编译器："相信我。现在不要检查我对这个变量做的任何操作。我们将在运行时解决这些问题。"

所有对动态变量方法、属性或运算符的调用都会被封装起来，仅在程序实际运行时才进行解析。如果你尝试调用的成员在运行时存在，代码就能正常工作；如果不存在，程序将抛出 `RuntimeBinderException` 异常而崩溃。

### 为何及何时使用
使用 `dynamic` 时需要格外谨慎，因为它牺牲了 C# 静态类型安全的核心优势。其主要合法使用场景是互操作性：
1. 动态语言协作： 与 Python 或 IronRuby 等语言的库进行交互。
2. HTML 中的 DOM 操作： 在特定网页环境中操控 HTML 元素。
3. 解析动态数据格式： 处理如 JSON 这类结构可能事先未知的数据。
4. COM 互操作： 与旧版 COM 组件交互。

### var 与 dynamic：核心面试题
这是检验你对 C#类型系统理解的基础问题。

| 特性         | var（隐式类型）| dynamic（动态类型）|
|--------------|---------------------|----------------------|
| 类型系统     | 静态类型            | 动态类型             |
| 检查时机     | 编译时确定并检查    | 运行时解析           |
| 智能感知     | 完整支持            | 不支持               |
| 错误         | 编译时错误          | 运行时异常           |
| 示例         | `var x=10;x="hello"` 编译错误 | `dynamic x=10;x="hello"` 无错误 |

### 示例
```csharp
// 编译器知道 x 是一个整数类型。
var x = 10;
// Console.WriteLine(x.ToUpper()); // 编译时错误：'int'不包含'ToUpper'的定义

// 编译器对 y 能做什么一无所知。它信任你的判断。
dynamic y = 10;

// 下一行能通过编译，但会在运行时崩溃，因为整型数字10
// 没有 ToUpper()方法
try
{
    Console.WriteLine(y.ToUpper());
}
catch (Microsoft.CSharp.RuntimeBinder.RuntimeBinderException ex)
{
    Console.WriteLine("运行时错误: " + ex.Message);
}

// 这是有效的，因为 y 的类型在运行时可以改变
y = "hello";
Console.WriteLine(y.ToUpper()); // 现在可以正常工作了。输出：HELLO
```

## 第五讲：内部类（嵌套类）

### 引言
一个**内部类**，更正式的名称是**嵌套类**，是在另一个包含类的作用域内声明的类。这是一种将紧密耦合的类分组并控制其可见性的方式。

### 工作原理
嵌套类是其包含类的成员。这带来两个关键影响：
1. **访问权限**： 嵌套类可以访问其包含类的所有成员，包括私有字段和方法。这是因为嵌套类被视为包含类实现的一部分。
2. **可见性**： 嵌套类本身可以拥有访问修饰符（public、private 等）。如果将嵌套类声明为 private，它将对外部完全隐藏，只能被包含类使用。

### 为何及何时使用
当一个类在逻辑上仅在其包含类的上下文中才有意义，且不打算供公众使用时，应使用嵌套类。这是一种用于更好组织和封装的工具。

最经典的现实例子是 `LinkedList` 与其 `Node` 之间的关系。`Node`（包含值和指向下一个节点的指针）的概念是链表的实现细节。外部代码永远不需要直接创建或操作 `Node`。通过将 `Node` 类作为 `LinkedList` 内部的私有嵌套类，可以完全隐藏这一实现细节。

### 示例：链表与节点
```csharp
public class MyLinkedList<T>
{
    // 这是一个私有嵌套类，仅对 MyLinkedList<T> 类可见且可用
    private class Node
    {
        public T Value { get; set; }
        public Node Next { get; set; } // 可访问其他 Node 对象

        public Node(T value)
        {
            this.Value = value;
        }
    }

    // 包含类拥有一个字段，该字段是嵌套类的实例。
    private Node _head;

    public void Add(T value)
    {
        Node newNode = new Node(value); // 我们可以在此处创建 Node 对象。
        if (_head == null)
        {
            _head = newNode;
        }
        else
        {
            Node current = _head;
            while (current.Next != null)
            {
                current = current.Next;
            }
            current.Next = newNode;
        }
    }
}

// --- 在你的 Main 方法中 ---
MyLinkedList<int> list = new MyLinkedList<int>();
list.Add(1);
list.Add(2);

// 下面这行代码会导致编译错误，因为 Node 类
// 是私有且嵌套在 MyLinkedList 内部的。
// MyLinkedList<int>.Node myNode = new MyLinkedList<int>.Node(3); // 错误！无法访问。
```

