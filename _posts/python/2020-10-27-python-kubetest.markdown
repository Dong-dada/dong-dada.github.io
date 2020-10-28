---
layout: post
title:  "Python pytest kubetest"
date:   2020-10-27 14:50:30 +0800
categories: python
---

* TOC
{:toc}



# 简介

kubetest 是一个 pytest 插件，可以用来编写需要与 kubernetes 交互的测试用例。比如测试部署行为是否正确。

使用以下命令来安装 kubetest:

```bash
pip install kubetest
```

安装之后，就可以在命令行里使用以下参数：

```
pytest \
    [--kube-error-log-lines <COUNT>] \
    [--suppress-insecure-request] \
    [--kube-log-level <LEVEL>] \
    [--kube-context <CONTEXT>] \
    [--kube-config <PATH>] \
    [--kube-disable] \
    [--in-cluster]
```

这些参数的含义见下表：

| 参数 | 含义 |
| --- | --- |
| --kube-config=path | 指向一个配置文件，其中指定了连接 k8s 集群所需的配置。 |
| --kube-context=context | 指定 kubeconfig 中需要用到的上下文。 |
| --in-cluster | 使用 kubernetes 的 in-cluster 配置。 |
| --kube-log-level=KUBE_LOG_LEVEL | 设置 kubetest 日志级别，默认为 warning。设置为 info 级别将显示 kubetest 执行的动作，设置为 debug 级别将输出动作之后的 k8s 对象状态。 |
| --kube-error-log-lines=KUBE_ERROR_LOG_LINES | 指定测试失败时要从容器日志中打印的日志行数。默认为 50，0 表示不打印，-1 表示全部打印。 |
| --suppress-insecure-request=SUPPRESS_INSECURE_REQUEST | 屏蔽 InsecureRequestWarning 警告，如果要对未设置 HTTPS 的集群进行测试，可以指定这个参数 |


# Fixtures

kubetest 提供了多个 fixture 方便你编写 k8s 相关的测试用例，下面逐个介绍：

## kube fixture

这个 fixture 将返回一个 test client 来管理 k8s cluster。每次执行完毕后，kube fixture 会清理这个 test client 以及它使用到的各种资源。

test client 中提供了各种方法来访问 k8s cluster, 比如部署 deployment 到 cluster 上，或者获取 cluster 中部署的 pods.

以下是使用 kube fixture 的简单例子：

```py
def test_simple_deployment(kube):
    # 创建一个 deployment
    deployment = kube.load_deployment('./k8s_resources/nginx_deployment.yaml')
    deployment.create()

    # 等待 deployment 变为 ready 状态
    deployment.wait_until_ready(timeout=60)
    deployment.refresh()

    # 获取 deployment 完成之后的 pod 数量，检查其是否符合预期
    pods = deployment.get_pods()
    assert len(pods) == 1

    # 校验 pod 中的 container 数量是否符合预期
    pod = pods[0]
    pod.wait_until_ready(timeout=60)

    containers = pod.get_containers()
    assert len(containers) == 1
```

其中的 nginx_deployment.yaml 内容如下，注意我们定义的 pod 数量是 2 个，这样可以观察测试失败时的输出：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 8088
```

运行上述用例需要有一个 k8s 集群，我们可以在本机启动 minikube, 使用它来作为测试用的 k8s 集群。

```bash
# 启动 minikube, 其 kube config 保存在 ~/.kube/config 目录
minikube start

# 运行测试用例
pytest --kube-config=~/.kube/config kubetest_demo.py
```

由于测试用例里断言 pod 数量是 1 个，但 nginx_deployment.yaml 中定义的 pod 数量是 2 个，因此测试会失败，输出一下内容：

```text
============================== test session starts ==============================
platform darwin -- Python 3.8.2, pytest-6.1.1, py-1.9.0, pluggy-0.13.1
kubetest config file: ~/.kube/config
kubetest context: current context
rootdir: /Users/dongyu/Documents/data/code/python/pytest-demo
plugins: kubetest-0.8.1
collected 1 item

kubetest_demo.py F                                                        [100%]

=================================== FAILURES ====================================
____________________________ test_simple_deployment _____________________________

kube = <kubetest.client.TestClient object at 0x10afc7f10>

    def test_simple_deployment(kube):
        # 创建一个 deployment
        deployment = kube.load_deployment('./k8s_resources/nginx_deployment.yaml')
        deployment.create()

        # 等待 deployment 变为 ready 状态
        deployment.wait_until_ready(timeout=60)
        deployment.refresh()

        # 获取 deployment 完成之后的 pod 数量，检查其是否符合预期
        pods = deployment.get_pods()
>       assert len(pods) == 1
E       AssertionError: assert 2 == 1
E        +  where 2 = len([{'api_version': None,\n 'kind': None,\n 'metadata': {'annotations': None,\n              'cluster_name': None,\n         ...rt',\n            'reason': None,\n            'start_time': datetime.datetime(2020, 10, 27, 9, 4, 33, tzinfo=tzutc())}}])

kubetest_demo.py:14: AssertionError
------------------------------ Captured log setup -------------------------------
WARNING  kubetest:api_object.py:111 unknown version (None), falling back to preferred version
============================ short test summary info ============================
FAILED kubetest_demo.py::test_simple_deployment - AssertionError: assert 2 == 1
=============================== 1 failed in 4.64s ===============================
```

## kubeconfig fixture

返回已配置的 kube config 文件的名称。这个名称与运行测试用例时指定的 --kube-config 参数应该是一致的。

```py
def test_kube_config(kubeconfig):
    assert kubeconfig == '~/.kube/config'
```

运行测试用例：

```bash
$pytest --kube-config=~/.kube/config kubetest_demo.py::test_kube_config
============================= test session starts ==============================
platform darwin -- Python 3.8.2, pytest-6.1.1, py-1.9.0, pluggy-0.13.1
kubetest config file: ~/.kube/config
kubetest context: current context
rootdir: /Users/dongyu/Documents/data/code/python/pytest-demo
plugins: kubetest-0.8.1
collected 1 item

kubetest_demo.py .                                                       [100%]

============================== 1 passed in 0.01s ===============================
```

## clusterinfo fixture

clusterinfo 这个 fixture 中包含了测试用例连接的 cluster 的基本信息:

```py
def test_clusterinfo(kube, clusterinfo):
    # 通过 clusterinfo fixture 获取被测集群的信息

    assert clusterinfo.user == 'minikube'
    assert clusterinfo.context == 'minikube'
```


# Markers

marker 是 pytest 的工具，可以让你为测试用例设置一些元数据。kubetest 提供了一些 k8s 相关的 marker。

## applymanifest marker

applymanifest marker 可以让你根据 k8s yaml 部署资源到 cluster 上。类似于 k8s 的 `kubectl apply -f <file>` 命令。

```py
@pytest.mark.applymanifest('./k8s_resources/nginx_deployment.yaml')
def test_apply_manifest(kube):
    kube.wait_for_registered(timeout=60)

    pods = kube.get_pods()
    print(pods)
    assert len(pods) == 2
```

还有其他的一些 marker, 可以查看 [官方文档](https://kubetest.readthedocs.io/en/latest/markers.html)
