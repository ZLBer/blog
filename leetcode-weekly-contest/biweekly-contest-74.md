---
description: https://leetcode-cn.com/contest/biweekly-contest-74/
---

# Biweekly Contest 74

### [**2206. 将数组划分成相等数对**](https://leetcode-cn.com/problems/divide-array-into-equal-pairs/) **** :star:****

> 给你一个整数数组 nums ，它包含 2 \* n 个整数。
>
> 你需要将 nums 划分成 n 个数对，满足：
>
> 每个元素 只属于一个 数对。 同一数对中的元素 相等 。 如果可以将 nums 划分成 n 个数对，请你返回 true ，否则返回 false 。
>
> **提示：**
>
> * `nums.length == 2 * n`
> * `1 <= n <= 500`
> * `1 <= nums[i] <= 500`

{% hint style="info" %}
简单题，判断每个数出现的频率是不是2的倍数，因为nums\[i]<=500，所以开个500的数组即可。
{% endhint %}

```
      public boolean divideArray(int[] nums) {
     int []count=new int[501];
        for (int num : nums) {
            count[num]++;
        }
        for (int i : count) {
            if(i%2!=0) return false;
        }
        return true;
    }

```

### [**2207. 字符串中最多数目的子字符串**](https://leetcode-cn.com/problems/maximize-number-of-subsequences-in-a-string/) **** :star::star:

> 给你一个下标从 0 开始的字符串 text 和另一个下标从 0 开始且长度为 2 的字符串 pattern ，两者都只包含小写英文字母。
>
> 你可以在 text 中任意位置插入 一个 字符，这个插入的字符必须是 pattern\[0] 或者 pattern\[1] 。注意，这个字符可以插入在 text 开头或者结尾的位置。
>
> 请你返回插入一个字符后，text 中最多包含多少个等于 pattern 的 子序列 。
>
> 子序列 指的是将一个字符串删除若干个字符后（也可以不删除），剩余字符保持原本顺序得到的字符串。
>
> **提示：**
>
> * `1 <= text.length <= 1e5`
> * `pattern.length == 2`
> * `text` 和 `pattern` 都只包含小写英文字母。

{% hint style="info" %}
中等题，简单的贪心，将pattern的字符放在收尾是最优的解法，接下来就是遍历求子序列的个数即可。
{% endhint %}

```
  //一次遍历
    public long maximumSubsequenceCount(String text, String pattern) {
        int a=0,b=0;
        char as=pattern.charAt(0),bs=pattern.charAt(1);
        long sum=0;
        for (char c : text.toCharArray()) {
            if(c==bs){
                b++;
                sum+=a;
            }
            if(c==as) {
                a++;
            }
        }
        return sum+Math.max(a,b);
    }
    //周赛两次遍历写的
   /* public long maximumSubsequenceCount(String text, String pattern) {
        char[] chars = pattern.toCharArray();
        long a = 0, b = 0;
        int n = text.length();
        for (int i = 0; i < n; i++) {
            char c = text.charAt(i);
            if (c == chars[0]) {
                a++;
            } else if (c == chars[1]) {
                b++;
            }
        }
        //当两个字符相等，无所谓插入位置
        if (chars[0] == chars[1]) {
            return a * (a + 1) / 2;
        }
        //贪心，只有添加最开始或最结束的位置是最多的 ,且是pattern中最多的字符
        long res = Math.max(a, b);
        //判断有多少个=pattern的子序列
        for (int i = 0; i < n; i++) {
            char c = text.charAt(i);
            if (c == chars[0]) {
                res += b;
            } else if (c == chars[1]) {
                b--;
            }
        }
        return res;
    }*/

```

### [**2208. 将数组和减半的最少操作次数**](https://leetcode-cn.com/problems/minimum-operations-to-halve-array-sum/) **** :star::star:

> 给你一个正整数数组 nums 。每一次操作中，你可以从 nums 中选择 任意 一个数并将它减小到 恰好 一半。（注意，在后续操作中你可以对减半过的数继续执行操作）
>
> 请你返回将 nums 数组和 至少 减少一半的 最少 操作数。
>
> **提示：**
>
> * `1 <= nums.length <= 1e5`
> * `1 <= nums[i] <= 1e7`

{% hint style="info" %}
中等题，没太搞懂这个题的意义，就简单的用个优先队列每次取最大的砍一半，当时做的时候以为会有精度问题，但实际提交过了。后面看别人将每个数左移放大，然后再砍半，防止出现精度问题。
{% endhint %}

```
   public int halveArray(int[] nums) {
        long sum = 0;
        PriorityQueue<Double> priorityQueue = new PriorityQueue<>(Comparator.reverseOrder());

        for (int num : nums) {
            sum += num;
            priorityQueue.add((double) num);
        }
        double half = (double) sum / 2;
        double temp = 0;
        int count = 0;
        while (temp < half) {
            Double poll = priorityQueue.poll();
            temp += poll / 2;
            priorityQueue.add(poll / 2);
            count++;
        }
        return count;

    }
```

### [**2209. 用地毯覆盖后的最少白色砖块**](https://leetcode-cn.com/problems/minimum-white-tiles-after-covering-with-carpets/) **** :star::star::star:

> 给你一个下标从 0 开始的 二进制 字符串 floor ，它表示地板上砖块的颜色。
>
> floor\[i] = '0' 表示地板上第 i 块砖块的颜色是 黑色 。 floor\[i] = '1' 表示地板上第 i 块砖块的颜色是 白色 。 同时给你 numCarpets 和 carpetLen 。你有 numCarpets 条 黑色 的地毯，每一条 黑色 的地毯长度都为 carpetLen 块砖块。请你使用这些地毯去覆盖砖块，使得未被覆盖的剩余 白色 砖块的数目 最小 。地毯相互之间可以覆盖。
>
> 请你返回没被覆盖的白色砖块的 最少 数目。
>
> **提示：**
>
> * `1 <= carpetLen <= floor.length <= 1000`
> * `floor[i]` 要么是 `'0'` ，要么是 `'1'` 。
> * `1 <= numCarpets <= 1000`

{% hint style="info" %}
困难提，不太难的动态规划，dp(i,j)表示用i快地毯时，前j块地板白色砖块的数目。

dp(0,j)表示有地毯，求白色砖块的前缀和

dp(i,j)表示有i块地毯:

&#x20;      要么选择从j位置往前铺， dp(i-1,j-地毯长度)，即用i-1地毯, j-地毯长度时，白砖的最小数目。

&#x20;      要么选择不在j位置铺，dp(i,j-1)+判断i位置是不是白色砖，即前面位置白砖数+当前位置的白砖数。

对dp还是不熟悉，转移方程还是要考虑很久。
{% endhint %}

```
   static public int minimumWhiteTiles(String floor, int numCarpets, int carpetLen) {
        char[] floors = floor.toCharArray();
        int n = floor.length();
        //直接返回可以全部覆盖的情况
        if (numCarpets * carpetLen >= n) {
            return 0;
        }

        int[][] dp = new int[numCarpets + 1][n + 1];
        for (int[] ints : dp) {
            Arrays.fill(ints, (int) 1e9 + 7);
        }
        //没有地板初试成0
        for (int i = 0; i < dp.length; i++) {
            dp[i][0] = 0;
        }
        //当没有地毯的时候计算前缀和
        for (int i = 1; i <= n; i++) {
            if (floors[i - 1] == '1') {
                dp[0][i] = dp[0][i - 1] + 1;
            } else {
                dp[0][i] = dp[0][i - 1];
            }
        }
        //从一个地毯开始遍历
        for (int i = 1; i < dp.length; i++) {
            for (int j = 1; j < dp[0].length; j++) {
                //小于地毯的长度直接可以全覆盖
                if (j < carpetLen) {
                    dp[i][j] = 0;
                } else {
                    //判断 在这个地方往前铺  和  不在这个地方铺(不在这个地方铺是要判断地板颜色) 哪个更小
                    dp[i][j] = Math.min(dp[i][j - 1] + (floors[j - 1] == '1' ? 1 : 0), dp[i - 1][j - carpetLen]);
                }
            }
        }
        return dp[numCarpets][n];
    }
```
