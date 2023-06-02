# Ray的下载及部署
官网:
https://docs.ray.io/en/latest/ray-overview/installation.html#building-ray-from-source


## 从Pypi下载:

`# Install Ray with support for the dashboard + cluster launcher`

`pip install -U "ray[default]"`

`pip install -U "ray[air]" # installs Ray + dependencies for Ray AI Runtime`

## 从docker hub中下载

https://hub.docker.com/r/rayproject/ray

The rayproject/ray images include Ray and all required dependencies. It comes with anaconda and various versions of Python.

目前测试未发现docker image是否指定python版本会对测试造成任何影响，但是在issues中发现python3.6应该有若干bug，推测3.7以后既可以正常兼容。

查看docker是否正常引入
`docker images`

运行docker
`docker run --shm-size=<shm-size> -t -i rayproject/ray`
shm-size 推荐使用512M或2G,可以自定义
可以去掉中间的--shm-size字段，这时使用默认空间划分。

## 直接拉取源码

`git clone git@github.com:ray-project/ray.git`

按照官网指南进行测试
`python -m pytest -v python/ray/tests/test_mini.py`


1. 如果报pytest不存在，需要`pip install pytest`
2. 如果报`ERROR: file or directory not found: python/ray/tests/test_mini.py`,这个是需要在git 仓库下根目录进行
3. 如果报`ImportError: cannot import name 'find_available_port' from 'ray._private.test_utils'`,需要进入 python/ray/tests/conftest.py:28 将find_available_port注释掉，实测可以正常通过PASS

# Ray test_bench
官网
https://docs.ray.io/en/latest/ray-air/benchmarks.html

## 先启动Ray服务
在命令行中
`ray start --head`

> 其他命令:
> `ray stop`
> `ray status`

## 查看仪表盘
127.0.0.1:8265

普罗米修斯下载(图形化工具，可选)
https://prometheus.io/download/

grafana下载
https://grafana.com/grafana/download

使用方法
https://www.anyscale.com/blog/monitoring-and-debugging-ray-workloads-ray-metrics


Kuberay使用example
https://docs.ray.io/en/latest/cluster/kubernetes/examples/ml-example.html#kuberay-ml-example

https://docs.ray.io/en/latest/cluster/kubernetes/examples/gpu-training-example.html#kuberay-gpu-training-example