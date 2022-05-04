---
description: https://leetcode-cn.com/contest/biweekly-contest-75/
---

# Biweekly Contest 76

810 / 4477   被打扰的一周！

### [**2239. 找到最接近 0 的数字**](https://leetcode-cn.com/problems/find-closest-number-to-zero/) **** :star:

{% hint style="info" %}
简单题 绝对值小 或者 绝对值相等并且此时num更大 的时候就更新
{% endhint %}

```
  public int findClosestNumber(int[] nums) {
        int ans = Integer.MAX_VALUE;
        for (int i = 0; i < nums.length; i++) {
            if (Math.abs(nums[i]) < Math.abs(ans) || (Math.abs(nums[i]) == Math.abs(ans) && nums[i] > ans)) {
                ans = nums[i];
            }
        }
        return ans;
    }
```

### [**2240. 买钢笔和铅笔的方案数**](https://leetcode-cn.com/problems/number-of-ways-to-buy-pens-and-pencils/) **** :star::star:****

{% hint style="info" %}
中等题 ，枚举买了多少只钢笔，然后计算还能还多少只铅笔 注意可以买0只就可以
{% endhint %}

```
    public long waysToBuyPensPencils(int total, int cost1, int cost2) {
        long ans = 0;
        for (int i = 0; i * cost1 <= total; i++) {
            ans += (total - i * cost1) / cost2 + 1;
        }
        System.out.println(ans);
        return ans;
    }
```

### [**2241. 设计一个 ATM 机器**](https://leetcode-cn.com/problems/design-an-atm-machine/) **** :star::star:****

{% hint style="info" %}
中等题，任性atm, 关键是withdraw方法的实现，每次要取min(现有张数，最多能拿几张)
{% endhint %}

```
  static class ATM {

        long[] money = new long[5];
        int[] arr = new int[]{20, 50, 100, 200, 500};

        public ATM() {

        }

        public void deposit(int[] banknotesCount) {
            for (int i = 0; i < banknotesCount.length; i++) {
                money[i] += banknotesCount[i];
            }
        }

        public int[] withdraw(int amount) {

            boolean check = true;
            int temp = amount;
            for (int i = money.length - 1; i >= 0; i--) {
                long d = Math.min(money[i], temp / arr[i]);
                temp -= arr[i] * d;
            }

            if (temp > 0) return new int[]{-1};
            temp = amount;
            int[] ans = new int[5];
            for (int i = money.length - 1; i >= 0; i--) {
                ans[i] = (int) Math.min(temp / arr[i], money[i]);
                temp -= arr[i] * ans[i];
                money[i] -= ans[i];
            }
            return ans;
        }
    }
```

### [**2242. 节点序列的最大得分**](https://leetcode-cn.com/problems/maximum-score-of-a-node-sequence/) **** :star::star::star:****

{% hint style="info" %}
困难题，试过很多超时的方法，先是暴力探测，然后记录每个节点的所有邻接点，每次遍历一次 其实没必要每次遍历节点的所有邻接点， 我们遍历edges的时候，比如是 a-b，没必要遍历ab的所有邻接点 只需要找出ab的最大的三个邻接点即可，因为可能得到b或者 ab的公共邻接点，需要排除这两个，那么剩下一个一定符合要求的 所以啊 用堆保存这三个最大值或者 先找到然后排序均可
{% endhint %}

```
 public int maximumScore(int[] scores, int[][] edges) {
        Map<Integer, PriorityQueue<Integer>> map = new HashMap<>();


        for (int[] edge : edges) {
            if (!map.containsKey(edge[0])) map.put(edge[0], new PriorityQueue<>((a, b) -> scores[a] - scores[b]));
            if (!map.containsKey(edge[1])) map.put(edge[1], new PriorityQueue<>((a, b) -> scores[a] - scores[b]));
            PriorityQueue<Integer> set1 = map.get(edge[0]);
            PriorityQueue<Integer> set2 = map.get(edge[1]);
            set1.add(edge[1]);
            set2.add(edge[0]);
            if (set1.size() > 3) {
                set1.poll();
            }
            if (set2.size() > 3) {
                set2.poll();
            }
        }

        int res = -1;
        for (int i = 0; i < edges.length; i++) {
            int a = edges[i][0];
            int b = edges[i][1];
            PriorityQueue<Integer> aset = map.getOrDefault(a, new PriorityQueue<>());
            PriorityQueue<Integer> bset = map.getOrDefault(b, new PriorityQueue<>());

            //注意integer和int判断相等的区别
            for (int x : aset) {
                for (int y : bset) {
                    if ((x == b) || (x == y) || (y == a)) continue;
                    res = Math.max(res, scores[x] + scores[y] + scores[a] + scores[b]);
                    if (res == 3981) System.out.println(res + " " + x + " " + y + " " + a + " " + b);
                }
            }

        }
        return res;
    }

```
