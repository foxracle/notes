Python性能分析

Python的高效体现在它的开发效率，完善的类库支持，特别是这几年在数据科学中的流行，使得Python开始在各种语言排行榜独占鳌头，但是Python代码运行效率低这一点也一直被诟病，要想让Python高效运行，掌握一套性能分析的方法和工具，就显得很重要。

一般通过profiling找到代码瓶颈，然后针对这个瓶颈代码进行优化，从而可以到达事半功倍的效果，即用最少的工作量来获得最大的性能的提升，这在任何场景下都是最优的选择。任何可衡量的资源都可以进行profiling，除了CPU，内存还包括网络带宽，磁盘I/O。

profiling的目的就是通过对系统进行分析，找出哪里比较慢，哪里消耗内存比较多，哪里会引起更多的磁盘I/O或者网络I/O。但是profiling通常会增加代码的额外开销，这种开销有时候是非常大，会10x甚至100x的降低代码运行效率。所以profiling一般不能直接针对线上的代码进行，一般是构建一套类线上环境，或者把需要做profiling的代码拿出来单独进行分析。

Python常用的profiling方法：

## Timing

最基础的就是使用time.time()来计时，这个方法简单有效，也许所有写过Python代码的人都用过。
```
import time
...
start_time = time.time()
output = foo(a, b, c)
end_time = time.time()
secs = end_time - start_time
print foo.func_name + " took", secs, "seconds"
```

我们可以创建一个decorator使它用起来更方便。
```
import time
from functools import wraps
...
def simple_profiling(fn):
    @wraps(fn)  # 对外暴露调用装饰器函数的函数名和docstring
    def wrapped(*args, **kwargs):
        t1 = time.time()
        result = fn(*args, **kwargs)
        t2 = time.time()
        print (
            "@simple_profiling:" + fn.func_name + " took " + str(t2 - t1) + " seconds")
        return result
    return wrapped
...
@simple_profiling
def foo(a, b, c)
    ...
```
这个方法的优点是简单，额外开效非常低（大部分情况下可以忽略不计）。但缺点也很明显，除了总用时，没有任何其他信息。

Python的timeit模块也提供了测量小段代码执行时间的方法。需要注意在使用timeit模块时，GC会被临时关闭，所以这个可能导致测试结果跟真实运行结果有差距。
```
python -m timeit [-n N] [-r N] [-s S] [-t] [-c] [-h] [statement ...]
-n : 执行指定语句的次数
-r : 重复测量的次数（默认3次）
-s : 指定初始化代码或构建环境的导入语句
-t : 使用time.time() (Windows平台以外的默认值)
-c : 使用time.clock() (Windows平台默认值)
```
举一个-s的例子，假设我们在test.py里面定义了一个函数foo
```
python -m timeit -n 5 -r 5 -s "import test" "test.foo(a,b,c)"
5 loops, best of 5: 28.6 usec per loop
```

timtie还可以在IPython环境下使用：
```
In [6]: timeit '"-".join(map(str, xrange(100)))'
100000000 loops, best of 3: 9.75 ns per loop
```

另外unix系统也提供time工具，用来统计脚本执行的时间，它的特点是把运行时间分成real,user,sys三部分。
```
$ time python test_time.py

real    0m0.069s
user    0m0.038s
sys     0m0.033s
```


## cProfile

cProfile是Python标准库中的一个模块，它可以非常仔细地分析代码执行过程中所有函数调用的用时和次数。cProfile最简单的用法是用cProfile.run来执行一段代码，或是用python -m cProfile myscript.py来执行一个脚本。例如
```
$ python -m cProfile -s cumulative -o profile.stats test_time.py
         5 function calls in 0.232 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.232    0.232 test_time.py:3(<module>)
        1    0.018    0.018    0.231    0.231 test_time.py:3(foo)
        1    0.177    0.177    0.177    0.177 {map}
        1    0.036    0.036    0.036    0.036 {method 'join' of 'str' objects}
        1    0.000    0.000    0.000    0.000 {method 'disable' of '_lsprof.Profiler' objects}
```

整个输出结果给出了每一个函数的耗时信息。每个列的字段含义如下：

- ncalls:  函数被调用次数
- tottime: 函数总耗时，子函数的执行时间不计算在内
- percall: tottime / ncalls
- cumtime: 函数加上其所有子函数的总耗时
- percall: cumtime / ncalls

这个例子比较简单，所有输出很少，真实的例子估计输出会非常多，我们可以把结果通过 -o profile.stats 把结果保存到文件，然后通过强大的pstats模块来进行分析。关于pstats的具体使用可以看官方文档。
关于使用cProfile进行性能分析时，推荐[profilehooks](https://mg.pov.lt/profilehooks/)，使用方法如下：
```
from profilehooks import profile

class SampleClass(ParentClass):
    @profile(filename="/tmp/SampleClass_do_something.stats", immediate=True, stdout=False)
    def do_something(self):
        ...
```

这里把输出结果不输出到stdout而是直接保存到/tmp/SampleClass_do_something.stats，因为是对长期运行的代码进行profiling，所以这里把immediate设置成True，表示代码执行完立即输出，而不是等程序结束。profile还有很多其他参数，详细请看github上源码注释[profilehooks.py](https://raw.githubusercontent.com/mgedmin/profilehooks/master/profilehooks.py)

如果觉得pstats使用不方便，还可以使用一些图形化工具，比如[gprof2dot](https://github.com/jrfonseca/gprof2dot)和[RunSnakeRun](https://www.jianshu.com/p/26ccb05bad9e)来可视化分析cProfile的诊断结果。这两个工具推荐一起使用，RunSnakeRun能很快发现那些函数执行时间占比比较大，然后通过gprof2dot画的函数调用图来具体分析。

##Line Profiler

我们通过cProfile定位到了具体的耗时函数，下面就需要具体定位瓶颈出在哪行代码，这个时候就到了Line Profiler出场的时候了。与cProfile相比，Line Profiler的结果更加直观，它可以告诉你一个函数中每一行的耗时。Line Profiler并不在标准库中，需要用pip来安装。
```
pip install line_profiler
```
line_profiler的使用特别简单，在需要监控的函数前面加上@profile装饰器。然后用它提供的 kernprof -l -v source_code.py 进行诊断。
