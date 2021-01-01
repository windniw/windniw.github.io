---
title: 最大矩阵和问题
date: 2020-11-20 00:18:35
math: true
categories:
- algorithm
tags:
- DP
---
子矩阵的最大和。

<!--more-->

> [363. 矩形区域不超过 K 的最大数值和](https://leetcode-cn.com/problems/max-sum-of-rectangle-no-larger-than-k/)
> 
> 给矩阵 matrix，求其子矩阵和最大且小于 k 的值

简单说，二维数组有很大一类思路是压缩成一维数组求解，本题就是这样。在聊这题怎么写之前，先来看两道简化版的题目。

## 最大子序和

> [53. 最大子序和](https://leetcode-cn.com/problems/maximum-subarray/)
>
> 给数组 nums，求其连续子序列中，最大的和

思路：DP。令 dp[i] 记录以 nums[i] 为最后子序列最后一个元素的最大和。有如下状态转移公式，若当前最大连续子序列和小于 0，则直接放弃所有前缀数字，从 0 开始。

$$
dp[i] =
\begin{cases}
\max(nums[0], 0), & \text{n = 0}  \\
\max(dp[i-1] + nums[i], 0), & \text{n $\neq$ 0}
\end{cases}
$$

**时间复杂度 $O(n)$, 压缩状态后空间 $O(1)$。这个解法也称为 kadane 算法。**

若要求子序列长度不能为 0 ，简单修改状态方程即可

$$
dp[i] =
\begin{cases}
nums[0], & \text{n = 0}  \\
\max(dp[i-1] + nums[i], nums[i]), & \text{n $\neq$ 0}
\end{cases}
$$

```python
def max_subarray(A):
    max_ending_here = max_so_far = A[0]
    for i in range(1,lengh(A)):
        max_ending_here = max(max_ending_here, max_ending_here + A[i])
        max_so_far = max(max_so_far, max_ending_here)
    return max_so_far
```

## 最大子序和 + 

接下来在上题补充一个条件

> 给数组 nums, 求其连续子序列中，**最大且不超过 k** 的和

虽然只增加了一个条件，但不能再沿用之前的 DP 做法。这里依然是无后效性的，但只记录以 nums[i] 为最末元素的最大和或不超过k的最大和，并不足以推出 nums[i:1] 的状态。所以，增加这个约束后，需要换一种思路。

思路：记最优解为 $res = sum(nums[i:j+1])$，有 

$$res = sum(nums[i:j+1]) $$

$$res = sum(nums[0:j+1]) - sum(nums[0:i])$$

用 arr 来记录扫描过程中的前缀和, cur 记录扫描过程的当前前缀和，维护 arr 有序检索尽量满足 cur - p <= k 的前缀和 p。arr 需要支持动态插入与二分查询，用平衡树可以得到最优的性能。

**时间复杂度 $O(nlog_n)$ ，空间复杂度 $O(n)$**

```python
# 这里用 bisect 只是因为 python 没有现成的平衡树，bisect.insort_left 并没有达到 log_n 的插入能力
# arr 初始化为 [0] 而不是 []，预处理子序列不为空的逻辑
cur, arr = 0, [0]
for s in nums:
    cur += s
    p = bisect.bisect_left(arr, cur - k)
    if p < len(arr):
        res = max(res, cur - arr[p])
    bisect.insort_left(arr, cur)
```

## 最大矩阵和

解决完一维的问题，可以回头看二维的问题了

> 给矩阵 matrix，求其子矩阵和最大且小于 k 的值

思路：在本题中，题目声明了行高远大于列宽。所以我们枚举列，压缩行，将二维问题转换为一维问题。

```python
def maxSumSubmatrix(self, matrix: List[List[int]], k: int) -> int:
    if not matrix or not matrix[0]:
        return 0
    n, m = len(matrix), len(matrix[0])
    res = float("-inf")
    for i in range(m):
        sum_list = [0] * n
        for j in range(i, m):
            for l in range(n):
                sum_list[l] += matrix[l][j]
            t, visit = 0, [0]
            for s in sum_list:
                t += s
                p = bisect.bisect_left(visit, t - k)
                if p < len(visit):
                    res = max(res, t - visit[p])
                bisect.insort_left(visit, t)
    return res
```

**时间复杂度 $O(m^2nlog_n)$，时间复杂度 $O(n)$**