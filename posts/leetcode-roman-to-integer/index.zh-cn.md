---
title: "Leetcode - 罗马数转整数"
date: 2024-02-26T00:00:00+08:00
draft: false
description: "罗马数转整数问题解析"
featuredImage: "cover.webp"

tags: ["Leetcode", "Java", "Algorithm"]
categories: ["Leetcode"]
series: ["Leetcode 问题学习与解析"]
lightgallery: true

math:
  enable: true
---

<!--more-->

## TL;DR

```java
public int romanToInt(String s) {
  char[] charArray = s.toCharArray();
  int result = 0;
  for (int i = 0; i < charArray.length; i++) {
    char c = charArray[i];
    int number = getValue(c);
    if (i + 1 < charArray.length && number < getValue(charArray[i + 1])) {
      result -= number;
    } else {
      result += number;
    }
  }
  return reslut;
}

private int getValue(char c) {
  switch (c) {
    case 'I' -> {
      return 1;
    }
    case 'V' -> {
      return 5;
    }
    case 'X' -> {
      return 10;
    }
    case 'L' -> {
      return 50;
    }
    case 'C' -> {
      return 100;
    }
    case 'D' -> {
      return 500;
    }
    case 'M' -> {
      return 1000;
    }
    default -> {
      return 0;
    }
  }
}

```
## 思路

老样子，开始分析思路。

题中罗马数字有 `I` `V` `X` `L` `C` `D` `M` 7 种，其中 `IV` `IX` `XL` `XC` `CD` `CM` 这六种特殊，其值为后面的数减去前面的数。
以 `IV` 为例，它的值是 $5 - 1 = 4$ ，可以转化为 $-1 + 5 = 4$ 。而且，需要注意的是罗马数字中的数字是相加求和的，而且所有大数都会在左侧，
也就是说会有 `LIX = 54` 而不会有 `LIIX` 这种情况。这就代表着，你按顺序读的过程中，当大数字出现在小数字前方时一定是结合数字，而不是其他情况。

以这种思路我们就可以将字符串转换为字符列表，然后逐一判断其值，并判断是否为 `I` `X` `C` 这几个特殊字符，如果是就判断它的后一个字符是否是特殊字符，
是的话我们就将这个值转换为负数。这样我们就可以的到最终值。

举个例子：

  `MCMXCIV` 拆解一下： $M + CM + XC + IV =  1000 + (-100 + 1000) + (-10 + 100) + (-1 + 5) = 1994$

## 暴力解法

```java
  public int romanToInt(String s) {
    HashMap<Character, Integer> romanMap = new HashMap<>();
    romanMap.put('I', 1);
    romanMap.put('V', 5);
    romanMap.put('X', 10);
    romanMap.put('L', 50);
    romanMap.put('C', 100);
    romanMap.put('D', 500);
    romanMap.put('M', 1000);
  
    char[] charArray = s.toCharArray();
    int result = 0;
    for (int i = 0; i < charArray.length; i++) {
      char c = charArray[i];
      int number = romanMap.get(c);
      if (i + 1 < charArray.length) {
        char nextC = charArray[i + 1];
        switch (c) {
          case 'I' -> {
            if (nextC == 'V' || nextC == 'X') {
              number = -romanMap.get(c);
            }
          }
          case 'X' -> {
            if (nextC == 'L' || nextC == 'C') {
              number = -romanMap.get(c);
            }
          }
          case 'C' -> {
            if (nextC == 'D' || nextC == 'M') {
              number = -romanMap.get(c);
            }
          }
        }
      }
      result += number;
    }
    return reslut;
  }
```

这里用了 `switch` 判断三种情况。这也是比较容易理解的方式。

时间复杂度： $O(n)$ (需要索引所有字符)

空间复杂度： $O(n)$ (需要存储所有字符)

## 也许更好

对于这道题来说，有许多更好的解法，因为题目中给定输入值必然合法，所以可以有以下解法：

罗马数字的左侧值是必然大于右侧值的，不会存在 `IVXLCDM` 这种值，如果右边出现大值，那一定是出现了减法。

| X | X | V | I | I | 27 |
|---|---|---|---|---|---|
| 10 | 10 | 5 | 1 | 1 | 27 |

| X | X | I | V | 24 |
|---|---|---|---|---|
| 10 | 10 | -1 | 5 | 24 |

也就是说，只需要判断右侧是否大于左侧值，如果大了就直接对左侧值做减法

```java
  public int romanToInt(String s) {
    char[] charArray = s.toCharArray();
    int result = 0;
    for (int i = 0; i < charArray.length; i++) {
      char c = charArray[i];
      int number = getValue(c);
      if (i + 1 < charArray.length && number < getValue(charArray[i + 1])) {
        result -= number;
      } else {
        result += number;
      }
    }
    return reslut;
  }
  
  private int getValue(char c) {
    switch (c) {
      case 'I' -> {
        return 1;
      }
      case 'V' -> {
        return 5;
      }
      case 'X' -> {
        return 10;
      }
      case 'L' -> {
        return 50;
      }
      case 'C' -> {
        return 100;
      }
      case 'D' -> {
        return 500;
      }
      case 'M' -> {
        return 1000;
      }
      default -> {
        return 0;
      }
    }
  }
  
```

其中 `switch` 直接存储数据的方式，我是从一个题解中学到的。我之前倒是没想过可以这样。

这样的话空间可以少占一些，与 `Map` 相比相差不了多少。（leetcode 上面显示节省 1m 空间，可以约等于没有了。仅限 Java）

时间复杂度： $O(n)$ (需要索引所有字符)

空间复杂度： $O(n)$ (需要存储所有字符)
