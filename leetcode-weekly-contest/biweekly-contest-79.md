---
description: https://leetcode-cn.com/contest/biweekly-contest-79/
---

# Biweekly Contest 79

ranking: 569 / 4250



### [**2283. 判断一个数的数字计数是否等于数位的值**](https://leetcode.cn/problems/check-if-number-has-equal-digit-count-and-digit-value/)

{% hint style="info" %}
简单题，统计判断即可
{% endhint %}

```
    public boolean digitCount(String num) {
        int[] count = new int[10];

        for (int i = 0; i < num.length(); i++) {
            count[num.charAt(i) - '0']++;
        }

        for (int i = 0; i < num.length(); i++) {
            if (count[i] != num.charAt(i) - '0') return false;
        }
        return true;
    }
```



### [**2284. 最多单词数的发件人**](https://leetcode.cn/problems/sender-with-largest-word-count/)

{% hint style="info" %}
中等题，先统计每个人的单词书，然后O(n)选出最多的
{% endhint %}

```
 public String largestWordCount(String[] messages, String[] senders) {
        Map<String, Integer> map = new HashMap<>();
        for (int i = 0; i < messages.length; i++) {
            int count = messages[i].split(" ").length;
            map.put(senders[i], map.getOrDefault(senders[i], 0) + count);
        }
        String res = senders[0];
        int count = map.get(senders[0]);

        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            if (entry.getValue() > count || (entry.getValue() == count && entry.getKey().compareTo(res) > 0)) {
                res = entry.getKey();
                count = entry.getValue();
            }
        }
        return res;
    }

```



### [**2285. 道路的最大总重要性**](https://leetcode.cn/problems/maximum-total-importance-of-roads/)

{% hint style="info" %}
中等题，贪心按照点的度数排序，然后依次分配value
{% endhint %}

```
 public long maximumImportance(int n, int[][] roads) {
        long[] count = new long[n];
        for (int[] road : roads) {
            count[road[0]]++;
            count[road[1]]++;
        }

        Arrays.sort(count);
        long sum = 0;
        for (int i = count.length - 1; i >= 0; i--) {
            sum += (count[i] * (n--));
        }
        return sum;
    }
```



### [**2286. 以组为单位订音乐会的门票**](https://leetcode.cn/problems/booking-concert-tickets-in-groups/)

{% hint style="info" %}
//困难题，比赛的时候优化了好久，用了treeMap来记录，就是过不了最后的几个用例

&#x20;//线段树求区间和+区间最小值值
{% endhint %}

```
class BookMyShow {
        int n, m;
        int[] min;
        long[] sum;

        public BookMyShow(int n, int m) {
            this.n = n;
            this.m = m;
            min = new int[n * 4];
            sum = new long[n * 4];
        }

        public int[] gather(int k, int maxRow) {
            int i = index(1, 1, n, maxRow + 1, m - k);
            if (i == 0) return new int[]{}; // 不存在
            var seats = (int) query_sum(1, 1, n, i, i);
            add(1, 1, n, i, k); // 占据 k 个座位
            return new int[]{i - 1, seats};
        }

        public boolean scatter(int k, int maxRow) {
            if ((long) (maxRow + 1) * m - query_sum(1, 1, n, 1, maxRow + 1) < k) return false; // 剩余座位不足 k 个
            // 从第一个没有坐满的排开始占座
            for (var i = index(1, 1, n, maxRow + 1, m - 1); ; ++i) {
                var left_seats = m - (int) query_sum(1, 1, n, i, i);
                if (k <= left_seats) { // 剩余人数不够坐后面的排
                    add(1, 1, n, i, k);
                    return true;
                }
                k -= left_seats;
                add(1, 1, n, i, left_seats);
            }
        }

        // 将 idx 上的元素值增加 val
        void add(int o, int l, int r, int idx, int val) {
            if (l == r) {
                min[o] += val;
                sum[o] += val;
                return;
            }
            var m = (l + r) / 2;
            if (idx <= m) add(o * 2, l, m, idx, val);
            else add(o * 2 + 1, m + 1, r, idx, val);
            //更新改点的最小值和sum
            min[o] = Math.min(min[o * 2], min[o * 2 + 1]);
            sum[o] = sum[o * 2] + sum[o * 2 + 1];
        }

        // 返回区间 [L,R] 内的元素和
        long query_sum(int o, int l, int r, int L, int R) { // L 和 R 在整个递归过程中均不变，将其大写，视作常量
            if (L <= l && r <= R) return sum[o];
            var sum = 0L;
            var m = (l + r) / 2;
            if (L <= m) sum += query_sum(o * 2, l, m, L, R);
            if (R > m) sum += query_sum(o * 2 + 1, m + 1, r, L, R);
            return sum;
        }

        //二分查找最小的位置
        // 返回区间 [1,R] 中 <= val 的最靠左的位置，不存在时返回 0
        int index(int o, int l, int r, int R, int val) { // R 在整个递归过程中均不变，将其大写，视作常量
            if (min[o] > val) return 0; // 说明整个区间的元素值都大于 val
            if (l == r) return l;
            var m = (l + r) / 2;
            if (min[o * 2] <= val) return index(o * 2, l, m, R, val); // 看看左半部分
            if (m < R) return index(o * 2 + 1, m + 1, r, R, val); // 看看右半部分
            return 0;
        }
    }
```

