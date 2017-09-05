
## mapWithState

最近在项目中使用了PairDStreamFucntions的mapWithState方法。这个名字起的比较好，首先这个方法是map，那么生成的DStream的过程将是： DStream[(K,V)] => DStream[(M)]

* tips:

```
spark 中的命名有规律。 map 与 mapValues 不同.

map： （K, V） => U
mapValues：（K, V）=> （K, U）
```

然后withState，DStream内部维护了一个state。这个状态是怎么维护呢？

mapWithState的直觉是将增量（当前batch）和状态（上个batch）进行合并操作。对，spark-streaming中也是这么实现的。DStream中有个属性generatedRDDs，这个会保存最近若干(rememberDuration)批次产生的RDD，所以state就在这个generatedRDDs中。

```scala
abstract class DStream[T: ClassTag] (
    @transient private[streaming] var ssc: StreamingContext
  ) extends Serializable with Logging {

  // RDDs generated, marked as private[streaming] so that testsuites can access it
  @transient
  private[streaming] var generatedRDDs = new HashMap[Time, RDD[T]] ()
  
  /**
   * Get the RDD corresponding to the given time; either retrieve it from cache
   * or compute-and-cache it.
   */
  private[streaming] final def getOrCompute(time: Time): Option[RDD[T]] = {
    // If RDD was already generated, then retrieve it from HashMap,
    // or else compute the RDD
    generatedRDDs.get(time).orElse {
      // Compute the RDD if time is valid (e.g. correct time in a sliding window)
      // of RDD generation, else generate nothing.
      if (isTimeValid(time)) {

        val rddOption = createRDDWithLocalProperties(time, displayInnerRDDOps = false) {
          // Disable checks for existing output directories in jobs launched by the streaming
          // scheduler, since we may need to write output to an existing directory during checkpoint
          // recovery; see SPARK-4835 for more details. We need to have this call here because
          // compute() might cause Spark jobs to be launched.
          PairRDDFunctions.disableOutputSpecValidation.withValue(true) {
            compute(time)
          }
        }

        rddOption.foreach { case newRDD =>
          // Register the generated RDD for caching and checkpointing
          if (storageLevel != StorageLevel.NONE) {
            newRDD.persist(storageLevel)
            logDebug(s"Persisting RDD ${newRDD.id} for time $time to $storageLevel")
          }
          if (checkpointDuration != null && (time - zeroTime).isMultipleOf(checkpointDuration)) {
            newRDD.checkpoint()
            logInfo(s"Marking RDD ${newRDD.id} for time $time for checkpointing")
          }
          generatedRDDs.put(time, newRDD)
        }
        rddOption
      } else {
        None
      }
    }
  }
  
  /**
   * Clear metadata that are older than `rememberDuration` of this DStream.
   * This is an internal method that should not be called directly. This default
   * implementation clears the old generated RDDs. Subclasses of DStream may override
   * this to clear their own metadata along with the generated RDDs.
   */
  private[streaming] def clearMetadata(time: Time) {
    val unpersistData = ssc.conf.getBoolean("spark.streaming.unpersist", true)
    val oldRDDs = generatedRDDs.filter(_._1 <= (time - rememberDuration))
    logDebug("Clearing references to old RDDs: [" +
      oldRDDs.map(x => s"${x._1} -> ${x._2.id}").mkString(", ") + "]")
    generatedRDDs --= oldRDDs.keys
    if (unpersistData) {
      logDebug("Unpersisting old RDDs: " + oldRDDs.values.map(_.id).mkString(", "))
      oldRDDs.values.foreach { rdd =>
        rdd.unpersist(false)
        // Explicitly remove blocks of BlockRDD
        rdd match {
          case b: BlockRDD[_] =>
            logInfo("Removing blocks of RDD " + b + " of time " + time)
            b.removeBlocks()
          case _ =>
        }
      }
    }
    logDebug("Cleared " + oldRDDs.size + " RDDs that were older than " +
      (time - rememberDuration) + ": " + oldRDDs.keys.mkString(", "))
    dependencies.foreach(_.clearMetadata(time))
  }
```

如果每个批次的RDD的做cache操作，那还是挺耗内存的。

PairsDStreamFunctions.mapWithState方法生成DStream的类型为：

```
DStream[(K, V)] -----mapWithState------> DStream[(MappedType)]
```
这个是结果，这个是和中间结果是不同的。那这个可能使用了门面模式。涉及到的相关class的UML图如下所示：

![](http://note.youdao.com/yws/public/resource/a01e58c34305e0be0d8b83c20de98013/xmlnote/5D3288CE9A8B4309BAED5618FAE1C245/13161)

从上图可以发现MapWithStateDStreamImpl是一个门面类，InternalMapWithStateDStream是真正做事的类。

MapWithStateDStreamImpl的部分代码如下：

```scala
/** Internal implementation of the [[MapWithStateDStream]] */
private[streaming] class MapWithStateDStreamImpl[
    KeyType: ClassTag, ValueType: ClassTag, StateType: ClassTag, MappedType: ClassTag](
    dataStream: DStream[(KeyType, ValueType)],
    spec: StateSpecImpl[KeyType, ValueType, StateType, MappedType])
  extends MapWithStateDStream[KeyType, ValueType, StateType, MappedType](dataStream.context) {

  private val internalStream =
    new InternalMapWithStateDStream[KeyType, ValueType, StateType, MappedType](dataStream, spec)

  override def slideDuration: Duration = internalStream.slideDuration

  override def dependencies: List[DStream[_]] = List(internalStream)

```

从上的代码可以发现 MapWithStateDStreamImpl的parent是internalStream，而不是通过构造函数传进来的dataStream。当然insterStream的parent会是dataStream。

```scala
private[streaming]
class InternalMapWithStateDStream[K: ClassTag, V: ClassTag, S: ClassTag, E: ClassTag](
    parent: DStream[(K, V)], spec: StateSpecImpl[K, V, S, E])
  extends DStream[MapWithStateRDDRecord[K, S, E]](parent.context) {

  persist(StorageLevel.MEMORY_ONLY)

  private val partitioner = spec.getPartitioner().getOrElse(
    new HashPartitioner(ssc.sc.defaultParallelism))

  private val mappingFunction = spec.getFunction()

  override def slideDuration: Duration = parent.slideDuration

  override def dependencies: List[DStream[_]] = List(parent)
  
  /** Enable automatic checkpointing */
  override val mustCheckpoint = true

  /** Override the default checkpoint duration */
  override def initialize(time: Time): Unit = {
    if (checkpointDuration == null) {
      checkpointDuration = slideDuration * DEFAULT_CHECKPOINT_DURATION_MULTIPLIER
    }
    super.initialize(time)
  }
```

DStreamGraph的就串联起来了。从上面InternalMapWithStateDStream的代码可以看到，由这个DStream生成的RDD会自动的做persist和checkpoint，所以我们在代码中就没有必要显示声明checkpoint了。

所以，mapWithState的重头戏就是InternalMapWithStateDStream了。我们在分析DStream的时候要知道看哪些函数，DStream中重要函数执行的序列如下：

![](http://note.youdao.com/yws/public/resource/a01e58c34305e0be0d8b83c20de98013/xmlnote/0F1A529CADE94016AA466FC1276A3651/13191)

嗯，看InternalMapWithStateDStream.compute方法

```scala
/** Method that generates a RDD for the given time */
  override def compute(validTime: Time): Option[RDD[MapWithStateRDDRecord[K, S, E]]] = {
    // Get the previous state or create a new empty state RDD
    val prevStateRDD = getOrCompute(validTime - slideDuration) match {
      case Some(rdd) =>
        if (rdd.partitioner != Some(partitioner)) {
          // If the RDD is not partitioned the right way, let us repartition it using the
          // partition index as the key. This is to ensure that state RDD is always partitioned
          // before creating another state RDD using it
          MapWithStateRDD.createFromRDD[K, V, S, E](
            rdd.flatMap { _.stateMap.getAll() }, partitioner, validTime)
        } else {
          rdd
        }
      case None =>
        MapWithStateRDD.createFromPairRDD[K, V, S, E](
          spec.getInitialStateRDD().getOrElse(new EmptyRDD[(K, S)](ssc.sparkContext)),
          partitioner,
          validTime
        )
    }


    // Compute the new state RDD with previous state RDD and partitioned data RDD
    // Even if there is no data RDD, use an empty one to create a new state RDD
    val dataRDD = parent.getOrCompute(validTime).getOrElse {
      context.sparkContext.emptyRDD[(K, V)]
    }
    val partitionedDataRDD = dataRDD.partitionBy(partitioner)
    val timeoutThresholdTime = spec.getTimeoutInterval().map { interval =>
      (validTime - interval).milliseconds
    }
    Some(new MapWithStateRDD(
      prevStateRDD, partitionedDataRDD, mappingFunction, validTime, timeoutThresholdTime))
  }
```
这个里面的逻辑就比较简单了。不再赘述

## checkpoint

开始在测试mapWithState的稳定性，担心如果spark-streaming中有executor挂掉了，中间的状态会不会丢失。发现测试的结果是不会丢失，及时是3个executors中挂掉了2个也依然没有丢失。当时就比较懵逼了（因为没有看源码）。mapWithState的稳定性在于RDD的稳定性。

因为不了解RDD checkpoint在稳定性发挥的作用，之前一直自认为只有在application重启的时候才生效，才导致有上面的疑惑。

首先checkpointed的RDD会在后续的使用中生效（不是application级别重启）， 它要比persist更稳定但性能偏差（需要从reliable的存储读取数据）。当RDD既做persist又做checkpoint时，当再次使用该RDD时，时从persist上读取，还是从checkpoint读取？ - persist

checkpoint写的调用序列：

![](http://note.youdao.com/yws/public/resource/a01e58c34305e0be0d8b83c20de98013/xmlnote/B58FC2428F104D15A35458B84E164C19/13245)

对应SparkContext中的方法为：

```scala
/**
   * Run a function on a given set of partitions in an RDD and pass the results to the given
   * handler function. This is the main entry point for all actions in Spark.
   */
  def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
      throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
      logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
  }
```

从上图可以发现，checkpoint是在job运行完之后做的，所以如果对中间的RDD做checkpoint，做会重新计算一遍，如果是最后一个RDD则不会。

doCheckpoint做了哪些事情呢？

```scala
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging {

private[spark] var checkpointData: Option[RDDCheckpointData[T]] = None

private def checkpointRDD: Option[CheckpointRDD[T]] = checkpointData.flatMap(_.checkpointRDD)

/**
   * Performs the checkpointing of this RDD by saving this. It is called after a job using this RDD
   * has completed (therefore the RDD has been materialized and potentially stored in memory).
   * doCheckpoint() is called recursively on the parent RDDs.
   */
  private[spark] def doCheckpoint(): Unit = {
    RDDOperationScope.withScope(sc, "checkpoint", allowNesting = false, ignoreParent = true) {
      if (!doCheckpointCalled) {
        doCheckpointCalled = true
        if (checkpointData.isDefined) {
          checkpointData.get.checkpoint()
        } else {
          dependencies.foreach(_.rdd.doCheckpoint())
        }
      }
    }
  }
  
  def checkpoint(): Unit = RDDCheckpointData.synchronized {
    // NOTE: we use a global lock here due to complexities downstream with ensuring
    // children RDD partitions point to the correct parent partitions. In the future
    // we should revisit this consideration.
    if (context.checkpointDir.isEmpty) {
      throw new SparkException("Checkpoint directory has not been set in the SparkContext")
    } else if (checkpointData.isEmpty) {
      checkpointData = Some(new ReliableRDDCheckpointData(this))
    }
  }

  /**
   * Changes the dependencies of this RDD from its original parents to a new RDD (`newRDD`)
   * created from the checkpoint file, and forget its old dependencies and partitions.
   */
  private[spark] def markCheckpointed(): Unit = {
    clearDependencies()
    partitions_ = null
    deps = null    // Forget the constructor argument for dependencies too
  }
```

从上面的代码中可以发下，doCheckpoint的逻辑主要是调用：ReliableRDDCheckpointData.checkpoint，它做了哪些事情呢？看下面的代码：

```scala
final def checkpoint(): Unit = {
    // Guard against multiple threads checkpointing the same RDD by
    // atomically flipping the state of this RDDCheckpointData
    RDDCheckpointData.synchronized {
      if (cpState == Initialized) {
        cpState = CheckpointingInProgress
      } else {
        return
      }
    }

    val newRDD = doCheckpoint()

    // Update our state and truncate the RDD lineage
    RDDCheckpointData.synchronized {
      cpRDD = Some(newRDD)
      cpState = Checkpointed
      rdd.markCheckpointed()
    }
  }
  
  /**
   * Materialize this RDD and write its content to a reliable DFS.
   * This is called immediately after the first action invoked on this RDD has completed.
   */
  protected override def doCheckpoint(): CheckpointRDD[T] = {
    val newRDD = ReliableCheckpointRDD.writeRDDToCheckpointDirectory(rdd, cpDir)

    // Optionally clean our checkpoint files if the reference is out of scope
    if (rdd.conf.getBoolean("spark.cleaner.referenceTracking.cleanCheckpoints", false)) {
      rdd.context.cleaner.foreach { cleaner =>
        cleaner.registerRDDCheckpointDataForCleanup(newRDD, rdd.id)
      }
    }

    logInfo(s"Done checkpointing RDD ${rdd.id} to $cpDir, new parent is RDD ${newRDD.id}")
    newRDD
  }
```

ReliableCheckpointRDD.writeRDDToCheckpointDirectory(rdd, cpDir) 的逻辑就是将数据写入到reliable的目录，这里不再赘述。有一个点比较有意思就是 rdd.markCheckpointed()。这个调用在checkpoint的读取中比较重要，它做了哪些事情呢？

```scala
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging {
  
    private[spark] def markCheckpointed(): Unit = {
        clearDependencies()
        partitions_ = null
        deps = null    // Forget the constructor argument for dependencies too
    }
    
    protected def clearDependencies() {
         dependencies_ = null
    }
```

**这个方法将deps、partitions_、dependencies_都设置为null，这有什么用意么？**

如果看代码的时候，怎样验证RDD在做完checkpoint后，再次消费时，是否会读取checkpoint呢？入口是哪里？（RDD有很多方法：iterator、partitions、dependencies、getPartitions、getDependencies）

目前我认为的入口是org.apache.spark.scheduler.Task.runTask(TaskContext)。我们看一下这个方法(这个类有两个实现)：
```scala
private[spark] class ShuffleMapTask(
    stageId: Int,
    stageAttemptId: Int,
    taskBinary: Broadcast[Array[Byte]],
    partition: Partition,
    @transient private var locs: Seq[TaskLocation],
    internalAccumulators: Seq[Accumulator[Long]])
  extends Task[MapStatus](stageId, stageAttemptId, partition.index, internalAccumulators)
  with Logging {
  
    override def runTask(context: TaskContext): MapStatus = {
    // Deserialize the RDD using the broadcast variable.
    val deserializeStartTime = System.currentTimeMillis()
    val ser = SparkEnv.get.closureSerializer.newInstance()
    val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime

    metrics = Some(context.taskMetrics)
    var writer: ShuffleWriter[Any, Any] = null
    try {
      val manager = SparkEnv.get.shuffleManager
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
      writer.stop(success = true).get
    } catch {
      case e: Exception =>
        try {
          if (writer != null) {
            writer.stop(success = false)
          }
        } catch {
          case e: Exception =>
            log.debug("Could not stop writer", e)
        }
        throw e
    }
  }
```


```scala
private[spark] class ResultTask[T, U](
    stageId: Int,
    stageAttemptId: Int,
    taskBinary: Broadcast[Array[Byte]],
    partition: Partition,
    locs: Seq[TaskLocation],
    val outputId: Int,
    internalAccumulators: Seq[Accumulator[Long]])
  extends Task[U](stageId, stageAttemptId, partition.index, internalAccumulators)
  with Serializable {

  @transient private[this] val preferredLocs: Seq[TaskLocation] = {
    if (locs == null) Nil else locs.toSet.toSeq
  }

  override def runTask(context: TaskContext): U = {
    // Deserialize the RDD and the func using the broadcast variables.
    val deserializeStartTime = System.currentTimeMillis()
    val ser = SparkEnv.get.closureSerializer.newInstance()
    val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime

    metrics = Some(context.taskMetrics)
    func(context, rdd.iterator(partition, context))
  }

```

这两个类都调用了RDD.iterator方法，这个方法代码如下：

```scala
/**
   * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
   * This should ''not'' be called by users directly, but is available for implementors of custom
   * subclasses of RDD.
   */
  final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```

可以发现，如果RDD设置了persist策略，则走persist；没有读取checkpoint或者计算（没有做checkpoint）。这个有个疑惑为什么方法名要为computeOrReadCheckpoint?(compute在前、readCheckpoint在后) ？

我们在看一下computeOrReadCheckpoint：

```scala
/**
   * Compute an RDD partition or read it from a checkpoint if the RDD is checkpointing.
   */
  private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] =
  {
    if (isCheckpointedAndMaterialized) {
      firstParent[T].iterator(split, context)
    } else {
      compute(split, context)
    }
  }
  
  protected[spark] def firstParent[U: ClassTag]: RDD[U] = {
    dependencies.head.rdd.asInstanceOf[RDD[U]]
  }
  
  final def dependencies: Seq[Dependency[_]] = {
    checkpointRDD.map(r => List(new OneToOneDependency(r))).getOrElse {
      if (dependencies_ == null) {
        dependencies_ = getDependencies
      }
      dependencies_
    }
  }
```

从这段代码可以看出：

* DAG被改变了，当前RDD的parent被指向了该RDD内的checkpointRDD， 最终是从 checkpointRDD里面读取数据的。
* 这个和mapWithState的实现类似，使用了门面模式或者代理模式（对这两个模式不敏感）
