---
layout: post
title:  "mac Catalina升级Python2.7 到3.7"
date:   2020-03-31 19:02:10 +0800
tag: Python
---

首先去[Python官网](https://www.python.org/downloads/)下载最新的Python安装包。

安装完成需要添加环境变量，编辑 `~/.bash_profile`

```bash
# Setting PATH for Python 3.7
 # The original version is saved in .bash_profile.pysave
 export PATH="/Library/Frameworks/Python.framework/Versions/3.7/bin:${PATH}"
 alias python="/Library/Frameworks/Python.framework/Versions/3.7/bin/python3.7" # 新增
 export PATH
```

编辑完成后，需要用`source ~/.bash_profile`使其生效。但是我发现重启Terminal后，刚刚配置的Python3.7并没有生效，Python还是之前的Python2.7。搜了一下发现，原来我的macOS已经用zsh，需要在 `~/.zshrc`文件的末尾添加如下的代码：

```bash
source ~/.bash_profile
```

Python 3.7中自带了pip， 但是名字是pip3，我们可以在`.bash_profile`中设置别名为pip，这样就可以像Python2.7中用pip一样使用了。

```bash
# Setting PATH for Python 3.7
 # The original version is saved in .bash_profile.pysave
 export PATH="/Library/Frameworks/Python.framework/Versions/3.7/bin:${PATH}"
 alias python="/Library/Frameworks/Python.framework/Versions/3.7/bin/python3.7" # 新增
 alias pip="/Library/Frameworks/Python.framework/Versions/3.7/bin/pip3.7" # 新增
 13 export PATH
```