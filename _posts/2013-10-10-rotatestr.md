---
layout: post
title:  "字符串旋转"
author: "daweibayu"
tags: 算法
excerpt_separator: <!--more-->
---

<!--more-->

​
问题：

把字符串前面的若干个字符移动到字符串的尾部。如把字符串abcdef前2位字符移到后面得到字符串cdefab。要求时间对长度为n的字符串操作的复杂度为O(n)，辅助内存为O(1)。

看到大多数的帖子都是进行三次旋转

如：[程序员编程艺术：第一章、左旋转字符串_v_JULY_v的博客-CSDN博客](https://blog.csdn.net/v_JULY_v/article/details/6322882)

个人感觉这思路确实比较新颖，但是总感觉有点麻烦了，个人思路如下：

​

```c++
#include <string>
#include <iostream>
using namespace std;

void swap(char& c1, char& c2)
{
        char tmp = c1;
        c1 = c2;
        c2 = tmp;
}

void rotateStr(string& str, int n)
{
        int length = str.length();
        for(int i = 0; i < length - n; i++)
        {
                swap(str[i], str[i + n]);
        }
}
int main()
{
        while(true)
        {
                string str;
                int n;
                cin >> str >> n; （所要旋转的字符串与旋转的位树）
                rotateStr(str, n);
                cout << str << endl;
        }
}
```

思路比较简单，而且步骤也比较简单。