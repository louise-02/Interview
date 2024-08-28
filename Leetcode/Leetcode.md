# 1. [两数之和](https://leetcode.cn/problems/two-sum/description/)『数组 哈希表』

## **1、暴力解**

时间复杂度：O(N^2)

空间复杂度：O(1)

```java
public int[] twoSum(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++) {
        for (int j = i + 1; j < nums.length; j++) {
            if (nums[i] + nums[j] == target) {
                return new int[]{i, j};
            }
        }
    }
    return null;
}
```

## **2、哈希表**

时间复杂度：O(N)

空间复杂度：O(N)

```java
public int[] twoSum1(int[] nums, int target) {
    HashMap<Integer, Integer> diffIndexMap = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        Integer index = diffIndexMap.get(nums[i]);
        if (index != null) {
            return new int[]{index, i};
        }
        diffIndexMap.put(target - nums[i], i);
    }
    return null;
}
```

# 2. [两数相加](https://leetcode.cn/problems/add-two-numbers/description/)『链表 迭代 递归』

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
```

## **1、迭代**

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    ListNode first = new ListNode();
    ListNode temp = first;
    int up = 0;
    while (l1 != null || l2 != null) {
        int n1 = 0, n2 = 0;
        if (l1 != null) {
            n1 = l1.val;
            l1 = l1.next;
        }
        if (l2 != null) {
            n2 = l2.val;
            l2 = l2.next;
        }
        temp.val = (n1 + n2 + up) % 10;
        up = (n1 + n2 + up) / 10;
        if (l1 != null || l2 != null) {
            temp.next = new ListNode();
            temp = temp.next;
        }
    }
    if (up == 1) {
        temp.next = new ListNode(1);
    }
    return first;
}
```

## **2、递归**

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
	return addTwoNumbers1(l1, l2, 0);
}


public ListNode addTwoNumbers1(ListNode l1, ListNode l2, int up) {
    if (l1 == null && l2 == null) {
        return up == 0 ? null : new ListNode(up);
    }
    int sum = (l1 == null ? 0 : l1.val) + (l2 == null ? 0 : l2.val) + up;
    int val = sum % 10;
    up = sum / 10;
    ListNode listNode = new ListNode(val);
    listNode.next = addTwoNumbers1(l1 == null ? null : l1.next, l2 == null ? null : l2.next, up);
    return listNode;
}
```

# 3. [无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/)『字符串 滑动窗口』

## **1、滑动窗口**

定义左右指针，右指针在添加的时候如何发现有重复内容就去掉左指针所在位置的字符。

```java
public int lengthOfLongestSubstring(String s) {
    if (s == null) {
        return 0;
    }

    HashSet<Character> set = new HashSet<>();

    char[] chars = s.toCharArray();

    int left = 0, maxLength = 0;

    for (int i = 0; i < chars.length; i++) {
        while (!set.add(chars[i])) {
            set.remove(chars[left++]);
        }
        maxLength = Math.max(maxLength,i - left + 1);
    }

    return maxLength;
}
```

# 4. [寻找两个正序数组的中位数](https://leetcode.cn/problems/median-of-two-sorted-arrays/)『数组 二分』

## **1、合并数组**

时间复杂度：O(m+n)

## **2、模拟合并数组**

时间复杂度：O(m+n)

合并后数组长度为 len，如果合并后为奇数，那就取第 len/2 即可，如果是偶数，那就取第 len/2 和前一位数。

```java
public double findMedianSortedArrays(int[] nums1, int[] nums2) {
    //用来记录两个数组左索引
    int n1l = 0, n2l = 0;

    int mid = (nums1.length + nums2.length) / 2;

    //定义最终取到的中位数，奇数取 r，偶数取 (l+r)/2
    int l = 0, r = 0;

    for (int i = 0; i <= mid; i++) {
        l = r;
        if (n1l < nums1.length && n2l < nums2.length) {
            if (nums1[n1l] < nums2[n2l]) {
                r = nums1[n1l++];
            } else {
                r = nums2[n2l++];
            }
        } else if (n1l >= nums1.length) {
            r = nums2[n2l++];
        } else {
            r = nums1[n1l++];
        }
    }

    if ((nums1.length + nums2.length) % 2 == 0) {
        return (l + r) / 2.0;
    } else {
        return r;
    }
}
```

## **3、二分法**（待写）

# 5. [最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/solutions/255195/zui-chang-hui-wen-zi-chuan-by-leetcode-solution/)『字符串 动态规划』

1、中心扩展算法

2、动态规划

3、