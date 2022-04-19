---
description: https://leetcode-cn.com/contest/weekly-contest-288/
---

# Weekly Contest 288

ranking：371 / 6926

### [**2231. 按奇偶性交换后的最大数字**](https://leetcode-cn.com/problems/largest-number-after-digit-swaps-by-parity/) **** :star:****

> 给你一个正整数 num 。你可以交换 num 中 奇偶性 相同的任意两位数字（即，都是奇数或者偶数）。
>
> 返回交换 任意 次之后 num 的 最大 可能值。
>
> **提示：**
>
> * `1 <= num <= 1e9`

{% hint style="info" %}
简单题，可以冒泡排序哦，在冒泡的基础上加上abs(a-b)是偶数的判断哦。要是不这样，硬座也行，就是把奇偶数都单独拿出来重新排序，然后填回去。
{% endhint %}

```
 static public int largestInteger(int num) {
        char[] chars = (num + "").toCharArray();
        int n = chars.length;
        for (int i = 0; i < n; i++) {
            int a = chars[i];
            for (int j = i + 1; j < n; j++) {
                int b = chars[j];
                if (Math.abs(chars[i] - chars[j]) % 2 == 0 && chars[i] < chars[j]) {
                    swap(chars, i, j);
                }
            }
        }
        return Integer.parseInt(new String(chars));
    }

    static void swap(char[] chars, int i, int j) {
        char temp = chars[i];
        chars[i] = chars[j];
        chars[j] = temp;
    }
```

### [**2232. 向表达式添加括号后的最小结果**](https://leetcode-cn.com/problems/minimize-result-by-adding-parentheses-to-expression/) **** :star::star:****

> 给你一个下标从 0 开始的字符串 expression ，格式为 "+" ，其中 和 表示正整数。
>
> 请你向 expression 中添加一对括号，使得在添加之后， expression 仍然是一个有效的数学表达式，并且计算后可以得到 最小 可能值。左括号 必须 添加在 '+' 的左侧，而右括号必须添加在 '+' 的右侧。
>
> 返回添加一对括号后形成的表达式 expression ，且满足 expression 计算得到 最小 可能值。如果存在多个答案都能产生相同结果，返回任意一个答案。
>
> 生成的输入满足：expression 的原始值和添加满足要求的任一对括号之后 expression 的值，都符合 32-bit 带符号整数范围。
>
>
>
> **提示：**
>
> * `3 <= expression.length <= 10`
> * `expression` 仅由数字 `'1'` 到 `'9'` 和 `'+'` 组成
> * `expression` 由数字开始和结束
> * `expression` 恰好仅含有一个 `'+'`.
> * `expression` 的原始值和添加满足要求的任一对括号之后 `expression` 的值，都符合 32-bit 带符号整数范围

{% hint style="info" %}
中等题，左右两边各自枚举括号的位置求最大值即可，需要非常注意细节。
{% endhint %}

```
  static public String minimizeResult(String expression) {
        String[] split = expression.split("\\+");
        long ans = Long.MAX_VALUE;
        int[] res = new int[2];
        for (int i = 0; i < split[0].length(); i++) {
            long left1 = i == 0 ? 1 : Long.parseLong(split[0].substring(0, i));
            long left2 = Long.parseLong(split[0].substring(i));
            for (int j = 1; j <= split[1].length(); j++) {
                long right1 = Long.parseLong(split[1].substring(0, j));
                long right2 = j == split[1].length() ? 1 : Long.parseLong(split[1].substring(j));
                long temp = left1 * (left2 + right1) * right2;
                if (temp < ans) {
                    res = new int[]{i, j};
                    ans = temp;
                }
            }
        }
        StringBuilder stringBuilder = new StringBuilder(expression);
        stringBuilder.insert(res[0], '(');
        stringBuilder.insert(res[1] + 1 + split[0].length() + 1, ')');
        return stringBuilder.toString();
    }
```

### [**2233. K 次增加后的最大乘积**](https://leetcode-cn.com/problems/maximum-product-after-k-increments/) **** :star::star:****

> 给你一个非负整数数组 nums 和一个整数 k 。每次操作，你可以选择 nums 中 任一 元素并将它 增加 1 。
>
> 请你返回 至多 k 次操作后，能得到的 nums的 最大乘积 。由于答案可能很大，请你将答案对 109 + 7 取余后返回。
>
> **提示：**
>
> * `1 <= nums.length, k <= 1e5`
> * `0 <= nums[i] <= 1e6`

{% hint style="info" %}
中等题，贪心，先看1次增加后怎么乘积最大，肯定是将最小的数+1，这样乘积会增加其余数的和。那么k次也是一样的效果。
{% endhint %}

```
  public int maximumProduct(int[] nums, int k) {
        PriorityQueue<Long> priorityQueue = new PriorityQueue<>();
        for (int i = 0; i < nums.length; i++) {
            priorityQueue.add((long) nums[i]);
        }
        while (k-- > 0) {
            Long poll = priorityQueue.poll();
            poll++;
            priorityQueue.add(poll);
        }
        long ans = 1;
        while (!priorityQueue.isEmpty()) {
            ans *= priorityQueue.poll();
            ans %= (int) 1e9 + 7;
        }
        return (int) ans;
    }
```

### [**2234. 花园的最大总美丽值**](https://leetcode-cn.com/problems/maximum-total-beauty-of-the-gardens/) **** :star::star::star:****

> Alice 是 n 个花园的园丁，她想通过种花，最大化她所有花园的总美丽值。
>
> 给你一个下标从 0 开始大小为 n 的整数数组 flowers ，其中 flowers\[i] 是第 i 个花园里已经种的花的数目。已经种了的花 不能 移走。同时给你 newFlowers ，表示 Alice 额外可以种花的 最大数目 。同时给你的还有整数 target ，full 和 partial 。
>
> 如果一个花园有 至少 target 朵花，那么这个花园称为 完善的 ，花园的 总美丽值 为以下分数之 和 ：
>
> 完善 花园数目乘以 full. 剩余 不完善 花园里，花的 最少数目 乘以 partial 。如果没有不完善花园，那么这一部分的值为 0 。 请你返回 Alice 种最多 newFlowers 朵花以后，能得到的 最大 总美丽值。
>
> **提示：**
>
> * `1 <= flowers.length <= 1e5`
> * `1 <= flowers[i], target <= 1e5`
> * `1 <= newFlowers <= 1e10`
> * `1 <= full, partial <= 1e5`

{% hint style="info" %}
困难题，flowers数组从大到小排序，然后其达到target需要的花的数目就是依次递减了，然后我们从左到右依次枚举能达到target的花园的数目，需要将>target的花园提前剔除，然后将剩下的花分给剩下的花园，其最大的花的数目只能是target-1，那个score=左边的full+右边的partial。

1.怎么计算右边的partial得分呢？换种说法，也就是如何计算最多种植x朵花(x是最优的，因为flowers数组是降序，其剩下的花朵数是最多的)，右边花园的最小花朵数。可以用二分的方法，最少是0，最多是target-1，然后二分判断能否达到这个最小值，注意这里用遍历判断是会超时的，周赛的时候就犯了这个错误。那么还有什么办法呢，因为flowers数组是递减的，肯定存在一个位置左边的是>=这个二分值，右边<这个二分值，那么只需要将右边部分的和算出来，然后判断一下即可，这里又需要用到前缀和。&#x20;

所以这个题还是挺复杂的，首先要明确目标，然后用两个二分+前缀和去判断。

代码有详细注释。

2\. 还可以用双指针做，要知道随着target枚举值的增加，partial枚举值的减小，右边部分的最小值是递减的(好好想想便知道，左边侵占的花越来越多)，那么便可以枚举这个最小值，从target-1开始一直到数组右端点。
{% endhint %}

```
 static public long maximumBeauty(int[] flowers, long newFlowers, int target, int full, int partial) {
        List<Integer> sortFlowers = new ArrayList<>();
        for (int flower : flowers) {
            sortFlowers.add(flower);
        }
        //将flowers数组排序
        Collections.sort(sortFlowers, Comparator.reverseOrder());
        //前缀和
        long[] preSum = new long[sortFlowers.size() + 1];
        for (int i = 0; i < sortFlowers.size(); i++) {
            preSum[i + 1] = preSum[i] + sortFlowers.get(i);
        }
        long ans = 0;
        long fullScore = 0;

        //跳过已经达到full的花园
        int i = 0;
        while (i < sortFlowers.size() && sortFlowers.get(i) >= target) {
            i++;
            fullScore += full;
        }
        //求一次不填充到full的情况
        ans = fullScore + getPartial(preSum, sortFlowers, i, partial, target, newFlowers);
        //从i开始依次开始填充到full
        for (; i < sortFlowers.size(); i++) {
            Integer flower = sortFlowers.get(i);
            if (flower < target) {
                newFlowers -= (target - flower);
            }
            //不能再种了
            if (newFlowers < 0) break;
            //更新full的得分
            fullScore += full;
            //求partial的得分
            Long partialScore = getPartial(preSum, sortFlowers, i + 1, partial, target, newFlowers);
            ans = Math.max(fullScore + partialScore, ans);
        }
        return ans;

    }

    static long getPartial(long[] preSum, List<Integer> sortFlowers, int from, long partial, int target, long newFlowers) {
        if (from >= sortFlowers.size()) return 0;
        //二分求在newFlowers朵花的情况下，能将flowers的最下值提高到什么位置
        int left = sortFlowers.get(sortFlowers.size() - 1), right = target - 1;
        while (left < right) {
            int mid = (left + right + 1) / 2;
            long sum = 0;
            //方法1 遍历的时间复杂度太高，导致超时了
        /*    for (int i = from; i < sortFlowers.size(); i++) {
                if (sortFlowers.get(i) < mid) {
                    sum += mid - sortFlowers.get(i);
                }
            }*/
            // 方法2 利用sortFlowers的递减性，二分查找<=mid的最左位置   这样不会超时了
            int index = helper(sortFlowers, from, mid);
            // 算一下当最小值是mid的时候，需要用多少朵花
            sum = mid * ((long) sortFlowers.size() - index) - (preSum[sortFlowers.size()] - preSum[index]);
            if (sum <= newFlowers) {
                left = mid;
            } else {
                right = mid - 1;
            }
        }
        return partial * left;
    }

    static int helper(List<Integer> sortFlowers, int from, int m) {
        int left = from, right = sortFlowers.size() - 1;
        while (left < right) {
            int mid = (left + right + 1) / 2;
            if (sortFlowers.get(mid) > m) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        return left;
    }
```

```
//方法3  利用patial部分最小值递减特性
    static public long maximumBeauty(int[] flowers, long newFlowers, int target, int full, int partial) {
        List<Integer> sortFlowers = new ArrayList<>();
        for (int flower : flowers) {
            sortFlowers.add(flower);
        }
        //将flowers数组排序
        Collections.sort(sortFlowers, Comparator.reverseOrder());
        //前缀和

        long ans = 0;

        //跳过已经达到full的花园
        int i = 0;
        while (i < sortFlowers.size() && sortFlowers.get(i) >= target) {
            sortFlowers.set(i,target);
            i++;
        }
        long[] preSum = new long[sortFlowers.size() + 1];
        for (int j = 0; j < sortFlowers.size(); j++) {
            preSum[j + 1] = preSum[j] + sortFlowers.get(j);
        }
        //最小值枚举
        long T = target - 1;
        //双指针
        int j = i;
        int n = sortFlowers.size();
        
        //判断全部target是否可行
        if (((long) target) * n - preSum[n] <= newFlowers) {
            ans = ((long) full) * n;
        }

        for (; i < sortFlowers.size(); i++) {
            Integer flower = sortFlowers.get(i);

            //j不能小于i
            while (j < i) {
                j++;
            }
            //让T移动
            while (T * (n - j) - (preSum[n] - preSum[j]) > newFlowers) {
                T--;
                //同时j不能大于T
                while (j < n && sortFlowers.get(j) > T) {
                    j++;
                }
            }
            if (flower < target) {
                newFlowers -= (target - flower);
            }
            ans = Math.max(ans, (long) full * i + (long) partial * T);
            //剩余花的数量为0 说明此次分配target失效 直接返回
            if(newFlowers<0) break;

        }
        return ans;

    }
```
