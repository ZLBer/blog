---
description: https://leetcode-cn.com/contest/weekly-contest-285/
---

# Weekly Contest 285

[**2210. 统计数组中峰和谷的数量**](https://leetcode-cn.com/problems/count-hills-and-valleys-in-an-array/)

> 给你一个下标从 0 开始的整数数组 nums 。如果两侧距 i 最近的不相等邻居的值均小于 nums\[i] ，则下标 i 是 nums 中，某个峰的一部分。类似地，如果两侧距 i 最近的不相等邻居的值均大于 nums\[i] ，则下标 i 是 nums 中某个谷的一部分。对于相邻下标 i 和 j ，如果 nums\[i] == nums\[j] ， 则认为这两下标属于 同一个 峰或谷。
>
> 注意，要使某个下标所做峰或谷的一部分，那么它左右两侧必须 都 存在不相等邻居。
>
> 返回 nums 中峰和谷的数量。
>
> **提示：**
>
> * `3 <= nums.length <= 100`
> * `1 <= nums[i] <= 100`

{% hint style="info" %}
简单题，当时打比赛的时候没自己想，直接一通暴力判断峰和谷。其实只要判断下降点和上升点即可，而且也不会有重复。
{% endhint %}

```
    public int countHillValley(int[] nums) {
        int flag=0; // 1表示上升 2表示下降
        int res=0;
        for (int i = 1; i < nums.length; i++) {
               if(nums[i]>nums[i-1]){
                  if(flag==2) res++;
                   flag=1;
               }else if(nums[i]<nums[i-1]){
                   if(flag==1) res++;
                   flag=2;
               }
        }
        return res;
    }
```

[**2211. 统计道路上的碰撞次数**](https://leetcode-cn.com/problems/count-collisions-on-a-road/)

> 在一条无限长的公路上有 n 辆汽车正在行驶。汽车按从左到右的顺序按从 0 到 n - 1 编号，每辆车都在一个 独特的 位置。
>
> 给你一个下标从 0 开始的字符串 directions ，长度为 n 。directions\[i] 可以是 'L'、'R' 或 'S' 分别表示第 i 辆车是向 左 、向 右 或者 停留 在当前位置。每辆车移动时 速度相同 。
>
> 碰撞次数可以按下述方式计算：
>
> 当两辆移动方向 相反 的车相撞时，碰撞次数加 2 。 当一辆移动的车和一辆静止的车相撞时，碰撞次数加 1 。 碰撞发生后，涉及的车辆将无法继续移动并停留在碰撞位置。除此之外，汽车不能改变它们的状态或移动方向。
>
> 返回在这条道路上发生的 碰撞总次数 。
>
> **提示：**
>
> * `1 <= directions.length <= 1`**e**`5`
> * `directions[i]` 的值为 `'L'`、`'R'` 或 `'S'`

{% hint style="info" %}
中等题，一般的想法就是模拟，用变量暂存向右转的车的数目，可能发生碰撞亦可能不。停止的车或向左转的就发生碰撞。

然而，有更简洁的想法，先去除最左边向左转的，再去除最右边向右转的，中间剩下的一定会发生彭专，切向左向右转的各累计一次，暂停的车不累加。
{% endhint %}

```
   //结果又被降维打击了
    public int countCollisions(String directions) {
        int l = 0, r = directions.length() - 1;
        while (l <= r && directions.charAt(l) == 'L') ++l;
        while (l <= r && directions.charAt(r) == 'R') --r;
        int res = 0;
        //中间的lr一定会发生碰撞,各自累加一次，静止的相当与不累加
        for (int i = l; i <= r; ++i) if (directions.charAt(i) == 'L' || directions.charAt(i) == 'R') ++res;
        return res;
    }
    //周赛的写法
/*    public int countCollisions(String directions) {
        int left = 0;//向右转的有多少
        int ans = 0;
        boolean block = false; //之前是否发生碰撞
        for (int i = 0; i < directions.length(); i++) {
            char c = directions.charAt(i);
            if (c == 'R') { //加入向右转的车辆
                left++;
            } else if (c == 'S') { //当前车辆提在原地，之前向右转的都发生碰撞
                ans += left;
                left = 0;
                block = true;
            } else if (c == 'L') { //当前车辆向左转
                if (left > 0) { //和向右转的发生碰撞 +2
                    ans += 2;
                    ans += (left - 1);
                    left = 0;
                    block = true;
                } else if (block) {// 否则，查看之前是否发生碰撞
                    ans++;
                }
            }
        }
        return ans;
    }*/
```

[**2212. 射箭比赛中的最大得分**](https://leetcode-cn.com/problems/maximum-points-in-an-archery-competition/)

> Alice 和 Bob 是一场射箭比赛中的对手。比赛规则如下：
>
> Alice 先射 numArrows 支箭，然后 Bob 也射 numArrows 支箭。 分数按下述规则计算： 箭靶有若干整数计分区域，范围从 0 到 11 （含 0 和 11）。 箭靶上每个区域都对应一个得分 k（范围是 0 到 11），Alice 和 Bob 分别在得分 k 区域射中 ak 和 bk 支箭。如果 ak >= bk ，那么 Alice 得 k 分。如果 ak < bk ，则 Bob 得 k 分 如果 ak == bk == 0 ，那么无人得到 k 分。 例如，Alice 和 Bob 都向计分为 11 的区域射 2 支箭，那么 Alice 得 11 分。如果 Alice 向计分为 11 的区域射 0 支箭，但 Bob 向同一个区域射 2 支箭，那么 Bob 得 11 分。
>
> 给你整数 numArrows 和一个长度为 12 的整数数组 aliceArrows ，该数组表示 Alice 射中 0 到 11 每个计分区域的箭数量。现在，Bob 想要尽可能 最大化 他所能获得的总分。
>
> 返回数组 bobArrows ，该数组表示 Bob 射中 0 到 11 每个 计分区域的箭数量。且 bobArrows 的总和应当等于 numArrows 。
>
> 如果存在多种方法都可以使 Bob 获得最大总分，返回其中 任意一种 即可。
>
> **提示：**
>
> * `1 <= numArrows <= 105`
> * aliceArrows.length == bobArrows.length == 12
> * &#x20;0 <= aliceArrows\[i], bobArrows\[i] <= numArrows
> * &#x20;sum(aliceArrows\[i]) == numArrows

{% hint style="info" %}
中等题，比赛的时候把数据量想错了，只有2的11次方，以为成了10的11次方。然后以为暴力一定出不来，就用了01背包的做法。dp\[i]\[j]表示第i局的时候用j根弓箭能得到的最大分数。求这个最大分数不难，然后求状态的时候想了好久。

其实暴力做起来更简单：二级制枚举，将所有的比分状态（ 1的12次方)进行遍历求出最大的积分，同时记录状态即可。
{% endhint %}

```
  //01背包
    static public int[] maximumBobPoints(int numArrows, int[] aliceArrows) {
        int[][] dp = new int[aliceArrows.length][numArrows + 1];
        for (int i = 1; i < dp.length; i++) {
            int score = i;
            int arrow = aliceArrows[i] + 1;
            for (int j = 0; j < dp[0].length; j++) {
                //不赢得这个积分
                dp[i][j] = dp[i - 1][j];
                if (j >= arrow) {
                    //与赢得这个积分判断
                    dp[i][j] = Math.max(Math.max(dp[i][j], dp[i][j - 1]), dp[i - 1][j - arrow] + score);
                }
            }
        }
        int[] res = new int[aliceArrows.length];
        int j = numArrows;
        //逆序查找赢了哪些积分
        for (int i = res.length - 1; i > 0; i--) {
            int arrow = aliceArrows[i] + 1;
            if (j - arrow >= 0 && dp[i - 1][j - arrow] + i == dp[i][j]) {
                res[i] = arrow;
                j -= arrow;
            }

        }
        //将剩下的给0分，保证弓箭用完
        res[0] = j;
        return res;
    }
```

[**2213. 由单个字符重复的最长子字符串**](https://leetcode-cn.com/problems/longest-substring-of-one-repeating-character/)

> 给你一个下标从 0 开始的字符串 s 。另给你一个下标从 0 开始、长度为 k 的字符串 queryCharacters ，一个下标从 0 开始、长度也是 k 的整数 下标 数组 queryIndices ，这两个都用来描述 k 个查询。
>
> 第 i 个查询会将 s 中位于下标 queryIndices\[i] 的字符更新为 queryCharacters\[i] 。
>
> 返回一个长度为 k 的数组 lengths ，其中 lengths\[i] 是在执行第 i 个查询 之后 s 中仅由 单个字符重复 组成的 最长子字符串 的 长度 。
>
> **提示：**
>
> * `1 <= s.length <= 1e5`
> * s 由小写英文字母组成
> * &#x20;k == queryCharacters.length == queryIndices.length&#x20;
> * 1 <= k <= 1e5
> * &#x20;queryCharacters 由小写英文字母组成
> * &#x20;0 <= queryIndices\[i] < s.length

{% hint style="info" %}
困难题，周赛的时候没做出来，后面看了一下思路，自己也写了好久，是最近写的最长的题目了。

纯模拟，思路如下：用TreeMap保存区间的左区间和区间字符，另用一个TreeMap来保存区间长度和频率 counter。遍历queryIndices:

&#x20;           1.找出indice的区间，将区间分成三部分(删除原区间，添加左中右三个区间，可能<3个区间，因为indice可能位于区间左端点\右断点)，且需要更新counter(删除原区间长度，更新三个区间长度)。

&#x20;           2.判断左区间和中间区间能不能合并，能合并的话，需要再次更新TreeMap和counter

&#x20;           3.判断中间区间和右区间能不能合并，这里需要重新获取indice的区间，因为左区间可能发生合并。
{% endhint %}

```
    static public int[] longestRepeating(String s, String queryCharacters, int[] queryIndices) {
        TreeMap<Integer, Character> map = new TreeMap<>();
        TreeMap<Integer, Integer> counter = new TreeMap<>();
        char[] chars = s.toCharArray();
        int n = s.length();
        for (int i = 0; i < chars.length; ) {
            int j = 0;
            while (i + j < chars.length && chars[i] == chars[i + j]) {
                j++;
            }
            map.put(i, chars[i]);
            counter.put(j, counter.getOrDefault(j, 0) + 1);
            i += j;
        }
        int[] res = new int[queryIndices.length];
        map.put(n, ' ');//处理边界
        map.put(-1, ' ');
        for (int i = 0; i < res.length; i++) {
            int index = queryIndices[i];
            //要改的字符不变
            if (chars[index] != queryCharacters.charAt(i)) {
                chars[index] = queryCharacters.charAt(i);
                Integer lessOrEqual = map.floorKey(index);
                Integer greater = map.higherKey(index);

                int len = greater - lessOrEqual;
                counter.put(len, counter.get(len) - 1);
                if (counter.get(len) == 0) {
                    counter.remove(len);
                }

                //左区间
                if (index != lessOrEqual) {
                    counter.put(index - lessOrEqual, counter.getOrDefault(index - lessOrEqual, 0) + 1);
                    map.put(index, chars[index]);
                }
                //中间
                counter.put(1, counter.getOrDefault(1, 0) + 1);
                //右区间
                if (index + 1 != greater) {
                    counter.put(greater - index - 1, counter.getOrDefault(greater - index - 1, 0) + 1);
                    map.put(index + 1, chars[index + 1]);
                }
                //合并左区间
                if (index > 0 && chars[index] == chars[index - 1]) {
                    Integer lowerKey = map.lowerKey(index);
                    map.remove(index);
                    counter.put(index - lowerKey, counter.getOrDefault(index - lowerKey, 0) - 1);
                    if (counter.get(index - lowerKey) == 0) {
                        counter.remove(index - lowerKey);
                    }
                    counter.put(1, counter.getOrDefault(1, 0) - 1);
                    if (counter.get(1) == 0) {
                        counter.remove(1);
                    }
                    counter.put(index - lowerKey + 1, counter.getOrDefault(index - lowerKey + 1, 0) + 1);
                }
                //合并右区间
                if (index < n - 1 && chars[index] == chars[index + 1]) {
                    Integer higherKey = map.higherKey(index);
                    map.remove(higherKey);
                    Integer hhigherKey = map.higherKey(higherKey);
                    counter.put(hhigherKey - higherKey, counter.getOrDefault(hhigherKey - higherKey, 0) - 1);
                    if (counter.get(hhigherKey - higherKey) == 0) {
                        counter.remove(hhigherKey - higherKey);
                    }

                    Integer floorKey = map.floorKey(index);
                    counter.put(hhigherKey - floorKey, counter.getOrDefault(hhigherKey - floorKey, 0) + 1);
                    counter.put(index - floorKey + 1, counter.getOrDefault(index - floorKey + 1, 0) - 1);
                    if (counter.get(index - floorKey + 1) == 0) {
                        counter.remove(index - floorKey + 1);
                    }
                }

            }
            res[i] = counter.lastKey();

        }
        return res;
    }
```
