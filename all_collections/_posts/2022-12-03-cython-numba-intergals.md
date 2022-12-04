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

Although doing integrals like this is fun, it quickly becomes apparent that many integrals that are impossible to compute by hand. This is where numerical integration comes into play! Numerical integration allows us to appoximate the value of a definite integral. In general, [numerical integration](https://en.wikipedia.org/wiki/Numerical_integration) (or numerical quadrature) algorithms work by summing up the products of weights and nodes. By choosing the weights and nodes carefully, we can get very precise approximations of complicated integrals. In this blog post, we're going to use two compiler libraries: Numba and Cython to speed up our numerical calculations.


### Problem
This is the triple integral we're going to calculate.

$$
\int_{t_0+t_1-1}^{t_0+t_1+1} \int_{x_2+t_0^2 t_1^3-1}^{x_2+t_0^2 t_1^3+1} \int_{t_0 x_1+t_1 x_2-1}^{t_0 x_1+t_1 x_2+1} f\left(x_0, x_1, x_2, t_0, t_1\right) d x_0 d x_1 d x_2
\approx 36.1$$

where 

$$
f\left(x_0, x_1, x_2, t_0, t_1\right)= \begin{cases}x_0 x_2^2+\sin x_1+2 & \left(x_0+t_1 x_1-t_0>0\right) \\ x_0 x_2^2+\sin x_1+1 & \left(x_0+t_1 x_1-t_0 \leq 0\right)\end{cases}
$$

It's from the Scipy [documentation](https://docs.scipy.org/doc/scipy/index.html) for [`scipy.integrate.nquad`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.integrate.nquad.html#scipy.integrate.nquad). The function `scipy.integrate.nquad` allows us to do integration over several variables. It wraps the SciPy function `scipy.integrate.quad` which does integration over one variable using an advanced numerical integration technique from the Fortran library [QUADPACK](https://en.wikipedia.org/wiki/QUADPACK).

### Python
Below is how we would calculate the integral without Numba or Cython.
```python
import numpy as np
from scipy import integrate

def func(x0, x1, x2, t0, t1):
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

I timed the integral and got 300 ms.

```python
%timeit integrate.nquad(func, [lim0, lim1, lim2], args=(0,1), opts=[opts0, opts1, opts2])
```

```python
300 ms ± 12.6 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

### Numba
Now let's do the intgral using Numba! Numba is a Just-In-Time (JIT) compiler for python. It works by compiling python code to optimized machine code at runtime using the LLVM compiler library. All we need to do is add the `numba.jit` decorator to the function we want to compile.

```python
import numpy as np
from scipy import integrate
from numba import jit

@jit(nopython=True)
def func(x0, x1, x2, t0, t1):
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

I timed the integral and got 122 ms. This is 2.5x speed up!

```python
%timeit integrate.nquad(func, [lim0, lim1, lim2], args=(0,1), opts=[opts0, opts1, opts2])
```

```python
122 ms ± 2.16 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
```

### Cython
Now let's use Cython! Cython is a compiler for python that makes writing C-extensions for Python easier. Cython code looks mostly like Python with some C/C++ typing. We can "cythonize" the python code by providing types for the function arguments. The C-types we're using are `double` and `int`. We're also sine function from the C standard library of trig functions for improved performance.

```cython
from scipy import integrate

cdef extern from "math.h":
    double sin(double)

cdef double func(double x0, double x1, double x2, int t0, int t1):
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

I timed the integral and got 95.8 ms. This is 3x speed over the python code.

```python
%timeit integrate.nquad(func, [lim0, lim1, lim2], args=(0,1), opts=[opts0, opts1, opts2])
```

```python
95.8 ms ± 2.34 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
```

### Conclusion
In both cases, we we're able to speed up the numerical integration. We got a slighly higher speed up using cython, but we only had to add one decorator to the function with Numba! Both Cython and Numba are intersting libraries for speed up numerical code, and I'm excited work with them more in the future!