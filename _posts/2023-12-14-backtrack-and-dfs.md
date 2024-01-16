---
layout: default
title: 回溯和DFS
date: 2023-12-14
category: algo
---


[回溯](http://en.wikipedia.org/wiki/Backtracking) 是一种更加通用的算法。

[DFS](http://en.wikipedia.org/wiki/Depth-first_search) 是一种基于搜索树结构的数据的回溯。

## leetcode例题：[括号生成](https://leetcode.cn/problems/generate-parentheses/description/)，分别使用回溯和dfs的解法

```java
public List<String> generateParenthesis(int n) {
        List<String> res = new ArrayList<>();
        if (n <= 0) return res;
        dfs(n, n, res, "");
//        backtracking(n, n, res, new StringBuilder());
        return res;
    }

    /**
     * 回溯：需要通过撤回当前状态，再继续进入新的状态来寻找新的结果
     * @param leftNum
     * @param rightNum
     * @param res
     * @param sb 回溯需要记录的执行路径，每次进入递归都是同一个StringBuilder对象
     */
    private void backtracking(int leftNum, int rightNum, List<String> res, StringBuilder sb) {
        if (leftNum == 0 && rightNum == 0) {
            res.add(sb.toString());
            return;
        }
        if (leftNum > rightNum) {
            return;
        }

        if (leftNum > 0) {
            backtracking(leftNum - 1, rightNum, res, sb.append("("));
            sb.deleteCharAt(sb.length() - 1);
        }
        if (rightNum > 0) {
            backtracking(leftNum, rightNum - 1, res, sb.append(")"));
            sb.deleteCharAt(sb.length() - 1);
        }
    }


    /**
     * dfs:不需要记录递归的路径
     * @param leftNum
     * @param rightNum
     * @param res
     * @param sb 当前的sb是string，每次进入递归都是一个新的对象，不需要撤回状态
     */
    private void dfs(int leftNum, int rightNum, List<String> res, String sb) {
        if (leftNum == 0 && rightNum == 0) {
            res.add(sb);
            return;
        }
        if (leftNum > rightNum) {
            return;
        }

        if (leftNum > 0) {
            dfs(leftNum - 1, rightNum, res, sb + "(");
        }
        if (rightNum > 0) {
            dfs(leftNum, rightNum - 1, res, sb + ")");
        }
    }
```

backtraking的代码确实有个特点是recursive call之后要退回到之前的一个状态；

这里使用了dfs是将解题的方式抽象成了树；

区别不大，回溯像是dfs的更高层的抽象;


*回溯搜索是深度优先搜索（DFS）的一种对于某一个搜索树来说（搜索树是起记录路径和状态判断的作用）回溯和DFS其主要的区别是回溯法在求解过程中不保留完整的树结构而深度优先搜索则记下完整的搜索树为了减少存储空间在深度优先搜索中用标志的方法记录访问过的状态这种处理方法使得深度优先搜索法与回溯法没什么区别了*


ref：
- https://stackoverflow.com/questions/1294720/whats-the-difference-between-backtracking-and-depth-first-search
