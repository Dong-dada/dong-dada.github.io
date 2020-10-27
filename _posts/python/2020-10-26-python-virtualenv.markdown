---
layout: post
title:  "Python virtualenv"
date:   2020-10-26 21:00:30 +0800
categories: python
---

* TOC
{:toc}

virtualenv 用于创建、管理独立的 python 运行环境。简单来说就是把 python 代码所依赖的各种包都放到一个单独的文件夹里。

# 概括

```bash

# 安装 virtualenv
pip3 intall virtualenv

# 创建名为 demo-env 的虚拟环境
virtualenv demo-env

# 激活 demo-env 虚拟环境
source demo-env/bin/activate

# 在虚拟环境中安装三方库
pip install pytest

# 在虚拟环境中执行 python 脚本 
python demo.py

# 退出虚拟环境
deactivate

# 验证当前是否在虚拟环境中，检查是否有打印 (demo-env)
pip --version

# 删除虚拟环境
rm -rf demo-env
```

# 步骤

使用 `pip3 intall virtualenv` 安装 virtualenv 组件。

进入项目目录。执行以下命令，将创建一个名为 demo-env 的 python 环境。

```bash
$virtualenv demo-env
```

上述命令执行结束后，可以看到在项目目录下多出来一个 demo-env 文件夹，其中就包含了我们项目可能依赖的各种 python 库。

执行以下命令，将该环境激活：

```bash
source demo-env/bin/activate
```

执行之后命令提示符之前会多一个 (demo-venv)，表示已经进入了 demo-env 独立环境。

这时候再去执行 pip install pytest, 可以看到 pytest 这个三方库被安装到了 demo-env 文件夹下：

```bash
(demo-env)  dongyu@dada-mac ~/Documents/data/code/python/virutalenv-demo$ tree -L 3
.
└── demo-env
    ├── bin
    │   ├── activate
    │   ├── activate.csh
    │   ├── activate.fish
    │   ├── activate.ps1
    │   ├── activate.xsh
    │   ├── activate_this.py
    │   ├── easy_install
    │   ├── easy_install-3.8
    │   ├── easy_install3
    │   ├── easy_install3.8
    │   ├── pip
    │   ├── pip-3.8
    │   ├── pip3
    │   ├── pip3.8
    │   ├── py.test
    │   ├── pytest
    │   ├── python -> /usr/local/opt/python@3.8/bin/python3.8
    │   ├── python3 -> python
    │   ├── python3.8 -> python
    │   ├── wheel
    │   ├── wheel-3.8
    │   ├── wheel3
    │   └── wheel3.8
    ├── lib
    │   └── python3.8
    └── pyvenv.cfg

4 directories, 24 files
```

使用 deactivate 命令可以退出当前环境：

```
(demo-env)  dongyu@dada-mac ~/Documents/data/code/python/virutalenv-demo$ deactivate
 dongyu@dada-mac ~/Documents/data/code/python/virutalenv-demo$
```
