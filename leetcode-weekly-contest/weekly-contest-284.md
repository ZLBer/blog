---
description: https://leetcode-cn.com/contest/weekly-contest-284/
---

# Weekly Contest 284

### [**2200. 找出数组中的所有 K 近邻下标**](https://leetcode-cn.com/problems/find-all-k-distant-indices-in-an-array/) **** :star:

> 给你一个下标从 0 开始的整数数组 nums 和两个整数 key 和 k 。K 近邻下标 是 nums 中的一个下标 i ，并满足至少存在一个下标 j 使得 |i - j| <= k 且 nums\[j] == key 。
>
> 以列表形式返回按 递增顺序 排序的所有 K 近邻下标。
>
> **提示：**
>
> * `1 <= nums.length <= 1000`
> * `1 <= nums[i] <= 1000`
> * `key` 是数组 `nums` 中的一个整数
> * `1 <= k <= nums.length`

{% hint style="info" %}
简单题，判断i的左k个和右k个是否存在j，使得nums\[j] == key，可以直接循环，也可以先求出左右边界循环。
{% endhint %}

```
      public List<Integer> findKDistantIndices(int[] nums, int key, int k) {
        List<Integer> res = new ArrayList<>();
        for (int i = 0; i < nums.length; i++) {
            int left = Math.max(0, i - k);
            int right = Math.min(nums.length - 1, i + k);
            for (int j = left; j <= right; j++) {
                if (nums[j] == key) {
                    res.add(i);
                    break;
                }
            }
        }
        return res;
    }
```

### [**2201. 统计可以提取的工件**](https://leetcode-cn.com/problems/count-artifacts-that-can-be-extracted/) **** :star::star:

> 存在一个 n x n 大小、下标从 0 开始的网格，网格中埋着一些工件。给你一个整数 n 和一个下标从 0 开始的二维整数数组 artifacts ，artifacts 描述了矩形工件的位置，其中 artifacts\[i] = \[r1i, c1i, r2i, c2i] 表示第 i 个工件在子网格中的填埋情况：
>
> (r1i, c1i) 是第 i 个工件 左上 单元格的坐标，且 (r2i, c2i) 是第 i 个工件 右下 单元格的坐标。 你将会挖掘网格中的一些单元格，并清除其中的填埋物。如果单元格中埋着工件的一部分，那么该工件这一部分将会裸露出来。如果一个工件的所有部分都都裸露出来，你就可以提取该工件。
>
> 给你一个下标从 0 开始的二维整数数组 dig ，其中 dig\[i] = \[ri, ci] 表示你将会挖掘单元格 (ri, ci) ，返回你可以提取的工件数目。
>
> 生成的测试用例满足：
>
> 不存在重叠的两个工件。 每个工件最多只覆盖 4 个单元格。 dig 中的元素互不相同。
>
> ****
>
> **提示：**
>
> * `1 <= n <= 1000`
> * `1 <= artifacts.length, dig.length <= min(n2, 1e5)`
> * `artifacts[i].length == 4`
> * `dig[i].length == 2`
> * `0 <= r1i, c1i, r2i, c2i, ri, ci <= n - 1`
> * `r1i <= r2i`
> * 不存在重叠的两个工件
> * 每个工件 **最多** 只覆盖 `4` 个单元格
> * `dig` 中的元素互不相同

{% hint style="info" %}
中等题，判断按照dig数组进行挖掘后，可以挖出几个零件。主要是怎么记录dig问题，因为dig有xy两个维度，可以搞成x-y的string形式存储，这里我用了x\*1000+y的int形式存储，优于string，乘1000是因为xy的最大值为1000, 所以考虑如此映射。最后遍历artifacts判断是否全部存在这些坐标即可。
{% endhint %}

```
   public int digArtifacts(int n, int[][] artifacts, int[][] dig) {
        Set<Integer> set = new HashSet<>();
        for (int[] ints : dig) {
            int key = ints[0] * 1000 + ints[1];
            set.add(key);
        }
        int ans = 0;
        for (int[] artifact : artifacts) {
            int r1 = artifact[0], c1 = artifact[1];
            int r2 = artifact[2], c2 = artifact[3];
            boolean flag = true;
            for (int i = r1; i <= r2; i++) {
                for (int j = c1; j <= c2; j++) {
                    int key = i * 1000 + j;
                    if (!set.contains(key)) {
                        flag = false;
                        break;
                    }
                }
                if (!flag) break;
            }
            if (flag) ans++;
        }
        return ans;
    }
```

### [**2202. K 次操作后最大化顶端元素**](https://leetcode-cn.com/problems/maximize-the-topmost-element-after-k-moves/) **** :star::star:

> 给你一个下标从 0 开始的整数数组 nums ，它表示一个 栈 ，其中 nums\[0] 是栈顶的元素。
>
> 每一次操作中，你可以执行以下操作 之一 ：
>
> 如果栈非空，那么 删除 栈顶端的元素。 如果存在 1 个或者多个被删除的元素，你可以从它们中选择任何一个，添加 回栈顶，这个元素成为新的栈顶元素。 同时给你一个整数 k ，它表示你总共需要执行操作的次数。
>
> 请你返回 恰好 执行 k 次操作以后，栈顶元素的 最大值 。如果执行完 k 次操作以后，栈一定为空，请你返回 -1 。
>
> **提示：**
>
> * `1 <= nums.length <= 1e5`
> * `0 <= nums[i], k <= 1e9`

{% hint style="info" %}
中等题，一度以为自己看错题目了，实际坑比较多。加上周赛事翻译有很大问题。就说有个栈，你每次可以进行两种操作，要么弹出栈顶元素，要么将之前弹出的元素再放到栈顶，问进行k次操作，返回栈顶最大值。

这样考虑，删除前k-1个元素，第k步返回前k-1的最大值。或者，删除前k个，返回k+1的值。剩下的就是比较这两种情况的最大值。需要考虑的边界情况比较多，k==0时不能操作，k是奇数并且nums只有一个元素，这时候栈一定为空，返回-1。
{% endhint %}

```
    public int maximumTop(int[] nums, int k) {
        int n = nums.length;
        if (k == 0) return nums[0];
        if (k % 2 == 1 && nums.length == 1) return -1;
        int max = -1;
        if (k < n) {
            max = nums[k];
        }
        for (int i = 0; i < Math.min(k - 1, n); i++) {
            max = Math.max(nums[i], max);
        }
        return max;
    }
```

### [**2203. 得到要求路径的最小带权子图**](https://leetcode-cn.com/problems/minimum-weighted-subgraph-with-the-required-paths/) **** :star::star::star:

> 给你一个整数 n ，它表示一个 带权有向 图的节点数，节点编号为 0 到 n - 1 。
>
> 同时给你一个二维整数数组 edges ，其中 edges\[i] = \[fromi, toi, weighti] ，表示从 fromi 到 toi 有一条边权为 weighti 的 有向 边。
>
> 最后，给你三个 互不相同 的整数 src1 ，src2 和 dest ，表示图中三个不同的点。
>
> 请你从图中选出一个 边权和最小 的子图，使得从 src1 和 src2 出发，在这个子图中，都 可以 到达 dest 。如果这样的子图不存在，请返回 -1 。
>
> 子图 中的点和边都应该属于原图的一部分。子图的边权和定义为它所包含的所有边的权值之和。
>
> **提示：**
>
> * `3 <= n <= 1e5`
> * `0 <= edges.length <= 1e5`
> * `edges[i].length == 3`
> * `0 <= fromi, toi, src1, src2, dest <= n - 1`
> * `fromi != toi`
> * `src1` ，`src2` 和 `dest` 两两不同。
> * `1 <= weight[i] <= 1e5`

{% hint style="info" %}
困难题，思路倒是不难想啊，就求dis(src1->i)+dis(src2->i)+dis(i->dest)的距离，然后遍历求全局最小值，这样题目就变成了最短路径问题。一看是多源最短路径，然后直接开始Floyd算法了，最后肯定超时了啊，三次方的时间复杂度确实很高了。然后一想其实就是求三个单源最短路径啊，肯定要用Dijkstra啊。然后用优先队列进行Dijkstra算法，需要注意求dis(i->dest)需要将图反向。

奈何Dijkstra写起来不顺畅，周赛五分钟结束才做出来。
{% endhint %}

```
 //三次Dijkstra算法
 static public long minimumWeight(int n, int[][] edges, int src1, int src2, int dest) {
        Map<Integer, List<int[]>> map = new HashMap<>();
        for (int[] edge : edges) {
            if (!map.containsKey(edge[0])) {
                map.put(edge[0], new ArrayList<>());
            }
            map.get(edge[0]).add(new int[]{edge[1], edge[2]});
        }
        long[] srcc1 = dijstra(n, edges, src1, map);
        long[] srcc2 = dijstra(n, edges, src2, map);
        Map<Integer, List<int[]>> map1 = new HashMap<>();
        for (int[] edge : edges) {
            if (!map1.containsKey(edge[1])) {
                map1.put(edge[1], new ArrayList<>());
            }
            map1.get(edge[1]).add(new int[]{edge[0], edge[2]});
        }
        long[] destt = dijstra(n, edges, dest, map1);
        long res = Long.MAX_VALUE;
        for (int i = 0; i < n; i++) {
            if (srcc1[i] == MAX || srcc2[i] == MAX || destt[i] == MAX) continue;
            long total = srcc1[i] + srcc2[i] + destt[i];
            res = Math.min(total, res);
        }
        return res==Long.MAX_VALUE?-1:res;
    }

    static long MAX = Long.MAX_VALUE / 2;

    static long[] dijstra(int n, int[][] edges, long from, Map<Integer, List<int[]>> map) {
        long[] dp = new long[n];
        Arrays.fill(dp, MAX);
        PriorityQueue<long[]> queue = new PriorityQueue<>((a, b) -> (int) (a[1] - b[1]));
        queue.add(new long[]{from, 0});
        while (!queue.isEmpty()) {
            long[] poll = queue.poll();
            int index = (int) poll[0];
            if(dp[index]!=MAX) continue;
            dp[index]=poll[1];
            for (int[] next : map.getOrDefault(index, new ArrayList<>())) {
                long cost = next[1] + poll[1];
                queue.add(new long[]{next[0], cost});
            }
        }
        return dp;
    }
```

```
//超时的Floyd算法
static   public long minimumWeight(int n, int[][] edges, int src1, int src2, int dest) {
       long[][] floyd = new long[n][n];
       for (int[] edge : edges) {
           if (floyd[edge[0]][edge[1]] == 0) {
               floyd[edge[0]][edge[1]] = edge[2];
           } else {
               floyd[edge[0]][edge[1]] = Math.min(edge[2], floyd[edge[0]][edge[1]]);
           }
       }
           for (int k = 0; k < n; k++) {
               for (int i = 0; i < n; i++) {
                   for (int j = 0; j < n; j++) {
                       if (i == j || i == k || j == k) {
                           continue;
                       }
                       if (floyd[i][k] != 0 && floyd[k][j] != 0) {
                           long total = floyd[i][k] + floyd[k][j];
                           if (total < floyd[i][j] || floyd[i][j] == 0)
                               floyd[i][j] = total;
                       }
                   }

               }
           }

       long res = Long.MAX_VALUE;
       for (int i = 0; i < floyd.length; i++) {
           for (int j = 0; j < floyd.length; j++) {
               if (i == j) continue;
               if (floyd[i][j] == 0) {
                   floyd[i][j] = -1;
               }
           }
       }
       for (int i = 0; i < n; i++) {
           if( floyd[src1][i]==-1|| floyd[src2][i]==-1|| floyd[i][dest]==-1)continue;
           long total = floyd[src1][i] + floyd[src2][i] + floyd[i][dest];
           res=Math.min(total,res);
       }
       return res==Long.MAX_VALUE?-1:res;
   }
```
