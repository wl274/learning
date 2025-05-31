# 继续学习Spark

## 处理中的 MapReduce 和弹性分布式数据集（RDD）相关内



### 内存计算的容错性

首先介绍内存计算的容错方法，包括：

- **全部计算复制（Replicate all computations）**：该方法成本高昂，会降低峰值吞吐量。
- **检查点与回滚（Checkpoint and rollback）**：定期将程序状态保存到持久存储中，节点发生故障时从最后一个检查点重新启动。
- **维护更新日志（Maintain log of updates）**：记录命令和数据更新，但维护日志开销较大。同时回顾了MapReduce解决方案中的相关要点，如在每个Map/Reduce步骤后设置检查点等。

### 设计MapReduce作业的实现

提出设计 `runMapReduceJob` 的任务，引导思考MapReduce作业的实际实现过程 。

### 作业调度器的职责

作业调度器有以下重要职责：

- **利用数据局部性（Exploit data locality）**：遵循“将计算移到数据处”原则，在包含输入块的节点上运行Mapper作业，在已拥有特定键大部分数据的节点上运行Reducer作业。
- **处理节点故障（Handling node failures）**：调度器检测到作业失败后，在新机器上重新运行作业，这得益于输入数据存储在持久存储（分布式文件系统）中，还可在多台机器上复制作业以减少节点故障导致的处理延迟。 
- **处理慢速机器（Handling slow machines）**：在多台机器上复制作业。 

### runMapReduceJob存在的问题

`runMapReduceJob` 存在以下局限：

- **程序结构简单（Permits only a very simple program structure）**：程序必须按Map、然后按键Reduce的结构编写，如需实现更通用的有向无环图（DAG）结构可参考DryadLINQ 。
- **迭代算法的磁盘读取问题（Iterative algorithms must load from disk each iteration）**：以图处理为例，迭代算法每次迭代都需从磁盘加载数据，效率较低 。

### MapReduce在词频统计中的数据流

以经典的词频统计（Word Count）为例，展示MapReduce的数据流过程：输入文本经过Map阶段将单词拆分并标记计数，再通过GroupByKey阶段分组，最后在Reduce阶段对相同单词的计数进行汇总，得出每个单词的出现频率 。

### 数据收集与运行Reducer的步骤

讲解了MapReduce中数据收集和运行Reducer的步骤：

- **步骤2**：为Reducer准备中间数据。
- **步骤3**：在所有键上运行Reducer函数。并提出两个关键问题，一是如何分配Reducer任务，二是如何将某个键的所有数据传送到正确的Reduce工作节点 。

### 集群计算的目标

阐述集群规模计算的目标：

- **编程模型需求**：构建适用于集群规模计算的编程模型，支持中间数据集的显著复用，适用于迭代机器学习和图算法，以及交互式数据挖掘场景（将大型数据集加载到集群内存后执行多个即席查询） 。
- **避免低效存储**：避免将中间数据写入持久分布式文件系统带来的低效问题，希望将数据保留在内存中，同时面临如何为大规模分布式内存计算高效实现容错性的挑战 。

### MapReduce的局限性

MapReduce虽极大简化了大数据分析，但用户很快有了更多需求：

- **复杂应用需求**：需要支持更复杂的多阶段应用，如迭代机器学习和图处理。
- **交互式查询需求**：需要更多交互式即席查询功能 。

### 弹性分布式数据集（RDD）

RDD是Spark的关键编程抽象：

- **特性**：是只读的有序记录集合（不可变）。
- **创建方式**：只能通过对持久存储中的数据或现有RDD进行确定性转换来创建。
- **操作**：对RDD的操作会将数据返回给应用程序。还通过代码示例展示了从文件系统数据创建RDD，以及对RDD进行过滤（filter）转换和计数（count）操作的过程 。

### MapReduce编程模型

最后展示了MapReduce编程模型的代码示例，用于计算每种移动设备的页面浏览量：

- **Mapper函数**：输入为输入文件中的一行，解析用户代理，判断是否为移动客户端，若是则将其添加到结果映射中。
- **Reducer函数**：对每个唯一键（用户代理）关联的值列表进行求和操作。
- **整体流程**：通过 `LineByLineReader` 逐行读取文件，使用 `Writer` 存储输出，最后调用 `runMapReduceJob` 执行作业 。 

###  “Another Spark Program”

代码展示了一个Spark程序，从文件系统数据创建RDD（弹性分布式数据集） ，并进行一系列转换操作：

- 首先从文件 `hdfs://cs149log.txt` 创建 `lines` RDD。
- 然后通过过滤（`filter`）操作，筛选出移动客户端相关记录创建 `mobileViews` RDD 。这里 `mobileViews.persist()` 是优化点，它指示Spark运行时尽量将 `mobileViews` 数据保留在内存中，避免重复从磁盘读取，加快后续对该RDD的操作速度。
- 接着进一步过滤出包含 “Safari” 的记录并统计数量，还展示了另一个操作，筛选出包含 “Chrome” 的记录，对每个元素拆分字符串并提取时间戳 。

###  “Review: why did we perform this transform?”

对比了两个程序，核心是图像模糊处理代码的转换：

- **Program 1**：对整个图像数据进行水平和垂直方向的模糊处理，数据操作在较大的数组范围内进行。
- **Program 2**：采用 “tiling”（分块）技术，将图像数据划分为大小为 `CHUNK_SIZE` 的块 。按块进行水平和垂直方向的模糊处理，好处是减少了处理过程中所需的内存量 。比如在处理大图像时，不需要一次性将整个图像数据都加载到内存，而是分块处理，降低内存压力。

### “RDD partitioning and dependencies”

展示了RDD的分区和依赖关系：

- 从文件 `hdfs://cs149log.txt` 创建 `lines` RDD后，经过 `map` 操作转换为 `lower` RDD（将文本转为小写），再经过 `filter` 操作得到 `mobileViews` RDD（筛选移动客户端记录），最后通过 `count` 操作统计数量 。
- 数据在不同节点（Node 0 - Node 3）上进行分区存储，每个节点包含不同的数据块（block） 。黑色线条表示RDD分区之间的依赖关系 。这张图探讨的问题是如何优化RDD的实现，比如可以通过合理安排分区和利用依赖关系，减少数据传输和计算开销，提高Spark应用程序的性能 。 