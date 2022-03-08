---
description: https://leetcode-cn.com/contest/weekly-contest-283/
---

# Weekly Contest 283

### [**2194. Excel 表中某个范围内的单元格**](https://leetcode-cn.com/problems/cells-in-a-range-on-an-excel-sheet/)  ****  :star:****

> Excel 表中的一个单元格 (r, c) 会以字符串 "" 的形式进行表示，其中：
>
> 即单元格的列号 c 。用英文字母表中的 字母 标识。 例如，第 1 列用 'A' 表示，第 2 列用 'B' 表示，第 3 列用 'C' 表示，以此类推。 即单元格的行号 r 。第 r 行就用 整数 r 标识。 给你一个格式为 ":" 的字符串 s ，其中 表示 c1 列， 表示 r1 行， 表示 c2 列， 表示 r2 行，并满足 r1 <= r2 且 c1 <= c2 。
>
> 找出所有满足 r1 <= x <= r2 且 c1 <= y <= c2 的单元格，并以列表形式返回。单元格应该按前面描述的格式用 字符串 表示，并以 非递减 顺序排列（先按列排，再按行排）。
>
> **提示：**
>
> * `s.length == 5`
> * `'A' <= s[0] <= s[3] <= 'Z'`
> * `'1' <= s[1] <= s[4] <= '9'`
> * `s` 由大写英文字母、数字、和 `':'` 组成

{% hint style="info" %}
简单题，两个维度遍历就可:

&#x20;  for  A->Z

&#x20;     for 1->n
{% endhint %}

```
   public List<String> cellsInRange(String s) {
       int a=s.charAt(0)-'A';
       int b=s.charAt(1)-'0';
       int c=s.charAt(3)-'A';
       int d=s.charAt(4)-'0';
       List<String> res=new ArrayList<>();
       for(int i=a;i<=c;i++){
           for(int j=b;j<=d;j++){
               String t=(char)(i+'A')+""+j;
               res.add(t);
           }
       }
       return res;
    }
```

### [**2195. 向数组中追加 K 个整数**](https://leetcode-cn.com/problems/append-k-integers-with-minimal-sum/)  ****  :star::star:

> 给你一个整数数组 nums 和一个整数 k 。请你向 nums 中追加 k 个 未 出现在 nums 中的、互不相同 的 正 整数，并使结果数组的元素和 最小 。
>
> 返回追加到 nums 中的 k 个整数之和。
>
> **提示：**
>
> * `1 <= nums.length <= 1e5`
> * `1 <= nums[i], k <= 1e9`

{% hint style="info" %}
中等题，添加k个元素使结果数组的元素和最小，其实就是从1开始添加k个元素，需要注意这k个元素不能在nums数组中，最理想的情况是添加(1..k)k个元素，但其中nums数组可能存在这些元素。

一种可行的方法是先从1加到k得到sum，然后将nums数组排序，从小到大开始遍历，遇到比k小的就说明nums存在重复元素，需要减去这个元素，然后计算新的上界令k++，然后将新的k加到sum里。

需要注意这里k的最大值是1e9，所以从1遍历到k相加的复杂度很高，需要等差数列求和`Sn=(a1+qn)*n/2`;
{% endhint %}

```
  public long minimalKSum(int[] nums, int k) {
        long sum = 0;
        long kk = k;//
        sum=kk*(kk+1)/2;
        Arrays.sort(nums);
        for (int i = 0; i < nums.length; i++) {
            if (i > 0 && nums[i] == nums[i - 1]) continue;
            if (nums[i] <= kk) {
                sum -= nums[i];
                kk++;
                sum += kk;
            } else {
                break;
            }
        }
        return sum;
    }
```

### [**2196. 根据描述创建二叉树**](https://leetcode-cn.com/problems/create-binary-tree-from-descriptions/)  ****  :star::star:

> 给你一个二维整数数组 descriptions ，其中 descriptions\[i] = \[parenti, childi, isLefti] 表示 parenti 是 childi 在 二叉树 中的 父节点，二叉树中各节点的值 互不相同 。此外：
>
> 如果 isLefti == 1 ，那么 childi 就是 parenti 的左子节点。 如果 isLefti == 0 ，那么 childi 就是 parenti 的右子节点。 请你根据 descriptions 的描述来构造二叉树并返回其 根节点 。
>
> 测试用例会保证可以构造出 有效 的二叉树。
>
> **提示：**
>
> * `1 <= descriptions.length <= 1e4`
> * `descriptions[i].length == 3`
> * `1 <= parenti, childi <= 1e5`
> * `0 <= isLefti <= 1`
> * `descriptions` 所描述的二叉树是一棵有效二叉树

{% hint style="info" %}
中等题，就是把\[from,to,value]的数据结构转成二叉树的数据结构，入度为0的就是根节点，所以在遍历过程中需要找出根节点。周赛的时候我是先建图，然后再广度优先建立二叉树。其实可以再简单点写，遍历descriptions数组的时候就可以建立起二叉树关系。
{% endhint %}

```java
     public TreeNode createBinaryTree(int[][] descriptions) {
        Map<Integer, int[]> map = new HashMap<>();
        Set<Integer> set=new HashSet<>();
        for (int[] description : descriptions) {
            if (!map.containsKey(description[0])) map.put(description[0], new int[2]);
            map.get(description[0])[description[2]] = description[1];
            set.add(description[1]);
        }
        int rootV=-1;
        for (int[] description : descriptions) {
            if(!set.contains(description[0])){
                rootV=description[0];
                break;
            }
        }
      TreeNode root=new TreeNode(rootV);  
      Queue<TreeNode> queue=new LinkedList<>();
      queue.add(root);
      while (!queue.isEmpty()){
          TreeNode poll = queue.poll();
          int[] aDefault = map.getOrDefault(poll.val, new int[2]);
         if(aDefault[1]>0){
           TreeNode node=new TreeNode(aDefault[1]);  
           poll.left=node;
           queue.add(node);
         }
         if(aDefault[0]>0){
             TreeNode node=new TreeNode(aDefault[0]);
             poll.right=node;
             queue.add(node);   
         }
      }
      return root;
    }
```

### [**2197. 替换数组中的非互质数**](https://leetcode-cn.com/problems/replace-non-coprime-numbers-in-array/)  ****  :star::star::star:

> 给你一个整数数组 nums 。请你对数组执行下述操作：
>
> 从 nums 中找出 任意 两个 相邻 的 非互质 数。 如果不存在这样的数，终止 这一过程。 否则，删除这两个数，并 替换 为它们的 最小公倍数（Least Common Multiple，LCM）。 只要还能找出两个相邻的非互质数就继续 重复 这一过程。 返回修改后得到的 最终 数组。可以证明的是，以 任意 顺序替换相邻的非互质数都可以得到相同的结果。
>
> 生成的测试用例可以保证最终数组中的值 小于或者等于 108 。
>
> 两个数字 x 和 y 满足 非互质数 的条件是：GCD(x, y) > 1 ，其中 GCD(x, y) 是 x 和 y 的 最大公约数 。
>
> **提示：**
>
> * `1 <= nums.length <= 1e5`
> * `1 <= nums[i] <= 1e5`
> * 生成的测试用例可以保证最终数组中的值 **小于或者等于** `108` 。

{% hint style="info" %}
困难题，简单描述就是按照GCD>=1两两合并成LCM，直至不能再合并，返回结果数组。

想不到用栈是不应该的，每次将合并后的值或者不能合并的值都入栈，每次取栈顶的两个数判断能不能合并，如果能就继续出栈。直至数组遍历完成，那栈里的数就是结果数组。

千万不要想着每次遍历完成去复制数组，然后继续遍历数组去判断，时间复杂度太高了，周赛犯了不应该的错。
{% endhint %}

```
   static public List<Integer> replaceNonCoprimes(int[] nums) {
        List<Integer> list = new ArrayList<>();

        Deque<Integer> deque = new LinkedList<>();
            for (int i = 0; i < nums.length; i++) {
            int num = nums[i];
            deque.add(num);
            while (deque.size() >= 2) {
                int a = deque.pollLast();
                int b = deque.pollLast();
                int check = check(a, b);
                if (check == -1) {
                    deque.add(b);
                    deque.add(a);
                    break;
                } else {
                    deque.add(check);
                }
            }
        }
        List<Integer> res=new ArrayList<>();
        while (!deque.isEmpty()){
            res.add(deque.pollFirst());
        }
      return res;
    }
        static int check(long a, long b) {
        long g = gcd(a, b);
        if (g <= 1) return -1;
        if (a != 0 || b != 0) {
            return (int) (a * b / g);
        } else {
            return 0;
        }
    }

    static long gcd(long a, long b) {
        while (b != 0) {
            long temp = a % b;
            a = b;
            b = temp;
        }
        return a;
    }
```

