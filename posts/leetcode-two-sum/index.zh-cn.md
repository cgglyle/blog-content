---
title: "Leetcode - 两数之和问题解析"
date: 2024-02-22T00:00:00+08:00
lastmod: 2024-02-22T00:00:00+08:00
draft: false
description: "两数之和问题解析"
featuredImage: "cover.webp"

tags: ["Leetcode", "Java", "Algorithm"]
categories: ["Leetcode"]
series: ["Leetcode 问题学习与解析"]
lightgallery: true

math:
  enable: true
---

<!--more-->

{{< admonition warning"注意" >}}
因为最近想要去欧美发展，而那边更注重算法，所以想开始学习算法并练习。所以，此系列文章可能都会比较啰嗦，
也可能会包含部分错误， 因为都在学习中，为了更好的生活，共勉！
{{< /admonition >}}

## TL;DR
```java
// One-pass Hash Table
public int[] twoSum(int[] nums, int target) {
  Map<Integer, Integer> hashTable = new HashMap<Integer, Integer>();
  for (int i = 0; i < nums.length; i++) {
    if (hashTable.containsKey(target - nums[i])) {
      return new int[] {hashTable.get(target - nums[i]), i};
    }
    hashTable.put(nums[i], i);
  }
  throw new RuntimeException();
}
```

{{< admonition question "原始问题" >}}
给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。

 - 示例 1：
    输入：nums = [2,7,11,15], target = 9  
    输出：[0,1]  
    解释：因为 nums[0] + nums[1] == 9 ，返回 [0, 1] 。

 - 示例 2：
    输入：nums = [3,2,4], target = 6  
    输出：[1,2]  

 - 示例 3：
    输入：nums = [3,3], target = 6  
    输出：[0,1]  

提示：

    2 <= nums.length <= 104
    -109 <= nums[i] <= 109
    -109 <= target <= 109
    只会存在一个有效答案

进阶：你可以想出一个时间复杂度小于 O(n2) 的算法吗？

{{< /admonition >}}

## 思路

### 暴力解法

两个嵌套循环解决，他既然说是要找两个数字之和 `Num1 + Num2 = Target` 第一个想法就是循环，这非常简单。也就是暴力解法。时间复杂度是 $O(n^2)$，
因为需要使用嵌套循环，最坏的情况每个元素都要遍历 `n*n` 遍。 

```java
  // Brute Force
  public int[] twoSum(int[] nums, int target) {
    for (int i = 0; i < nums.length; i++){
      for (int j = i + 1; j < nums.length; j++) {
        if (nums[i] + nums[j] == target){
          return new int[] {i, j};
        }
      }
    }
    throw new RuntimeException();
  }
```

### 双循环哈希表

有没有实践复杂度更低的方式解决呢？有。如果我们可以记住每一个我们已经遍历过的数字，我们就可以快速的找到他对应的索引。例如：

```
Input: nums = [3,2,4], target = 6
Output: [1,2]  
```

我们可以考虑使用 `target - num1 == num2` 来代替 `num1 + num2 == target`，这样也更符合直觉，
也就是说我们寻找数组中是否存在一个数让目标数减去其中一个数等于数组中另外一个数。说人话就是 `6-3=3` 数组中剩下两个数中有没有 `3`，
如果有那就找到了，没有就找下一个。（题目要求不能使用相同索引的数，且必定含有答案）

我们可以设计一个可以存储数字的容器，将所有数字放入，每当我们减去一个数字的时候我们就去容器中找有没有这个数字，如果有就在判断一下是不是相同索引，
如果不是那就找到答案了，没有就循环找下一个。这样我们就可以设计出一个抛弃嵌套循环的算法。

这样我们就想到了使用 `Map` 也就是 `Hash Table` 来处理。哈希表的时间复杂度是 $O(1)$，非常适合这种场景。而且我们需要查找数字，
所以哈希表的键值对应该是 `<数字，索引>`。

```java
  // Two-pass Hash Table
  public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> hashTable = new HashMap<Integer, Integer>();
    for (int i = 0; i < nums.length; i++) {
      hashTable.put(nums[i], i);
    }
    for (int i = 0; i < nums.length; i++) {
      int complement = target - nums[i];
      if (hashTable.containsKey(complement) && hashTable.get(complement) != i) {
        return new int[]{i, hashTable.get(target - nums[i])};
      }
    }
    throw new RuntimeException();
  }
```

还需要考虑的事情是：输出的值要按顺序，也就是小序号在前，大序号在后，我们大可以在写一个判断，来处理这件事情。但实际上并不需要，
因为在循环的过程是递增的，如果小序号只能在大序号中找最终值，而不能在小序号中找，因为已经找过了。
（正如交换律一样，不可能存在递增循环中找到比当前序号小的正确答案，有点不好描述，你可以自己想想）

当前算法因为抛弃了嵌套循环，再存入数据的过程中需要循环放入所有数据，而查找过程中 `Hash Table` 复杂度是 $O(1)$，所以时间复杂度 $O(n)$，
空间复杂度因为添加了一个容器存放所有数据所以是 $O(n)$。

### 单循环哈希表（最佳）
貌似这就是最好的方式了，但是如果需要查询的数字有一亿个呢？即便是 `O(n)` 的空间复杂度也是撑不住的。
有没有其他的方式稍微减少一下空间复杂度甚至是实践复杂度呢？

正如刚刚说的：

> 正如交换律一样，不可能存在递增循环中找到比当前序号小的正确答案  

上面那句话的前提是我们是提前存储了所有数据，并且递增查询。当我们不提前存储，直接查询，如果容器中不存在对应的差值，
我们就把差值和当前索引存入 `Map` 中，当查到第二个值的时候如果差值存在，那就是答案，这巧妙的利用了交换律。

```java
  // One-pass Hash Table
  public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> hashTable = new HashMap<Integer, Integer>();
    for (int i = 0; i < nums.length; i++) {
      if (hashTable.containsKey(target - nums[i])) {
        return new int[] {hashTable.get(target - nums[i]), i};
      }
      hashTable.put(nums[i], i);
    }
    throw new RuntimeException();
  }
```

这相当于我们在查询的过程中提前将答案存入容器，如果真的存在这个数字那这就是答案。需要注意的是，既然我们容器中存放的是答案和存放时的索引，
也就是`target - num1 == num2` 中的 `num1` 的索引，所以我们要返回的答案是 
`<num1 的索引（也就是容器中的值，因为它是小序号）, num2 的索引（也就是当前的索引 i ，它是大序号）>`

这样我们时间复杂度是 $O(n)$ 因为最坏的情况是遍历所有数字，但是这要优于上个算法，上个算法需要两个 $O(n)$ 也就是 $2O(n)$，
虽然常数 2 会被舍弃，但是要稍微慢一些，而且上个算法必定会执行至少 n 次，而这个算法最好的情况下只需要执行 2 次（第一次执行必定为空）。

而空间复杂度在最坏的情况下需要存储所有数字，也就是  $O(n)$ 但是也要优于上面算法，因为上面算法的  $O(n)$ 也是必定的。
这个最好的情况只需要存储 1 个数字。