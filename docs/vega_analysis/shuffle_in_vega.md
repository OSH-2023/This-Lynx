# shuffle_in_vega


> 本笔记主要用于理清散落在项目各处的shuffle代码逻辑


## /shuffle

### 总结

> 注：`mod.rs`里面定义的是各shuffle_error类型


### /shuffle/shuffle_fetcher

主要逻辑为fetch函数，它获取shuffle与reduce的task_id
我们首先要理清这个关系：一个shuffle_id对应多个uri，一个uri对应多个index（意义是input_id）
其执行过程如下：
1. 我们知道，一个shuffle_id对应的任务分成多部分散布在各机器上执行，每台机器有URI，一台机器同时执行多个部分，按照shuffle_id获取其对应的URI得一数组，又按URI对数组下标进行分组，得到uri->[index1,index2,...]的（K,V）对。完成后，将这些（K,V）对装入队列。
2. 然后，该函数为每个服务器URI生成一个异步任务。每个异步任务都会从服务器队列中获取某URI指定的input_id（即原来的index元组），并令HTTP客户端从shuffle_uri/input_id/reduce_id获取数据，加进shuffle_chunks里面并返回
3. 合并所有异步任务结果

综上所述，该函数并行地从多个服务器上的shuffle文件中读入数据（数量为分区数*reduce任务数，路径为`shuffle_uri/input_id/reduce_id`），并返回反序列化后的结果数组迭代器

### /shuffle/shuffle_manager
里面有两个类：ShuffleManager和ShuffleService
#### ShuffleManager
在每台机子上都开一个，用于管理shuffle数据传输与文件存储
**new()**:包含以下工作：新建shuffle文件夹，获取服务器网络参数，以此新建ShuffleManager
**get_output_file()**:创建output文件并返回其路径`shuffle_dir/shuffle_id/input_id/output_id`（output_id会不会就是reduce_id）
**check_status()**:检查通信channel的状态



#### ShuffleService
从.../{shuffleid}/{inputid}/{reduceid}获取数据，经过一层cache来读取



### /shuffle/shuffle_map_task
整个类用于存储ShuffleMapTask的各项信息


## /rdd/shuffle_rdd




## dependency




## map_output_tracker












## 附录
测试代码格式：
```bash
cargo test --package vega --lib -- shuffle::shuffle_fetcher::tests::fetch_ok --exact --nocapture 
```


















