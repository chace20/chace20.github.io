---
title: LeetCode题解-Best Time to Buy and Sell Stock
date: 2019-09-09 19:46:04
tags: 算法
---
对LeetCode上遇到的股票问题做一个总结，涉及到6道题：121, 122, 123, 188, 309, 714

题目地址：[Best Time to Buy and Sell Stock](https://leetcode.com/problemset/all/?search=Best%20Time%20to%20Buy%20and%20Sell%20Stock)

这些题的总体要求都是**根据每天的股价涨跌来决定买卖股票，使得总体的利润最大**，但是对于不同题有不同的限定条件，难度也是越来越大，其中4道题都是DP求解。

## 121(Easy)
题目：最多进行一次买卖。求最大利润。

这个就很简单了，直接找到最低价格买入，再在最高价格卖出就OK了。
可以用暴力的方法去进行两层遍历，先在外层循环找波谷再在内层循环找波峰，复杂度O(N^2)；
另一种方法是**记录波谷的价格和最大利润，在遍历过程中更新波谷和最大利润就OK了**，复杂度O(N)
``` java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length < 1) return 0;
        
        int minprice = Integer.MAX_VALUE;
        int maxprofit = 0;
        for (int p: prices) {
            if (p < minprice)
                minprice = p;
            else if (p - minprice > maxprofit)
                maxprofit = p - minprice;
        }
        return maxprofit;
    }
}
```

## 122(Easy)
题目：可以进行任意次买卖。求最大利润。

这题的思路就是**每次波谷买入，每次波峰卖出就可以**。PS：在188题还会用到这种解法。

``` java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length < 1) return 0;
        // 寻找波谷和波峰，每次都在波谷买入，波峰卖出就可以让利润最大
        int i = 0;
        int valley = prices[0];
        int peak = prices[0];
        int maxprofit = 0;
        while (i < prices.length-1){
            // 寻找波谷买入
            while (i < prices.length-1 && prices[i] >= prices[i+1])
                i++;
            valley = prices[i];
            // 寻找波峰卖出
            while (i < prices.length-1 && prices[i] <= prices[i+1])
                i++;
            peak = prices[i];
            maxprofit += peak - valley;
        }
        return maxprofit;
    }
}

// 方法2
// 这种方法是只要能赚钱就计算利润，也就是类比于：在这一天卖出再买入就可以得到利润
// 跟上面的方法的结果是一样的
/*
class Solution {
    public int maxProfit(int[] prices) {
        int maxprofit = 0;
        for (int i = 1; i < prices.length; i++) {
            if (prices[i] > prices[i - 1])
                maxprofit += prices[i] - prices[i - 1];
        }
        return maxprofit;
    }
}
*/
```

## 123(Hard)
题目：最多进行两次买卖。求最大利润。

看起来题目跟121差不多，但是难度却大了很多。

``` java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length < 1) return 0;
        
        int sell1 = 0, sell2 = 0;
        int buy1 = Integer.MIN_VALUE, buy2 = Integer.MIN_VALUE;
        for (int p : prices) {
            // 第一次买，让借的钱尽可能少。因为是负数，所以也用Math.max
            // 这里可以看成Math.max(buy1, 0-p)，这样跟下面的buy2就统一了
            buy1 = Math.max(buy1, -p);
            // 第一次卖，让利润尽量多，因为buy1是负数，所以还是取p+buy1
            sell1 = Math.max(sell1, buy1+p);
            // 第二次买，让第一次卖的利润跟当前价格之差最大，即让买完以后利润依然最大
            buy2 = Math.max(buy2, sell1-p);
            // 第二次卖，让利润最大
            sell2 = Math.max(sell2, buy2+p);
        }
        return sell2;
    }
}
```

## 188(Hard)
题目：最多进行k次买卖。求最大利润。

在123题基础上又加大难度了。

``` java
class Solution {
    public int maxProfit(int k, int[] prices) {
        if (prices == null || prices.length < 1) return 0;
        
        if (prices.length/2 <= k) {
            // 可以随便交易，所以变成了122题
            simpleProfit(prices)
        }
        
        int[][] sell = new int[k + 1][prices.length];
        for (int i = 1; i <= k; i++) {
            int buy = -prices[0];
            // 仔细看会发现这里每一次交易的解法跟123题是一样的
            for (int j = 1; j < prices.length; j++) {
                // 这里buy可以用一个临时变量替换，只对当前这次i有用
                // 但是sell不能弄成一维的，因为会引用到sell[i-1][j-1]
                // 如果只用一维的，那么每次的sell[i-1]都只能是这次i的
                buy = Math.max(buy, sell[i - 1][j - 1] - prices[j]);
                sell[i][j] = Math.max(sell[i][j - 1], buy + prices[j]);
            }
        }
        return sell[k][prices.length-1];
    }
    
    // 这里的解法就是122题的解法
    public int simpleProfit(int[] prices) {
        int result = 0;
        for (int i=1; i<prices.length; i++) {
            if (prices[i] > prices[i-1]) {
                result += prices[i] - prices[i-1];
            }
        }
        return result;
    }
}
```

## 309(Medium)
题目：在卖出以后必须冷却1天才能再次买入。求最大利润。

有了前面和buy和sell，这道题要解决起来也比较容易。
**冷却1天可以用`buy[i]=sell[i-2]-prices[i]`来表达**。

``` java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices == null || prices.length < 1) return 0;

        // 为了节省空间，没有用数组，而是直接用的buy和sell
        int sell = 0, prev_sell = 0, buy = -prices[0];
        for (int i=1; i< prices.length; i++) {
            // 今天没有买；或者今天买了，注意buy[i]=sell[i-2]-prices[i]
            buy = Math.max(buy, prev_sell-prices[i]);
            // 只有赋值放在这里prev_cash才是cash[i-2]
            prev_sell = sell;
            // 今天没有卖；或者今天卖了
            sell = Math.max(sell, buy+prices[i]);
        }
        return sell;
    }
}
```

## 714(Medium)
题目：卖出的时候有手续费。求最大利润。

**计算的时候把手续费考虑进来：`sell[i]=buy[i-1]+prices[i]-fee`。**

``` java
class Solution {
    public int maxProfit(int[] prices, int fee) {
        if (prices == null || prices.length < 1) return 0;

        int sell = 0, buy = -prices[0];
        for (int i = 1; i < prices.length; i++) {
            // 今天没有买；或者今天买了
            buy = Math.max(buy, sell - prices[i]);
            // 今天没有卖；或者今天卖了，注意有手续费
            sell = Math.max(sell, buy + prices[i] - fee);
        }
        return sell;
    }
}
```
