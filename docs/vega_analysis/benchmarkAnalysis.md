## perf CPU性能分析工具
下载
`apt install linux-tools-common`

直接使用
```bash
sudo perf stat ./target/debug/vega
```
> 备注:如果显示Workload failed: Permission denied，需要检查权限问题，使用`chmod -R 777 target_file`即可，前面sudo必须要有
输出结果
```bash{.line-numbers}
"/root"
get lock for scheduler
scheduler get job_id complete！
result: [[], [0], [0, 1], [0, 1, 2], [0, 1, 2, 3], [0, 1, 2, 3, 4], [0, 1, 2, 3, 4, 5], [0, 1, 2, 3, 4, 5, 6], [0, 1, 2, 3, 4, 5, 6, 7], [0, 1, 2, 3, 4, 5, 6, 7, 8]]

 Performance counter stats for './target/debug/vega':

            391.03 msec task-clock                #    0.213 CPUs utilized          
               942      context-switches          #    2.409 K/sec                  
                13      cpu-migrations            #   33.246 /sec                   
               748      page-faults               #    1.913 K/sec                  
       630,725,775      cycles                    #    1.613 GHz                      (47.29%)
                 0      stalled-cycles-frontend                                       (51.38%)
                 0      stalled-cycles-backend    #    0.00% backend cycles idle      (54.35%)
                 0      instructions              #    0.00  insn per cycle           (54.94%)
                 0      branches                  #    0.000 /sec                     (50.76%)
                 0      branch-misses             #    0.00% of all branches          (48.07%)

       1.834332719 seconds time elapsed

       0.127230000 seconds user
       0.269728000 seconds sys
```
可以发现使用了1.834的time elapsed但是task-clock仅仅使用了391.03ms,CPU利用率只有0.213，证明大部分时间被占用的任务并不是CPU使用密集型的，而是IO密集型的。
## 火焰图分析使用
``` bash
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph
sudo perf record --call-graph=dwarf mytest  #将这里的mytest替换成目标程序位置,或者注释掉这行，将在vega目录下产生的perf.data文件拷贝到当前目录下
sudo perf script | ./stackcollapse-perf.pl > out.perf-folded
./flamegraph.pl out.perf-folded > perf.svg
```

## 火焰图分析
火焰图的纵轴代表了函数调用栈，横轴代表了占用CPU资源的比例，跨度越大代表占用的CPU资源越多，从火焰图中我们可以更直观的看到程序中CPU资源的占用情况以及函数的调用关系。

-----
|测试内容|数据参数及规模|运行结果|运行环境|备注|
|----|----|-----|---|---|
|pi|N=100,000,000,num_slices=2|result: 3.14178564 196.554028501s|yzx ubuntu release|使用新版iterator()试图减少拷贝次数|
|pi|N=100,000,000,num_slices=2|result: 3.14138532 118.899680573s|yzx ubuntu release|使用新版iterator()试图减少拷贝次数|
|pi|N=100,000,000,num_slices=2|result: 3.14136944 123.480603307s|yzx ubuntu release|使用新版iterator()试图减少拷贝次数|
|pi|N=100,000,000,num_slices=2|result: 3.14156692 150.976415653s|yzx ubuntu release|原版iterator()产生两次拷贝|
|pi|N=100,000,000,num_slices=2|result: 3.14170632 119.148737699s|yzx ubuntu release|原版iterator()产生两次拷贝|
|pi|N=100,000,000,num_slices=2|result: 3.14166852 114.434112669s|yzx ubuntu release|原版iterator()产生两次拷贝|
|pi|N=100,000,num_slices=2|result: 3.13 39.911636275s|yzx ubuntu release|原版iterator()产生两次拷贝|
|wordcount|100MB|用时52.2066s|yzx ubuntu pyspark|使用timeit进行计时|
|wordcount|100MB|用时52.4881s|yzx ubuntu pyspark|使用timeit进行计时|
|wordcount|100MB|用时26.5643s|yzx ubuntu pyspark|使用timeit进行计时，连续运行，使用缓存|
|wordcount|100MB|用时30.7437s|yzx ubuntu vega release|使用perf进行分析|
|wordcount|100MB|用时10.7568s|yzx ubuntu vega release|使用内置计时器|
|wordcount|100MB|用时9.72145s|yzx ubuntu vega release|使用内置计时器|
|wordcount|100MB|用时8.69915s|yzx ubuntu vega release|使用内置计时器|


