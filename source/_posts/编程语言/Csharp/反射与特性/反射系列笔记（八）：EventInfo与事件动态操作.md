---
title: 反射系列笔记（八）：EventInfo与事件动态操作
date: 2026-04-06 14:05:56
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


# C# 反射系列笔记（八）：EventInfo与事件动态操作

> 目标：全面掌握事件的反射操作，理解事件本质（委托多播），学会动态订阅/取消订阅事件，实现事件总线

---

## 一、事件的本质：委托的“安全包装”

### 事件是什么？

```csharp
// 你写的代码
public class Button
{
    public event EventHandler Click;
}

// 编译器实际生成的代码（简化版）
public class Button
{
    private EventHandler _click;  // 委托字段
    
    public event EventHandler Click
    {
        add { _click = (EventHandler)Delegate.Combine(_click, value); }
        remove { _click = (EventHandler)Delegate.Remove(_click, value); }
    }
    
    // 触发事件的方法（通常为 protected virtual）
    protected virtual void OnClick(EventArgs e)
    {
        _click?.Invoke(this, e);
    }
}
```

**核心理解：**
- **事件不是委托**，而是委托的 **add/remove 访问器**
- `EventInfo` 封装了 `add` 和 `remove` 两个方法
- 事件只能从声明类内部触发，外部只能订阅/取消订阅

### 验证事件本质

```csharp
public class Demo
{
    public event EventHandler MyEvent;
}

Type t = typeof(Demo);

// 查看事件相关的方法
MethodInfo[] methods = t.GetMethods(BindingFlags.Public | BindingFlags.Instance | BindingFlags.NonPublic);
foreach (var m in methods.Where(m => m.IsSpecialName))
{
    Console.WriteLine($"特殊方法: {m.Name}");
}
// 输出：
// 特殊方法: add_MyEvent
// 特殊方法: remove_MyEvent

// EventInfo 暴露了 add/remove 方法
EventInfo evt = t.GetEvent("MyEvent");
MethodInfo addMethod = evt.AddMethod;    // add_MyEvent
MethodInfo removeMethod = evt.RemoveMethod;  // remove_MyEvent
Console.WriteLine($"Add: {addMethod?.Name}, Remove: {removeMethod?.Name}");
```

---

## 二、EventInfo 核心属性

### 基本信息

```csharp
public class EventDemo
{
    public event EventHandler PublicEvent;
    private event EventHandler PrivateEvent;
    public static event EventHandler StaticEvent;
    public event EventHandler<CustomEventArgs> GenericEvent;
}

public class CustomEventArgs : EventArgs { }

Type t = typeof(EventDemo);

foreach (EventInfo evt in t.GetEvents(BindingFlags.Public | BindingFlags.NonPublic | 
                                       BindingFlags.Instance | BindingFlags.Static))
{
    Console.WriteLine($"事件名: {evt.Name}");
    Console.WriteLine($"  委托类型: {evt.EventHandlerType.Name}");
    Console.WriteLine($"  是否为 public: {evt.AddMethod?.IsPublic ?? false}");
    Console.WriteLine($"  是否为 static: {evt.AddMethod?.IsStatic ?? false}");
    Console.WriteLine($"  是否为虚方法: {evt.AddMethod?.IsVirtual ?? false}");
    Console.WriteLine();
}
```

### 事件的 add/remove 方法

```csharp
public class Button
{
    public event EventHandler Click;
}

EventInfo evt = typeof(Button).GetEvent("Click");

MethodInfo addMethod = evt.GetAddMethod();        // 获取 add 访问器
MethodInfo removeMethod = evt.GetRemoveMethod();  // 获取 remove 访问器
MethodInfo raiseMethod = evt.GetRaiseMethod();    // 获取 raise 访问器（罕见）

Console.WriteLine($"Add 方法是否为 public: {addMethod?.IsPublic}");
Console.WriteLine($"Remove 方法是否为 public: {removeMethod?.IsPublic}");

// 检查是否有自定义的 add/remove 实现  DeclaringType获取声明此成员的类。(子类实例父类声明时获取父类
bool isCustomImplemented = addMethod.IsVirtual && addMethod.DeclaringType == evt.DeclaringType;
```

---

## 三、事件的动态订阅与取消订阅

### 基础订阅

```csharp
public class Button
{
    public event EventHandler Click;
    
    public void SimulateClick()
    {
        Console.WriteLine("按钮被点击");
        Click?.Invoke(this, EventArgs.Empty);
    }
}

public class Program
{
    // 事件处理方法
    private static void OnClick(object sender, EventArgs e)
    {
        Console.WriteLine($"事件处理: {sender.GetType().Name} 被点击");
    }
    
    public static void Main()
    {
        Button btn = new Button();
        Type btnType = typeof(Button);
        
        // 获取事件
        EventInfo clickEvent = btnType.GetEvent("Click");
        
        // 创建委托
        EventHandler handler = new EventHandler(OnClick);
        
        // 订阅事件
        clickEvent.AddEventHandler(btn, handler);
        
        // 测试触发
        btn.SimulateClick();  // 输出: 按钮被点击 \n 事件处理: Button 被点击
        
        // 取消订阅
        clickEvent.RemoveEventHandler(btn, handler);
        
        btn.SimulateClick();  // 输出: 按钮被点击（没有事件处理输出）
    }
}
```

### 泛型事件（`EventHandler<T>`）

```csharp
public class ValueChangedEventArgs : EventArgs
{
    public int OldValue { get; set; }
    public int NewValue { get; set; }
}

public class Counter
{
    private int _value;
    public event EventHandler<ValueChangedEventArgs> ValueChanged;
    
    public int Value
    {
        get => _value;
        set
        {
            if (_value != value)
            {
                var old = _value;
                _value = value;
                ValueChanged?.Invoke(this, new ValueChangedEventArgs { OldValue = old, NewValue = value });
            }
        }
    }
}

// 动态订阅泛型事件
public class GenericEventSubscriber
{
    public static void Subscribe(Counter counter)
    {
        Type counterType = typeof(Counter);
        EventInfo valueChangedEvent = counterType.GetEvent("ValueChanged");
        
        // 创建泛型委托
        Type handlerType = typeof(EventHandler<>).MakeGenericType(typeof(ValueChangedEventArgs));
        
        // 创建处理方法
        MethodInfo handlerMethod = typeof(GenericEventSubscriber).GetMethod("OnValueChanged", 
            BindingFlags.NonPublic | BindingFlags.Static);
        
        Delegate handler = Delegate.CreateDelegate(handlerType, null, handlerMethod);
        
        // 订阅
        valueChangedEvent.AddEventHandler(counter, handler);
    }
    
    private static void OnValueChanged(object sender, ValueChangedEventArgs e)
    {
        Console.WriteLine($"值变化: {e.OldValue} -> {e.NewValue}");
    }
}

// 使用
Counter counter = new Counter();
GenericEventSubscriber.Subscribe(counter);
counter.Value = 10;  // 输出: 值变化: 0 -> 10
counter.Value = 20;  // 输出: 值变化: 10 -> 20
```

### 静态事件

```csharp
using System;
using System.Reflection;

public class GlobalManager
{
    // 定义一个静态事件
    public static event EventHandler<string> OnGlobalMessage;

    public static void Trigger(string msg)
    {
        Console.WriteLine($"[系统] 正在触发静态事件...");
        OnGlobalMessage?.Invoke(null, msg);
    }
}

public class ReflectiveSubscriber
{
    public static void Run()
    {
        // 1. 获取目标类型的 Type 对象
        Type targetType = typeof(GlobalManager);

        // 2. 找到名为 "OnGlobalMessage" 的静态事件
        // 注意：静态成员需要 BindingFlags.Static
        EventInfo eventInfo = targetType.GetEvent("OnGlobalMessage", 
            BindingFlags.Public | BindingFlags.Static);

        if (eventInfo != null)
        {
            // 3. 获取我们要绑定的方法信息 (Handler)
            MethodInfo handlerMethod = typeof(ReflectiveSubscriber)
                .GetMethod(nameof(MyHandler), BindingFlags.NonPublic | BindingFlags.Static);

            // 4. 将 MethodInfo 转换为该事件要求的委托类型
            Delegate handler = Delegate.CreateDelegate(eventInfo.EventHandlerType, handlerMethod);

            // 5. 订阅事件
            // 对于静态事件，第一个参数（target）必须传 null
            eventInfo.AddEventHandler(null, handler);

            Console.WriteLine("成功通过反射订阅了静态事件！");
        }

        // 触发测试
        GlobalManager.Trigger("Hello Reflection!");
    }

    // 事件处理函数（必须匹配 EventHandler<string> 的签名）
    private static void MyHandler(object sender, string message)
    {
        Console.WriteLine($"[订阅者] 收到消息: {message}");
    }
}


// 使用
ReflectiveSubscriber.Run();
// 成功通过反射订阅了静态事件！
// [系统] 正在触发静态事件...
// [订阅者] 收到消息: Hello Reflection!
```

### 私有事件

获取私有事件的add方法，强行订阅
如果事件是私有的，那么编译器生成的 `add` 方法默认也是私有的。普通的 `GetAddMethod()` 会返回 `null`，必须传入 `true` 才能拿到这个“秘密通道”。


```csharp
public class SecretEventSource
{
    private event Action<string> InternalEvent;
    
    public void Trigger()
    {
        InternalEvent?.Invoke("秘密消息");
    }
}

// 订阅私有事件（需要 BindingFlags）
Type sourceType = typeof(SecretEventSource);
object source = Activator.CreateInstance(sourceType);

EventInfo privateEvent = sourceType.GetEvent("InternalEvent", 
    BindingFlags.NonPublic | BindingFlags.Instance);

if (privateEvent != null)
{
    Action<string> handler = (msg) => Console.WriteLine($"收到: {msg}");
    
    // 私有事件的 add 方法也是非公开的
    MethodInfo addMethod = privateEvent.GetAddMethod(true);  // true = 非公开访问器
    addMethod.Invoke(source, new object[] { handler });
    
    // 触发事件
    MethodInfo triggerMethod = sourceType.GetMethod("Trigger");
    triggerMethod.Invoke(source, null);  // 输出: 收到: 秘密消息
}
```

---

## 四、事件的完整元数据操作

### 获取事件的委托类型信息

```csharp
public class MultiEventDemo
{
    public event Action<int> IntEvent;
    public event Func<string, bool> PredicateEvent;
    public event EventHandler<EventArgs> StandardEvent;
}

Type t = typeof(MultiEventDemo);

foreach (EventInfo evt in t.GetEvents())
{
    Type delegateType = evt.EventHandlerType;
    Console.WriteLine($"事件: {evt.Name}");
    Console.WriteLine($"  委托类型: {delegateType.Name}");
    
    // 获取委托的 Invoke 方法
    MethodInfo invokeMethod = delegateType.GetMethod("Invoke");
    Console.WriteLine($"  返回类型: {invokeMethod.ReturnType.Name}");
    
    // 获取委托的参数
    ParameterInfo[] parameters = invokeMethod.GetParameters();
    Console.WriteLine($"  参数: {string.Join(", ", parameters.Select(p => $"{p.ParameterType.Name} {p.Name}"))}");
    Console.WriteLine();
}
```

### 判断事件的特性

```csharp
public class AttributedEvents
{
    [Obsolete("请使用 NewEvent")]
    public event EventHandler OldEvent;
    
    public event EventHandler NewEvent;
    
    // 这个特性挂在自动生成的后台字段上，而不是挂在事件本身上。
    [field: NonSerialized]
    public event EventHandler NonSerializedEvent;
}

Type t = typeof(AttributedEvents);

foreach (EventInfo evt in t.GetEvents())
{
    Console.WriteLine($"事件: {evt.Name}");
    
    // 检查特性
    var obsoleteAttr = evt.GetCustomAttribute<ObsoleteAttribute>();
    if (obsoleteAttr != null)
    {
        Console.WriteLine($"  已过时: {obsoleteAttr.Message}");
    }
    
    // 检查事件上的特性
    var nonSerializedAttr = evt.GetCustomAttribute<NonSerializedAttribute>();
    if (nonSerializedAttr != null)
    {
        Console.WriteLine($"  不可序列化");
    }
    
    // 2. 检查背后的字段特性 (修复方案)
// 注意：后台字段通常是同名的私有实例字段
    var field = t.GetField(evt.Name, BindingFlags.NonPublic | BindingFlags.Instance);
    nonSerializedAttr = field?.GetCustomAttribute<NonSerializedAttribute>();
    if (nonSerializedAttr != null)
    {
        Console.WriteLine($"  不可序列化字段 (从字段找到)");
    }
}
```

---

## 五、自定义事件访问器（高级）

### 理解自定义 add/remove

```csharp
public class CustomEventClass
{
    private EventHandler _handlers;
    
    // 自定义事件访问器
    public event EventHandler CustomEvent
    {
        add
        {
            Console.WriteLine($"添加处理器: {value.Method.Name}");
            _handlers = (EventHandler)Delegate.Combine(_handlers, value);
        }
        remove
        {
            Console.WriteLine($"移除处理器: {value.Method.Name}");
            _handlers = (EventHandler)Delegate.Remove(_handlers, value);
        }
    }
    
    public void Raise()
    {
        _handlers?.Invoke(this, EventArgs.Empty);
    }
}

// 反射调用自定义事件
Type t = typeof(CustomEventClass);
object obj = Activator.CreateInstance(t);

EventInfo customEvent = t.GetEvent("CustomEvent");

// 订阅（会触发自定义 add 逻辑）
EventHandler handler = (s, e) => Console.WriteLine("事件触发");
customEvent.AddEventHandler(obj, handler);  // 输出: 添加处理器: <Main>b__0_0

// 取消订阅（会触发自定义 remove 逻辑）
customEvent.RemoveEventHandler(obj, handler);  // 输出: 移除处理器: <Main>b__0_0
```

### 通过 MethodInfo 直接调用 add/remove

```csharp
EventInfo evt = typeof(CustomEventClass).GetEvent("CustomEvent");
object obj = Activator.CreateInstance(typeof(CustomEventClass));

// 方式1：EventInfo 方法
EventHandler handler = (s, e) => Console.WriteLine("Hello");
evt.AddEventHandler(obj, handler);

// 方式2：直接调用 add 方法
MethodInfo addMethod = evt.GetAddMethod();
addMethod.Invoke(obj, new object[] { handler });

// 方式3：直接调用 remove 方法
MethodInfo removeMethod = evt.GetRemoveMethod();
removeMethod.Invoke(obj, new object[] { handler });
```

---

## 六、事件总线实现（实战）

### 简单事件总线

```csharp
/// <summary>
/// 简单的事件总线：实现对象之间的松耦合通信
/// </summary>
public class EventBus
{
    // 存储结构：Key 为事件的 Type，Value 为订阅该事件的所有回调函数（委托）列表
    private readonly Dictionary<Type, List<Delegate>> _handlers = new();
    
    /// <summary>
    /// 订阅事件：注册一个当特定类型事件发生时的处理函数
    /// </summary>
    /// <typeparam name="TEvent">事件类型（必须是引用类型）</typeparam>
    /// <param name="handler">处理逻辑回调</param>
    public void Subscribe<TEvent>(Action<TEvent> handler) where TEvent : class
    {
        Type eventType = typeof(TEvent);
        // 使用 lock 确保多线程环境下向字典添加处理程序时的线程安全
        lock (_handlers)
        {
            if (!_handlers.ContainsKey(eventType))
                _handlers[eventType] = new List<Delegate>();
            
            _handlers[eventType].Add(handler);
        }
    }
    
    /// <summary>
    /// 取消订阅：移除之前注册的处理函数，防止内存泄漏
    /// </summary>
    public void Unsubscribe<TEvent>(Action<TEvent> handler) where TEvent : class
    {
        Type eventType = typeof(TEvent);
        lock (_handlers)
        {
            if (_handlers.TryGetValue(eventType, out var handlers))
            {
                // Delegate 内部重写了 Equals，可以准确找到匹配的方法引用
                handlers.Remove(handler);
                // 如果该事件类型下没有订阅者了，则清理 Key，释放内存
                if (handlers.Count == 0)
                    _handlers.Remove(eventType);
            }
        }
    }
    
    /// <summary>
    /// 发布事件（泛型版）：触发所有注册了 TEvent 类型的处理函数
    /// </summary>
    public void Publish<TEvent>(TEvent @event) where TEvent : class
    {
        Type eventType = typeof(TEvent);
        lock (_handlers)
        {
            if (_handlers.TryGetValue(eventType, out var handlers))
            {
                // 使用 .ToList() 创建副本进行遍历
                // 理由：防止在执行 handler 时，内部逻辑又调用了 Unsubscribe 导致遍历集合崩溃
                foreach (var handler in handlers.ToList())
                {
                    // 使用 DynamicInvoke 兼容不同的 Action<T> 委托
                    handler.DynamicInvoke(@event);
                }
            }
        }
    }
    
    /// <summary>
    /// 发布事件（反射版本）：支持在运行时仅通过 object 实例触发事件
    /// 场景：当你有一个 object 列表，且不知道它们的具体泛型类型时非常有用
    /// </summary>
    public void Publish(object @event)
    {
        // 动态获取运行时的真实类型，而不是编译时的 object 类型
        Type eventType = @event.GetType();
        lock (_handlers)
        {
            if (_handlers.TryGetValue(eventType, out var handlers))
            {
                foreach (var handler in handlers.ToList())
                {
                    // 同样通过动态调用来分发事件
                    handler.DynamicInvoke(@event);
                }
            }
        }
    }
}

// 使用
public class UserLoggedInEvent
{
    public string UserName { get; set; }
    public DateTime Time { get; set; }
}

var bus = new EventBus();

// 订阅
bus.Subscribe<UserLoggedInEvent>(e => 
{
    Console.WriteLine($"用户登录: {e.UserName} at {e.Time}");
});

// 发布
bus.Publish(new UserLoggedInEvent { UserName = "张三", Time = DateTime.Now });
```

### 反射驱动的事件总线（自动发现）

```csharp
public class EventBusReflection
{
    private readonly Dictionary<Type, List<object>> _subscribers = new();
    
    /// <summary>
    /// 自动注册对象中所有带 SubscribeAttribute 的方法
    /// </summary>
    public void Register(object subscriber)
    {
        Type type = subscriber.GetType();
        var methods = type.GetMethods(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
        
        foreach (var method in methods)
        {
            var attr = method.GetCustomAttribute<SubscribeAttribute>();
            if (attr == null) continue;
            
            // 检查方法参数（必须是单个参数）
            var parameters = method.GetParameters();
            if (parameters.Length != 1)
            {
                Console.WriteLine($"警告: {method.Name} 参数数量不为1，跳过");
                continue;
            }
            
            Type eventType = parameters[0].ParameterType;
            
            // 创建委托
            Type delegateType = typeof(Action<>).MakeGenericType(eventType);
            Delegate handler = Delegate.CreateDelegate(delegateType, subscriber, method);
            
            // 存储
            lock (_subscribers)
            {
                if (!_subscribers.ContainsKey(eventType))
                    _subscribers[eventType] = new List<object>();
                
                _subscribers[eventType].Add(handler);
            }
        }
    }
    
    public void Unregister(object subscriber)
    {
        // 简化版本：重新构建所有订阅（或复杂引用追踪）
        // 实际实现需要追踪每个订阅者和委托的关系
    }
    
    public void Publish(object @event)
    {
        Type eventType = @event.GetType();
        lock (_subscribers)
        {
            if (_subscribers.TryGetValue(eventType, out var handlers))
            {
                foreach (var handler in handlers.ToList())
                {
                    ((Delegate)handler).DynamicInvoke(@event);
                }
            }
        }
    }
}

[AttributeUsage(AttributeTargets.Method)]
public class SubscribeAttribute : Attribute { }

// 使用
public class NotificationService
{
    [Subscribe]
    public void OnUserLoggedIn(UserLoggedInEvent e)
    {
        Console.WriteLine($"通知: 用户 {e.UserName} 登录了");
    }
    
    [Subscribe]
    private void OnAnyEvent(object e)
    {
        Console.WriteLine($"收到事件: {e.GetType().Name}");
    }
}

var busReflection = new EventBusReflection();
var service = new NotificationService();
busReflection.Register(service);

busReflection.Publish(new UserLoggedInEvent { UserName = "李四", Time = DateTime.Now });
```

---

## 七、性能优化与注意事项

### 委托缓存

```csharp
public static class EventDelegateCache
{
    // 用于存储已生成的委托，Key 通常由事件名和处理方法名组成，Value 是生成的委托实例
    private static readonly Dictionary<string, Delegate> _cache = new();

    /// <summary>
    /// 获取或创建一个事件处理委托
    /// </summary>
    /// <param name="evt">事件的反射信息 (EventInfo)</param>
    /// <param name="target">响应事件的对象实例（如果是静态方法则为 null）</param>
    /// <param name="handlerMethod">处理事件的方法反射信息 (MethodInfo)</param>
    /// <returns>构造好的委托</returns>
    public static Delegate GetOrCreateHandler(EventInfo evt, object target, MethodInfo handlerMethod)
    {
        // 生成唯一的缓存键。
        // 注意：如果 target 是实例对象，建议将 target 的 HashCode 也计入 Key，否则不同实例会串样。
        string key = $"{evt.DeclaringType.FullName}.{evt.Name}_{handlerMethod.Name}";

        // 线程锁，确保在多线程环境下字典的操作是安全的
        lock (_cache)
        {
            if (!_cache.TryGetValue(key, out var handler))
            {
                // 使用反射动态创建符合事件处理器类型的委托
                handler = Delegate.CreateDelegate(evt.EventHandlerType, target, handlerMethod);
                _cache[key] = handler;
            }
            return handler;
        }
    }
}

// 1. 定义一个拥有事件的类
public class MyButton
{
    public event EventHandler Click;

    public void SimulateClick()
    {
        Console.WriteLine("按钮被点击了...");
        Click?.Invoke(this, EventArgs.Empty);
    }
}

// 2. 定义处理逻辑的类
public class UIHandler
{
    public void OnButtonClicked(object sender, EventArgs e)
    {
        Console.WriteLine("UIHandler 接收到了点击事件！");
    }
}

// 使用
MyButton btn = new MyButton();
UIHandler handlerObj = new UIHandler();

// 获取 EventInfo (点击事件)
EventInfo clickEvent = typeof(MyButton).GetEvent("Click");

// 获取 MethodInfo (处理方法)
MethodInfo method = typeof(UIHandler).GetMethod("OnButtonClicked");

// 使用你的缓存类创建委托
Delegate del = EventDelegateCache.GetOrCreateHandler(clickEvent, handlerObj, method);

// 将生成的委托通过反射添加到事件中
// 相当于执行了 btn.Click += handlerObj.OnButtonClicked;
clickEvent.AddEventHandler(btn, del);

// 测试效果
btn.SimulateClick();

```

### 避免内存泄漏

```csharp
// 事件订阅容易导致内存泄漏（强引用）
// 解决方案1：及时取消订阅
clickEvent.RemoveEventHandler(btn, handler);

// 解决方案2：使用弱引用
public class WeakEventManager
{
    private readonly Dictionary<EventInfo, List<WeakDelegate>> _subscriptions = new();
    
    public void Subscribe(EventInfo evt, object target, Delegate handler)
    {
        lock (_subscriptions)
        {
            if (!_subscriptions.ContainsKey(evt))
                _subscriptions[evt] = new List<WeakDelegate>();
            
            _subscriptions[evt].Add(new WeakDelegate(handler));
            evt.AddEventHandler(target, handler);
        }
    }
    
    private class WeakDelegate
    {
        private readonly WeakReference _target;
        private readonly MethodInfo _method;
        
        public WeakDelegate(Delegate d)
        {
            _target = new WeakReference(d.Target);
            _method = d.Method;
        }
        
        public bool IsAlive => _target.IsAlive;
        
        public Delegate GetDelegate(Type delegateType)
        {
            object target = _target.Target;
            if (target == null) return null;
            return Delegate.CreateDelegate(delegateType, target, _method);
        }
    }
}
```

---

## 八、常见陷阱与最佳实践

### 陷阱：多线程订阅

```csharp
// EventInfo 的 AddEventHandler/RemoveEventHandler 本身不是线程安全的
// 需要额外同步

lock (syncObj)
{
    clickEvent.AddEventHandler(btn, handler);
}
```

### 最佳实践：检查事件是否存在

```csharp
public static bool TrySubscribe(object target, string eventName, Delegate handler)
{
    Type type = target.GetType();
    EventInfo evt = type.GetEvent(eventName);
    
    if (evt == null)
    {
        Console.WriteLine($"事件 {eventName} 不存在");
        return false;
    }
    
    if (evt.AddMethod == null)
    {
        Console.WriteLine($"事件 {eventName} 没有 add 访问器");
        return false;
    }
    
    evt.AddEventHandler(target, handler);
    return true;
}
```

---

## 九、完整实战：动态事件绑定器

```csharp
public class DynamicEventBinder
{
    private readonly Dictionary<object, List<(EventInfo, Delegate)>> _bindings = new();
    
    /// <summary>
    /// 将源对象的事件绑定到目标对象的方法
    /// </summary>
    public void Bind(object source, string sourceEventName, object target, string targetMethodName)
    {
        Type sourceType = source.GetType();
        Type targetType = target.GetType();
        
        // 获取源事件
        EventInfo sourceEvent = sourceType.GetEvent(sourceEventName);
        if (sourceEvent == null)
            throw new ArgumentException($"事件 {sourceEventName} 不存在于 {sourceType.Name}");
        
        // 获取目标方法
        MethodInfo targetMethod = targetType.GetMethod(targetMethodName, 
            BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
        if (targetMethod == null)
            throw new ArgumentException($"方法 {targetMethodName} 不存在于 {targetType.Name}");
        
        // 检查方法签名是否匹配事件委托
        ParameterInfo[] methodParams = targetMethod.GetParameters();
        ParameterInfo[] eventParams = sourceEvent.EventHandlerType.GetMethod("Invoke").GetParameters();
        
        if (methodParams.Length != eventParams.Length)
            throw new ArgumentException("方法签名与事件委托不匹配");
        
        // 创建委托
        Delegate handler = Delegate.CreateDelegate(sourceEvent.EventHandlerType, target, targetMethod);
        
        // 订阅
        sourceEvent.AddEventHandler(source, handler);
        
        // 记录以便取消
        lock (_bindings)
        {
            if (!_bindings.ContainsKey(source))
                _bindings[source] = new List<(EventInfo, Delegate)>();
            
            _bindings[source].Add((sourceEvent, handler));
        }
    }
    
    /// <summary>
    /// 取消对象的所有事件绑定
    /// </summary>
    public void UnbindAll(object source)
    {
        lock (_bindings)
        {
            if (_bindings.TryGetValue(source, out var bindings))
            {
                foreach (var (evt, handler) in bindings)
                {
                    evt.RemoveEventHandler(source, handler);
                }
                _bindings.Remove(source);
            }
        }
    }
}

// 使用示例
public class Button
{
    public event EventHandler Click;
    public event EventHandler DoubleClick;
    
    public void SimulateClick() => Click?.Invoke(this, EventArgs.Empty);
    public void SimulateDoubleClick() => DoubleClick?.Invoke(this, EventArgs.Empty);
}

public class Logger
{
    public void LogClick(object sender, EventArgs e)
    {
        Console.WriteLine($"[LOG] 点击事件: {DateTime.Now}");
    }
    
    public void LogDoubleClick(object sender, EventArgs e)
    {
        Console.WriteLine($"[LOG] 双击事件: {DateTime.Now}");
    }
}

var button = new Button();
var logger = new Logger();
var binder = new DynamicEventBinder();

// 动态绑定
binder.Bind(button, "Click", logger, "LogClick");
binder.Bind(button, "DoubleClick", logger, "LogDoubleClick");

// 测试
button.SimulateClick();       // [LOG] 点击事件: ...
button.SimulateDoubleClick(); // [LOG] 双击事件: ...

// 取消所有绑定
binder.UnbindAll(button);
button.SimulateClick();       // 无输出
```

---

## 十、本篇速查表

| 操作 | 代码 |
|------|------|
| 获取事件 | `typeof(T).GetEvent("EventName")` |
| 订阅事件 | `evt.AddEventHandler(target, handler)` |
| 取消订阅 | `evt.RemoveEventHandler(target, handler)` |
| 获取 add 方法 | `evt.GetAddMethod()` |
| 获取 remove 方法 | `evt.GetRemoveMethod()` |
| 获取委托类型 | `evt.EventHandlerType` |
| 静态事件 | target 参数传 `null` |
| 私有事件 | 需要 `BindingFlags.NonPublic` |
| 创建委托 | `Delegate.CreateDelegate(handlerType, target, method)` |
| 泛型事件 | 用 `MakeGenericType` 构造 `EventHandler<T>` |
| 自定义事件访问器 | 通过 `GetAddMethod` 直接调用 |

---

## 十一、思考题

1. 事件和委托的根本区别是什么？为什么事件不能被外部直接调用？

2. 如何通过反射找到触发事件的方法（通常命名为 `OnXxx`）？

3. 事件订阅可能导致内存泄漏的原因是什么？如何避免？

4. 如何实现一个支持异步事件处理的事件总线？

---

**下一篇预告：** 《C# 反射系列笔记（九）：Attribute 特性深度解析》

下一篇将深入讲解特性的定义、使用和反射读取，包括自定义特性、特性参数限制、特性在序列化和验证中的应用。