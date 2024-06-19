# 1. [两数之和](https://leetcode.cn/problems/two-sum/description/)『数组 哈希表』

**1、暴力解**

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

**2、哈希表**

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

**迭代**

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

**递归**

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

# 3.[无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/description/)『字符串 滑动窗口』

1、**滑动窗口**

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

# 4.[寻找两个正序数组的中位数](https://leetcode.cn/problems/median-of-two-sorted-arrays/)『数组 二分』