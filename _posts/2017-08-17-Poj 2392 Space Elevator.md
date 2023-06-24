## 题目
[Space Elevator](http://poj.org/problem?id=2392)

```
Description
The cows are going to space! They plan to achieve orbit by building a sort of space elevator: a giant tower of blocks. They have K (1 <= K <= 400) different types of blocks with which to build the tower. Each block of type i has height h_i (1 <= h_i <= 100) and is available in quantity c_i (1 <= c_i <= 10). Due to possible damage caused by cosmic rays, no part of a block of type i can exceed a maximum altitude a_i (1 <= a_i <= 40000). 

Help the cows build the tallest space elevator possible by stacking blocks on top of each other according to the rules.
Input

* Line 1: A single integer, K 

* Lines 2..K+1: Each line contains three space-separated integers: h_i, a_i, and c_i. Line i+1 describes block type i.
Output

* Line 1: A single integer H, the maximum height of a tower that can be built

Sample Input
3
7 40 3
5 23 8
2 52 6

Sample Output
48
```
依然是背包问题，一共 k 种类型的砖块，分别给出每种砖块的高度、最大可以放置的高度、数量（例如第一组数据，分别为7 40 3）。
输出：输出可以垒起的最大高度


## 代码
不做过多解释，与 [Poj 1742](http://www.jianshu.com/p/51898d991a68) 类似，稍作修改，排序，然后依然 bitset A 掉。

```
#include <stdio.h>
#include <iostream>
#include <bitset>
#include <algorithm>
using namespace std;

struct Block {
    int height;
    int number;
    int maxAltitude;
};

bool compare(Block a,Block b) {
    return a.maxAltitude < b.maxAltitude;
}

int main(int argc, const char * argv[]) {
    
    int k;
    Block blockList[405];
    bitset<40005> resultBitSet;
    
    scanf("%d", &k);
    for (int i = 0; i < k; i++) {
        scanf("%d %d %d", &blockList[i].height, &blockList[i].maxAltitude, &blockList[i].number);
    }
    
    sort(blockList, blockList + k, compare);
    resultBitSet.reset();
    
    int shiftNumber = 0;
    for (int i = 0; i < k; i++) {
        for (int t = 1; t <= blockList[i].number && t * blockList[i].height <= blockList[i].maxAltitude; t++) {
            resultBitSet |= (resultBitSet << blockList[i].height);
            resultBitSet.set(t * blockList[i].height);
            
            //以下三行比较重要，目的就是清除超过 maxAltitude 的数据
            shiftNumber = 40005 - (blockList[i].maxAltitude + 1);
            resultBitSet <<= shiftNumber;
            resultBitSet >>= shiftNumber;
        }
    }
    
    int result = 0;
    for (int i = blockList[k - 1].maxAltitude; i >= 1; i--) {
        if (resultBitSet.test(i)) {
            result = i;
            break;
        }
    }
    printf("%d\n", result);
}
```