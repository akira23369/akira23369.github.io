---
title: CSharp语言基础014~024Note
date: 2026-03-17 12:39:57
toc: true
categories:
  - 编程语言
  - Csharp
  - 02CSharp语言基础

tags:
  - 编程语言
  - Csharp
  - 02CSharp语言基础

---


# C#语言基础

## 第一讲：Visual Studio 安装指南

### 简介

Visual Studio 是一款专业的集成开发环境（IDE），专为 C# 开发设计。它不仅是代码编辑器，更是一个功能完备的开发工作台，集成了构建、调试及高效管理应用程序所需的各类专业工具。

### 运作原理与最佳实践

安装 Visual Studio 时需选择工作负载，即针对不同开发类型预置的工具包。其中".NET 桌面开发"工作负载是必备项，它包含了 C#编译器、.NET 框架及项目模板，这正是我们所需的。

### 面试视角

若面试官问及你最喜欢的 Visual Studio 功能，可以重点介绍调试器和智能感知(IntelliSense)这两个特性。

- "调试器：说明这是你排查问题的主要工具。通过设置断点可暂停程序运行，支持逐行执行代码并实时查看变量状态。"
    
- IntelliSense：一种智能代码补全工具，能够实时推荐代码、检测输入错误，并辅助探索 C#语言的各项特性。
    

## 第二讲：创建首个 C#应用程序

### 简介

"Hello, World!"程序是入门的第一个基础步骤。它能验证开发环境配置是否正确，并帮助您了解 C#程序的基本架构。创建项目时，请务必选择“控制台应用(.NET Framework)”模板。

### 工作原理：程序的结构解析

Visual Studio 生成的代码存放在 Program.cs 文件中。

​  
namespace HelloWorld  
{  
    class Program  
    {  
        static void Main(string[] args)  
        {  
            Console.WriteLine("Hello, World!");  
        }  
    }  
}

- `namespace`：代码整理工具，类似文件夹的功能。
    
- `class Program`：用于封装方法和数据的容器。
    
- `static void Main(string[] args)`：此处为程序入口点。运行程序时，.NET 运行时会自动定位并从此特定方法开始执行。
    

### 面试视角

"解释这行代码： static void Main(string[] args) ”：这是面试中经常被问到的问题。"

- 该方法是直接通过 Program 类本身调用的，而非其实例化的对象。
    
- 此方法无返回值。
    
- `Main`: 该入口点的约定命名。
    
- 该参数用于接收以字符串数组形式传递的命令行参数。
    

## 第三讲： System.Console 类

### 简介

Console 类是与文本控制台窗口交互的核心工具，提供了基础的输入输出功能方法。

### 常用方法

- `Console.WriteLine(string value)`：将指定文本输出至控制台，并自动换行。
    
- `Console.Write(string value)`: 打印文本但光标不换行。
    
- `Console.ReadLine()`：读取用户输入的整行文本，并将其作为 string 返回。
    
- `Console.ReadKey()`：等待一次按键输入。
    
- `Console.Clear()`: 清空控制台窗口内的全部文字内容。
    

### 面试视角

- Write 与 WriteLine 的区别： WriteLine 打印后会自动换行，而 Write 则不会。
    
- 输入处理：一个常见问题是，“当要求用户输入年龄时，该如何存储？”关键在于理解 Console.ReadLine() 始终返回的是 string 。因此，必须通过类似 int.Parse() 的方法显式地将该字符串转换为数字。
    

## 第四讲：变量

### 简介

变量是内存中的命名存储单元，用于存放程序可操作的数据值。

### 运行机制：作用域与内存管理

在方法内部声明基本类型变量时，其值会存储在栈内存中——这是一块访问速度极快的内存区域。该变量的生命周期与其作用域（即声明该变量的代码块{...}）相关联。

### 最佳实践及面试要点

- 命名规范：局部变量统一采用驼峰式命名法（如 userAge 、 firstName ）。
    
- 作用域：需准备好应对类似这样的问题，例如“若在 if 代码块中声明一个变量，能否在 else 块中使用？”答案是否定的，因为它们属于不同的作用域。
    

## 第五讲：基本数据类型

### 简介

原始类型是基础的数据结构单元。C#属于静态类型语言，因此必须先声明变量类型才能使用。

### 整数类型（整型数）

这些数据类型用于存储不带小数部分的整数值，主要区别在于其存储范围大小以及是否支持负数。

- `sbyte`：8 位有符号整型（取值范围为-128 至 127）。
    
- `byte`: 8 位无符号整型（取值范围 0-255）。适用于不能出现负值的场景，如文件或网络流中的原始数据。
    
- `short`：16 位有符号整型（取值范围-32,768 到 32,767）。
    
- `ushort`：16 位无符号整数（取值范围 0 至 65,535）。
    
- `int`：32 位有符号整数（取值范围-21 亿至 21 亿）。此为默认且最常用的整数类型，若无特殊需求，建议直接使用。
    
- `uint`：32 位无符号整型（取值范围 0 至 42 亿）。
    
- `long`：64 位有符号整数（取值范围极大）。当 int 的范围不足时使用。如需定义长整型字面值，请添加 L 后缀： `long bigNumber = 9000000000L;`。
    
- `ulong`：64 位无符号整数。
    

### 浮点数类型（十进制数字）

这类数据类型用于存储带小数点的数值，选择时需在取值范围、精度和性能之间进行权衡。

- `float`：32 位单精度浮点数（精度约为 6-9 位）。适用于需要小数但对精度要求不高的场景，例如图形编程。使用时添加 F 后缀： `float height = 1.88F;`。
    
- `double`：64 位双精度浮点数（精度约为 15-17 位）。此为默认且最常用的浮点类型，适用于科学计算及常规运算。
    
- `decimal`：128 位高精度（精确到 28-29 位数字）。该类型虽比 double 慢，但能避免二进制浮点数固有的微小舍入误差。进行财务或货币计算时务必使用 decimal 类型，并使用 M 后缀： `decimal accountBalance = 100.05M;`。
    

### 其他基本类型

- `char`：代表一个 16 位的 Unicode 字符，其值由单引号包裹，如： `char grade = 'A';`。
    
- `bool`：代表一个逻辑值，其取值可能为 true 或 false 。
    

## 第六讲：操作符

### 简介

运算符是一种特殊符号，用于对变量及数值（即操作数）执行运算操作。

### 算术运算符

- `+`（加法运算）：对两个操作数进行相加。 `int sum = 10 + 5; // 15`
    
- `-`（减法运算）：用左操作数减去右操作数。 `int diff = 10 - 5; // 5`
    
- `*`（乘法）：用于计算两个操作数的乘积。 `int prod = 10 * 5; // 50`
    
- `/`（除法）：用右操作数除左操作数，具体运算行为由操作数的类型决定。
    
    - 整数相除：若两操作数均为整数，则结果取整，小数部分直接舍去。 `int result = 10 / 4; // result is 2`
        
    - 浮点数除法规则：若任一操作数为浮点类型，则结果也为浮点类型。 `double result = 10.0 / 4; // result is 2.5`
        
- `%`（取模运算）：返回两数相除后的余数。 `int remainder = 10 % 3; // remainder is 1` ，常用于判断一个数是否为偶数（ `num % 2 == 0`）。
    

### 赋值操作符

- `=`（简单赋值）：将右侧操作数的值赋予左侧操作数。 `int x = 10;`
    
- `+=`、 `-=`、 `*=`、 `/=`、 `%=`（复合赋值运算符）：可一步完成运算与赋值操作。 `x += 5;` 实际上是 `x = x + 5;` 的简写形式。
    

### 比较操作符

它们对两个操作数进行比较，并始终返回一个 bool（ true 或 false ）。

- `==`（等同于）： `bool areEqual = (name == "John");`
    
- `!=`（不等于） `bool areNotEqual = (age != 30);`
    
- `>`、 `<`、 `>=`、 `<=`（如大于、小于等关系）： `bool isAdult = (age >= 18);`
    

### 逻辑运算符

这些用于合并 bool 表达式。

- `&&`（逻辑与运算）：当且仅当两个操作数均为真时，才返回 true 。
    
- `||`（逻辑或运算）：当任意一个操作数为真时，返回 true 。
    
- `!`（逻辑非运算符）：用于取反布尔值。 `!true` 即为 false 。
    

### 三目运算符

- `?:`（条件运算符）：简化 if-else 语句的快捷方式。 `string message = (score >= 50) ? "Pass" : "Fail";`
    

### 访谈观点：数据类型及其赋值操作

- 字面量：支持多种赋值格式，面试中可能会被考察这一知识点。
    
    - 十六进制： `int hex = 0x2A; // 42 in decimal`
        
    - 二进制: `int bin = 0b00101010; // 42 in decimal`
        
- 类型转换与数据丢失：这是面试中的重点考察内容。
    
    - 隐式类型转换（安全）：由较小数据类型自动转换为较大类型且不会造成数据丢失的情况，C#默认支持此类转换。 `int i = 100; double d = i; // d is 100.0`
        
    - 显式转换（不安全）：将较大类型转换为较小类型时可能导致数据丢失。你必须使用强制转换 `(type)` 明确告知编译器你已知晓此风险。 `double d = 9.78; int i = (int)d; // i is 9. The decimal part is lost.` 面试官通常会要求你解释这种潜在数据丢失的概念。
        

## 第七讲： if 及其变体形式

### 简单的 if 声明

适用场景：当且仅当满足某一条件时执行特定代码块，否则不进行任何操作。

运行机制：首先计算括号内的布尔表达式。若结果为真，则执行大括号内的代码块；若为假，则跳过该代码块。

示例：

​  
int currentStock = 12;  
if (currentStock < 10)  
{  
    Console.WriteLine("Warning: Stock is low. Time to re-order.");  
}

### if-else 声明

适用场景：当条件判断为真时执行某段代码，为假时则执行另一段代码的情况。

运行机制：当条件成立时，执行第一个代码块（if）；否则执行第二个代码块（else）。两者必居其一，总有一个代码块会被执行。

示例：

​  
int userAge = 17;  
if (userAge >= 18)  
{  
    Console.WriteLine("Access granted to the restricted area.");  
}  
else  
{  
    Console.WriteLine("Access denied. You must be 18 or older.");  
}

### if-else if 梯子

使用场景：当需要依次检查多个相关条件时。

运行机制：系统会从上至下依次判断条件。当遇到第一个满足的条件时，执行对应的代码块，并跳过之后所有的 else if 和 else 语句。最后的 else 语句是可选的，用于处理所有前置条件均不满足的情况。

示例：

​  
double score = 85.5;  
if (score >= 90)  
{  
    Console.WriteLine("Grade: A");  
}  
else if (score >= 80)  
{  
    Console.WriteLine("Grade: B");  
}  
else if (score >= 70)  
{  
    Console.WriteLine("Grade: C");  
}  
else  
{  
    Console.WriteLine("Grade: F");  
}

### 嵌套的 if 语句

适用场景：当某个条件的检查前提是另一个外部条件必须首先成立时。

运行机制：仅当外层 if 语句的条件成立时，才会执行内层 if 语句的判断。

示例：

​  
bool hasTicket = true;  
bool isVip = false;  
if (hasTicket)  
{  
    Console.WriteLine("Welcome! Please proceed.");  
    if (isVip)  
    {  
        Console.WriteLine("Please enjoy the VIP lounge.");  
    }  
}  
else  
{  
    Console.WriteLine("You need a ticket to enter.");  
}

最佳实践：避免过度嵌套 if 语句（超过 2 至 3 层），否则会导致代码可读性大幅降低，难以理解。

## 第八讲: switch-case

适用场景：当需要对单个变量进行多重条件判断（且条件均为特定常量值）时，使用 switch 语句比冗长的 if-else if 条件链更简洁清晰，可读性也更强。

其工作原理如下：程序会将 switch()中的变量值依次与每个 case 的值进行比对。一旦发现匹配项，即执行对应的 case 代码块，直至遇到 break 语句为止，此时程序将跳出整个 switch 结构。若所有 case 均不匹配，则自动执行 default 部分的代码。

示例：

​  
string userRole = "admin";  
switch (userRole.ToLower()) // Good practice to normalize string input  
{  
    case "admin":  
        Console.WriteLine("You have full access.");  
        break;  
    case "editor":  
        Console.WriteLine("You can create and edit content.");  
        break;  
    case "viewer":  
        Console.WriteLine("You can only view content.");  
        break;  
    default:  
        Console.WriteLine("Unknown role. Access denied.");  
        break;  
}

## 第九讲： while 循环

适用场景：当需要根据条件进行循环且无法预先确定具体迭代次数时。

运行机制：每次循环开始前都会先判断布尔条件。若条件为真，则执行循环体内的代码；该过程会不断重复，直至条件变为假。若初始条件即为假，则循环体内的代码不会被执行。

示例：

### 计数控制循环：

​  
int ticketsLeft = 5;  
while (ticketsLeft > 0)  
{  
    Console.WriteLine($"Selling a ticket... {ticketsLeft} tickets remaining.");  
    ticketsLeft--; // This change is crucial to prevent an infinite loop  
}  
Console.WriteLine("All tickets sold!");

### 哨兵控制循环（等待用户输入）：

​  
string command = "";  
while (command != "quit")  
{  
    Console.Write("Enter a command ('help', 'run', 'quit'): ");  
    command = Console.ReadLine();  
    // ... process the command ...  
}

## 第十讲： do-while 循环

适用场景：当您需要确保循环体至少执行一次的特殊情况下使用。

运行机制：先执行循环体内的代码，随后判断布尔条件是否成立。若成立，则继续循环。

示例（经典应用场景：输入验证）：

​  
int age;  
do  
{  
    Console.Write("Please enter your age (must be 18 or older): ");  
    // This code must run at least once to get the input  
    string input = Console.ReadLine();  
    age = int.Parse(input);  
    if (age < 18)  
    {  
        Console.WriteLine("Invalid age. Please try again.");  
    }  
} while (age < 18); // Check the condition after the first run  
Console.WriteLine("Age successfully validated.");

## 第 11 讲： for 循环

适用场景：在循环开始前已知迭代次数的情况下使用。

### 运行机制解析：

for 循环（初始化；条件判断；迭代）

- initializer 会在初始阶段运行一次。
    
- 每次循环开始前都会检查 condition 。
    
- 若条件成立，则执行循环体。
    
- 每次迭代后，iterator 都会执行。
    

### 常见示例：

#### 从 0 开始递增至 n-1（针对数组的情况）：

​  
string[] shoppingList = { "Apples", "Bananas", "Carrots" };  
for (int i = 0; i < shoppingList.Length; i++)  
{  
    Console.WriteLine($"Item {i + 1}: {shoppingList[i]}");  
}

#### 倒计时循环（火箭发射）：

​  
for (int i = 10; i > 0; i--)  
{  
    Console.WriteLine($"{i}...");  
}  
Console.WriteLine("Liftoff!");

#### 跳过特定数字的循环（偶数部分）：

​  
for (int i = 2; i <= 10; i += 2)  
{  
    Console.WriteLine(i);  
}

## 第十二讲： break 与 continue

使用场景：在循环体内部手动控制循环流程时。

- `break`：当需要立即跳出循环时使用，无论循环条件是否满足。例如，已找到目标项时即可使用。
    
- `continue`：用于跳过当前项并直接进入下一次循环的情况。例如，可用来忽略列表中的无效数据。
    

示例：

​  
// Find the first multiple of 7, but ignore numbers less than 20.  
for (int i = 1; i <= 100; i++)  
{  
    if (i < 20)  
    {  
        continue; // Skip this number and go to the next i  
    }  
    if (i % 7 == 0)  
    {  
        Console.WriteLine($"Found the first multiple of 7 greater than 20: {i}");  
        break; // Found it, exit the loop completely  
    }  
}

## 第 13 讲：嵌套循环

适用场景：当您需要处理二维结构时，例如网格、乘法表或游戏棋盘等场景。

运行机制：每当外部循环进行一次迭代时，内部循环都会完整执行一遍所有步骤。

示例（乘法口诀表）：

​  
// Outer loop for the first number in the multiplication  
for (int i = 1; i <= 5; i++)  
{  
    // Inner loop for the second number  
    for (int j = 1; j <= 5; j++)  
    {  
        // Use Write to keep everything on the same line for this row  
        Console.Write($"{i * j}\t"); // \t adds a tab for spacing  
    }  
    // After the inner loop finishes, move to the next line for the next row  
    Console.WriteLine();  
}

## 第 14 讲： goto

说明与警示：goto 语句可实现无条件跳转至 label 。尽管该语法属于编程语言的一部分，但在现代编程实践中强烈不建议使用。它会导致代码结构混乱（俗称“面条代码”），难以理解和维护。结构化控制流程（如 if 、 switch 及循环结构）才是更佳选择。面试时，考官会默认你知晓这一原则，并期待你明确反对使用 goto 。

## 第 15 讲：利用嵌套 for 循环实现图案打印

引言：此经典练习旨在考察你对嵌套循环及算法思维的理解与应用能力。

### 示例 1：直角三角形

​  
int size = 5;  
for (int row = 1; row <= size; row++)  
{  
    for (int col = 1; col <= row; col++)  
    {  
        Console.Write("*");  
    }  
    Console.WriteLine();  
}

运行机制：外层循环负责控制行数。内层循环的条件 `col <= row` 是核心所在，它会根据当前行号打印相应数量的星号。

### 示例 2：倒直角三角形

​  
int size = 5;  
for (int row = size; row >= 1; row--)  
{  
    for (int col = 1; col <= row; col++)  
    {  
        Console.Write("*");  
    }  
    Console.WriteLine();  
}

实现原理：逻辑顺序被反转。外层循环从 size 开始递减计数，内层循环依然输出与当前递减的 row 值对应的星号数量。

### 示例三：金字塔

这是一种更复杂的模式，需通过第三个循环控制间距。

​  
int size = 5;  
for (int row = 1; row <= size; row++)  
{  
    // Loop for leading spaces  
    for (int space = 1; space <= size - row; space++)  
    {  
        Console.Write(" ");  
    }  
    // Loop for the stars  
    for (int star = 1; star <= (2 * row) - 1; star++)  
    {  
        Console.Write("*");  
    }  
    Console.WriteLine();  
}

工作原理：

- 外层循环：控制编号 row 的数量。
    
- 空格循环：针对每个 row ，程序会输出 `size - row` 个空格。例如，第 1 行输出 4 个空格，第 5 行则输出 0 个空格，以此实现图案居中效果。
    
- 星形循环：打印 `(2 * row) - 1` 颗星。首行打印 1 颗，第二行 3 颗，第三行 5 颗，以此类推形成金字塔形状。