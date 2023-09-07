---
layout:     post
title:      "内存泄漏治理shark解析hprof文件"
subtitle:   ""
date:       2022-09-30 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - LeakCanary-shark
---

解析hprof文件主要分为如下几个步骤，逐一来介绍。

# 1.PARSING_HEAP_DUMP

这一步是解析hprof文件，包含两部分：

- 解析Header部分
- 解析Records部分

hprof文件格式参考：http://hg.openjdk.java.net/jdk6/jdk6/jdk/raw-file/tip/src/share/demo/jvmti/hprof/manual.html



重点在读取Records部分，读取逻辑在 StreamingHprofReader 这个类，处理逻辑在 

HprofInMemoryIndex 这个类中，这里共读取了两遍：

第1遍，收集数据信息，包括 类/实例/object数组/原始类型数组 的数量、数据最大长度等等。

第2遍，生成 类/实例/object数组/原始类型数组 的信息，以支持随机读取。



|               | 支持随机读取所需信息，以Byte数组形式存储             |
| ------------- | ---------------------------------------------------- |
| 类            | ID，文件中位置，父类ID，实例大小，数据长度，数组下标 |
| 实例          | ID，文件中位置，类ID，数据长度                       |
| object数组    | ID，文件中位置，类ID，数据长度                       |
| primitive数组 | ID，文件中位置，原始类型，数据长度                   |

这里解释下为什么要读两遍：

主要是从节省内存的角度考虑，比如 文件中位置 这个数值，第一遍读取后我们可以获得文件的大小，从而可以确定存储 文件中位置 这个数值所需要的byte数量，从而避免直接使用int或long存储节省内存。



这里有两个知识点：

**1.Shark 比 Haha 占用内存小的原因？**

Shark并不会直接把所有的Records都读取到内存中。



**2.gc root对象都有哪些？**

```Kotlin
fun defaultIndexedGcRootTags(): EnumSet<HprofRecordTag> = EnumSet.of(
  HprofRecordTag.ROOT_JNI_GLOBAL,
  HprofRecordTag.ROOT_JAVA_FRAME,
  HprofRecordTag.ROOT_JNI_LOCAL,
  HprofRecordTag.ROOT_MONITOR_USED,
  HprofRecordTag.ROOT_NATIVE_STACK,
  HprofRecordTag.ROOT_STICKY_CLASS,
  HprofRecordTag.ROOT_THREAD_BLOCK,
  HprofRecordTag.ROOT_THREAD_OBJECT,
  HprofRecordTag.ROOT_JNI_MONITOR
)
```



# 2.EXTRACTING_METADATA

这一步是解析一些常量值，具体参考 AndroidMetadataExtractor 这个类。

```Kotlin
override fun extractMetadata(graph: HeapGraph): Map<String, String> {
  val metadata = mutableMapOf<String, String>()

  val build = AndroidBuildMirror.fromHeapGraph(graph)
  metadata["Build.VERSION.SDK_INT"] = build.sdkInt.toString()
  metadata["Build.MANUFACTURER"] = build.manufacturer
  metadata["LeakCanary version"] = readLeakCanaryVersion(graph)
  metadata["App process name"] = readProcessName(graph)
  metadata["Class count"] = graph.classCount.toString()
  metadata["Instance count"] = graph.instanceCount.toString()
  metadata["Primitive array count"] = graph.primitiveArrayCount.toString()
  metadata["Object array count"] = graph.objectArrayCount.toString()
  metadata["Thread count"] = readThreadCount(graph).toString()
  metadata["Heap total bytes"] = readHeapTotalBytes(graph).toString()
  metadata.putBitmaps(graph)
  metadata.putDbLabels(graph)

  return metadata
}
```

# 3.FINDING_RETAINED_OBJECTS

这一步是找出泄漏对象。Android中常见的泄漏对象定义在 AndroidObjectInspectors 这个类中，

以 Activity 为例，当 Activity 实例中 mDestroyed == true 时，也就是 Activity 执行完 onDestroyed 回调，Activity 实例仍存在则判定为内存泄漏。

```Kotlin
override val leakingObjectFilter = { heapObject: HeapObject ->
  heapObject is HeapInstance &&
    heapObject instanceOf "android.app.Activity" &&
    heapObject["android.app.Activity", "mDestroyed"]?.value?.asBoolean == true
}
```

# 4.FINDING_PATHS_TO_RETAINED_OBJECTS

这一步是找出泄漏最短路径。核心逻辑在 PathFinder 类，首先遍历所有的 gc roots 对象，然后依次递归遍历其所引用的子对象，直至找出包含上一步找出的所有泄漏对象。因为在遍历过程中可以创建父子关系，所以从泄漏对象一直找parent，即为最短泄漏路径。

```Kotlin
private fun State.findPathsFromGcRoots(): PathFindingResults {
  enqueueGcRoots()

  val shortestPathsToLeakingObjects = mutableListOf<ReferencePathNode>()
  visitingQueue@ while (queuesNotEmpty) {
    val node = poll()
    if (leakingObjectIds.contains(node.objectId)) {
      shortestPathsToLeakingObjects.add(node)
      // Found all refs, stop searching (unless computing retained size)
      if (shortestPathsToLeakingObjects.size == leakingObjectIds.size()) {
        if (computeRetainedHeapSize) {
          listener.onAnalysisProgress(FINDING_DOMINATORS)
        } else {
          break@visitingQueue
        }
      }
    }

    val heapObject = graph.findObjectById(node.objectId)
    objectReferenceReader.read(heapObject).forEach { reference ->
      val newNode = ChildNode(
        objectId = reference.valueObjectId,
        parent = node,
        lazyDetailsResolver = reference.lazyDetailsResolver
      )
      enqueue(newNode, isLowPriority = reference.isLowPriority)
    }
  }
  return PathFindingResults(
    shortestPathsToLeakingObjects,
    if (visitTracker is Dominated) visitTracker.dominatorTree else null
  )
}
```



这里还包含了关键一步，去重最短路径，逻辑在 HeapAnalyzer.deduplicateShortestPaths() ，也就是相同泄漏路径会被合并。举个例子，比如2个不同Activity都是注册了同一listener导致的泄漏，因此泄漏路径是一致的，会被自动合并。

# 5.FINDING_DOMINATORS

如果需要计算泄漏对象导致内存泄漏大小，会在上一步的基础上遍历所有对象。

备注：为了提升效率，我们去掉了这部分功能。

# 6.INSPECTING_OBJECTS

这一步是描述对象是否有泄漏及泄漏原因，还是以 Activity 为例：

```Kotlin
override fun inspect(
  reporter: ObjectReporter
) {
  reporter.whenInstanceOf("android.app.Activity") { instance ->
    // Activity.mDestroyed was introduced in 17.
    // https://android.googlesource.com/platform/frameworks/base/+
    // /6d9dcbccec126d9b87ab6587e686e28b87e5a04d
    val field = instance["android.app.Activity", "mDestroyed"]

    if (field != null) {
      if (field.value.asBoolean!!) {
        leakingReasons += field describedWithValue "true"
      } else {
        notLeakingReasons += field describedWithValue "false"
      }
    }
  }
}
```

# 7.COMPUTING_NATIVE_RETAINED_SIZE

这一步是计算 Native 内存泄漏大小，主要逻辑在 AndroidNativeSizeMapper 类。

# 8.COMPUTING_RETAINED_SIZE

这一步是计算泄漏对象导致内存泄漏大小，主要逻辑在 DominatorTree 类。

# 9.BUILDING_LEAK_TRACES

这一步是生成泄漏堆栈，包括：ApplicationLeak，LibraryLeak。

```Kotlin
private fun FindLeakInput.buildLeakTraces(
  shortestPaths: List<ShortestPath>,
  inspectedObjectsByPath: List<List<InspectedObject>>,
  retainedSizes: Map<Long, Pair<Int, Int>>?
): Pair<List<ApplicationLeak>, List<LibraryLeak>> {
  listener.onAnalysisProgress(BUILDING_LEAK_TRACES)

  val applicationLeaksMap = mutableMapOf<String, MutableList<LeakTrace>>()
  val libraryLeaksMap =
    mutableMapOf<String, Pair<LibraryLeakReferenceMatcher, MutableList<LeakTrace>>>()

  shortestPaths.forEachIndexed { pathIndex, shortestPath ->
    val inspectedObjects = inspectedObjectsByPath[pathIndex]

    val leakTraceObjects = buildLeakTraceObjects(inspectedObjects, retainedSizes)

    val referencePath = buildReferencePath(shortestPath, leakTraceObjects)

    val leakTrace = LeakTrace(
      gcRootType = GcRootType.fromGcRoot(shortestPath.root.gcRoot),
      referencePath = referencePath,
      leakingObject = leakTraceObjects.last()
    )

    val firstLibraryLeakMatcher = shortestPath.firstLibraryLeakMatcher()
    if (firstLibraryLeakMatcher != null) {
      val signature: String = firstLibraryLeakMatcher.pattern.toString()
        .createSHA1Hash()
      libraryLeaksMap.getOrPut(signature) { firstLibraryLeakMatcher to mutableListOf() }
        .second += leakTrace
    } else {
      applicationLeaksMap.getOrPut(leakTrace.signature) { mutableListOf() } += leakTrace
    }
  }
  val applicationLeaks = applicationLeaksMap.map { (_, leakTraces) ->
    ApplicationLeak(leakTraces)
  }
  val libraryLeaks = libraryLeaksMap.map { (_, pair) ->
    val (matcher, leakTraces) = pair
    LibraryLeak(leakTraces, matcher.pattern, matcher.description)
  }
  return applicationLeaks to libraryLeaks
}
```

# 10.REPORTING_HEAP_ANALYSIS

最后一步，报告分析结果，包括：HeapAnalysisSuccess，HeapAnalysisFailure。