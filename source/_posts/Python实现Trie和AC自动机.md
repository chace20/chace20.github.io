---
title: Python实现Trie和AC自动机
date: 2017-11-12 16:04:42
tags: 
- Python
- 数据结构
---

上课的时候，其中一个task需要用前缀结构来降低空间开销，同时为了满足查找的需要，要用到AC自动机，因此接触了这两个数据结构。

本文只是对这两个知识点的Note，并没有太多原创的东西。

## 介绍
字典树(Trie)是一种前缀结构，每个节点值是一个字符，从根节点到某节点所经过的所有节点值即代表了该节点保存的字符串，因此具有相同前缀的字符串不会重复保存前缀，只会保存一次，因此可以节省空间开销且查找速度极快。**字典树为了保证效率，适合节点值集合较小（如英文的26个字母）且前缀较多的符号集（中英文文章都满足这两个要求）**。我做的task为了满足这两个要求，需要做一些特殊处理。

AC自动机(Aho-Corasick Automaton)其实可以看做状态转移矩阵，主要用于多模式匹配。其在字典树的基础上增加了一个fail指针，指向叶子节点的最长后缀。在匹配的过程中，当匹配到叶子节点的时候，会跳转到叶子节点的最长后缀继续匹配，这样就可以节省一部分回溯的时间。

原理及详细介绍可以阅读本文后的参考文献或者原始论文，本文不再赘述。

## Python代码实现
在建立字典树的过程中，我们使用Python中的dict结构来保存节点值和子节点的链接关系，即在节点的children域中保存的是该节点所有的子节点。即node.chidren={child_node_value1: child_node1, child_node_value2:child_node2,…}。

因为dict结构的内部实现使用的是hash表，相对于使用左孩子右兄弟表示树的方式，dict嵌套表示树的方式在查找插入的时候效率更高。当然，hash是用空间换时间，空间开销自然就会上升。

``` 
import sys
import copy

class Node:
    def __init__(self):
        self.value = None
        self.fail = None  # fail指针是构造AC自动机的关键
        self.children = {}  # dict {value: node}

class Trie:
    def __init__(self):
        self.root = Node()

    def insert(self, data):
        current = self.root
        for s in data:
            if s not in current.children:
                child = Node()
                child.value = s
                current.children[s] = child
                current = child
            else:
                current = current.children.get(s)

    def build_ac_automation(self):
        queue = [self.root]
        while len(queue) > 0:
            # 本质上来说是层次遍历
            temp = queue.pop()
            for k in temp.children:
                if temp == self.root:
                    # 第一层节点的fail指向root
                    temp.children[k].fail = self.root
                else:
                    p = temp.fail
                    while p is not None:
                        # p==None即代表找到了root
                        if k in p.children:
                            # 如果在当前节点的fail指针指向的节点的子节点中找到了k
                            temp.children[k].fail = p.children[k]
                            break
                        # 没有找到则继续沿着fail指针回溯找
                        p = p.fail
                    if p is None:
                        # 将k的fail指针指向root
                        temp.children[k].fail = self.root
                # 将子节点加进队列
                queue.append(temp.children[k])

```

---
## 参考文献
1. [【模式匹配】Aho-Corasick自动机](http://www.cnblogs.com/en-heng/p/5247903.html)
2. [基于trie树做一个ac自动机](http://chuxiuhong.com/2016/12/14/%E5%9F%BA%E4%BA%8Etrie%E6%A0%91%E5%81%9A%E4%B8%80%E4%B8%AAac%E8%87%AA%E5%8A%A8%E6%9C%BA/)
3. [Python3实现AC自动机](https://www.ctolib.com/topics-106266.html)
4. Aho, Alfred V., and Margaret J. Corasick. "Efficient string matching: an aid to bibliographic search." Communications of the ACM 18.6 (1975): 333-340.
