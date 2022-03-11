---
description: https://leetcode-cn.com/contest/biweekly-contest-73/
---

# Biweekly Contest 73

### [**2190. 数组中紧跟 key 之后出现最频繁的数字**](https://leetcode-cn.com/problems/most-frequent-number-following-key-in-an-array/) **** :star:****

> 给你一个下标从 0 开始的整数数组 nums ，同时给你一个整数 key ，它在 nums 出现过。
>
> 统计 在 nums 数组中紧跟着 key 后面出现的不同整数 target 的出现次数。换言之，target 的出现次数为满足以下条件的 i 的数目：
>
> 0 <= i <= n - 2 nums\[i] == key 且 nums\[i + 1] == target 。 请你返回出现 最多 次数的 target 。测试数据保证出现次数最多的 target 是唯一的。
>
> **提示：**
>
> * `2 <= nums.length <= 1000`
> * `1 <= nums[i] <= 1000`
> * 测试数据保证答案是唯一的。

{% hint style="info" %}
简单题，由于num的范围比较小，可以开一个count数组来统计i出现的次数，最后遍历count数组找出最大的i。
{% endhint %}

```
   public int mostFrequent(int[] nums, int key) {
        int[] count = new int[10001];
        for (int i = 1; i < nums.length; i++) {
            if (nums[i - 1] == key) {
                count[nums[i]]++;
            }
        }
        int res = -1;
        int max = 0;
        for (int i = 1; i < count.length; i++) {
            if (count[i] > max) {
                max = count[i];
                res = i;
            }
        }
        return res;
    }
```

### [**2191. 将杂乱无章的数字排序**](https://leetcode-cn.com/problems/sort-the-jumbled-numbers/)  ****  :star::star:

> 给你一个下标从 0 开始的整数数组 mapping ，它表示一个十进制数的映射规则，mapping\[i] = j 表示这个规则下将数位 i 映射为数位 j 。
>
> 一个整数 映射后的值 为将原数字每一个数位 i （0 <= i <= 9）映射为 mapping\[i] 。
>
> 另外给你一个整数数组 nums ，请你将数组 nums 中每个数按照它们映射后对应数字非递减顺序排序后返回。
>
> 注意：
>
> 如果两个数字映射后对应的数字大小相同，则将它们按照输入中的 相对顺序 排序。 nums 中的元素只有在排序的时候需要按照映射后的值进行比较，返回的值应该是输入的元素本身。
>
> **提示：**
>
> * `mapping.length == 10`
> * `0 <= mapping[i] <= 9`
> * `mapping[i]` 的值 **互不相同** 。
> * `1 <= nums.length <= 3 * 1e4`
> * `0 <= nums[i] < 1e9`

{% hint style="info" %}
中等题，将old num替换成new num，然后保持新旧之间的映射关系，最后按照新数排序，输出旧数。注意当num为0的时候要特殊处理下，或者可以直接对<10的数特殊处理，这里WA了一发。
{% endhint %}

```
   static public int[] sortJumbled(int[] mapping, int[] nums) {
        int[] res = new int[nums.length];
        int[][] help = new int[nums.length][2];
        for (int i = 0; i < nums.length; i++) {
            int num = nums[i];
            int t = 0;
            if (num == 0) t = mapping[num];
            else {
                int step = 1;
                for (int j = 0; num > 0; j++) {
                    t = mapping[num % 10] * step + t;
                    num /= 10;
                    step *= 10;
                }
            }
            help[i] = new int[]{t, nums[i]};
        }
        Arrays.sort(help, (a, b) -> a[0] - b[0]);
        for (int i = 0; i < help.length; i++) {
            res[i] = help[i][1];
        }
        return res;
    }
```

### [**2192. 有向无环图中一个节点的所有祖先**](https://leetcode-cn.com/problems/all-ancestors-of-a-node-in-a-directed-acyclic-graph/) **** :star::star:

> 给你一个正整数 n ，它表示一个 有向无环图 中节点的数目，节点编号为 0 到 n - 1 （包括两者）。
>
> 给你一个二维整数数组 edges ，其中 edges\[i] = \[fromi, toi] 表示图中一条从 fromi 到 toi 的单向边。
>
> 请你返回一个数组 answer，其中 answer\[i]是第 i 个节点的所有 祖先 ，这些祖先节点 升序 排序。
>
> 如果 u 通过一系列边，能够到达 v ，那么我们称节点 u 是节点 v 的 祖先 节点。

{% hint style="info" %}
中等题，有向无环图输出节点的所有祖先，注意不是直接祖先。用Map\<Integer,TreeSet>保存节点的所有祖先，保证不重复。然后做拓扑排序(防止节点多次入队出队)，依次找出入度为0的节点出队，当找到pre->next 是，对next添加pre的所有祖先，同时需要添加pre。最后将Map转成List返回。
{% endhint %}

```
    static public List<List<Integer>> getAncestors(int n, int[][] edges) {
        Map<Integer, List<Integer>> map = new HashMap<>();
        int[] in = new int[n];
        for (int[] edge : edges) {
            if (!map.containsKey(edge[0])) {
                map.put(edge[0], new ArrayList<>());
            }
            in[edge[1]]++;
            map.get(edge[0]).add(edge[1]);
        }
        List<Set<Integer>> res = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            res.add(new TreeSet<>());
        }
        Queue<Integer> queue = new LinkedList<>();
        for (int i = 0; i < n; i++) {
            if (in[i] == 0) {
                queue.add(i);
            }
        }
        while (!queue.isEmpty()) {
            Integer poll = queue.poll();
            for (Integer next : map.getOrDefault(poll, new ArrayList<>())) {
                res.get(next).addAll(res.get(poll));
                res.get(next).add(poll);
                in[next]--;
                if (in[next] == 0) {
                    queue.add(next);
                }
            }
        }
        List<List<Integer>> ans = new ArrayList<>();
        for (int i = 0; i < res.size(); i++) {
            ans.add(new ArrayList<>(res.get(i)));
        }
        return ans;
    }
```

### [**2193. 得到回文串的最少操作次数**](https://leetcode-cn.com/problems/minimum-number-of-moves-to-make-palindrome/) **** :star::star::star:

> 给你一个只包含小写英文字母的字符串 s 。
>
> 每一次 操作 ，你可以选择 s 中两个 相邻 的字符，并将它们交换。
>
> 请你返回将 s 变成回文串的 最少操作次数 。
>
> 注意 ，输入数据会确保 s 一定能变成一个回文串。
>
> **提示：**
>
> * `1 <= s.length <= 2000`
> * `s` 只包含小写英文字母。
> * `s` 可以通过有限次操作得到一个回文串。

{% hint style="info" %}
困难题，隐隐约约能想到是贪心，但不知道怎么贪。回文串左右对称，可以考虑从两侧依次往内转换成子问题。 从s\[left]开始，贪心的从右侧right位置往左找到等于s\[left]的位置，然后依次交换回s\[right]，记录交换次数，然后『left+1，right-1』，转换成子问题继续求解。若找不到与s\[left]相等的字符，则说明该字符需要放到中间，则计算交换到中间的次数，此时只需要left+1继续求解。证明不了...只能从直觉上感受..肯定是从右往左找交换次数最小。
{% endhint %}

```
    public int minMovesToMakePalindrome(String s) {
        char[] chars = s.toCharArray();
        int n = chars.length;
        int ans = 0;
        for (int i = 0, j = n - 1; i < j; i++) {
            boolean swapped = false;
            for (int k = j; k > i; k--) {
                if (chars[k] == chars[i]) {
                    for (; k < j; k++) {
                        swap(k, k + 1, chars);
                        ans++;
                    }
                    j--;
                    swapped = true;
                    break;
                }
                
            }

            if (!swapped) {
                ans += n / 2 - i;
            }
        }
        return ans;
    }
    void swap(int a, int b, char[] arr) {
        char temp = arr[a];
        arr[a] = arr[b];
        arr[b] = temp;
    }
```
