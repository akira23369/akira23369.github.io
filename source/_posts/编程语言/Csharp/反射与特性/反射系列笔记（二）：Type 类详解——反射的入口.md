---
title: 反射系列笔记（二）：Type 类详解——反射的入口
date: 2026-04-06 11:20:56
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



# C# 反射系列笔记（二）：Type 类详解——反射的入口

> 目标：彻底掌握 `Type` 类的每一个重要成员，学会如何通过 `Type` 获取类型的完整信息

---

## 一、为什么 Type 是反射的“总入口”？

### 核心定位

```
所有反射操作的第一步：拿到 Type 对象
         ↓
   Type 对象包含了一个类型的所有元数据
         ↓
从 Type 出发，可以获取：构造函数、方法、属性、字段、事件、接口、特性...
```

**一句话：没有 Type，就没有反射。**

### Type 的继承体系

```
System.Object
    ↓
System.Reflection.MemberInfo   ← 所有类型成员的基类
    ↓
System.Type                    ← 代表类型本身
    ↓
System.RuntimeType             ← CLR 内部使用的具体类型（开发者通常不直接接触）
```

`MemberInfo` 是所有类型成员（类、方法、属性、字段等）的基类，定义了通用的成员属性（如 `Name`、`DeclaringType`、`GetCustomAttributes()`）。

---

## 二、Type 的基础属性（快速查阅表）

### 身份信息

| 属性              | 返回值示例                                                                                                                                         | 说明                |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| `Name`          | `List1`                                                                                                                                       | 类型名称（不含命名空间）      |
| `FullName`      | `System.Collections.Generic.List1[[System.Int32, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]` | 完全限定名（含命名空间和泛型参数） |
| `Namespace`     | `"System.Collections.Generic"`                                                                                                                | 命名空间              |
| `Assembly`      | `Assembly` 对象                                                                                                                                 | 类型所在的程序集          |
| `Module`        | `Module` 对象                                                                                                                                   | 类型所在的模块           |
| `MetadataToken` | `0x10000001`                                                                                                                                  | 元数据表中的令牌（用于调试）    |

### 类型分类判断（返回 bool）

| 属性 | true 时表示 | 直觉理解 |
|------|------------|----------|
| `IsClass` | 是类 | "这是一个类图纸" |
| `IsInterface` | 是接口 | "这是一份协议图纸" |
| `IsEnum` | 是枚举 | "这是一组常量图纸" |
| `IsValueType` | 是值类型（struct/enum） | "这是一个轻量级图纸" |
| `IsArray` | 是数组 | "这是一排格子图纸" |
| `IsDelegate` | 是委托 | "这是一个函数指针图纸" |
| `IsPrimitive` | 是基元类型（int, bool, char...） | "这是基础零件图纸" |

### 修饰符判断（返回 bool）

| 属性 | true 时表示 | 示例 |
|------|------------|------|
| `IsPublic` | 是 public | `public class Foo` |
| `IsNotPublic` | 是 internal（或 private 嵌套类） | `internal class Bar` |
| `IsAbstract` | 是抽象类 | `public abstract class Base` |
| `IsSealed` | 是密封类 | `public sealed class Final` |
| `IsNested` | 是嵌套类型 | `class Outer { class Inner }` |
| `IsNestedPublic` | 嵌套且 public | `public class Outer { public class Inner }` |

### 泛型相关判断

| 属性 | 说明 |
|------|------|
| `IsGenericType` | 是泛型类型（如 `List<int>`） |
| `IsGenericTypeDefinition` | 是泛型定义（如 `List<>`） |
| `ContainsGenericParameters` | 包含未指定的泛型参数 |
| `IsConstructedGenericType` | 是构造好的泛型（已指定类型参数） |
| `GenericParameterPosition` | 泛型参数的位置（0, 1, 2...） |

### 继承相关

| 属性/方法 | 说明 |
|-----------|------|
| `BaseType` | 直接基类（`object` 的基类是 `null`） |
| `IsSubclassOf(Type)` | 判断是否为某个类的子类 |
| `IsAssignableFrom(Type)` | 判断是否可以安全赋值 |
| `IsInstanceOfType(object)` | 判断对象是否是该类型的实例 |

---

## 三、Type 的核心方法（获取成员）

### 方法总览

| 方法 | 返回类型 | 说明 |
|------|---------|------|
| `GetConstructors()` | `ConstructorInfo[]` | 获取构造函数 |
| `GetMethods()` | `MethodInfo[]` | 获取方法 |
| `GetProperties()` | `PropertyInfo[]` | 获取属性 |
| `GetFields()` | `FieldInfo[]` | 获取字段 |
| `GetEvents()` | `EventInfo[]` | 获取事件 |
| `GetInterfaces()` | `Type[]` | 获取实现的接口 |
| `GetMembers()` | `MemberInfo[]` | 获取所有成员（方法、属性、字段等） |
| `GetNestedTypes()` | `Type[]` | 获取嵌套类型 |
| `GetCustomAttributes()` | `object[]` | 获取自定义特性 |

### 获取单个成员的版本

每个获取集合的方法都有对应的**获取单个**的重载：

```csharp
// 获取单个构造函数（需要指定参数类型）
ConstructorInfo ctor = t.GetConstructor(new Type[] { typeof(int), typeof(string) });

// 获取单个方法（需要指定名称和参数类型）
MethodInfo method = t.GetMethod("Add", new Type[] { typeof(int), typeof(int) });

// 获取单个属性（需要指定名称）
PropertyInfo prop = t.GetProperty("Name");

// 获取单个字段（需要指定名称）
FieldInfo field = t.GetField("_value");

// 获取单个事件（需要指定名称）
EventInfo evt = t.GetEvent("Click");
```

**重要：当存在重载时，`GetMethod(string)` 可能返回 null 或不明确的匹配，必须指定参数类型数组。**

---

## 四、BindingFlags：搜索的“放大镜”

### 默认行为 vs 使用 BindingFlags
GetProperties()、GetMethods() 等方法的**默认行为（不带 BindingFlags 的重载）：**
- 只返回 **public** 成员
- 只返回 **实例** 成员（对于 `GetMethods()` 等）
- 返回**继承链**上的成员

```csharp
// 默认：只获取公共实例方法（包括继承的）
MethodInfo[] publicMethods = t.GetMethods();
```

**使用 BindingFlags 可以精确控制：**

```csharp
using System.Reflection;

// 获取所有私有实例字段（不含继承）
FieldInfo[] privateFields = t.GetFields(
    BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.DeclaredOnly);

// 获取所有公共静态方法（包括继承的）
MethodInfo[] staticMethods = t.GetMethods(
    BindingFlags.Public | BindingFlags.Static | BindingFlags.FlattenHierarchy);
```

必须同时指定 **可见性**（Public 或 NonPublic） 和 **作用域**（Instance 或 Static），否则返回结果大概率为空！

### BindingFlags 完整列表

| 类别       | 标志                                       | 说明                                  |
| -------- | ---------------------------------------- | ----------------------------------- |
| **访问级别** | `Public`                                 | 公共成员                                |
|          | `NonPublic`                              | 非公共成员（private, protected, internal） |
| **成员类型** | `Instance`                               | 实例成员                                |
|          | `Static`                                 | 静态成员                                |
| **继承控制** | `DeclaredOnly`                           | 只在本类型声明的（不包括基类）                     |
|          | `FlattenHierarchy`                       | 包含继承链中的静态成员                         |
| **名称匹配** | `IgnoreCase`                             | 忽略大小写                               |
| **其他**   | `GetField` 专用：`SetField` / `SetProperty` | 用于绑定目标                              |

### 常用组合模式

```csharp
// 组合1：获取所有私有实例成员（仅本类型）
var flags1 = BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.DeclaredOnly;

// 组合2：获取所有公共成员（实例+静态，包括基类）
var flags2 = BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static;

// 组合3：获取所有成员（公开+私有，实例+静态，仅本类型）
var flags3 = BindingFlags.Public | BindingFlags.NonPublic | 
             BindingFlags.Instance | BindingFlags.Static | BindingFlags.DeclaredOnly;
```

---

## 五、深入理解每个获取方法

### GetConstructors()——构造函数
`public ConstructorInfo? GetConstructor(Type[] types);`
这是最重要的参数。它是一个**类型数组**，代表你想寻找的构造函数的**参数列表**。
**顺序必须一致**：数组中类型的顺序必须与构造函数定义的参数顺序完全对应。
**空数组**：传入 `Type.EmptyTypes`（或 `new Type[0]`）表示寻找**无参构造函数**。



`public ConstructorInfo? GetConstructor(BindingFlags bindingAttr, Binder? binder, Type[] types, ParameterModifier[]? modifiers);`
**进阶重载：控制访问权限**
当你需要查找 `private`、`protected` 或 `static` 构造函数时，需要使用带有 `BindingFlags` 的重载：
**关键参数详解：**
**`BindingFlags bindingAttr`**:见上面
**`Binder? binder`**:通常传 `null`。它用于控制类型转换规则（例如将 `int` 自动匹配到 `long`），默认 Binder 已经足够。
`Type[] types`:**类型数组**
**`ParameterModifier[]? modifiers`**:通常传 `null`。仅在处理 COM 互操作时，用于标识参数是否为 `ref` 或 `out`。



```csharp
public class Person
{
    public Person() { }
    public Person(string name) { }
    private Person(int id, string name) { }
}

Type t = typeof(Person);

// 获取所有公共构造函数
ConstructorInfo[] ctors = t.GetConstructors();
foreach (var ctor in ctors)
{
    Console.WriteLine($"{ctor} - 参数个数: {ctor.GetParameters().Length}");
}

// 获取私有构造函数（需要 BindingFlags）
ConstructorInfo privateCtor = t.GetConstructor(
    BindingFlags.NonPublic | BindingFlags.Instance,
    null, 
    new Type[] { typeof(int), typeof(string) }, 
    null);
```

### GetMethods()——方法

```csharp
Type t = typeof(string);

// 获取所有公共实例方法（包括继承的）
MethodInfo[] allMethods = t.GetMethods();
foreach (var m in allMethods)
{
    Console.WriteLine(m.Name);
}

// 获取特定方法（需要指定参数类型）
MethodInfo substring = t.GetMethod("Substring", new Type[] { typeof(int), typeof(int) });

// 获取私有方法
MethodInfo privateMethod = t.GetMethod("InternalSubString", 
    BindingFlags.NonPublic | BindingFlags.Instance);
```

**注意：`GetMethods()` 返回的方法包含编译器生成的特殊方法：**
- 属性 getter/setter（`get_Name`, `set_Name`）
- 事件 add/remove（`add_Click`, `remove_Click`）
- 泛型相关方法

### GetProperties()——属性

```csharp
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; private set; }
    public string Description { get; set; }
}

Type t = typeof(Product);

// 获取所有公共属性
PropertyInfo[] props = t.GetProperties();
foreach (var p in props)
{
    Console.WriteLine($"{p.Name}: 可读={p.CanRead}, 可写={p.CanWrite}");
}

// 获取特定属性
PropertyInfo nameProp = t.GetProperty("Name");

// 检查属性是否有 getter/setter
if (nameProp.CanRead) { /* 可以读取 */ }
if (nameProp.CanWrite) { /* 可以写入 */ }

// 获取属性的 GetMethod 和 SetMethod
MethodInfo getter = nameProp.GetMethod;      // public string get_Name()
MethodInfo setter = nameProp.SetMethod;      // public void set_Name(string)
```

### GetFields()——字段

```csharp
public class Config
{
    public const string AppName = "MyApp";
    public static readonly int Version = 1;
    public int NormalField;
    private string _privateField;
}

Type t = typeof(Config);

// 默认只获取公共实例字段（注意：const/static 需要用 BindingFlags）
FieldInfo[] defaultFields = t.GetFields();  // 只有 NormalField

// 获取所有公共字段（包括静态）
FieldInfo[] allPublic = t.GetFields(BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static);
// 得到：AppName, Version, NormalField

// 获取私有字段
FieldInfo privateField = t.GetField("_privateField", 
    BindingFlags.NonPublic | BindingFlags.Instance);
```

### GetEvents()——事件

```csharp
public class Button
{
    public event EventHandler Click;
    public event EventHandler DoubleClick;
    private event EventHandler InternalEvent;
}

Type t = typeof(Button);

// 获取公共事件
EventInfo[] events = t.GetEvents();
foreach (var e in events)
{
    Console.WriteLine(e.Name);
    Console.WriteLine($"  事件类型: {e.EventHandlerType}");
    
    // 获取 add/remove 方法
    MethodInfo addMethod = e.AddMethod;
    MethodInfo removeMethod = e.RemoveMethod;
}
```

### GetInterfaces()——接口

```csharp
Type t = typeof(List<int>);

Type[] interfaces = t.GetInterfaces();
foreach (var i in interfaces)
{
    Console.WriteLine(i.Name);
}
// 输出：
// IList`1
// ICollection`1
// IEnumerable`1
// IEnumerable
// ...
```

### GetMembers()——一次性获取所有成员

```csharp
Type t = typeof(string);

// 获取所有公共成员（方法、属性、字段、事件、嵌套类型）
MemberInfo[] members = t.GetMembers();

foreach (var m in members)
{
    // MemberTypes 枚举：Constructor, Method, Property, Field, Event, NestedType...
    Console.WriteLine($"{m.MemberType}: {m.Name}");
}
```

---

## 六、继承相关的判断方法（重要）

### IsSubclassOf——判断子类关系

```csharp
public class Animal { }
public class Dog : Animal { }
public class Cat : Animal { }

Type animalType = typeof(Animal);
Type dogType = typeof(Dog);
Type catType = typeof(Cat);

Console.WriteLine(dogType.IsSubclassOf(animalType));  // True
Console.WriteLine(catType.IsSubclassOf(animalType));  // True
Console.WriteLine(animalType.IsSubclassOf(animalType)); // False（自己不算是子类）
```

### IsAssignableFrom——判断赋值兼容性

```csharp
// 更强大的判断：a 类型的变量能否指向 b 类型的实例？
// 即：typeof(a).IsAssignableFrom(typeof(b))

Type animalType = typeof(Animal);
Type dogType = typeof(Dog);
Type objectType = typeof(object);

Console.WriteLine(animalType.IsAssignableFrom(dogType));   // True（Animal animal = new Dog()）
Console.WriteLine(dogType.IsAssignableFrom(animalType));   // False（Dog dog = new Animal()）
Console.WriteLine(objectType.IsAssignableFrom(dogType));   // True（object obj = new Dog()）
Console.WriteLine(typeof(IEnumerable).IsAssignableFrom(typeof(List<int>))); // True
```

**区别总结：**

| 方法 | 适用场景 | 包含自身？ |
|------|---------|-----------|
| `IsSubclassOf` | 严格的继承关系（类-类） | ❌ 否 |
| `IsAssignableFrom` | 赋值兼容（类/接口/值类型） | ✅ 是 |

### IsInstanceOfType——判断对象是否为实例

```csharp
Animal animal = new Dog();

Type dogType = typeof(Dog);
Type animalType = typeof(Animal);

Console.WriteLine(dogType.IsInstanceOfType(animal));      // True（animal 实际是 Dog）
Console.WriteLine(animalType.IsInstanceOfType(animal));   // True
Console.WriteLine(dogType.IsInstanceOfType(new Cat()));    // False
```

---

## 七、综合实战：类型浏览器

```csharp
using System.Reflection;

public class TypeBrowser
{
    public static void Browse(Type type)
    {
        Console.WriteLine($"═══════════════════════════════════════");
        Console.WriteLine($"类型名称: {type.Name}");
        Console.WriteLine($"完全限定名: {type.FullName}");
        Console.WriteLine($"命名空间: {type.Namespace ?? "(无)"}");
        Console.WriteLine($"程序集: {type.Assembly.GetName().Name}");
        
        // 类型分类
        Console.WriteLine($"\n【类型分类】");
        Console.WriteLine($"  类: {type.IsClass}");
        Console.WriteLine($"  接口: {type.IsInterface}");
        Console.WriteLine($"  枚举: {type.IsEnum}");
        Console.WriteLine($"  值类型: {type.IsValueType}");
        Console.WriteLine($"  数组: {type.IsArray}");
        Console.WriteLine($"  基元类型: {type.IsPrimitive}");
        
        // 修饰符
        Console.WriteLine($"\n【修饰符】");
        Console.WriteLine($"  Public: {type.IsPublic}");
        Console.WriteLine($"  Abstract: {type.IsAbstract}");
        Console.WriteLine($"  Sealed: {type.IsSealed}");
        
        // 继承
        if (type.BaseType != null && type.BaseType != typeof(object))
        {
            Console.WriteLine($"\n基类: {type.BaseType.Name}");
        }
        
        // 接口
        Type[] interfaces = type.GetInterfaces();
        if (interfaces.Length > 0)
        {
            Console.WriteLine($"\n【实现的接口】");
            foreach (var i in interfaces)
            {
                Console.WriteLine($"  {i.Name}");
            }
        }
        
        // 构造函数
        Console.WriteLine($"\n【构造函数】");
        foreach (var ctor in type.GetConstructors())
        {
            var parameters = ctor.GetParameters();
            string paramList = string.Join(", ", parameters.Select(p => $"{p.ParameterType.Name} {p.Name}"));
            Console.WriteLine($"  {type.Name}({paramList})");
        }
        
        // 方法（只显示自定义方法，过滤掉继承的）
        Console.WriteLine($"\n【方法（仅本类型声明）】");
        var methods = type.GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.DeclaredOnly);
        foreach (var m in methods)
        {
            if (m.IsSpecialName) continue; // 跳过属性/事件的访问器
            var parameters = m.GetParameters();
            string paramList = string.Join(", ", parameters.Select(p => $"{p.ParameterType.Name} {p.Name}"));
            Console.WriteLine($"  {m.ReturnType.Name} {m.Name}({paramList})");
        }
        
        // 属性
        Console.WriteLine($"\n【属性】");
        foreach (var p in type.GetProperties())
        {
            Console.WriteLine($"  {p.PropertyType.Name} {p.Name} {{ {(p.CanRead ? "get;" : "")} {(p.CanWrite ? "set;" : "")} }}");
        }
        
        // 字段
        Console.WriteLine($"\n【字段】");
        foreach (var f in type.GetFields(BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static))
        {
            string staticFlag = f.IsStatic ? "static " : "";
            Console.WriteLine($"  {staticFlag}{f.FieldType.Name} {f.Name}");
        }
    }
}

// 使用示例
TypeBrowser.Browse(typeof(List<string>));
```

---

## 八、常见陷阱与最佳实践

### 陷阱：GetMethod 可能返回 null

```csharp
// ❌ 错误：如果存在重载，GetMethod("Add") 可能返回 null
MethodInfo method = typeof(List<int>).GetMethod("Add");  // 实际上能工作，但不可靠

// ✅ 正确：指定参数类型
MethodInfo method = typeof(List<int>).GetMethod("Add", new Type[] { typeof(int) });
```

### 陷阱：GetProperties 不会返回显式实现接口的属性

```csharp
interface IMyInterface { string Name { get; } }
class MyClass : IMyInterface 
{ 
    string IMyInterface.Name => "explicit";  // 这个得不到
    public string Name => "public";  // 只得到这个
}

Type t = typeof(MyClass);
var props = t.GetProperties();  // 只得到 public Name，得不到显式实现的
```

**解决方案：** 通过接口 Type 获取
```csharp
var interfaceProp = typeof(IMyInterface).GetProperty("Name");
```

### 最佳实践：缓存反射结果

```csharp
public static class TypeCache
{
    private static readonly Dictionary<Type, PropertyInfo[]> _properties = new();
    
    public static PropertyInfo[] GetProperties(Type type)
    {
        lock (_properties)
        {
            if (!_properties.TryGetValue(type, out var props))
            {
                props = type.GetProperties();
                _properties[type] = props;
            }
            return props;
        }
    }
}
```

### 最佳实践：使用 nameof 避免魔法字符串

```csharp
// ❌ 不好：魔法字符串，重构时容易出错
MethodInfo method = t.GetMethod("Add");

// ✅ 好：nameof 编译时检查
MethodInfo method = t.GetMethod(nameof(List<int>.Add));
```

---

## 九、本篇速查表

| 想要获取 | 方法 | 注意事项 |
|---------|------|----------|
| 构造函数 | `GetConstructors()` | 用 `BindingFlags.NonPublic` 获取私有构造 |
| 方法 | `GetMethods()` | 包含属性/事件的 get/set 方法 |
| 属性 | `GetProperties()` | 不包含显式接口实现 |
| 字段 | `GetFields()` | 默认不含静态字段 |
| 事件 | `GetEvents()` | 可通过 `AddMethod`/`RemoveMethod` 获取委托方法 |
| 接口 | `GetInterfaces()` | 包含所有继承的接口 |
| 所有成员 | `GetMembers()` | 最全，但需要过滤 |
| 基类 | `BaseType` | `object` 的基类是 `null` |
| 判断继承 | `IsSubclassOf` / `IsAssignableFrom` | 前者不含自身，后者含 |
| 判断实例 | `IsInstanceOfType` | 等价于 `is` 关键字 |

---

## 十、思考题

1. **`typeof(List<>).GetMethods()` 和 `typeof(List<int>).GetMethods()` 返回的结果有什么不同？**

| **特性**                          | **typeof(List<>)** | **typeof(List<int>)**   |
| ------------------------------- | ------------------ | ----------------------- |
| **术语**                          | 泛型类型定义 (Unbound)   | 已构造的泛型类型 (Bound/Closed) |
| **`ContainsGenericParameters`** | `true`             | `false`                 |
| **方法参数类型**                      | 占位符 `T`            | 明确的 `Int32`             |
| **能否直接 Invoke**                 | **否** (会抛出异常)      | **是**                   |
| **元数据标记**                       | 属于通用模板             | 属于该特定类型的运行时生成实例         |

```cs
var openMethod = typeof(List<>).GetMethod("Add");
var closedMethod = typeof(List<int>).GetMethod("Add");

Console.WriteLine(openMethod.GetParameters()[0].ParameterType.IsGenericParameter); // True (它是 T)
Console.WriteLine(closedMethod.GetParameters()[0].ParameterType.IsGenericParameter); // False (它是 int)

Console.WriteLine(openMethod.GetParameters()[0].ParameterType); // 输出 T
Console.WriteLine(closedMethod.GetParameters()[0].ParameterType); // 输出 System.Int32
```


2. **`IsAssignableFrom` 和 `is` 关键字有什么本质区别？**

操作对象不同
**`is` 关键字**：操作的是**对象实例**。它检查一个具体的“东西”是否兼容于某种类型。
_语义：_ “这个苹果是一个水果吗？”
**`IsAssignableFrom`**：操作的是**类型对象 (`Type`)**。它检查两个“类别”之间是否存在派生或实现关系。
_语义：_ “水果这个分类可以用来存放苹果这个分类的对象吗？”


绑定时间不同
**`is`**：通常在**编译期**确定部分逻辑（尽管结果在运行时判定），语法简洁，常用于类型转换前的安全检查。
**`IsAssignableFrom`**：是**反射 (Reflection)** 的一部分。它完全在**运行期**动态解析。当你只知道两个 `Type` 变量，而没有具体实例时，它是唯一的选择。

核心语法对比

| **特性**      | **obj is T**                   | **typeA.IsAssignableFrom(typeB)** |
| ----------- | ------------------------------ | --------------------------------- |
| **输入**      | 左边是**实例**，右边是**类型名**           | 两边都是 **`Type` 实例**                |
| **Null 处理** | 如果 `obj` 为 `null`，始终返回 `false` | 不涉及实例，仅对比类型定义                     |
| **性能**      | 极快（编译器优化）                      | 较慢（涉及反射元数据查找）                     |
| **适用场景**    | 业务逻辑中的类型判断与转换                  | 插件系统、依赖注入、泛型约束检查                  |

---
