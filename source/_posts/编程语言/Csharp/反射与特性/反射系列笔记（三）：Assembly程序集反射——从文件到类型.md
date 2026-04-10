---
title: 反射系列笔记（三）：Assembly程序集反射——从文件到类型
date: 2026-04-06 11:21:32
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



# C# 反射系列笔记（三）：Assembly 程序集反射——从文件到类型

> 目标：掌握 `Assembly` 类的所有核心功能，学会加载外部程序集、扫描类型、读取资源，理解程序集在反射中的核心地位

---

## 一、程序集：反射的“图书馆”

### 为什么要理解 Assembly？

```
你的程序（.exe）
    │
    ├── 引用了 System.dll ──────────→ Assembly 对象
    ├── 引用了 Newtonsoft.Json.dll → Assembly 对象
    ├── 自己的代码 ─────────────────→ Assembly 对象
    └── 运行时动态加载的插件.dll ───→ Assembly 对象
```

**直觉理解：**
- **Type** = 一本书（某个类型的完整信息）
- **Assembly** = 一个书架/图书馆（多个类型的集合）
- 要找到一本书，先去图书馆；要获取一个类型，先去它的程序集

### Assembly 到底是什么？

> **程序集（Assembly）** 是 .NET 应用程序的部署单元，可以是 `.exe` 或 `.dll` 文件。它包含：
> - **IL 代码**：所有方法的实现逻辑
> - **元数据**：所有类型的描述信息（Type 的数据来源）
> - **资源**：图片、字符串、配置文件等
> - **清单（Manifest）**：程序集自身的元数据（版本、依赖项等）

**一个关键点：** 一个程序集可以包含多个命名空间、多个类型。

---

## 二、获取 Assembly 对象的 6 种方式

### 方法总览

| 方法                                | 说明                | 使用场景             |
| --------------------------------- | ----------------- | ---------------- |
| `Assembly.GetExecutingAssembly()` | 获取当前正在执行的代码所在的程序集 | 最常用，获取自己         |
| `Assembly.GetCallingAssembly()`   | 获取调用当前方法的方法所在的程序集 | 调试、日志、AOP        |
| `Assembly.GetEntryAssembly()`     | 获取入口程序集（.exe）     | 获取应用程序主程序集       |
| `typeof(T).Assembly`              | 从已知类型获取           | 获取某个类型所在的程序集     |
| `Assembly.Load(name)`             | 按程序集名称加载          | 加载 GAC 或已知名称的程序集 |
| `Assembly.LoadFrom(path)`         | 按文件路径加载           | 加载外部 DLL（插件）     |
| `Assembly.LoadFile(path)`         | 按文件路径加载（不加载依赖）    | 特殊场景，一般不常用       |
| `Assembly.Load(byte[])`           | 从字节数组加载           | 网络传输、内存加载        |

### 详细示例

```csharp
using System.Reflection;

// 1. 获取当前程序集（Main 方法所在的程序集）
Assembly current = Assembly.GetExecutingAssembly();
Console.WriteLine($"当前程序集: {current.GetName().Name}");

// 2. 获取调用者程序集
void MethodA()
{
    Assembly caller = Assembly.GetCallingAssembly();  // 谁调用了 MethodA？
    Console.WriteLine($"调用者: {caller.GetName().Name}");
}
MethodA();

// 3. 获取入口程序集（应用程序的 .exe）
Assembly entry = Assembly.GetEntryAssembly();
Console.WriteLine($"入口程序集: {entry?.GetName().Name ?? "无"}");

// 4. 从类型获取
Assembly fromType = typeof(List<int>).Assembly;
Console.WriteLine($"List<T> 所在程序集: {fromType.GetName().Name}");  // System.Private.CoreLib

// 5. 按名称加载（会从 GAC 和当前目录查找）
Assembly loaded = Assembly.Load("System.Text.Json");
Console.WriteLine($"加载 System.Text.Json: {loaded.GetName().Version}");

// 6. 从文件路径加载（最常用，用于插件）
Assembly fromFile = Assembly.LoadFrom(@"C:\MyPlugins\MyPlugin.dll");
```

### Load vs LoadFrom vs LoadFile 的区别（重要！）

| 方法               | 行为                                  | 何时用            |
| ---------------- | ----------------------------------- | -------------- |
| `Load(name)`     | 只加载名称，CLR 按规则查找（GAC → 当前目录 → 探测路径）  | 加载框架程序集        |
| `LoadFrom(path)` | 加载指定路径，会解析依赖，**可能将同一程序集加载多次到不同上下文** | 加载插件，但要注意上下文问题 |
| `LoadFile(path)` | 只加载指定文件，**不解析依赖**，非常危险              | 几乎不用           |

**经典陷阱：**
```csharp
// 假设 C:\Plugins\A.dll 和 C:\Plugins\B.dll 都引用了 Newtonsoft.Json
// 用 LoadFrom 可能导致同一个 Json.dll 被加载到两个不同的加载上下文中
// 导致类型转换失败："无法将类型 A.JsonSerializer 转换为 B.JsonSerializer"
```

**最佳实践：**
- 插件系统推荐用 `AssemblyLoadContext`（.NET Core 3.0+）
- 简单场景用 `LoadFrom` 没问题，但要理解限制

---

## 三、Assembly 的核心属性和方法

### 身份信息

```csharp
Assembly asm = Assembly.GetExecutingAssembly();

// 程序集名称（AssemblyName 对象）
AssemblyName name = asm.GetName();
Console.WriteLine($"名称: {name.Name}");
Console.WriteLine($"版本: {name.Version}");
Console.WriteLine($"文化: {name.CultureName ?? "neutral"}");
Console.WriteLine($"公钥令牌: {BitConverter.ToString(name.GetPublicKeyToken() ?? new byte[0])}");

// 程序集全名（用于 Load 方法）
string fullName = asm.FullName;
// 格式：MyApp, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null
Console.WriteLine($"全名: {fullName}");

// 文件位置
Console.WriteLine($"位置: {asm.Location}");  // 文件路径
Console.WriteLine($"代码库: {asm.CodeBase}");  // URL 格式（弃用）
```

### 类型获取（最重要！）

```csharp
Assembly asm = typeof(string).Assembly;

// 1. 获取所有公共类型（最常用）
Type[] allTypes = asm.GetTypes();
Console.WriteLine($"System.Private.CoreLib 中共有 {allTypes.Length} 个公共类型");

// 2. 仅获取导出的公共类型（效果和 GetTypes 几乎一样）
Type[] exportedTypes = asm.GetExportedTypes();

// 3. 按名称获取单个类型（需要完整名称）
Type stringType = asm.GetType("System.String");
Type listType = asm.GetType("System.Collections.Generic.List`1");  // 泛型需要 `1

// 4. 获取所有模块
Module[] modules = asm.GetModules();
foreach (var module in modules)
{
    Console.WriteLine($"模块: {module.Name}");
}
```

**注意：`GetType(name)` 需要的是类型的**完全限定名 **（含命名空间），且区分大小写。**

### 创建实例（Assembly 的工厂能力）

```csharp
Assembly asm = Assembly.Load("System.Private.CoreLib");

// 方法1：Assembly.CreateInstance（内部调用 Activator.CreateInstance）
object sb = asm.CreateInstance("System.Text.StringBuilder");
// 等价于：Activator.CreateInstance(asm.GetType("System.Text.StringBuilder"))

// 方法2：配合 Activator.CreateInstance 使用
Type type = asm.GetType("System.Collections.ArrayList");
object list = Activator.CreateInstance(type);

// 方法3：创建泛型类型实例
Type genericDictDef = asm.GetType("System.Collections.Generic.Dictionary`2");
Type constructed = genericDictDef.MakeGenericType(typeof(string), typeof(int));
object dict = Activator.CreateInstance(constructed);
```

### 资源读取（嵌入式资源）

```csharp
// 假设在项目中添加了一个嵌入式资源文件：Data/config.json
// 属性中设置"生成操作" = "嵌入的资源"

Assembly asm = Assembly.GetExecutingAssembly();

// 获取所有资源名称
string[] resourceNames = asm.GetManifestResourceNames();
foreach (var name in resourceNames)
{
    Console.WriteLine($"资源: {name}");
}

// 读取特定资源
using (Stream stream = asm.GetManifestResourceStream("MyApp.Data.config.json"))
using (StreamReader reader = new StreamReader(stream))
{
    string content = reader.ReadToEnd();
    Console.WriteLine(content);
}

// 获取资源流（不关闭，用于其他处理）
Stream resourceStream = asm.GetManifestResourceStream("MyApp.Images.logo.png");
```

### 特性读取（程序集级别的 Attribute）

```csharp
Assembly asm = Assembly.GetExecutingAssembly();

// 获取程序集标题
var titleAttr = asm.GetCustomAttribute<AssemblyTitleAttribute>();
Console.WriteLine($"标题: {titleAttr?.Title}");

// 获取版本信息
var versionAttr = asm.GetCustomAttribute<AssemblyFileVersionAttribute>();
Console.WriteLine($"文件版本: {versionAttr?.Version}");

// 获取所有自定义特性
object[] attrs = asm.GetCustomAttributes(false);
foreach (var attr in attrs)
{
    Console.WriteLine($"特性: {attr.GetType().Name}");
}

// 常见的程序集特性
// [assembly: AssemblyTitle("My Application")]
// [assembly: AssemblyDescription("A sample app")]
// [assembly: AssemblyCompany("MyCompany")]
// [assembly: AssemblyProduct("MyProduct")]
// [assembly: AssemblyCopyright("Copyright © 2024")]
// [assembly: AssemblyVersion("1.0.0.0")]
// [assembly: AssemblyFileVersion("1.0.0.0")]
```

---

## 四、扫描程序集中的类型（实战）

### 基础扫描：找到所有类

```csharp
public static void ScanAssembly(string assemblyPath)
{
    try
    {
        Assembly asm = Assembly.LoadFrom(assemblyPath);
        Console.WriteLine($"扫描程序集: {asm.GetName().Name}");
        
        Type[] types = asm.GetTypes();
        
        foreach (Type type in types)
        {
            // 过滤掉编译器生成的类型
            if (type.IsClass && !type.IsAbstract && !type.IsInterface)
            {
                Console.WriteLine($"  - {type.FullName}");
            }
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"加载失败: {ex.Message}");
    }
}
```

### 高级扫描：按条件筛选类型

```csharp
/// <summary>
/// 在指定的程序集中查找所有实现了接口或继承了基类 T 的具体类。
/// </summary>
/// <typeparam name="T">要查找的接口类型或基类类型</typeparam>
/// <param name="assembly">要扫描的程序集</param>
/// <returns>符合条件的类型列表</returns>
public static List<Type> FindTypesImplementing<T>(Assembly assembly)
{
    return assembly.GetTypes()
        .Where(t => t.IsClass          // 必须是类（排除接口、结构体等）
                 && !t.IsAbstract      // 必须是具体类（排除抽象类，确保可以被实例化）
                 && typeof(T).IsAssignableFrom(t)) // 检查 t 是否实现了接口 T 或继承自 T
        .ToList();
}

/// <summary>
/// 在指定的程序集中查找所有标记了特定特性（Attribute）的类。
/// </summary>
/// <typeparam name="TAttribute">要查找的特性类型</typeparam>
/// <param name="assembly">要扫描的程序集</param>
/// <returns>带有该特性的类型列表</returns>
public static List<Type> FindTypesWithAttribute<TAttribute>(Assembly assembly)
    where TAttribute : Attribute // 泛型约束：确保 TAttribute 必须是一个特性类
{
    return assembly.GetTypes()
        .Where(t => t.GetCustomAttribute<TAttribute>() != null) // 如果能获取到该特性，说明标记了它
        .ToList();
}

// --- 使用示例 ---

// 1. 动态加载插件：从 plugins.dll 中找到所有实现了 IPlugin 接口的类
var plugins = FindTypesImplementing<IPlugin>(Assembly.LoadFrom("plugins.dll"));

// 2. 自动化扫描：在当前运行的程序集中，找到所有标记了 [TestClass] 特性的类
var testClasses = FindTypesWithAttribute<TestClassAttribute>(Assembly.GetExecutingAssembly());


```

### 完整插件扫描器示例

```csharp
public interface IPlugin
{
    string Name { get; }
    void Execute();
}

public class PluginInfo
{
    public string FilePath { get; set; }
    public Type PluginType { get; set; }
    public Assembly Assembly { get; set; }
}

public class PluginScanner
{
    public List<PluginInfo> ScanDirectory(string directoryPath)
    {
        var plugins = new List<PluginInfo>();
        
        foreach (string dllPath in Directory.GetFiles(directoryPath, "*.dll"))
        {
            try
            {
                Assembly asm = Assembly.LoadFrom(dllPath);
                
                var pluginTypes = asm.GetTypes()
                    .Where(t => t.IsClass && !t.IsAbstract && typeof(IPlugin).IsAssignableFrom(t));
                
                foreach (var type in pluginTypes)
                {
                    plugins.Add(new PluginInfo
                    {
                        FilePath = dllPath,
                        PluginType = type,
                        Assembly = asm
                    });
                    
                    Console.WriteLine($"发现插件: {type.FullName} (来自 {Path.GetFileName(dllPath)})");
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"扫描 {dllPath} 失败: {ex.Message}");
            }
        }
        
        return plugins;
    }
    
    public void ExecutePlugins(List<PluginInfo> plugins)
    {
        foreach (var plugin in plugins)
        {
            try
            {
                IPlugin instance = (IPlugin)Activator.CreateInstance(plugin.PluginType);
                Console.WriteLine($"执行插件: {instance.Name}");
                instance.Execute();
            }
            catch (Exception ex)
            {
                Console.WriteLine($"执行 {plugin.PluginType.Name} 失败: {ex.Message}");
            }
        }
    }
}
```

---

## 五、程序集版本和依赖

### 读取程序集版本

```csharp
Assembly asm = Assembly.GetExecutingAssembly();
Version version = asm.GetName().Version;
Console.WriteLine($"主版本: {version.Major}");
Console.WriteLine($"次版本: {version.Minor}");
Console.WriteLine($"内部版本: {version.Build}");
Console.WriteLine($"修订号: {version.Revision}");
```

### 获取程序集依赖

```csharp
Assembly asm = Assembly.Load("System.Text.Json");

// 获取引用的程序集
AssemblyName[] referencedAssemblies = asm.GetReferencedAssemblies();
foreach (var refAsm in referencedAssemblies)
{
    Console.WriteLine($"引用: {refAsm.Name}, 版本: {refAsm.Version}");
}

// 递归获取所有依赖
public static void PrintDependencyTree(Assembly asm, int indent = 0)
{
    string indentStr = new string(' ', indent * 2);
    Console.WriteLine($"{indentStr}{asm.GetName().Name} v{asm.GetName().Version}");
    
    foreach (var refName in asm.GetReferencedAssemblies())
    {
        try
        {
            Assembly refAsm = Assembly.Load(refName);
            PrintDependencyTree(refAsm, indent + 1);
        }
        catch
        {
            Console.WriteLine($"{indentStr}  [无法加载: {refName.Name}]");
        }
    }
}
```

---

## 六、模块（Module）概念

### 什么是模块？

> **模块（Module）** 是程序集内部的逻辑分组。大多数程序集只有一个模块（主模块），但理论上可以有多模块程序集。

```csharp
Assembly asm = Assembly.GetExecutingAssembly();

Module[] modules = asm.GetModules();
foreach (Module module in modules)
{
    Console.WriteLine($"模块名称: {module.Name}");
    Console.WriteLine($"  是否主模块: {asm.ManifestModule == module}");
    Console.WriteLine($"  MDStreamVersion: {module.MDStreamVersion}");
    
    // 获取模块中的类型
    Type[] types = module.GetTypes();
    Console.WriteLine($"  包含 {types.Length} 个类型");
}
```

### 何时需要关心模块？

- 多模块程序集（罕见，通常由工具生成）
- 调试和符号文件（.pdb）相关
- 高级代码生成场景

**99% 的情况下，你只需要关心 Assembly 和 Type，不需要直接操作 Module。**

---

## 七、常见陷阱与最佳实践

### 陷阱：加载同一个 DLL 多次

```csharp
// ❌ 危险：同一个物理文件被加载两次，产生两个不同的 Assembly 对象
Assembly asm1 = Assembly.LoadFrom(@"C:\Plugins\MyPlugin.dll");
Assembly asm2 = Assembly.LoadFrom(@"C:\Plugins\MyPlugin.dll");
Console.WriteLine(asm1 == asm2);  // False！同一份文件，两个不同对象

// 类型也不相等
Type t1 = asm1.GetType("MyPlugin.PluginClass");
Type t2 = asm2.GetType("MyPlugin.PluginClass");
Console.WriteLine(t1 == t2);  // False！
```

**解决方案：** 使用 `AssemblyLoadContext`（.NET Core 3.0+）或缓存已加载的程序集。

### 陷阱：LoadFrom 导致的类型不匹配

```csharp
// 主程序引用了 Newtonsoft.Json 11.0
// 插件也引用了 Newtonsoft.Json 11.0，但用 LoadFrom 加载
// 结果：JsonConvert 类型可能来自两个不同的上下文，导致转换失败
```

**解决方案：** 
- 确保依赖版本一致
- 使用 `AssemblyLoadContext.Default.LoadFromAssemblyPath()`（.NET Core）
- 或将依赖合并到单个上下文

### 最佳实践：缓存 Assembly 对象
**确保同一个 DLL 文件在内存中只被加载并管理一次**。
避免重复加载的性能开销
保证“引用一致性”
```csharp
public static class AssemblyCache
{
    private static readonly Dictionary<string, Assembly> _cache = new();
    
    public static Assembly Load(string path)
    {
        string fullPath = Path.GetFullPath(path);
        
        lock (_cache)
        {
            if (!_cache.TryGetValue(fullPath, out var asm))
            {
                asm = Assembly.LoadFrom(fullPath);
                _cache[fullPath] = asm;
            }
            return asm;
        }
    }
}
```

### 最佳实践：使用 AssemblyLoadContext（.NET Core 3.0+）
之前的 `AssemblyCache` 是为了 **“存着不丢”**，那么这段代码就是为了 **“用完就扔”**。
**它在解决什么痛点？**
在传统的 .NET 开发中，程序集（DLL）一旦加载到内存中，就像“泼出去的水”，是**无法单独卸载**的。如果你想更新一个插件 DLL，你必须关闭整个程序，替换文件，再重启。

`AssemblyLoadContext` (简称 **ALC**) 就像给插件提供了一个**隔离的“沙盒”**。当这个沙盒被销毁（Unload）时，里面加载的所有 DLL 都会被从内存中抹除，释放文件占用。
```csharp
using System.Runtime.Loader;

// 继承 AssemblyLoadContext 相当于定义了一个可以回收的“容器”
public class IsolatedAssemblyLoadContext : AssemblyLoadContext
{
    // 构造函数通常会设置 isCollectible: true，表示这个容器支持卸载
    public IsolatedAssemblyLoadContext() : base(isCollectible: true) { }

    protected override Assembly Load(AssemblyName assemblyName)
    {
        // 自定义加载逻辑
        // 这里返回 null 是告诉程序：
        // “如果在我的沙盒里找不到某个依赖包，请去主程序那里找。”
        return null; 
    }
}

// 使用独立的上下文，可以卸载程序集
var context = new IsolatedAssemblyLoadContext();
// 将插件加载进这个特定的沙盒，而不是主程序内存
Assembly asm = context.LoadFromAssemblyPath(@"C:\Plugins\MyPlugin.dll");
// ... 使用插件 ...
context.Unload();  // 卸载，释放内存（.NET Core 3.0+ 支持）
```

---

## 八、完整实战：程序集信息查看器

```csharp
public class AssemblyInspector
{
    public static void Inspect(string assemblyPath)
    {
        Console.WriteLine($"═══════════════════════════════════════════════════");
        Console.WriteLine($"程序集信息查看器");
        Console.WriteLine($"文件: {assemblyPath}");
        
        try
        {
            Assembly asm = Assembly.LoadFrom(assemblyPath);
            
            // 1. 基本标识
            var name = asm.GetName();
            Console.WriteLine($"\n【程序集标识】");
            Console.WriteLine($"  名称: {name.Name}");
            Console.WriteLine($"  版本: {name.Version}");
            Console.WriteLine($"  文化: {name.CultureName ?? "(neutral)"}");
            Console.WriteLine($"  公钥令牌: {(name.GetPublicKeyToken()?.Length > 0 ? BitConverter.ToString(name.GetPublicKeyToken()) : "null")}");
            Console.WriteLine($"  全名: {name.FullName}");
            
            // 2. 文件和位置
            Console.WriteLine($"\n【文件信息】");
            Console.WriteLine($"  位置: {asm.Location}");
            Console.WriteLine($"  是否动态: {asm.IsDynamic}");
            Console.WriteLine($"  是否完全信任: {asm.IsFullyTrusted}");
            
            // 3. 程序集特性
            Console.WriteLine($"\n【程序集特性】");
            var title = asm.GetCustomAttribute<AssemblyTitleAttribute>();
            if (title != null) Console.WriteLine($"  标题: {title.Title}");
            var desc = asm.GetCustomAttribute<AssemblyDescriptionAttribute>();
            if (desc != null) Console.WriteLine($"  描述: {desc.Description}");
            var company = asm.GetCustomAttribute<AssemblyCompanyAttribute>();
            if (company != null) Console.WriteLine($"  公司: {company.Company}");
            var copyright = asm.GetCustomAttribute<AssemblyCopyrightAttribute>();
            if (copyright != null) Console.WriteLine($"  版权: {copyright.Copyright}");
            
            // 4. 依赖项
            Console.WriteLine($"\n【依赖程序集】");
            foreach (var refName in asm.GetReferencedAssemblies())
            {
                Console.WriteLine($"  {refName.Name} v{refName.Version}");
            }
            
            // 5. 类型统计
            Console.WriteLine($"\n【类型统计】");
            var types = asm.GetTypes();
            int classes = types.Count(t => t.IsClass && !t.IsInterface);
            int interfaces = types.Count(t => t.IsInterface);
            int enums = types.Count(t => t.IsEnum);
            int structs = types.Count(t => t.IsValueType && !t.IsEnum);
            
            Console.WriteLine($"  类: {classes}");
            Console.WriteLine($"  接口: {interfaces}");
            Console.WriteLine($"  枚举: {enums}");
            Console.WriteLine($"  结构体: {structs}");
            Console.WriteLine($"  总计: {types.Length}");
            
            // 6. 资源
            var resources = asm.GetManifestResourceNames();
            if (resources.Length > 0)
            {
                Console.WriteLine($"\n【嵌入式资源】");
                foreach (var res in resources)
                {
                    Console.WriteLine($"  {res}");
                }
            }
            
            // 7. 模块
            Console.WriteLine($"\n【模块】");
            foreach (var module in asm.GetModules())
            {
                Console.WriteLine($"  {module.Name} ({(module == asm.ManifestModule ? "主模块" : "子模块")})");
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine($"\n❌ 加载失败: {ex.Message}");
        }
    }
}

// 使用
AssemblyInspector.Inspect(typeof(string).Assembly.Location);
```

---

## 九、本篇速查表

| 需求 | 方法 | 返回 |
|------|------|------|
| 获取当前程序集 | `Assembly.GetExecutingAssembly()` | `Assembly` |
| 获取入口程序集 | `Assembly.GetEntryAssembly()` | `Assembly` |
| 从类型获取程序集 | `typeof(T).Assembly` | `Assembly` |
| 按名称加载 | `Assembly.Load("System.Text.Json")` | `Assembly` |
| 按路径加载 | `Assembly.LoadFrom(@"C:\lib.dll")` | `Assembly` |
| 获取所有类型 | `asm.GetTypes()` | `Type[]` |
| 获取单个类型 | `asm.GetType("Namespace.Class")` | `Type` |
| 创建实例 | `asm.CreateInstance("Namespace.Class")` | `object` |
| 读取资源 | `asm.GetManifestResourceStream("name")` | `Stream` |
| 获取程序集版本 | `asm.GetName().Version` | `Version` |
| 获取依赖 | `asm.GetReferencedAssemblies()` | `AssemblyName[]` |

---

## 十、思考题

1. `Assembly.Load` 和 `Assembly.LoadFrom` 的核心区别是什么？为什么会导致类型不等问题？

2. 如果你的插件系统需要支持卸载插件（释放内存），应该使用什么技术？

3. 如何判断一个程序集是 `.exe` 还是 `.dll`？（提示：`EntryPoint` 属性）

4. 嵌入式资源和磁盘文件资源各有什么优缺点？什么场景下用嵌入式资源？

---

**下一篇预告：** 《C# 反射系列笔记（四）：动态创建对象——不止 new 一种方式》

下一篇将深入讲解 `Activator`、`ConstructorInfo.Invoke`、委托编译等多种对象创建方式，对比性能，并揭示创建泛型实例的正确方法。