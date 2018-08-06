RunSnakeRun on Mac 

在做Python性能分析时，需要对cProfile的输出结果进行分析，如果有一个好用的图形分析工具会更加直观，[gprof2dot](https://github.com/jrfonseca/gprof2dot)和[RunSnakeRun](http://www.vrplumber.com/programming/runsnakerun/)就是这样的两个工具。gprof2dot安装比较简单，只要确保系统安装了graphviz就可以出图。但是RunSnakeRun在Mac上就没那么容易跑起来，这里记录一下碰到的坑(如果完全不用brew和virtualenv的话，会简单很多)。

环境：

- macOS 10.13.6
- pyenv 1.2.6 installed by brew
- pyenv-virtualenv 1.1.3 installed by brew
- python 2.7.15 installed by pyenv
- wxPython 3.0.2.0_1 installed by brew
- 虚拟环境python_2.7 using python 2.7.15


主要几个问题：

- [wxPython](https://www.wxpython.org/)有两个版本，一个是被称为[classic版本](https://sourceforge.net/projects/wxpython/files/wxPython/)，被托管在SourceForge上，新的4.x版本是新的，可以直接用pip安装。runSnakerun依赖的是classic版本，Mac上用brew安装的版本就是classic版本，版本号是3.0.2.0_1
- 因为wxPython是通过brew安装的，虚拟环境python_2.7中安装的RunSnakeRun无法访问wxPython，需要把把wxPython和虚拟环境python_2.7链接在一起，否则会提示如下错误。
```
ImportError: No module named wx
```
- 虚拟环境python_2.7安装的Python不是Framework build的Python，无法启动GUI程序，提示下面的错误。这就需要用系统的Python来启动RunSnakeRun，同时把PYTHONHOME设置成虚拟环境的Python
```
This program needs access to the screen.
Please run with a Framework build of python, and only when you are
logged in on the main display of your Mac.
```
- 一直提示这个错，没找到解决方案，但是不影响使用
```
UserWarning: wxPython/wxWidgets release number mismatch
  warnings.warn("wxPython/wxWidgets release number mismatch")
```


###安装软件
```
pip install SquareMap RunSnakeRun
brew install wxPython wxmac
```

###链接wxPython和虚拟环境python_2.7

- 确认wxPython安装路径
```
/usr/local/Cellar/wxpython/3.0.2.0_1/lib/python2.7/site-packages/wx-3.0-osx_cocoa/wx
```
- 在虚拟环境python_2.7的下面建一个软链到wxPython的安装路径
```
cd ~/.pyenv/versions/2.7.15/envs/python_2.7/lib/python2.7/site-packages/
ln -s /usr/local/Cellar/wxpython/3.0.2.0_1/lib/python2.7/site-packages/wx-3.0-osx_cocoa/wx wx
```

###用系统Python启动runsnake
```
#!/bin/bash -ex

WXPYTHON_APP="runsnakerun/runsnake.py"
PYVER=2.7

if [ -z "$VIRTUAL_ENV" ] ; then
    echo "You must activate your virtualenv: set '$VIRTUAL_ENV'"
    exit 1
fi

BREW_PYTHON_ROOT="$(brew --prefix)/Cellar/python@2/2.7.15_1/Frameworks/Python.framework/Versions/$PYVER"

PYTHON_BINARY="bin/python$PYVER"
FRAMEWORK_PYTHON="$BREW_PYTHON_ROOT/$PYTHON_BINARY"

VENV_SITE_PACKAGES="$VIRTUAL_ENV/lib/python$PYVER/site-packages"

# Use the Framework Python to run the app
export PYTHONHOME=$VIRTUAL_ENV
exec "$FRAMEWORK_PYTHON" "$VENV_SITE_PACKAGES/$WXPYTHON_APP" $*
```

参考：
- https://wiki.wxpython.org/wxPythonVirtualenvOnMac
- https://www.georgevreilly.com/blog/2015/09/20/RunSnakeRun-WxPython-Brew-Virtualenv.html
