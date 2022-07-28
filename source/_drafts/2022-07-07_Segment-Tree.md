---
title: Segment-Tree
date: 2022-07-07
tags: [Segment-Tree]
categories: [算法]
---

算法数据结构之 - 线段树

<!-- more -->

## 线段树

线段树本质上是一个**满二叉数**形式的数据结构，用来组织离散型的区间数据，

### 线段树的构建

下面，以求任意区间的和为例，构建一棵线段树

#### 构建
```java
/*
* @param datas      表示需要构建线段树的原始数据
* @param nodes      表示截止目前递归为止的已经构建好的线段树
* @param start      表示需要构建线段树的原始数据的起点(包含)
* @param end        表示需要构建线段树的原始数据的终点(包含)
* @param node_idx   表示在当次递归中，需要构建的线段树在tree内的节点索引号。
*/
public void buildTree(int[] datas, int[] tree, int start, int end, int node_idx) {

    if (start == end) {
        tree[node_idx] = datas[start]; // 递归结束条件
        return;
    }

    int mid             =  start + (end - start) / 2; // mid 为原始数据在[start, end]范围内的需要做分割的点位
    int left_node_idx    = node_idx * 2 + 1;
    int right_node_idx   = node_idx * 2 + 2;

    buildTree(datas, nodes, start, mid, left_node_idx);
    buildTree(datas, nodes, mid + 1, end, right_node_idx);

    nodes[node_idx] = nodes[left_node_idx] + nodes[right_node_idx]; // fun(nodes[left_nodeIdx], nodes[right_nodeIdx]);
}
```

#### 查询
查询某个范围内的和

```java
/*
*/
public int queryTree(int[] tree, int start, int end, int node_idx, int L, int R, int val) {

    if (L <= start && R >= end) {
        return tree[node_idx];
    } else if (L > end && R < start) {
        return 0;
    }

    int mid = start + (end - start) / 2;;
    int left_node_idx = node_idx * 2 + 1;
    int right_node_idx = node_idx * 2 + 2;

    int left_sum = queryTree(tree, start, mid, left_node_idx, L, R, val);
    int right_sum = queryTree(tree, mid + 1, end, right_node_idx, L, R, val);

    return left_sum + right_sum;
}
```

#### 更新
更新某个叶子节点上的值

```java
public void updateTree(int[] tree, int start, int end, int node_idx, int idx, int new_val) {
    
    if (start == end && start == idx) { // 如果要求更新的idx没有出错的话，叶子节点一定就是需要更新的这个节点
        tree[node_idx] = new_val; 
        return;
    }

    int mid = start + (end - start) / 2;
    int left_node_idx = node_idx * 2 + 1;
    int right_node_idx = node_idx * 2 + 2;
    
    if (idx <= mid) {
        updateTree(tree, start, mid, left_node_idx, idx, new_val);
    } else {
        updateTree(tree, mid + 1, end, right_node_idx, idx, new_val);
    }
    
    tree[node_idx] = tree[left_node_idx] + tree[right_node_idx];
}
```

#### 懒标记

```java

```
## 实例