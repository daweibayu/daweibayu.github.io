---
layout: post
title:  "Poj 1458 Common Subsequence"
author: "daweibayu"
excerpt_separator: <!--more-->
---

<!--more-->

Poj 1458 [Common Subsequence](http://poj.org/problem?id=1458)

## 题目

```
Description
A subsequence of a given sequence is the given sequence with some elements (possible none) left out. Given a sequence X = < x1, x2, ..., xm > another sequence Z = < z1, z2, ..., zk > is a subsequence of X if there exists a strictly increasing sequence < i1, i2, ..., ik > of indices of X such that for all j = 1,2,...,k, xij = zj. For example, Z = < a, b, f, c > is a subsequence of X = < a, b, c, f, b, c > with index sequence < 1, 2, 4, 6 >. Given two sequences X and Y the problem is to find the length of the maximum-length common subsequence of X and Y.

Input
The program input is from the std input. Each data set in the input contains two strings representing the given sequences. The sequences are separated by any number of white spaces. The input data are correct.

Output
For each set of data the program prints on the standard output the length of the maximum-length common subsequence from the beginning of a separate line.

Sample Input
abcfbc         abfcab
programming    contest 
abcd           mnp

Sample Output
4
2
0
```
最长公共子序列问题。两个字符串，一个为目标串（如第一个用例中的 abcfbc），一个为子串（如第一个用例中的 abfcab），目标是求出这两个字符串中的公共部分（可以不连续，但是顺序要保证，第一个用例的最长公共部分为 abcb、abfb）。此题为计算最长公共子序列的长度问题。

## 解题思路
设目标串为数组 a[]，其长度为 aLength
设子串为数组 b[]，其长度为 bLength
设 f(aStart, bStart) 代表 a[aStart, aLength)、b[bStart, bLength) 两个子串的最长公共子序列的长度
设 i(aStart, bIndex) 代表 b[bIndex] 这个字符在 a 的左闭右开区间[aStart, aLength) 的索引值

单独考虑 b[bStart] 这个字符，只有两种情况，一种是最长公共部分包含此字符，另一种是不包含此字符，而最终结果则是取这两种情况的最大值，那接下来我们分别求取其值：
对于第一种情况：即一定包含 b[bStart] 字符，则可得到 f(i(aStart, bStart), bStart + 1) + 1
对于第二种情况：即一定不包含 b[bStart] 字符，则可得到 f(aStart, bStart + 1)

由此可推到出公式 f(aStart, bStart) = max(f(i(aStart, bStart), bStart + 1) + 1, f(aStart, bStart + 1))


## 代码
```c++
#include <stdio.h>
#include <cstring>

#define MAX_LENGTH 502
#define max(x,y) (x > y ? x : y)

char a[MAX_LENGTH];
char b[MAX_LENGTH];
int resultArray[502][502];
int aLength, bLength;

// 得到字符 c 在 a[] 中的索引值（从 aStart 开始计算）
int getFirstIndexFromA(char c, int aStart) {
    for (int i = aStart; i < aLength; i++) {
        if (a[i] == c) {
            return i;
        }
    }
    return -1;
}

// 获取字符串的长度
int getArrayLength(char* array) {
    for (int i = 0; i < MAX_LENGTH; i++) {
        if (array[i] == '\0') {
            return i;
        }
    }
    return 0;
}

// 计算 a[aStart, aLength) 区间字符串与 b[bStart, bLength] 区间字符串的最长公共子序列的长度
int getMaxLength(int aStart, int bStart) {
    if (resultArray[aStart][bStart] == -1) {
        return 0;
    }
    if (resultArray[aStart][bStart] != 0) {
        return resultArray[aStart][bStart];
    }
    
    if (aStart == aLength) {
        resultArray[aStart][bStart] = -1;
        return 0;
    }
    int index = getFirstIndexFromA(b[bStart], aStart);
    int result;
    if (bStart == bLength) {
        result = (index == -1 ? 0 : 1);
    } else if (index == -1) {
        result = getMaxLength(aStart, bStart + 1);
    } else if (index == aStart) {
        result = getMaxLength(index + 1, bStart + 1) + 1;
    } else {
        int maxNumWithoutAStart = getMaxLength(index + 1, bStart + 1) + 1;
        int maxNumWithoutBStart = getMaxLength(aStart, bStart + 1);
        result = max(maxNumWithoutAStart, maxNumWithoutBStart);
    }
    resultArray[aStart][bStart] = (result == 0 ? -1 : result);
    return result;
}

int main(int argc, const char * argv[]) {
    while (scanf("%s", a) != EOF) {
        scanf("%s", b);
        aLength = getArrayLength(a);
        bLength = getArrayLength(b);
        memset(resultArray, 0, sizeof(resultArray));
        printf("%d\n", getMaxLength(0, 0));
    }
}
```
其中代码做了一些优化，比如用 resultArray[][] 记录计算结果，避免重复计算，这里需要说明一下，因为不想写 for 循环来初始化这个数组，所以 resultArray[][] 的默认值为 0，代表这个值未被计算过，当其值为 -1 时，认为这个值是计算过的，但是结果为 0，当其值既非 0 也非 -1 时，则代表这个值是计算过的结果。
另外就是对于 a[aStart, aLength)、b[bStart, bLength]，如果 a[aStart] == b[bStart]，则其最长子串中肯定包含这个字符，所以其结果肯定为 getMaxLength(aStart + 1, bStart + 1) + 1。
