---
description: https://leetcode-cn.com/contest/biweekly-contest-75/
---

# Biweekly Contest 75

Ranking: 534/4335

### [**2220. 转换数字的最少位翻转次数**](https://leetcode-cn.com/problems/minimum-bit-flips-to-convert-number/) **** :star::star:****

> 一次 位翻转 定义为将数字 x 二进制中的一个位进行 翻转 操作，即将 0 变成 1 ，或者将 1 变成 0 。
>
> 比方说，x = 7 ，二进制表示为 111 ，我们可以选择任意一个位（包含没有显示的前导 0 ）并进行翻转。比方说我们可以翻转最右边一位得到 110 ，或者翻转右边起第二位得到 101 ，或者翻转右边起第五位（这一位是前导 0 ）得到 10111 等等。 给你两个整数 start 和 goal ，请你返回将 start 转变成 goal 的 最少位翻转 次数。
>
> **提示：**
>
> * `0 <= start, goal <= 1e9`

{% hint style="info" %}
简单题，其实就是比较start和goal两个数字二进制相同位置不同值的个数，对两个数字求异或，相同的变0，不同的变1，统计1的个数即可。
{% endhint %}

```
    public int minBitFlips(int start, int goal) {
        int a = start ^ goal;
        return Integer.bitCount(a);
    }
```

### [**2221. 数组的三角和**](https://leetcode-cn.com/problems/find-triangular-sum-of-an-array/) **** :star::star:****

> 给你一个下标从 0 开始的整数数组 nums ，其中 nums\[i] 是 0 到 9 之间（两者都包含）的一个数字。
>
> nums 的 三角和 是执行以下操作以后最后剩下元素的值：
>
> nums 初始包含 n 个元素。如果 n == 1 ，终止 操作。否则，创建 一个新的下标从 0 开始的长度为 n - 1 的整数数组 newNums 。 对于满足 0 <= i < n - 1 的下标 i ，newNums\[i] 赋值 为 (nums\[i] + nums\[i+1]) % 10 ，% 表示取余运算。 将 newNums 替换 数组 nums 。 从步骤 1 开始 重复 整个过程。 请你返回 nums 的三角和。
>
> **提示：**
>
> * `1 <= nums.length <= 1000`
> * `0 <= nums[i] <= 9`

{% hint style="info" %}
中等题，没啥好想的，就是不断地求和。
{% endhint %}

```
    public int triangularSum(int[] nums) {
        int len = nums.length;
        while (len > 1) {
            for (int i = 0; i < len - 1; i++) {
                nums[i] = (nums[i] + nums[i + 1]) % 10;
            }
            len--;
        }
        return nums[0];
    }
```

### [2222**. 选择建筑的方案数**](https://leetcode-cn.com/problems/number-of-ways-to-select-buildings/) **** :star::star:****

> 给你一个下标从 0 开始的二进制字符串 s ，它表示一条街沿途的建筑类型，其中：
>
> s\[i] = '0' 表示第 i 栋建筑是一栋办公楼， s\[i] = '1' 表示第 i 栋建筑是一间餐厅。 作为市政厅的官员，你需要随机 选择 3 栋建筑。然而，为了确保多样性，选出来的 3 栋建筑 相邻 的两栋不能是同一类型。
>
> 比方说，给你 s = "001101" ，我们不能选择第 1 ，3 和 5 栋建筑，因为得到的子序列是 "011" ，有相邻两栋建筑是同一类型，所以 不合 题意。 请你返回可以选择 3 栋建筑的 有效方案数 。
>
> **提示：**
>
> * `3 <= s.length <= 1e5`
> * `s[i]` 要么是 `'0'` ，要么是 `'1'` 。

{% hint style="info" %}
中等题，1. 前缀和，对于1求左边和右边0的数目，对于0求左右1的数目。&#x20;

2\. 用三个变量a b c 表示 0  01  010出现的次数，当i==0时，a++, c+=b; i==1时 b++；从而统计出前缀的次数。
{% endhint %}

```
   //前缀和
   static public long numberOfWays(String s) {
        //统计0和1的前缀和
        int[] zero = new int[s.length() + 1];
        int[] one = new int[s.length() + 1];
        for (int i = 0; i < s.length(); i++) {
            zero[i + 1] = zero[i];
            one[i + 1] = one[i];
            if (s.charAt(i) == '1') {
                one[i + 1]++;
            } else {
                zero[i + 1]++;
            }
        }
        long ans = 0;
        for (int i = 0; i < s.length(); i++) {
            if (s.charAt(i) == '1') {
                long left = zero[i + 1];
                long right = zero[s.length()] - zero[i + 1];
                ans += left * right;
            } else {
                long left = one[i + 1];
                long right = one[s.length()] - one[i + 1];
                ans += (left * right);
            }
        }
        return ans;
    }
```

```
    //2.统计前缀
      pubic long numberOfWays(String s) {
        char[] chars = s.toCharArray();
        char[] char1 = {'0','1','0'};
        char[] char2 = {'1','0','1'};
        
        return help(chars,char1)+help(chars,char2);
    }
    public long help(char[] chars,char[] goal){
        long a=0,b=0,c=0;
        for(int i=0;i<chars.length; i++){
            if (chars[i] == goal[0]) a++;
            if (chars[i] == goal[1]) b +=a;
            if (chars[i] == goal[2]) c +=b;
        }
        return c;
    }
```

### [**2223. 构造字符串的总得分和**](https://leetcode-cn.com/problems/sum-of-scores-of-built-strings/)  ****  :star::star::star:****

> 你需要从空字符串开始 构造 一个长度为 n 的字符串 s ，构造的过程为每次给当前字符串 前面 添加 一个 字符。构造过程中得到的所有字符串编号为 1 到 n ，其中长度为 i 的字符串编号为 si 。
>
> 比方说，s = "abaca" ，s1 == "a" ，s2 == "ca" ，s3 == "aca" 依次类推。 si 的 得分 为 si 和 sn 的 最长公共前缀 的长度（注意 s == sn ）。
>
> 给你最终的字符串 s ，请你返回每一个 si 的 得分之和 。
>
> **提示：**
>
> * `1 <= s.length <= 1e5`
> * `s` 只包含小写英文字母。

{% hint style="info" %}
困难题，当时是真心没想到怎么做。考虑字符串hash，求h和p数组，这样我们就可以求出任意区间的hash值。然后对后缀进行枚举， 枚举的时候判断前缀hash值相同的长度，因为这里有单调性(即相同字符串的前缀肯定还是相同的),所以考虑用二分法降低时间复杂度。

z函数做法不考虑，不经常用。
{% endhint %}

```
 long[] h;
    long[] p;

    //构造h和p
    public void build(String s) {
        h = new long[s.length() + 1];
        p = new long[s.length() + 1];

        p[0] = 1;
        int t = 171;
        for (int i = 0; i < s.length(); i++) {
            h[i + 1] = h[i] * t + (s.charAt(i));
            p[i + 1] = p[i] * t;
        }
    }
    //求从start到end的区间hash
    public long get(int start, int end) {
        return h[end + 1] - h[start] * p[end - start + 1];
    }

    public long sumScores(String s) {
        build(s);
        long ans = 0;
        int n = s.length();
        for (int i = n - 1; i >= 0; i--) {
            //二分后缀的长度进行判断
            int left = 0, right = n - i;
            while (left < right) {
                int mid = (left + right + 1) / 2;
                long g = get(0, mid - 1);
                long goal = get(i, i + mid - 1);
                if (g == goal) {
                    left = mid;
                } else {
                    right = mid - 1;
                }
            }
            ans += left;
        }
        return ans;
    }
```
