## 题目
Poj 3211 [Washing Clothes](http://poj.org/problem?id=3211)
```
Description
Dearboy was so busy recently that now he has piles of clothes to wash. Luckily, he has a beautiful and hard-working girlfriend to help him. The clothes are in varieties of colors but each piece of them can be seen as of only one color. In order to prevent the clothes from getting dyed in mixed colors, Dearboy and his girlfriend have to finish washing all clothes of one color before going on to those of another color.

From experience Dearboy knows how long each piece of clothes takes one person to wash. Each piece will be washed by either Dearboy or his girlfriend but not both of them. The couple can wash two pieces simultaneously. What is the shortest possible time they need to finish the job?

Input
The input contains several test cases. Each test case begins with a line of two positive integers M and N (M < 10, N < 100), which are the numbers of colors and of clothes. The next line contains M strings which are not longer than 10 characters and do not contain spaces, which the names of the colors. Then follow N lines describing the clothes. Each of these lines contains the time to wash some piece of the clothes (less than 1,000) and its color. Two zeroes follow the last test case.

Output
For each test case output on a separate line the time the couple needs for washing.


Sample Input
3 4
red blue yellow
2 red
3 blue
4 blue
6 red
0 0

Sample Output
10
```
大意就是题目的主人公与其女票一起洗衣服，两个人可以一起洗（双线程），但是衣服有很多种颜色，为了不让衣服色混，所以俩人必须要同时洗一种颜色的衣服，如果有一个人还没有洗完，另一个人就只能等待，直到俩人都洗完，才能洗下一个颜色的衣服，计算最小的耗时。

## 解题思路
1. 因为颜色不能混，所以其实也就是不同颜色之间的衣服互不干扰，计算每种颜色的耗时，累加得到计算结果。
2. 计算每种颜色的时候，可以转化为背包问题。因为是双线程，如果单人总耗时为 total 的话，两个人中肯定有一个的耗时是 >= total/2 的，所以可以理解为容量为 total/2 的背包，计算背包的最大重量，然后 “total - 最大重量” 就是单色的耗时了。

## 代码
```
#include <stdio.h>
#include <iostream>
#include <bitset>
#include <cstring>
#include <algorithm>

using namespace std;

char colorList[10][15];
int getColorIndex(int colorNum, char* color) {
    for (int i = 0; i < colorNum; i++) {
        bool isEqual = true;
        for (int t = 0; t < 15; t++) {
            if (color[t] != colorList[i][t]) {
                isEqual = false;
                break;
            } else if (color[t] == '\0') {
                break;
            }
        }
        if (isEqual) {
            return i;
        }
    }
    return 0;
}

struct ClothePile {
    int time;
    int index;
};
ClothePile clothePile[1005];

bool compare(ClothePile a,ClothePile b) {
    if (a.index != b.index) {
        return a.index < b.index;
    } else {
        return a.time < b.time;
    }
}

// 计算每种颜色所耗费的时间，[start, end)，左闭右开，代表一个颜色衣服在 clothePile 中的索引
int calculateOneColor(int start, int end, int pileNum) {
    int total = 0;
    for (int i = start; i < end; i++) {
        total += clothePile[i].time;
    }
    int maxTime = total / 2;
    
    bitset<100005> resultBitSet(0);
    
    for (int i = start; i < end; i++) {
        resultBitSet |= (resultBitSet << clothePile[i].time);
        resultBitSet.set(clothePile[i].time);
    }
        
    int result = 0;
    for (int i = maxTime; i >= 1; i--) {
        if (resultBitSet.test(i)) {
            result = i;
            break;
        }
    }
    return total - result;
}

// 给定 clothePile （有序），然后计算每种颜色衣服耗费的时间，累加得到结果并返回
int calculateResult(int pileNum) {
    
    // 排序
    sort(clothePile, clothePile + pileNum, compare);
    
    int start = 0, end = 0;
    int total = 0;
    for (int i = 1; i <= pileNum; i++) {
        if (i == pileNum || clothePile[i].index != clothePile[i - 1].index) {
            end = i;
            total += calculateOneColor(start, end, pileNum);
            start = i;
        }
    }
    return total;
}

// 处理输入、格式化数据、计算、输出
bool processInput() {
    
    // 输入
    int colorNum, pileNum;
    scanf("%d %d", &colorNum, &pileNum);
    if (colorNum == 0 && pileNum == 0) {
        return false;
    }
    
    for (int i = 0; i < colorNum; i++) {
        scanf("%s", colorList[i]);
    }
    
    int time;
    char closeColor[15] = {0};
    for(int i = 0; i < pileNum; i++) {
        scanf("%d %s", &time, closeColor);
        // 格式化数据，因为 string 比较难处理，所以这里把字符串映射到 colorList 的 index 上
        clothePile[i].index = getColorIndex(colorNum, closeColor);
        clothePile[i].time = time;
    }
    
    // 计算并输出
    printf("%d\n", calculateResult(pileNum));
    return true;
}

int main(int argc, const char * argv[]) {
    while (processInput());
    return 0;
}
```