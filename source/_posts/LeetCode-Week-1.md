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
## Unique Paths II
Follow up for "Unique Paths":

Now consider if some obstacles are added to the grids. How many unique paths would there be?

An obstacle and empty space is marked as `1` and `0` respectively in the grid.

For example,
There is one obstacle in the middle of a 3x3 grid as illustrated below.
```
[
  [0,0,0],
  [0,1,0],
  [0,0,0]
]
```
The total number of unique paths is `2`.

Note: m and n will be at most 100.

### 解决思路
这道题采用DP算法，到达当前点的路径数量等于左边和上面点的路径之和，同时要考虑边界的问题以及碰到障碍物时的处理办法
###代码
```
int uniquePathsWithObstacles(vector<vector<int>>& obstacleGrid) {
	int row = obstacleGrid.size();
	int col = obstacleGrid[0].size();
	int i, j;
	vector<vector<int>> dp(row, vector<int>(col, 0));
	for (i = 0; i < row; i++)
	{
		for (j = 0; j < col; j++)
		{
			if (obstacleGrid[i][j] == 1)
				dp[i][j] = 0;
			else if (i == 0 && j == 0)
				dp[i][j] = 1;
			else if (i == 0 && j > 0)
				dp[i][j] = dp[i][j - 1];
			else if (j == 0 && i > 0)
				dp[i][j] = dp[i - 1][j];
			else if (i > 0 && j > 0)
				dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
		}
	}
	return dp[row - 1][col - 1];
}
```
还有直接将边界扩大一圈，直接考虑边界的问题，代码如下
```
 int uniquePathsWithObstacles(vector<vector<int>>& obstacleGrid) {
        int m = obstacleGrid.size(), n = obstacleGrid[0].size();
        vector<vector<int> > dp(m + 1, vector<int> (n + 1, 0));
        dp[0][1] = 1;
        for (int i = 1; i <= m; i++)
            for (int j = 1; j <= n; j++)
                if (!obstacleGrid[i - 1][j - 1])
                    dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        return dp[m][n];
    } 
```

## 93. Restore IP Addresses
Given a string containing only digits, restore it by returning all possible valid IP address combinations.

For example:
Given `"25525511135"`,

return `["255.255.11.135", "255.255.111.35"]`. (Order does not matter)
### 解题思路
采用dfs，每一次长度为1~3，如果这一小块的长度为3，且值>255不符合；如果起始值为0且长度>1则不符合
最终当count==4并且指针指向结尾的时候符合要求并添加进结果中
### 代码
```
void dfs(string ip,vector<string>& res, int idx, int count, string s)
{
	if (count > 4)
		return;
	if (count == 4 && idx == ip.length())
	{
		res.push_back(s);
		return;
	}
	for (int i = 1; i < 4; i++)
	{
		if (idx + i > ip.length())
			break;
		string t = ip.substr(idx , i);
		if ((t[0] == '0' && t.length() > 1) || (i == 3 && atoi(t.c_str()) > 255))
			continue;
		dfs(ip, res, idx + i, count + 1, s + t + (count == 3 ? "" : "."));
	}
}


vector<string> restoreIpAddresses(string s) {
	vector<string>res;
	dfs(s, res, 0, 0, "");
	return res;
}
```
## 79. Word Search
Given a 2D board and a word, find if the word exists in the grid.

The word can be constructed from letters of sequentially adjacent cell, where "adjacent" cells are those horizontally or vertically neighboring. The same letter cell may not be used more than once.

For example,
Given **board** =
```
[
  ['A','B','C','E'],
  ['S','F','C','S'],
  ['A','D','E','E']
]
```
word = `"ABCCED"`, -> returns `true`,
word = `"SEE"`, -> returns `true`,
word = `"ABCB"`, -> returns `false`.
### 解题思路
这一次真的可以采用深搜的方法，从一个匹配的字母处开始寻找，规定查找的四个方向，如果当前字母与字符串中的字母相同，搜索下一个字母。
同时，不能重复字母，因此可以创建一个与`board`相同大小的二维向量用于表示是否访问过
记得回溯！
## 代码
```
void find(int i, int j, int index, string word, vector<vector<char>>& board, int& res, vector<vector<int>>& visited)
{
    int idx[] = { 1, -1, 0, 0 };
    int idy[] = { 0, 0, -1, 1 };
	if (res == 0)
	{
		if (i < 0 || i >= board.size() || j < 0 || j >= board[i].size()||visited[i][j])
			return;
		if (board[i][j] != word[index])
			return;
		if (index == word.length() - 1)
		{
			res = 1;
			return;
		}
		for (int k = 0; k < 4; k++)
		{
			int tx = i + idx[k];
			int ty = j + idy[k];
			visited[i][j] = 1;
			find(tx, ty, index + 1, word, board, res,visited);
			visited[i][j] = 0;

		}
	}
}

bool exist(vector<vector<char>>& board, string word) {
	vector<vector<int>> visited(board.size(), vector<int>(board[0].size(), 0));
	int res = 0;
	int i, j, k;
	for (i = 0; i < board.size(); i++)
	{
		for (j = 0; j < board[i].size(); j++)
		{
			if (board[i][j] == word[0])
			{
				find(i, j, 0, word, board, res,visited);
				if (res)
					break;
			}
			if (res)
				break;
		}
	}
	return res;
}
```
## 后记
这一周也终于结束了，感觉好像有点堕落有点拖拉，效率很低，希望加把劲咯
总是需要一门很熟悉的语言！go 和 C++?






