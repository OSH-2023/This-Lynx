# shuffle_in_vega


> 本笔记主要用于理清散落在项目各处的shuffle代码逻辑

## 变量名
partition:分区号，map端RDD的分区号即input_id，reduce端RDD的分区号即reduce_id
split:reduce_id
shuffle_id:shuffle任务编号


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
其本质属性主要就是shuffle_id

RDD主要功能在于compute函数
shuffle_rdd主要功能是用shuffle_fetcher获取shuffle文件读出的反序列化结果(k,v对来的)，然后将每种k对应的值都拼接进一个数组里

（要是这个k,v对就是map_id和reduce_id的映射关系呢？）

## /rdd/co_grouped_rdd

## dependency
shuffle写文件的逻辑都在这里！

**do_shuffle_task()**:
1. 获取分区号partition，分区数n=num_output_splits，reduce_id(split)，空桶buckets
2. 得到iter的(K,V)对
3. 把iter里面的所有(K,V)对，按照K的hash值，将V分配到n个HashMap里面，最后插进内存(self.shuffle_id, partition, i)处(i是桶编号)



## map_output_tracker




## SHUFFLE_CACHE
SHUFFLE_CACHE在`env.rs`里定义，是一个全局的Dashmap，索引`(shuffle_id, partition, i)=(shuffle_id, input_id, reduce_id)`的位置用于存储`hash(K)=i=reduce_id`时的所有`(K,V)`对，其中`i=reduce_id`是桶编号，`partition=input_id`是分区号，`shuffle_id`是shuffle任务编号


## 总结









## 附录
测试代码格式：
```bash
cargo test --package vega --lib -- shuffle::shuffle_fetcher::tests::fetch_ok --exact --nocapture 
```


















