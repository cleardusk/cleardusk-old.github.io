---
layout: post
title: 优化角度解数独
category: 博客
keywords: 博客
---

这是一篇[旧文](https://www.zybuluo.com/cleardusk/note/638899)。

## 介绍
MIP(Mixed Interger Programming) 即整数规划可以解数独，以及数独的一些变种，如对角数独、杀手数独(Killer Sudoku)。

## 标准数独
标准 9x9 数独的规则是：每行、每列及 9 个 3x3 的子区域必须由 1 到 9 这 9 个数组成。

那么如何将解数独转换为一个整数规划问题呢？就像 [*算法设计与分析*](http://bioinfo.ict.ac.cn/~dbu/AlgorithmCourses/CS711008Z/CS711008Z_2016.html) 课上卜老师说的那句：一层窗户纸捅破了一文不值。


### Solving

> Mixed Integer Linear Programming problems are generally solved using a linear-programming based branch-and-bound algorithm.

大致思想就是将 MIP 不停地 branch 为 LP 问题，子问题的最优解构成原问题的最优解，思路还是 divide and conquer.
![branch-and-bound.png-21kB](http://static.zybuluo.com/cleardusk/x17qzslyxmr0yrjpon75598x/branch-and-bound.png)

有兴趣可以去仔细研究下算法，我就直接调用 [GUROBI](http://www.gurobi.com/) 的 Python API 来求解了。

我把核心代码贴在这(不是完整代码)：
```
from gurobipy import *
import math

# suppose puzzle is a nested list representing the origion sudoku problem，`*` will be set 0 in puzzle
"""
* * 5 3 * * * * * 
8 * * * * * * 2 * 
* 7 * * 1 * 5 * * 
4 * * * * 5 3 * * 
* 1 * * 7 * * * 6 
* * 3 2 * * * 8 * 
* 6 * 5 * * * * 9 
* * 4 * * * * 3 * 
* * * * * 9 7 * * 
"""

model = Model('sudoku')
# 3D array, n**3 variable
vars = model.addVars(n, n, n, vtype=GRB.BINARY, name='G')

# Fix specified value
for i in range(n):
    for j in range(n):
        if puzzle[i][j] > 0:
            v = puzzle[i][j] - 1
            vars[i, j, v].LB = 1  # each var in vars are BINARY, if set LB to 1, it is 1 exactly
            
# Constraint 1: each cell must take one variable
model.addConstrs((vars.sum(i, j, '*') == 1 for i in range(n) for j in range(n)), name='V')

# Constraint 2: each value must appears once per row
model.addConstrs((vars.sum(i, '*', v) == 1 for i in range(n) for v in range(n)), name='R')

# Constraint 3: each value must appears once per column
model.addConstrs((vars.sum('*', j, v) == 1 for j in range(n) for v in range(n)), name='C')

# Constraint 4: each value must appears each subgrid
s = int(math.sqrt(n))  # subgrid size
for v in range(n):
    for i0 in range(s):
        for j0 in range(s):
            model.addConstr(quicksum(vars[i, j, v] for i in range(i0 * s, (i0 + 1) * s)
                                     for j in range(j0 * s, (j0 + 1) * s)) == 1, name='Sub')

model.optimize()

# parse the result

"""
1 4 5 3 2 7 6 9 8 
8 3 9 6 5 4 1 2 7 
6 7 2 9 1 8 5 4 3 
4 9 6 1 8 5 3 7 2 
2 1 8 4 7 3 9 5 6 
7 5 3 2 9 6 4 8 1 
3 6 7 5 4 2 8 1 9 
9 8 4 7 6 1 2 3 5 
5 2 1 8 3 9 7 6 4
"""
```

Gurobi 的 Python API 初步使用不太容易，语法稍微晦涩，just accept it.

至于速度，Gurobi 真是快，本机测试一般是 **0.01s** 内解决。

### 补充
$d = 9$ 时是标准数独，$d = n^2$ 时基本都能求解的，我只试了下 $d=16$ 的，也是毫秒级别内解决。MIP 的复杂度，有兴趣可以仔细研究下。

## 对角数独
与标准数独相比，添加了对角线的约束：对角线上的元素。加个约束即可。

```
for v in range(n):
    model.addConstr(quicksum(vars[i, i, v] for i in range(n)) == 1, name='D')
    model.addConstr(quicksum(vars[i, n - 1 - i, v] for i in range(n)) == 1, name='D')
```

```
9 * * 1 * 5 * * 8 
* * * * * * * * * 
* * * * 4 * * * * 
* * 5 * * * 8 * * 
* * * * * * * * * 
6 9 * * * * * 3 4 
* * * * * * * * * 
4 7 * * 5 * * 2 3 
* 6 2 7 * 1 4 8 * 

# 注意对角线元素
9 2 4 1 3 5 6 7 8 
5 1 6 2 8 7 3 4 9 
7 8 3 6 4 9 1 5 2 
1 3 5 4 7 2 8 9 6 
2 4 8 9 6 3 5 1 7 
6 9 7 5 1 8 2 3 4 
8 5 9 3 2 4 7 6 1 
4 7 1 8 5 6 9 2 3 
3 6 2 7 9 1 4 8 5 
```

## 杀手数独
[规则参考](https://en.wikipedia.org/wiki/Killer_sudoku?oldformat=true)，只给定同一色块的和。同样是加约束，不过跟标准数独不大一样了。

![Killersudoku_color.svg-8.3kB](http://static.zybuluo.com/cleardusk/co76dusbps3dlbaq9a584o2b/Killersudoku_color.svg)

用 gurobi 求解：

```
2 1 5 6 4 7 3 9 8 
3 6 8 9 5 2 1 7 4 
7 9 4 3 8 1 6 5 2 
5 8 6 2 7 4 9 3 1 
1 4 2 5 9 3 8 6 7 
9 7 3 8 1 6 4 2 5 
8 2 1 7 3 9 5 4 6 
6 5 9 4 2 8 7 1 3 
4 3 7 1 6 5 2 8 9 
```

给出求解这个杀手数独的 demo 源码（由于 Gurobi 的 license 限制，代码仅供学术交流使用）
```
#!/usr/bin/env python
# coding: utf-8

from gurobipy import *
from collections import namedtuple

# 4x4 demo
# grid = [[1, 1, 2, 2], [1, 1, 2, 2], [3, 4, 5, 6], [3, 4, 5, 6]]
# grid_sum = {1: 10, 2: 10, 3: 5, 4: 5, 5: 5, 6: 5}

# 9x9 demo
grid = [[ 1,  1,  2,  2,  2,  3,  4,  5,  6],
        [ 7,  7,  8,  8,  3,  3,  4,  5,  6],
        [ 7,  7,  9,  9,  3, 10, 11, 11,  6],
        [12, 13, 13,  9, 14, 10, 11, 15,  6],
        [12, 16, 16, 17, 14, 10, 15, 15, 18],
        [19, 16, 20, 17, 14, 21, 22, 22, 18],
        [19, 20, 20, 17, 23, 21, 21, 24, 24],
        [19, 25, 26, 23, 23, 27, 27, 24, 24],
        [19, 25, 26, 23, 28, 28, 28, 29, 29]]
grid_sum = {1: 3, 2: 15, 3: 22, 4: 4, 5: 16, 6: 15, 7: 25, 8: 17, 9: 9, 10: 8, 11: 20, 12: 6, 13: 14, 14: 17, 15: 17,
            16: 13, 17: 20, 18: 12, 19: 27, 20: 6, 21: 20, 22: 6, 23: 10, 24: 14, 25: 8, 26: 16, 27: 15, 28: 13, 29: 17}


def fancy_print_sol(sol):
    n = len(sol[0])
    out = '\n'
    for i in range(n):
        for j in range(n):
            out += str(sol[i][j]) + ' '
        out += '\n'
    print(out)


def solve():
    n = len(grid[0])
    model = Model('sudoku_killer')
    vars = model.addVars(n, n, n, vtype=GRB.BINARY, name='G')

    # s.t
    model.addConstrs((vars.sum(i, j, '*') == 1 for i in range(n) for j in range(n)), name='V')
    model.addConstrs((vars.sum(i, '*', v) == 1 for i in range(n) for v in range(n)), name='R')
    model.addConstrs((vars.sum('*', j, v) == 1 for j in range(n) for v in range(n)), name='C')

    # get (i,j) pair
    grid_type = {grid[i][j]: [] for i in range(n) for j in range(n)}
    Pos = namedtuple('Pos', ('row', 'col'))
    for i in range(n):
        for j in range(n):
            grid_type[grid[i][j]].append(Pos(i, j))

    # s.t
    for tp in grid_type.keys():
        ps = grid_type.get(tp)
        model.addConstr(quicksum(vars[i, j, v] * (v + 1) for v in range(n) for i, j in ps) == grid_sum.get(tp), name='S')

    model.optimize()

    # parse solution
    solution = model.getAttr('X', vars)
    sol = []
    for i in range(n):
        row = []
        for j in range(n):
            for v in range(n):
                if solution[i, j, v] == 1:
                    row.append(v + 1)
        sol.append(row)
    fancy_print_sol(sol)
    
if __name__ == '__main__':
    solve()
```

## 小结
本文是从优化角度看数独的，即 MIP。至于难度，我觉得跟算法课的作业题目在一个水平。
数独这个问题已经被研究得透彻了，本文也并未提出什么有价值的新东西，全是陈旧之物，所以写出来直接分享给诸位了。

我曾经下过一个求解数独的程序（应该是 backtrack 算法），多线程跑的，跑本文标准数独（难度系数很高）的那个例子，花了十分钟跑完；现在从优化的角度，0.01s 就搞定了。优化是一种工具，也是一种看待问题的方式；动态规划、贪心等基础算法，也可以理解为优化。

我不希望本文会给玩数独的人带来失望，我只能说：在计算机强大的计算能力面前，我们人类的计算能力不值一提。围棋都已被征服，更何况这数独。

---

若有问题，欢迎[交流](https://github.com/cleardusk)
