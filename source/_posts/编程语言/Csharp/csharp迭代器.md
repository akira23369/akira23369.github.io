---
title: csharp迭代器
date: 2025-03-03 20:31:58
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---

# 迭代器是什么
在 C# 中，**迭代器（Iterator）** 是一种用于遍历集合（如数组、列表等）元素的机制，它允许你按顺序访问集合中的每个元素而不必暴露集合的内部结构。迭代器的核心是通过 `IEnumerable` 和 `IEnumerator` 接口实现的，但 C# 提供了更简洁的语法（`yield` 关键字）来简化迭代器的创建。

在表现效果上看，迭代器是可以在容器对象（例如链表或数组）上遍历访问的接口。设计人员无需关心容器对象的内存分配的实现细节，可以用foreach遍历的类，都是实现了迭代器的。

# **迭代器的基本概念**
- **`IEnumerable` 接口**：表示一个集合可以被迭代，包含一个 `GetEnumerator` 方法。
- **`IEnumerator` 接口**：提供了遍历集合的能力，包含 `MoveNext()`、`Reset()` 和 `Current` 属性。

**标准迭代器实现（手动版）**
**接口定义**：
```cs
public interface IEnumerable
{
    IEnumerator GetEnumerator();
}

public interface IEnumerator
{
    bool MoveNext();
    object Current { get; }
    void Reset(); // 实际开发中通常不实现
}
```
**完整实现示例**：
```cs
public class MyCollection : IEnumerable
{
    private int[] data = { 1, 2, 3 };

    public IEnumerator GetEnumerator() => new MyEnumerator(data);

    // 自定义枚举器
    private class MyEnumerator : IEnumerator
    {
        private int[] data;
        private int index = -1;

        public MyEnumerator(int[] data) => this.data = data;

        public bool MoveNext()
        {
            index++;
            return index < data.Length;
        }

        public object Current => data[index];
        public void Reset() => index = -1;
    }
}

// 使用：
foreach (int num in new MyCollection())
{
    Console.WriteLine(num); // 输出 1 2 3
}
```

# foreach本质
**foreach本质**:
1. 先获取in后面这个对象的 `IEnumerator` 会调用对象其中的`GetEnumerator`方法
2. 执行得到这个`IEnumerator`对象中的 `MoveNext`方法
3. 只要`MoveNext()`方法的返回值时true 就会去得到`Current`然后复制给 item
```cs
foreach (var item in collection) 
{
    // code
}

// 等价于：
IEnumerator enumerator = collection.GetEnumerator();
try 
{
    while (enumerator.MoveNext()) 
    {
        var item = enumerator.Current;
        // code
    }
}
finally 
{
    IDisposable disposable = enumerator as IDisposable;
    disposable?.Dispose();
}

```

# `yield return` 语法糖
C# 的 `yield return` 关键字可以快速定义迭代器，无需手动实现 `IEnumerator`

## **基本作用：简化迭代器**
`yield return` 的作用是告诉编译器：“帮我自动生成一个迭代器，按我写的顺序逐个返回这些值”。例如：
```cs
IEnumerable<int> GetNumbers()
{
    yield return 10;
    yield return 20;
    yield return 30;
}
```
当调用 `GetNumbers()` 时，它不会一次性返回所有值，而是返回一个“待执行的迭代器”。只有当你用 `foreach` 遍历时，才会按顺序取出每个值。

## **关键特性：延迟执行（Lazy）**
`yield return` 的代码不会一次性全部执行，而是按需执行。看这个例子：
```cs
IEnumerable<string> GetMessages()
{
    Console.WriteLine("开始迭代");
    yield return "第一步";
    Console.WriteLine("执行到中间");
    yield return "第二步";
    Console.WriteLine("结束迭代");
}

// 测试代码：
var messages = GetMessages();  // 这里不会输出任何内容！
Console.WriteLine("准备遍历");
foreach (var msg in messages)  // 此时才开始真正执行
{
    Console.WriteLine(msg);
}
```

**关键点**：
- 调用 `GetMessages()` 时，代码不会运行，只是返回一个“迭代器对象”。
- 当 `foreach` 开始遍历时，代码才从头执行，每次遇到 `yield return` 时：
    - 返回一个值，
    - **暂停当前方法**，记录当前位置（状态）。
    - 下次循环时，从暂停的位置继续执行。

**底层原理：状态机**
编译器会将 `yield return` 代码转换为一个隐藏的“状态机”类。例如，上述 `GetMessages()` 会被编译成一个类似如下的类：
```cs
class GeneratedStateMachine : IEnumerator<string>
{
    private int _state = 0;
    public string Current { get; private set; }

    public bool MoveNext()
    {
        switch (_state)
        {
            case 0:
                Console.WriteLine("开始迭代");
                Current = "第一步";
                _state = 1;
                return true;
            case 1:
                Console.WriteLine("执行到中间");
                Current = "第二步";
                _state = 2;
                return true;
            case 2:
                Console.WriteLine("结束迭代");
                return false; // 结束
            default:
                return false;
        }
    }
    // 其他接口方法（略）
}
```

## yield break
**`yield break` 的基本作用**
- **功能**：在迭代器方法中，`yield break` 会立即终止迭代，不再生成后续的值。
```cs
IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
    yield break; // 这里终止，后续的 yield return 不会执行
    yield return 3; // 永远不会执行
}

// 使用
foreach (var num in GetNumbers())
{
    Console.WriteLine(num);
}
// 输出：1, 2
```

## **实际应用案例**
**分页加载数据**
```cs
IEnumerable<int> LoadBigData()
{
    int pageSize = 1000;
    for (int page = 0; ; page++)
    {
        var data = QueryDatabase(page, pageSize);
        if (data.Count == 0) yield break;
        
        foreach (var item in data)
        {
            yield return item;
        }
    }
}
```


**游戏技能连招系统**：
```cs
IEnumerator ComboAttack()
{
    yield return PlayAnimation("Attack1");
    if (Input.GetButton("Attack"))
    {
        yield return PlayAnimation("Attack2");
        if (Input.GetButton("Attack"))
        {
            yield return PlayAnimation("FinalBlow");
        }
    }
}
```

**树形结构遍历**：
```cs
public IEnumerable<Node> Traverse(Node root)
{
    yield return root;
    foreach (var child in root.Children)
    {
        foreach (var node in Traverse(child))
        {
            yield return node;
        }
    }
}
```

