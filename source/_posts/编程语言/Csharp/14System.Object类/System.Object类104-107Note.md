---
title: System.Object类104-107Note
date: 2026-03-21 22:05:14
toc: true
categories:
  - 编程语言
  - Csharp
  - 14System.Object类

tags:
  - 编程语言
  - Csharp
  - 14System.Object类

---

# 第15节：System.Object 类

## 第一讲：System.Object 类概述

### 引言
在.NET 类型系统中，**System.Object（或其 C#别名 object）** 是最重要的类。它是**终极基类**，是 C#中所有类型隐式继承的通用祖先。无论你创建的是 class、struct、enum，还是使用像 int 或 double 这样的基本类型，它们最终都源自 System.Object。这一设计选择构成了 C#强大而统一的类型系统基础。

### 内部类层次结构与设计
这种层级结构确保每种类型都有共同的基础。其结构如下：
- System.Object：所有类型的根基。
- 引用类型（class、delegate、string 等）：这些类型直接继承自 System.Object。
- 值类型（struct、enum、int、double 等）：这些类型继承自名为 System.ValueType 的特殊抽象类，而该类又直接继承自 System.Object。

### 为什么这样设计？统一类型系统
这种设计的主要目的是建立一个**统一类型系统**。通过拥有一个共同的基类，.NET 框架确保每个变量，无论其具体类型如何，都能共享一组基础功能。这带来了几个强大的优势：
1. **多态性**：它允许你编写可以操作任何类型的方法。你可以创建一个 object 类型的变量并为其分配任何值，因为所有类型都"是一个"object。
2. **通用服务**：它确保每个对象都能访问一组基本方法，用于执行比较相等性(Equals)、获取字符串表示(ToString)和检索类型信息(GetType)等基本操作。
3. **通用集合**：在泛型出现之前，这种设计允许创建像 ArrayList 这样的集合，可以存储任意数据类型的混合，因为它们都可以被视为其基础 object 类型。（这导致了装箱/拆箱操作，我们稍后会讲到）。

### 面试视角
**问题**："System.Object 在 C#中扮演什么角色？"

**理想答案**："System.Object 是.NET 框架中所有类型的终极基类，它创建了一个统一的类型系统。包括引用类型和值类型在内的每种类型都隐式继承自它。这种设计保证了每个对象都拥有一组通用方法，如 ToString()、Equals() 和 GetHashCode()，实现了强大的多态性，使得任何类型都可以被视为 object。"

## 第二讲：理解并重写 Object 类的方法

### 引言
由于您创建的每个类都隐式继承自 System.Object，因此也会继承其公共和受保护的实例方法。这些方法提供了基础行为，但其默认实现通常过于通用。为了让您的类表现出智能行为，您需要经常重写这些方法以提供自定义逻辑。

### 四个重要的实例方法

#### ToString()
- **默认行为**：ToString() 的默认实现并不实用，它仅返回包含类型完全限定名的 string（例如 "MyProject.Person"）。
- **重写 (virtual)**：此方法被标记为 virtual，意味着它被设计为可重写。你应在自己的类中重写 ToString() 方法，以提供对象当前状态有意义且人类可读的字符串表示。这对日志记录和调试非常宝贵。

**示例**：
```csharp
public class Car
{
    public string Model { get; set; }
    public int Year { get; set; }

    // 重写 ToString() 以提供有用的描述。
    public override string ToString()
    {
        return $"{Year} {Model}";
    }
}

// --- 在 Main 方法中 ---
Car myCar = new Car { Model = "Mustang", Year = 2023 };
Console.WriteLine(myCar.ToString()); // 输出: 2023 Mustang
```

#### Equals(object obj)
- **默认行为**：这是面试中的关键考点。默认行为取决于类型：
  - 对于引用类型（class），它执行**引用相等性**比较。只有当两个变量指向内存中的同一个对象时才会返回 true。
  - 对于值类型（struct），它执行**值相等性**比较。通过反射比较两个结构体的所有字段是否相等。
- **重写 (virtual)**：通过重写 Equals() 方法，可以为类定义自定义的"相等"含义，通常是通过比较关键字段或属性的值来实现。

**示例**：
```csharp
public class Point
{
    public int X { get; set; }
    public int Y { get; set; }

    public override bool Equals(object obj)
    {
        // 检查是否为 null 以及类型是否匹配
        if (obj == null || this.GetType() != obj.GetType()) return false;

        Point other = (Point)obj;
        // 将相等性定义为具有相同的 X 和 Y 值。
        return (this.X == other.X) && (this.Y == other.Y);
    }
}

Point p1 = new Point { X = 10, Y = 20 };
Point p2 = new Point { X = 10, Y = 20 };
Point p3 = new Point { X = 5, Y = 5 };

Console.WriteLine(p1.Equals(p2)); // 输出: True
Console.WriteLine(p1.Equals(p3)); // 输出: False
```

#### GetHashCode()
- **什么是哈希码？** 哈希码是从对象生成的固定大小的数值。它并非唯一，但良好的哈希算法会产生广泛分布的值，从而最小化冲突（当两个不同对象产生相同哈希码时）。哈希码的主要目的是在基于哈希的集合（如 `Dictionary<TKey, TValue> 和 HashSet<T>`）中实现高性能查找。
- **基于哈希的集合如何工作？** 当你向 Dictionary 添加项时，字典不会遍历所有现有项来检查键是否已存在。而是：
  1. 它调用键的 GetHashCode() 方法来获取一个整数。
  2. 它使用这个整数计算索引，该索引指向其内部数组中的特定"桶"或"槽"。
  3. 它将键值对放入该桶中。
  
  当查找键时，它会重复这个过程：计算哈希码，直接定位到正确的桶，然后使用 Equals()方法仅检查该桶内的少量项。这就是字典查找速度极快的原因（平均时间复杂度为 O(1)）。

- **重写的黄金法则**：如果重写了 Equals()，就必须同时重写 GetHashCode()。
- **为什么？** 如果两个对象通过你的 Equals() 方法判定为相等，但它们产生不同的哈希码，基于哈希的集合就会出错。这可能导致它们被放入不同的存储桶中。当你尝试查找其中一个对象时，字典会访问错误的存储桶，无法找到该对象（即使存在另一个"相等"的对象），并错误地报告该元素不在集合中。**契约规定**：如果 a.Equals(b) 为真，那么 a.GetHashCode() 必须等于 b.GetHashCode()。

**示例**：
```csharp
public class Point // 延续上面的代码
{
    // ... X 和 Y 属性以及 Equals 方法 ...

    public override int GetHashCode()
    {
        // 一种现代、简单且有效的组合哈希码的方式。
        return HashCode.Combine(X, Y);
    }
}
```

#### GetType()
- **行为**：此方法不可被重写且非虚方法。它始终返回一个包含对象确切运行时类型详细元数据的 Type 对象。
- **使用场景**：当需要在运行时检查对象类型时（通常用于反射等进阶主题）。

**示例**：
```csharp
Car myCar = new Car { Model = "Mustang", Year = 2023 };
Type carType = myCar.GetType();
Console.WriteLine($"The object is of type: {carType.FullName}");
```

## 第三讲：装箱

### 引言
**装箱**是将值类型实例（如 int、double 或自定义 struct）转换为引用类型（object 或其实现的接口）的过程。这使得通常存在于快速栈上的值类型能够像对象一样被处理，并存储在堆上，从而可以在需要 object 的集合或方法中使用。

### 工作原理：内部机制
当值类型被装箱时，.NET 运行时在幕后执行以下步骤：
1. **内存分配**：在堆上分配一小块内存。这个内存块将作为"装箱容器"。
2. **值复制**：值类型变量（位于栈上）的值会被复制到堆上新分配的装箱对象中。
3. **引用返回**：结果变量现在是一个引用（内存地址），指向堆上的这个新装箱对象。

### 发生时机
当您将值类型赋值给 object 类型变量或接口类型变量时，装箱会隐式发生。

### 性能影响
装箱是一个**计算上开销高昂的操作**，在性能关键代码中应避免使用。其成本主要来自两个方面：
1. 堆上的内存分配速度明显慢于栈分配。
2. 在堆上新创建的装箱对象会给垃圾回收器(GC) 带来压力，最终 GC 不得不清理这些对象。

### 示例
```csharp
// 1. 'i' 是值类型，存储在栈上。
int i = 123;

// 2. 'o' 是引用类型变量。
// 当 'i' 赋值给 'o' 时，装箱发生。
object o = i; 

// 发生的过程：
// - 在堆上分配一个新的对象装箱容器。
// - 栈上的值类型变量的值（123）被复制到堆上的这个新装箱对象中。
// - 变量 'o' 现在持有这个装箱对象的内存地址。

// 最常见的历史示例是非泛型集合。
System.Collections.ArrayList list = new System.Collections.ArrayList();
list.Add(456); // 此处发生装箱！456 被放入堆上的装箱容器中。
```

## 第四讲：拆箱

### 引言
**拆箱**是装箱的逆过程。它是将对象引用（指向已装箱的值类型）显式转换回其原始值类型的操作。

### 工作原理：内部机制
拆箱同样是一个多步骤、可能耗费资源的操作：
1. **类型检查**：运行时首先检查被拆箱的对象是否确实是目标值类型的精确装箱实例。如果对象为 null 或是其他类型的装箱值，则会抛出 InvalidCastException 异常。
2. **值复制**：若类型检查通过，该值会从堆上的装箱对象中复制到一个新的栈分配值类型变量中。


拆箱是显式转换，需要进行强制类型转换。

### 性能影响
拆箱操作由于涉及类型检查和内存复制，也存在性能开销，尽管通常比装箱操作略轻，因为它不涉及堆内存分配。

### 示例
```csharp
// 从一个已装箱的整数开始。
object o = 123;

// 要取回值，必须显式转换。这就是拆箱。
int j = (int)o;

// 发生的过程：
// 1. 运行时检查 'o' 是否确实指向一个已装箱的 'int'。是的。
// 2. 值 123 从堆上的装箱对象复制到栈上的变量 'j' 中。
Console.WriteLine($"Unboxed value: {j}");

// --- 拆箱失败的示例 ---
object anotherObject = 456; // 已装箱的 int

try
{
    // 这会失败。即使转换在正常情况下是安全的（如 int 转 long），
    // 也不能直接拆箱为不同的类型。必须先拆箱为原始的精确类型。
    long l = (long)anotherObject; 
}
catch (InvalidCastException ex)
{
    Console.WriteLine("\nError: " + ex.Message);
}

// 正确的转换方式：
long correctLong = (int)anotherObject; // 1. 拆箱为 int，2. 隐式转换 int 为 long。
```

## 第五讲：重要知识点备忘

1. **Object 是万物之源**：.NET 中的每个类型都派生自 System.Object。这形成了统一的类型系统，任何变量都能被视为 object，实现了强大的多态性。
2. **重写 ToString() 以实现有意义的调试**：在所有自定义类中重写 ToString() 方法是最佳实践基础。良好的 ToString() 实现能通过提供清晰可读的对象状态表示，极大简化日志记录和调试过程。
3. **关于 Equals() 和 GetHashCode() 的契约**：这是面试中的关键考点。如果重写了 Equals() 方法来实现基于值的相等性比较，就必须同时重写 GetHashCode()。规则很简单：被视为相等的两个对象必须返回相同的哈希码。
4. **装箱和拆箱是性能陷阱**：需要特别注意，由于涉及堆内存分配、垃圾回收器压力以及类型检查，这些操作的计算开销很大。泛型（例如使用 `List<int> 替代 ArrayList`）作为重要语言特性被引入，正是为了通过创建类型安全的集合来消除这些隐藏的性能损耗——值类型在这种集合中无需装箱操作。
5. **理解 GetType() 与 is 的区别**：myObject.GetType() == typeof(MyClass) 检查对象是否为**精确类型**。而 is 运算符（myObject is MyBaseClass）更灵活；它会检查继承层次结构中的兼容性（如果对象是该类型或其任何派生类型，则返回 true）。了解这一区别体现了对运行时类型检查的深入理解。

