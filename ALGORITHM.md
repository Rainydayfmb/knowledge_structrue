# 算法

## 数据结构

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
         for (int j = 0; j < n - i - 1; j++) {
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

countdownlatch
```Java
public class CountDownLatchTest {

    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(2);
        System.out.println("主线程开始执行…… ……");
        //第一个子线程执行
        ExecutorService es1 = Executors.newSingleThreadExecutor();
        es1.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                    System.out.println("子线程："+Thread.currentThread().getName()+"执行");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                latch.countDown();
            }
        });
        es1.shutdown();

        //第二个子线程执行
        ExecutorService es2 = Executors.newSingleThreadExecutor();
        es2.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("子线程："+Thread.currentThread().getName()+"执行");
                latch.countDown();
            }
        });
        es2.shutdown();
        System.out.println("等待两个线程执行完毕…… ……");
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("两个子线程都执行完毕，继续执行主线程");
    }
}

```


## leetcode

283 Move Zeroes
Given an array nums, write a function to move all 0's to the end of it while maintaining the relative order of the non-zero elements.

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

**2. 两数相加**
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

**70.爬楼梯**
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

**1.两数相加**
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

linux 统计相当uri的多少，并排序。如何解决hash函数的相关问题，hash碰撞的相关问题。



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
