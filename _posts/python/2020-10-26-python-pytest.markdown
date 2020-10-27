---
layout: post
title:  "Python pytest"
date:   2020-10-26 21:30:30 +0800
categories: python
---

* TOC
{:toc}

# 简单使用

```bash
# 安装 pytest
pip install pytest 

# 运行当前目录的所有测试文件 (文件名中包含 test 的文件)
pytest

# 运行指定测试文件中的所有测试用例 (测试用例是指函数名中包含 test 的函数)
pytest max_min_test.py

# 运行指定测试文件中的指定测试用例
pytest max_min_test.py::test_min
```

在测试代码中使用 assert 进行断言:

```py
#!/usr/bin/env python3

import algo


def test_min():
    values = (2, 3, 1, 4, 6)
    value = algo.min(values)
    assert value == 1


def test_max():
    values = (2, 3, 1, 4, 6)
    value = algo.max(values)
    assert value == 6
```

# 跳过用例

使用 @pytest.mark.skip 来跳过指定用例:

```py
#!/usr/bin/env python3
import pytest

import algo


@pytest.mark.skip
def test_min():
    values = (2, 3, 1, 4, 6)
    value = algo.min(values)
    assert value == 1


def test_max():
    values = (2, 3, 1, 4, 6)
    value = algo.max(values)
    assert value == 6
```

# 将测试用例归组

使用 @pytest.mark 来将测试用例归组，随后使用 pytest -m xxx 来执行指定归组的测试用例：

```py
#!/usr/bin/env python3

# pytest -m a marking.py
# pytest -m b marking.py

import pytest


@pytest.mark.a
def test_a1():
    assert (1) == (1)


@pytest.mark.a
def test_a2():
    assert (1, 2) == (1, 2)


@pytest.mark.a
def test_a3():
    assert (1, 2, 3) == (1, 2, 3)


@pytest.mark.b
def test_b1():
    assert "falcon" == "fal" + "con"


@pytest.mark.b
def test_b2():
    assert "falcon" == f"fal{'con'}"
```

# 参数化测试

使用 @pytest.mark.parameterize 可以进行参数化测试：

```py
#!/usr/bin/env python3
import pytest

import algo


@pytest.mark.parametrize("data, expected", [((2, 3, 1, 4, 6), 1),
    ((5, -2, 0, 9, 12), -2), ((200, 100, 0, 300, 400), 0)])
def test_min(data, expected):
    value = algo.min(data)
    assert value == expected


@pytest.mark.parametrize("data, expected", [((2, 3, 1, 4, 6), 6),
    ((5, -2, 0, 9, 12), 12), ((200, 100, 0, 300, 400), 400)])
def test_max(data, expected):
    value = algo.max(data)
    assert value == expected
```


# Fixtures

fixture 可以为测试用例准备一些对象。比如你有好几个用例都用到了同样的数据，就可以把这些数据放到 fixture 里面。

以下是要测试的选择排序函数：

```py
def sel_sort(data):
    if not isinstance(data, list):
        values = list(data)
    else:
        values = data

    size = len(values)
    for i in range(0, size):
        for j in range(i + 1, size):
            if values[j] < values[i]:
                _min = values[j]
                values[j] = values[i]
                values[i] = _min
    return values
```

排序函数可能有好多种，为此我们准备一个 fixture, 这样它可以用在多个排序函数的测试用例里。

```py
@pytest.fixture
def data():
    return [3, 2, 1, 5, -3, 2, 0, -2, 11, 9]


# 在参数中使用 data 来引用之前定义好的 fixture
def test_sel_sort(data):
    sorted_values = algo.sel_sort(data)
    assert sorted_values == sorted(data)
```