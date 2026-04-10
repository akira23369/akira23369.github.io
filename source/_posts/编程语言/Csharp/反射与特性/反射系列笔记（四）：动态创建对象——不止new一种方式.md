---
title: 反射系列笔记（四）：动态创建对象——不止new一种方式
date: 2026-04-06 11:22:19
toc: true
categories:
  - 编程语言
  - Csharp
  - 反射与特性

tags:
  - 编程语言
  - Csharp
  - 反射与特性

---



# C# 反射系列笔记（四）：动态创建对象——不止new一种方式

> 目标：掌握反射创建对象的所有方法，理解不同方式的性能差异、适用场景，以及泛型类型的动态创建

---

## 一、为什么需要“动态创建对象”？

### 问题的本质

```csharp
// 编译时已知类型 → 直接 new
List<int> list = new List<int>();

// 编译时未知类型 → 无法 new，必须动态创建
Type unknownType = Assembly.Load("plugin.dll").GetType("Plugin.MyClass");
// 如何创建 unknownType 的实例？？？
```

**动态创建对象 = 在运行时，根据 Type 对象创建出该类型的实例**

### 直觉理解

```
正常创建：你有一张具体图纸（知道是别墅图纸）→ 直接盖别墅
动态创建：你手上有好几张图纸，运行时才知道用哪一张 → 看图纸盖房子
```

---

## 二、动态创建对象的四种方法

| 方法                            | 性能  | 灵活性 | 适用场景           |
| ----------------------------- | --- | --- | -------------- |
| `Activator.CreateInstance`    | 中等  | 高   | 通用场景，简单易用      |
| `ConstructorInfo.Invoke`      | 稍慢  | 最高  | 需要细粒度控制（如私有构造） |
| `Activator.CreateInstance<T>` | 快   | 低   | 已知类型参数（其实不算动态） |
| 委托/表达式树编译                     | 最快  | 高   | 高频调用，性能敏感      |

### 方法一：Activator.CreateInstance（最常用）

```csharp
// 1. 无参构造函数
Type t = typeof(StringBuilder);
object sb = Activator.CreateInstance(t);
Console.WriteLine(sb.GetType().Name);  // StringBuilder

// 2. 有参构造函数（参数按顺序传递）
object sbWithCapacity = Activator.CreateInstance(t, 100);  // new StringBuilder(100)

// 3. 多个参数
Type listType = typeof(List<int>);
object list = Activator.CreateInstance(listType, 10);  // new List<int>(10)

// 4. 泛型版本（编译时已知类型，不算真正的动态）
List<int> knownList = Activator.CreateInstance<List<int>>();
```

**参数匹配规则：**
- `Activator.CreateInstance` 会自动根据参数类型和数量匹配构造函数
- 如果找不到匹配的构造函数，抛出 `MissingMethodException`

### 方法二：ConstructorInfo.Invoke（更精细控制）

```csharp
Type t = typeof(StringBuilder);

// 1. 获取特定的构造函数
ConstructorInfo ctor1 = t.GetConstructor(Type.EmptyTypes);  // 无参
ConstructorInfo ctor2 = t.GetConstructor(new[] { typeof(int) });  // 有参

// 2. 调用构造函数
object sb1 = ctor1.Invoke(null);           // 无参，参数传 null 或空数组
object sb2 = ctor2.Invoke(new object[] { 50 });  // 有参

// 3. 调用私有构造函数（Activator 做不到！）
public class Singleton
{
    private Singleton() { }
    public static Singleton Instance { get; } = new Singleton();
}

Type singletonType = typeof(Singleton);
ConstructorInfo privateCtor = singletonType.GetConstructor(
    BindingFlags.NonPublic | BindingFlags.Instance,
    null, Type.EmptyTypes, null);
object singletonInstance = privateCtor.Invoke(null);  // 强制创建第二个实例！

// 4. 调用带复杂参数的构造函数
public class Person
{
    public Person(string name, int age, bool isActive) { }
}

Type personType = typeof(Person);
ConstructorInfo ctor = personType.GetConstructor(new[] { typeof(string), typeof(int), typeof(bool) });
object person = ctor.Invoke(new object[] { "张三", 25, true });
```

**ConstructorInfo.Invoke 的优势：**
- 可以调用私有构造函数（破坏单例模式，一般不要这样做）
- 可以精确选择重载（当有多个匹配时）
- 可以获得构造函数的元数据信息

### 方法三：`Activator.CreateInstance<T>`（泛型版本）

```csharp
// 注意：这不是真正的动态创建，因为 T 在编译时已知
List<int> list = Activator.CreateInstance<List<int>>();

// 实际用处不大，因为直接 new List<int>() 更简单
// 少数场景：作为泛型方法的返回值
public T Create<T>() where T : new()
{
    return Activator.CreateInstance<T>();
}
```

### 方法四：委托/表达式树（高性能）

简单来说，它的作用是：**把一个通过反射找到的、慢速的 `MethodInfo`，转化成一个像普通方法一样快速调用的“委托（Delegate）”。**

**为什么要用它？（性能翻倍的关键）**
当你用反射调用方法时：
- **`MethodInfo.Invoke`**：每次都要检查参数类型、安全权限、拆箱装箱。它很**慢**（相对而言）。
- **`Delegate.CreateDelegate`**：只在创建时检查一次。一旦创建成功，它在内存中就是一个指向方法的直接指针。调用它的速度**几乎等同于直接调用方法**。

假设我们有一个简单的类：
```cs
public class Calculator {
    public int Add(int a, int b) => a + b;
}


// 传统反射方式（慢）：
var method = typeof(Calculator).GetMethod("Add");
var result = method.Invoke(calcInstance, new object[] { 1, 2 }); // 慢：涉及对象数组包装和类型检查


// 使用 CreateDelegate 方式（快）：
// 1. 获取方法元数据
var method = typeof(Calculator).GetMethod("Add");
// 2. 创建一个强类型的委托 (需要提前定义好签名一致的 delegate)
// 参数：(目标委托类型, 目标实例对象(包含这个方法的那个“对象实体”), 方法元数据)
var fastAdd = (Func<int, int, int>)Delegate.CreateDelegate(typeof(Func<int, int, int>), calcInstance, method);
// 3. 像普通方法一样调用
int sum = fastAdd(10, 20); // 极快！

```




```csharp
using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.Linq.Expressions;

namespace FastInstantiation
{
    // 测试用的目标类
    public class User
    {
        public int Id { get; set; }
        public string Name { get; set; }
    }

    /// <summary>
    /// 高性能实例创建器
    /// </summary>
    public static class FastActivator
    {
        // 使用 ConcurrentDictionary 缓存编译后的委托，保证线程安全
        private static readonly ConcurrentDictionary<Type, Func<object>> _creatorCache = new();

        /// <summary>
        /// 创建实例（适用于运行时才知道 Type 的情况）
        /// </summary>
        public static object CreateInstance(Type type)
        {
            // 如果缓存中有，直接取；如果没有，则编译并存入缓存
            var creator = _creatorCache.GetOrAdd(type, CompileCreator);

            // 执行委托，速度极快
            return creator();
        }

        /// <summary>
        /// 泛型版本（如果编译时已经知道类型 T，性能更高且免去装箱拆箱）
        /// 利用静态泛型类的特性，天然实现每个类型缓存一个委托
        /// </summary>
        public static T CreateInstance<T>()
        {
            return GenericCache<T>.Creator();
        }

        // 核心：将类型的无参构造函数编译为委托
        private static Func<object> CompileCreator(Type type)
        {
            // 1. 获取无参构造函数
            var ctor = type.GetConstructor(Type.EmptyTypes);
            if (ctor == null)
            {
                throw new ArgumentException($"Type {type.Name} does not have a parameterless constructor.");
            }

            // 2. 创建 new T() 的表达式
            var newExp = Expression.New(ctor);

            // 3. 将 new T() 转换为返回 object 类型的表达式 (装箱)
            // 相当于：() => (object)new T();
            var castExp = Expression.Convert(newExp, typeof(object));

            // 4. 生成 Lambda 表达式并编译成委托
            var lambda = Expression.Lambda<Func<object>>(castExp);
            return lambda.Compile();
        }

        // 泛型缓存类
        private static class GenericCache<T>
        {
            public static readonly Func<T> Creator = CompileGenericCreator();

            private static Func<T> CompileGenericCreator()
            {
                var ctor = typeof(T).GetConstructor(Type.EmptyTypes);
                if (ctor == null) throw new ArgumentException($"No parameterless constructor for {typeof(T).Name}");

                var newExp = Expression.New(ctor);
                var lambda = Expression.Lambda<Func<T>>(newExp);
                return lambda.Compile();
            }
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            int iterations = 10_000_000; // 1千万次
            Type targetType = typeof(User);

            Console.WriteLine($"开始测试，循环次数: {iterations:N0}\n");

            // 预热 (消除首次编译带来的时间影响)
            Activator.CreateInstance(targetType);
            FastActivator.CreateInstance(targetType);
            FastActivator.CreateInstance<User>();

            var sw = new Stopwatch();

            // 1. 原生 direct new
            sw.Start();
            for (int i = 0; i < iterations; i++)
            {
                var obj = new User();
            }
            sw.Stop();
            Console.WriteLine($"直接 new User():         {sw.ElapsedMilliseconds} ms");

            // 2. 反射 Activator.CreateInstance
            sw.Restart();
            for (int i = 0; i < iterations; i++)
            {
                var obj = Activator.CreateInstance(targetType);
            }
            sw.Stop();
            Console.WriteLine($"Activator (反射):        {sw.ElapsedMilliseconds} ms");

            // 3. 表达式树缓存委托 (返回 object)
            sw.Restart();
            for (int i = 0; i < iterations; i++)
            {
                var obj = FastActivator.CreateInstance(targetType);
            }
            sw.Stop();
            Console.WriteLine($"表达式树委托 (Type 参数): {sw.ElapsedMilliseconds} ms");

            // 4. 表达式树缓存委托 (泛型版本)
            sw.Restart();
            for (int i = 0; i < iterations; i++)
            {
                var obj = FastActivator.CreateInstance<User>();
            }
            sw.Stop();
            Console.WriteLine($"表达式树委托 (泛型):      {sw.ElapsedMilliseconds} ms");

            Console.ReadLine();
        }
    }
}
```



## 三、动态创建对象的方法对比

```cs
using System;
using System.Diagnostics;
using System.Linq.Expressions;
using System.Reflection;

public class TestTarget { } // 测试类

class Program
{
    private const int Iterations = 10_000_000;

    static void Main()
    {
        Console.WriteLine($"测试开始，循环次数: {Iterations:N0}\n");

        // 1. 直接 new (基准)
        Measure("直接 New (基准)", () => {
            for (int i = 0; i < Iterations; i++) { var obj = new TestTarget(); }
        });

        // 2. Activator.CreateInstance (非泛型)
        var type = typeof(TestTarget);
        Measure("Activator.CreateInstance (Object)", () => {
            for (int i = 0; i < Iterations; i++) { var obj = Activator.CreateInstance(type); }
        });

        // 3. Activator.CreateInstance<T> (泛型)
        Measure("Activator.CreateInstance<T>", () => {
            for (int i = 0; i < Iterations; i++) { var obj = Activator.CreateInstance<TestTarget>(); }
        });

        // 4. ConstructorInfo.Invoke
        var ctor = type.GetConstructor(Type.EmptyTypes);
        Measure("ConstructorInfo.Invoke", () => {
            for (int i = 0; i < Iterations; i++) { var obj = ctor.Invoke(null); }
        });

        // 5. 表达式树编译 (Expression Tree)
        var lambda = Expression.Lambda<Func<TestTarget>>(Expression.New(type)).Compile();
        Measure("表达式树编译 (Delegate)", () => {
            for (int i = 0; i < Iterations; i++) { var obj = lambda(); }
        });
    }

    static void Measure(string name, Action action)
    {
        var sw = Stopwatch.StartNew();
        action();
        sw.Stop();
        Console.WriteLine($"{name,-35} | 耗时: {sw.ElapsedMilliseconds,6} ms");
    }
}


```

```txt
测试开始，循环次数: 10,000,000

直接 New (基准)                         | 耗时:    121 ms
Activator.CreateInstance (Object)   | 耗时:    257 ms
Activator.CreateInstance<T>         | 耗时:    263 ms
ConstructorInfo.Invoke              | 耗时:    255 ms
表达式树编译 (Delegate)                   | 耗时:    124 ms
```


---

## 四、动态创建泛型类型实例

### 泛型类型的基本概念

```csharp
// 泛型定义（开放类型）
Type openType = typeof(List<>);     // List<T>
Type openDict = typeof(Dictionary<,>);  // Dictionary<TKey, TValue>

// 构造好的泛型（封闭类型）
Type closedType = typeof(List<int>);
```

### 创建泛型实例的完整流程

```csharp
// 步骤1：获取泛型定义
Type openList = typeof(List<>);

// 步骤2：指定泛型参数，构造封闭类型
Type closedList = openList.MakeGenericType(typeof(string));

// 步骤3：创建实例
object stringList = Activator.CreateInstance(closedList);
// 等价于：new List<string>()

// 完整示例：动态创建 Dictionary<string, int>
Type openDict = typeof(Dictionary<,>);
Type closedDict = openDict.MakeGenericType(typeof(string), typeof(int));
object dict = Activator.CreateInstance(closedDict);

// 验证
Console.WriteLine(dict.GetType());  // System.Collections.Generic.Dictionary`2[System.String,System.Int32]
```

### 泛型类型参数的约束检查

```csharp
// 如果泛型参数有约束，MakeGenericType 会在运行时检查
public class Repository<T> where T : class, new() { }

Type openRepo = typeof(Repository<>);

// int 满足 struct？不，int 是值类型，不满足 class 约束
// Type closed = openRepo.MakeGenericType(typeof(int));  // 运行时异常！

// ✅ 正确：string 是引用类型，有无参构造
Type closed = openRepo.MakeGenericType(typeof(string));
object repo = Activator.CreateInstance(closed);
```

### 创建带多个泛型参数的实例

```csharp
// 场景：运行时才知道泛型参数类型
public object CreateGenericList(Type elementType)
{
    Type openList = typeof(List<>);
    Type closedList = openList.MakeGenericType(elementType);
    return Activator.CreateInstance(closedList);
}

// 使用
object intList = CreateGenericList(typeof(int));     // List<int>
object stringList = CreateGenericList(typeof(string)); // List<string>
object personList = CreateGenericList(typeof(Person)); // List<Person>
```

### 泛型方法中的动态创建

```csharp
public class GenericFactory
{
    // 方式1：反射调用泛型方法
    public object CreateGeneric(Type elementType)
    {
        // 获取泛型方法定义
        MethodInfo method = typeof(GenericFactory)
            .GetMethod(nameof(CreateList), BindingFlags.NonPublic | BindingFlags.Instance);
        
        // 构造封闭方法
        MethodInfo closedMethod = method.MakeGenericMethod(elementType);
        
        // 调用
        return closedMethod.Invoke(this, null);
    }
    
    private List<T> CreateList<T>()
    {
        return new List<T>();
    }
}

// 使用
var factory = new GenericFactory();
object list = factory.CreateGeneric(typeof(double));  // List<double>
```

---

## 五、特殊情况处理

### 创建数组

```csharp
// 方法1：Type.MakeArrayType
Type elementType = typeof(int);
Type arrayType = elementType.MakeArrayType();      // int[]
Type multiArrayType = elementType.MakeArrayType(2); // int[,]

object array = Activator.CreateInstance(arrayType, 10);  // new int[10]

// 方法2：Array.CreateInstance（更直接）
Array dynamicArray = Array.CreateInstance(typeof(string), 5);
dynamicArray.SetValue("hello", 0);
```

### 创建委托

```csharp
// 动态创建委托类型实例
public delegate int MathOperation(int a, int b);

Type delegateType = typeof(MathOperation);
// 委托没有构造函数，需要通过 Delegate.CreateDelegate 创建

MethodInfo method = typeof(Math).GetMethod("Max", new[] { typeof(int), typeof(int) });
Delegate del = Delegate.CreateDelegate(delegateType, method);
object result = del.DynamicInvoke(10, 20);  // 20
```

### 创建枚举

```csharp
// 枚举本质上是值类型
Type enumType = typeof(DayOfWeek);
object monday = Activator.CreateInstance(enumType, 1);  // DayOfWeek.Monday
Console.WriteLine(monday);  // Monday

// 或者使用 Enum.ToObject
object tuesday = Enum.ToObject(enumType, 2);
```

### 创建接口和抽象类（不可能！）

```csharp
// ❌ 以下代码会抛出异常
Type interfaceType = typeof(IDisposable);
// Activator.CreateInstance(interfaceType);  // MissingMethodException

Type abstractType = typeof(Stream);
// Activator.CreateInstance(abstractType);  // MissingMethodException

// ✅ 需要创建具体的实现类
Type concreteType = typeof(MemoryStream);
object stream = Activator.CreateInstance(concreteType);
```

---

## 六、完整实战：通用对象工厂

```csharp
public class ObjectFactory
{
    // 缓存：类型 -> 构造函数委托
    private static readonly Dictionary<Type, Func<object[], object>> _factories = new();
    
    /// <summary>
    /// 创建对象（自动匹配构造函数参数）
    /// </summary>
    public static object Create(Type type, params object[] args)
    {
        // 如果没有传参，走无参构造函数逻辑
        if (args == null || args.Length == 0)
        {
            return CreateWithDefaultConstructor(type);
        }
        // 有参数，走参数匹配逻辑
        return CreateWithArguments(type, args);
    }
    
    private static object CreateWithDefaultConstructor(Type type)
    {
        // 值类型可以直接创建
        if (type.IsValueType)
        {
            return Activator.CreateInstance(type);
        }
        
        // 检查是否有公共无参构造函数
        var ctor = type.GetConstructor(Type.EmptyTypes);
        if (ctor != null)
        {
            return Activator.CreateInstance(type);
        }
        
        // 没有公共无参构造，尝试查找私有无参构造
        ctor = type.GetConstructor(BindingFlags.NonPublic | BindingFlags.Instance, 
            null, Type.EmptyTypes, null);
        
        if (ctor != null)
        {
            return ctor.Invoke(null);
        }
        
        throw new MissingMethodException($"类型 {type.FullName} 没有可访问的无参构造函数");
    }
    
    private static object CreateWithArguments(Type type, object[] args)
    {
        // 获取参数类型
        Type[] argTypes = args.Select(a => a.GetType()).ToArray();
        
        // 查找匹配的构造函数
        ConstructorInfo ctor = type.GetConstructor(argTypes);
        
        if (ctor == null)
        {
            // 尝试查找最佳匹配（处理继承和隐式转换）
            ctor = FindBestMatchConstructor(type, argTypes);
        }
        
        if (ctor == null)
        {
            string argTypeNames = string.Join(", ", argTypes.Select(t => t.Name));
            throw new MissingMethodException($"找不到匹配的参数类型的构造函数: ({argTypeNames})");
        }
        
        return ctor.Invoke(args);
    }
    
    private static ConstructorInfo FindBestMatchConstructor(Type type, Type[] argTypes)
    {
        var ctors = type.GetConstructors();
        
        foreach (var ctor in ctors)
        {
            var parameters = ctor.GetParameters();
            if (parameters.Length != argTypes.Length) continue;
            
            bool match = true;
            for (int i = 0; i < parameters.Length; i++)
            {
                if (!parameters[i].ParameterType.IsAssignableFrom(argTypes[i]))
                {
                    match = false;
                    break;
                }
            }
            
            if (match) return ctor;
        }
        
        return null;
    }
    
    /// <summary>
    /// 获取高性能工厂委托（缓存）
    /// 原理：第一次通过反射生成一段“创建对象”的代码并编译，后续直接调用这段编译好的代码。
    /// </summary>
    public static Func<object[], object> GetFactory(Type type)
    {
        lock (_factories)
        {
            if (!_factories.TryGetValue(type, out var factory))
            {
                factory = BuildFactory(type);
                _factories[type] = factory;
            }
            return factory;
        }
    }
    
    private static Func<object[], object> BuildFactory(Type type)
    {
        // 获取参数最多的构造函数（示例简化）
        var ctor = type.GetConstructors()
            .OrderByDescending(c => c.GetParameters().Length)
            .FirstOrDefault();
        
        if (ctor == null)
        {
            throw new InvalidOperationException($"类型 {type.Name} 没有公共构造函数");
        }
        
        // --- 表达式树的核心：手写 IL 指令的逻辑 ---

        var parameters = ctor.GetParameters();
        // 定义 Lambda 的输入参数：object[] args
        var paramArray = Expression.Parameter(typeof(object[]), "args");
                
        // 相当于在代码里写：(TypeA)args[0], (TypeB)args[1]...
        var arguments = parameters.Select((p, i) =>
            Expression.Convert(
                Expression.ArrayIndex(paramArray, Expression.Constant(i)),  // 取数组中的第 i 个元素
                p.ParameterType // 强制转换为构造函数需要的类型
            )).ToArray();
        
        // 构造“New”表达式：相当于写了 new MyClass(arg1, arg2...)
        var newExpr = Expression.New(ctor, arguments);
        
        // 式如：(object[] args) => new MyClass((TypeA)args[0], ...)
        var lambda = Expression.Lambda<Func<object[], object>>(newExpr, paramArray);
        
        return lambda.Compile();
    }
}

// 使用示例
public class Product
{
    public string Name { get; }
    public decimal Price { get; }
    
    public Product(string name, decimal price)
    {
        Name = name;
        Price = price;
    }
    
    private Product() { }  // 私有构造
}

// 动态创建
Type productType = typeof(Product);
object product = ObjectFactory.Create(productType, "iPhone", 999.99m);
Console.WriteLine(product);  // Product { Name = "iPhone", Price = 999.99 }

// 高性能版本
var factory = ObjectFactory.GetFactory(productType);
object another = factory(new object[] { "MacBook", 1299.99m });
```

---

## 七、常见异常及处理

| 异常 | 原因 | 解决方案 |
|------|------|----------|
| `MissingMethodException` | 找不到匹配的构造函数 | 检查参数类型和数量是否正确 |
| `TargetInvocationException` | 构造函数内部抛出了异常 | 查看 `InnerException` |
| `ArgumentException` | 参数类型不匹配 | 确保参数类型与构造函数参数类型兼容 |
| `NotSupportedException` | 尝试创建接口/抽象类 | 创建具体的实现类 |
| `VerificationException` | 类型安全验证失败 | 检查泛型约束 |

---

## 八、本篇速查表

| 需求 | 代码 |
|------|------|
| 无参创建 | `Activator.CreateInstance(typeof(T))` |
| 有参创建 | `Activator.CreateInstance(typeof(T), arg1, arg2)` |
| 调用私有构造 | `ctor.Invoke(null)` 配合 `BindingFlags.NonPublic` |
| 创建泛型实例 | `typeof(List<>).MakeGenericType(typeof(int))` 然后 `Activator.CreateInstance` |
| 创建数组 | `Array.CreateInstance(typeof(int), 10)` |
| 高性能缓存 | 表达式树编译委托 |
| 值类型创建 | `Activator.CreateInstance` 直接可用 |

---

## 九、思考题

1. `Activator.CreateInstance` 内部是如何找到正确的构造函数的？如果存在多个匹配会怎样？

2. 为什么 `Activator.CreateInstance` 比 `new` 慢那么多？反射查找构造函数的开销在哪里？

3. 如何实现一个支持依赖注入的容器（根据类型自动创建其依赖的对象）？

4. `FormatterServices.GetUninitializedObject` 是什么？和 `Activator.CreateInstance` 有什么区别？

---

**下一篇预告：** 《C# 反射系列笔记（五）：MethodInfo 与方法动态调用》

下一篇将深入讲解方法的反射调用，包括静态/实例方法、重载解析、ref/out 参数处理、异步方法调用，以及性能优化技巧。