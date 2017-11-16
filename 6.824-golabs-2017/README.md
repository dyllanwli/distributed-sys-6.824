### 序言： 了解source

mapreduce包提供一个简单的Map / Reduce库。应用程序通常应该调用Distributed()(master.go)。开始工作，但也可以Sequential()(master.go)。执行用于调试的顺序执行。

该代码执行以下工作:

+ 该应用程序提供了一些输入文件、映射函数、reduce函数和reduce tasks(nReduce)。
+ master通过这个原理创造，从RPC server开始 -- master rpc.go --直到它被注册，使用Register()/master.go
+ 当task availiable的时候，schedule()[schedule.go] 决定怎样分配这些任务，那些任务被fail掉
+ master将每个输入文件作为一个map task 并调用doMap()[common_map] 对每个map任务至少执行一次。它可以直接(Sequential())或将DoTask RPC发送给一个worker(worker.go)。
+ 每个调用doMap()的调用都读取适当的文件，调用该文件内容上的map函数，并将结果 键值对 写入nReduce中间文件。doMap()哈希每个键来选择中间文件，从而减少处理密钥的任务。所有map任务完成后，将会有nMap x nReduce文件。每个文件名都包含一个前缀、map任务号和reduce任务编号。如果有两个map任务和三个reduce任务，map任务将创建这六个中间文件:

每个worker需要读入其他worker的文件，以及输入文件。实际的开发使用分布式储存设备比如GFS来允许这个运行在不同的机器上。在这个实验中，我们将运行所有的worker在同一个机器，使用本地文件系统。

+ master 然后调用doReduce 在common_reduce.go 里面， 至少进行一次reduce task。 在doMap() 直接穿过一个worker。 doReduce 对于reduce task r 从每个map task中收集到r个 中间文件，然后调用reduce函数给文件的每个键，然后reduce task 给结果文件执行nReduce
+ master 调用mr.merge() master_splitmerge.go 将nReduce个文件合并在一起，文件由前一步产生
+ 最后master发送shutdown RPC给每个worker然后结束RPC server

- - -

note：

    Over the course of the following exercises, you will have to write/modify doMap, doReduce, and schedule yourself. These are located in common_map.go, common_reduce.go, and schedule.go respectively. You will also have to write the map and reduce functions in ../main/wc.go.

- - -

### part 1 ： map/reduce 输入和输出

map/reduce 的实现缺少了几个模块，在写你自己的map/reduce函数之前，需要将sequential implementation实现。特别的，代码将去除两个重要的模块。

+ 一个进行分配output map task的函数 
+ 一个能够收集进行reduce task 的函数

这些task将由doMap() 执行在common_map.go ， 和doReduce() 在common_reduce.go 中。 在文件中有注释。

为了帮助你决定你是否能够正确的实现doMap() 和doReduce() 我们提供了GO test suite 用啦检查这些实现。 在test_test.go 文件汇总，运行test 进行序列化测试比如

    $ cd 6.824
    $ export "GOPATH=$PWD"  # go needs $GOPATH to be set to the project's working directory
    $ cd "$GOPATH/src/mapreduce"
    $ go test -run Sequential
    ok  	mapreduce	2.694s
