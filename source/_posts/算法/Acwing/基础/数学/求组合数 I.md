---
title: 求组合数 I
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

[885. 求组合数 I - AcWing题库](https://www.acwing.com/problem/content/887/)

### 题目描述
给定 $n$ 组询问，每组询问给定两个整数 $a，b$，请你输出 $C_a^b \bmod (10^9 + 7)$ 的值。

#### 输入格式

第一行包含整数 $n$。

接下来 $n$ 行，每行包含一组 $a$ 和 $b$。

#### 输出格式

共 $n$ 行，每行输出一个询问的解。

#### 数据范围

$1 \le n \le 10000$,  
$1 \le b \le a \le 2000$

#### 输入样例：

```
3
3 1
5 3
2 2
```

#### 输出样例：

```
3
10
1
```

---
### 算法

```cpp
// 求组合数 1 ： 递推式
#include<iostream>
using namespace std;

const int N = 2010, mod = 1e9 + 7;
int c[N][N];
// 预处理所有的组合数
void init()
{
    for (int i = 0; i < N; i++)
        for (int j = 0; j <= i; j++)
        {
            if (!j) c[i][j] = 1;
            else c[i][j] = (c[i - 1][j] + c[i - 1][j - 1]) % mod;
        }
}


int main()
{
    int n;
    cin >> n;
    init();
    while (n--)
    {
        int a, b;
        cin >> a >> b;
        cout << c[a][b] << endl;
    }
    return 0;
}

作者：爱不被爱的爱酱
链接：https://www.acwing.com/activity/content/code/content/4757290/
来源：AcWing
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```