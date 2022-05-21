---
description: https://leetcode-cn.com/contest/biweekly-contest-78/
---

# Biweekly Contest 78

ranking: 688 / 4347

### [**2269. 找到一个数字的 K 美丽值**](https://leetcode.cn/problems/find-the-k-beauty-of-a-number/) **** :star:

{% hint style="info" %}
简单题，直接截取字符串判断是不是符合条件
{% endhint %}

```
    public int divisorSubstrings(int num, int k) {
        String n = num + "";
        int ans = 0;
        for (int i = k - 1; i < n.length(); i++) {
            String sub = n.substring(i - k + 1, i + 1);
            if (Integer.parseInt(sub) == 0) continue;
            if (num % Integer.parseInt(sub) == 0) ans++;
        }
        return ans;
    }
```

### [**2270. 分割数组的方案数**](https://leetcode.cn/problems/number-of-ways-to-split-array/) **** :star::star:****

{% hint style="info" %}
中等题，前缀和算就好了，注意int数组会溢出，改成long
{% endhint %}

```
   public int waysToSplitArray(int[] nums) {
        long[] sum = new long[nums.length + 1];
        for (int i = 0; i < nums.length; i++) {
            sum[i + 1] = sum[i] + nums[i];
        }

        int ans = 0;
        for (int i = 0; i < nums.length; i++) {
            long pre = sum[i + 1];
            long last = sum[nums.length] - sum[i + 1];
            if (pre >= last && i != nums.length - 1) ans++;
        }
        return ans;


    }
```

### [**2271. 毯子覆盖的最多白色砖块数**](https://leetcode.cn/problems/maximum-white-tiles-covered-by-a-carpet/) **** :star::star:****

{% hint style="info" %}
中等题，排序后双指针做
{% endhint %}

```
   public int maximumWhiteTiles(int[][] tiles, int carpetLen) {
        Arrays.sort(tiles, (a, b) -> a[1] - b[1]);
        int max = 0;
        int left = 0;
        int count = 0;
        int sub = 0;
        for (int i = 0; i < tiles.length; i++) {
            //每次都与这块砖的右侧对其
            int[] block = tiles[i];
            //求左侧位置
            int pre = block[1] - carpetLen + 1;
            count += block[1] - block[0] + 1;
            //去掉没有覆盖到的
            while (left < i && tiles[left][1] < pre) {
                count -= (tiles[left][1] - tiles[left][0] + 1);
                //再减去这块砖的时候，要看下之前有没有减去
                count += sub;
                sub = 0;
                left++;
            }
            //判断下left有没有没被覆盖的
            if (tiles[left][0] < pre) {
                int all = (pre - tiles[left][0]);

                //这里也需要加上之前的sub
                count -= (all - sub);
                //表示这块砖已经减去了这些
                sub = all;
            }
            max = Math.max(max, count);
        }
        return max;
    }

     //优化代码逻辑，去掉了sub变量
    public int maximumWhiteTiles(int[][] tiles, int carpetLen) {
        Arrays.sort(tiles, (a, b) -> a[1] - b[1]);
        int max = 0;
        int left = 0;
        int count = 0;
        for (int i = 0; i < tiles.length; i++) {
            //每次都与这块砖的右侧对其
            int[] block = tiles[i];
            //求左侧位置
            int pre = block[1] - carpetLen + 1;
            count += block[1] - block[0] + 1;
            //去掉没有覆盖到的
            while (left < i && tiles[left][1] < pre) {
                count -= (tiles[left][1] - tiles[left][0] + 1);
                left++;
            }
            //判断下left有没有没被覆盖的
            if (tiles[left][0] < pre) {
                int all = (pre - tiles[left][0]);
                //更新左侧端点，防止重复计算
                tiles[left][0]=pre;
                count -= all;
            }
            max = Math.max(max, count);
        }
        return max;
    }
```

### [**2272. 最大波动的子字符串**](https://leetcode.cn/problems/substring-with-largest-variance/) **** :star::star::star:****

{% hint style="info" %}
//困难题，真的没想到，二重循环枚举所有字母的，依次求这两个字母的波动值&#x20;

//比如 a b a看成+1 ，b看成-1,模型就成了最大子数组和，但是限制b必须出现至少一次

&#x20;//diff表示ab的差，最小是0，diffB表示带有b的ab的差，diffB可以初始成负无穷，表示答案不存在

&#x20;//遇到a diff++，diffB++，遇到b diff--，diffB=diff，然后每次求max(diffB)
{% endhint %}

```
    public int largestVariance(String s) {
        int n = s.length();

        int ans = 0;
        for (int a = 'a'; a <= 'z'; a++) {
            for (int b = 'a'; b <= 'z'; b++) {
                if (a == b) continue;
                int diff = 0;
                int diffB = -n;
                for (int i = 0; i < n; i++) {
                    if (s.charAt(i) == a) {
                        diff++;
                        diffB++;
                    } else if (s.charAt(i) == b) {
                        diffB = --diff;
                        diff = Math.max(0, diff);
                    }
                    ans = Math.max(ans, diffB);
                }
            }
        }
        return ans;
    }
```
