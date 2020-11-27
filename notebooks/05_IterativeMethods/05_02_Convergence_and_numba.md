---
jupytext:
  formats: ipynb,md:myst
  notebook_metadata_filter: toc
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.12
    jupytext_version: 1.6.0
kernelspec:
  display_name: Python 3
  language: python
  name: python3
toc:
  base_numbering: 1
  nav_menu: {}
  number_sections: true
  sideBar: true
  skip_h1_title: true
  title_cell: Table of Contents
  title_sidebar: Contents
  toc_cell: true
  toc_position: {}
  toc_section_display: true
  toc_window_display: false
---

<div class="copyright" property="vk:rights">&copy;
  <span property="vk:dateCopyrighted">2020</span>
  <span property="vk:publisher">B. Knaepen & Y. Velizhanina</span>
</div>

+++

# Boosting Python

+++ {"toc": true}

<h1>Table of Contents<span class="tocSkip"></span></h1>
<div class="toc"><ul class="toc-item"><li><span><a href="#Introduction" data-toc-modified-id="Introduction-1"><span class="toc-item-num">1&nbsp;&nbsp;</span>Introduction</a></span></li><li><span><a href="#Gauss-Seidel-with-numba" data-toc-modified-id="Gauss-Seidel-with-numba-2"><span class="toc-item-num">2&nbsp;&nbsp;</span>Gauss-Seidel with numba</a></span></li></ul></div>

+++

## Introduction

+++

Python has plenty of appeal to the programming community: it's simple, interactive and free. But Fortran, C, C++ dominate high-performance programming. Why? Python is *slow*. There are two major reasons for that: **Python is a dynamically typed language** and **Python is an interpreted language**.

It doesn't make Python *a bad* programming language. On the contrary, Python is a great tool for various tasks that do not require running expensive simulations (web-development, scripting). Python also dominates the data-science due to availability of such packages as NumPy and SciPy.

As we discussed earlier, NumPy and SciPy integrate optimized and precompiled C code into Python and, therefore, might provide significant speed up. Though, there are serious limitations to the optimization that can be done by using NumPy and SciPy. We have already encountered situations when it was not possible to avoid running Python loops that easily end up being painfully slow.

In this notebook we are going to discuss tools designed to provide C-like performance to the Python code: *Cython* and *Numba*. What are they, what are they good for and why people still use C/C++ with much more complicated syntax?

+++

## Interpreters VS compilers

+++

Before we can precisely understand what are Cython and Numba, let's discuss the basics of what are exactly the compilers and interpreters.

The common mistake when understanding compiler is to think that the output of the compiler is necessarily the [*machine code*][3] - code written in a low-level programming language that is used to directly manipulate the central processing unit (CPU). In fact, *compiler* is simply a program that *translates* a source code written in certain programming language into any other programming language (normally lower-lever). *Interpreter* is also a computer program. It executes the source code without it been translated into the machine code. *But it doesn't make compilation and interpretation mutually exclusive:*

> [...most interpreting systems also perform some translation work, just like compilers][4].

The relevant example is that of CPython. CPython is the default implementation of Python and is a [bytecode interpreter][5]. At the intermediate stage it compiles the source code into the lower-level so-called bytecode, or p-code (portable code). Bytecode is universal to all platforms, or platform-independent. There is no further compilation of the bytecode into the machine code. Bytecode is executed *at the runtime* by so-called Python Virtual Machine (PVM). PVM is a program that is part of CPython. When virtual machines, or p-code machines, are explained, it is often said that the p-code can be viewed as a machine code of the *hypothetical* CPU. The virtual machines, unlike real CPUs, are implemented as part of interpreter program not in the hardware of the platform.

Whenever you hear that "interpreters are slower than compilers", you must have in mind that it is not just a generic translator program that is meant by compiler in that context. CPython performs compilation itself, as we know. The compilers meant in the present context are those programs that translate the source code into the machine code. Why are interpreters slower? While the fact that the p-code is *interpreted* and *executed* by the interpreters *at the runtime* is partially the answer, we still have to understand what are the compiler systems do.

There are two distinct types of compilers: *ahead-of-time* (AOT) compilers and *just-in-time* (JIT) compilers.

> [AOT compilation][6] is the act of compiling a higher-level programming language ... into a native (system-dependent) machine code so that the resulting binary file can execute natively.

In this way, the code executed at the runtime produces behaviour that has been predefined at the compile time. While the interpreter must both define and produce desired behaviour at the runtime performing statement-by-statement analysis. Note though that

> [It generally takes longer][4] to run a program under an interpreter than to run the compiled code but it can take less time to interpret it than the total time required to compile and run it.

What about the JIT compilers then? First of all,

> [JIT compilation][7] ... is a way of executing computer code that involves compilation during execution of a program – at runtime – rather than before execution.

Following the logic used to explain speed gain for AOT compilers versus interpreters, you might wonder: how then the JIT compiler can be faster if it as well defines the desired behaviour at the runtime? There is good example that could explain the difference in performance between typical interpreter and JIT compiler. Suppose the source code contains a loop that has to be executed $n$ times. Interpreter will have to analyze the bytecode statement-by-statement at each iteration. Well-implemented JIT compiler will produce the translation into the machine code, which is the most computationally expensive operation, only once.

How the JIT compilers compare to the AOT compilers is a somewhat more sophisticated discussion that we won't enter but you are free to investigate this question on your own.  

**Cython** is a *programming language* itself. It aims to run at C-like speed using Python-like syntax. Normally any Python code can be "translated" into Cython, which, if well designed, may speed your code up by the factor of order(s) of magnitude.

**Numba** is a so-called *just-in-time compiler*. To be more precise, it is the just-in-time (JIT) compiler, meaning that it does *not* require compilation prior to execution but compiles the source code at the run time.

You should not confuse Cython with CPython. Python as you know it is itself implemented in other programming languages. CPython is a default implementation of Python written in C. Other popular implementations of Python are [Jython][8], [PyPy][9]. CPython is also sometimes called an *interpreter*. Note that 

[3]: <https://en.wikipedia.org/wiki/Machine_code> "Machine code"
[4]: <https://en.wikipedia.org/wiki/Interpreter_(computing)> "Interpreter"
[5]: <https://en.wikipedia.org/wiki/Interpreter_(computing)#Bytecode_interpreters> "Bytecode interpreters"
[6]: <https://en.wikipedia.org/wiki/Ahead-of-time_compilation> "AOT compilation"
[7]: <https://en.wikipedia.org/wiki/Just-in-time_compilation> "JIT compilation"
[8]: <https://en.wikipedia.org/wiki/Jython> "Jython"
[9]: <https://en.wikipedia.org/wiki/PyPy> "PyPy"

+++

## Cython and Numba: why and how?

+++

### Python decorators

+++

So called wrapper functions are widely used in various programming languages. They take function or method as parameter extending it behaviour. Wrapper functions usually intend to *abstract* the code. They, therefore, might shorten and simplify it *by a lot*. Python decorator is unique to Python - it is basically a shortcut (syntactic sugar) for calling a wrapper function. Consider implementation of sample decorator function:

```{code-cell} ipython3
def decorator(func):
    def wrapper(*args, **kwargs):
        print('Preprocessing...')
        
        res = func(*args, **kwargs)
        
        print('Postprocessing...')
        
        return res
    return wrapper
```

*Note* that choice of names for `decorator` and `wrapper` is not restricted in any way in Python. Whatever you are allowed to name the regular Python function, you are also allowed to name the decorating and wrapping functions.

This decorator does nothing but simply prints something before and after the internal function is called. We propose you perceive it as an abstraction for some computations. Note that a decorator function returns *a function*. When decorating functions, you will ultimately want to normally return what the internal function returns *but* also perform certain computations before and after it executes.

If we proceed without using Python decorators, two more principal steps are required. First, we must implement the function we want to decorate. Let's go for something trivial:

```{code-cell} ipython3
def print_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')
```

Second, we have to perform actual decoration:

```{code-cell} ipython3
print_identity_and_more = decorator(print_identity)
```

`print_identity_and_more` is a function that accepts the same parameters as `print_identity` and prints certain string before and after it executes.

```{code-cell} ipython3
print_identity_and_more('Marichka', 42)
```

Python decorator decorates the function in a single step:

```{code-cell} ipython3
@decorator
def print_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')
```

We can simply call `print_identity` now:

```{code-cell} ipython3
print_identity('Mao', 33)
```

Let's consider slightly less trivial but a very useful example. Until now, whenever we needed to time execution of the code, we were using `time` or `timeit` magic. Magic commands is a nice tool but they are unique to IPython and usage of IPython is quite limited. The programmer is paying the price of lowered performance for the graphical interface. So, after all IPython is great for debugging, testing and visualization but in the optimized code you will have it disabled. Let's then implement a *decorator* that will be a timer for arbitrary function:

```{code-cell} ipython3
from timeit import default_timer

def timing(func):
    def wrapper(*args, **kwargs):
        t_init = default_timer()
        res = func(*args, **kwargs)
        t_fin = default_timer()
        
        print(f'Time elapsed: {t_fin-t_init} s')
        
        return res
    return wrapper
```

```{code-cell} ipython3
@timing
def print_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')
```

```{code-cell} ipython3
print_identity('Mark', 21)
```

It is possible to wrap the function in multiple decorators if necessary. Note that the outer decorator must go before the inner one:

```{code-cell} ipython3
@timing
@decorator
def print_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')
```

```{code-cell} ipython3
print_identity('Jacques', 9)
```

In the end of the day Python decorators are simply functions, so it is possible to pass parameters to the decorator. The syntax for that, though, is a bit different and requires additional layer of decoration called the *decorator factory*. Decorator factory is a decorator function that accepts certain parameters and returns the actual decorator. Consider the example where the timing is made optional using the decorator factory:

```{code-cell} ipython3
def timing(timing_on=True):
    def inner_timing(func):
        def wrapper(*args, **kwargs):
            if not timing_on:
                return func(*args, **kwargs)
            
            t_init = default_timer()
            res = func(*args, **kwargs)
            t_fin = default_timer()

            print(f'Time elapsed: {t_fin-t_init} s')

            return res
        
        return wrapper
    return inner_timing
```

We can now have timing enabled or disabled with the same decorator:

```{code-cell} ipython3
@timing()
def time_printing_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')

@timing(False)
def print_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')

time_printing_identity('Donald', 74)

print('\n')

print_identity('Joe', 78)
```

Implementation of the timing function is accessible in `modules/timers.py` by the name `dummy_timer`. You are free to use it as an alternative to `time` magic. Note that it does not implement the decorator factory, as in the above example, and does not provide functionality of the `timeit` magic that estimated the time averaged among $n$ runs. Consider example using `dummy_timer`:

```{code-cell} ipython3
import timers

@timers.dummy_timer
def print_identity(name, age):
    print(f'Your name is {name}. You are {age} year(s) old.')

print_identity('Jacob', 65)
```

## Gauss-Seidel with numba

```{code-cell} ipython3
from numba import njit
```

```{code-cell} ipython3
# Grid parameters.
nx = 61                  # number of points in the x direction
ny = 61                  # number of points in the y direction
xmin, xmax = 0.0, 1.0     # limits in the x direction
ymin, ymax = -0.5, 0.5    # limits in the y direction
lx = xmax - xmin          # domain length in the x direction
ly = ymax - ymin          # domain length in the y direction
dx = lx / (nx - 1)        # grid spacing in the x direction
dy = ly / (ny - 1)        # grid spacing in the y direction

# Create the gridline locations and the mesh grid;
# see notebook 02_02_Runge_Kutta for more details
x = np.linspace(xmin, xmax, nx)
y = np.linspace(ymin, ymax, ny)
X, Y = np.meshgrid(x, y, indexing ='ij')

# Compute the rhs
b = (np.sin(np.pi * X / lx) * np.cos(np.pi * Y / ly) +
     np.sin(5.0 * np.pi * X / lx) * np.cos(5.0 * np.pi * Y / ly))
```

```{code-cell} ipython3
def gauss_seidel(p, b, tolerance, max_iter):
    
    nx, ny = b.shape
    iter = 0
    diff = 1
    tol_hist_gs = []
    
    pnew = p.copy()
    
    while (diff > tolerance):
    
        if iter > max_iter:
            print('\nSolution did not converged within the maximum'
                ' number of iterations')
            break
        
        for i in range(1, nx-1):
            for j in range(1, ny-1):
                pnew[i, j] = ( 0.25*(pnew[i-1, j] + p[i+1, j] + pnew[i, j-1]
                                + p[i, j+1] - b[i, j]*dx**2 ))
        
        diff = l2_diff(pnew, p)
        tol_hist_gs.append(diff)

        # Show iteration progress (I would like to add iter but cannot do it)
        # Problems in numba here
        # print('\r', f'diff: {diff:5.2e}', end='')
        
        p = pnew.copy()
        iter += 1

    else:
        print('The solution converged after {:d} iterations'.format(iter))
        return p, tol_hist_gs

@njit
def gauss_seidel_numba(p, b, tolerance, max_iter):
    
    nx, ny = b.shape
    iter = 0
    diff = 1
    tol_hist_gs = []
    
    pnew = p.copy()
    
    while (diff > tolerance):
    
        if iter > max_iter:
            print('\nSolution did not converged within the maximum'
                ' number of iterations')
            break
        
        for i in range(1, nx-1):
            for j in range(1, ny-1):
                pnew[i, j] = ( 0.25*(pnew[i-1, j] + p[i+1, j] + pnew[i, j-1]
                                + p[i, j+1] - b[i, j]*dx**2 ))
        
        diff = l2_diff(pnew, p)
        tol_hist_gs.append(diff)

        # Show iteration progress (I would like to add iter but cannot do it)
        # Problems in numba here
        # print('\r', f'diff: {diff:5.2e}', end='')
        
        p = pnew.copy()
        iter += 1

    else:
        print('The solution converged after iterations', iter)
        return p, tol_hist_gs
```

```{code-cell} ipython3
p0 = np.zeros((nx,ny))
```

```{code-cell} ipython3
# 20 seconds on my computer (7 times)
%timeit p, tol_hist_gs = gauss_seidel(p0.copy(), b, tolerance = 1e-10, max_iter = 1e6)
```

```{code-cell} ipython3
# 80ms on my computer (70 times) !!!
%timeit p, tol_hist_gs = gauss_seidel_numba(p0.copy(), b, tolerance = 1e-10, max_iter = 1e6)
```

```{code-cell} ipython3
# Compute the exact solution (and p for the plot %%timeit does not return it)
p, tol_hist_gs = gauss_seidel_numba(p0.copy(), b, tolerance = 1e-10, max_iter = 1e6)
p_e = p_exact_2d(X, Y)
```

```{code-cell} ipython3
fig, (ax_1, ax_2, ax_3) = plt.subplots(1, 3, figsize=(16,5))
# We shall now use the
# matplotlib.pyplot.contourf function.
# As X and Y, we pass the mesh data.
#
# For more info
# https://matplotlib.org/3.1.1/api/_as_gen/matplotlib.pyplot.contourf.html
#
ax_1.contourf(X, Y, p, 20)
ax_2.contourf(X, Y, p_e, 20)

# plot along the line y=0:
jc = int(ly/(2*dy))
ax_3.plot(x, p_e[:,jc], '*', color='red', markevery=2, label=r'$p_e$')
ax_3.plot(x, p[:,jc], label=r'$pnew$')

# add some labels and titles
ax_1.set_xlabel(r'$x$')
ax_1.set_ylabel(r'$y$')
ax_1.set_title('Exact solution')

ax_2.set_xlabel(r'$x$')
ax_2.set_ylabel(r'$y$')
ax_2.set_title('Numerical solution')

ax_3.set_xlabel(r'$x$')
ax_3.set_ylabel(r'$p$')
ax_3.set_title(r'$p(x,0)$')

ax_3.legend();
```

```{code-cell} ipython3
from IPython.core.display import HTML
css_file = '../styles/notebookstyle.css'
HTML(open(css_file, 'r').read())
```

```{code-cell} ipython3

```