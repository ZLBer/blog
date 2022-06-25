---
description: https://leetcode.cn/contest/weekly-contest-298/
---

# Weekly Contest 298

ranking: 1590 / 6228 行,在菜的路上一去不复返



### [**2309. 兼具大小写的最好英文字母**](https://leetcode.cn/problems/greatest-english-letter-in-upper-and-lower-case/)

{% hint style="info" %}
简单题，统计频率即可
{% endhint %}

```
   public String greatestLetter(String s) {
        int[] count = new int[26];
        int[] count1 = new int[26];
        for (char c : s.toCharArray()) {
            if (Character.isUpperCase(c)) {
                count[c - 'A']++;
            } else {
                count1[c - 'a']++;
            }
        }
        String res = "";
        for (int i = count.length - 1; i >= 0; i--) {
            if (count[i] > 0 && count1[i] > 0) {
                res = ((char) (i + 'A')) + "";
                break;
            }
        }
        return res;
    }
```



### [**2310. 个位数字为 K 的整数之和**](https://leetcode.cn/problems/sum-of-numbers-with-units-digit-k/)

{% hint style="info" %}
中等题，想太多了，只考虑个位数出现几次就行啊，还是种贪心策略
{% endhint %}

```
  public int minimumNumbers(int num, int k) {
        if (num == 0) return 0;
        if (k == 0) return num % 10 == 0 ? 1 : -1;
        int count = 0;
        while (num >= 0) {
            if ((num - k) % 10 != 0) {
                num -= k;
                count++;
            } else {
                if (num != 0) count++;
                break;
            }
        }
        return num > 0 ? count : -1;
    }
```



### [**2311. 小于等于 K 的最长二进制子序列**](https://leetcode.cn/problems/longest-binary-subsequence-less-than-or-equal-to-k/)

{% hint style="info" %}
//中等题，这个题也是做的不好

&#x20;//我的想法是：依次检查k长度的字符串，k前面的1删掉，k里面的要删掉之后比k小，k后面的全部删掉
{% endhint %}

```
  public int longestSubsequence(String s, int k) {
        String binaryString = Integer.toBinaryString(k);
        if (binaryString.length() > s.length()) return s.length();

        int n = s.length();

        int pre = 0;
        int ans = Integer.MAX_VALUE;
        for (int i = 0; i + binaryString.length() <= n; i++) {

            int count = 0;
            boolean flag = false;
            for (int j = 0; j < binaryString.length(); j++) {
                if (binaryString.charAt(j) > s.charAt(i + j)) {
                    break;
                } else if (binaryString.charAt(j) < s.charAt(i + j)) {
                    count++;
                    if (flag) break; //必定有前缀1相等，此时删除需要后移，必定比k小了，所以直接返回
                } else {
                    flag = true;
                    continue;
                }
            }

            ans = Math.min(ans, pre + count + (s.length() - i - binaryString.length()));
            //             System.out.println(pre+" "+count);

            if (s.charAt(i) == '1') pre++;
        }
        return s.length() - ans;
    }
```

```
    //超级贪心，只看m长度的后缀和m-1长度的后缀
 /*   public int longestSubsequence(String s, int k) {
        int n = s.length(), m = 32 - Integer.numberOfLeadingZeros(k);
        if (n < m) return n;
        int ans = Integer.parseInt(s.substring(n - m), 2) <= k ? m : m - 1;
        return ans + (int) s.substring(0, n - m).chars().filter(c -> c == '0').count();
    }*/
```

### [**2312. 卖木头块**](https://leetcode.cn/problems/selling-pieces-of-wood/)

{% hint style="info" %}
困难题目，要注意是一刀切
{% endhint %}

```
  public long sellingWood(int m, int n, int[][] prices) {
        long[][] dp = new long[m + 1][n + 1];

        for (int[] price : prices) {
            dp[price[0]][price[1]] = price[2];
        }
        for (int i = 0; i <= m; i++) {
            for (int j = 0; j <= n; j++) {
                for (int k = 1; k <= i; k++) {
                    dp[i][j]=Math.max(dp[i][j],dp[k][j]+dp[i-k][j]);
                }

                for (int k = 1; k <= j; k++) {
                    dp[i][j]=Math.max(dp[i][j],dp[i][k]+dp[i][j-k]);
                }
            }
        }
        return dp[m][n];
    }
```

