---
description: https://leetcode-cn.com/contest/weekly-contest-286/
---

# Weekly Contest 286

### [**2215. 找出两数组的不同**](https://leetcode-cn.com/problems/find-the-difference-of-two-arrays/) **** :star::star:

> 给你两个下标从 0 开始的整数数组 nums1 和 nums2 ，请你返回一个长度为 2 的列表 answer ，其中：
>
> answer\[0] 是 nums1 中所有 不 存在于 nums2 中的 不同 整数组成的列表。 answer\[1] 是 nums2 中所有 不 存在于 nums1 中的 不同 整数组成的列表。 注意：列表中的整数可以按 任意 顺序返回。
>
> **提示：**
>
> * `1 <= nums1.length, nums2.length <= 1000`
> * `-1000 <= nums1[i], nums2[i] <= 1000`

{% hint style="info" %}
简单题，就是说找出存在于一个数组，且不存于另一个数组的数，且不能含有重复数。因为涉及查找和去重，用set模拟下即可。
{% endhint %}

```
   public List<List<Integer>> findDifference(int[] nums1, int[] nums2) {
        Set<Integer> set1 = new HashSet<>();
        for (int i : nums1) {
            set1.add(i);
        }
        Set<Integer> set2 = new HashSet<>();
        for (int i : nums2) {
            set2.add(i);
        }

        List<Integer> l1 = new ArrayList<>();
        for (Integer integer : set1) {
            if (!set2.contains(integer)) {
                l1.add(integer);
            }
        }

        List<Integer> l2 = new ArrayList<>();

        for (Integer integer : set2) {
            if (!set1.contains(integer)) {
                l2.add(integer);
            }
        }
        List<List<Integer>> res = new ArrayList<>();
        res.add(l1);
        res.add(l2);
        return res;
    }
```

### [**2216. 美化数组的最少删除数**](https://leetcode-cn.com/problems/minimum-deletions-to-make-array-beautiful/) **** :star::star:

> 给你一个下标从 0 开始的整数数组 nums ，如果满足下述条件，则认为数组 nums 是一个 美丽数组 ：
>
> nums.length 为偶数 对所有满足 i % 2 == 0 的下标 i ，nums\[i] != nums\[i + 1] 均成立 注意，空数组同样认为是美丽数组。
>
> 你可以从 nums 中删除任意数量的元素。当你删除一个元素时，被删除元素右侧的所有元素将会向左移动一个单位以填补空缺，而左侧的元素将会保持 不变 。
>
> 返回使 nums 变为美丽数组所需删除的 最少 元素数目。
>
> **提示：**
>
> * `1 <= nums.length <= 1e5`
> * `0 <= nums[i] <= 1e5`

{% hint style="info" %}
中等题，典型的贪心。遇到与后面相等的就赶紧删除。

注意删除元素后，后面元素的坐标会发生变化，计算坐标的时候要减去前面删除的元素个数。
{% endhint %}

```
 static public int minDeletion(int[] nums) {
        //空数组返回
        if (nums.length == 0) return 0;
        int del = 0;

        for (int i = 0; i < nums.length-1; i++) {
            if ((i - del) % 2 == 0 && nums[i] == nums[i+1]) {
                del++;
            }
        }
      //判断最后奇数要+1
        return (nums.length - del) % 2 == 0 ? del : del + 1;
    }
```

### [**2217. 找到指定长度的回文数**](https://leetcode-cn.com/problems/find-palindrome-with-fixed-length/) **** :star::star:

> 给你一个整数数组 queries 和一个 正 整数 intLength ，请你返回一个数组 answer ，其中 answer\[i] 是长度为 intLength 的 正回文数 中第 queries\[i] 小的数字，如果不存在这样的回文数，则为 -1 。
>
> 回文数 指的是从前往后和从后往前读一模一样的数字。回文数不能有前导 0 。
>
> **提示：**
>
> * `1 <= queries.length <= 5 * 1e4`
> * `1 <= queries[i] <= 109`
> * `1 <= intLength <= 15`

{% hint style="info" %}
中等题，周赛一开始想的非常复杂，类似于求第k大的数那种思想，用递归来做。从1XX..XX1开始判断，判断此长度最多有多少个回文串，如有\<query就进入下一层递归 11X..X11，如果>query就开始判断2XX..XX2。依次类推。

然而其实有更简单的办法，只需要看回文串的左半边，其实是依次递增的一个数列，这样直接+query就是要找的回文串，然后逆序拼接上右半边即可。
{% endhint %}

```
   //递归做法，类似于求第k大
   public long[] kthPalindrome(int[] queries, int intLength) {
        long[] res = new long[queries.length];
        for (int i = 0; i < queries.length; i++) {
            res[i] = Long.parseLong(helper(intLength, queries[i], true));
        }
        return res;
    }

   //统计len长度个数
    long count(int len) {
        if (len == 0) return 1;
        int c = 0;
        if (len % 2 == 0) {
            c = len / 2;
        } else {
            c = (len / 2) + 1;
        }
        long res = 1;
        for (int i = 0; i < c; i++) {
            res *= 10;
        }
        return res;
    }
     //egge用来表示是不是边界，边界位置不能从0开始
    String helper(int intLength, int query, boolean edge) {
        //特殊判断intLength为1和0的情况
        if (intLength == 0) {
            return "";
        }
        if (intLength == 1) {
            if (!edge) {
                if (query > 10) return "-1";
                return query - 1 + "";

            } else {
                if (query >= 10) return "-1";
                return query + "";

            }
        }
        //判断是不是边界 边界不能是0啊
        int i = 0;
        if (edge) i++;
        for (; i <= 9; i++) {
            long c = count(intLength - 2);
            if (query <= c) {//表示query的数在下一个递归中，进入
                return i + helper(intLength - 2, query, false) + i;
            }
            query -= c;
        }
        return "-1";
    }
```

```
//数学方法
     static public long[] kthPalindrome(int[] queries, int intLength) {
        long[] res = new long[queries.length];
        for (int i = 0; i < queries.length; i++) {
            long num = (long) Math.pow(10, (intLength - 1) / 2);
            num += queries[i] - 1;
            StringBuilder stringBuilder = new StringBuilder(num + "");
            if (intLength % 2 == 1 ? stringBuilder.length() * 2 - 1 > intLength : stringBuilder.length() * 2 > intLength) {
                res[i] = -1;
                continue;
            }
            if (intLength % 2 == 0) {
                res[i] = Long.parseLong(stringBuilder.toString() + stringBuilder.reverse());
            } else {
                res[i] = Long.parseLong(stringBuilder.toString() + stringBuilder.reverse().substring(1));
            }
        }
        return res;
    }
```

### [**2218. 从栈中取出 K 个硬币的最大面值和**](https://leetcode-cn.com/problems/maximum-value-of-k-coins-from-piles/)  ****  :star::star::star:****

> 一张桌子上总共有 n 个硬币 栈 。每个栈有 正整数 个带面值的硬币。
>
> 每一次操作中，你可以从任意一个栈的 顶部 取出 1 个硬币，从栈中移除它，并放入你的钱包里。
>
> 给你一个列表 piles ，其中 piles\[i] 是一个整数数组，分别表示第 i 个栈里 从顶到底 的硬币面值。同时给你一个正整数 k ，请你返回在 恰好 进行 k 次操作的前提下，你钱包里硬币面值之和 最大为多少 。
>
> **提示：**
>
> * `n == piles.length`
> * `1 <= n <= 1000`
> * `1 <= piles[i][j] <= 1e5`
> * `1 <= k <= sum(piles[i].length) <= 2000`

{% hint style="info" %}
困难题，相对是不难的背包dp，dp\[i]\[j]，表示前i堆取j个能取到的最大值。三层遍历解决问题：

&#x20;  for  i   in piles：

&#x20;     for   j  in  k：

&#x20;         for  l  in this  pile：
{% endhint %}

```
  static public int maxValueOfCoins(List<List<Integer>> piles, int k) {
        int n = piles.size();
        int[][] dp = new int[n + 1][k + 1];
        //i表示第几堆  j表示k的枚举  l表示从该堆里取几个
        for (int i = 0; i < piles.size(); i++) {
            List<Integer> pile = piles.get(i);
            //前缀和
            for (int j = 1; j < pile.size(); j++) {
                pile.set(j, pile.get(j) + pile.get(j - 1));
            }
            for (int j = 0; j < k; j++) {
                //继承之前的
                dp[i + 1][j + 1] = dp[i][j + 1];
                for (int  l= 0; l < pile.size(); l++) {
                    if (l > j) break;
                    //当总共取j个时，在该堆取l个
                    dp[i + 1][j + 1] = Math.max(dp[i + 1][j + 1], dp[i][j - l] + pile.get(l));
                }
            }
        }
        return dp[n][k];
    }
```
