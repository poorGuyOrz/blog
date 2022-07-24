---
title: "Summery"
date: 2022-05-18T11:42:41+08:00
draft: true
---

## 动态规划

重叠子问题，最有子结构，记录中间结果，结果记录之后就不会修改，无后效性，由于子问题重叠，所以可以使用中间结果剪枝。

可以使用自底向上和自顶向下的方法解决问题，


## 滑动窗口

不会回溯，一遍遍历过程中，求解某个连续范围内满足要求的条件。例如字符串的不重复子数组最大长度，子数组最大值等。
记录窗口滑动条件，窗口内计算，模板如下。按需变动
* 数组是字符串是使用vector直接保存状态，使用数值是按范围确定，必要时使用map
```c++
class Solution
{
public:
  int lengthOfLongestSubstring(string s)
  {
    vector<int> status(128, 0);
    int left = 0;
    int ret = 0;
    for (int i = 0; i < s.length(); i++)
    {
      while (status[s[i]] != 0)
      {
        status[s[left]]--;
        left++;
      }
      status[s[i]]++;

      ret = std::max(ret, i - left + 1);
    }
    return ret;
  }
};
```