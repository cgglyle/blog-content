---
title: "Leetcode - 回文数问题解析"
date: 2024-02-23T00:00:00+08:00
draft: false
description: "回文数问题解析"
featuredImage: "cover.webp"

tags: ["Leetcode", "Java", "Algorithm"]
categories: ["Leetcode"]
series: ["Leetcode 问题学习与解析"]
lightgallery: true

math:
  enable: true
---


开始先吐槽一下：Leetcode 的代码编辑器真的是太难受了，没有智能提示，也不能 Debug，
需要每个月交 79！！这也太贵了！
<!--more-->

{{< admonition warning"注意" >}}
因为最近想要去欧美发展，而那边更注重算法，所以想开始学习算法并练习。所以，此系列文章可能都会比较啰嗦，
也可能会包含部分错误， 因为都在学习中，为了更好的生活，共勉！
{{< /admonition >}}

## TL;DR

```java
public boolean isPalindrome(int x) {
    if (x < 0 || (x % 10 == 0 && x != 0)) {
        return false;
    }

    int endNumber = 0;
    while (endNumber < x) {
        endNumber = x % 10 + endNumber * 10;
        x = x / 10;
    }
    if (endNumber == x || endNumber / 10 == x) {
        return true;
    }
    return false;
}
```

{{< admonition question "原始问题" >}}
给你一个整数 x ，如果 x 是一个回文整数，返回 true ；否则，返回 false 。

回文数是指正序（从左向右）和倒序（从右向左）读都是一样的整数。

    例如，121 是回文，而 123 不是。

示例 1：

输入：x = 121
输出：true

示例 2：

输入：x = -121
输出：false
解释：从左向右读, 为 -121 。 从右向左读, 为 121- 。因此它不是一个回文数。

示例 3：

输入：x = 10
输出：false
解释：从右向左读, 为 01 。因此它不是一个回文数。

提示：

    -231 <= x <= 231 - 1

进阶：你能不将整数转为字符串来解决这个问题吗？

{{< /admonition >}}

## 分析题目

如果 x 是回文数那就返回 `true` 否则返回 `false`。按照题目中给的示例，可以明显的看出如果 x 是负数必然不是回文数，所以我们可以快速的排除掉一半的情况。

如果使用暴力解法，我们需要考虑的情况就是将数字转成 `char` 数组，然后分析对比每个相对索引是否相同即可。

## 暴力解法

```java
   public boolean isPalindrome(int x) {
    if (x < 0) {
        return false;
    }
    String xString = Integer.toString(x);
    char[] xChars = xString.toCharArray();
    for (int i = 0; i < xChars.length; i++) {
        if (i == xChars.length / 2) {
            break;
        }
        if (xChars[i] != xChars[xChars.length - i - 1]) {
            return false;
        }
    }
    return true;
}
```

我们开始就判断是否为负数，排除一半的情况。

之后将数字转成 `char` 数组，方便循环处理。

然后判断第一个字符和最后一个字符是否相同，并依次增加。

这个解法比较简单，时间复杂度是 $ O(\log_{10}(x)) $ ，因为要查询所有字符，空间复杂度也是 $O(n)$，因为要存储所有字符。

{{< admonition tip "为什么是 $ O(\log_{10}(x)) $" >}}
有些人可能疑惑：既然数字的每一个位数都要循环一遍，那不应该是 $O(n)$ 么？你说的对，但是现在 `n` 实际上是输入 `x`，也就是 `121`
之类的。它的位数是 3，实际上循环两次解决问题。而 $ O(\log_{10}(x)) $ 的值，也就是 2。
{{< /admonition >}}

## 进阶解法（要求不转换字符）

如果只使用数字的话，只要将数字反转就好了，例如 `12345678` 反转成 `87654321` 就好了，最先考虑到的就是除余。

```java
  public boolean isPalindrome(int x) {
    if (x < 0 || x % 10 == 0) {
        return false;
    }

    int endNumber = 0;
    int finalX = x;
    while (x != 0) {
        endNumber = x % 10 + endNumber * 10;
        x = x / 10;
    }
    if (endNumber == finalX) {
        return true;
    }
    return false;
}
```

通过循环除余就可以拿到最终的结果，只需要判断是否与原数字是否相同就好。

至于数字大小超过数字范围，如果反转过程中超过范围那一定不是回文数，回文数正反相同，所以绝对不会超范围。

这种方式时间复杂度为 $ O(\log_{10}(x)) $ 因为要循环。空间复杂度是 $O(1)$。


## 最终解法

```java
   public boolean isPalindrome(int x) {
    if (x < 0 || x % 10 == 0) {
        return false;
    }

    int endNumber = 0;
    while (endNumber < x) {
        endNumber = x % 10 + endNumber * 10;
        x = x / 10;
    }
    if (endNumber == x || endNumber / 10 == x) {
        return true;
    }
    return false;
}
```

刚刚的方法是循环了所有位，而判断回文数只需要对折一半就好了。也就是循环一半就好。

需要考虑的是，我们怎么知道是不是到达一半了？我们只需要判断反转的数字是否大于需要处理的数字就好。可以看个例子：

| 需要处理的值 | 反转值 | 等待处理的值 |
|---|---|---|
| 121 | 1 | 12 |
| 12 | 12 | 1 |

在做第三次处理的时候发现反转值是 `12` 剩余需要反转的值为 `1` 所以就知道已经到一半了

这样我们就可以得到反转值的一半了。而且可以发现如果数字的位数是奇数，中间会有一个多余值 `2` 在判断的时候我们只要移除多余值就好。

这样，我们的时间复杂度是 $ O(\log_{10}(x)) $ 空间复杂度是 $O(1)$，整体上是优于上个算法的，因为整个算法只需要处理一半的值。

---

### 纠错

需要注意的是 `0` 也是回文数，需要注意。

```java {linenos=table,hl_lines=[2]}
    public boolean isPalindrome(int x) {
        if (x < 0 || (x % 10 == 0 && x != 0)) {
            return false;
        }

        int endNumber = 0;
        while (endNumber < x) {
            endNumber = x % 10 + endNumber * 10;
            x = x / 10;
        }
        if (endNumber == x || endNumber / 10 == x) {
            return true;
        }
        return false;
    }
```
