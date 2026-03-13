# 1、应用场景

有一个字符串 str1= ""硅硅谷 尚硅谷你尚硅 尚硅谷你尚硅谷你尚硅你好""，和一个子串 str2="尚硅谷你尚硅你"。

现在要判断 str1 是否含有 str2, 如果存在，就返回第一次出现的位置, 如果没有，则返回 -1。

# 2、算法介绍

Knuth-Morris-Pratt 字符串查找算法，简称为 “KMP算法”，常用于在一个文本串S内查找一个模式串P 的出现位置。

KMP方法算法就利用之前判断过信息，通过一个next数组，保存模式串中前后最长公共子序列的长度，每次回溯时，通过next数组找到，前面匹配过的位置，省去了大量的计算时间。

# 3、暴力匹配

```java
public int violence(String str1, String str2) {
    char[] chars1 = str1.toCharArray();
    char[] chars2 = str2.toCharArray();
    int p1 = 0;
    int p2 = 0;

    while (p1 < chars1.length && p2 < chars2.length) {
        if (chars1[p1] == chars2[p2]) {
            p1++;
            p2++;
        } else {
            p2 = 0;
            p1 = p1 - p2 + 1;
        }
        if (p2 == chars2.length) {
            return p1 - p2;
        }
    }
    return -1;
}
```

# 4、Kmp算法

获取next数组

ABC 前缀 A AB       后缀 C BC 没有重复 数组应该为 0 0 0

ABCA 前缀 A AB ABC  后缀 A CA BCA 重复 A 数组应该为 0 0 0 1

ABCABB 数组应该为 0 0 0 1 2 0

```java
public int[] getKmpArr(String str) {
    char[] chars = str.toCharArray();
    int[] arr = new int[str.length()];

    for (int i = 1, j = 0; i < chars.length; i++) {
        //AAACBAAACAAACBAAAAA 理解一下
        //AAACBAAAAA 理解一下
        while (j > 0 && chars[i] != chars[j]) {
            j = arr[j - 1];
        }

        if (chars[i] == chars[j]) {
            j++;
            arr[i] = j;
        }
    }
    return arr;
}
```

kmp

```java
public int kmp(String str1, String str2) {
    int[] kmpArr = getKmpArr(str2);
    char[] chars1 = str1.toCharArray();
    char[] chars2 = str2.toCharArray();

    //AAACBAAACAAACBAAAAA 理解一下
    //AAACBAAAAA 理解一下
    for (int i = 0, j = 0; i < chars1.length; i++) {
        while (j > 0 && chars1[i] != chars2[j]) {
            j = kmpArr[j - 1];
        }
        if (chars1[i] == chars2[j]) {
            j++;
        }
        if (j == chars2.length) {
            return i - j + 1;
        }
    }
    return -1;
}
```

