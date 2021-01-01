---
title: 中位数在线算法
date: 2020-12-30 23:31:15
math: true
categories:
- algorithm
---

动态返回输入序列的中位数。

<!--more-->

## 思考

中位数的离线算法没有任何难度，排序取中位元素即可，但在线算法并没有这么直观。显然，这个问题在不卡输入数据的情况下，不可能通过动态规划等思路压缩状态来降低时间的消耗，则需要在插入时维护已有数据集，来加速查找中位数的操作。那么对一个在线的算法需要考虑的操作需包括：

- 查找插入位置
- 插入
- 查找中位数

可以简单考虑下几种常用数据结构能否直接满足此诉求

- 链表

寻找插入点 $O(n)$，插入 $O(1)$，查找中位点 $O(n)$。

一种可能的优化点：由于每次仅插入一个元素，取 p 常驻于中点，根据插入点在 p 的左右，相应移动 p 维护其中点位置，可以将查找优化到 $O(1)$。

- 数组

寻找插入点 $O(log_2(n))$，插入 $O(n)$，查找中位点 $O(1)$。

- 堆

寻找插入点 $O(log_2(n))$，插入 $O(log_2(n))$，查找中位点 $O(log_2(n)*n)$ (先 pop 一半再 push 回去)。

- 排序树

寻找插入点 $O(log_2(n))$，插入 $O(1)$，查找中位点 $O(n)$。

一种可能的优化点：对每个节点维护其子节点个数，在树的旋转时维护该值，可以将查找优化到 $O(log_2(n))$

另一种可能的优化点：类似上面提到的链表优化，先取 p 常驻于中点，根据插入点维护 p 的中点位置，可以将查找优化到 $O(1)$

## 解法

显然，上面几个思路中，只有通过排序树的搞法是最优的。但在实际实现中，依赖具体语言的容器类是否支持此类操作，否则还需要自行实现一棵排序树。一般而言，会使用另外一种更通用便于实现的方法。

**大顶堆 + 小顶堆**

每次插入元素时，维护两个容量相差不大于 1 的大顶堆和小顶堆，且满足对于小顶堆中每个元素，均大于或等于大顶堆的每个元素。执行以下算法：

```python
add k:
  # 新元素 k 与两堆顶元素比较，加入满足要求的一个堆
  if k > max_heap.top:
      max_heap.push(k)
  else:
      min_heap.push(k)
  # 调整两个堆的大小直至相差在 1 以内
  while max_heap.len + 1 < min_heap.len:
      max_heap.push(min_heap.pop) 
  while min_heap.len + 1 < max_heap.len:
      min_heap.push(max_heap.pop) 
```

注意在每次添加完元素后，都需要调整两个堆的元素。且由于之前插入时已经调整过一次，那么每次至多两个堆各调整一次即可重新平衡，插入元素时间复杂度为 $O(log_2n)$。

而对于查找中位数，显然有中位数必然为两堆顶之一，或两堆顶的平均数，时间复杂度 $O(1)$。

在此，我们通过两道题目来解决这个问题。

## 数据流的中位数

> [295. 数据流的中位数](https://leetcode-cn.com/problems/find-median-from-data-stream)
> 
> 初始数组为空，持续向数组添加元素，并随时返回其中位数。

标准题目，思路如上所述，以 `add 5, 10, 20, 30` 为例，流程见下图

![295_flow](/img/median_0.png)

```python
class Heap:
    def __init__(self):
        self.data = []
        self.k = 1

    def max_heap(self):
        self.k = -1
        return self

    def top(self):
        return self.data[0] * self.k

    def pop(self):
        return self.k * heapq.heappop(self.data)

    def push(self, item):
        return heapq.heappush(self.data, item * self.k)

    def len(self):
        return len(self.data)

    def empty(self):
        return len(self.data) == 0


class MedianFinder:

    def __init__(self):
        self.min_part = Heap().max_heap()
        self.max_part = Heap()

    def addNum(self, num: int) -> None:
        m1, m2 = self.min_part, self.max_part
        m1.push(num) if not m1.empty() and num < m2.top() else m2.push(num)
        while m2.len() > m1.len() + 1:
            m1.push(m2.pop())
        while m1.len() > m2.len() + 1:
            m2.push(m1.pop())

    def findMedian(self) -> float:
        l1, l2 = self.min_part.len(), self.max_part.len()
        if l1 > l2:
            return self.min_part.top()
        elif l1 < l2:
            return self.max_part.top()
        else:
            return (self.min_part.top() + self.max_part.top()) / 2
```

## 滑动窗口中位数

> [480. 滑动窗口中位数](https://leetcode-cn.com/problems/sliding-window-median)
> 
> 给定数组 nums 与窗口大小 k，求数组从左向右每个窗口的中位数

大的思路不变，还是维护双堆。与 **295** 相比，多了一个窗口限制，即不能只考虑向两个堆中加元素，还得考虑移除的情况。然而，从堆中挪走一个特定元素时间复杂度为 $O(nlog_2n)$。若每次窗口滑动时，都去即时移除堆中元素，复杂度会直接退化到维护一个堆的做法。考虑到在本题的要求下，即时移除元素是不必要的操作，因为要求的是中位数，若同时向两个堆中塞入不等于堆顶的元素，中位数不改变。也就是说，不在堆顶的元素实际上不影响我们对中位数的求解，可以做延迟删除。

引入`balance` 变量来记录**实际**上大顶堆比小顶堆多的元素数量，每次用 `multiset` 来记录移除元素。思路类似，每次调整到 `0 <= balance <= 1`，调整策略为若堆顶元素为已删除元素，直接抛出，否则在两堆间迁移直至平衡。 


```python
class Heap:
    def __init__(self):
        self.data = []
        self.k = 1

    def max_heap(self):
        self.k = -1
        return self

    def top(self):
        return self.data[0] * self.k

    def pop(self):
        return self.k * heapq.heappop(self.data)

    def push(self, item):
        return heapq.heappush(self.data, item * self.k)

    def len(self):
        return len(self.data)

    def empty(self):
        return len(self.data) == 0


class MultiSet:
    def __init__(self):
        self.data = {}

    def add(self, x):
        if x not in self.data:
            self.data[x] = 0
        self.data[x] += 1

    def remove(self, x):
        self.data[x] -= 1
        if self.data[x] == 0:
            del (self.data[x])

    def contains(self, x) -> bool:
        return x in self.data


class Solution:
    def medianSlidingWindow(self, nums: List[int], k: int) -> List[float]:
        res, m1, m2, cache, balance = [], Heap().max_heap(), Heap(), MultiSet(), 0
        for i, x in enumerate(nums):
            if m1.empty() or x <= m1.top():
                m1.push(x)
                balance += 1
            else:
                m2.push(x)
                balance -= 1
            if i >= k:
                cache.add(nums[i - k])
                balance += -1 if nums[i - k] <= m1.top() else 1
            while balance != 0 and balance != 1 or not m1.empty() and cache.contains(
                    m1.top()) or not m2.empty() and cache.contains(
                    m2.top()):
                if balance > 1:
                    m2.push(m1.pop())
                    balance -= 2
                if balance < 0:
                    m1.push(m2.pop())
                    balance += 2
                while not m1.empty() and cache.contains(m1.top()):
                    cache.remove(m1.pop())
                while not m2.empty() and cache.contains(m2.top()):
                    cache.remove(m2.pop())
            if i >= k - 1:
                if balance == 0:
                    res.append((m1.top() + m2.top()) / 2)
                else:
                    res.append(m1.top() if balance == 1 else m2.top())
        return res
```