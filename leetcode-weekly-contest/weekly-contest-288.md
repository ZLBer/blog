---
description: https://leetcode-cn.com/contest/weekly-contest-288/
---

# Weekly Contest 288

ranking：371 / 6926

[**2231. 按奇偶性交换后的最大数字**](https://leetcode-cn.com/problems/largest-number-after-digit-swaps-by-parity/)

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

[**2232. 向表达式添加括号后的最小结果**](https://leetcode-cn.com/problems/minimize-result-by-adding-parentheses-to-expression/)

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

[**2233. K 次增加后的最大乘积**](https://leetcode-cn.com/problems/maximum-product-after-k-increments/)

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

[**2234. 花园的最大总美丽值**](https://leetcode-cn.com/problems/maximum-total-beauty-of-the-gardens/)

{% hint style="info" %}
困难题，flowers数组从大到小排序，然后其达到target需要的花的数目就是依次递减了，然后我们从左到右依次枚举能达到target的花园的数目，需要将>target的花园提前剔除，然后将剩下的花分给剩下的花园，其最大的花的数目只能是target-1，那个score=左边的full+右边的partial。

怎么计算右边的partial得分呢？换种说法，也就是如何计算最多种植x朵花(x是最优的，因为flowers数组是降序，其剩下的花朵数是最多的)，右边花园的最小花朵数。可以用二分的方法，最少是0，最多是target-1，然后二分判断能否达到这个最小值，注意这里用遍历判断是会超时的，周赛的时候就犯了这个错误。那么还有什么办法呢，因为flowers数组是递减的，肯定存在一个位置左边的是>=这个二分值，右边<这个二分值，那么只需要将右边部分的和算出来，然后判断一下即可，这里又需要用到前缀和。&#x20;

所以这个题还是挺复杂的，首先要明确目标，然后用两个二分+前缀和去判断。

代码有详细注释。
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
