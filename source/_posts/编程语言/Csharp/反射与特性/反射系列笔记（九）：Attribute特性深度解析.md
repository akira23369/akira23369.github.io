---
title: 反射系列笔记（九）：Attribute特性深度解析
date: 2026-04-07 08:55:58
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
[属性和反射 - C# | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/csharp/advanced-topics/reflection-and-attributes/)

# C# 反射系列笔记（九）：Attribute特性深度解析

这是一份**C#特性的系统性完整教程**，涵盖从基础概念到高级应用的方方面面。建议按顺序学习，并配合代码实践。

---

## 一、特性基础概念

### 什么是特性？
特性（Attribute）是一种**声明性标签**，用于向程序集、类型、成员等**添加元数据**。这些元数据可在运行时被反射（Reflection）读取，用于控制程序行为。

    **本质**：特性是继承自 `System.Attribute` 的类实例，编译时作为元数据嵌入程序集。

### 基本语法
```csharp
[特性名(参数列表)]
public class MyClass { }

// 示例
[Obsolete("此方法已过时", true)]
public void OldMethod() { }
```

### 可应用的目标（AttributeTargets 枚举）
可应用的目标（AttributeTargets 枚举）

```csharp
// AttributeTargets 枚举（可组合）
public enum AttributeTargets
{
    Assembly    = 0x0001,   // 程序集
    Module      = 0x0002,   // 模块
    Class       = 0x0004,   // 类
    Struct      = 0x0008,   // 结构体
    Enum        = 0x0010,   // 枚举
    Constructor = 0x0020,   // 构造函数
    Method      = 0x0040,   // 方法
    Property    = 0x0080,   // 属性
    Field       = 0x0100,   // 字段
    Event       = 0x0200,   // 事件
    Interface   = 0x0400,   // 接口
    Parameter   = 0x0800,   // 参数
    Delegate    = 0x1000,   // 委托
    ReturnValue = 0x2000,   // 返回值
    GenericParameter = 0x4000,  // 泛型参数（C# 2.0+）
    All         = 0x7FFF    // 所有目标
}



[assembly: MyAttr(1)]         // 应用于程序集
[module: MyAttr(2)]           // 应用于模块

[type: MyAttr(3)]             // 应用于类型（可省略）
internal sealed class SomeType<[typevar: MyAttr(4)] T> { 

   [field: MyAttr(5)]         // 应用于字段（可省略）
   public Int32 SomeField = 0;

   [return: MyAttr(6)]        // 应用于返回值
   [method: MyAttr(7)]        // 应用于方法（可省略）
   public Int32 SomeMethod(
      [param: MyAttr(8)]      // 应用于参数
      Int32 SomeParam) { return SomeParam; }

   [property: MyAttr(9)]      // 应用于属性
   public String SomeProp {
      [method: MyAttr(10)]    // 应用于访问器方法
      get { return null; }
   }

   [event: MyAttr(11)]        // 应用于事件
   public event EventHandler SomeEvent;
}
```

---

## 二、自定义特性开发

### 定义最简单的特性
```csharp
// 1. 继承System.Attribute
public class MyFirstAttribute : Attribute
{
    // 2. 可包含字段、属性、构造函数
    public string Description { get; set; }
    
    public MyFirstAttribute(string desc)
    {
        Description = desc;
    }
}

// 使用
[MyFirst("这是一个测试类")]
public class TestClass { }
```

### 特性命名约定
- **必须**以 "Attribute" 结尾（如 `MyFirstAttribute`）
- **使用时可省略** "Attribute" 后缀（`[MyFirst]` 等价于 `[MyFirstAttribute]`）

### AttributeUsage特性
控制特性如何被使用：

```csharp
[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Method, // 允许的目标
    AllowMultiple = false,                          // 是否允许多次使用
    Inherited = true                               // 是否可被派生类继承
)]
public class MyCustomAttribute : Attribute
{
    public string Name { get; set; }
    public int Priority { get; set; }
    
    // 位置参数（构造函数）
    public MyCustomAttribute(string name)
    {
        Name = name;
    }
}

// 使用示例
[MyCustom("用户管理", Priority = 1)]  // Name是位置参数，Priority是命名参数
public class UserController
{
    [MyCustom("获取用户")]  // Error! 只允许在类和方法上使用
    public string Name { get; set; }
}
```

### 特性的参数类型
特性参数**仅限以下类型**：
- 基本类型：`bool`, `byte`, `char`, `short`, `int`, `long`, `float`, `double`
- `string`
- `System.Type`
- `enum`
- 上述类型的一维数组
- `object`（但值必须是以上类型之一）

```csharp
public class ComplexAttribute : Attribute
{
    public Type TargetType { get; set; }        // Type参数
    public string[] Roles { get; set; }         // 数组参数
    public int MaxLength { get; set; } = 100;  // 带默认值
    
    public ComplexAttribute(string name, int version)
    {
        // 构造函数参数
    }
}

[Complex("test", 1, 
    TargetType = typeof(string), 
    Roles = new[] { "Admin", "User" }, 
    MaxLength = 200)]
public class MyClass { }
```

---

## 三、反射读取特性（核心）

### 获取Type对象
```csharp
// 方式1：typeof
Type type1 = typeof(MyClass);

// 方式2：GetType()
MyClass obj = new MyClass();
Type type2 = obj.GetType();

// 方式3：Assembly.GetType()
Assembly assembly = Assembly.GetExecutingAssembly();
Type type3 = assembly.GetType("Namespace.MyClass");
```

### 核心读取方法
```csharp
// 定义测试特性
[AttributeUsage(AttributeTargets.All, AllowMultiple = true)]
public class TestAttribute : Attribute
{
    public string Name { get; set; }
    public TestAttribute(string name) => Name = name;
}

// 应用特性
[TestAttribute("类级别")]
public class TargetClass
{
    [TestAttribute("方法级别")]
    public void MyMethod() { }
}
```

#### 获取单个特性
```csharp
Type type = typeof(TargetClass);

// 获取类上的特性
TestAttribute attr = type.GetCustomAttribute<TestAttribute>();
Console.WriteLine(attr?.Name); // "类级别"

// 获取方法上的特性
MethodInfo method = type.GetMethod("MyMethod");
TestAttribute methodAttr = method.GetCustomAttribute<TestAttribute>();
Console.WriteLine(methodAttr?.Name); // "方法级别
```

#### 获取所有特性
```csharp
// 获取类上的所有TestAttribute
IEnumerable<TestAttribute> attrs = type.GetCustomAttributes<TestAttribute>();

// 获取所有类型的特性
IEnumerable<Attribute> allAttrs = type.GetCustomAttributes();

// 检查是否包含特性
bool hasAttr = type.IsDefined(typeof(TestAttribute), false);
```

#### 考虑继承链
```csharp
// Inherited参数控制是否检查基类
var attr = type.GetCustomAttribute<TestAttribute>(inherit: true);
var attrs = type.GetCustomAttributes<TestAttribute>(inherit: false);
```

### 不同层级的特性获取

#### Assembly级别
```csharp
// 在AssemblyInfo.cs或任意文件中
[assembly: AssemblyTitle("MyApp")]
[assembly: AssemblyVersion("1.0.0.0")]
[assembly: MyCustomAttribute("程序集级别")]

// 读取
Assembly assembly = Assembly.GetExecutingAssembly();
var attr = assembly.GetCustomAttribute<AssemblyTitleAttribute>();
Console.WriteLine(attr.Title); // "MyApp"
```

#### Module级别
```csharp
[module: MyCustomAttribute("模块级别")]

Module module = typeof(Program).Module;
var attr = module.GetCustomAttribute<MyCustomAttribute>();
```

#### 成员级别（方法/属性/字段/事件）
```csharp
public class Sample
{
    [TestAttribute("字段")]
    private int _field;

    [TestAttribute("属性")]
    public string Prop { get; set; }

    [TestAttribute("方法", Order = 1)] // AllowMultiple = true
    [return:TestAttribute("返回值")]
    public string Method([TestAttribute("参数")] string param) => param;

    [TestAttribute("事件")]
    public event EventHandler Event;
}
// 定义测试特性
[AttributeUsage(AttributeTargets.All, AllowMultiple = true)]
public class TestAttribute : Attribute
{
    public string Name { get; set; }
    public int Order;
    public TestAttribute(string name) => Name = name;
    public void Print() => Console.WriteLine($"Name:{Name}, Order:{Order}");
}
class Program
{
    static void Main()
    {
        Type type = typeof(Sample);

        // 字段
        FieldInfo field = type.GetField("_field", BindingFlags.NonPublic | BindingFlags.Instance);
        var fieldAttr = field.GetCustomAttribute<TestAttribute>();
        fieldAttr.Print();

        // 属性
        PropertyInfo prop = type.GetProperty("Prop");
        var propAttr = prop.GetCustomAttribute<TestAttribute>();
        propAttr.Print();

        // 方法
        MethodInfo method = type.GetMethod("Method");
        var methodAttr = method.GetCustomAttribute<TestAttribute>();
        methodAttr.Print();

        // 方法参数
        ParameterInfo param = method.GetParameters()[0];
        var paramAttr = param.GetCustomAttribute<TestAttribute>();
        paramAttr.Print();

        // 返回值
        var returnAttr = method.ReturnParameter.GetCustomAttribute<TestAttribute>();
        returnAttr.Print();
    }
}
```

---

## 四、高级特性主题

### 特性继承行为
```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = true)]
public class BaseAttribute : Attribute { }

[BaseAttribute]
public class BaseClass { }

public class DerivedClass : BaseClass { }

// 读取
var attr = typeof(DerivedClass).GetCustomAttribute<BaseAttribute>();
// Inherited = true 时，attr不为null
// Inherited = false 时，attr为null
```

### 重复使用特性
```csharp
class Program
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
    public class AuthorAttribute : Attribute
    {
        public string Name { get; }
        public DateTime Date { get; }

        public AuthorAttribute(string name, string date)
        {
            Name = name;
            Date = DateTime.Parse(date);
        }
    }

    [Author("张三", "2024-01-01")]
    [Author("李四", "2024-02-01")]
    public void CollaborativeMethod() { }

    static void Main()
    {
        // 读取所有
        var method = typeof(Program).GetMethod("CollaborativeMethod");
        var authors = method.GetCustomAttributes<AuthorAttribute>();
        foreach (var author in authors)
        {
            Console.WriteLine($"{author.Name} - {author.Date}");
        }
    }
}
```

### 条件编译特性
```csharp
[Conditional("DEBUG")]
public class DebugOnlyAttribute : Attribute { }

// 仅在DEBUG模式下编译
[DebugOnly]
public void DebugMethod() { }
```

### 特性与反射的性能
```csharp
// 性能注意事项：
// 1. 反射有开销，应避免在热路径频繁调用
// 2. 缓存特性结果
private static readonly Lazy<TestAttribute> _cachedAttr = new Lazy<TestAttribute>(() =>
    typeof(TargetClass).GetCustomAttribute<TestAttribute>()
);

// 3. 使用AttributeUsage的Inherited和AllowMultiple优化查找
```

---

## 五、.NET内置重要特性清单

### 编译器服务特性

这些特性用于指导编译器生成特定代码行为。

**`[Obsolete]` - 标记过时成员**

```csharp
// 基本用法：标记方法已过时
[Obsolete("此方法已过时，请使用 NewMethod 替代")]
public void OldMethod() { }

// 强制错误：使用时产生编译错误而非警告
[Obsolete("此方法已移除", error: true)]
public void RemovedMethod() { }

// 实际应用：API版本迁移
public class LegacyApi
{
    [Obsolete("v2.0 起废弃，使用 GetUserAsync 替代", false)]
    public User GetUser(int id) { return null; }
    
    public Task<User> GetUserAsync(int id) { return null; }
}
```

**`[Conditional]` - 条件编译**

```csharp
// 定义条件符号
#define DEBUG
#define TRACE

public class ConditionalDemo
{
    // 仅在定义了 DEBUG 符号时编译调用
    [Conditional("DEBUG")]
    public void DebugLog(string message)
    {
        Console.WriteLine($"[DEBUG] {message}");
    }
    
    // 多个条件符号（OR关系）
    [Conditional("DEBUG"), Conditional("TRACE")]
    public void TraceLog(string message)
    {
        Console.WriteLine($"[TRACE] {message}");
    }
}

// 使用场景
var demo = new ConditionalDemo();
demo.DebugLog("这条日志仅在Debug模式出现"); // 发布版自动移除调用
```

**`[CallerMemberName]` / `[CallerFilePath]` / `[CallerLineNumber]` - 调用者信息**

```csharp
public class Logger
{
    // 自动获取调用者信息，无需手动传入
    public void Log(
        string message,
        [CallerMemberName] string memberName = "",
        [CallerFilePath] string sourceFilePath = "",
        [CallerLineNumber] int sourceLineNumber = 0)
    {
        Console.WriteLine($"{sourceFilePath}:{sourceLineNumber} - {memberName}: {message}");
    }
}

// 使用示例
public class BusinessService
{
    private Logger _logger = new Logger();
    
    public void DoWork()
    {
        _logger.Log("开始执行业务逻辑"); 
        // 输出：C:\Projects\Demo.cs:15 - DoWork: 开始执行业务逻辑
    }
}
```

---

### 序列化特性

控制对象序列化/反序列化行为。

**`[Serializable]` / `[NonSerialized]`**

```csharp
[Serializable]
public class UserProfile
{
    public string Username { get; set; }
    public string Email { get; set; }
    
    // 敏感字段不参与序列化
    [NonSerialized]
    private string _passwordHash;
    
    // 运行时临时数据不序列化
    [NonSerialized]
    private Dictionary<string, object> _cache;
}
```

**`[DataContract]` / `[DataMember]` - WCF/DataContract序列化**

```csharp
[DataContract(Namespace = "http://schemas.mycompany.com/user")]
public class Employee
{
    [DataMember(Order = 1, IsRequired = true, Name = "id")]
    public int Id { get; set; }
    
    [DataMember(Order = 2, EmitDefaultValue = false)]
    public string Name { get; set; }
    
    // 非DataMember字段不参与序列化
    public decimal Salary { get; set; } // 被忽略
}
```

**`[JsonProperty]` / `[JsonIgnore]` - Newtonsoft.Json**

```csharp
public class Product
{
    [JsonProperty("product_id")] // 自定义JSON字段名
    public int Id { get; set; }
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public string Description { get; set; }
    
    [JsonIgnore] // 完全忽略
    public decimal CostPrice { get; set; }
    
    [JsonProperty]
    [JsonConverter(typeof(IsoDateTimeConverter))] // 自定义转换器
    public DateTime CreatedAt { get; set; }
}
```

---

### 反射与元数据特性

**`[AttributeUsage]` - 自定义特性的元特性**

```csharp
// 限制特性的使用范围
[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Method, // 仅可用于类和方法
    AllowMultiple = true,      // 允许在同一目标上多次使用
    Inherited = false)]        // 不继承给派生类
public class AuditAttribute : Attribute
{
    public string Action { get; }
    public AuditAttribute(string action) => Action = action;
}

// 使用
[Audit("Create")]
[Audit("Validate")] // 允许多次使用
public class OrderService { }
```

**`[DefaultMember]` - 指定默认索引器**

```csharp
[DefaultMember("Item")] // 默认成员，通常用于索引器
public class CustomCollection
{
    // 这是默认成员，可通过 instance[index] 访问
    public string this[int index] => $"Item at {index}";
    
    public string GetByName(string name) => $"Named {name}";
}

// 使用
var col = new CustomCollection();
Console.WriteLine(col[0]);        // 通过默认成员访问
Console.WriteLine(col.Item[0]);   // 显式调用
```

---

### 互操作特性

**`[DllImport]` - 平台调用（P/Invoke）**

```csharp
public static class NativeMethods
{
    // 导入Windows API
    [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    public static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
    
    // 调用约定、入口点自定义
    [DllImport("kernel32.dll", EntryPoint = "GetPrivateProfileString")]
    public static extern uint GetIniString(
        string lpAppName,
        string lpKeyName,
        string lpDefault,
        StringBuilder lpReturnedString,
        uint nSize,
        string lpFileName);
}

// 使用
NativeMethods.MessageBox(IntPtr.Zero, "Hello", "Title", 0);
```

**`[MarshalAs]` - 指定封送行为**

```csharp
public struct NativeStruct
{
    // 字符串封送为LPStr
    [MarshalAs(UnmanagedType.LPStr)]
    public string AnsiString;
    
    // 数组封送
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 64)]
    public byte[] Buffer;
    
    // 布尔值封送
    [MarshalAs(UnmanagedType.Bool)]
    public bool IsActive;
}
```

**`[StructLayout]` - 控制结构体布局**

```csharp
// 顺序布局（默认），字段按声明顺序排列
[StructLayout(LayoutKind.Sequential, Pack = 4)]
public struct SystemTime
{
    public ushort Year;
    public ushort Month;
    public ushort Day;
}

// 显式布局，精确控制偏移量（联合体场景）
[StructLayout(LayoutKind.Explicit)]
public union UnionExample
{
    [FieldOffset(0)] public int Integer;
    [FieldOffset(0)] public float Float; // 与Integer共享内存
}
```

---

### 组件与资源管理特性

**`[DefaultValue]` - 指定默认值**

```csharp
public class Configuration
{
    [DefaultValue(8080)]
    public int Port { get; set; }
    
    [DefaultValue("localhost")]
    public string Host { get; set; }
    
    // 配合ResetValue方法使用
    public void Reset()
    {
        foreach (PropertyDescriptor prop in TypeDescriptor.GetProperties(this))
        {
            var attr = prop.Attributes[typeof(DefaultValueAttribute)] as DefaultValueAttribute;
            if (attr != null)
                prop.SetValue(this, attr.Value);
        }
    }
}
```

**`[Description]` / `[DisplayName]` / `[Category]` - 设计时支持**

```csharp
public class SettingsViewModel
{
    [Category("连接")]
    [DisplayName("服务器地址")]
    [Description("数据库服务器的主机名或IP地址")]
    [DefaultValue("localhost")]
    public string Server { get; set; }
    
    [Category("连接")]
    [DisplayName("端口号")]
    [Description("数据库服务监听端口")]
    [DefaultValue(1433)]
    public int Port { get; set; }
    
    [Browsable(false)] // 属性窗口中隐藏
    public string InternalToken { get; set; }
}
```

**`[TypeConverter]` - 类型转换**

```csharp
// 自定义类型转换器
[TypeConverter(typeof(PointConverter))]
public class Point
{
    public int X { get; set; }
    public int Y { get; set; }
}

public class PointConverter : TypeConverter
{
    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
        => sourceType == typeof(string) || base.CanConvertFrom(context, sourceType);
    
    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
    {
        if (value is string str)
        {
            var parts = str.Split(',');
            return new Point { X = int.Parse(parts[0]), Y = int.Parse(parts[1]) };
        }
        return base.ConvertFrom(context, culture, value);
    }
}
```

---

### 线程与异步特性

**`[STAThread]` / `[MTAThread]` - COM线程模型**

```csharp
class Program
{
    // Windows Forms/WPF 必须单线程单元
    [STAThread]
    static void Main()
    {
        Application.Run(new MainForm());
    }
}

// 控制台/后台服务可能使用MTA
class ServiceProgram
{
    [MTAThread]
    static void Main()
    {
        // MTA更适合多线程服务器场景
    }
}
```

**`[AsyncStateMachine]` - 编译器生成（了解即可）**

```csharp
// 此特性由编译器自动添加，标记异步状态机
// 开发者通常不直接使用，但了解其存在有助于理解async/await机制
public async Task Example()
{
    // 编译后：方法被标记[AsyncStateMachine(typeof(<Example>d__1))]
    await Task.Delay(1000);
}
```

---

### 诊断与调试特性

**`[DebuggerDisplay]` - 调试器显示格式**

```csharp
[DebuggerDisplay("Customer: {Name} (ID: {Id}) - Orders: {Orders.Count}")]
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Order> Orders { get; set; }
    
    // 复杂表达式
    [DebuggerDisplay("{GetDebuggerDisplay(),nq}")]
    public class Order
    {
        public int Id { get; set; }
        private string GetDebuggerDisplay() => $"Order #{Id}";
    }
}
```

**`[DebuggerStepThrough]` / `[DebuggerHidden]`**

```csharp
public class HelperMethods
{
    // 调试时跳过此方法，不进入内部
    [DebuggerStepThrough]
    public static void LogDebug(string msg) => Console.WriteLine(msg);
    
    // 完全隐藏在调用堆栈中
    [DebuggerHidden]
    private static void InternalUtility() { }
}

// 属性同样适用
public class Wrapper
{
    [DebuggerStepThrough]
    public string Value 
    { 
        get => _value;
        set => _value = value ?? throw new ArgumentNullException();
    }
    private string _value;
}
```

**`[StackTraceHidden]` - .NET 6+ 堆栈隐藏**

```csharp
public static class Guard
{
    // 抛出异常时，此方法不出现在堆栈跟踪中
    [StackTraceHidden]
    public static void AgainstNull(object argument, [CallerArgumentExpression("argument")] string paramName = "")
    {
        if (argument is null)
            throw new ArgumentNullException(paramName);
    }
}

// 调用 Guard.AgainstNull(obj) 时，如果异常，堆栈中只显示调用处，不显示Guard方法内部
```

---

### 代码分析特性

**`[GeneratedCode]` - 标记工具生成代码**

```csharp
// 代码生成工具自动添加，区分手写代码和生成代码
[System.CodeDom.Compiler.GeneratedCode("EntityFrameworkCore", "7.0.0")]
public partial class ApplicationDbContext : DbContext
{
    // 生成的DbSet属性
    public virtual DbSet<User> Users { get; set; }
}

// 配合代码分析规则：排除生成代码的某些检查
```

**`[ExcludeFromCodeCoverage]` - 覆盖率排除**

```csharp
public class Infrastructure
{
    // 日志辅助类通常不需要单元测试覆盖
    [ExcludeFromCodeCoverage]
    public static void LogToFile(string message)
    {
        File.AppendAllText("log.txt", message);
    }
}

// 属性getter/setter排除
public class Dto
{
    [ExcludeFromCodeCoverage]
    public string AutoProperty { get; set; } // 简单属性无需测试
}
```

**`[SuppressMessage]` - 抑制警告**

```csharp
[SuppressMessage("Design", "CA1062:Validate arguments of public methods", 
    Justification = "由调用方保证非空，内部高频调用避免重复检查")]
public void ProcessData(DataItem item)
{
    // 假设item不为null
    item.Execute();
}

// 范围抑制
[module: SuppressMessage("Style", "IDE0005", Scope = "namespace", Target = "MyApp.Generated")]
```

---

### ASP.NET Core 核心特性（Web开发）

**`[ApiController]` / `[Route]` - 路由与API行为**

```csharp
[ApiController] // 启用API约定：自动模型验证、绑定源推断等
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 属性路由
    [HttpGet("{id:int}")]
    [ProducesResponseType(typeof(ProductDto), 200)]
    [ProducesResponseType(404)]
    public IActionResult Get(int id) { }
    
    [HttpPost]
    [ValidateAntiForgeryToken] // 防伪验证
    [Consumes("application/json")]
    public IActionResult Create([FromBody] CreateProductDto dto) { }
}
```

**`[FromServices]` / `[FromBody]` / `[FromQuery]` - 参数绑定**

```csharp
public class OrderController : ControllerBase
{
    [HttpPost("complex")]
    public IActionResult Create(
        [FromHeader(Name = "X-Request-ID")] string requestId,  // 从Header
        [FromQuery] int? pageSize,                            // 从QueryString
        [FromBody] OrderRequest request,                      // 从JSON Body
        [FromServices] IOrderValidator validator)             // 从DI容器
    {
        // 参数绑定自动完成
    }
}
```

**`[Authorize]` / `[AllowAnonymous]` - 权限控制**

```csharp
[Authorize(Roles = "Admin,Manager")] // 角色授权
[Authorize(Policy = "CanEditProducts")] // 策略授权
public class AdminController : Controller
{
    [AllowAnonymous] // 覆盖类级授权
    public IActionResult Login() { }
    
    [Authorize(AuthenticationSchemes = "Bearer")] // 指定认证方案
    public IActionResult ApiEndpoint() { }
}
```

---

### Entity Framework Core 特性（ORM）

**`[Key]` / `[DatabaseGenerated]` - 主键配置**

```csharp
public class Entity
{
    [Key] // 主键
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)] // 自增
    public int Id { get; set; }
    
    [DatabaseGenerated(DatabaseGeneratedOption.Computed)] // 计算列
    public DateTime LastModified { get; set; }
}
```

**`[Column]` / `[Table]` / `[Index]` - 映射配置**

```csharp
[Table("tbl_Users", Schema = "security")]
[Index(nameof(Email), IsUnique = true)]
[Index(nameof(CreatedAt), nameof(Status), Name = "IX_Created_Status")]
public class User
{
    [Column("user_id", TypeName = "varchar(36)")]
    public string Id { get; set; }
    
    [Required]
    [MaxLength(256)]
    [Column(TypeName = "nvarchar(256)")]
    public string Email { get; set; }
}
```

**`[NotMapped]` / `[ForeignKey]` - 关系配置**

```csharp
public class Order
{
    public int Id { get; set; }
    
    // 外键关系
    public int CustomerId { get; set; }
    [ForeignKey("CustomerId")]
    public Customer Customer { get; set; }
    
    // 内存计算属性，不映射到数据库
    [NotMapped]
    public decimal TotalAmount => Items.Sum(i => i.Price * i.Quantity);
    
    // 复杂类型映射
    [Owned]
    public Address ShippingAddress { get; set; }
}
```

---

### System.Text.Json 特性（.NET Core 3+）

**`[JsonPropertyName]` / `[JsonIgnore]` - JSON映射**

```csharp
public class WeatherForecast
{
    [JsonPropertyName("date")] // 小写命名
    public DateTime Date { get; set; }
    
    [JsonPropertyName("temp_c")] // 蛇形命名
    public int TemperatureC { get; set; }
    
    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)] // 条件忽略
    public string Summary { get; set; }
    
    [JsonInclude] // 包含非公共属性
    [JsonPropertyName("internal_id")]
    internal int InternalId { get; set; }
}
```

**`[JsonConverter]` - 自定义序列化**

```csharp
public class Product
{
    public string Name { get; set; }
    
    // 使用自定义转换器处理复杂类型
    [JsonConverter(typeof(DecimalPrecisionConverter))]
    public decimal Price { get; set; }
    
    // 枚举转字符串
    [JsonConverter(typeof(JsonStringEnumConverter))]
    public Status Status { get; set; }
}

public class DecimalPrecisionConverter : JsonConverter<decimal>
{
    public override decimal Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        => reader.GetDecimal();
    
    public override void Write(Utf8JsonWriter writer, decimal value, JsonSerializerOptions options)
        => writer.WriteNumberValue(Math.Round(value, 2)); // 保留2位小数
}
```

---

### 其他重要特性

**`[InternalsVisibleTo]` - 友元程序集（Assembly级别）**

```csharp
// AssemblyInfo.cs 或项目文件中
[assembly: InternalsVisibleTo("MyProject.Tests")]
[assembly: InternalsVisibleTo("DynamicProxyGenAssembly2")] // Moq等框架需要

// 使用
internal class InternalService { } // 测试项目可访问
```

**`[Extension]` - 扩展方法标记（编译器使用）**

```csharp
// 实际开发中不显式使用，但了解其存在
public static class Extensions
{
    // 编译器将其标记为[Extension]
    public static bool IsNullOrEmpty(this string str) => string.IsNullOrEmpty(str);
}
```

**`[TupleElementNames]` - 具名元组（编译器生成）**

```csharp
// 编译器自动生成此特性存储元组元素名
public (int Id, string Name) GetUser() => (1, "张三");
// 实际编译后：返回值标记[TupleElementNames(new[] {"Id", "Name"})]
```

---

### 特性分类速查表

| 类别 | 核心特性 | 典型应用场景 |
|------|---------|------------|
| **编译器服务** | `Obsolete`, `Conditional`, `CallerMemberName` | API版本管理、调试日志、条件编译 |
| **序列化** | `Serializable`, `DataContract`, `JsonProperty` | 数据持久化、API通信、配置文件 |
| **反射元数据** | `AttributeUsage`, `DefaultMember` | 自定义特性、索引器定义 |
| **互操作** | `DllImport`, `MarshalAs`, `StructLayout` | Win32 API调用、硬件通信 |
| **组件设计** | `DefaultValue`, `TypeConverter`, `Description` | 属性网格、设计时支持 |
| **诊断调试** | `DebuggerDisplay`, `DebuggerStepThrough` | 调试体验优化 |
| **Web开发** | `ApiController`, `Route`, `Authorize` | REST API、MVC应用 |
| **ORM** | `Key`, `Column`, `NotMapped`, `ForeignKey` | 数据库模型映射 |
| **代码质量** | `ExcludeFromCodeCoverage`, `SuppressMessage` | 静态分析、测试覆盖 |

---

## 六、实战应用场景

### 构建简单的ORM映射
```csharp
// 1. 定义特性
[AttributeUsage(AttributeTargets.Class)]
public class TableAttribute : Attribute
{
    public string Name { get; }
    public TableAttribute(string name) => Name = name;
}

[AttributeUsage(AttributeTargets.Property)]
public class ColumnAttribute : Attribute
{
    public string Name { get; }
    public bool IsPrimaryKey { get; set; }
    public bool IsNullable { get; set; }
    public int MaxLength { get; set; }
    
    public ColumnAttribute(string name) => Name = name;
}

// 2. 应用特性
[Table("Users")]
public class User
{
    [Column("user_id", IsPrimaryKey = true)]
    public int Id { get; set; }
    
    [Column("user_name", MaxLength = 50)]
    public string Name { get; set; }
    
    [Column("email", IsNullable = false)]
    public string Email { get; set; }
}

// 3. 反射读取并生成SQL
public static class OrmHelper
{
    public static string BuildInsert<T>(T entity)
    {
        Type type = typeof(T);
        var tableAttr = type.GetCustomAttribute<TableAttribute>();
        string tableName = tableAttr?.Name ?? type.Name;
        
        var columns = new List<string>();
        var values = new List<string>();
        
        foreach (var prop in type.GetProperties())
        {
            var colAttr = prop.GetCustomAttribute<ColumnAttribute>();
            string colName = colAttr?.Name ?? prop.Name;
            
            columns.Add(colName);
            values.Add($"'{prop.GetValue(entity)}'");
        }
        
        return $"INSERT INTO {tableName} ({string.Join(", ", columns)}) VALUES ({string.Join(", ", values)})";
    }
}

// 使用
var user = new User { Id = 1, Name = "张三", Email = "zhang@example.com" };
string sql = OrmHelper.BuildInsert(user);
Console.WriteLine(sql);
// 输出: INSERT INTO Users (user_id, user_name, email) VALUES ('1', '张三', 'zhang@example.com')
```

### 实现AOP日志
**AOP（面向切面编程）日志** 的核心思想：**把日志记录从业务代码中"抽离"出来，像一层透明的"包装纸"包裹在方法外面，业务代码完全感知不到日志的存在。**

```csharp
// 步骤1：定义标记特性
[AttributeUsage(AttributeTargets.Method)]
public class LogAttribute : Attribute { }

// 步骤2：业务接口和实现
public interface ICalculator
{
    [Log] int Add(int a, int b);  // 需要记录日志
    int Multiply(int a, int b);   // 不需要日志
}

public class Calculator : ICalculator
{
    public int Add(int a, int b) => a + b;
    public int Multiply(int a, int b) => a * b;
}

// 步骤3：动态代理拦截器（AOP核心）
public class LoggingProxy<T> : DispatchProxy where T : class
{
    private T _target;

    public static T Create(T target)
    {
        object proxy = Create<T, LoggingProxy<T>>();
        ((LoggingProxy<T>)proxy)._target = target;
        return (T)proxy;
    }

    protected override object Invoke(MethodInfo targetMethod, object[] args)
    {
        // 检查是否有[Log]标记
        if (targetMethod.GetCustomAttribute<LogAttribute>() != null)
        {
            Console.WriteLine($"[AOP日志] 调用 {targetMethod.Name}({string.Join(",", args)})");
            var result = targetMethod.Invoke(_target, args);
            Console.WriteLine($"[AOP日志] 返回 {result}");
            return result;
        }

        return targetMethod.Invoke(_target, args); // 无标记，直接执行
    }
}

class Program
{
    static void Main()
    {
        // 步骤4：使用
        var realCalc = new Calculator();
        var proxyCalc = LoggingProxy<ICalculator>.Create(realCalc);

        proxyCalc.Add(1, 2);        // 输出日志
        proxyCalc.Multiply(3, 4);   // 不输出日志
    }
}
```

### 验证框架
```csharp
// 1. 验证特性
public class ValidationAttribute : Attribute
{
    public virtual bool IsValid(object value) => true;
}

public class RequiredAttribute : ValidationAttribute
{
    public override bool IsValid(object value) => value != null && !string.IsNullOrEmpty(value.ToString());
}

public class RangeAttribute : ValidationAttribute
{
    public double Min { get; }
    public double Max { get; }
    public RangeAttribute(double min, double max) => (Min, Max) = (min, max);
    
    public override bool IsValid(object value)
    {
        if (value is IConvertible convertible)
        {
            double num = convertible.ToDouble(CultureInfo.InvariantCulture);
            return num >= Min && num <= Max;
        }
        return false;
    }
}

// 2. 验证器
public static class Validator
{
    public static List<string> Validate<T>(T obj)
    {
        var errors = new List<string>();
        Type type = typeof(T);
        
        foreach (var prop in type.GetProperties())
        {
            var attrs = prop.GetCustomAttributes<ValidationAttribute>();
            foreach (var attr in attrs)
            {
                var value = prop.GetValue(obj);
                if (!attr.IsValid(value))
                {
                    errors.Add($"{prop.Name} 验证失败");
                }
            }
        }
        
        return errors;
    }
}

// 3. 使用
public class Product
{
    [Required]
    public string Name { get; set; }
    
    [Range(0, 1000)]
    public double Price { get; set; }
}

var product = new Product { Name = null, Price = 2000 };
var errors = Validator.Validate(product);
// 输出: Name 验证失败, Price 验证失败
foreach (var it in errors)
{
    Console.WriteLine(it);
}
```

### 依赖注入标记
**不自己创建依赖的对象，而是由"外部容器"创建后"注入"进来。对象只负责使用，不负责创建。**
```csharp
using System.Reflection;

// ========== 1. 定义抽象（接口）==========
public interface IRepository
{
    void Save();
}

public interface ILogger
{
    void Log(string msg);
}

// ========== 2. 具体实现 ==========
public class SqlRepository : IRepository
{
    public void Save() => Console.WriteLine("保存到SQL数据库");
}

public class FileLogger : ILogger
{
    public void Log(string msg) => Console.WriteLine($"[文件日志] {msg}");
}

// ========== 3. 需要依赖的类 ==========
public class UserService
{
    private readonly IRepository _repo;
    private readonly ILogger _logger;
    
    // 标记：这个构造函数需要注入
    [Inject]  // ← 自定义特性，标记注入点
    public UserService(IRepository repo, ILogger logger)
    {
        _repo = repo;
        _logger = logger;
    }
    
    public void CreateUser()
    {
        _logger.Log("开始创建用户");
        _repo.Save();
        _logger.Log("用户创建完成");
    }
}

// ========== 4. 自定义 [Inject] 特性 ==========
[AttributeUsage(AttributeTargets.Constructor | AttributeTargets.Property)]
public class InjectAttribute : Attribute { }

// ========== 5. 简易DI容器（用反射实现）==========
public class SimpleContainer
{
    // 注册：接口 → 实现类
    private Dictionary<Type, Type> _registrations = new();
    
    public void Register<TInterface, TImplementation>() where TImplementation : TInterface
    {
        _registrations[typeof(TInterface)] = typeof(TImplementation);
    }
    
    // 解析：创建对象并自动注入依赖
    public T Resolve<T>()
    {
        return (T)Resolve(typeof(T));
    }
    
    private object Resolve(Type type)
    {
        // 1. 找标记了[Inject]的构造函数
        var ctor = type.GetConstructors()
            .FirstOrDefault(c => c.GetCustomAttribute<InjectAttribute>() != null)
            ?? type.GetConstructors().First(); // 没标记就取第一个
        
        // 2. 获取构造函数的参数（就是依赖）
        var parameters = ctor.GetParameters();
        var args = new object[parameters.Length];
        
        // 3. 递归解析每个参数（依赖）
        for (int i = 0; i < parameters.Length; i++)
        {
            var paramType = parameters[i].ParameterType;
            
            // 如果是接口，找注册的具体实现
            if (_registrations.ContainsKey(paramType))
            {
                var implType = _registrations[paramType];
                args[i] = Resolve(implType);  // 递归创建依赖
            }
            else
            {
                args[i] = Resolve(paramType); // 具体类型直接创建
            }
        }
        
        // 4. 创建实例，注入解析好的依赖
        return ctor.Invoke(args);
    }
}

// ========== 6. 使用 ==========
class Program
{
    static void Main()
    {
        // 创建容器
        var container = new SimpleContainer();
        
        // 注册依赖关系（配置）
        container.Register<IRepository, SqlRepository>();
        container.Register<ILogger, FileLogger>();
        
        // 解析：容器自动创建UserService，并注入Repository和Logger
        var userService = container.Resolve<UserService>();
        
        // 使用：完全不知道具体实现是什么
        userService.CreateUser();
    }
}
```

---

## 七、最佳实践与性能优化

### 性能优化技巧
```csharp
// ❌ 坏：频繁反射读取
public void BadMethod()
{
    for (int i = 0; i < 1000; i++)
    {
        var attr = typeof(MyClass).GetCustomAttribute<MyAttribute>(); // 重复操作
    }
}

// ✅ 好：缓存特性
private static readonly MyAttribute _cachedAttribute = 
    typeof(MyClass).GetCustomAttribute<MyAttribute>();

public void GoodMethod()
{
    for (int i = 0; i < 1000; i++)
    {
        UseAttribute(_cachedAttribute);
    }
}

// ✅ 更好：使用静态泛型缓存
public static class AttributeCache<T, TAttribute> where TAttribute : Attribute
{
    public static readonly TAttribute Value = typeof(T).GetCustomAttribute<TAttribute>();
}

var attr = AttributeCache<MyClass, MyAttribute>.Value;
```

### 设计原则
1. **单一职责**：每个特性只处理一个关注点
2. **不可变**：特性属性应在构造后保持不变
3. **轻量级**：避免在特性中存储大量数据
4. **命名清晰**：特性名应明确表达意图（如`RequiredAttribute`, `MaxLengthAttribute`）

### 常见陷阱
```csharp
// ❌ 错误：特性中不能使用泛型（编译器限制）
// public class GenericAttribute<T> : Attribute { }

// ❌ 错误：特性参数不能使用null（除非是可空类型）
// [MyAttr(null)] // 编译错误

// ✅ 正确：使用特殊值或额外属性
[MyAttr("")]
public class MyClass { }

// ❌ 错误：运行时修改特性（特性是只读的）
var attr = type.GetCustomAttribute<MyAttribute>();
// attr.Name = "New"; // 虽然编译通过，但实际不会修改元数据

// ❌ 错误：忘记检查null
var attr = type.GetCustomAttribute<MyAttribute>();
// Console.WriteLine(attr.Name); // 可能NullReferenceException
```

### 调试技巧
```csharp
// 在Visual Studio中查看特性
// 1. 调试时查看类型/成员的"Attributes"属性
// 2. 使用Immediate Window：
typeof(MyClass).GetCustomAttributes().ToList()

// 3. 自定义特性添加DebuggerDisplay
[DebuggerDisplay("Name={Name}, Priority={Priority}")]
public class MyCustomAttribute : Attribute { }
```
