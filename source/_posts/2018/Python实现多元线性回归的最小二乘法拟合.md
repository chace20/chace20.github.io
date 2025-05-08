---
title: Python实现多元线性回归的最小二乘法拟合
date: 2018-04-15 12:56:30
tags: 机器学习
---

最近用到了多元线性回归，因为不能调库，所以手动实现了一下，其实最麻烦的还是在矩阵操作上面。

# 理论部分
多元线性回归定义如下：
![多元线性回归](/img/2018/04/多元线性回归.png)
![多元线性回归矩阵](/img/2018/04/多元线性回归矩阵.png)

**多元线性回归的拟合一般使用最小二乘法，所谓最小二乘法即是使得残差平方和最小来求参数估计量的一种方法。**如下图所示：
![最小二乘法](/img/2018/04/最小二乘法.png)

**上图和下图说的都是满秩矩阵的情况，也就是说数据条数要大于等于数据特征数。大部分情况下应该都是满秩矩阵，但是在样本少、特征数多的情况下，就会出现奇异矩阵的情况。**

推导过程如下：
![最小二乘法的矩阵形式推导](/img/2018/04/最小二乘法矩阵推导.png)


# 封装的矩阵操作
```
# coding=utf8
from __future__ import division
import copy


def list_to_matrix(l):
    """
    列表转变为1*n矩阵
    :param l: list
    :return: 矩阵
    """
    m = [l]
    return m


def inverse(matrix):
    """
    矩阵高斯消元法求逆
    :param matrix: 
    :return: 逆矩阵
    """
    extend_matrix = copy.deepcopy(matrix)
    l = len(matrix)
    for i in range(0, l):    # 在矩阵右边补充一个单位矩阵，使用初等变换求逆矩阵
        extend_matrix[i].extend([0]*i)
        extend_matrix[i].extend([1])
        extend_matrix[i].extend([0]*(l-i-1))
    for i in range(0, len(extend_matrix)):    # 判断矩阵对角线上是否有0，有0则置换，如置换不了，则没有逆矩阵
        if extend_matrix[i][i] == 0:
            for j in range(i, len(extend_matrix)):
                if extend_matrix[j][i] != 0:  # 进行行交换
                    extend_matrix[i], extend_matrix[j] = extend_matrix[j], extend_matrix[i]
                    break
            if j >= len(extend_matrix)-1:
                print('没有逆矩阵')
                return 0
            break
    for i in range(0, len(extend_matrix)):    # 开始计算逆矩阵
        f = extend_matrix[i][i]
        for j in range(0, len(extend_matrix[i])):    # 先把对角元素换为1
            extend_matrix[i][j] /= f
        for m in range(0, len(extend_matrix)):
            if m == i:
                continue
            b = extend_matrix[m][i]
            for n in range(0, len(extend_matrix[i])):    # 再把对角元素所在列的其余元素换为0
                extend_matrix[m][n] -= extend_matrix[i][n] * b
    for i in range(0, len(extend_matrix)):
        extend_matrix[i] = extend_matrix[i][l:]
    return extend_matrix


def transpose(matrix):
    """
    矩阵转置
    :param matrix: 
    :return: 
    """
    return list(map(list, zip(*matrix)))


def dot(A, B):
    """
    矩阵乘法
    :param A: 
    :param B: 
    :return: 
    """
    return [[sum(x*y for x, y in zip(a, b)) for b in zip(*B)] for a in A]


def shape(matrix):
    return len(matrix), len(matrix[0])


def add_intercept(matrix):
    """
    在矩阵的第一列插入全为1的一列
    :param matrix: 待插入矩阵
    :return: 新的矩阵
    """
    for row in matrix:
        row.insert(0, 1)
    return matrix

```

# 最小二乘法实现
```
# coding=utf-8
import matrix


class LinearRegression:
    def __init__(self):
        # param保存估计参数
        self.param = matrix.list_to_matrix([])

    def fit(self, X, y):
        # 在最后添加一列1
        X = matrix.add_intercept(X)
        X_T = matrix.transpose(X)
        # 最小二乘法估计参数
        self.param = matrix.dot(matrix.dot(matrix.inverse(matrix.dot(X_T, X)), X_T), y)

    def predict(self, X):
        X = matrix.add_intercept(X)
        result = matrix.dot(X, self.param)
        return result

```

# 非线性回归
一些特殊的非线性模型可以线性化，比如“对数线性回归”。

另一类是多项式回归模型，可以通过转变原特征的方式转换为多元线性回归。比如，对于一元多项式回归，就可通过下图所示方式转变为普通的多元线性回归模型，其中X是一个[范德蒙德矩阵](https://zh.wikipedia.org/wiki/%E8%8C%83%E5%BE%B7%E8%92%99%E7%9F%A9%E9%99%A3)。
![一元多项式回归](/img/2018/04/一元多项式回归.png)

对于更一般的多元多项式回归，则需要通过将原特征转变为多项式特征的方式。在sklearn中，可以使用[PolynomialFeatures](http://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.PolynomialFeatures.html)将原特征矩阵转变为多项式特征矩阵。

多项式回归比线性回归具有更好的拟合效果，但是也会带来一些问题。首先是多项式特征矩阵的维度将会随着最高次数的增加急剧上升，从而导致矩阵不可逆。这个时候可以使用**广义逆矩阵**进行求解，解的公式与上文的公式略有不同，详见**参考资料1**。多项式最高次数增加带来的另一个问题是过拟合，这个问题笔者还未做研究，故暂且搁置不提。

另一方面，对于多项式回归问题，因为上述的问题，放弃最小二乘，转而使用[岭回归](https://en.wikipedia.org/wiki/Tikhonov_regularization)可能效果更好。


# 参考资料
**理论**
1. [一般多元线性回归模型](http://read.pudn.com/downloads164/ebook/750809/chapter2.pdf)
2. [多元线性回归](http://www.cnblogs.com/zgw21cn/archive/2008/12/24/1361287.html)

**实践**
1. [Python计算大矩阵的逆的精度问题？](https://segmentfault.com/q/1010000006120590/a-1020000006120650)
2. [Python LU 分解](http://www.sharejs.com/codes/python/7414)
3. [机器学习：scipy和sklearn中普通最小二乘法与多项式回归的使用](http://www.cnblogs.com/lc1217/p/6829319.html)
