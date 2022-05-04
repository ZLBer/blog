---
description: https://leetcode-cn.com/contest/weekly-contest-291/
---

# Weekly Contest 291

ranking: 342 / 6574

### [**2259. 移除指定数字得到的最大结果**](https://leetcode-cn.com/problems/remove-digit-from-number-to-maximize-result/) **** :star:****

{% hint style="info" %}
简单题，求出每个发个要求的字符串然后比较大小
{% endhint %}

```
   public String removeDigit(String number, char digit) {

        List<String> list = new ArrayList<>();
        for (int i = 0; i < number.length(); i++) {
            char c = number.charAt(i);
            if (c != digit) continue;
            String s = number.substring(0, i) + number.substring(i + 1);
            list.add(s);
        }
        Collections.sort(list);
        return list.get(list.size() - 1);
    }
```



### [**2260. 必须拿起的最小连续卡牌数**](https://leetcode-cn.com/problems/minimum-consecutive-cards-to-pick-up/) **** :star::star:

{% hint style="info" %}
中等题，map记录同一个数字之前的位置，然后求个最小间距即可
{% endhint %}

```
  public int minimumCardPickup(int[] cards) {
        Map<Integer, Integer> map = new HashMap<>();
        int ans = Integer.MAX_VALUE;
        for (int i = 0; i < cards.length; i++) {
            int card = cards[i];
            if (map.containsKey(card)) {
                ans = Math.min(ans, i - map.get(card) + 1);
            }
            map.put(card, i);
        }
        return ans == Integer.MAX_VALUE ? -1 : ans;
    }
```

### [**2261. 含最多 K 个可整除元素的子数组**](https://leetcode-cn.com/problems/k-divisible-elements-subarrays/)  ****  :star::star:****

{% hint style="info" %}
中等题，滑动窗口，保证窗口内符合要求的数字数目<=k，但题目让你求不同数组的数目，需要用set对数组去重
{% endhint %}

```
public int countDistinct(int[] nums, int k, int p) {
        int left = 0;
        int count = 0;

        Set<List<Integer>> set = new HashSet<>();
        for (int i = 0; i < nums.length; i++) {

            if (nums[i] % p == 0) {
                count++;
            }
            while (left < i && count > k) {
                if (nums[left] % p == 0) count--;
                left++;
            }
            List<Integer> list = new ArrayList<>();
            for (int j = i; j >= left; j--) {
                list.add(nums[j]);
                set.add(new ArrayList<>(list));
            }

        }
        return set.size();
    }
```



### [**2262. 字符串的总引力**](https://leetcode-cn.com/problems/total-appeal-of-a-string/) **** :star::star::star:****

{% hint style="info" %}
//困难题，要思考引力值的计算方式与什么有关。&#x20;

//定义一个sum数组，表示截止到此位置子字符串总引力值。i位置的引力值如何计算？

&#x20;// 若之前没有和i位置重复的字母，那么对前面所有子字符串的引力值贡献+1。引力值为：sum\[i-1] + i + 1;

&#x20;// 若之前有和i位置重复的字母，位置为j ，那么 引力值计算分成两部分 j及之前没有引力值贡献，j之后引力值贡献+1，引力值为： sum\[i-1] + i - j
{% endhint %}

```
 public long appealSum(String s) {
        int n = s.length();
        long ans = 0;
        long[] sum = new long[s.length() + 1];
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < n; i++) {
            int c = s.charAt(i) - 'a';
            long cur = 0;
            if (map.containsKey(c)) {
                cur = sum[i] + i - map.get(c);
            } else {
                cur = sum[i] + i + 1;
            }
            sum[i + 1] = cur;
            ans += sum[i + 1];
            map.put(c, i);
        }
        return ans;

    }
```
