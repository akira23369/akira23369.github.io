---
title: 反射系列笔记（六）：PropertyInfo与属性动态操作
date: 2026-04-06 13:52:18
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


# C# 反射系列笔记（六）：PropertyInfo 与属性动态操作

> 目标：全面掌握属性的反射读写，理解属性与方法的本质关系，实现通用的对象映射和数据绑定

---

## 一、属性的本质：语法糖

### 属性是什么？

```csharp
// 你写的代码（语法糖）
public class Person
{
    public string Name { get; set; }
}

// 编译器实际生成的代码
public class Person
{
    private string _Name;  // 编译器生成的隐藏字段
    
    public string get_Name() { return _Name; }  // getter 方法
    public void set_Name(string value) { _Name = value; }  // setter 方法
}
```

**核心理解：**
- **属性不是字段**，本质上是 **get_XXX** 和 **set_XXX** 两个方法
- `PropertyInfo` 是这两个方法的“包装器”
- 反射操作属性，本质是在调用这两个方法

### 验证属性本质

```csharp
public class Demo
{
    public string Name { get; set; }
}

Type t = typeof(Demo);

// 查看方法中是否包含 get_Name 和 set_Name
MethodInfo[] methods = t.GetMethods(BindingFlags.Public | BindingFlags.Instance);
foreach (var m in methods.Where(m => m.IsSpecialName))
{
    Console.WriteLine($"特殊方法: {m.Name}");
}
// 输出：
// 特殊方法: get_Name
// 特殊方法: set_Name

// PropertyInfo 暴露了这两个方法
PropertyInfo prop = t.GetProperty("Name");
MethodInfo getter = prop.GetMethod;    // get_Name
MethodInfo setter = prop.SetMethod;    // set_Name
Console.WriteLine($"Getter: {getter?.Name}, Setter: {setter?.Name}");
```

---

## 二、PropertyInfo 核心属性

### 基本信息

```csharp
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; private set; }
    public static string Category { get; set; }
    public string Description { get; }
}

Type t = typeof(Product);

foreach (PropertyInfo prop in t.GetProperties())
{
    Console.WriteLine($"属性名: {prop.Name}");
    Console.WriteLine($"  类型: {prop.PropertyType.Name}");
    Console.WriteLine($"  可读: {prop.CanRead}");
    Console.WriteLine($"  可写: {prop.CanWrite}");
    Console.WriteLine($"  是否为静态: {prop.GetMethod?.IsStatic ?? prop.SetMethod?.IsStatic ?? false}");
    Console.WriteLine($"  是否为索引器: {prop.GetIndexParameters().Length > 0}");
    Console.WriteLine();
}
```

### GetMethod 和 SetMethod

```csharp
PropertyInfo prop = typeof(Product).GetProperty("Price");

MethodInfo getter = prop.GetMethod;      // 获取 getter 方法
MethodInfo setter = prop.SetMethod;      // 获取 setter 方法

// 判断访问级别
if (setter != null && setter.IsPublic)
    Console.WriteLine("Setter 是 public");
else if (setter != null && setter.IsPrivate)
    Console.WriteLine("Setter 是 private");  // Price 的 setter 是 private

// 判断是否为自动实现的属性
bool isAutoProperty = getter != null && getter.IsDefined(typeof(CompilerGeneratedAttribute), false);
```

---

## 三、属性的读写操作

### 基础读写

```csharp
public class Student
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string School { get; set; } = "Unknown";
}

Type studentType = typeof(Student);
object student = Activator.CreateInstance(studentType);

// 写入属性值
PropertyInfo nameProp = studentType.GetProperty("Name");
nameProp.SetValue(student, "张三");

PropertyInfo ageProp = studentType.GetProperty("Age");
ageProp.SetValue(student, 20);

// 读取属性值
string name = (string)nameProp.GetValue(student);
int age = (int)ageProp.GetValue(student);

Console.WriteLine($"姓名: {name}, 年龄: {age}");  // 姓名: 张三, 年龄: 20
```

### 静态属性

```csharp
public class Config
{
    public static string AppName { get; set; } = "MyApp";
    public static int Version { get; private set; } = 1;
}

Type configType = typeof(Config);

// 静态属性：实例参数传 null
PropertyInfo appNameProp = configType.GetProperty("AppName");
appNameProp.SetValue(null, "NewAppName");
string appName = (string)appNameProp.GetValue(null);

// 私有 setter 的静态属性：只能读不能写
PropertyInfo versionProp = configType.GetProperty("Version");
int version = (int)versionProp.GetValue(null);  // 可读
// versionProp.SetValue(null, 2);  // 运行时异常：setter 不可访问
```

### 只读属性（只有 getter）

```csharp
public class ReadOnlyDemo
{
    public string Computed => DateTime.Now.ToString();
    public string Backed { get; } = "fixed value";
}

Type t = typeof(ReadOnlyDemo);
object obj = Activator.CreateInstance(t);

PropertyInfo computedProp = t.GetProperty("Computed");
string computedValue = (string)computedProp.GetValue(obj);  // ✅ 可读

PropertyInfo backedProp = t.GetProperty("Backed");
string backedValue = (string)backedProp.GetValue(obj);  // ✅ 可读

// ❌ 以下会抛出异常：Property set method not found
// backedProp.SetValue(obj, "new value");
```

### 私有属性的访问（需 BindingFlags）

```csharp
public class Secret
{
    private string Hidden { get; set; } = "secret";
    protected string ProtectedProp { get; set; } = "protected";
}

Type secretType = typeof(Secret);
object secret = Activator.CreateInstance(secretType);

// 获取私有属性
PropertyInfo hiddenProp = secretType.GetProperty("Hidden", 
    BindingFlags.NonPublic | BindingFlags.Instance);

// 读写私有属性
hiddenProp.SetValue(secret, "new secret");
string hiddenValue = (string)hiddenProp.GetValue(secret);
Console.WriteLine(hiddenValue);  // "new secret"

// 获取 protected 属性
PropertyInfo protectedProp = secretType.GetProperty("ProtectedProp",
    BindingFlags.NonPublic | BindingFlags.Instance);
```

---

## 四、索引器（Indexer）的反射操作

### 索引器的本质
编译器会将索引器重命名为一个名为 **`Item`** 的特殊属性，并自动生成对应的 **`get_Item`** 和 **`set_Item`** 方法。



```csharp
public class StringCollection
{
    private List<string> _items = new List<string>();
    
    // 索引器
    public string this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }
    
    public void Add(string item) => _items.Add(item);
}

// 索引器在元数据中的表示
Type t = typeof(StringCollection);
PropertyInfo indexerProp = t.GetProperty("Item");  // 默认名称是 "Item"
Console.WriteLine($"是否为索引器: {indexerProp.GetIndexParameters().Length > 0}");

ParameterInfo[] indexParams = indexerProp.GetIndexParameters();
Console.WriteLine($"索引参数类型: {indexParams[0].ParameterType.Name}");
```

在常规属性（Property）中，`GetIndexParameters()` 通常返回一个空数组；但在**索引器**中，它会返回一个包含索引参数信息的数组。它是区分“普通属性”与“索引器”的金标准。

**普通属性**（如 `public int Id { get; set; }`）：没有参数。
**索引器**（如 `public string this[int i, string label] { ... }`）：有参数。


### 调用索引器

```csharp
Type t = typeof(StringCollection);
object collection = Activator.CreateInstance(t);

// 添加数据（通过普通方法）
MethodInfo addMethod = t.GetMethod("Add");
addMethod.Invoke(collection, new object[] { "first" });
addMethod.Invoke(collection, new object[] { "second" });

// 通过索引器读取
PropertyInfo indexer = t.GetProperty("Item");
// firstValue = coolection[0];
object firstValue = indexer.GetValue(collection, new object[] { 0 });
Console.WriteLine(firstValue);  // "first"

// 通过索引器写入
// collection[0] = "modified";
indexer.SetValue(collection, "modified", new object[] { 0 });

// 验证
object modifiedValue = indexer.GetValue(collection, new object[] { 0 });
Console.WriteLine(modifiedValue);  // "modified"
```

### 多维索引器

```csharp
public class Matrix
{
    private int[,] _data = new int[3, 3];
    
    public int this[int x, int y]
    {
        get => _data[x, y];
        set => _data[x, y] = value;
    }
}

Type matrixType = typeof(Matrix);
object matrix = Activator.CreateInstance(matrixType);

PropertyInfo indexer = matrixType.GetProperty("Item");

// 多维索引器：传入多个参数
// matrix[0,0] = 100;
indexer.SetValue(matrix, 100, new object[] { 0, 0 });
int value = (int)indexer.GetValue(matrix, new object[] { 0, 0 });
Console.WriteLine(value);  // 100
```

---

## 五、自动实现属性的幕后

### 识别自动实现属性

```csharp
public class AutoPropDemo
{
    public string Normal { get; set; }
    public string ReadOnly { get; }
    private string Private { get; set; }
    
    public string Custom
    {
        get => _custom;
        set => _custom = value;
    }
    private string _custom;
}

Type t = typeof(AutoPropDemo);

foreach (PropertyInfo prop in t.GetProperties(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance))
{
    // 检查是否有 CompilerGeneratedAttribute
    bool isAutoProperty = prop.GetMethod?.IsDefined(typeof(CompilerGeneratedAttribute), false) == true
                       || prop.SetMethod?.IsDefined(typeof(CompilerGeneratedAttribute), false) == true;
    
    Console.WriteLine($"{prop.Name}: {(isAutoProperty ? "自动实现" : "手动实现")}");
    
    // 如果是自动实现属性，可以找到对应的 backing field
    if (isAutoProperty)
    {
        string backingFieldName = $"<{prop.Name}>k__BackingField";
        FieldInfo backingField = t.GetField(backingFieldName, 
            BindingFlags.NonPublic | BindingFlags.Instance);
        if (backingField != null)
        {
            Console.WriteLine($"  对应的后备字段: {backingField.Name}");
        }
    }
}
```

### 直接操作后备字段（不推荐）

```csharp
// ⚠️ 警告：直接操作后备字段是危险的，框架升级可能改变命名规则
// 仅用于演示目的

public class Demo
{
    public string Name { get; set; }
}

Type t = typeof(Demo);
object obj = new Demo();

// 找到后备字段
FieldInfo backingField = t.GetField("<Name>k__BackingField", 
    BindingFlags.NonPublic | BindingFlags.Instance);

// 直接修改字段（绕过属性逻辑）
backingField.SetValue(obj, "直接设置");

// 通过属性读取验证
PropertyInfo prop = t.GetProperty("Name");
string value = (string)prop.GetValue(obj);
Console.WriteLine(value);  // "直接设置"

// ✅ 推荐：始终通过 PropertyInfo 操作
prop.SetValue(obj, "正确方式");
```

---

## 六、性能优化：Getter/Setter 委托

### Delegate.CreateDelegate 方式

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

// 通用 Getter 委托
public static Func<object, object> CreateGetter(PropertyInfo property)
{
    MethodInfo getter = property.GetMethod;
    if (getter == null)
        throw new InvalidOperationException("属性没有 getter");
    
    // 创建委托：object Getter(object instance)
    var instanceParam = Expression.Parameter(typeof(object), "instance");
    var castInstance = Expression.Convert(instanceParam, property.DeclaringType);
    var propertyAccess = Expression.Property(castInstance, property);
    var convertResult = Expression.Convert(propertyAccess, typeof(object));
    
    var lambda = Expression.Lambda<Func<object, object>>(convertResult, instanceParam);
    return lambda.Compile();
}

// 通用 Setter 委托
public static Action<object, object> CreateSetter(PropertyInfo property)
{
    MethodInfo setter = property.SetMethod;
    if (setter == null)
        throw new InvalidOperationException("属性没有 setter");
    
    // 创建委托：void Setter(object instance, object value)
    var instanceParam = Expression.Parameter(typeof(object), "instance");
    var valueParam = Expression.Parameter(typeof(object), "value");
    var castInstance = Expression.Convert(instanceParam, property.DeclaringType);
    var castValue = Expression.Convert(valueParam, property.PropertyType);
    var propertyAccess = Expression.Property(castInstance, property);
    var assign = Expression.Assign(propertyAccess, castValue);
    
    var lambda = Expression.Lambda<Action<object, object>>(assign, instanceParam, valueParam);
    return lambda.Compile();
}

// 使用
Person person = new Person();
PropertyInfo nameProp = typeof(Person).GetProperty("Name");

var fastSetter = CreateSetter(nameProp);
var fastGetter = CreateGetter(nameProp);

fastSetter(person, "张三");
string name = (string)fastGetter(person);
Console.WriteLine(name);  // "张三"
```

### 强类型委托（性能最优）

```csharp
public static class PropertyDelegateFactory
{
    public static Func<T, TValue> CreateGetter<T, TValue>(string propertyName)
    {
        PropertyInfo prop = typeof(T).GetProperty(propertyName);
        MethodInfo getter = prop.GetMethod;
        
        var instanceParam = Expression.Parameter(typeof(T), "instance");
        var propertyAccess = Expression.Property(instanceParam, prop);
        var lambda = Expression.Lambda<Func<T, TValue>>(propertyAccess, instanceParam);
        return lambda.Compile();
    }
    
    public static Action<T, TValue> CreateSetter<T, TValue>(string propertyName)
    {
        PropertyInfo prop = typeof(T).GetProperty(propertyName);
        MethodInfo setter = prop.SetMethod;
        
        var instanceParam = Expression.Parameter(typeof(T), "instance");
        var valueParam = Expression.Parameter(typeof(TValue), "value");
        var propertyAccess = Expression.Property(instanceParam, prop);
        var assign = Expression.Assign(propertyAccess, valueParam);
        var lambda = Expression.Lambda<Action<T, TValue>>(assign, instanceParam, valueParam);
        return lambda.Compile();
    }
}

// 使用（编译时已知 T）
var getName = PropertyDelegateFactory.CreateGetter<Person, string>("Name");
var setName = PropertyDelegateFactory.CreateSetter<Person, string>("Name");

Person p = new Person();
setName(p, "李四");
Console.WriteLine(getName(p));  // "李四"
```

---

## 七、完整实战：通用对象映射器（Mapper）

```csharp
public static class ObjectMapper
{
    // 缓存属性信息
    private static readonly Dictionary<Type, PropertyInfo[]> _propertyCache = new();
    
    /// <summary>
    /// 将源对象的属性值复制到目标对象（同名同类型）
    /// </summary>
    public static TTarget Map<TSource, TTarget>(TSource source) 
        where TTarget : new()
    {
        if (source == null) throw new ArgumentNullException(nameof(source));
        
        TTarget target = new TTarget();
        Map(source, target);
        return target;
    }
    
    /// <summary>
    /// 将源对象的属性值复制到已存在的目标对象
    /// </summary>
    public static void Map<TSource, TTarget>(TSource source, TTarget target)
    {
        if (source == null) throw new ArgumentNullException(nameof(source));
        if (target == null) throw new ArgumentNullException(nameof(target));
        
        Type sourceType = typeof(TSource);
        Type targetType = typeof(TTarget);
        
        PropertyInfo[] sourceProps = GetCachedProperties(sourceType);
        PropertyInfo[] targetProps = GetCachedProperties(targetType);
        
        // 创建属性名到属性的映射
        var targetPropDict = targetProps.ToDictionary(p => p.Name);
        
        foreach (var sourceProp in sourceProps)
        {
            if (!targetPropDict.TryGetValue(sourceProp.Name, out var targetProp))
                continue;
            
            // 类型必须兼容
            if (!targetProp.PropertyType.IsAssignableFrom(sourceProp.PropertyType))
                continue;
            
            // 目标属性必须可写
            if (!targetProp.CanWrite)
                continue;
            
            // 源属性必须可读
            if (!sourceProp.CanRead)
                continue;
            
            object value = sourceProp.GetValue(source);
            targetProp.SetValue(target, value);
        }
    }
    
    /// <summary>
    /// 支持不同属性名的映射（使用 Attribute 标记）
    /// </summary>
    public static void MapWithAttribute<TSource, TTarget>(TSource source, TTarget target)
    {
        Type sourceType = typeof(TSource);
        Type targetType = typeof(TTarget);
        
        var sourceProps = GetCachedProperties(sourceType);
        var targetProps = GetCachedProperties(targetType);
        
        // 创建目标属性名到属性的映射
        var targetPropDict = targetProps.ToDictionary(p => p.Name);
        
        foreach (var sourceProp in sourceProps)
        {
            // 检查是否有 MapToAttribute
            var mapAttr = sourceProp.GetCustomAttribute<MapToAttribute>();
            string targetPropName = mapAttr?.TargetName ?? sourceProp.Name;
            
            if (!targetPropDict.TryGetValue(targetPropName, out var targetProp))
                continue;
            
            if (!targetProp.CanWrite || !sourceProp.CanRead)
                continue;
            
            if (!targetProp.PropertyType.IsAssignableFrom(sourceProp.PropertyType))
                continue;
            
            targetProp.SetValue(target, sourceProp.GetValue(source));
        }
    }
    
    private static PropertyInfo[] GetCachedProperties(Type type)
    {
        lock (_propertyCache)
        {
            if (!_propertyCache.TryGetValue(type, out var props))
            {
                props = type.GetProperties(BindingFlags.Public | BindingFlags.Instance);
                _propertyCache[type] = props;
            }
            return props;
        }
    }
}

// 自定义特性
[AttributeUsage(AttributeTargets.Property)]
public class MapToAttribute : Attribute
{
    public string TargetName { get; }
    public MapToAttribute(string targetName) => TargetName = targetName;
}

// 使用示例
public class UserDto
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Email { get; set; }
}

public class UserEntity
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    [MapTo("Email")]  // 注意：这里标记的是源属性，表示映射到 Email
    public string UserEmail { get; set; }
}

// 测试
var dto = new UserDto { Name = "张三", Age = 25, Email = "zhangsan@example.com" };
var entity = ObjectMapper.Map<UserDto, UserEntity>(dto);
Console.WriteLine($"Name: {entity.Name}, Age: {entity.Age}, Email: {entity.UserEmail}");
```

---

## 八、常见陷阱与最佳实践

### 陷阱：值类型的属性修改

```csharp
struct Point
{
    public int X { get; set; }
    public int Y { get; set; }
}

Point point = new Point();
Type pointType = typeof(Point);
PropertyInfo xProp = pointType.GetProperty("X");

// ❌ 错误：点被装箱，修改的是副本
object boxed = point;
xProp.SetValue(boxed, 10);
Console.WriteLine(point.X);  // 还是 0

// ✅ 正确：操作后重新赋值
boxed = point;
xProp.SetValue(boxed, 10);
point = (Point)boxed;
```

### 陷阱：Nullable 类型的处理

```csharp
public class NullableDemo
{
    public int? Count { get; set; }
    public DateTime? Date { get; set; }
}

Type t = typeof(NullableDemo);
object obj = Activator.CreateInstance(t);

PropertyInfo countProp = t.GetProperty("Count");

// 设置 null 值
countProp.SetValue(obj, null);  // ✅ 可以

// 设置非 null 值
countProp.SetValue(obj, 42);    // ✅ 自动装箱

int? value = (int?)countProp.GetValue(obj);
Console.WriteLine(value);  // 42

// 判断是否为 Nullable 类型
bool isNullable = Nullable.GetUnderlyingType(countProp.PropertyType) != null;
```

### 最佳实践：批量属性操作

```csharp
public static class PropertyBatchHelper
{
    public static Dictionary<string, object> GetPropertyValues(object obj)
    {
        var result = new Dictionary<string, object>();
        var props = obj.GetType().GetProperties(BindingFlags.Public | BindingFlags.Instance);
        
        foreach (var prop in props)
        {
            if (prop.CanRead)
            {
                result[prop.Name] = prop.GetValue(obj);
            }
        }
        return result;
    }
    
    public static void SetPropertyValues(object obj, Dictionary<string, object> values)
    {
        var props = obj.GetType().GetProperties(BindingFlags.Public | BindingFlags.Instance);
        var propDict = props.ToDictionary(p => p.Name);
        
        foreach (var kvp in values)
        {
            if (propDict.TryGetValue(kvp.Key, out var prop) && prop.CanWrite)
            {
                // 类型转换
                object convertedValue = Convert.ChangeType(kvp.Value, prop.PropertyType);
                prop.SetValue(obj, convertedValue);
            }
        }
    }
}
```

---

## 九、本篇速查表

| 操作 | 代码 |
|------|------|
| 获取属性 | `typeof(T).GetProperty("Name")` |
| 读取值 | `prop.GetValue(instance)` |
| 写入值 | `prop.SetValue(instance, value)` |
| 静态属性 | 实例参数传 `null` |
| 只读属性 | 检查 `prop.CanWrite` |
| 私有属性 | 需要 `BindingFlags.NonPublic` |
| 索引器 | `prop.GetValue(instance, new object[]{index})` |
| 获取 getter 方法 | `prop.GetMethod` |
| 获取 setter 方法 | `prop.SetMethod` |
| 判断自动实现 | 检查 `CompilerGeneratedAttribute` |
| 高性能委托 | 表达式树编译 `Func<object, object>` |

---

## 十、思考题

1. 为什么说“属性不是字段”？反射如何体现这一点？

2. 索引器在元数据中叫什么名字？如何获取多维索引器的参数信息？

3. 如何实现一个通用的对象克隆器（深拷贝），使用反射复制所有属性？

4. 属性的 `SetValue` 内部做了什么类型转换？如果传入的类型不匹配会发生什么？

---

**下一篇预告：** 《C# 反射系列笔记（七）：FieldInfo 与字段动态操作》

下一篇将深入讲解字段的反射操作，包括静态字段、常量字段、只读字段，以及与属性的对比，并探讨字段在序列化中的作用。