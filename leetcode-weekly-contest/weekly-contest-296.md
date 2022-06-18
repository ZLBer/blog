---
description: https://leetcode.cn/contest/weekly-contest-296/
---

# Weekly Contest 296

ranking: 109 / 5721



### [**2293. 极大极小游戏**](https://leetcode.cn/problems/min-max-game/) ****&#x20;

{% hint style="info" %}
简单题，模拟
{% endhint %}

```
    public int minMaxGame(int[] nums) {

        while (nums.length > 1) {
            int[] newNum = new int[nums.length / 2];
            for (int i = 0; i < newNum.length; i++) {
                if (i % 2 == 0) {
                    newNum[i] = Math.min(nums[2 * i], nums[2 * i + 1]);
                } else {
                    newNum[i] = Math.max(nums[2 * i], nums[2 * i + 1]);
                }
            }
            nums = newNum;
        }
        return nums[0];
    }
```

### [**2294. 划分数组使最大差为 K**](https://leetcode.cn/problems/partition-array-such-that-maximum-difference-is-k/)

{% hint style="info" %}
中等题，排序，然后间距k为一组
{% endhint %}

```
    public int partitionArray(int[] nums, int k) {
        Arrays.sort(nums);
        int ans = 0;
        int pre = 0;
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] - nums[pre] <= k) {
                continue;
            } else {
                pre = i;
                ans++;
            }
        }
        ans++;

        return ans;
    }

```



### [**2295. 替换数组中的元素**](https://leetcode.cn/problems/replace-elements-in-an-array/)

{% hint style="info" %}
//中等题，map记录数字的index，保证不会重复

&#x20;//或者先将operation操作连成一条链，只需要记录头尾两个操作，中间都是无关操作
{% endhint %}

```
  public int[] arrayChange(int[] nums, int[][] operations) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int[] operation : operations) {
            if (!map.containsKey(operation[0])) {
                map.put(operation[1], operation[0]);
            } else {
                Integer begin = map.get(operation[0]);
                map.remove(operation[0]);
                map.put(operation[1], begin);
            }
        }

        Map<Integer, Integer> mmap = new HashMap<>();
        for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
            mmap.put(entry.getValue(), entry.getKey());
        }

        for (int i = 0; i < nums.length; i++) {
            if (mmap.containsKey(nums[i])) {
                nums[i] = mmap.get(nums[i]);
            }
        }
        return nums;
    }
```

```
    //map保存num,index
   public int[] arrayChange(int[] nums, int[][] operations) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            map.put(nums[i], i);
        }

        for (int[] operation : operations) {
            Integer index = map.get(operation[0]);
            nums[index] = operation[1];
            map.remove(operation[0]);
            map.put(operation[1], index);
        }
        return nums;
    }
```

### [**2296. 设计一个文本编辑器**](https://leetcode.cn/problems/design-a-text-editor/)

{% hint style="info" %}
//困难题，直接用StringBuilder模拟，没想到一次就过了..&#x20;

//还可以用双向链表、对顶栈等
{% endhint %}

```
 class TextEditor {
        StringBuilder sb = new StringBuilder();
        int cur = 0;

        public TextEditor() {

        }

        public void addText(String text) {
            sb.insert(cur, text);
            cur += text.length();
        }

        public int deleteText(int k) {

            int ans = Math.min(k, cur);
            int begin = cur - ans;
            sb.delete(begin, begin + ans);
            cur -= ans;
            return ans;
        }

        public String cursorLeft(int k) {
            int ans = Math.min(k, cur);
            cur -= ans;
            ans = Math.min(cur, 10);
            int begin = cur - ans;
            return sb.substring(begin, begin + ans);
        }

        public String cursorRight(int k) {
            int ans = Math.min(k, sb.length() - cur);
            cur += ans;
            ans = Math.min(cur, 10);
            int begin = cur - ans;
            return sb.substring(begin, begin + ans);
        }
    }
```

