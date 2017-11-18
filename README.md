# distributed-sys-6.824
A distributed systems build from scratch/ MIT 6.824 course



#### what is 6.824 about?

6.824 is a core 12-unit graduate subject with lectures, readings, programming labs, an optional project, a mid-term exam, and a final exam. It will present abstractions and implementation techniques for engineering distributed systems. Major topics include fault tolerance, replication, and consistency. Much of the class consists of studying and discussing case studies of distributed systems.

### mapreduce 文章review

简单地说，MapReduce是一个受限的分布式并行编程模型，可用于处理和输出很大的数据集。而编写MapReduce任务的用户只需要实现两个函数：

Map函数：输入一个key/value数据，输出一个key/value形式的中间数据集。
Reduce函数：输入是一个中间数据的key和一个与这个key对应的value集合。它负责将这些值按照一定的规则进行“合并”。

例子——WordCount

过程

+ MapReduce框架对输入的文件数据分成M片，每份数据的大小为16~64MB（可由用户配置）。
+ 在多台机器上开始运行User Program：包括一个master、多个map worker和多个reduce worker。
+ master主要负责map worker和reduce worker的状态管理和任务分发。
+ map worker从GFS读取分配到的文件数据，并进行相应的处理。MapReduce框架的调度会尽量使map worker运行的机器与数据靠近，以提高数据传输的效率。所以，数据传输可以是本地，也可能是网络。
+ map worker的输出缓存在内存中，并定期刷到本地磁盘上。这些中间数据的位置信息会通过心跳信息告诉master，master记下这些信息后，通知reduce worker。数据存储在本地。
+ reduce worker通过RPC从map worker读取需要的中间数据。数据通过网络传输。
+ reduce worker对中间数据进行“合并”处理后，输出结果。

容错

+ map worker

    map任务执行完成后宕机：因为中间数据存储在本地磁盘，需要重新执行。
    map任务执行完成前宕机：需要重新执行。

+ reduce worker

    reduce任务执行完成后宕机：因为数据存储在GFS，不需要重新执行。
    reduce任务执行完成前宕机：需要重新执行，输出文件可以覆盖原来的（文件名一样）。

    master宕机，任务失败。（master是个单点）

优化

    局部性：MapReduce用于大数据集的处理，其主要瓶颈是网络带宽。通过优化调度，可以让执行MapReduce任务的机器尽可能靠近机器。（同一机器==>同一机架==>同一机房...）
    任务粒度：执行MapReduce任务的过程其实就是M个Map任务+R个Reduce任务。M和R必须比机器数大很多才会有利于负载均衡。
    备份任务：当MapReduce任务即将执行完成时，MapReduce框架会针对那些还在执行的任务，启动一个对应的备份任务。之后，只要主任务或备份任务执行完成，MapReduce任务就完成了。这样可以有效避免整个MapReduce任务被少部分比较慢的机器拖死。


所以说 mapreduce的使用：

比如workcount一个hello的字，先用map，计算所有的网页，每一个字的位置，然后用reduce，在每一个字的清单中，选出hello。

1. 将执行的mapreduce复制道master每一个worker中
2. master决定map和reduce
3. 所有的资料分块，分配到执行map的worker中进行map
4. 将map后的结果存入worker机器的本地区块
5. 执行reduce的worker机器，读取每一份的map结果，执行整理排序，执行reduce
6. 将使用者需要的运算结果输出

如：首先将任务分成5份，接着master将5个任务分别分配给3个worker，每一个执行map的worker会产生2个中间文件（这个2就是源码中nReduce，也就是执行reduce的worker的数量）。map产生的中间文件的内容是，("he","1") ; ("love","1")......这样的键值对，每一个单词都是1。接着reduce处理这些中间文件，将相同的单词的"1"加起来，得到每一个中间文件中每一个单词的个数，并产生一个输出结果文件。后面还会有一个merge函数会统计所有的输出结果文件。


