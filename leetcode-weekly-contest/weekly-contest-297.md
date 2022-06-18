---
description: https://leetcode.cn/contest/weekly-contest-297/
---

# Weekly Contest 297

ranking: 399 / 5915



[**2303. 计算应缴税款总额**](https://leetcode.cn/problems/calculate-amount-paid-in-taxes/)

{% hint style="info" %}
简单题，理解题目可以直接做
{% endhint %}

```
 public double calculateTax(int[][] brackets, int income) {
        double res = 0;
        int pre = 0;
        for (int[] bracket : brackets) {
            int min = Math.min(income, bracket[0]);
            double add = (min - pre) * (double) bracket[1] / 100;
            //System.out.println(add);
            res += add;
            pre = bracket[0];
            if (min == income) break;
        }
        return res;
    }
```



[**2304. 网格中的最小路径代价**](https://leetcode.cn/problems/minimum-path-cost-in-a-grid/)

{% hint style="info" %}
中等题,三重遍历考虑行、列、从前一列哪个来的

可以将二维dp降成一维dp
{% endhint %}

```
  public int minPathCost(int[][] grid, int[][] moveCost) {
        int m = grid.length, n = grid[0].length;
        int[][] dp = new int[m][n];

        for (int[] ints : dp) {
            Arrays.fill(ints, Integer.MAX_VALUE);
        }
        for (int i = 0; i < dp[0].length; i++) {
            dp[0][i] = grid[0][i];
        }

        for (int i = 1; i < grid.length; i++) {
            for (int j = 0; j < grid[0].length; j++) {
                for (int k = 0; k < grid.length; k++) {
                    dp[i][j] = Math.min(dp[i][j], dp[i - 1][k] + grid[i][j] + moveCost[grid[i - 1][k]][j]);
                }
            }
        }
        int res = Integer.MAX_VALUE;
        for (int i : dp[m - 1]) {
            res = Math.min(i, res);
        }
        return res;

    }
```



[**2305. 公平分发饼干**](https://leetcode.cn/problems/fair-distribution-of-cookies/)

{% hint style="info" %}
中等题，没看数据范围翻车了&#x20;

//2 <= cookies.length <= 8 也就说暴力即可&#x20;

//或 状态压缩+枚举子集
{% endhint %}

```
  //子集枚举+状态压缩
    public int distributeCookies(int[] cookies, int kk) {
        int[] sum = new int[1 << cookies.length];
        for (int i = 0; i < cookies.length; i++) {
            for (int j = 0, bit = 1 << i; j < bit; j++) {
                sum[bit | j] = sum[j] + cookies[i];
            }
        }

        int[] dp = Arrays.copyOf(sum, sum.length);

        for (int i = 1; i < kk; i++) {
            for (int j = (1<<cookies.length)-1; j >= 0; j--) {
                for (int k = j; k > 0; k = (k - 1) & j) {
                    dp[j]=Math.min(dp[j],Math.max(dp[j^k],sum[k]));
                }
            }
        }
        return dp[(1<<cookies.length)-1];
    }
```

```
//暴力dfs
public int distributeCookies(int[] cookies, int k) {
        dfs(cookies, 0, new int[cookies.length]);
        return res;
    }

    int res = Integer.MAX_VALUE;

    void dfs(int[] cookies, int index, int[] children) {

        if (index >= cookies.length) {
            int min = 0;
            for (int child : children) {
                min = Math.max(min, child);
            }
            res = Math.min(res, min);
            return;
        }
        for (int i = 0; i < children.length; i++) {
            children[i] += cookies[index];
            dfs(cookies, index + 1, children);
            children[i] -= cookies[index];
        }
    }
```



[**2306. 公司命名**](https://leetcode.cn/problems/naming-a-company/)

{% hint style="info" %}
困难题，周赛想了半天，最后好歹是做出来了

&#x20;//枚举首字母
{% endhint %}

```
 static public long distinctNames(String[] ideas) {

        Set<String> set = new HashSet<>(Arrays.asList(ideas));

        //dp[a][b] 表示a开头可以匹配到b的数目
        long[][] dp = new long[26][26];

        for (int i = 0; i < ideas.length; i++) {
            String s = ideas[i].substring(1);
            char c = ideas[i].charAt(0);
            for (int j = 0; j < 26; j++) {
                String ss = (char) ('a' + j) + s;
                if (!set.contains(ss)) {
                    dp[c - 'a'][j]++;
                }
            }
        }

        long res = 0;
        //最后的结果是 dp[a][b] * dp[b][a]
        //即 a开头可以换成b的  b开头可以换成a的 两两组合
        for (int i = 0; i < dp.length; i++) {
            for (int j = 0; j < dp.length; j++) {
                if (i == j) continue;
                res += dp[i][j] * dp[j][i];
            }
        }
        return res;

    }
```
