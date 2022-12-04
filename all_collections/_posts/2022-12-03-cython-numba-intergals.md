---
layout: post
title: Fast Numerical Integration with Numba and Cython
author: Matthew Murray
categories: [Python, Numba, Cython, Numerical Integration, Numerical Compilers]
---

### Introduction
I've always been excited about doing complex integrals like this one:

$$
\int_0^{\frac{\pi}{2}} \arccos \left(\frac{\cos x}{1+2 \cos x}\right) d x = \frac{5\pi^2}{24}
$$

Although doing integrals like this is fun, it becomes apparent quickly that many integrals that are impossible to compute by hand. This is where numerical integration comes into play! Numerical integration allows us to appoximate the value of a definite integral. In general, numerical integration algorithms work by summing up the products of weights and nodes. By choosing the weights and nodes carefully, we can get very precise approximations of complicated integrals. 

$$
\int_{-0.15}^1 \int_{0.13}^{0.8} \int_{-1}^1 \int_0^1 f\left(x_0, x_1, x_2, x_3\right) d x_0 d x_1 d x_2 d x_3
$$

$$
f\left(x_0, x_1, x_2, x_3\right)= \begin{cases}x_0^2+x_1 x_2-x_3^3+\sin x_0+1 & \left(x_0-0.2 x_3-0.5-0.25 x_1>0\right) \\ x_0^2+x_1 x_2-x_3^3+\sin x_0+0 & \left(x_0-0.2 x_3-0.5-0.25 x_1 \leq 0\right)\end{cases}
$$

### Numba

```python
import numpy as np
from scipy import integrate
from numba import jit

@jit(nopython=True)
def func2(x0, x1, x2, t0, t1):
    return x0*x2**2 + np.sin(x1) + 1 + (1 if x0+t1*x1-t0>0 else 0)

def lim0(x1, x2, t0, t1):
    return [t0*x1 + t1*x2 - 1, t0*x1 + t1*x2 + 1]

def lim1(x2, t0, t1):
    return [x2 + t0**2*t1**3 - 1, x2 + t0**2*t1**3 + 1]

def lim2(t0, t1):
    return [t0 + t1 - 1, t0 + t1 + 1]

def opts0(x1, x2, t0, t1):
    return {'points' : [t0 - t1*x1]}

def opts1(x2, t0, t1):
    return {}

def opts2(t0, t1):
    return {}
```

### Cython

```cython
from scipy import integrate

cdef extern from "math.h":
    double sin(double)

cdef double func2(double x0, double x1, double x2, int t0, int t1):
    return x0*x2**2 + sin(x1) + 1 + (1 if x0+t1*x1-t0>0 else 0)

def lim0(double x1, double x2, int t0, int t1):
    return [t0*x1 + t1*x2 - 1, t0*x1 + t1*x2 + 1]

def lim1(double x2, int t0, int t1):
    return [x2 + t0**2*t1**3 - 1, x2 + t0**2*t1**3 + 1]

def lim2(int t0, int t1):
    return [t0 + t1 - 1, t0 + t1 + 1]

def opts0(double x1, double x2, int t0, int t1):
    return {'points' : [t0 - t1*x1]}

def opts1(double x2, int t0, int t1):
    return {}

def opts2(int t0, int t1):
    return {}
```