---
layout:     post
title:      "Android meminfo 字段详解"
subtitle:   ""
date:       2022-10-10 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - Memory
---

对于Android内存的统计拆分，需要从系统到进程，再细分到内存分类。Android的内存通常被多个进程共享，一个进程占用的内存涉及到虚拟内存、物理内存、私有、共享、图形显存等等。想要详细统计一个进程的内存占用，就需要了解Android 的内存管理机制和对内存的分类拆解。

# 1.Android 内存管理

Android内存的管理核心是**paging**和**memory-mapping（mmap）**。

![img](/img/memory/mmap.png)



## 1.1 paging

Andoid系统中使用虚拟内存地址来索引内存，虚拟内存被划分为固定大小的page页，典型的页大小为4K。内存分配最开始都是在虚拟内存上分配，当需要访问这段内存的时候，如果发现它没有存在于物理内存上（即MMU不能找到这个虚地址va对应的物理地址pa），即发生了缺页（page fault），缺页有几种可能：

1. Bug，程序访问了它不应该访问的虚地址空间，android系统会触发访问不合法，kill掉进程。​
2. Va是合法的，但是这块va对应的pa还从来没有被分配出来过（例如你mmap的一段内存空间，但是从来没用过，这时第一次在这块内存上写入），这叫做lazy-allocation，这时系统会真正分配一段物理内存给你用，然后在页表上对应好这段pa和va。注意第一次写入这里才算真正占用了物理内存，mmap的分配并不算。​
3. Va是合法的，但是这va对应的pa内容当前并没有在物理内存上，而是被swap到一个backup的file上，这时系统会给这个page在pa上分配物理内存，然后将这块内容从文件读回到物理内存上（swap-in）。

![img](/img/memory/page.png)



## 1.2 swap & zram

典型的linux系统的虚拟内存都有swap操作，即一段物理内存在一段时间不用的时候，为了节省物理内存将他们备份到它的backup file上，一段时间后缺页时再换回。

但是在android上大多数情况是没有这套swap机制的，因为对于移动端的IO代价太大，所以大多数情况被映射到pa的page是不能被swap的。只有一种情况除外，即如果这段虚拟地址段具有backup file，并且当他被swap-in到pa后是只读的，那么它是有机会被swap-out回disk的，因为swap这种内存的代价很小，他们不会在物理内存上被更改，通常这类情况包括那些代码文件的mmap（如dex、so等）。

此外android上还使用了一种特殊的ZRAM机制来压缩一些物理内存上的page，但是并不把他们swap-out到disk上，而是仍然在ram中，这是一段被压缩了的page，系统会选择压缩一些page存储在内存中以腾出一些物理内存的占用。



## 1.3 mmap

Linux上一个重要的特性是memory mapping。如上面所述，va和pa之间通过mmu建立了一个对应关系，通过page fault来触发pa的分配，此外va还会对应一个backup file，作为swap-in/out时的备份存储。这个va和backup file之间的对应关系就叫做memory mapping，我们可以利用这个机制做文件读取。

第一种是映射文件到内存做读取， 这时提供一个文件句柄，然后将文件内容映射到一段虚拟内存地址空间，这样就做到了通过访问虚拟地址空间--造成缺页—swapin backup file的方式来根据需要读取文件。他相比传统的文件读取效率更高。 

第二种是创建一个匿名映射，匿名映射没有backupfile，只是单纯分配一块虚拟内存空间，这其实是android上调用new一个对象做的事情，我们new一块int的数组，事实上在new后很可能是通过mmap分配了一块虚拟内存空间，而只有当第一次写入的时候才触发缺页而占用真正的物理内存，所以统计android的物理内存占用不是看new了多少，或者文件映射了多少，而是实际上虚拟内存缺页了多少。



# 2.meminfo 数据详解

通过命令 dumpsys meminfo package_name&#124;pid 查看指定进程的内存使用情况。通过输出的信息，可以看出来应用在内存哪里分配出现了问题，比如native heap分配高，就要查一下自己的native部分的代码哪里分配后没有及时释放。

下面是system_server进程的meminfo。
![img](/img/memory/meminfo.png)



这里面可以看到pss、privatedirty、privateclean、swapped diry这几个重要的指标。

| **PSS**<br/>*Proportional Set Size*   | 进程实际占用的物理内存大小，但是android内存涉及到在系统中同其他进程共享一部分库（可能是so，可能是字体文件等的mmap），所以这里面会考虑这个因素计算你的进程平摊的这部分共享库的大小，这里的proportional就意味着统计了这个平摊后的物理内存。这是衡量进程占用内存的最真实指标。 |
| ------------------------------------- | ------------------------------------------------------------ |
| **RSS**<br/>*Resident Set Size*       | 常驻内存大小，是单个进程实际占用的内存大小；RSS 易被误导的原因在于， 它包括了该进程所使用的所有共享库的全部内存大小。对于单个共享库， 尽管无论多少个进程使用， 实际该共享库只会被装入内存一次。 对于单个进程的内存使用大小， RSS  不是一个精确的描述。<br>Rss=Shared_Clean+Shared_Dirty+Private_Clean+Private_Dirty |
| **USS**<br/>*Unique Set Size*         | 进程独占的内存                                               |
| **VSS**<br/>*Virtual Set Size*        | VSS= RSS+ 未分配实际物理内存<br/>内存的大小关系：VSS >= RSS >= PSS >= USS |
| **swappablePss**                      | 能够被共享的内存。<br/>*sharing_proportion* = (pss - uss) / (shared_clean + shared_dirty)<br/>*swapable_pss* = (*sharing_proportion* * shared_clean) + private_clean |
| **privateDirty**<br/>**privateClean** | 它们指的是每个进程独有的内存页面。PrivateDirty指的是进程独有的脏页面，即进程已经修改但尚未提交到磁盘的页面；PrivateClean指的是进程独有的干净页面，即进程已经提交到磁盘的页面。 |
| **sharedDirty**<br/>**sharedClean**   | 指的是进程和别的进程共享的内存页面。sharedDirty是该过程和至少一个其他过程所引用的映射中的页面，并且由这些过程中的至少一个过程写入的页面；sharedClean是指该进程和至少一个其他进程已引用但未由任何进程修改的映射中的页面。 |
| **Swapped Dirty**                     | 指的并不是被swap-out出去的内存，而是android系统中zram机制压缩掉的部分Private Dirty部分的不常用物理内存，这个重要度等同于Private Dirty，因为哪些Private Dirty会被压缩不能被控制，所以这部分多通常也是Private Dirty 的。 |

因为PSS中已经包含了Private Dirty和Private Clean，但是没包含swapped dity，所以最终衡量你的进程对物理内存的占用应该是取PSS+Swapped Dirty。

这里统计的所有值都是物理内存大小，即缺页的那些虚拟内存的大小，而不是真正虚拟内存的大小，注意这里看到的所有内存大小都是真真实实占着你的物理内存的。比如，new 1个int[1024]，在对这块内存写入之前它都不会占用物理内存并体现在这里。



## 2.1 数据来源

根据属性的不同，进程的虚拟地址空间被划分成若干个vma，每一个vma通过vm_next和vm_prev组成双向链表，链表头位于进程的task->mm->mmap。通过cat proc/<pid>/maps可以得到该进程的maps数据如下。每一行的vma对应着起始地址、结束地址、属性、偏移地址、主从设备号、inode编号和文件名。当通过proc接口读取进程的smaps文件时，内核会首先找到该进程的vma链表头，遍历链表中的每一个vma， 通过walk_page_vma统计这块vma的使用情况,最后显示出来。

![img](/img/memory/vma.png)



**smaps** 文件是基于maps 的扩展，它展示了一个进程的内存消耗，比同一目录下的maps文件更为详细。如下是maps中一条vma的详细数据，全部数据可通过cat /proc/<pid>/smaps获取。

> 7fce3a5000-7fceba4000 rw-p 00000000 00:00 0                            [stack]<br>
> Size:               8188 kB<br>
> KernelPageSize:        4 kB<br>
> MMUPageSize:           4 kB<br>
> Rss:                 156 kB<br>
> Pss:                 152 kB<br>
> Shared_Clean:          0 kB<br>
> Shared_Dirty:          4 kB<br>
> Private_Clean:         0 kB<br>
> Private_Dirty:       152 kB<br>
> Referenced:          156 kB<br>
> Anonymous:           156 kB<br>
> LazyFree:              0 kB<br>
> AnonHugePages:         0 kB<br>
> ShmemPmdMapped:        0 kB<br>
> FilePmdMapped:         0 kB<br>
> Shared_Hugetlb:        0 kB<br>
> Private_Hugetlb:       0 kB<br>
> Swap:                  0 kB<br>
> SwapPss:               0 kB<br>
> Locked:                0 kB<br>
> THPeligible:    0<br>
> VmFlags: rd wr mr mw me gd ac <br>



AMS中dump meminfo功能最终也是通过JNI调用libmeminfo读取进程smaps的节点。

![img](/img/memory/smaps.png)

![img](/img/memory/libmeminfo.png)



## 2.2 内存分类

**Native Heap**：C/C++层直接通过malloc分配的内存。

**Dalvik Heap/other**：Java堆分配的内存。

**Stack**：栈分配的内存。

**Ashmem**：进程的匿名共享内存 Anonymous Shared Memory。

**GFX dev**：通俗来说是显存，Android 显存和内存在同样的物理设备上，所以统计的总内存是包括显存的，至于adb如何知道哪些是显存，是因为gles和egl的so库在分配显存的时候也是使用的带backup文件的mmap，adb只是简单的统计了所有gles和egl的mmap将其视为显存，这部分通常就是在gpu上的资源，gpu上的资源由很多种，占比较多的就是texture，buffer，shader programe，通常是游戏应用。

**Other dev**：除显卡外所有其他硬件设备的mmap后的物理内存，可能包括声卡等，通常不多。

**.so mmap**：这个就是so库本身文件mmap占用的物理内存，随着程序运行会逐渐的读取so文件，造成和缺页的部分就是在物理内存产生占用，这部分大就是so库太大了，但是这部分因为有很多是readonly的mmap，所以有更大的机会被swap-out出去。

**.apk .dex oat .art mmap**：这些都是android 程序文件本身被mmap占用的内存，和so的性质差不多。

**Other mmap**：是所有除了上面的之外其他的所有非匿名方式的mmap。

**Unkonwn**：在meminf中它指除了部分已知的有名字的匿名映射（以anon:为前缀，例如"[anon:dalvik-"）之外的所有的匿名mmap，还有其他的未命名的vma（name字段不显示），因为无法分辨来源，就统计在unkown中。例如，看到anon:dalvik-*，那它一般是dalvik 虚拟机的分配，这个最终会被统计到meminfo的dalvik里。

**anon**：这个就是匿名mmap映射，除了部分已知的匿名映射如"[anon:dalvik-", "[anon:stack_and_tls:"等，其他匿名映射所占用的内存最终会被统计到meminfo的unkown里。

**anon:libc_malloc** ：这个是通过malloc方法进入的mmap，即所有的new，malloc调用。

**Kgsl-3d**：则是gles对显存硬件的虚拟内存映射，换句话说就是显存，meminf正是统计这个标签来获取gfx的大小。

更详细的拆解在[android_os_Debug.cpp](https://android.googlesource.com/platform/frameworks/base/+/master/core/jni/android_os_Debug.cpp)中。



## 2.3 Unknown

根据[android_os_Debug.cpp](https://android.googlesource.com/platform/frameworks/base/+/master/core/jni/android_os_Debug.cpp)中对smaps的分类，当前meminfo中对内存的统计，匿名映射除了下面这几个字段前缀的可以被统计以外，其他有名的匿名映射和没有名字的vma则被统计到Unknown字段中。

> "[anon:dalvik-", "[anon:stack_and_tls:", "[anon:libc_malloc]", 
>
> "[anon:scudo:", "[anon:GWP-ASan"



## 2.4 Alloc 内存

在数据面板右侧可以看到对已分配和释放内存的累计记录，包括native和dalvik分配（alloc）和释放（free）内存的统计。

![img](/img/memory/alloc.png)



## 2.5 Total

![img](/img/memory/total.png)

TOTAL是对每一列的汇总求和，其中Pss列有所不同，Pss列的TOTAL值是Pss Total和SwapPss两列之和。



## 2.6 App Summary

App Summary部分是对上述细分领域的聚合，提供更直观的统计数据。

**Java Heap**：（Dalvik Heap 的 Private Dirty 数据) + （ .art mmap 部分的 Private Dirty 和 Private Clean 数据) + getOtherPrivate ( OTHER_ART ) 。这里的 .art 是应用的 dex 文件预编译后的 art 文件，所以也是属于该应用的 JavaHeap。

**Native Heap**：Native Heap 的 Private Dirty 数据。

**Code**：.so .jar .apk .ttf .dex .oat 等资源加总。

**Stack**：getOtherPrivateDirty ( OTHER_STACK )。

**Graphics**：gl，gfx，egl 的数据加总。

**Private Other**：Native Heap + Dalvik Heap - 其他的 Private Dirty + Private Clean

**System**：Total Pss  -  Private Dirty - Private Clean。主要是系统占用的内存，如共享的字体、图像资源等。



# 3.Graphics memory

| **name**              | **description**                                              |
| --------------------- | ------------------------------------------------------------ |
| EGL mtrack            | 这个部分是gralloc的内存使用。主要是SurfaceView和TextureView的总和。它也包括了帧缓冲区，因此大小也会取决于framebuffers的尺寸。支持的屏幕分辨率越高，EGL mtrack的数目越高。在这个测试中，帧缓冲区的分辨率被降低了来确保比较好的性能。降低帧缓存的大小也会降低这些缓存需要的内存量。 |
| GL mtrack   & Gfx dev | GL和Gfx是驱动反馈的GPU内存，主要是GL纹理大小的总和，GL命令缓冲区，固定的全局驱动RAM消耗以及Shader。注意：客户空间驱动和内核空间驱动共享同一个内存空间。在某些Android版本上，这个部分会被重复计算两次，因此Gfx dev要比实际上使用的数值更大。 |



## 3.1 Gfx dev

Gfx dev的值是从/sys/class/kgsl/kgsl/proc/<pid>/gpumem_mapped下读取的，这里主要为/dev/kgsl-3d0占用的空间，gpu使用的内存。smaps中所有kgsl-3d0内存加起来等于gpumem_mapped中读到的内存。



## 3.2 EGL mtrack

主要是APP eglWindowSurface，包括normal HWUI based app windowsurface，SurfaceView，TextureView。这部分内存是从SurfaceFlinger 分配的，可以跟 dumpsys SurfaceFlingher 的 Alloated buffers 可以对应上。
