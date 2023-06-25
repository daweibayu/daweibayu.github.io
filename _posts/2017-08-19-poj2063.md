---
layout: post
title:  "Poj 2063 Investment"
author: "daweibayu"
tags: 解题报告
excerpt_separator: <!--more-->
---

<!--more-->

## 题目
Poj 2063 [Investment](http://poj.org/problem?id=2063)
```
Description
John never knew he had a grand-uncle, until he received the notary's letter. He learned that his late grand-uncle had gathered a lot of money, somewhere in South-America, and that John was the only inheritor. 
John did not need that much money for the moment. But he realized that it would be a good idea to store this capital in a safe place, and have it grow until he decided to retire. The bank convinced him that a certain kind of bond was interesting for him. 
This kind of bond has a fixed value, and gives a fixed amount of yearly interest, payed to the owner at the end of each year. The bond has no fixed term. Bonds are available in different sizes. The larger ones usually give a better interest. Soon John realized that the optimal set of bonds to buy was not trivial to figure out. Moreover, after a few years his capital would have grown, and the schedule had to be re-evaluated. 
Assume the following bonds are available: 
Value	Annual
interest
4000
3000	400
250

With a capital of e10 000 one could buy two bonds of $4 000, giving a yearly interest of $800. Buying two bonds of $3 000, and one of $4 000 is a better idea, as it gives a yearly interest of $900. After two years the capital has grown to $11 800, and it makes sense to sell a $3 000 one and buy a $4 000 one, so the annual interest grows to $1 050. This is where this story grows unlikely: the bank does not charge for buying and selling bonds. Next year the total sum is $12 850, which allows for three times $4 000, giving a yearly interest of $1 200. 
Here is your problem: given an amount to begin with, a number of years, and a set of bonds with their values and interests, find out how big the amount may grow in the given period, using the best schedule for buying and selling bonds.

Input
The first line contains a single positive integer N which is the number of test cases. The test cases follow. 
The first line of a test case contains two positive integers: the amount to start with (at most $1 000 000), and the number of years the capital may grow (at most 40). 
The following line contains a single number: the number d (1 <= d <= 10) of available bonds. 
The next d lines each contain the description of a bond. The description of a bond consists of two positive integers: the value of the bond, and the yearly interest for that bond. The value of a bond is always a multiple of $1 000. The interest of a bond is never more than 10% of its value.
Output

For each test case, output – on a separate line – the capital at the end of the period, after an optimal schedule of buying and selling.

Sample Input
1
10000 4
2
4000 400
3000 250

Sample Output
14050
```
具体的内容就是一共有多组债券，分别给出了每组债券的价格与每年的利息，然后根据投资的年数与投资的初始钱数，计算最终的总价值。
输入：
N(测试数据数量)
amount(初始钱数) years(年数)
d(债券数量)
value(债券价格) interest(每年的利息)
...(一共 d 组)


## 思路
核心其实还是背包问题，关键有几个点：
1. 初始钱数最大为 1000000，年数最大为 40，利息最大为 10%，有此我们可以计算出最终的结果最大约为 45260000，按照背包的思路，如果直接开这么大的数据内存就直接爆掉了，而题目给了每组债券都是 1000 的整数倍，所以 1000 以下的部分其实跟利息的计算是毫无关系的(仅仅是每年哟)，而 1000 以上的部分可以直接通过除掉 1000 来减小数组的大小，这样数组其实开到 45260 就可以了。
2. 先暴力计算出所有可能金钱得到的利息数（即 result[]），这样在以后每年计算的时候都可以直接取值就可以了
3. 剩下的问题就是计算 result[] 了，标准的背包问题，这里我就不细说了，如果不了解的话直接看《背包九讲》吧。

## 代码
```c++
// http://poj.org/problem?id=2063
#include <stdio.h>
#include <iostream>
#include <algorithm>
#include <cstring>
#include<math.h>
using namespace std;

#define max(x,y) (x > y ? x : y)

//债券
struct Bond {
    // 债券的价格
    int value;
    
    // 债券的利息
    int interest;
};

// 债券的总列表
Bond bondList[15];

// index 代表金额（除以 1000 后的值），result[index] 为这个金额最大的利息数
int result[50000];

/**
 * 根据债券的信息，更新 result 数据
 **/
void updateResult(int maxAmount, int boudNum) {
    for (int i = 0; i < boudNum; i++) {
        int value = bondList[i].value;
        int interest = bondList[i].interest;
        
        int lastMax = 0;
        for(int t = 0; t <= maxAmount - value; t++) {
            if (result[t] != 0) {
                lastMax = result[t];
            }
            result[t + value] = max(result[t + value], result[t] + interest);
            if (result[t] == 0) {
                result[t] = lastMax;
            }
        }
    }
}

/**
 * 根据 updateResult 计算出来的结果，计算最终结果
 **/
int calculate(int amount, int years) {
    int total = amount;
    for(int i = 0; i < years; i++) {
        total += result[total / 1000];
    }
    return total;
}

int main(int argc, const char * argv[]) {
    
    // 处理输入数据，不多解释
    int n, amount, years, boudNum;
    scanf("%d", &n);
    
    for (int i = 0; i < n; i++) {
        scanf("%d %d %d", &amount, &years, &boudNum);
        for (int t = 0; t < boudNum; t++) {
            scanf("%d %d", &bondList[t].value, &bondList[t].interest);
            // 因为债券的价格都是 1000 的整数倍，所以这里可以除掉 1000
            bondList[t].value /= 1000;
        }

        // 初始化数据，避免每个 testcase 互相影响
        memset(result, 0, sizeof(result));
        
        // 复利，每年最高 10%，可以计算出最大的可能结果，剪枝，避免过多的计算
        int maxAmount = ((int)(pow(1.1, years) * amount / 1000) + 1);
        
        updateResult(maxAmount, boudNum);
        printf("%d\n", calculate(amount, years));
    }
}
```