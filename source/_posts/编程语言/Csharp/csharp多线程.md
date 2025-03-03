---
title: csharp多线程
date: 2025-03-03 20:20:16
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---




[c#多线程总结（纯干货） - 一个大西瓜咚咚咚 - 博客园](https://www.cnblogs.com/wyt007/p/9486752.html)

# 线程基础
## 创建线程
`Thread t = new Thread(PrintNumbers)`
```cs
static void Main(string[] args)
{
    Thread t = new Thread(PrintNumbers);
    t.Start();//线程开始执行
    PrintNumbers();
    Console.ReadKey();
}

static void PrintNumbers()
{
    Console.WriteLine("Starting...");
    for (int i = 1; i < 10; i++)
    {
        Console.WriteLine(i);
    }
}
```


## 暂停线程
`Thread.Sleep`
工作原理：
当线程处于休眠状态时，它会占用尽可能少的 CPU 时间。结果我们会发现通常后运行的 PrintNumbers 方法中的代码会比独立线程中的 PrintNumbersWithDelay暂停2S 方法中的代码先执行。
```cs
class Program
{
    static void Main(string[] args)
    {
        Thread t = new Thread(PrintNumbersWithDelay);
        t.Start();
        PrintNumbers();
        Console.ReadKey();
    }

    static void PrintNumbers()
    {
        Console.WriteLine("Starting...");
        for (int i = 1; i < 10; i++)
        {
            Console.WriteLine(i);
        }
    }

    static void PrintNumbersWithDelay()
    {
        Console.WriteLine("Starting...");
        for (int i = 1; i < 10; i++)
        {
            Thread.Sleep(TimeSpan.FromSeconds(2));//暂停2S
            Console.WriteLine(i);
        }
    }
}
```


## 线程等待
`t.Join()`
工作原理：
但我们在主程序中调用了 t.Join 方法，该方法允许我们等待直到线程 t 完成。当线程 t 完成时，主程序会继续运行。
借助该技术可以实现在两个线程间同步执行步骤。第一个线程会等待另一个线程完成后再继续执行。第一个线程等待时是处于阻塞状态 (正如暂停线程中调用 Thread.Sleep 方法一样)。
```cs
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Starting program...");
        Thread t = new Thread(PrintNumbersWithDelay);
        t.Start();
        t.Join();
        Console.WriteLine("Thread completed");
    }

    static void PrintNumbersWithDelay()
    {
        Console.WriteLine("Starting...");
        for (int i = 1; i < 10; i++)
        {
            Thread.Sleep(TimeSpan.FromSeconds(2));
            Console.WriteLine(i);
        }
    }
}
```


## 终止线程
` t.Abort()` 弃用了
工作原理：
当主程序和单独的数字打印线程运行时，我们等待 6 秒后对线程调用了 t.Abort 方法。
这给线程注入了 ThreadAbortException 方法，导致线程被终结。这非常危险，因为该异常可以在任何时刻发生并可能彻底摧毁应用程序。
另外，使用该技术也不一定总能终止线程。目标线程可以通过处理该异常并调用 Thread.ResetAbort 方法来拒绝被终止。因此**并不推荐使用 Abort 方法来关闭线程。** 可优先使用一些其他方法，比如提供一个 CancellationToken 方法来取消线程的执行。
```cs
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Starting program...");
        Thread t = new Thread(PrintNumbersWithDelay);
        t.Start();
        Thread.Sleep(TimeSpan.FromSeconds(6));
        t.Abort();
        Console.WriteLine("A thread has been aborted");
    }

    static void PrintNumbersWithDelay()
    {
        Console.WriteLine("Starting...");
        for (int i = 1; i < 10; i++)
        {
            Thread.Sleep(TimeSpan.FromSeconds(2));
            Console.WriteLine(i);
        }
    }
}
```


## 监测线程状态
`Thread 对象的 ThreadState`:
- Unstarted：线程已被创建，但尚未调用 `Start` 方法启动。
- Running ：线程正在执行。当调用 `Start` 方法后，线程进入此状态并开始执行其关联的方法。
- WaitSleepJoin：线程正在等待、休眠或被阻塞。当线程调用 `Sleep`、`Wait` 或 `Join` 方法时，会进入此状态。
- Suspended（不推荐使用）：线程已被挂起。可以使用 `Suspend` 方法将线程挂起，但该方法已被弃用，因为它可能会导致死锁等问题。
- Stopped：线程已完成执行或被终止，处于停止状态。
- Aborted：线程已被调用 `Abort` 方法终止，但尚未完全停止执行。
- 。。。。。。。。。
工作原理：
当主程序启动时定义了两个不同的线程。一个将被终止，另一个则会成功完成运行。
**线程状态位于 Thread 对象的 ThreadState 属性中。ThreadState 属性是一个 C# 枚举对象。**
刚开始线程状态为 ThreadState.Unstarted, 然后我们启动线程，并估计在一个周期为 30 次迭代的区间中，线程状态会从 ThreadState.Running 变为 ThreadState.WaitSleepJoin。  
请注意始终可以通过 `Thread.CurrentThread` 静态属性获得当前 Thread 对象。  
如果实际情况与以上不符，请增加迭代次数。终止第一个线程后，会看到现在该线程状态为 ThreadState.Aborted, 程序也有可能会打印出 ThreadState.AbortRequested 状态。这充分说明了同步两个线程的复杂性。请记住不要在程序中使用线程终止。我在这里使用它只是为了展示相应的线程状态。  
最后可以看到第二个线程 t2 成功完成并且状态为 ThreadState.Stopped。另外还有一些其他的线程状态，但是要么已经被弃用，要么没有我们实验过的几种状态有用。


```cs
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Starting program...");

        Thread t = new Thread(PrintNumbersWithStatus);
        Thread t2 = new Thread(DoNothing);

        // 线程 t 的初始状态
        Console.WriteLine(t.ThreadState.ToString());

        t2.Start();
        t.Start();

        // 循环 29 次，输出线程 t 的状态
        for (int i = 1; i < 30; i++)
        {
            Console.WriteLine(t.ThreadState.ToString());
        }

        // 主线程休眠 6 秒
        Thread.Sleep(TimeSpan.FromSeconds(6));

        // 终止线程 t
        t.Abort();
        Console.WriteLine("A thread has been aborted");

        // 线程 t 终止后的状态
        Console.WriteLine(t.ThreadState.ToString());
        // 线程 t2 的状态
        Console.WriteLine(t2.ThreadState.ToString());

        Console.ReadKey();
    }

    static void DoNothing()
    {
        Thread.Sleep(TimeSpan.FromSeconds(2));
    }

    static void PrintNumbersWithStatus()
    {
        Console.WriteLine("Starting...");
        // 输出当前线程的状态
        Console.WriteLine(Thread.CurrentThread.ThreadState.ToString());

        // 循环 9 次，每次打印一个数字，每次循环间隔 2 秒
        for (int i = 1; i < 10; i++)
        {
            Thread.Sleep(TimeSpan.FromSeconds(2));
            Console.WriteLine(i);
        }
    }
}
```


## 线程优先级
`Thread.CurrentThread.Priority`

工作原理：
当主程序启动时定义了两个不同的线程。
第一个线程优先级为 ThreadPriority.Highest, 即具有最高优先级。
第二个线程优先级为 ThreadPriority.Lowest, 即具有最低优先级。
我们先打印出主线程的优先级值，然后在所有可用的 CPU 核心上启动这两个线程。如果拥有一个 1 以上的计算核心，将在两秒钟内得到初步结果。最高优先级的线程通常会计算更多的迭代，但是两个值应该很接近。然而，如果有其他程序占用了所有的 CPU 核心运行负载，结果则会截然不同。  
为了模拟该情形，我们设置了 `ProcessorAffinity` 选项，让操作系统将所有的线程运行在单个 CPU 核心 (第一个核心) 上。现在结果完全不同，并且计算耗时将超过 2 秒钟。这是因为 CPU 核心大部分时间在运行高优先级的线程，只留给剩下的线程很少的时间来运行。  
请注意这是操作系统使用线程优先级的一个演示。通常你无需使用这种行为编写程序。


```cs
class Program
{
    static void Main(string[] args)
    {
        // 输出当前线程的优先级
        Console.WriteLine("Current thread priority: {0}", Thread.CurrentThread.Priority);
        Console.WriteLine("Running on all cores available");
        // 调用 RunThreads 方法启动线程, 程序将在所有可用核心上运行
        RunThreads();
        // 主线程休眠 2 秒
        Thread.Sleep(TimeSpan.FromSeconds(2));

        Console.WriteLine("Running on a single core");
        // 设置当前进程仅使用一个核心
        Process.GetCurrentProcess().ProcessorAffinity = new IntPtr(1);
        // 启动线程
        RunThreads();
    }

    // 启动两个线程并设置不同优先级的方法
    static void RunThreads()
    {
        var sample = new ThreadSample();

        var threadOne = new Thread(sample.CountNumbers);
        threadOne.Name = "ThreadOne";  // 线程命名
        
        var threadTwo = new Thread(sample.CountNumbers);
        threadTwo.Name = "ThreadTwo";

        // 设置第一个线程的优先级为最高
        threadOne.Priority = ThreadPriority.Highest;
        // 设置第二个线程的优先级为最低
        threadTwo.Priority = ThreadPriority.Lowest;

        threadOne.Start();
        threadTwo.Start();

        // 主线程休眠 2 秒
        Thread.Sleep(TimeSpan.FromSeconds(2));
        // 调用 Stop 方法停止线程计数
        sample.Stop();

        Console.ReadKey();
    }

    class ThreadSample
    {
        // 用于控制线程计数循环的标志
        private bool _isStopped = false;

        // 停止线程计数的方法
        public void Stop()
        {
            _isStopped = true;
        }

        // 线程执行的计数方法
        public void CountNumbers()
        {
            // 初始化计数器
            long counter = 0;

            // 只要未停止，就持续计数
            while (!_isStopped)
            {
                counter++;
            }

            // 输出线程名称、优先级和计数结果
            Console.WriteLine("{0} with {1,11} priority " +
                        "has a count = {2,13}", Thread.CurrentThread.Name,
                        Thread.CurrentThread.Priority,
                        counter.ToString("N0"));
        }
    }
}
```


## 前台线程和后台线程
`thread.IsBackground = true`
工作原理：
当主程序启动时定义了两个不同的线程。默认情况下，显式创建的线程是前台线程。通过手动的设置 threadTwo 对象的 IsBackground 属性为 ture 来创建一个后台线程(可以在主线程设置其它线程执行完毕的条件)。通过配置来实现第一个线程会比第二个线程先完成。然后运行程序。  
第一个线程完成后，程序结束并且后台线程被终结。这是前台线程与后台线程的主要区别：**进程会等待所有的前台线程完成后再结束工作，但是如果只剩下后台线程，则会直接结束工作。**
一个重要注意事项是：如果程序定义了一个不会完成的前台线程，主程序并不会正常结束。
```cs
class Program
{
    static void Main(string[] args)
    {
        var sampleForeground = new ThreadSample(10);
        var sampleBackground = new ThreadSample(20);

        var threadOne = new Thread(sampleForeground.CountNumbers);
        threadOne.Name = "ForegroundThread";
        var threadTwo = new Thread(sampleBackground.CountNumbers);
        threadTwo.Name = "BackgroundThread";
        threadTwo.IsBackground = true;

        threadOne.Start();
        threadTwo.Start();

        Console.ReadKey();
    }

    class ThreadSample
    {
        private readonly int _iterations;

        public ThreadSample(int iterations)
        {
            _iterations = iterations;
        }
        public void CountNumbers()
        {
            for (int i = 0; i < _iterations; i++)
            {
                Thread.Sleep(TimeSpan.FromSeconds(0.5));
                Console.WriteLine("{0} prints {1}", Thread.CurrentThread.Name, i);
            }
        }
    }
}
```


## 向线程传递参数
**通过实例字段间接传递参数**

**Thread.Start 方法传入object类型参数**
另一种传递数据的方式是使用 Thread.Start 方法。该方法会接收一个对象，并将该对象传递给线程。为了应用该方法，在线程中启动的方法必须接受 object 类型的单个参数。在创建 threadTwo 线程时演示了该方式。我们将 8 作为一个对象传递给了 Count 方法，然后 Count 方法被转换为整型。



**使用 Lambda 表达式捕获变量**
接下来的方式是使用 lambda 表达式。lambda 表达式定义了一个不属于任何类的方法。我们创建了一个方法，该方法使用需要的参数调用了另一个方法，并在另一个线程中运行该方法。当启动 threadThree 线程时，打印出了 12 个数字，这正是我们通过 lambda 表达式传递的数字。  

使用 lambda 表达式引用另一个 C# 对象的方式被称为{% post_link csharp的Lambda表达式#闭包与变量捕获 %}。当在 lambda 表达式中使用任何局部变量时，C# 会生成一个类，并将该变量作为该类的一个属性。所以实际上该方式与 threadOne 线程中使用的一样，但是我们无须定义该类，C# 编译器会自动帮我们实现。  
这可能会导致几个问题。例如，如果在多个 lambda 表达式中使用相同的变量，它们会共享该变量值。在前一个例子中演示了这种情况。当启动 threadFour 和 threadFive 线程时，它们都会打印 20, 因为在这两个线程启动之前变量被修改为 20。

```cs
using System;
using System.Threading;

class Program
{
    static void Main(string[] args)
    {
        // 指定迭代次数为 10
        var sample = new ThreadSample(10);

        // 创建第一个线程，执行 sample 实例的 CountNumbers 方法
        var threadOne = new Thread(sample.CountNumbers);
        threadOne.Name = "ThreadOne";
        threadOne.Start();
        // 主线程等待 threadOne 执行完毕
        threadOne.Join();

        Console.WriteLine("--------------------------");

        // 创建第二个线程，执行静态方法 Count
        var threadTwo = new Thread(Count);
        threadTwo.Name = "ThreadTwo";
        // 启动线程并传递参数 8
        threadTwo.Start(8);
        // 主线程等待 threadTwo 执行完毕
        threadTwo.Join();

        Console.WriteLine("--------------------------");

        // 创建第三个线程，使用 Lambda 表达式调用静态方法 CountNumbers
        var threadThree = new Thread(() => CountNumbers(12));
        threadThree.Name = "ThreadThree";
        
        threadThree.Start();
        // 主线程等待 threadThree 执行完毕
        threadThree.Join();
        Console.WriteLine("--------------------------");

        // 定义一个整数变量 i 并初始化为 10
        int i = 10;
        // 创建第四个线程，使用 Lambda 表达式调用静态方法 PrintNumber
        var threadFour = new Thread(() => PrintNumber(i));
        // 修改变量 i 的值为 20
        i = 20;
        // 创建第五个线程，使用 Lambda 表达式调用静态方法 PrintNumber
        var threadFive = new Thread(() => PrintNumber(i));
        threadFour.Start(); 
        threadFive.Start();
    }

    static void Count(object iterations)
    {
        CountNumbers((int)iterations);
    }

    // 静态方法，用于循环打印线程名称和迭代次数，每次循环间隔 0.5 秒
    static void CountNumbers(int iterations)
    {
        for (int i = 1; i <= iterations; i++)
        {
            Thread.Sleep(TimeSpan.FromSeconds(0.5));
            Console.WriteLine("{0} prints {1}", Thread.CurrentThread.Name, i);
        }
    }

    static void PrintNumber(int number)
    {
        Console.WriteLine(number);
    }

    class ThreadSample
    {
        // 迭代次数
        private readonly int _iterations;

        public ThreadSample(int iterations)
        {
            _iterations = iterations;
        }
        // 循环打印线程名称和迭代次数，每次循环间隔 0.5 秒
        public void CountNumbers()
        {
            for (int i = 1; i <= _iterations; i++)
            {
                Thread.Sleep(TimeSpan.FromSeconds(0.5));
                Console.WriteLine("{0} prints {1}", Thread.CurrentThread.Name, i);
            }
        }
    }
}
```


## 使用 C# 中的 lock 关键字
**lock 的作用**
- **线程同步**：确保在多线程环境下，同一时间只有一个线程可以访问临界区（被锁保护的代码块）。
- **解决竞态条件**：防止多个线程同时修改共享资源导致数据不一致。

**基本语法**
```cs
private readonly object _lockObj = new object(); // 锁对象必须是引用类型

void ThreadSafeMethod()
{
    lock (_lockObj)
    {
        // 临界区代码（同一时间仅一个线程可执行）
    }
}
```

**底层原理**
**基于 Monitor 类**：`lock` 是语法糖，编译后等价于：
```cs
bool lockTaken = false;
try
{
    Monitor.Enter(_lockObj, ref lockTaken);
    // 临界区代码
}
finally
{
    if (lockTaken)
        Monitor.Exit(_lockObj);
}
```

**不安全的计数器（Counter 类）** 与 **安全的计数器（CounterWithLock 类）**
```cs
class Program
{
    static void Main(string[] args)
    {
        // 不正确的计数器
        Console.WriteLine("Incorrect counter");

        var c = new Counter();

        var t1 = new Thread(() => TestCounter(c));
        var t2 = new Thread(() => TestCounter(c));
        var t3 = new Thread(() => TestCounter(c));
        t1.Start();
        t2.Start();
        t3.Start();
        t1.Join();
        t2.Join();
        t3.Join();

        Console.WriteLine("Total count: {0}",c.Count);
        Console.WriteLine("--------------------------");

        // 正确的计数器
        Console.WriteLine("Correct counter");

        var c1 = new CounterWithLock();

        t1 = new Thread(() => TestCounter(c1));
        t2 = new Thread(() => TestCounter(c1));
        t3 = new Thread(() => TestCounter(c1));
        t1.Start();
        t2.Start();
        t3.Start();
        t1.Join();
        t2.Join();
        t3.Join();
        Console.WriteLine("Total count: {0}", c1.Count);

        Console.ReadKey();

    }

    static void TestCounter(CounterBase c)
    {
        for (int i = 0; i < 100000; i++)
        {
            c.Increment();
            c.Decrement();
        }
    }

    // 没加锁的计数器
    class Counter : CounterBase
    {
        public int Count { get; private set; }

        public override void Increment()
        {
            Count++;
        }

        public override void Decrement()
        {
            Count--;
        }
    }

    // 加了锁的计数器
    class CounterWithLock : CounterBase
    {
        private readonly object _syncRoot = new Object();

        public int Count { get; private set; }

        public override void Increment()
        {
            lock (_syncRoot)
            {
                Count++;
            }
        }

        public override void Decrement()
        {
            lock (_syncRoot)
            {
                Count--;
            }
        }
    }

    abstract class CounterBase
    {
        public abstract void Increment();

        public abstract void Decrement();
    }
}
```
**lock相关注意：**
**(1) 锁对象的选择**
- **锁对象必须是引用类型**：值类型会被装箱，每次装箱生成新对象，导致锁失效。
- **私有且只读**：防止锁对象被修改。推荐模式：`private readonly object _lockObj = new object(); // 专用锁对象`
**(2) 临界区最小化**
- **锁的粒度要细**：仅保护真正需要同步的资源，减少锁持有时间
```cs
// 错误：锁范围过大
lock (_lockObj)
{
    ReadData();
    ProcessData(); // 长时间计算
    WriteData();
}

// 正确：仅锁共享数据操作
var data = ReadData();
var result = ProcessData(data); // 无锁计算
lock (_lockObj)
{
    WriteData(result);
}

```
**(3) 避免嵌套锁**
- **死锁风险**：若多个锁以不同顺序嵌套获取，可能引发死锁。
```cs
// 线程1
lock (A)
{
    lock (B) { ... }
}

// 线程2
lock (B) // 若线程1已持A等B，线程2持B等A → 死锁
{
    lock (A) { ... }
}
```

## 使用Monitor类锁定资源*
**代码行为分析**

**第一部分（使用 TryEnter 避免死锁）**
1. **线程1**执行 `LockTooMuch`：获取 `lock1` → 休眠 1 秒 → 尝试获取 `lock2`。
2. **主线程**：获取 `lock2` → 休眠 1 秒 → 尝试通过 `TryEnter` 获取 `lock1`（超时 5 秒）。


**可能的执行结果**：
- **线程1**持有 `lock1`，等待 `lock2`；**主线程**持有 `lock2`，等待 `lock1` → **死锁条件成立**。
- 但由于主线程使用 `TryEnter`，5 秒后会超时并释放 `lock2`，随后线程1最终能获取 `lock2`，继续执行。
- 输出：`Timeout acquiring a resource!`。

**第二部分（直接死锁）**
1. **线程2**再次执行 `LockTooMuch`（同上）。
2. **主线程**再次尝试以 `lock2 → lock1` 顺序获取锁。  
    **结果**：主线程持有 `lock2`，等待 `lock1`；线程2持有 `lock1`，等待 `lock2` → **死锁**，程序永久阻塞。
```cs
class Program
{
    static void Main(string[] args)
    {
        // 创建两个锁对象
        object lock1 = new object();
        object lock2 = new object();

        // 启动第一个线程：尝试以 lock1 → lock2 顺序获取锁
        new Thread(() => LockTooMuch(lock1, lock2)).Start();

        // 主线程尝试以 lock2 → lock1 顺序获取锁
        lock (lock2) // 主线程获取 lock2
        {
            Thread.Sleep(1000); // 等待 1 秒（让第一个线程先获取 lock1）
            Console.WriteLine("Monitor.TryEnter 允许不阻塞，在超时后返回 false");
            
            // 尝试在 5 秒内获取 lock1（非阻塞式）
            if (Monitor.TryEnter(lock1, TimeSpan.FromSeconds(5)))
            {
                Console.WriteLine("成功获取受保护资源！");
                Monitor.Exit(lock1); // 释放锁（实际代码中应使用 finally 确保释放）
            }
            else
            {
                Console.WriteLine("获取资源超时！"); 
                // 此处未获取到 lock1，但 lock2 会在当前代码块结束后自动释放
            }
        }

        // 第二次启动线程：再次尝试 lock1 → lock2 顺序
        new Thread(() => LockTooMuch(lock1, lock2)).Start();

        Console.WriteLine("----------------------------------");
        
        // 主线程再次尝试以 lock2 → lock1 顺序获取锁（此次无超时机制）
        lock (lock2) // 获取 lock2
        {
            Console.WriteLine("这将引发死锁！");
            Thread.Sleep(1000); 
            
            // 尝试获取 lock1（此时第一个线程可能持有 lock1 并等待 lock2）
            lock (lock1) // 阻塞在此处，无法继续执行
            {
                Console.WriteLine("成功获取受保护资源！"); // 永远不会执行
            }
        }

        Console.ReadKey();
    }

    static void LockTooMuch(object lock1, object lock2)
    {
        lock (lock1) // 先获取 lock1
        {
            Thread.Sleep(1000); // 等待 1 秒（让主线程有机会获取 lock2）
            lock (lock2) // 再尝试获取 lock2（可能导致死锁）
            { 
                // 临界区代码
            }
        }
    }
}
```

## **多线程异常处理**
**异常作用域规则**
- **线程是独立执行路径**：每个线程拥有自己的异常处理上下文。
- **主线程无法直接捕获子线程的异常**：`try-catch` 在父线程中无法捕获子线程未处理的异常。
```cs
class Program
{
    static void Main(string[] args)
    {
        // 第一个线程：内部捕获异常
        var t = new Thread(FaultyThread);
        t.Start();    // 启动线程
        t.Join();     // 等待线程结束（此时异常已被内部处理，不会传播到主线程）

        // 第二个线程：未处理异常
        try
        {
            t = new Thread(BadFaultyThread);
            t.Start(); // 启动线程（但未调用 Join）
            // 主线程不会等待此线程结束，try-catch 无法捕获子线程异常！
        }
        catch (Exception ex)
        {
            // 此处永远不会执行，因为子线程的异常不会传递到主线程
            Console.WriteLine("We won't get here!");
        }
    }

    // 方法：内部捕获异常
    static void FaultyThread()
    {
        try
        {
            Console.WriteLine("Starting a faulty thread...");
            Thread.Sleep(TimeSpan.FromSeconds(1));
            throw new Exception("Boom!"); // 抛出异常
        }
        catch (Exception ex)
        {
            // 异常在此处被捕获并处理
            Console.WriteLine("Exception handled: {0}", ex.Message);
        }
    }

    // 方法：未处理异常
    static void BadFaultyThread()
    {
        Console.WriteLine("Starting a faulty thread...");
        Thread.Sleep(TimeSpan.FromSeconds(2));
        throw new Exception("Boom!"); // 抛出未捕获的异常
    }
}
```

**正确捕获子线程异常的方法**
**(1) 在线程方法内部处理异常**
```cs
void SafeThreadMethod()
{
    try { /* 代码 */ }
    catch (Exception ex) { /* 记录或处理异常 */ }
}
```

**(2) 使用 `Task` 替代 `Thread`**
- **优势**：通过 `Task.Exception` 属性聚合所有异常（`AggregateException`）。
```cs
Task.Run(() => 
{
    // 可能抛异常的代码
}).ContinueWith(t => 
{
    if (t.Exception != null)
    {
        Console.WriteLine("捕获异常: " + t.Exception.InnerException.Message);
    }
}, TaskContinuationOptions.OnlyOnFaulted);
```

**(3) 全局异常捕获（不推荐）**
```cs
AppDomain.CurrentDomain.UnhandledException += (sender, e) => 
{
    Console.WriteLine("全局捕获: " + e.ExceptionObject.ToString());
};
```

