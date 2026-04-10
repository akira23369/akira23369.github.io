---
title: 反射系列笔记（七）：FieldInfo与字段动态操作
date: 2026-04-06 14:00:16
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


# C# 反射系列笔记（七）：FieldInfo 与字段动态操作

> 目标：全面掌握字段的反射操作，理解字段与属性的本质区别，学会处理常量、只读字段，以及字段在序列化/反序列化中的应用

---

## 一、字段的本质：真正的数据容器

### 字段 vs 属性

| 维度 | 字段 (Field) | 属性 (Property) |
|------|-------------|-----------------|
| 本质 | 真正的数据存储位置 | get/set 方法包装 |
| 是否可被验证 | 直接读写，无中间逻辑 | 可在 getter/setter 中添加逻辑 |
| 数据绑定支持 | 部分支持 | 完全支持 |
| 序列化 | 默认序列化字段 | 默认序列化属性 |
| 版本兼容性 | 修改字段名破坏二进制兼容 | 可改变内部实现保持兼容 |

```csharp
public class Comparison
{
    // 字段：直接存储数据
    public string _name;
    
    // 属性：方法包装
    public string Name 
    { 
        get => _name;
        set => _name = value ?? throw new ArgumentNullException();
    }
}

// 反射视角
Type t = typeof(Comparison);
FieldInfo field = t.GetField("_name");
PropertyInfo prop = t.GetProperty("Name");

Console.WriteLine($"字段成员类型: {field.MemberType}");   // Field
Console.WriteLine($"属性成员类型: {prop.MemberType}");     // Property
```

### 字段的元数据结构

```csharp
public class FieldMetadataDemo
{
    public const int Constant = 100;
    public static readonly int StaticReadOnly = 200;
    public static int StaticField = 300;
    public readonly int InstanceReadOnly = 400;
    public int NormalField;
    private int _privateField;
    internal int InternalField;
    protected int ProtectedField;
}

Type t = typeof(FieldMetadataDemo);
foreach (FieldInfo field in t.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                         BindingFlags.Instance | BindingFlags.Static))
{
    Console.WriteLine($"字段名: {field.Name}");
    Console.WriteLine($"  类型: {field.FieldType.Name}");
    Console.WriteLine($"  是否为 public: {field.IsPublic}");
    Console.WriteLine($"  是否为 static: {field.IsStatic}");
    Console.WriteLine($"  是否为 readonly: {field.IsInitOnly}");      // readonly 字段
    Console.WriteLine($"  是否为 literal: {field.IsLiteral}");        // const 字段
    Console.WriteLine($"  是否为 private: {field.IsPrivate}");
    Console.WriteLine($"  是否为 family (protected): {field.IsFamily}");
    Console.WriteLine($"  是否为 assembly (internal): {field.IsAssembly}");
    Console.WriteLine();
}
```

---

## 二、FieldInfo 核心属性

### 字段修饰符判断

```csharp
FieldInfo field = typeof(string).GetField("Empty");  // public static readonly

Console.WriteLine($"IsPublic: {field.IsPublic}");      // True
Console.WriteLine($"IsStatic: {field.IsStatic}");      // True
Console.WriteLine($"IsInitOnly: {field.IsInitOnly}");  // True (readonly)
Console.WriteLine($"IsLiteral: {field.IsLiteral}");    // False (const 才是 true)
Console.WriteLine($"IsFamily: {field.IsFamily}");      // False (protected)
Console.WriteLine($"IsPrivate: {field.IsPrivate}");    // False
Console.WriteLine($"IsAssembly: {field.IsAssembly}");  // False (internal)
```

### 字段值获取与设置的基础能力

```csharp
public class Sample
{
    public int PublicField;
    private string _privateField = "secret";
    public static int StaticField = 10;
    public const int ConstField = 100;
    public readonly int ReadOnlyField = 200;
}

Type t = typeof(Sample);
object obj = Activator.CreateInstance(t);

// 1. 普通字段：可读可写
FieldInfo publicField = t.GetField("PublicField");
publicField.SetValue(obj, 42);
int val = (int)publicField.GetValue(obj);
Console.WriteLine(val);  // 42

// 2. 私有字段：需要 BindingFlags
FieldInfo privateField = t.GetField("_privateField", 
    BindingFlags.NonPublic | BindingFlags.Instance);
string secret = (string)privateField.GetValue(obj);
Console.WriteLine(secret);  // "secret"
privateField.SetValue(obj, "new secret");

// 3. 静态字段：实例参数传 null
FieldInfo staticField = t.GetField("StaticField");
staticField.SetValue(null, 20);
int staticVal = (int)staticField.GetValue(null);
Console.WriteLine(staticVal);  // 20

// 4. const 字段：只能读取，不能修改
FieldInfo constField = t.GetField("ConstField");
int constVal = (int)constField.GetValue(null);  // ✅ 可读
// constField.SetValue(null, 200);  // ❌ FieldAccessException

// 5. readonly 字段：实例构造后可读，但反射可修改（⚠️ 危险操作）
FieldInfo readonlyField = t.GetField("ReadOnlyField");
int readonlyVal = (int)readonlyField.GetValue(obj);  // 200
// 虽然可以修改，但强烈不推荐
readonlyField.SetValue(obj, 999);  // ⚠️ 会成功，但破坏语义！
```

---

## 三、特殊字段类型处理

### const 字段（编译时常量）

```csharp
public class ConstDemo
{
    public const int IntConst = 42;
    public const string StringConst = "Hello";
    public const double DoubleConst = 3.14;
}

Type t = typeof(ConstDemo);

FieldInfo intConst = t.GetField("IntConst");
Console.WriteLine($"IsLiteral: {intConst.IsLiteral}");  // True
Console.WriteLine($"IsInitOnly: {intConst.IsInitOnly}"); // False

// const 字段的值在编译时就嵌入到 IL 中
// 反射读取时直接从元数据获取
int value = (int)intConst.GetValue(null);
Console.WriteLine(value);  // 42

// ⚠️ 重要：如果修改了 const 值，引用它的程序集需要重新编译
// 因为值在编译时被复制到了引用代码中
```

### readonly 字段

```csharp
public class ReadOnlyDemo
{
    public readonly int InitializedInDeclaration = 100;
    public readonly int InitializedInConstructor;
    
    public ReadOnlyDemo(int value)
    {
        InitializedInConstructor = value;
    }
}

Type t = typeof(ReadOnlyDemo);

FieldInfo declaredField = t.GetField("InitializedInDeclaration");
Console.WriteLine($"IsInitOnly: {declaredField.IsInitOnly}");  // True

// 正常情况：实例创建后不能修改
object obj = Activator.CreateInstance(t, 200);
int val = (int)declaredField.GetValue(obj);
Console.WriteLine(val);  // 100

// ⚠️ 反射可以绕过 readonly 限制（破坏语义，慎用）
declaredField.SetValue(obj, 999);
Console.WriteLine(declaredField.GetValue(obj));  // 999
```

### 静态构造函数中的字段

```csharp
public class StaticCtorDemo
{
    public static readonly int StaticReadOnly;
    public static int StaticField;
    
    static StaticCtorDemo()
    {
        StaticReadOnly = 100;
        StaticField = 200;
    }
}

// 注意：静态构造函数在首次访问类型时执行
// 反射不会自动触发静态构造函数？
Type t = typeof(StaticCtorDemo);

// 触发静态构造函数的方式
RuntimeHelpers.RunClassConstructor(t.TypeHandle);  // 显式触发

FieldInfo readonlyField = t.GetField("StaticReadOnly");
Console.WriteLine(readonlyField.GetValue(null));  // 100
```

### 枚举字段

```csharp
public enum Color
{
    Red = 1,
    Green = 2,
    Blue = 3
}

Type enumType = typeof(Color);

// 枚举本身有隐藏的 value__ 字段
FieldInfo[] fields = enumType.GetFields();
foreach (FieldInfo field in fields)
{
    Console.WriteLine($"{field.Name}: IsStatic={field.IsStatic}, IsSpecialName={field.IsSpecialName}");
}
// 输出：
// value__: IsStatic=False, IsSpecialName=True
// Red: IsStatic=True, IsSpecialName=False
// Green: IsStatic=True, IsSpecialName=False
// Blue: IsStatic=True, IsSpecialName=False

// 获取枚举值
FieldInfo redField = enumType.GetField("Red");
object redValue = redField.GetValue(null);
Console.WriteLine(redValue);  // Red
Console.WriteLine((int)redValue);  // 1

// 通过枚举数值创建枚举值
object blue = Enum.ToObject(enumType, 3);
Console.WriteLine(blue);  // Blue
```

---

## 四、字段与序列化

### 字段的序列化特性

```csharp
using System.Runtime.Serialization;

[Serializable]
public class SerializableDemo
{
    public int PublicField;                    // 会序列化
    
    [NonSerialized]
    public int NonSerializedField;             // 不会序列化
    
    private int _privateField;                 // 默认会序列化（BinaryFormatter）
    
    [field: NonSerialized]
    public event EventHandler MyEvent;         // 事件默认不序列化字段
}

// 自定义序列化逻辑
[Serializable]
public class CustomSerialization : ISerializable
{
    public int Value;
    public string Name;
    
    public CustomSerialization(int value, string name)
    {
        Value = value;
        Name = name;
    }
    
    protected CustomSerialization(SerializationInfo info, StreamingContext context)
    {
        // 反序列化：反射读取字段
        Value = info.GetInt32(nameof(Value));
        Name = info.GetString(nameof(Name));
    }
    
    public void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        // 序列化：反射写入字段
        info.AddValue(nameof(Value), Value);
        info.AddValue(nameof(Name), Name);
    }
}
```

### 手动实现序列化（基于字段反射）

```csharp
public class SimpleSerializer
{
    public static Dictionary<string, object> Serialize(object obj)
    {
        var result = new Dictionary<string, object>();
        Type type = obj.GetType();
        
        // 获取所有实例字段（包括私有）
        var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                     BindingFlags.Instance);
        
        foreach (var field in fields)
        {
            // 跳过标记了 NonSerialized 的字段
            if (field.IsDefined(typeof(NonSerializedAttribute), false))
                continue;
            
            object value = field.GetValue(obj);
            result[field.Name] = value;
        }
        
        return result;
    }
    
    public static T Deserialize<T>(Dictionary<string, object> data) where T : new()
    {
        T obj = new T();
        Type type = typeof(T);
        
        var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                     BindingFlags.Instance);
        
        foreach (var field in fields)
        {
            if (!data.TryGetValue(field.Name, out object value))
                continue;
            
            // 类型转换
            object converted = Convert.ChangeType(value, field.FieldType);
            field.SetValue(obj, converted);
        }
        
        return obj;
    }
}

// 使用
public class Person
{
    public string Name;
    private int _age;
    [NonSerialized] public string TempData;
    
    public Person(string name, int age)
    {
        Name = name;
        _age = age;
    }
}

var person = new Person("张三", 25);
var data = SimpleSerializer.Serialize(person);
var restored = SimpleSerializer.Deserialize<Person>(data);
```

---

## 五、性能优化：字段访问委托

### 通用字段访问委托工厂

```csharp
public static class FieldDelegateFactory
{
    public static Func<object, object> CreateGetter(FieldInfo field)
    {
        var instanceParam = Expression.Parameter(typeof(object), "instance");
        var castInstance = Expression.Convert(instanceParam, field.DeclaringType);
        var fieldAccess = Expression.Field(castInstance, field);
        var convertResult = Expression.Convert(fieldAccess, typeof(object));
        
        var lambda = Expression.Lambda<Func<object, object>>(convertResult, instanceParam);
        return lambda.Compile();
    }
    
    public static Action<object, object> CreateSetter(FieldInfo field)
    {
        var instanceParam = Expression.Parameter(typeof(object), "instance");
        var valueParam = Expression.Parameter(typeof(object), "value");
        var castInstance = Expression.Convert(instanceParam, field.DeclaringType);
        var castValue = Expression.Convert(valueParam, field.FieldType);
        var fieldAccess = Expression.Field(castInstance, field);
        var assign = Expression.Assign(fieldAccess, castValue);
        
        var lambda = Expression.Lambda<Action<object, object>>(assign, instanceParam, valueParam);
        return lambda.Compile();
    }
    
    public static Func<T, TValue> CreateStrongGetter<T, TValue>(string fieldName)
    {
        FieldInfo field = typeof(T).GetField(fieldName, 
            BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
        
        var instanceParam = Expression.Parameter(typeof(T), "instance");
        var fieldAccess = Expression.Field(instanceParam, field);
        var lambda = Expression.Lambda<Func<T, TValue>>(fieldAccess, instanceParam);
        return lambda.Compile();
    }
    
    public static Action<T, TValue> CreateStrongSetter<T, TValue>(string fieldName)
    {
        FieldInfo field = typeof(T).GetField(fieldName,
            BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
        
        var instanceParam = Expression.Parameter(typeof(T), "instance");
        var valueParam = Expression.Parameter(typeof(TValue), "value");
        var fieldAccess = Expression.Field(instanceParam, field);
        var assign = Expression.Assign(fieldAccess, valueParam);
        
        var lambda = Expression.Lambda<Action<T, TValue>>(assign, instanceParam, valueParam);
        return lambda.Compile();
    }
}

// 使用
class Demo { private int _secret = 100; }

var getter = FieldDelegateFactory.CreateStrongGetter<Demo, int>("_secret");
var setter = FieldDelegateFactory.CreateStrongSetter<Demo, int>("_secret");

Demo d = new Demo();
Console.WriteLine(getter(d));  // 100
setter(d, 200);
Console.WriteLine(getter(d));  // 200
```

---

## 六、字段遍历与批量操作

### 遍历所有字段

```csharp
public static class FieldWalker
{
    public static void WalkFields(object obj, Action<FieldInfo, object> action)
    {
        Type type = obj.GetType();
        var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                     BindingFlags.Instance | BindingFlags.Static);
        
        foreach (var field in fields)
        {
            object value = field.GetValue(obj);
            action(field, value);
        }
    }
    
    public static void SetAllFields<T>(object obj, T value)
    {
        Type type = obj.GetType();
        var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                     BindingFlags.Instance);
        
        foreach (var field in fields)
        {
            if (field.FieldType.IsAssignableFrom(typeof(T)))
            {
                field.SetValue(obj, value);
            }
        }
    }
    
    public static Dictionary<string, object> GetFieldValues(object obj)
    {
        var result = new Dictionary<string, object>();
        Type type = obj.GetType();
        var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                     BindingFlags.Instance);
        
        foreach (var field in fields)
        {
            result[field.Name] = field.GetValue(obj);
        }
        
        return result;
    }
}
```

### 深度克隆（基于字段）

type.GetElementType();表示这个数组 / 指针里装的是什么类型

```csharp
public static class DeepCopyUtils
{
    public static T DeepClone<T>(T obj)
    {
        // 使用字典记录已克隆的对象，处理循环引用
        var visited = new Dictionary<object, object>(ReferenceComparer.Instance);
        return (T)CloneInternal(obj, visited);
    }

    private static object CloneInternal(object obj, Dictionary<object, object> visited)
    {
        if (obj == null) return null;

        var type = obj.GetType();

        // 1. 如果是值类型（int, struct等）或字符串，直接返回（字符串在C#中虽是引用但行为类似值）
        if (type.IsValueType || type == typeof(string))
        {
            return obj;
        }

        // 2. 检查循环引用：如果该对象已经克隆过，直接返回克隆后的引用
        if (visited.ContainsKey(obj))
        {
            return visited[obj];
        }

        // 3. 处理数组
        if (type.IsArray)
        {
            var elementType = type.GetElementType();
            var array = (Array)obj;
            var clonedArray = Array.CreateInstance(elementType, array.Length);
            visited[obj] = clonedArray; // 先存入字典，再递归填充元素

            for (int i = 0; i < array.Length; i++)
            {
                clonedArray.SetValue(CloneInternal(array.GetValue(i), visited), i);
            }
            return clonedArray;
        }

        // 4. 处理普通对象
        // 创建新实例（避开构造函数，防止逻辑干扰）
        var clonedObj = RuntimeHelpers.GetUninitializedObject(type);
        visited[obj] = clonedObj;

        // 递归克隆字段（包含私有和继承字段）
        while (type != null)
        {
            // 获取所有实例字段
            var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
            foreach (var field in fields)
            {
                var fieldValue = field.GetValue(obj);
                field.SetValue(clonedObj, CloneInternal(fieldValue, visited));
            }
            // 移动到父类，确保处理 private 字段
            type = type.BaseType;
        }

        return clonedObj;
    }
}

// 辅助类：确保通过引用地址比较对象，而非 Equals 重载
// 有时用户会自定义比较自己的类，我们必须绕过所有用户自定义的 `Equals` 逻辑，强制回到“内存地址”这一唯一的客观标准上。
public class ReferenceComparer : IEqualityComparer<object>
{
    public static readonly ReferenceComparer Instance = new ReferenceComparer();
    bool IEqualityComparer<object>.Equals(object x, object y) => ReferenceEquals(x, y);
    int IEqualityComparer<object>.GetHashCode(object obj) => System.Runtime.CompilerServices.RuntimeHelpers.GetHashCode(obj);
}

// 使用
public class Address
{
    public string City;
    public string Street;
}

public class Employee
{
    public string Name;
    public int Age;
    public Address Address;
}

var emp = new Employee
{
    Name = "张三",
    Age = 30,
    Address = new Address { City = "北京", Street = "长安街" }
};

var cloned = DeepCopyUtils.DeepClone(emp);
cloned.Address.City = "上海";

Console.WriteLine(emp.Address.City);    // 北京（未受影响）
Console.WriteLine(cloned.Address.City); // 上海
```

关于上面为什么type = type.BaseType;
核心是要复制从父类继承来的私有字段

```cs
public class Animal {
    private int _age = 5; // 父类的私有字段
}

public class Dog : Animal {
    private string _breed = "Labrador"; // 子类的私有字段
}
```

如果你克隆一个 `Dog` 对象：
1. **第一次循环：** `type` 是 `Dog`。`GetFields` 能找到 `_breed`，但**找不到** `_age`。
2. **执行 `type = type.BaseType`：** `type` 变成了 `Animal`。
3. **第二次循环：** `GetFields` 现在针对 `Animal` 类型运行，这次就能找到 `_age` 了。
4. **再次执行：** `type` 变成 `System.Object`，然后变成 `null`，循环结束。


---

## 七、字段 vs 属性的选择与互操作

### 同时获取字段和属性

```csharp
public static class MemberExtractor
{
    public static IEnumerable<(string Name, Type Type, object Value, bool IsProperty)> 
        GetAllMembers(object obj)
    {
        Type type = obj.GetType();
        
        // 获取字段
        foreach (var field in type.GetFields(BindingFlags.Public | BindingFlags.Instance))
        {
            yield return (field.Name, field.FieldType, field.GetValue(obj), false);
        }
        
        // 获取属性
        foreach (var prop in type.GetProperties(BindingFlags.Public | BindingFlags.Instance))
        {
            if (prop.CanRead)
            {
                yield return (prop.Name, prop.PropertyType, prop.GetValue(obj), true);
            }
        }
    }
}

// 使用
var person = new { Name = "张三", Age = 25 };
foreach (var member in MemberExtractor.GetAllMembers(person))
{
    Console.WriteLine($"{(member.IsProperty ? "属性" : "字段")}: {member.Name} = {member.Value}");
}
```

### 字段到属性的适配器

```csharp
public class PropertyToFieldAdapter
{
    private readonly object _target;
    private readonly Dictionary<string, FieldInfo> _fields;
    
    public PropertyToFieldAdapter(object target)
    {
        _target = target;
        _fields = target.GetType()
            .GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance)
            .ToDictionary(f => f.Name);
    }
    
    public object this[string name]
    {
        get => _fields.TryGetValue(name, out var field) ? field.GetValue(_target) : null;
        set
        {
            if (_fields.TryGetValue(name, out var field))
            {
                object converted = Convert.ChangeType(value, field.FieldType);
                field.SetValue(_target, converted);
            }
        }
    }
}


// 使用
// 假设你有一个第三方库提供的类 `BankCard`，它的关键字段是私有的，且没有提供公开的属性（Property）修改方法：
public class BankCard
{
    private string _cardNumber = "1234-5678";
    private decimal _balance = 100.0m;

    public void ShowStatus() 
    {
        Console.WriteLine($"卡号: {_cardNumber}, 余额: {_balance}");
    }
}

var myCard = new BankCard();
var adapter = new PropertyToFieldAdapter(myCard);

// 1. 动态读取私有字段
var balance = adapter["_balance"];
Console.WriteLine($"当前余额: {balance}"); // 输出 100.0

// 2. 动态写入私有字段（甚至能自动处理类型转换）
// 注意：即使输入的是字符串 "999.99"，Convert.ChangeType 也会把它转成 decimal
adapter["_balance"] = "999.99"; 
adapter["_cardNumber"] = "8888-8888";

// 3. 验证结果
myCard.ShowStatus(); // 输出：卡号: 8888-8888, 余额: 999.99

```

---

## 八、常见陷阱与最佳实践

### 陷阱：值类型字段修改

```csharp
struct MutableStruct
{
    public int Value;
}

MutableStruct s = new MutableStruct { Value = 10 };
object boxed = s;
FieldInfo field = typeof(MutableStruct).GetField("Value");

// ❌ 错误：修改的是装箱副本
field.SetValue(boxed, 20);
Console.WriteLine(s.Value);  // 还是 10

// ✅ 正确：拆箱后重新装箱
boxed = s;
field.SetValue(boxed, 20);
s = (MutableStruct)boxed;
Console.WriteLine(s.Value);  // 20
```

### 陷阱：泛型类型的静态字段

```cs
public class Counter<T>
{
    public static int Count = 0;
}

// 看起来我们在累加同一个计数器
Counter<int>.Count++;
Counter<int>.Count++;
Counter<string>.Count++; 

Console.WriteLine(Counter<int>.Count);    // 输出 2
Console.WriteLine(Counter<string>.Count); // 输出 1（不是 3！）
```
**为什么会这样？**
在运行时（CLR），每当你为泛型提供一套不同的类型参数时，它都会生成一个**全新的、闭合的类型**。
- `Counter<int>` 是一个类型。
- `Counter<string>` 是另一个完全不同的类型。
就像 `Dog` 类和 `Cat` 类各自拥有独立的静态成员一样，这两个闭合类型也各自拥有自己的静态字段存储区。

**解决方案？**
这是最标准、最干净的做法。将需要共享的静态字段定义在一个**非泛型**的基类中。
```cs
// 1. 定义非泛型基类存储静态成员
public abstract class CounterBase
{
    public static int SharedCount = 0;
}

// 2. 泛型类继承它
public class Counter<T> : CounterBase
{
    public void Increment() => SharedCount++;
}

// 这样无论 T 是什么，SharedCount 永远只有一份
```

在反射中也是如此
```csharp
public class GenericStatic<T>
{
    public static int Counter;  // 每个封闭泛型类型有自己的静态字段
}

GenericStatic<int>.Counter = 10;
GenericStatic<string>.Counter = 20;

Type intType = typeof(GenericStatic<int>);
Type stringType = typeof(GenericStatic<string>);

FieldInfo counterField = typeof(GenericStatic<>).GetField("Counter");

// 注意：必须通过封闭类型获取字段值
// int intCounter = (int)counterField.GetValue(null);  // ❌ 错误！泛型定义没有静态字段

// ✅ 正确
int intCounterCorrect = (int)intType.GetField("Counter").GetValue(null);  // 10
int stringCounter = (int)stringType.GetField("Counter").GetValue(null);   // 20
```

### 最佳实践：字段访问权限设计

```csharp
// 推荐：使用属性而不是公共字段
// 原因：
// 1. 可以在 setter 中添加验证逻辑
// 2. 可以保持二进制兼容性（内部字段改名不影响调用方）
// 3. 数据绑定、序列化等框架支持更好

// ❌ 不推荐
public class BadDesign
{
    public string Name;  // 公共字段
}

// ✅ 推荐
public class GoodDesign
{
    private string _name;
    public string Name
    {
        get => _name;
        set => _name = value ?? throw new ArgumentNullException();
    }
}

// 反射时的判断
bool shouldUseProperty = propInfo != null && propInfo.CanRead && propInfo.CanWrite;
bool shouldUseField = fieldInfo != null && !fieldInfo.IsInitOnly && !fieldInfo.IsLiteral;
```

---

## 九、完整实战：通用 Diff 工具

**这段代码就是一个「对象对比工具」，专门用来找出两个同类型对象的哪些字段 / 属性不一样**。

```csharp
public class FieldDiffResult
{
    public string MemberName { get; set; }
    public object OldValue { get; set; }
    public object NewValue { get; set; }
    public bool IsField { get; set; }
}

public static class ObjectDiff
{
    public static List<FieldDiffResult> Compare(object oldObj, object newObj)
    {
        if (oldObj == null || newObj == null)
            throw new ArgumentNullException();
        
        if (oldObj.GetType() != newObj.GetType())
            throw new ArgumentException("对象类型不同");
        
        var results = new List<FieldDiffResult>();
        Type type = oldObj.GetType();
        
        // 比较字段
        var fields = type.GetFields(BindingFlags.Public | BindingFlags.NonPublic | 
                                     BindingFlags.Instance);
        foreach (var field in fields)
        {
            object oldValue = field.GetValue(oldObj);
            object newValue = field.GetValue(newObj);
            
            if (!Equals(oldValue, newValue))
            {
                results.Add(new FieldDiffResult
                {
                    MemberName = field.Name,
                    OldValue = oldValue,
                    NewValue = newValue,
                    IsField = true
                });
            }
        }
        
        // 比较属性（可读的公共属性）
        var props = type.GetProperties(BindingFlags.Public | BindingFlags.Instance);
        foreach (var prop in props)
        {
            if (!prop.CanRead) continue;
            
            object oldValue = prop.GetValue(oldObj);
            object newValue = prop.GetValue(newObj);
            
            if (!Equals(oldValue, newValue))
            {
                results.Add(new FieldDiffResult
                {
                    MemberName = prop.Name,
                    OldValue = oldValue,
                    NewValue = newValue,
                    IsField = false
                });
            }
        }
        
        return results;
    }
}

// 使用
public class Order
{
    public int Id;
    public string CustomerName;
    public decimal Amount { get; set; }
}

var oldOrder = new Order { Id = 1, CustomerName = "张三", Amount = 100m };
var newOrder = new Order { Id = 1, CustomerName = "李四", Amount = 150m };

var diffs = ObjectDiff.Compare(oldOrder, newOrder);
foreach (var diff in diffs)
{
    Console.WriteLine($"{(diff.IsField ? "字段" : "属性")} {diff.MemberName}: {diff.OldValue} -> {diff.NewValue}");
}
```

---

## 十、本篇速查表

| 需求 | 代码 |
|------|------|
| 获取字段 | `typeof(T).GetField("fieldName")` |
| 读取实例字段 | `field.GetValue(instance)` |
| 写入实例字段 | `field.SetValue(instance, value)` |
| 静态字段 | 实例参数传 `null` |
| const 字段 | `IsLiteral = true`，只读 |
| readonly 字段 | `IsInitOnly = true`，反射可修改（不推荐） |
| 私有字段 | 需要 `BindingFlags.NonPublic` |
| 枚举字段 | 通过 `Enum.ToObject` 或静态字段获取 |
| 判断字段类型 | `field.FieldType` |
| 高性能访问 | 表达式树编译委托 |
| 深度克隆 | 递归遍历所有字段 |

---

## 十一、思考题

1. 为什么 const 字段的值在编译时就确定了？这对反射有什么影响？

2. 泛型类的静态字段在不同封闭类型之间是共享的还是独立的？为什么？

3. 如何用反射实现一个通用的对象初始化器（类似 `new { Name = "xxx" }` 的匿名对象效果）？

4. 字段和属性在序列化时有什么不同的行为？BinaryFormatter 默认会序列化私有字段吗？

---

**下一篇预告：** 《C# 反射系列笔记（八）：EventInfo 与事件动态操作》

下一篇将深入讲解事件的反射操作，包括事件的 add/remove 方法、自定义事件实现、事件委托的动态绑定，以及实现事件总线的基础。