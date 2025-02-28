---
title: csharp特性
date: 2025-02-18 17:18:38
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---
# 特性

特性是一种允许我们向程序的[[程序集]]添加[[元数据]]的语言结构
它是用于保存程序结构信息的某种特殊类型的类

特性提供功能强大的方法以将声明信息与 C# 代码（类型、方法、属性等）相关联。
特性与程序实体关联后，即可在运行时使用[[csharp反射]]查询特性信息

特性的目的是告诉编译器把程序结构的某组[[元数据]]嵌入[[程序集]]中
它可以放置在几乎所有的声明中（类、变量、函数等等申明）

说人话：
特性本质是个类
我们可以利用特性类为[[元数据]]添加额外信息
比如一个类、成员变量、成员方法等等为他们添加更多的额外信息
之后可以通过[[csharp反射]]来获取这些额外信息


## 自定义特性
```csharp
class MyCustomAttribute : Attribute
{
    //特性中的成员 一般根据需求来写
    public string info;
    public MyCustomAttribute(string info)
    {
        this.info = info;
    }
    public void TestFun()
    {
        Console.WriteLine("特性的方法");
    }
}
```


## 特性的使用
基本语法: `[特性名(参数列表)]`
本质上 就是在调用特性类的构造函数
写在哪里？ 类、函数、变量上一行，表示他们具有该特性信息

```csharp
[MyCustom("这是我自己的类")]
class MyClass
{
    [MyCustom("这是一个int型的成员变量")]
    public int Value;
    [MyCustom("这是用来测试的成员函数")]
    public void TestFun([MyCustom("这是用来测试的函数参数")] int a)
    {
        Console.WriteLine(a);
    }
}

Type t = typeof(MyClass);
if (t.IsDefined(typeof(MyCustomAttribute), false))
{
    Console.WriteLine("判断一个类型是否使用了某个特性");
}
// 获取Type元数据中的所有特性
object[] objects = t.GetCustomAttributes(true);
for (int i = 0; i < objects.Length; i++)
    Console.WriteLine((objects[i] as MyCustomAttribute).Info);


```

## 为特性类加特性  限制其使用范围

```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Struct, AllowMultiple = true, Inherited = true)]
//参数一：AttributeTargets —— 特性能够用在哪些地方
//参数二：AllowMultiple —— 是否允许多个特性实例用在同一个目标上
//参数三：Inherited —— 特性是否能被派生类和重写成员继承
public class MyCustom2Attribute : Attribute { }
```


## 系统自带特性——过时特性 Obsolete
用于提示用户 使用的方法等成员已经过时 建议使用新方法

```csharp
//一般加在函数前的特性
[Obsolete("这个是旧方法", false)]
// 第一个参数是提示信息, 第二个参数 true直接报错, false警告
public void OldFun() { }


```

## 系统自带特性——调用者信息特性    (快速查出哪里的错误)
```csharp
//哪个文件调用？
//CallerFilePath特性
//哪一行调用？
//CallerLineNumber特性
//哪个函数调用？
//CallerMemberName特性

class TestClass
{
    [Obsolete("这个是旧方法", false)]
    // 第一个参数是提示信息, 第二个参数 true直接报错, false警告
    public void OldFun()
    {
    }
    public void NewFun(string str, [CallerFilePath] string fileName = "",
        [CallerLineNumber] int line = 0,
        [CallerMemberName] string memberName = "")
    {
        Console.WriteLine("这些使用的CallerXXX特性会在, 其它调用者调用时自动将调用者的额外信息填入参数中");
        Console.WriteLine("调用者的文件名 fileName = " + fileName);
        Console.WriteLine("调用者的所在的行数 line = " + line);
        Console.WriteLine("调用者的所处的调用函数名 memberName = " + memberName);
    }
}

```


## 系统自带特性——条件编译特性
Conditional 它会和[[预处理指令]] `#define配合使用`
```csharp
[Conditional("Fun")]        // 如果 #define 下面就会编译
static void Fun()
{
    Console.WriteLine("如果有#define 我就会执行");
}

```


## 系统自带特性——外部Dll包函数特性 DllImport
用来标记非.Net(C#)的函数，表明该函数在一个外部的DLL中定义。
一般用来调用 C或者C++的Dll包写好的方法
需要引用[[命名空间]] using System.Runtime.InteropServices

```csharp
[DllImport("Test.dll")]         // 下面这个函数一定在dll中有一模一样的
public static extern int Add(int a, int b);
//需要引用命名空间 using System.Runtime.CompilerServices;
```




