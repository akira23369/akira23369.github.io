---
title: 最长上升子序列 II
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

[896. 最长上升子序列 II - AcWing题库](https://www.acwing.com/problem/content/898/)


### 题目描述
给定一个长度为 $N$ 的数列，求数值严格单调递增的子序列的长度最长是多少。

#### 输入格式

第一行包含整数 $N$。

第二行包含 $N$ 个整数，表示完整序列。

#### 输出格式

输出一个整数，表示最大长度。

#### 数据范围

$1 \le N \le 100000$，  
$-10^9 \le 数列中的数 \le 10^9$

#### 输入样例：

```
7
3 1 2 1 8 5 6
```

#### 输出样例：

```
4
```

---
### 算法

![](最长上升子序列%20II/Pasted%20image%2020240513002118.png)

```cpp
/ 最难理解的地方在于栈中序列虽然递增，但是每个元素在原串中对应的位置其实可能是乱的，那为什么这个栈还能用于计算最长子序列长度？
// 实际上这个栈【不用于记录最终的最长子序列】，而是【以stk[i]结尾的子串长度最长为i】
// 或者说【长度为i的递增子串中，末尾元素最小的是stk[i]】。
// 理解了这个问题以后就知道为什么新进来的元素要不就在末尾增加，要不就替代第一个大于等于它元素的位置。
// 这里的【替换】就蕴含了一个贪心的思想，对于同样长度的子串，我当然希望它的末端越小越好，这样以后我也有更多机会拓展。

#include<iostream>
using namespace std;
const int N = 1e5 + 10;
int a[N], b[N], len;
int n;

int find(int a[], int l, int r, int x)
{
    while (l < r)
    {
        int mid = (l + r) >> 1;
        if (a[mid] >= x) r = mid;
        else l = mid + 1;
    }
    return l;
}

int main()
{
    cin >> n;
    for (int i = 0; i < n; i++) cin >> a[i];
    b[len++] = a[0];
    for (int i = 1; i < n; i++)
    {
        if (a[i] > b[len - 1]) b[len++] = a[i];
        else
        {
            int j = find(b, 0, len - 1, a[i]);
            b[j] = a[i];
        }
    }
    cout << len << endl;
    return 0;
}

作者：爱不被爱的爱酱
链接：https://www.acwing.com/activity/content/code/content/5339210/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```