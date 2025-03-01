---
title: csharp事件
date: 2025-02-28 19:31:10
toc: true
categories:
  - 编程语言
  - Csharp

tags:
  - 编程语言
  - Csharp
---

# **什么是事件？**
事件的核心是：**当某件事发生时，自动通知所有关心它的人**。
在 C# 中，事件是一种机制，允许一个对象（发布者）在特定动作发生时，通知其他对象（订阅者）执行某些代码。

# **事件的核心三要素**
1. **发布者（Publisher）**：定义事件并触发它
2. **订阅者（Subscriber）**：注册事件处理方法
3. **事件处理方法（Handler）**：当事件发生时执行的代码


# **游戏开发高频场景**

| **场景**     | **事件用法**           | **代码灵魂示例**                              |
| ---------- | ------------------ | --------------------------------------- |
| **角色受伤**   | 触发UI血条更新、音效、伤害数字   | `player.OnHurt += UpdateHealthBar;`     |
| **敌人死亡**   | 触发掉落物品、任务进度更新、成就解锁 | `enemy.OnDeath += DropLoot;`            |
| **技能释放**   | 触发特效、冷却计时、连击计数     | `skill.OnCast += PlayVFX;`              |
| **游戏状态切换** | 暂停/继续、关卡加载完成、游戏结束  | `GameManager.OnPause += FreezeEnemies;` |
| **UI交互**   | 按钮点击、菜单打开/关闭、道具拖动  | `button.OnClick += OpenInventory;`      |

# 代码示例
**角色受伤**触发UI血条更新、音效、伤害数字

## 第一步：定义事件参数类（传递伤害值）
**为什么事件参数要继承 `EventArgs`？**
- 这是一个约定，保持代码统一性。
- 如果需要传递数据，推荐使用自定义的 `EventArgs` 子类。
- **EventArgs 作用**：  
    通过自定义参数类，把伤害值传递给所有监听者，避免每个系统单独查询角色状态
```cs
public class DamageEventArgs : EventArgs
{
    public int Damage { get; } // 需要传递的伤害值
    
    public DamageEventArgs(int damage)
    {
        Damage = damage;
    }
}
```

## 第二步：创建角色类（事件发布者）
**`EventHandler<T>` 委托**
- 是 .NET 内置的泛型委托，无需自己定义。
- 签名：`void EventHandler<TEventArgs>(object sender, TEventArgs e)`。
- `sender`：触发事件的对象（通常是发布者自己）。
- `e`：事件参数（传递额外数据）。
- 事件名称以 `On` 开头（如 `OnClick`）。
```cs
public class Character
{
    // 声明事件（使用自定义的EventArgs）
    public event EventHandler<DamageEventArgs> Damaged;

    public void TakeDamage(int damage)
    {
        // 触发事件的通用写法
        OnDamaged(new DamageEventArgs(damage));
    }

    // 触发事件
    protected virtual void OnDamaged(DamageEventArgs e)
    {
        Damaged?.Invoke(this, e);  
    }
}
```

## 第三步：创建各种事件订阅者
```cs
// UI血条控制器
public class UIHealthBar
{
    // 别人sender发来通知，并携带了e的参数，你要做的事如下： 更新血条
    public void OnCharacterDamaged(object sender, DamageEventArgs e)
    {
        UpdateHealthBar(e.Damage);
        Debug.Log($"血条更新：减少{e.Damage}HP");
    }

    private void UpdateHealthBar(int damage) { /* 实际血条逻辑 */ }
}

// 音效系统
public class SoundSystem
{
    // 别人sender发来通知，并携带了e的参数，你要做的事如下： 播放受伤音效
    public void PlayHurtSound(object sender, DamageEventArgs e)
    {
        Audio.Play("受伤音效");
        Debug.Log("播放受伤音效");
    }
}

// 伤害数字系统
public class DamageNumbers
{
    // 别人sender发来通知，并携带了e的参数，你要做的事如下： 显示上海数字
    public void ShowDamagePopup(object sender, DamageEventArgs e)
    {
        CreateFloatingText(e.Damage);
        Debug.Log($"显示伤害数字：{e.Damage}");
    }

    private void CreateFloatingText(int damage) { /* 数字弹窗逻辑 */ }
}

```

## 第四步：连接事件订阅
```cs
// 初始化所有对象
Character player = new Character();
UIHealthBar ui = new UIHealthBar();
SoundSystem sound = new SoundSystem();
DamageNumbers numbers = new DamageNumbers();

// 订阅事件（+= 添加监听）
player.Damaged += ui.OnCharacterDamaged;
player.Damaged += sound.PlayHurtSound;
player.Damaged += numbers.ShowDamagePopup;
```

## 第五步：触发事件
```cs
// 当玩家受到伤害时
player.TakeDamage(50);

// 输出结果：
// 血条更新：减少50HP
// 播放受伤音效
// 显示伤害数字：50
```


# 事件与委托的区别


| **特性**            | **委托（Delegate）** | **事件（Event）**      |
| ----------------- | ---------------- | ------------------ |
| **访问权限**          | 可直接调用或赋值（`=`）    | 只能在类内部触发（`Invoke`） |
| **多播（Multicast）** | 支持（`+=`/`-=`）    | 支持（本质是委托的封装）       |
| **封装性**           | 低（外部可任意修改委托链）    | 高（外部只能订阅/取消订阅）     |
| **典型用途**          | 通用回调机制           | 发布-订阅模式的通知机制       |
