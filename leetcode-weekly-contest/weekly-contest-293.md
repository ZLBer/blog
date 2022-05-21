---
description: https://leetcode.cn/contest/weekly-contest-293/
---

# Weekly Contest 293

ranking: 1746 / 7357

### [**2273. 移除字母异位词后的结果数组**](https://leetcode.cn/problems/find-resultant-array-after-removing-anagrams/) **** :star:

{% hint style="info" %}
简单题，统计字母个数比较就可以了
{% endhint %}

```
  public List<String> removeAnagrams(String[] words) {
        List<String> res = new ArrayList<>();
        String pre = "";
        for (int i = 0; i < words.length; i++) {
            int[] count = new int[26];
            for (int j = 0; j < words[i].length(); j++) {
                int c = words[i].charAt(j) - 'a';
                count[c]++;
            }
            String s = "";
            for (int j : count) {
                s += j;
            }
            if (!pre.equals(s)) res.add(words[i]);
            pre = s;
        }
        return res;
    }
```

### [**2274. 不含特殊楼层的最大连续楼层数**](https://leetcode.cn/problems/maximum-consecutive-floors-without-special-floors/) **** :star::star:****

{% hint style="info" %}
中等题，对special排序，两个sp之间的距离就是连续楼层，求其最大值，注意上下边界
{% endhint %}

```
    static public int maxConsecutive(int bottom, int top, int[] special) {
        int pre = bottom;
        int max = 0;
        Arrays.sort(special);
        for (int i = 0; i < special.length; i++) {
            max = Math.max(max, special[i] - pre);
            pre = special[i] + 1;
        }
        max = Math.max(max, top - pre + 1);
        return max;
    }
```

### [**2275. 按位与结果大于零的最长组合**](https://leetcode.cn/problems/largest-combination-with-bitwise-and-greater-than-zero/) **** :star::star:****

{% hint style="info" %}
中等题，灵活转换，&操作下，只有都是1的位置才会结果为1，统计二进制下每个位置1的个数即可
{% endhint %}

```
    public int largestCombination(int[] candidates) {
        int[] count = new int[32];
        for (int i = 0; i < candidates.length; i++) {
            int num = candidates[i];
            for (int j = 0; j < 32; j++) {
                if (((num >> j) & 1) > 0) {
                    count[j]++;
                }
            }
        }
        int max = 0;
        for (int i = 0; i < count.length; i++) {
            max = Math.max(count[i], max);
        }
        return max;
    }
```

### [**2276. 统计区间中的整数数目**](https://leetcode.cn/problems/count-integers-in-intervals/) **** :star::star::star:****

{% hint style="info" %}
//困难题，合并区间，之前做过类似的题目，这次又没做出来，XXXX&#x20;

//treeMap保存区间的头尾，当有闲的区间进来的时候，不断的找到<=right的位置，并且要>=left，删除此区间，更新心的区间大小&#x20;

//线段树：动态开点线段树，防止范围太大 out of memory
{% endhint %}

```
/*   class CountIntervals {

        TreeMap<Integer, Integer> map = new TreeMap<>();

        int ans = 0;

        public CountIntervals() {

        }

        public void add(int left, int right) {
            Integer L = map.floorKey(right);
            int l = left, r = right;
            while (L != null && map.get(L) >= l) { //判断有交集
                l = Math.min(l, L);   //求最小左侧
                r = Math.max(r, map.get(L)); //求最大右侧
                ans -= (map.get(L) - L + 1);//count也要删掉
                map.remove(L); //删掉此段

                L = map.floorKey(right);
            }
            ans += (r - l + 1);
            map.put(l, r);
        }

        public int count() {
            return ans;
        }
    }*/


    //线段树：动态开点线段树，防止范围太大 out of memory
    class CountIntervals {
        int l, r, cnt;
        CountIntervals L, R;

        public CountIntervals() {
            this.l = 1;
            this.r = (int) 1e9;
        }

        public CountIntervals(int l, int r) {
            this.l = l;
            this.r = r;

        }

        public void add(int left, int right) {
            //表示这一段之前有过，再加还是这样， 直接返回
            if (cnt == r - l + 1) return;
            //表示完全覆盖了这段 直接返回
            if (left <= l && right >= r) {
                cnt = r - l + 1;
                return;
            }
            //均分这一段
            int mid = (l + r) / 2;
            if (L == null) L = new CountIntervals(l, mid);
            if (R == null) R = new CountIntervals(mid + 1, r);
            //和左边有没有交集
            if (left <= mid) L.add(left, right);
            //和右边有没有交集
            if (mid < right) R.add(left, right);

            cnt = L.cnt + R.cnt;
        }

        public int count() {
            return cnt;
        }
    }

```

