---
title: 最短Hamilton路径
date: 2000-01-01 00:00:00
toc: false
categories:
  - 算法
  - Acwing
  - 基础
  - 动态规划

tags:
  - 算法
  - Acwing
  - 基础
  - 动态规划
  - 模板题
---

[91. 最短Hamilton路径 - AcWing题库](https://www.acwing.com/problem/content/93/)
### 题目描述
给定一张 $n$ 个点的带权无向图，点从 $0 \sim n-1$ 标号，求起点 $0$ 到终点 $n-1$ 的最短 Hamilton 路径。

Hamilton 路径的定义是从 $0$ 到 $n-1$ 不重不漏地经过每个点恰好一次。

#### 输入格式

第一行输入整数 $n$。

接下来 $n$ 行每行 $n$ 个整数，其中第 $i$ 行第 $j$ 个整数表示点 $i$ 到 $j$ 的距离（记为 $a[i,j]$）。

对于任意的 $x,y,z$，数据保证 $a[x,x]=0，a[x,y]=a[y,x]$ 并且 $a[x,y]+a[y,z] \ge a[x,z]$。

#### 输出格式

输出一个整数，表示最短 Hamilton 路径的长度。

#### 数据范围

$1 \le n \le 20$  
$0 \le a[i,j] \le 10^7$

#### 输入样例：

```
5
0 2 4 5 1
2 0 6 5 3
4 6 0 8 3
5 5 8 0 5
1 3 3 5 0
```

#### 输出样例：

```
18
```

---
### 算法



```cpp
// 暴力枚举
// 0 -> 1 -> 2 -> 3  18wei √
// 0 -> 2 -> 1 -> 3  20wei ×    // 这种情况是可以去掉

// 上式规律：
// 1.只需要关心那些点被使用过，不关心顺序
// 2.目前停在哪个点上

// f[state][j] 来表示使用了那些点，终点为j的路径权值

// state用二进制来表示点是否被使用过
// eg：011 表示使用过0,1点

// 递推方程： 用k表示到j点之前那个点
// f[state][j] = min(f[state - j][k] + weight[k][j]);      ps: state - j 表示除去j点的集合的所有状态
// 初值：f[1][0] = 0;   只经过0，到0的的路径权值为0
// 目标值：f[(1 << n) - 1][n - 1]   走过所有店到n-1点的最小路径



#include<iostream>
#include<cstring>
const int N = 20, M = 1 << 20;
int f[M][N];
int weight[N][N];
int n;
int main()
{
    std::cin >> n;
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            std::cin >> weight[i][j];
    memset(f, 0x3f, sizeof(f));
    f[1][0] = 0;    
    for (int i = 0; i < 1 << n; i++)
        for (int j = 0; j < n; j++)
            if (i >> j & 1)     // i 这条路径是否走到过j点
                for (int k = 0; k < n; k++)
                    if (i >> k & 1) // i这条路径是否走到过k点
                    {
                        f[i][j] = std::min(f[i][j], f[i - (1 << j)][k] + weight[k][j]);     // i - (1 << j) 除去j点
                    }
    std::cout << f[(1 << n) - 1][n - 1] << std::endl;
    return 0;
}

作者：爱不被爱的爱酱
链接：https://www.acwing.com/activity/content/code/content/4866793/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```
