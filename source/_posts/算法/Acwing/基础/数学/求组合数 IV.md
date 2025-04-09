---
title: 求组合数 IV
date: 2000-01-01 00:00:00
toc: false
categories:
  - 算法
  - Acwing
  - 基础
  - 数学

tags:
  - 算法
  - Acwing
  - 基础
  - 数学
  - 模板题
---

[888. 求组合数 IV - AcWing题库](https://www.acwing.com/problem/content/890/)

### 题目描述
输入 $a, b$，求 $C_a^b$ 的值。

注意结果可能很大，需要使用高精度计算。

#### 输入格式

共一行，包含两个整数 $a$ 和 $b$。

#### 输出格式

共一行，输出 $C_a^b$ 的值。

#### 数据范围

$1 \le b \le a \le 5000$

#### 输入样例：

```
5 3
```

#### 输出样例：

```
10
```

---
### 算法

![](求组合数%20IV/Pasted%20image%2020240511003548.png)


![](求组合数%20IV/Pasted%20image%2020240511003552.png)

![](求组合数%20IV/Pasted%20image%2020240511003556.png)


```cpp
// 怎么求没有mod p 的组合数？

// 思路：
// 将阶乘质因数分解 将其转化为质因数相乘然后用高精度

// 如何求质因数的次数？
// 如上图


#include<iostream>
#include<algorithm>
#include<vector>
using namespace std;
const int N = 5010;
int primes[N], cnt;
bool st[N];    
int sum[N];         // primes[i] = p;  sum[i] = p的次数

void get_primes(int n)
{
    for (int i = 2; i <= n; i++)
    {
        if (!st[i]) primes[cnt++] = i;
        for (int j = 0; primes[j] <= n / i; j++)
        {
            st[primes[j] * i] = true;
            if (i % primes[j] == 0) break;
        }
    }
}

// 求阶乘n中有多少个质数p
int get(int n, int p)
{
    int res = 0;
    while (n)
    {
        res += n / p;
        n /= p;
    }
    return res;
}

vector<int> mul(vector<int>& A, int b)
{
    vector<int> c;
    int t = 0;
    for (int i = 0; i < A.size(); i++)
    {
        t += A[i] * b;
        c.push_back(t % 10);
        t /= 10;
    }
    if (t) c.push_back(t);
    while (c.size() > 1 && c.back() == 0) c.pop_back();
    return c;
}


int main()
{
    int a, b;
    cin >> a >> b;
    get_primes(a);      // 因为a大一点
    for (int i = 0; i < cnt; i++)
    {
        int  p = primes[i];
        sum[i] = get(a, p) - get(b, p) - get(a - b, p);
    }
    // 运用高精度将所有质因数乘起来就行
    vector<int> res = {1};
    for (int i = 0; i < cnt; i++)
        for (int j = 0; j < sum[i]; j++)
        {
            res = mul(res, primes[i]);
        }
    for (int i = res.size() - 1; i >= 0; i--)
    {
        cout << res[i];
    }

    return 0;
}

作者：爱不被爱的爱酱
链接：https://www.acwing.com/activity/content/code/content/4769643/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```