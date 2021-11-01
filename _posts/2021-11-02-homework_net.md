# 网络空间安全Homework
## 题目
在椭圆曲线  $y^2=x^3+x+1(mod\ 5)$ 上任意选择两个点，计算这两个点的阶数。
## 分析
1. 在椭圆曲线上任意选择两个点，此任意非真正的任意，而是要从椭圆曲线的有限域中存在的点当中选取。
2. 找到所有点后，再逐个计算其阶数

## 实现过程
### 1. 首先定义我们的椭圆曲线类：
```py
class Eclipse():
    def __init__(self, p, a, b, limit=1000):
        # 判断曲线参数是否满足奇异条件
        assert( 4 * a^3 + 27 * b ^ 3 != 0 )
        self.p = p
        self.a = a
        self.b = b
        self.limit = limit # 求阶数时的上限
        # 得到有限域上所有合理的点
        self.points = self._find_points()
```
> 其中`p`为椭圆曲线的取模因子


### 2. 计算出椭圆曲线有限域中所有合理的点
对于所有$x,y<p$，计算其是否满足椭圆曲线的公式：$y^2=x^3+ax+b(\bmod\ p)$，满足条件的，将其存入到一个列表中。
``` py

    # 找出所有曲线上存在的点
    def _find_points(self):
        result = []
        for x in range(self.p):
            for y in range(self.p):
                if(self._result_x(x) == y**2 % self.p):
                    result.append((x,y))
        return result
```

### 3. 定义各种运算操作
为了求出阶数，即 $nP=O_{\infty}$ 中的n，我们需要找出满足 $nP = -P$ 的 $n$，然后 $n+1$ 即为要求的阶数。为了计算 $nP$ 和 $-P$，我们需要定义点$P$的加法运算和负号运算，再利用加法运算定义 $P$ 的 数乘运算。
1. 加法运算:  
$P\left(x_{1}, y_{1}\right) + Q\left(x_{2}, y_{2}\right)$ 的和 $R\left(x_{3}, y_{3}\right)$ 有如下关系:

$$x_{3} \equiv k^{2}-x_{1}-x_{2} \quad(\bmod p)$$
$$y_{3} \equiv k\left(x_{1}-x_{3}\right)-y_{1} \quad(\bmod p)$$
$\quad$若 $P=Q$, 则 $$k=\frac{3 x_{1}^{2}+a}{2 y_{1}} \quad(\bmod p)$$
$\quad$ 若 $P \neq Q$, 则 $$k=\frac{y_{2}-y_{1}}{x_{2}-x_{1}} \quad(\bmod p)$$
``` py
    # 定义加法: 求 A+B 的和
    def plus(self, A, B):
        (x1,y1),(x2,y2) = A,B
        k = 0 # 中间变量k
        if A is B: # 如果A严格等于B
            k = ( (3 * x1**2 + self.a)/(2*y1) ) % self.p
        else:
            k = ( ( y2-y1 )/( x2-x1 ) ) % self.p
        x3 = k**2 - x1 - x2
        y3 = k*( x1-x3 ) - y1
        return( x3%self.p , y3%self.p )
```
2. 负号运算
``` py
    # 定义负号: 求-P
    def negativeP(self, P):
        return(( P[0] , (-P[1]) % self.p ))
```
3. 数乘运算:
```py
    # 定义数乘: 利用上面定义的加法求nP
    def nP(self, n, P):
        result = P
        for i in range(n):
            result = self.plus(result,P)
        return result
```
 
4. 最后，对找出的所有合理的点$P$，从$n=1$开始，计算是否满足$nP = -P$,若超出上限仍未找到，则返回ERR。 
```py
    def getOrder(self, P) -> int :
        for i in range(1, self.limit):
            a = self.nP(i,P)
            b = self.negativeP(P)
            if self.isEqual( a, b ):
                return i
        return -1
```

## 运行结果
对$y^2=x^3+1x+1(\bmod\ 5)$的配置：
```py
    e = Eclipse(5,1,1)
    for p in e.points:
        print(p, '=', e.getOrder(p)+1)
```
运行结果:
```s
$ python test.py
(0, 1) = 0
(0, 4) = 0
(2, 1) = 0
(2, 4) = 0
(3, 1) = 8
(3, 4) = 0
(4, 2) = 0
(4, 3) = 0
```

## 完整代码
```py
#! python3

# y^2 = x^3 + ax + b (mod p)
# p = 5
# a = 1
# b = 1

# Homework DDL: 11-05
# Given: y^2 = x^3 + x + 1 (mod 5)
# A.K.A: E5(1,1)

class Eclipse():
    def __init__(self, p, a, b, limit=1000):
        # 判断曲线参数是否满足奇异条件
        assert( 4 * a^3 + 27 * b ^ 3 != 0 )
        self.p = p
        self.a = a
        self.b = b
        self.limit = limit # 求阶数时的上限
        # 得到有限域上所有合理的点
        self.points = self._find_points()
    
    def getOrder(self, P) -> int :
        for i in range(1, self.limit):
            a = self.nP(i,P)
            b = self.negativeP(P)
            if self.isEqual( a, b ):
                return i
        return -1
        
    # 可视化原曲线
    def showOriginal(self,):
        import numpy as np
        import matplotlib.pyplot as plt
        y, x = np.ogrid[-5:5:100j, -5:5:100j]
        plt.contour(x.ravel(), y.ravel(), pow(y, 2) - pow(x, 3) - x * self.a - self.b, [0])
        plt.grid()
        plt.show()

    # 计算曲线方程等号右侧的数值
    def _result_x(self, x):
        return (x**3 + self.a * x + self.b) % self.p

    # 找出所有曲线上存在的点
    def _find_points(self):
        result = []
        for x in range(self.p):
            for y in range(self.p):
                if(self._result_x(x) == y**2 % self.p):
                    # print(x,y)
                    result.append((x,y))
        return result

    # 定义加法: 求 A+B 的和
    def plus(self, A, B):
        (x1,y1),(x2,y2) = A,B
        k = 0 # 中间变量k
        if A is B: # 如果A严格等于B
            k = ( (3 * x1**2 + self.a)/(2*y1) ) % self.p
        else:
            k = ( ( y2-y1 )/( x2-x1 ) ) % self.p
        x3 = k**2 - x1 - x2
        y3 = k*( x1-x3 ) - y1
        return( x3%self.p , y3%self.p )
    
    # 定义负号: 求-P
    def negativeP(self, P):
        return(( P[0] , (-P[1]) % self.p ))

    # 定义数乘: 利用上面定义的加法求nP
    def nP(self, n, P):
        # print('----',n,'----')
        # 用分治策略优化一下
        # if n == 1:
        #     return P
        # elif n%2 == 0: # n为偶数则折半计算 (n/2)P+ (n/2)P
        #     fac = self.nP( n/2, P )
        #     return self.plus( fac, fac )
        # else: # n为奇数，则计算 [ (n/2)P+ (n/2)P ] + P
        #     fac = self.nP( n//2, P )
        #     return self.plus( self.plus(fac,fac), P ) 
        result = P
        for i in range(n):
            result = self.plus(result,P)
        return result
    
    # 定义相等: 判断两个点是否相等
    @staticmethod
    def isEqual(A, B):
        return A[0] == B[0] and A[1] == B[1]


if __name__ == '__main__':
    e = Eclipse(5,1,1)
    for p in e.points:
        print(p, '=', e.getOrder(p)+1)


```