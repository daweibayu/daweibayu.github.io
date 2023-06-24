## 题目
[Cash Machine](http://poj.org/problem?id=1276)

```
Description

A Bank plans to install a machine for cash withdrawal. The machine is able to deliver appropriate @ bills for a requested cash amount. The machine uses exactly N distinct bill denominations, say Dk, k=1,N, and for each denomination Dk the machine has a supply of nk bills. For example, 

N=3, n1=10, D1=100, n2=4, D2=50, n3=5, D3=10 

means the machine has a supply of 10 bills of @100 each, 4 bills of @50 each, and 5 bills of @10 each. 

Call cash the requested amount of cash the machine should deliver and write a program that computes the maximum amount of cash less than or equal to cash that can be effectively delivered according to the available bill supply of the machine. 

Notes: 
@ is the symbol of the currency delivered by the machine. For instance, @ may stand for dollar, euro, pound etc. 
Input

The program input is from standard input. Each data set in the input stands for a particular transaction and has the format: 

cash N n1 D1 n2 D2 ... nN DN 

where 0 <= cash <= 100000 is the amount of cash requested, 0 <=N <= 10 is the number of bill denominations and 0 <= nk <= 1000 is the number of available bills for the Dk denomination, 1 <= Dk <= 1000, k=1,N. White spaces can occur freely between the numbers in the input. The input data are correct. 
Output

For each set of data the program prints the result to the standard output on a separate line as shown in the examples below. 
Sample Input

735 3  4 125  6 5  3 350
633 4  500 30  6 100  1 5  0 1
735 0
0 3  10 100  10 50  10 10
Sample Output

735
630
0
0
```
依然是背包问题，具体的输入为 `cash N n1 D1 n2 D2 ... nN DN`
cash 表示现金的最大值
N 代表具体有 N 种面值的钞票
ni Di 分别代表每种钞票的数量与价值

输出：输出小于等于 cash 的最大金额


## 代码
不做过多解释，与 [Poj 1742](http://www.jianshu.com/p/51898d991a68) 基本认为是一道题，直接 copy 过来，稍作修改，一次 A 过

```
#include <stdio.h>
#include <iostream>
#include <bitset>
using namespace std;

int main(int argc, const char * argv[]) {
    
    int cash, type, coinValue, coinNumber;
    bitset<100005> resultBitSet;
    
    while (scanf("%d", &cash) != EOF) {
        scanf("%d", &type);
        
        if (0 == type) {
            printf("0\n");
            continue;
        }
        
        resultBitSet.reset();
        
        for (int i = 0; i < type; i++) {
            scanf("%d %d",  &coinNumber, &coinValue);
            for (int t = 1; t <= coinNumber && t * coinValue <= cash; t++) {
                resultBitSet |= (resultBitSet << coinValue);
                resultBitSet.set(t * coinValue);
            }
        }
        
        int result = 0;
        for (int i = cash; i >= 1; i--) {
            if (resultBitSet.test(i)) {
                result = i;
                break;
            }
        }
        printf("%d\n", result);
    }
}
```

请叫我 bitset 小王子~~