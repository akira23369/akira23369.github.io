---
title: 反射系列笔记（五）：MethodInfo与方法动态调用
date: 2026-04-06 11:22:36
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


# C# 反射系列笔记（五）：MethodInfo 与方法动态调用

> 目标：全面掌握通过反射调用方法的各项技术，包括静态/实例方法、重载解析、ref/out参数、泛型方法、异步方法，以及性能优化

---

## 一、方法调用的三个层次

```
层次1：编译时调用（正常代码）
    list.Add(10);  ← 编译器知道一切，IL直接生成 call 指令

层次2：简单反射调用
    methodInfo.Invoke(obj, new object[]{10});  ← 运行时查找并调用

层次3：高性能动态调用（缓存委托）
    Action<List<int>, int> fastAdd = (Action<List<int>, int>)delegate;
    fastAdd(list, 10);  ← 接近直接调用的性能
```

**本篇重点：层次2 和层次3**

---

## 二、MethodInfo 核心属性

### 方法元数据属性

```csharp
MethodInfo method = typeof(string).GetMethod("Substring", new[] { typeof(int), typeof(int) });

Console.WriteLine($"方法名: {method.Name}");
Console.WriteLine($"返回类型: {method.ReturnType.Name}");
Console.WriteLine($"是否为公共: {method.IsPublic}");
Console.WriteLine($"是否为静态: {method.IsStatic}");
Console.WriteLine($"是否为抽象: {method.IsAbstract}");
Console.WriteLine($"是否为虚方法: {method.IsVirtual}");
Console.WriteLine($"是否为泛型方法: {method.IsGenericMethod}");
Console.WriteLine($"是否为泛型方法定义: {method.IsGenericMethodDefinition}");
Console.WriteLine($"是否为构造函数: {method.IsConstructor}");
Console.WriteLine($"是否为属性访问器: {method.IsSpecialName}");  // get_/set_
```

### 参数信息

```csharp
MethodInfo method = typeof(Console).GetMethod("WriteLine", new[] { typeof(string) });

ParameterInfo[] parameters = method.GetParameters();
foreach (ParameterInfo p in parameters)
{
    Console.WriteLine($"  参数名: {p.Name}");
    Console.WriteLine($"  参数类型: {p.ParameterType.Name}");
    Console.WriteLine($"  是否为 out: {p.IsOut}");
    Console.WriteLine($"  是否为 ref: {p.ParameterType.IsByRef}");
    Console.WriteLine($"  是否有默认值: {p.HasDefaultValue}");
    if (p.HasDefaultValue)
    {
        Console.WriteLine($"  默认值: {p.DefaultValue}");
    }
}
```

---

## 三、MethodInfo.Invoke 基础用法

### 调用实例方法

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
    private int Multiply(int a, int b) => a * b;
}

Type calcType = typeof(Calculator);
object calc = Activator.CreateInstance(calcType);

// 调用公共实例方法
MethodInfo addMethod = calcType.GetMethod("Add");
object result = addMethod.Invoke(calc, new object[] { 3, 5 });
Console.WriteLine($"3 + 5 = {result}");  // 8

// 调用私有实例方法
MethodInfo multiplyMethod = calcType.GetMethod("Multiply", 
    BindingFlags.NonPublic | BindingFlags.Instance);
object multiplyResult = multiplyMethod.Invoke(calc, new object[] { 4, 5 });
Console.WriteLine($"4 * 5 = {multiplyResult}");  // 20
```

### 调用静态方法

```csharp
Type mathType = typeof(Math);

// 静态方法，第一个参数传 null
MethodInfo absMethod = mathType.GetMethod("Abs", new[] { typeof(int) });
object absValue = absMethod.Invoke(null, new object[] { -42 });
Console.WriteLine(absValue);  // 42

// 调用带多个参数的静态方法
MethodInfo maxMethod = mathType.GetMethod("Max", new[] { typeof(int), typeof(int) });
object maxValue = maxMethod.Invoke(null, new object[] { 10, 20 });
Console.WriteLine(maxValue);  // 20
```

### 调用带 ref/out 参数的方法

```csharp
public class Parser
{
    public bool TryParse(string input, out int result)
    {
        return int.TryParse(input, out result);
    }
    
    public void Swap(ref int a, ref int b)
    {
        int temp = a;
        a = b;
        b = temp;
    }
}

Type parserType = typeof(Parser);
object parser = Activator.CreateInstance(parserType);

// out 参数示例
MethodInfo tryParseMethod = parserType.GetMethod("TryParse");
object[] tryParseArgs = new object[] { "123", null };  // out 参数用 null 占位
bool success = (bool)tryParseMethod.Invoke(parser, tryParseArgs);
int parsedValue = (int)tryParseArgs[1];  // out 参数值从数组中取回
Console.WriteLine($"解析成功: {success}, 值: {parsedValue}");

// ref 参数示例
MethodInfo swapMethod = parserType.GetMethod("Swap");
object[] swapArgs = new object[] { 10, 20 };
swapMethod.Invoke(parser, swapArgs);
Console.WriteLine($"交换后: a={swapArgs[0]}, b={swapArgs[1]}");  // a=20, b=10
```

**关键点：ref/out 参数的修改会反映在传入的 object 数组中，需要从数组中取回新值。**

---

## 四、方法重载解析

### 精确指定参数类型

```csharp
public class OverloadDemo
{
    public void Process(int x) => Console.WriteLine($"int: {x}");
    public void Process(string s) => Console.WriteLine($"string: {s}");
    public void Process(double d) => Console.WriteLine($"double: {d}");
    public void Process<T>(T t) => Console.WriteLine($"generic: {t}");
}

Type demoType = typeof(OverloadDemo);
object demo = Activator.CreateInstance(demoType);

// ✅ 正确：通过参数类型精确指定
MethodInfo intMethod = demoType.GetMethod("Process", new[] { typeof(int) });
intMethod.Invoke(demo, new object[] { 10 });  // "int: 10"

MethodInfo stringMethod = demoType.GetMethod("Process", new[] { typeof(string) });
stringMethod.Invoke(demo, new object[] { "hello" });  // "string: hello"

// ❌ 错误：不指定参数类型，可能返回 null 或 AmbiguousMatchException
MethodInfo ambiguous = demoType.GetMethod("Process");  // 返回 null！
```

### 处理参数类型兼容性

```csharp
// 场景：参数类型不完全匹配，但可以隐式转换
object value = 42;  // 实际是 int，但想调用 Process(long)

Type demoType = typeof(OverloadDemo);
object demo = Activator.CreateInstance(demoType);

// 方法1：使用 GetMethods 手动筛选
MethodInfo bestMatch = demoType.GetMethods()
    .Where(m => m.Name == "Process")
    .FirstOrDefault(m => m.GetParameters()[0].ParameterType.IsAssignableFrom(value.GetType()));

// 方法2：使用 Type.InvokeMember（自动进行类型转换）
object result = demoType.InvokeMember("Process", 
    BindingFlags.InvokeMethod | BindingFlags.Instance | BindingFlags.Public,
    null, demo, new object[] { 42L });  // long -> 匹配 double 或泛型
```

### 完整重载解析器

```csharp
public static class MethodResolver
{
    public static MethodInfo FindBestMatch(Type type, string methodName, object[] args)
    {
        var methods = type.GetMethods()
            .Where(m => m.Name == methodName && m.GetParameters().Length == args.Length)
            .ToList();
        
        if (methods.Count == 0)
            throw new MissingMethodException($"找不到名为 {methodName} 且参数个数为 {args.Length} 的方法");
        
        // 获取参数类型
        Type[] argTypes = args.Select(a => a?.GetType() ?? typeof(object)).ToArray();
        
        // 按匹配度排序
        var scoredMethods = methods.Select(m => new
        {
            Method = m,
            Score = CalculateMatchScore(m.GetParameters(), argTypes)
        }).Where(x => x.Score >= 0)
          .OrderByDescending(x => x.Score)
          .ToList();
        
        if (scoredMethods.Count == 0)
            throw new MissingMethodException($"找不到匹配参数类型的方法");
        
        return scoredMethods.First().Method;
    }
    
    private static int CalculateMatchScore(ParameterInfo[] parameters, Type[] argTypes)
    {
        int score = 0;
        for (int i = 0; i < parameters.Length; i++)
        {
            Type paramType = parameters[i].ParameterType;
            Type argType = argTypes[i];
            
            if (paramType == argType)
                score += 100;  // 完全匹配
            else if (paramType.IsAssignableFrom(argType))
                score += 50;   // 派生类匹配基类
            else if (CanImplicitlyConvert(argType, paramType))
                score += 10;   // 隐式转换
            else
                return -1;     // 无法匹配
        }
        return score;
    }
    
    private static bool CanImplicitlyConvert(Type from, Type to)
    {
        // 简化版本：检查常见的隐式转换
        if (to == typeof(long) && (from == typeof(int) || from == typeof(short)))
            return true;
        if (to == typeof(double) && (from == typeof(int) || from == typeof(float)))
            return true;
        // ... 更多转换规则
        return false;
    }
}
```

---

## 五、泛型方法的反射调用

### 基础：调用泛型方法

```csharp
public class GenericMethodDemo
{
    public T Echo<T>(T input) => input;
    
    public void Print<T>(T input) => Console.WriteLine($"类型: {typeof(T).Name}, 值: {input}");
    
    public List<T> CreateList<T>(params T[] items) => new List<T>(items);
}

Type demoType = typeof(GenericMethodDemo);
object demo = Activator.CreateInstance(demoType);

// 步骤1：获取泛型方法定义
MethodInfo echoMethod = demoType.GetMethod("Echo");

// 步骤2：指定具体类型，构造封闭方法
MethodInfo closedEcho = echoMethod.MakeGenericMethod(typeof(string));

// 步骤3：调用
object result = closedEcho.Invoke(demo, new object[] { "hello" });
Console.WriteLine(result);  // "hello"

// 更复杂的例子：带参数的泛型方法
MethodInfo createListMethod = demoType.GetMethod("CreateList");
MethodInfo closedCreateList = createListMethod.MakeGenericMethod(typeof(int));
object list = closedCreateList.Invoke(demo, new object[] { new object[] { 1, 2, 3 } });
Console.WriteLine(list.GetType());  // List`1[Int32]
```

### 运行时决定泛型参数类型

```csharp
public static object CallGenericMethod(object target, string methodName, Type genericArg, params object[] args)
{
    Type targetType = target.GetType();
    
    // 获取方法（可能有多个重载，这里取第一个匹配名称的）
    MethodInfo method = targetType.GetMethods()
        .FirstOrDefault(m => m.Name == methodName && m.IsGenericMethod);
    
    if (method == null)
        throw new MissingMethodException($"找不到泛型方法 {methodName}");
    
    // 构造封闭方法
    MethodInfo closedMethod = method.MakeGenericMethod(genericArg);
    
    // 调用
    return closedMethod.Invoke(target, args);
}

// 使用
var demo = new GenericMethodDemo();
object result = CallGenericMethod(demo, "Echo", typeof(int), 42);
Console.WriteLine(result);  // 42
```

### 泛型方法中的类型推断（不可能）

```csharp
// C# 编译器可以推断：
// var result = demo.Echo(42);  // 推断出 T = int

// 但反射无法自动推断！必须显式指定泛型参数
// ❌ 错误：没有 MakeGenericMethod 就直接调用会失败
MethodInfo echoMethod = typeof(GenericMethodDemo).GetMethod("Echo");
// echoMethod.Invoke(demo, new object[] { 42 });  // InvalidOperationException

// ✅ 必须：
MethodInfo closed = echoMethod.MakeGenericMethod(typeof(int));
closed.Invoke(demo, new object[] { 42 });
```

---

## 六、异步方法的反射调用

### 调用 async Task 方法

```csharp
public class AsyncDemo
{
    public async Task<string> GetDataAsync()
    {
        await Task.Delay(100);
        return "hello";
    }
    
    public async Task<int> CalculateAsync(int x)
    {
        await Task.Delay(50);
        return x * 2;
    }
}

// 同步等待异步方法的结果
public static object CallAsyncMethod(object target, string methodName, params object[] args)
{
    Type targetType = target.GetType();
    MethodInfo method = targetType.GetMethod(methodName);
    
    // 调用方法，得到 Task 对象
    object taskObj = method.Invoke(target, args);
    Task task = (Task)taskObj;
    
    // 等待完成
    task.GetAwaiter().GetResult();
    
    // 如果有返回值，获取 Result
    if (task.GetType().IsGenericType)
    {
        PropertyInfo resultProp = task.GetType().GetProperty("Result");
        return resultProp.GetValue(task);
    }
    
    return null;
}

// 使用
var demo = new AsyncDemo();
object result = CallAsyncMethod(demo, "GetDataAsync");
Console.WriteLine(result);  // "hello"
```

### 使用 await 语义（通过 GetAwaiter）

```csharp
public static async Task<T> CallAsyncMethodAsync<T>(object target, string methodName, params object[] args)
{
    Type targetType = target.GetType();
    MethodInfo method = targetType.GetMethod(methodName);
    
    // 动态调用并等待
    dynamic task = method.Invoke(target, args);
    return await task;
}

// 使用（需要在 async 方法中）
// string result = await CallAsyncMethodAsync<string>(demo, "GetDataAsync");
```

---

## 七、高性能动态调用：委托缓存

### Delegate.CreateDelegate

```csharp
public class PerformanceDemo
{
    public int Add(int a, int b) => a + b;
    public static int Multiply(int a, int b) => a * b;
}

// 实例方法委托
Type demoType = typeof(PerformanceDemo);
object demo = Activator.CreateInstance(demoType);

MethodInfo addMethod = demoType.GetMethod("Add");

// 创建强类型委托
Func<int, int, int> addDelegate = (Func<int, int, int>)Delegate.CreateDelegate(
    typeof(Func<int, int, int>), demo, addMethod);

int sum = addDelegate(3, 5);  // 快速调用！类似直接调用

// 静态方法委托
MethodInfo multiplyMethod = demoType.GetMethod("Multiply");
Func<int, int, int> multiplyDelegate = (Func<int, int, int>)Delegate.CreateDelegate(
    typeof(Func<int, int, int>), null, multiplyMethod);

int product = multiplyDelegate(4, 5);  // 20
```

### 通用委托缓存器

```csharp
public static class MethodDelegateCache
{
    // 缓存桶：Key 是方法特征字符串，Value 是已经转换好的委托实例
    private static readonly Dictionary<string, Delegate> _cache = new();

    public static TDelegate GetDelegate<TDelegate>(object target, string methodName) 
        where TDelegate : Delegate
    {
        // 1. 生成唯一的缓存键
        // 格式如: "Namespace.ClassName.MethodName_Func`3"
        // 确保同一个类、同一个方法、同一个委托签名的调用能指向同一个缓存
        string key = $"{target?.GetType()?.FullName ?? "static"}.{methodName}_{typeof(TDelegate).Name}";
        
        // 2. 加锁确保线程安全，防止多个线程同时写入缓存导致报错
        lock (_cache)
        {
            if (!_cache.TryGetValue(key, out var del))
            {
                // 3. 获取方法信息 (MethodInfo)
                // 如果 target 不为空，直接从实例类型里找方法
                // 如果 target 为空（静态方法情况），则通过泛型委托的第一个参数类型尝试推导类名并查找方法.所有的委托（Delegate）底层都有一个叫 Invoke 的方法。
                MethodInfo method = target?.GetType().GetMethod(methodName) 
                    ?? typeof(TDelegate).GetMethod("Invoke").GetParameters()[0].ParameterType.GetMethod(methodName);
                
                // 4. 【核心黑科技】
                // 将 MethodInfo 转换成具体的 TDelegate 委托
                // 这样后续调用就不再走反射路径，而是直接走 CPU 指令调用，速度提升几十倍
                del = Delegate.CreateDelegate(typeof(TDelegate), target, method);
                
                // 5. 存入缓存
                _cache[key] = del;
            }
            
            // 6. 强转回用户需要的委托类型
            return (TDelegate)(object)del;
        }
    }
}

// 使用
var calc = new PerformanceDemo();
var fastAdd = MethodDelegateCache.GetDelegate<Func<int, int, int>>(calc, "Add");
int result = fastAdd(10, 20);


```

关于这段代码的困惑
```cs
MethodInfo method = typeof(TDelegate).GetMethod("Invoke").GetParameters()[0].ParameterType.GetMethod(methodName);
```
委托当绑定实例方法时，委托对象内部不仅记录了方法的入口地址，还隐式保存了该对象实例的引用（即 `this` 指针，对应委托的 `Target` 属性）。


### 表达式树方式（更灵活）

```csharp
public static class ExpressionDelegateFactory
{
    public static Func<object, object[], object> CreateCallDelegate(MethodInfo method)
    {
        // 参数：实例（可能为 null），参数数组
        var instanceParam = Expression.Parameter(typeof(object), "instance");
        var argsParam = Expression.Parameter(typeof(object[]), "args");
        
        // 转换实例
        var instanceExpr = method.IsStatic 
            ? null 
            : Expression.Convert(instanceParam, method.DeclaringType);
        
        // 转换参数
        var parameters = method.GetParameters();
        var argExprs = new Expression[parameters.Length];
        for (int i = 0; i < parameters.Length; i++)
        {
            argExprs[i] = Expression.Convert(
                Expression.ArrayIndex(argsParam, Expression.Constant(i)),
                parameters[i].ParameterType);
        }
        
        // 调用方法
        var callExpr = method.IsStatic
            ? Expression.Call(method, argExprs)
            : Expression.Call(instanceExpr, method, argExprs);
        
        // 转换返回值
        var convertExpr = Expression.Convert(callExpr, typeof(object));
        
        // 编译委托
        var lambda = Expression.Lambda<Func<object, object[], object>>(
            convertExpr, instanceParam, argsParam);
        
        return lambda.Compile();
    }
}

// 使用
MethodInfo method = typeof(string).GetMethod("Substring", new[] { typeof(int), typeof(int) });
var fastCall = ExpressionDelegateFactory.CreateCallDelegate(method);

string s = "hello world";
object result = fastCall(s, new object[] { 0, 5 });
Console.WriteLine(result);  // "hello"
```

---

## 八、常见陷阱与最佳实践

### 陷阱：值类型的方法调用

```csharp
struct Point
{
    public int X;
    public void SetX(int x) { X = x; }
}

Type pointType = typeof(Point);
object point = Activator.CreateInstance(pointType);  // 装箱

MethodInfo setMethod = pointType.GetMethod("SetX");
setMethod.Invoke(point, new object[] { 10 });  // ❌ 修改的是装箱副本！

// 解决方法：先拆箱，修改，再装箱（但通常不推荐）
Point p = (Point)point;
p.SetX(10);
point = p;
```

### 陷阱：扩展方法的反射调用

```csharp
public static class StringExtensions
{
    public static int WordCount(this string str) => str.Split(' ').Length;
}

// 扩展方法本质上是静态方法，需要以静态方式调用
Type extensionsType = typeof(StringExtensions);
MethodInfo wordCountMethod = extensionsType.GetMethod("WordCount");

// 第一个参数是 this 参数
object result = wordCountMethod.Invoke(null, new object[] { "hello world" });
Console.WriteLine(result);  // 2
```

### 最佳实践：缓存 MethodInfo

```csharp
public static class MethodCache
{
    private static readonly Dictionary<string, MethodInfo> _cache = new();
    
    public static MethodInfo GetMethod(Type type, string name, Type[] argTypes)
    {
        string key = $"{type.FullName}.{name}({string.Join(",", argTypes.Select(t => t.Name))})";
        
        lock (_cache)
        {
            if (!_cache.TryGetValue(key, out var method))
            {
                method = type.GetMethod(name, argTypes);
                _cache[key] = method;
            }
            return method;
        }
    }
}
```

---

## 九、完整实战：动态方法调用器

```csharp
public class DynamicMethodInvoker
{
    private readonly object _target;
    private readonly Dictionary<string, Delegate> _delegateCache = new();
    
    public DynamicMethodInvoker(object target)
    {
        _target = target;
    }
    
    public object Invoke(string methodName, params object[] args)
    {
        Type targetType = _target.GetType();
        Type[] argTypes = args.Select(a => a?.GetType() ?? typeof(object)).ToArray();
        
        MethodInfo method = targetType.GetMethod(methodName, argTypes);
        if (method == null)
        {
            throw new MissingMethodException($"找不到方法 {methodName}");
        }
        
        return method.Invoke(_target, args);
    }
    
    public T Invoke<T>(string methodName, params object[] args)
    {
        return (T)Invoke(methodName, args);
    }
    
    public Func<TResult> GetFastDelegate<TResult>(string methodName)
    {
        return GetOrCreateDelegate<Func<TResult>>(methodName);
    }
    
    public Func<T1, TResult> GetFastDelegate<T1, TResult>(string methodName)
    {
        return GetOrCreateDelegate<Func<T1, TResult>>(methodName);
    }
    
    public Func<T1, T2, TResult> GetFastDelegate<T1, T2, TResult>(string methodName)
    {
        return GetOrCreateDelegate<Func<T1, T2, TResult>>(methodName);
    }
    
    private TDelegate GetOrCreateDelegate<TDelegate>(string methodName) where TDelegate : Delegate
    {
        lock (_delegateCache)
        {
            if (!_delegateCache.TryGetValue(methodName, out var del))
            {
                MethodInfo method = _target.GetType().GetMethod(methodName);
                del = Delegate.CreateDelegate(typeof(TDelegate), _target, method);
                _delegateCache[methodName] = del;
            }
            return (TDelegate)(object)del;
        }
    }
}

// 使用示例
class Demo
{
    public int Add(int a, int b) => a + b;
    public string Greet(string name) => $"Hello, {name}";
    public void Print(string msg) => Console.WriteLine(msg);
}

var demo = new Demo();
var invoker = new DynamicMethodInvoker(demo);

// 灵活调用
int sum = invoker.Invoke<int>("Add", 3, 5);
string greeting = invoker.Invoke<string>("Greet", "World");
invoker.Invoke("Print", "Hello!");

// 高性能调用
var fastAdd = invoker.GetFastDelegate<int, int, int>("Add");
int fastSum = fastAdd(10, 20);
```

---

## 十、本篇速查表

| 场景 | 代码 |
|------|------|
| 调用实例方法 | `method.Invoke(instance, new object[]{...})` |
| 调用静态方法 | `method.Invoke(null, new object[]{...})` |
| out 参数 | 参数数组占位，调用后从数组中取回 |
| ref 参数 | 同上 |
| 泛型方法 | `method.MakeGenericMethod(typeof(T)).Invoke(...)` |
| async 方法 | 调用后 await Task 或读取 Result |
| 高性能委托 | `Delegate.CreateDelegate` |
| 表达式树 | `Expression.Lambda` 编译 |
| 方法不存在 | `MissingMethodException` |
| 方法内部异常 | `TargetInvocationException`，看 `InnerException` |

---

## 十一、思考题

1. 为什么 `MethodInfo.Invoke` 对于值类型方法调用会有装箱问题？如何规避？

2. 扩展方法通过反射调用时，第一个参数应该传什么？

3. 如何判断一个 `MethodInfo` 是属性/事件的访问器（getter/setter）？

4. 如果方法的参数是 `params int[]`，反射调用时应该怎么传参？

---

**下一篇预告：** 《C# 反射系列笔记（六）：PropertyInfo 与属性动态操作》

下一篇将深入讲解属性的反射读写，包括自动实现的属性、索引器、属性的 GetMethod/SetMethod，以及实现通用的对象映射器（Mapper）。