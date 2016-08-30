---
title: LeetCode#136.Single Number
date: 2016-08-29 20:23:55
tags: [算法, LeetCode, Hash Table, Bit Manipulation]
---
# 题目描述
Given an array of integers, every element appears twice except for one. Find that single one.
# 解法一
使用位异或操作求解。
```
public class Solution {
    public int singleNumber(int[] nums) {
        int result = 0;
        for (int i = 0; i < nums.length; i++) {
            result ^= nums[i];
        }
        return result;
    }
}
```
<!--more-->
# 解法二
使用HashSet求解，遍历数组，如果set中存在这个数，则将其从set中移出，若不存在，则在set中保存这个数。
```
public class Solution {
    public int singleNumber(int[] nums) {
        Set set = new HashSet();
        for (int i = 0; i < nums.length; i++) {
            if (set.contains(nums[i])) {
                set.remove(nums[i]);
            } else {
                set.add(nums[i]);
            }
        }
        Iterator it = set.iterator();
        return (Integer)it.next();
    }
}
```