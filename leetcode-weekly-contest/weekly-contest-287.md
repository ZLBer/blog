---
description: https://leetcode-cn.com/contest/weekly-contest-287/
---

# Weekly Contest 287

ranking：257 / 6811

### [**2224. 转化时间需要的最少操作数**](https://leetcode-cn.com/problems/minimum-number-of-operations-to-convert-time/) **** :star::star:****

> 给你两个字符串 current 和 correct ，表示两个 24 小时制时间 。
>
> 24 小时制时间 按 "HH:MM" 进行格式化，其中 HH 在 00 和 23 之间，而 MM 在 00 和 59 之间。最早的 24 小时制时间为 00:00 ，最晚的是 23:59 。
>
> 在一步操作中，你可以将 current 这个时间增加 1、5、15 或 60 分钟。你可以执行这一操作 任意 次数。
>
> 返回将 current 转化为 correct 需要的 最少操作数 。
>
> **提示：**
>
> * `current` 和 `correct` 都符合 `"HH:MM"` 格式
> * `current <= correct`

{% hint style="info" %}
简单题，先转成分钟，然后贪心的增加最大的分钟即可。
{% endhint %}

```
    
    static public int convertTime(String current, String correct) {
        int ans = 0;
        String[] split = current.split(":");
        String[] split1 = correct.split(":");
        int hour = Integer.parseInt(split1[0]) - Integer.parseInt(split[0]);
        int minite = Integer.parseInt(split1[1]) - Integer.parseInt(split[1]);

        minite += hour * 60;

        int[] arr = new int[]{60, 15, 5};

        for (int i : arr) {
            ans += minite / i;
            minite %= i;
        }
        ans += minite;
        return ans;
    }
```

### [**2225. 找出输掉零场或一场比赛的玩家**](https://leetcode-cn.com/problems/find-players-with-zero-or-one-losses/) **** :star::star:****

> 给你一个整数数组 matches 其中 matches\[i] = \[winneri, loseri] 表示在一场比赛中 winneri 击败了 loseri 。
>
> 返回一个长度为 2 的列表 answer ：
>
> answer\[0] 是所有 没有 输掉任何比赛的玩家列表。 answer\[1] 是所有恰好输掉 一场 比赛的玩家列表。 两个列表中的值都应该按 递增 顺序返回。
>
> 注意：
>
> 只考虑那些参与 至少一场 比赛的玩家。 生成的测试用例保证 不存在 两场比赛结果 相同 。
>
> **注意：**
>
> * 只考虑那些参与 **至少一场** 比赛的玩家。
> * 生成的测试用例保证 **不存在** 两场比赛结果 **相同** 。

{% hint style="info" %}
中等题，用map保存人和输了的次数即可。
{% endhint %}

```
   public List<List<Integer>> findWinners(int[][] matches) {
        Map<Integer, Integer> map = new HashMap<>();
        Set<Integer> set = new HashSet<>();
        for (int[] match : matches) {
            set.add(match[0]);
            map.put(match[1], map.getOrDefault(match[1], 0) + 1);
        }
        List<Integer> ans0 = new ArrayList<>();
        List<Integer> ans1 = new ArrayList<>();
        for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
            if (entry.getValue() == 1) ans1.add(entry.getKey());
        }
        for (Integer key : set) {
            if (!map.containsKey(key)) {
                ans0.add(key);
            }
        }
        Collections.sort(ans0);
        Collections.sort(ans1);
        List<List<Integer>> res = new ArrayList<>();
        res.add(ans0);
        res.add(ans1);
        return res;
    }
```

### [**2226. 每个小孩最多能分到多少糖果**](https://leetcode-cn.com/problems/maximum-candies-allocated-to-k-children/) **** :star::star:****

> 给你一个 下标从 0 开始 的整数数组 candies 。数组中的每个元素表示大小为 candies\[i] 的一堆糖果。你可以将每堆糖果分成任意数量的 子堆 ，但 无法 再将两堆合并到一起。
>
> 另给你一个整数 k 。你需要将这些糖果分配给 k 个小孩，使每个小孩分到 相同 数量的糖果。每个小孩可以拿走 至多一堆 糖果，有些糖果可能会不被分配。
>
> 返回每个小孩可以拿走的 最大糖果数目 。
>
> **提示：**
>
> * `1 <= candies.length <= 105`
> * `1 <= candies[i] <= 107`
> * `1 <= k <= 1012`

{% hint style="info" %}
中等题，对可以拿走的糖果数目进行二分，然后每次check糖果数目是否可以满足。
{% endhint %}

```
    static public int maximumCandies(int[] candies, long k) {
        long sum = 0;
        for (int candy : candies) {
            sum += candy;
        }
        if (sum < k) return 0;
        long right = (sum / k);
        long left = 1;

        while (left < right) {
            long mid = (left + right + 1) / 2;
            if (check(mid, candies, k)) {
                left = mid;
            } else {
                right = mid - 1;
            }
        }
        return (int) left;
    }

    static boolean check(long mid, int[] candies, long k) {
        long ans = 0;
        for (int candy : candies) {
            ans += (candy / mid);
        }
        return ans >= k;
    }
```



### [**2227. 加密解密字符串**](https://leetcode-cn.com/problems/encrypt-and-decrypt-strings/)   ****   :star::star::star:****

> 给你一个字符数组 keys ，由若干 互不相同 的字符组成。还有一个字符串数组 values ，内含若干长度为 2 的字符串。另给你一个字符串数组 dictionary ，包含解密后所有允许的原字符串。请你设计并实现一个支持加密及解密下标从 0 开始字符串的数据结构。
>
> 字符串 加密 按下述步骤进行：
>
> 对字符串中的每个字符 c ，先从 keys 中找出满足 keys\[i] == c 的下标 i 。 在字符串中，用 values\[i] 替换字符 c 。 字符串 解密 按下述步骤进行：
>
> 将字符串每相邻 2 个字符划分为一个子字符串，对于每个子字符串 s ，找出满足 values\[i] == s 的一个下标 i 。如果存在多个有效的 i ，从中选择 任意 一个。这意味着一个字符串解密可能得到多个解密字符串。 在字符串中，用 keys\[i] 替换 s 。 实现 Encrypter 类：
>
> Encrypter(char\[] keys, String\[] values, String\[] dictionary) 用 keys、values 和 dictionary 初始化 Encrypter 类。 String encrypt(String word1) 按上述加密过程完成对 word1 的加密，并返回加密后的字符串。 int decrypt(String word2) 统计并返回可以由 word2 解密得到且出现在 dictionary 中的字符串数目。

{% hint style="info" %}
困难题，不困难。解密后的字符串必须在dictionary里，那我们反向思考，将dictionary里的字符串加密，并且记录加密串出现的次数，这样 encrypt(String word1)的复杂度就降低为O(1)。
{% endhint %}

```
class Encrypter {
        HashMap<Character, String> en = new HashMap<>();
        HashMap<String,Integer> de = new HashMap<>();

        public Encrypter(char[] keys, String[] values, String[] dictionary) {
            for (int i = 0; i < keys.length; i++) {
                en.put(keys[i], values[i]);
            }

            for (String s : dictionary) {
                String encrypt = encrypt(s);
                de.put(encrypt,de.getOrDefault(encrypt,0)+1);
            }
        }

        public String encrypt(String word1) {
            StringBuilder res = new StringBuilder();
            for (int i = 0; i < word1.length(); i++) {
                char c = word1.charAt(i);
                res.append(en.get(c));
            }
            return res.toString();
        }

        public int decrypt(String word2) {
            return de.getOrDefault(word2,0);
        }
    }
```
