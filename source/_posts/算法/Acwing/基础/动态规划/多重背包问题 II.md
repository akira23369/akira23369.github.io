---
title: 多重背包问题 II
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
---

[5. 多重背包问题 II - AcWing题库](https://www.acwing.com/problem/content/5/)

[E10 背包DP 多重背包 二进制优化——信息学奥赛算法_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1MA41177cg/?spm_id_from=333.1387.search.video_card.click&vd_source=2f348893e98a838d97300d2bf728b18b)


### 题目描述
有 $N$ 种物品和一个容量是 $V$ 的背包。

第 $i$ 种物品最多有 $s_i$ 件，每件体积是 $v_i$，价值是 $w_i$。

求解将哪些物品装入背包，可使物品体积总和不超过背包容量，且价值总和最大。  
输出最大价值。

#### 输入格式

第一行两个整数，$N，V$，用空格隔开，分别表示物品种数和背包容积。

接下来有 $N$ 行，每行三个整数 $v_i, w_i, s_i$，用空格隔开，分别表示第 $i$ 种物品的体积、价值和数量。

#### 输出格式

输出一个整数，表示最大价值。

#### 数据范围

$0 \lt N \le 1000$  
$0 \lt V \le 2000$  
$0 \lt v_i, w_i, s_i \le 2000$

##### 提示：

本题考查多重背包的二进制优化方法。

#### 输入样例

```
4 5
1 2 3
2 4 1
3 4 3
4 5 2
```

#### 输出样例：

```
10
```

---
### 算法

![](多重背包问题%20II/Pasted%20image%2020240511005450.png)


```cpp
#include<iostream>
using namespace std;
int n, m, s;
const int N = 12 * 1000 + 10;       
int vv[N], ww[N];
int f[N];

int main()
{
    cin >> n >> m;
    int cnt = 1;    // 从1开始存
    for (int i = 1; i <= n; i++)
    {
        int v, w, s;    // 体积， 价值， 数量
        cin >> v >> w >> s;
        // 二进制拆分
        for (int j = 1; j <= s; j <<= 1)
        {
            vv[cnt] = j * v;
            ww[cnt++] = j * w;
            s -= j;
        }
        if (s)
        {
            vv[cnt] = s * v;
            ww[cnt++] = s * w;
        }
    }
    // i < cnt  因为上面已经++
    for (int i = 1; i < cnt; i++)
        for (int j = m; j >= vv[i]; j--)
            f[j] = max(f[j], f[j - vv[i]] + ww[i]);
    cout << f[m] << endl;
    return 0;
}

作者：爱不被爱的爱酱
链接：https://www.acwing.com/activity/content/code/content/4820640/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```