---
title: csharp的using语句
date: 2025-03-06 17:11:17
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---

# using语句
**基本概念**
`using` 语句提供了一种简洁的方式来管理实现了 `System.IDisposable` 接口的对象。实现了 `IDisposable` 接口的对象通常包含需要手动释放的非托管资源，如文件句柄、数据库连接、网络连接、图形资源等。`using` 语句会在代码块结束时自动调用对象的 `Dispose` 方法，从而确保资源被正确释放，避免资源泄漏。

**工作原理**
`using` 语句的基本语法如下：
```cs
using (ResourceType resource = new ResourceType())
{
    // 使用资源的代码
}
```
当执行到 `using` 语句时，会创建 `ResourceType` 类型的对象并将其赋值给 `resource` 变量。然后进入 `using` 语句的代码块，在代码块中可以使用该资源。当代码块执行完毕（无论是正常结束还是因为异常而跳出），`using` 语句会自动调用 `resource` 对象的 `Dispose` 方法，释放其所占用的资源。


实际上，上述 `using` 语句等价于以下 `try - finally` 代码结构：
```cs
ResourceType resource = new ResourceType();
try
{
    // 使用资源的代码
}
finally
{
    if (resource != null)
        ((IDisposable)resource).Dispose();
}
```

# 使用场景
一般配合内存占用比较大，有读写操作使用
- **文件操作**：在使用 `FileStream`、`StreamReader`、`StreamWriter` 等类进行文件读写时，这些类都实现了 `IDisposable` 接口，使用 `using` 语句可以确保文件句柄在使用完毕后被正确关闭。
- **数据库连接**：在使用 `SqlConnection`、`MySqlConnection` 等数据库连接类时，使用 `using` 语句可以确保数据库连接在使用完毕后被关闭和释放。
- **图形资源**：在使用 `Bitmap`、`Pen`、`Brush` 等图形资源类时，使用 `using` 语句可以确保图形资源被正确释放。

## 文件操作示例
```cs
using System;
using System.IO;

class Program
{
    static void Main()
    {
        // 使用 using 语句创建 StreamWriter 对象
        using (StreamWriter writer = new StreamWriter("test.txt"))
        {
            // 向文件中写入内容
            writer.WriteLine("Hello, World!");
        } // 代码块结束，自动调用 writer.Dispose() 方法，关闭文件

        // 使用 using 语句创建 StreamReader 对象
        using (StreamReader reader = new StreamReader("test.txt"))
        {
            // 从文件中读取内容
            string line = reader.ReadLine();
            Console.WriteLine(line);
        } // 代码块结束，自动调用 reader.Dispose() 方法，关闭文件
    }
}
```


## 数据库连接示例（假设使用 SQL Server）
```cs
using System;
using System.Data.SqlClient;

class Program
{
    static void Main()
    {
        string connectionString = "Data Source=YOUR_SERVER;Initial Catalog=YOUR_DATABASE;User ID=YOUR_USER;Password=YOUR_PASSWORD";

        // 使用 using 语句创建 SqlConnection 对象
        using (SqlConnection connection = new SqlConnection(connectionString))
        {
            try
            {
                // 打开数据库连接
                connection.Open();
                Console.WriteLine("数据库连接已打开。");

                // 执行 SQL 查询等操作
                string query = "SELECT * FROM YourTable";
                using (SqlCommand command = new SqlCommand(query, connection))
                {
                    using (SqlDataReader reader = command.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            // 处理查询结果
                            Console.WriteLine(reader[0].ToString());
                        }
                    } // 代码块结束，自动调用 reader.Dispose() 方法，关闭数据读取器
                } // 代码块结束，自动调用 command.Dispose() 方法，释放命令对象
            }
            catch (Exception ex)
            {
                Console.WriteLine($"数据库操作出错: {ex.Message}");
            }
        } // 代码块结束，自动调用 connection.Dispose() 方法，关闭数据库连接
    }
}
```