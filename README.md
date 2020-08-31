我是光年实验室高级招聘经理。
我在github上访问了你的开源项目，你的代码超赞。你最近有没有在看工作机会，我们在招软件开发工程师，拉钩和BOSS等招聘网站也发布了相关岗位，有公司和职位的详细信息。
我们公司在杭州，业务主要做流量增长，是很多大型互联网公司的流量顾问。公司弹性工作制，福利齐全，发展潜力大，良好的办公环境和学习氛围。
公司官网是http://www.gnlab.com,公司地址是杭州市西湖区古墩路紫金广场B座，若你感兴趣，欢迎与我联系，
电话是0571-88839161，手机号：18668131388，微信号：echo 'bGhsaGxoMTEyNAo='|base64 -D ,静待佳音。如有打扰，还请见谅，祝生活愉快工作顺利。

# distributed-sys-6.824
A distributed systems build from scratch/ MIT 6.824 course

2019 再次更新，目前追到lab3

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


- - -
如何处理较慢的网络？参考论文3.4节减少网络带宽资源的浪费，都尽量让输入数据保存在构成集群机器的本地硬盘上，并通过使用分布式文件系统GFS进行本地磁盘的管理。尝试分配map任务到尽量靠近这个任务的输入数据库的机器上执行，这样从GFS读时大部分还是在本地磁盘读出来。中间数据传输（map到reduce）经过网络一次，但是分多个key并行执行

    3.4 本地存储
    网络带宽是我们的计算环境中相对稀缺的资源。 我们通过利用输入数据（由GFS [8]管理）存储在组成我们集群的机器的本地磁盘上的事实来节省网络带宽。 GFS将每个文件分成64 MB块，并在不同的机器上存储每个块的多个副本（通常是3个副本）。 MapReduce主机会将输入文件的位置信息考虑在内，并尝试在包含相应输入数据副本的计算机上调度映射任务。 否则，它将尝试在该任务的输入数据副本附近（例如，在与包含数据的计算机处于同一网络交换机上的工作机上）安排一个映射任务。 在群集中大部分工作人员上运行大型MapReduce操作时，大多数输入数据都是本地读取的，不会消耗网络带宽。


如何做负载均衡？某个task运行时间比较其他N-1个都长，大家都必须等其结束那就尴尬了，因此参考论文3.5节、3.6节系统设计保证task比worker数量要多，做的快的worker可以继续先执行其他task，减少等待。（框架的任务调度后来发现更值得研究）

    3.5 任务粒度
    如上所述，我们将地图阶段细分为M个部分，将剩余部分细分为R个部分。 理想情况下，M和R应该比工人机器的数量大得多。 使每个工作人员执行许多不同的任务可以改善动态负载平衡，并且还可以在工作人员失败时加快恢复速度：已经完成的许多地图任务可以分散到所有其他工作机器中。

    在我们的实现中，M和R有多大是有实际意义的，因为如上所述，主机必须做O（M + R）调度决策并保持O（M * R）状态。 （然而，内存使用的常量因素很小：状态的O（M * R）部分由每个映射任务/减少任务对的大约一个字节的数据组成）。

    而且，R常常受到用户的约束，因为每个reduce任务的输出都在一个单独的输出文件中。 在实践中，我们倾向于选择M，使得每个单独的任务大致为16MB到64MB的输入数据（以便上述的局部最优化是最有效的），并且使得R成为工人数量的小倍数 我们期望使用的机器。 我们经常使用2000台工作机器执行MapReduce计算，其中M = 200,000，R = 5,000。
    
    3.6 备份任务
    延长MapReduce操作总时间的一个常见原因是“失败者”（straggler）：需要花费异常长时间才能完成最后一个映射或减少计算中的任务之一的计算机。落后者可能会出现一系列的原因。例如，磁盘坏的机器可能会经历频繁的可纠正错误，使其读取性能从30 MB / s降至1 MB / s。集群调度系统可能在机器上安排了其他任务，由于竞争CPU，内存，本地磁盘或网络带宽，导致它更慢地执行MapReduce代码。我们遇到的一个最近的问题是机器初始化代码中的一个错误，导致处理器缓存被禁用：受影响的计算机的计算速度减慢了一百倍。

    我们有一个缓解落后者问题的普遍机制。当MapReduce操作接近完成时，主服务器会计划剩余进行中任务的备份执行。无论主或备份执行完成，任务都会标记为已完成。我们已经调整了这个机制，所以它通常增加了不到百分之几的操作使用的计算资源。我们发现这大大减少了完成大型MapReduce操作的时间。例如，5.3节中描述的排序程序在备份任务机制被禁用时需要花费44％的时间才能完成。

如何做容错性？参考论文3.3节重新执行那些失败的MR任务即可，因此需要保证MR任务本身是幂等且无状态的。

更特别一些，worker失效如何处理？将失败任务调配到其他worker重新执行，保证最后输出到GFS上的中间结果过程是原子性操作即可。（减少写错数据的可能）

Master失效如何处理？因为master是单点，只能人工干预，系统干脆直接终止，让用户重启重新执行这个计算
- - -


所以说 mapreduce的使用：

比如workcount一个hello的字，先用map，计算所有的网页，每一个字的位置，然后用reduce，在每一个字的清单中，选出hello。

1. 将执行的mapreduce复制道master每一个worker中
2. master决定map和reduce
3. 所有的资料分块，分配到执行map的worker中进行map
4. 将map后的结果存入worker机器的本地区块
5. 执行reduce的worker机器，读取每一份的map结果，执行整理排序，执行reduce
6. 将使用者需要的运算结果输出

如：首先将任务分成5份，接着master将5个任务分别分配给3个worker，每一个执行map的worker会产生2个中间文件（这个2就是源码中nReduce，也就是执行reduce的worker的数量）。map产生的中间文件的内容是，("he","1") ; ("love","1")......这样的键值对，每一个单词都是1。接着reduce处理这些中间文件，将相同的单词的"1"加起来，得到每一个中间文件中每一个单词的个数，并产生一个输出结果文件。后面还会有一个merge函数会统计所有的输出结果文件。


- - -
#### part1.

part1需要编写doMap和doReduce这两个函数，这两个函数是框架中的函数，不用考虑具体的事务。我说一下这两个函数的流程。

doMap，首先调用任务文件的内容为参数调用mapF，将结果存取键值对的数组（这里需要读文件，推荐ioutil库）。接着创建nReduce（nReduce在上面图片下面的讲解处）个中间文件，中间文件的名字使用reduceName（common_map.go）来产生。最后把键值对的内容存储到nReduce个文件中，那么有nReduce个文件，怎么知道把某一个key－value对存入到哪一个文件中呢。这里需要用ihash，我使用了如 int(ihash(x.Key))%nReduce 这样的代码，不知道有没有更好的办法。这里也要用到json库，查一查很简单。

doReduce，reduce的事情较多，我列出流程，方便理解。

1.找到属于自己的中间文件（这个可以参照图）

2.读出中间文件的每一个key／value对

3.sort，这里的sort指的是将相同的key／value对存入一个数组中，结果像这样{"he", "1", "1", "1"...}....（从doReduce函数的描述和mapF的参数得到的）

4.对每一个排序过的结果调用redeceF

5.使用mergername函数创建结果文件的名字，创建文件

6.将每个reduceF的结果写入到上一步产生的文件

*这个part是不用写mapF和reduceF的

#### part2.

这里就要写word count的mapF和reduceF，很简单就不说了。

#### part3.

前面的版本是顺序版本，这个part要实现并发的版本了，这里的并发是通过多个进程模拟的，不过也是通过rpc实现的，跟真实的情况还是很相近的。具体就不描述了，这里主要是运用这个registerChannel这个channel。注意到当有worker向master注册的时候，会向registerChannel发送消息，我们可以设计当某一个worker执行完之后也向这个registerChannel发送消息，这样程序的框架大概是这样

    for{

    str := <- mr.registerChannel

    //分析str....

    }

具体就不描述了，这个实验的乐趣就在这里了。我说一下我调试的方法，在关键点输出"@+信息"，然后执行并将程序的输出重定向到文件中，再在文件中搜索"@"。

#### part4.

能做出前面的话，这里就不成问题了。我这里在加锁解锁方面出了问题，调试了很久。
