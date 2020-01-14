# 算法

## 数据结构

### 查找
#### 二分查找
```Java
private static int binarySearch(int[] arr, int key) {
	int min = 0;
	int max = arr.length - 1;
	int mid = (max + min)/2;
	while(min <= max) {
		if(arr[mid] < key) {
			 min = mid + 1;
		}else if(arr[mid] > key) {
			 max = mid - 1;
		}else {
			 return mid;
		}
			mid = (max + min)/2;
		}
		return -1;
	}
```
#### 快速选择算法
[参考](https://zhuanlan.zhihu.com/p/64627590)
Top K 问题的最优解 - 快速选择算法（Quickselect）
**解法一**
快速排序算法
**解法二**
最小堆算法
**解法三**
快速选择
[快速选择算法](https://www.jianshu.com/p/52f90fe2b141)

### 排序

#### 冒泡排序
[参考](https://www.jianshu.com/p/88759596c944)
```java
private static void sort(int[] array) throws Exception {
     if (array == null || array.length == 0) {
         throw new Exception("the array is null or no element...");
     }
     System.out.println("冒泡排序优化前...");
     int n = array.length;
     for (int i = 0; i < n; i++) {
         for (int j = 0; j < n - i; j++) {
             if (array[j] > array[j + 1]) {
                 swap(array, j, j + 1);
             }
         }
         System.out.println("第" + (i + 1) + "轮后: " + Arrays.toString(array));
     }
 }

 private void swap(int[] array, int a, int b) {
        if (a == b) {
            return;
        }
        int temp = array[a];
        array[a] = array[b];
        array[b] = temp;
    }
```

优化版本
```java
public static void bubbleSort(int[] table){
  //是否进行数据交换
  boolean exchange = true;
  for(int i = 1;i<table.length&&exchange;i++){ //有交换时再进行下一趟，最多n-1趟
    exchange = false;
    for(int j = 0 ; j<table.length-i;j++){
      if(table[j]>table[j+1]){
        int temp =table[j];
        table[j] =table[j+1];
        table[j+1] = temp;
        exchange = true;
      }
    }
  }
}
```
#### 快速排序
[参考](https://blog.csdn.net/wehung/article/details/82704565)

```java
void quick_sort(int s[], int l, int r){
    if (l < r){
        //Swap(s[l], s[(l + r) / 2]);
        //将中间的这个数和第一个数交换 参见注1
        int i = l, j = r, x = s[l];
        while (i < j){
            // 从右向左找第一个小于x的数
            while(i < j && s[j] >= x) {
                j--;
            }
            if(i < j) {
                s[i++] = s[j];
            }
            // 从左向右找第一个大于等于x的数
            while(i < j && s[i] < x){
                i++;
            }
            if(i < j) {
                s[j--] = s[i];
            }
        }
        s[i] = x;
        quick_sort(s, l, i - 1); // 递归调用
        quick_sort(s, i + 1, r);
    }
}
```

进阶版
```java
public static void  quickSort(int[] table){ //快速排序
  quickSort(table,0,table.length-1);
}
//一趟快排，begin、end指定序列的下界和上届，递归算法
privte static void quickSort(int[] table, int begin,int end){
  if(begin < end){
    int i = begin;
    int j = end;
    int vot = table[i];
    while(i!=j){
      while(i<j && vot<=table[j]){
        j--;
      }
      if(i<j){
        table[i++]=table[j];
      }
      while(i<j && table[i]<=vot){
        i++;
      }
      if(i<j){
        table[j--]=table[i];
      }
    }
    table[i] = vot;
    quickSort(table,begin,j-1);
    quickSort(table,i+1,end);
  }
}
```


#### 堆排序
堆排序是完全二叉树的应用，是充分利用完全二叉树特性的一种选择排序。



## leetcode

### 1 两数相加
**描述**
给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。
你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

**示例**
```Java
给定 nums = [2, 7, 11, 15], target = 9
因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

**Solution**
```Java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        for(int i=0;i<nums.length;i++){
            for(int j=i+1;j<nums.length;j++){
                if(nums[j] == target - nums[i]){
                    return new int[]{i,j};
                }
            }
        }
        return new int[2];
    }
}
```

### 2 两数相加
给出两个 非空 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 逆序 的方式存储的，并且它们的每个节点只能存储 一位 数字。
如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。
您可以假设除了数字 0 之外，这两个数都不会以 0 开头。

**举例**
输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807

**Solution**:
```Java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head = new ListNode(0);
        ListNode curr = head;
        int temp = 0;
        int carry =0;
        while(l1 != null || l2 != null){
            int val1 = 0;
            int val2 = 0;
            if(l1 != null){
                val1 = l1.val;
                l1 = l1.next;
            }
            if(l2 != null){
                val2 = l2.val;
                l2 = l2.next;
            }
            int sum = val1+val2+carry;
            carry = sum/10;
            curr.next = new ListNode(sum%10);;
            curr = curr.next;
        }
        if(carry==1){
            curr.next = new ListNode(1);
        }
        return head.next;
    }
}
```
### 25. K 个一组翻转链表
**描述：**
给你一个链表，每 k 个节点一组进行翻转，请你返回翻转后的链表。
k 是一个正整数，它的值小于或等于链表的长度。
如果节点总数不是 k 的整数倍，那么请将最后剩余的节点保持原有顺序。
**示例：**
```Java
给定这个链表：1->2->3->4->5
当 k = 2 时，应当返回: 2->1->4->3->5
当 k = 3 时，应当返回: 3->2->1->4->5
```
**说明：**
- 你的算法只能使用常数的额外空间。
- 你不能只是单纯的改变节点内部的值，而是需要实际的进行节点交换。

**Solution:**
```Java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode pre = dummy;
        ListNode end = dummy;
        while(end!=null){
            for(int i = 0;i<k&&end!=null;i++){
                end = end.next;
            }
            if(end == null){
                break;
            }
            ListNode start = pre.next;
            ListNode next = end.next;
            end.next = null;
            pre.next = reverse(start);
            start.next = next;
            pre = start;
            end = pre;
        }
        return dummy.next;
    }

    public ListNode  reverse(ListNode start){
        ListNode pre = null;
        ListNode curr = start;
        while(curr !=null){
            ListNode next = curr.next;
            curr.next = pre;
            pre = curr;
            curr = next;
        }
        return pre;
    }
}
```


### 70 爬楼梯
假设你正在爬楼梯。需要 n 阶你才能到达楼顶。
每次你可以爬 1 或 2 个台阶。你有多少种不同的方法可以爬到楼顶呢？
注意：给定 n 是一个正整数。

**示例 1：**
```java
输入： 2
输出： 2
解释： 有两种方法可以爬到楼顶。
1.  1 阶 + 1 阶
2.  2 阶
```
**示例 2：**
```java
输入： 3
输出： 3
解释： 有三种方法可以爬到楼顶。
1.  1 阶 + 1 阶 + 1 阶
2.  1 阶 + 2 阶
3.  2 阶 + 1 阶
```
**思路**
n阶台阶的走法，由n-1种台阶的走法加上n-2种台阶的走法
**Solution**
```Java
class Solution {
    public int climbStairs(int n) {
        if(n == 1){
            return 1;
        }
        if(n == 2){
            return 2;
        }
        int one = 1;
        int two = 2;
        for(int i =3;i<=n;i++){
            int temp = one;
            one = two;
            two = temp + two;
        }
        return two;
    }
}
```

linux 统计相当uri的多少，并排序。如何解决hash函数的相关问题，hash碰撞的相关问题。

### 104. 二叉树的最大深度
**描述：**
给定一个二叉树，找出其最大深度。
二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。
说明: 叶子节点是指没有子节点的节点。

**示例：**
 给定二叉树 [3,9,20,null,null,15,7]，
 ```Java
   3
  / \
 9   20
    /  \
   15   7
 ```
 返回它的最大深度 3 。

 **Solution:**
 递归解法
 ```Java
 /**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public int maxDepth(TreeNode root) {
        if(root == null) return 0;
        int heightL = maxDepth(root.left);
        int heightR = maxDepth(root.right);
        return Math.max(heightL,heightR)+1;
    }
}
 ```

 非递归解法
```Java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public int maxDepth(TreeNode root) {
        if(root == null){
            return 0;
        }
        Queue<TreeNode> queue = new LinkedList<TreeNode>();
        int maxDepth = 0;
        queue.offer(root);
        while(!queue.isEmpty()){
            maxDepth++;
            int size = queue.size();
            for(int i = 0; i<size; i++){
                TreeNode node = queue.poll();
                if(node.left != null){
                    queue.offer(node.left);
                }
                if(node.right != null){
                    queue.offer(node.right);
                }
            }
        }
        return maxDepth;
    }
}
```

### 146 LRU缓存机制
**描述**
运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制。它应该支持以下操作： 获取数据 get 和 写入数据 put 。
获取数据 get(key) - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 put(key, value) - 如果密钥不存在，则写入其数据值。当缓存容量达到上限时，它应该在写入新数据之前删除最近最少使用的数据值，从而为新的数据值留出空间。

**示例**
```java
LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );

cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // 返回  1
cache.put(3, 3);    // 该操作会使得密钥 2 作废
cache.get(2);       // 返回 -1 (未找到)
cache.put(4, 4);    // 该操作会使得密钥 1 作废
cache.get(1);       // 返回 -1 (未找到)
cache.get(3);       // 返回  3
cache.get(4);       // 返回  4
```

**思路**
这个问题可以用哈希表，辅以双向链表记录键值对的信息。所以可以在 O(1)O(1)O(1) 时间内完成 put 和 get 操作，同时也支持 O(1)O(1)O(1) 删除第一个添加的节点。
![avatar](https://pic.leetcode-cn.com/815038bb44b7f15f1f32f31d40e75c250cec3c5c42b95175ec012c00a0243833-146-1.png)
使用双向链表的一个好处是不需要额外信息删除一个节点，同时可以在常数时间内从头部或尾部插入删除节点。

一个需要注意的是，在双向链表实现中，这里使用一个伪头部和伪尾部标记界限，这样在更新的时候就不需要检查是否是 null 节点。
![avatar](https://pic.leetcode-cn.com/48292c190e50537087ea8c60ed44062675d55a73d1a59035d26e277a36b7b8e2-146-2.png)

**实现**
```Java
import java.util.Hashtable;
public class LRUCache {

  class DLinkedNode {
    int key;
    int value;
    DLinkedNode prev;
    DLinkedNode next;
  }

  private void addNode(DLinkedNode node) {
    /**
     * Always add the new node right after head.
     */
    node.prev = head;
    node.next = head.next;

    head.next.prev = node;
    head.next = node;
  }

  private void removeNode(DLinkedNode node){
    /**
     * Remove an existing node from the linked list.
     */
    DLinkedNode prev = node.prev;
    DLinkedNode next = node.next;

    prev.next = next;
    next.prev = prev;
  }

  private void moveToHead(DLinkedNode node){
    /**
     * Move certain node in between to the head.
     */
    removeNode(node);
    addNode(node);
  }

  private DLinkedNode popTail() {
    /**
     * Pop the current tail.
     */
    DLinkedNode res = tail.prev;
    removeNode(res);
    return res;
  }

  private Hashtable<Integer, DLinkedNode> cache =
          new Hashtable<Integer, DLinkedNode>();
  private int size;
  private int capacity;
  private DLinkedNode head, tail;

  public LRUCache(int capacity) {
    this.size = 0;
    this.capacity = capacity;

    head = new DLinkedNode();
    // head.prev = null;

    tail = new DLinkedNode();
    // tail.next = null;

    head.next = tail;
    tail.prev = head;
  }

  public int get(int key) {
    DLinkedNode node = cache.get(key);
    if (node == null) return -1;

    // move the accessed node to the head;
    moveToHead(node);

    return node.value;
  }

  public void put(int key, int value) {
    DLinkedNode node = cache.get(key);

    if(node == null) {
      DLinkedNode newNode = new DLinkedNode();
      newNode.key = key;
      newNode.value = value;

      cache.put(key, newNode);
      addNode(newNode);

      ++size;

      if(size > capacity) {
        // pop the tail
        DLinkedNode tail = popTail();
        cache.remove(tail.key);
        --size;
      }
    } else {
      // update the value.
      node.value = value;
      moveToHead(node);
    }
  }
}
```

### 283 Move Zeroes
Given an array nums, write a function to move all 0s to the end of it while maintaining the relative order of the non-zero elements.

**Example:**

Input: [0,1,0,3,12]
Output: [1,3,12,0,0]

**Note:**

- You must do this in-place without making a copy of the array.
- Minimize the total number of operations.

**思路**
这道题可以用两个指针去做，第一个指针正常依次访问数组中的数据，第二个慢指针在第一个指针遇到非空的值后，快指针和慢指针所指的值交换，慢指针+1，慢指针所依次所指向的就是去掉零后的排序。

**Solution:**
```Java
class Solution {
    public void moveZeroes(int[] nums) {
        int l = nums.length;
        int j = 0;
        for(int i = 0;i<l;i++){
            if(nums[i] != 0){
                int temp = nums[j];
                nums[j] = nums[i];
                nums[i] = temp;
                j++;
            }
        }
    }
}
```

### 347. 前 K 个高频元素
**描述**
给定一个非空的整数数组，返回其中出现频率前 k 高的元素。

**示例**
示例 1:
```java
输入: nums = [1,1,1,2,2,3], k = 2
输出: [1,2]
```
示例 2:
```Java
输入: nums = [1], k = 1
输出: [1]
```
**说明**
- 你可以假设给定的 k 总是合理的，且 1 ≤ k ≤ 数组中不相同的元素的个数。
- 你的算法的时间复杂度必须优于 O(n log n) , n 是数组的大小。

**Solutin**
```Java
class Solution {
    public List<Integer> topKFrequent(int[] nums, int k) {
        Map<Integer,Integer> freqMap = new HashMap<>();
        for(int num : nums){
            freqMap.put(num,freqMap.getOrDefault(num,0)+1);
        }
        List<Integer>[] bucket = new List[nums.length + 1];
        for(Integer key: freqMap.keySet()){
            int freq = freqMap.get(key);
            if(bucket[freq] == null){
                bucket[freq] = new LinkedList<Integer>();
            }
            bucket[freq].add(key);
        }

        List<Integer> result = new LinkedList<Integer>();
        for(int i = nums.length;i>=0 && result.size()<k;i--){
            if(bucket[i] !=null ){
                result.addAll(bucket[i]);
            }
        }
        return result;
    }
}
```

### 543. 二叉树的直径
**描述：**
给定一棵二叉树，你需要计算它的直径长度。一棵二叉树的直径长度是任意两个结点路径长度中的最大值。这条路径可能穿过根结点。
**示例：**
给定二叉树
```Java
       1
      / \
     2   3
    / \
   4   5
```
返回 3, 它的长度是路径 [4,2,1,3] 或者 [5,2,1,3]。
注意：两结点之间的路径长度是以它们之间边的数目表示。

**Solution**
```Java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    int ans = 1;
    public int diameterOfBinaryTree(TreeNode root) {
        getDept(root);
        return ans-1;
    }

    public int getDept(TreeNode node){
        if(node == null) return 0;
        int heightL = getDept(node.left);
        int heightR = getDept(node.right);
        ans = Math.max(ans,heightL+heightR+1);
        return Math.max(heightL,heightR)+1;
    }
}
```




反转链表
```Java
class ListNode{
  int val;
  ListNode next;
  public ListNode(){
    ...
  }
}


class Solution {
    public ListNode reverse(ListNode head){

    }
}
```

### 剑指offer
孩子们的游戏(圆圈中最后剩下的数)
```Java
import java.util.LinkedList;
public class Solution {
    public int LastRemaining_Solution(int n, int m) {
        if(m == 0 || n == 0){
            return -1;
        }
        LinkedList<Integer> list = new LinkedList();
        for(int i = 0; i<n; i++){
            list.add(i);
        }
        int index = -1;
        while(list.size()>1){
            index = (index+m)%list.size();
            list.remove(index);
            index--;
        }
        return list.get(0);
    }
}
```
