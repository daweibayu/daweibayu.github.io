在搞男人八题之[An old Stone Game](http://poj.org/problem?id=1738)时竟没有思路了，其实问题不复杂，但是当数据过大时就没法用暴力解决了。感觉这题有点类似背包问题，但是发现竟然想不起背包问题的解决方案了，所以找个最简单的背包问题，先练练手。

题目在 [Charm Bracelet](http://poj.org/problem?id=3624)，具体如下：
```
Description
Bessie has gone to the mall's jewelry store and spies a charm bracelet. Of course, she'd like to fill it with the best charms possible from the N (1 ≤ N ≤ 3,402) available charms. Each charm i in the supplied list has a weight Wi (1 ≤ Wi ≤ 400), a 'desirability' factor Di (1 ≤ Di ≤ 100), and can be used at most once. Bessie can only support a charm bracelet whose weight is no more than M (1 ≤ M ≤ 12,880).

Given that weight limit as a constraint and a list of the charms with their weights and desirability rating, deduce the maximum possible sum of ratings.

Input
* Line 1: Two space-separated integers: N and M
* Lines 2..N+1: Line i+1 describes charm i with two space-separated integers: Wi and Di

Output
* Line 1: A single integer that is the greatest sum of charm desirabilities that can be achieved given the weight constraints

Sample Input
4 6
1 4
2 6
3 12
2 7

Sample Output
23
```

最简单的背包问题，一共 N 组数据，背包最大可以装 M 单位的物品，每组数据（物品）包含两部分，一部分是重量（W），一部分是价值（D），求背包最大可以装多少价值的东西。

这题其实在大学时期就已经 A 过了，昨天想搞这个题的时候，竟然发现完全忘了怎么搞了，也是悲催。静下心来，先自己慢慢推导出一个公式:

a(n) 表示第 n 项的 value，b(n) 表示第 n 项的 weight
f(n, m) 表示前 n 项中可以装的最大 value（weight 不超过 m）
则有：
![屏幕快照 2017-08-09 下午1.29.49.png](http://upload-images.jianshu.io/upload_images/2829180-b6b991a3b806d8d9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
根据公式写出如下代码：
```
int weight[4000];
int value[4000];

int getMax(int n, int m) {
    if (n == 0) {
        return 0;
    }
    int sum1 = getMax(n - 1, m);
    if (weight[n] > m) {
        return sum1;
    } else {
        int sum2 = getMax(n - 1, m - weight[n]) + value[n];
        return sum1 > sum2 ? sum1 : sum2;
    }
}

int main(int argc, const char * argv[]) {
    int n, m;
    scanf("%d %d", &n, &m);
    
    weight[0] = 0;
    value[0] = 0;
    for (int i = 1; i <= n; i++) {
        scanf("%d %d", &weight[i], &value[i]);
    }
    printf("%d\n", getMax(n, m));
}
```
这么写，逻辑没什么问题，但是感觉应该会超时，果不其然，`Time Limit Exceeded`。而且想了想，这种方式下确实没有比较好的剪枝方式，那就只能再换种方式了。
空间换时间，逐渐想到可以缓存前 n 项时背包的最大价值。这样当第 n + 1 组数据进来的时候，就只要去处理这个数组就可以了，时间复杂度最多也就是 n 的平方，应该是可以 hold 住的。随后又做了一些剪枝，具体代码如下：

```
#include <stdio.h>

#define max(x,y) (x > y ? x : y)

// total[n] 代表在重量 n 的时候，背包的最大价值
int total[13000];

int main(int argc, const char * argv[]) {
    
    int n, m;
    scanf("%d %d", &n, &m);

    total[0] = 0;
    
    int weight, value;
    int maxWeight = 0;
    
    for (int i = 1; i <= n; i++) {
        scanf("%d %d", &weight, &value);
        
        // 这里的 maxWeight 主要是用来剪枝，减少循环次数
        maxWeight = maxWeight + weight;
        if (maxWeight > m) {
            maxWeight = m;
        }
        
        // 注意，一定要倒着循环，为什么呢？客官可以想一下了
        for (int t = maxWeight; t >= weight; t--) {
            total[t] = max(total[t - weight] + value, total[t]);
        }
    }

    printf("%d", total[m]);
}
```