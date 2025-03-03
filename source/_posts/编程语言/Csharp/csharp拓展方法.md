---
title: csharp拓展方法
date: 2025-03-02 14:56:36
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp

---

# 拓展方法基本概念
C# 中的扩展方法是 C# 3.0 引入的一项特性，它允许开发者在不修改原始类或对象的情况下，向现有类添加新方法

**作用**
1. 提升程序拓展性
2. 不需要再对象中重新写方法
3. 不需要继承来添加方法
4. 为别人封装的类型写额外的方法

**特点**
1. 一定是写在静态类中
2. 一定是个静态函数
3. 第一个参数为拓展目标
4. 第一个参数用this修饰

# 基本语法
`访问修饰符 static 返回值 函数名(this 拓展类名 参数名, 参数类型 参数名,参数类型 参数名....)`


```c#
static class Tools
{
    //为int拓展了一个成员方法 
    //成员方法 是需要 实例化对象后 才 能使用的
    //value 代表 使用该方法的 实例化对象
    public static void SpeakValue(this int value)
    {
        //拓展的方法 的逻辑
        Console.WriteLine("为int拓展的方法" + value);
    }

    public static void SpeakStringInfo(this string str, string str2, string str3)
    {
        Console.WriteLine("为string拓展的方法");
        Console.WriteLine("调用方法的对象" + str);
        Console.WriteLine("传的参数" + str2 + str3);
    }

    public static void Fun3(this Test t)
    {
        Console.WriteLine("为test拓展的方法");
    }

}
// 为自定义的类型拓展方法
class Test
{
    public int i = 10;
    public void Fun1()
    {
        Console.WriteLine("123");
    }

    public void Fun2()
    {
        Console.WriteLine("456");
    }
}
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("拓展方法");
        int i = 10;
        i.SpeakValue();
        string str = "000";
        str.SpeakStringInfo("卧槽", "nm");
        Test t = new Test();
        t.Fun2();
    }
}

//总结：
//概念：为现有的非静态 变量类型 添加 方法
//作用：
// 提升程序拓展性
// 不需要再在对象中重新写方法
// 不需要继承来添加方法
// 为别人封装的类型写额外的方法

//特点：
//静态类中的静态方法
//第一个参数 代表拓展的目标
//第一个参数前面一定要加 this

//注意：
//可以有返回值 和 n个参数
//根据需求而定

```