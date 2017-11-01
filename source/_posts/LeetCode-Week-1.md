---
title: LeetCode-Week-1
date: 2017-10-28 15:13:19
categories:
- LeetCode
tags:
- LeetCode
---
>写在前面

实在是无法忍受自己写代码如此差劲，心里所想的东西完无法用代码表达出来，因此打算从头开始一点一点的刷 LeetCode 
我的计划是每天**至少**写两道题并进行总结，一周发一次Blog，虽然这个可能花费大量时间，但胜在能够打牢基础，我认为这还是值得的
每道题都会由以下几个部分组成：
- 题目
- 代码
- 如果没有代码，就写没有考虑到的地方
- 优秀的代码和解析


希望能坚持啊！兄弟！

## Spiral Matrix
Given a matrix of m x n elements (m rows, n columns), return all elements of the matrix in spiral order.

For example,
Given the following matrix:
```c++
[
 [ 1, 2, 3 ],
 [ 4, 5, 6 ],
 [ 7, 8, 9 ]
]
```
You should return `[1,2,3,6,9,8,7,4,5]`
### 解析
这道题就是以螺旋的方式遍历整个二维数组，我的思路是设置1个变量k用于计数，还有4个变量用于指向四个遍历方向，还有一个二维数组记录每一个位置上的数字是否被访问过，若是，跳过，有点傻有点啰嗦
### 代码
```c
 vector<int> spiralOrder(vector<vector<int>>& matrix) {
    vector<int>res;
    if(matrix.empty())
        return res;
	int m = matrix.size();
	int n = matrix[0].size();
	int a[50][50];
	memset(a, 0, sizeof(a));
	int k = 0, i, j;
	int q, w, e, r;
	q = r = 0;
	w = n-1;
	e = m-1;
	while (k < m*n)
	{
		for (i = 0; i < n; i++)
		{
			if (a[q][i] == 0)
			{
				res.push_back(matrix[q][i]);
				a[q][i] = 1;
				k++;
			}
		}
		q++;
		for (i = 0; i < m; i++)
		{
			if (a[i][w] == 0)
			{
				res.push_back(matrix[i][w]);
				a[i][w] = 1;
				k++;
			}
		}
		w--;
		for (i = n - 1; i >= 0; i--)
		{
			if (a[e][i] == 0)
			{
				res.push_back(matrix[e][i]);
				a[e][i] = 1;
				k++;
			}
		}
		e--;
		for (i = m - 1; i >= 0; i--)
		{
			if (a[i][r] == 0)
			{
				res.push_back(matrix[i][r]);
				a[i][r] = 1;
				k++;
			}
		}
		r++;
	}
	return res;
}
```
### 简便的代码
```c
public class Solution {
    public List<Integer> spiralOrder(int[][] matrix) {
        
        List<Integer> res = new ArrayList<Integer>();
        
        if (matrix.length == 0) {
            return res;
        }
        
        int rowBegin = 0;
        int rowEnd = matrix.length-1;
        int colBegin = 0;
        int colEnd = matrix[0].length - 1;
        
        while (rowBegin <= rowEnd && colBegin <= colEnd) {
            // Traverse Right
            for (int j = colBegin; j <= colEnd; j ++) {
                res.add(matrix[rowBegin][j]);
            }
            rowBegin++;
            
            // Traverse Down
            for (int j = rowBegin; j <= rowEnd; j ++) {
                res.add(matrix[j][colEnd]);
            }
            colEnd--;
            
            if (rowBegin <= rowEnd) {
                // Traverse Left
                for (int j = colEnd; j >= colBegin; j --) {
                    res.add(matrix[rowEnd][j]);
                }
            }
            rowEnd--;
            
            if (colBegin <= colEnd) {
                // Traver Up
                for (int j = rowEnd; j >= rowBegin; j --) {
                    res.add(matrix[j][colBegin]);
                }
            }
            colBegin ++;
        }
        
        return res;
    }
```
同样也是while中套着循环，同时也是利用了四个变量分别指向行、列的开始、结尾的位置，可以通过这四个变量设置循环的次数，而不用另外开辟一个数组来记录该数字是否被访问过。

## Maximum Subarray
Find the contiguous subarray within an array (containing at least one number) which has the largest sum.

For example, given the array`[-2,1,-3,4,-1,2,1,-5,4]`,
the contiguous subarray`[4,-1,2,1]`has the largest sum =`6`.
### 愚蠢的代码
这代码一看就是会超时的，想过使用DP算法，但是不知道怎么处理**连续**
```
int maxSubArray(vector<int>& nums) {
	int max_sum = INT_MIN;
	int i;
	for (i = 0; i < nums.size(); i++)
	{
		int temp = func(i, nums);
		max_sum = max(max_sum, temp);
	}
	return max_sum;
}

int func(int begin, vector<int> &nums)
{
	int sum = 0;
	int max_sum = nums[begin];
	int i;
	for (i = begin ; i < nums.size(); i++)
	{
		sum += nums[i];
		max_sum = max(sum, max_sum);
	}
	return max_sum;
}
```
### 优秀的代码
``` c
public int maxSubArray(int[] A) {
        int n = A.length;
        int[] dp = new int[n];//dp[i]表示从A[0]-A[i]中连续的字串最大值
        dp[0] = A[0];
        int max = dp[0];
        
        for(int i = 1; i < n; i++){
            dp[i] = A[i] + (dp[i - 1] > 0 ? dp[i - 1] : 0);
            max = Math.max(max, dp[i]);
        }
        
        return max;
}
```
当前的字串最大值，是当前的值 + 之前的的字串的值 是否大于0，若小于，相当于重新开始计算子串的最大值

## 56. Merge Intervals

Given a collection of intervals, merge all overlapping intervals.

For example,
Given`[1,3],[2,6],[8,10],[15,18]`,
return`[1,6],[8,10],[15,18]`.
### 解析
这道题看上去还是十分简单的，就是遍历如果前一个的end大于后一个start,则赋值并继续循环
如果不符合，就加到结果的`vector`中，并更新start和end的值
**注意点**

1. 首先要对其进行排序，按照start升序
2. 考虑多种情况,也有可能前一个包含后一个

### 代码
```
static bool cmp(const Interval &a, const Interval &b)
{
	return a.start < b.start;
}
    
vector<Interval> merge(vector<Interval>& intervals) {
	int i, j, k;
	vector<Interval>res;
    if (intervals.empty())
		return res;
    sort(intervals.begin(), intervals.end(), cmp);
	int start = intervals[i].start;
	int end = intervals[i].end;
	for (i = 1; i < intervals.size(); i++)
	{
        if (intervals[i].start >= start && intervals[i].end <= end)
            continue;
		if (intervals[i].start <= end)
		{
			end = intervals[i].end;
			continue;
		}
		else
		{
			Interval tmp = { start, end };
			res.push_back(tmp);
			start = intervals[i].start;
			end = intervals[i].end;
		}
	}
    Interval tmp = { start, end };
	res.push_back(tmp);
	return res;
    
}
```
## 55. Jump Game
Given an array of non-negative integers, you are initially positioned at the first index of the array.

Each element in the array represents your maximum jump length at that position.

Determine if you are able to reach the last index.

For example:
A = `[2,3,1,1,4]`, return `true`.

A = `[3,2,1,0,4]`, return `false`.

### 解题思路
用distance表示i之前最远的距离，如果 `i > diatance`，说明永远到达不了i,就退出循环
当distance >= n-1时，至少能跳到最后

夏令营时，我采用的是深搜，显然会超时

### 代码
```
bool canJump(vector<int>& nums) {
	int distance = 0;
	int n = nums.size();
	for (int i = 0; i < n && i <= distance; i++)
	{
		distance = max(nums[i] + i, distance);
	}
	if (distance < n - 1)
		return false;
	return true;
}
```
## 57.Insert Interval

Given a set of non-overlapping intervals, insert a new interval into the intervals (merge if necessary).

You may assume that the intervals were initially sorted according to their start times.

**Example 1:**
Given intervals`[1,3],[6,9]`, insert and merge `[2,5]` in as `[1,5],[6,9]`.

**Example 2:**
Given `[1,2],[3,5],[6,7],[8,10],[12,16]`, insert and merge `[4,9]` in as `[1,2],[3,10],[12,16]`.

This is because the new interval `[4,9]` overlaps with `[3,5],[6,7],[8,10]`.
### 解题思路
这个合并可以分为三个部分
1. `end < newInterval.stat`，直接将其添加至新的`res`中
2. `start > newInterval.end`，同上
3. 有部分相交，`start <= newInterval.end`的部分，则`start`取最小值，`end`取最大值，并`push`进入结果中
### 代码

```c
vector<Interval> insert(vector<Interval>& intervals, Interval newInterval) {
	int i = 0 ;
	int s = newInterval.start;
	int e = newInterval.end;
	vector<Interval>res;
	if (intervals.empty())
	{
		intervals.push_back(newInterval);
		return intervals;
	}
	//前部

	while (i < intervals.size() && newInterval.start > intervals[i].end)
	{
		res.push_back(intervals[i]);
		i++;
	}
	//中间
	while (i < intervals.size() && intervals[i].start <= newInterval.end)
	{
		s = min(s, intervals[i].start);
		e = max(e, intervals[i].end);
		i++;
	}
	Interval t{ s, e };
	res.push_back(t);

	//后部
	while (i < intervals.size() && newInterval.end < intervals[i].start)
	{
		res.push_back(intervals[i]);
		i++;
	}
	return res;
}
```

## Spiral Matrix II
Given an integer n, generate a square matrix filled with elements from 1 to n2 in spiral order.

For example,
Given n = `3`,

You should return the following matrix:
```
[
 [ 1, 2, 3 ],
 [ 8, 9, 4 ],
 [ 7, 6, 5 ]
]
```

### 解题思路
和之前的遍历类似，不再赘述，重要的是如何用verctor定义一个二维数组！！！
`vector<vector<int>> res(n,vector<int>(n))` !!!
`vector<vector<int>> res(n,vector<int>(n))` !!!
### 代码
```c
vector<vector<int>> generateMatrix(int n) {
	vector<vector<int>> res(n, vector<int>(n));//!!!!!!!!!!!!
	int i, j, k, l;
	int c = 1;
	i = j = 0;
	k = l = n - 1;
	while (c <= n*n) 
	{
		for (int m = j; m <= l; m++)
		{
			res[i][m] = c;
			c++;
		}
		i++;
		for (int m = i; m <= k; m++)
		{
			res[m][l] = c;
			c++;
		}
		l--;
		for (int m = l; m >= j; m--)
		{
			res[k][m] = c;
			c++;
		}
		k--;
		for (int m = k; m >= i; m--)
		{
			res[m][j] = c;
			c++;
		}
		j++;
	}
	return res;
}
```




