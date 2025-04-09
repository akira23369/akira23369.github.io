---
title: 求组合数 III
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

[887. 求组合数 III - AcWing题库](https://www.acwing.com/problem/content/889/)

![](求组合数%20III/Pasted%20image%2020240511003401.png)

![](求组合数%20III/Pasted%20image%2020240511003408.png)




```cpp

#include<iostream>
using namespace std;
typedef long long LL;

int qmi(int a, int k, int p)
{
    int res = 1;
    while (k)
    {
        if (k & 1) res =(LL) res * a % p;
        a =(LL) a * a % p;
        k >>= 1;
    }
    return res;
}

int C(int a, int b, int p)
{
    if (b > a) return 0;
    int res = 1;
    for (int i = a, j = b; j >= 1; j--, i--)
    {
        res =(LL) res * i % p;
        res =(LL) res * qmi(j, p - 2, p) % p;
    }
    return res;
}

int lucas(int a, int b, int p)
{
    if (a < p && b < p) return C(a, b, p);
    return (LL)C(a % p, b % p, p) * lucas(a / p, b / p, p) % p;
}

int main()
{
    int n;
    cin >> n;
    while (n--)
    {
        int a, b, p;
        cin >> a >> b >> p;
        cout << lucas(a, b, p) << endl;
    }
    return 0;
}

作者：爱不被爱的爱酱
链接：https://www.acwing.com/activity/content/code/content/4769656/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```