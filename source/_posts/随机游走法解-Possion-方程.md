---
title: 随机游走法解 Possion 方程
date: 2018-12-10 13:58:54
categories:
- 计算物理
tags:
top_img: ./黄昏.jpg
---


Possion 方程是指形如 $\Delta \varphi = f$ 的方程，在三维欧几里得空间中 $\Delta=\nabla^2=\frac{\partial^2}{\partial x^2}+\frac{\partial^2}{\partial y^2}+\frac{\partial^2}{\partial z^2}$ ，这类方程在物理上有很多应用，如在静电学中给定电荷分布求解电势的方程 $\nabla^2\Phi=-\rho/\epsilon_0$ ，牛顿力学中的引力场方程 $\nabla^2\phi=4\pi G\rho$，达到定态时的热传导方程 $k\nabla^2u=0$ 。本文介绍一种解 Possion 方程的数值方法，随机游走法。

作为一种数值解法，我们首先考虑将 Possion 方程离散化，只求解离散的格点上的函数值。

$$
\varphi(x,y) \mapsto \varphi_{ij},f(x,y) \mapsto f_{ij}
$$

然后是 $\nabla^2$ 的离散化：

$$
\frac{\partial \varphi}{\partial x}\mapsto \frac{\varphi(x+h/2,y)-\varphi(x-h/2,y)}{h}
$$

考虑格点间距 $h=1$ ，有

$$
\frac{\partial^2 \varphi}{\partial x^2}\mapsto\varphi(x+1,y)+\varphi(x-1,y)-2\varphi(x,y) 
$$

$$
\nabla^2 \varphi=\frac{\partial^2 \varphi}{\partial x^2}+\frac{\partial^2 \varphi}{\partial y^2}=f\mapsto \varphi_{i,j+1}+\varphi_{i,j-1}+\varphi_{i+1,j}+\varphi_{i-1,j}-4\varphi_{ij}=f_{ij} 
$$

上式即为 Possion 方程的离散形式，可以看到，这是一个极其稀疏的线性方程组 $A\varphi=f$ ，如果我们求解的区间内有 $100\times100$ 个格点， $A$ 即为一个 $100\times100$ 的矩阵，而其每一行只有 $5$ 个系数不为 $0$。我们可以通过直接求解这个线性方程组来解出 Possion 方程，但本文要介绍另外一种算法：随机游走法。

将离散化的 Possion 方程稍微改变一下形式：

$$
\varphi_{ij}=\frac{1}{4}(\varphi_{i,j+1}+\varphi_{i,j-1}+\varphi_{i+1,j}+\varphi_{i-1,j})+f'_{ij} 
$$

等式右边的第一项可以看作是一次对 

$$
\varphi_{i,j+1},\varphi_{i,j-1},\varphi_{i+1,j},\varphi_{i-1,j}
$$

这四个值的等概率随机抽样的均值，记单次抽样结果为 

$$
\varphi_{\{\varphi_{i,j+1},\va{% asset_img ./test.png title %}rphi_{i,j-1},\varphi_{i+1,j},\varphi_{i-1,j}\}}
$$

将单次随机抽样计算得到的 $\varphi \small{ij}$ 记为 $\phi \small{ij}$ ，上式可以改写为

$$
\varphi_{ij}=E(\phi_{ij})=E(\varphi_{\{\varphi_{i,j+1},\varphi_{i,j-1},\varphi_{i+1,j},\varphi_{i-1,j}\}}+f'_{ij})=E(\varphi_{\{\phi_{i,j+1},\phi_{i,j-1},\phi_{i+1,j},\phi_{i-1,j}\}}+f'_{ij}) 
$$

这个式子就是随机游走法的原理。

我们将 $E$ 移到最外层，即将求均值的计算放到最后一步，这样计算单个格点的值的抽样就变成了单次抽样，这等价于一个朝上下左右四个方向的随机游走。

我们从区间中的任意一个格点出发随机游走到区间的边界，该格点的抽样值即为边界条件所确定的边界值加上路径上的 $f'_{ij}$ 的求和，最后多次抽样取平均即可得到 Possion 方程的近似解。

我用 Julia 实现了这个算法，求解了几个方程并画出了函数图像：

- 边界为正弦函数的 Laplace 方程：

{% asset_img 00.jpg %}
{% asset_img 01.png %}

- 边界为阶跃函数的 Laplace 方程：

{% asset_img 10.png %}
{% asset_img 11.png %}

- 边界为 0 但非其次项在中心不为 0 的 Possion 方程：

{% asset_img 20.png %}
{% asset_img 21.png %}

附上代码：

```julia
isEdge(x::Int, y::Int) =  x == 1 || y == 1 || x == 50 || y == 50

function walk(x::Int, y::Int, m::Array{Float64, 2}, c::Array{Float64, 2},
              getEdgeVal::Function,f::Function)::Float64
    randv = rand()
    res = f(x, y) + if isEdge(x, y)
        getEdgeVal(x, y)
    elseif randv < 1/4
        walk(x-1, y, m, c, getEdgeVal, f)
    elseif randv < 1/2
        walk(x+1, y, m, c, getEdgeVal, f)
    elseif randv < 3/4
        walk(x, y+1, m, c, getEdgeVal, f)
    else
        walk(x, y-1, m, c, getEdgeVal, f)
    end
    m[x, y] += res
    c[x, y] += 1
    res
end

function solve(n::Int, getEdgeVal::Function, f::Function)
    accum = zeros(50, 50)
    counter = zeros(50,50)
    for _ in 1:n
        for i in 1:50
            for j in 1:50
                walk(i, j, accum, counter, getEdgeVal, f)
            end
        end
    end
    accum ./ counter
end
```