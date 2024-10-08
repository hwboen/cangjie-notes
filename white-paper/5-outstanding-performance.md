# 卓越性能

仓颉在计算机基本测试 Benchmarks Game 上，比 Java、Go 的性能还要好很多。

## 静态编译优化

仓颉编译采用模块化编译，将编译过程分解成多个独立的模块，每个模块负责一个特定的编译任务。编译流程间通过 IR（Intermediate Representation）代码中间表示 作为载体，这样的好处是将源代码抽象表示，将 IR 作为不同编译阶段的桥梁，因此能够做到不同编译优化之间解耦，互不影响。

仓颉还会通过静态编译，将仓颉程序，核心代码直接编译成机器代码，加速程序运行速度。

仓颉静态编译中，添加了许多运行时联合优化：

- 堆上对象读写优化
  - 内存对齐：64 位的计算机，处理器一次处理 8 字节的数据，如果不对数据进行 8 字节的对齐，那么一个 4 字节的数据，处理器可能需要进行两次内存访问来获取数据。总而言之，内存对齐，就像流水线上将商品按固定间距摆放，以统一的标准提升效率。
  - 延迟加载：对于不常用的对象属性，延迟加载，在需要的时候再加载到内存中。
- 堆对象创建优化：
  - 对象池：使用对象池复用对象实例，避免频繁的内存分配和垃圾回收。
  - 内存池：预先分配一大块内存，并且分割为固定大小的小内存块，当需要创建对象的时候，通过指针操作直接分配一块空闲的内存块，而不是重新`malloc`
- 堆内存管理信号机制的优化：当发生特定事件，例如内存分配失败、内存违规访问，操作系统会通过信号机制向进程异步发送信号。堆内存管理的信号机制的优化指的是通过减少内存分配失败，减少内存违规访问等等，从而减少信号发生的频率。

仓颉静态后端（将 IR 编译成目标机器代码）通过向量化（一次性处理数据集合中的多个元素，可以简单理解批量处理数据）减少函数调用对性能的影响。

活跃作用域分析：静态分析指针引用关系，确定活跃作用域，从而可以更好的分配和管理内存，例如只在它活跃的时候分配内存。

引用模式的垃圾回收机制需要从一组“根”对象开始扫描可达的对象，这组根集合通常包括：全局变量、静态变量、栈上的局部变量。栈帧通常在函数执行完后自动释放内存，但会存在一种情况，栈上的局部变量被堆上的对象引用。仓颉精确的记录了栈上的引用，可以及时回收栈上的局部变量，从而减少垃圾回收根集合的数量，加快垃圾回收的效率。

程序中存在很多非必要的安全检查，在静态编译的优化下，可以生成 `fastPath`，跳过这些检查，直接执行目的操作。仓颉对于对象的创建和读写进行了`fastPath`优化，在编译对应访存操作的时候，生成快速路径和高效判断快速路径的指令，减少性能开销。

仓颉在全局用进行引用逃逸分析，对于未逃逸的函数内的局部变量，采用栈上分配。栈上分配的内存在函数执行完后会自动释放，无需进行额外的内存管理。栈上分配可以减少堆上内存的使用，降低垃圾回收的频率。另外，访问堆上的内存时，需要采用读写屏障来保证数据的一致性，而栈上数据是直接通过栈指针访问，无需读写屏障。

对于栈上内存，又可以额外采取 SROA，DSE 等优化措施，减少内存的访问次数。

- SROA (Scalar Replacement of Aggregates)：分解大的聚合体，采用标量访问的形式，避免对整个聚合体的内存访问，减少内存带宽。
- DSE（Dead Store Elimination）：编译时自动识别不必要的内存写入，有时候在写入操作后数据不再被读取使用，这个写入操作就是没有意义的，浪费了写入内存的消耗。

仓颉在编译时会将部分虚函数、接口函数的调用改写为 Direct Call,也被称为去虚化，也就是在编译时就确定了要调用的对象，减少了额外查找的开销。

## 值类型优化

值类型的局部变量优先栈上分配，读写无需 GC 相关屏障，能够给直接访问，并且利用了栈的优点，跟随栈的回退，自动释放内存。

OSR(On Stack Replacement) 栈上替换，允许在运行时将正在执行的字节码替换为优化后的版本。仓颉使用 OSR 技术，可以将部分值类型数据直接打散到寄存器中，例如值类型对象 SA，有 a,b 两个成员变量，在没有打散之前，每次访问 a 或 b，都需要加载整个对象，浪费了寄存器空间，打散之后，就相当于将 a，b 进行了解耦，访问 a 的时候不需要额外加载无用的 b。

深拷贝一个引用类型时，需要根据内存地址间接寻址，而使用值类型时，无需间接寻址，直接访问，效率要高得多。

## 全并发整理 GC

### 削减 GC STW 时间

仓颉的 GC 的底座：全并发(full concurrent)内存标记整理算法，延迟极低，碎片率极低，内存利用率高。全并发指的是不存在 STW，业务线程不会因为回收线程而阻塞。GC 通常分为以下几类：

- STW GC（Stop-The-World GC）：垃圾回收时，会暂停所有的业务线程，直到垃圾回收完成。
- 近似并发 GC（Mostly Concurrent GC）：垃圾回收与应用线程并发执行，在标记开始和清理结束阶段任然需要 STW，STW 时间显著减少。
- 全并发 GC（Full Concurrent GC）：垃圾回收的整个过程与应用线程并发执行，理论上不存在 STW。

近似并发 GC 在千级至万级的并发场景中，单次 STW 里需要完成上千了调用栈的扫描，可能使得 STW 时间超过 10 毫秒。现代的 120Hz 的移动场景，要求单帧的绘制时间小于 8 毫秒。而仓颉的全并发 GC 完成 GC 同步的耗时小于百微秒，典型情况下只需数十微秒就可以完成 GC 同步。GC 同步指的是在 GC 的特定过程中，没有应用线程的干扰。

仓颉全并发 GC 主要依赖以下两个关键因素：

- 安全点：安全点指的是能让程序安全暂停，不会破坏程序执行的状态，稍后可以完全恢复的位置。常见的有函数执行的开始和结束，循环的末尾，异常抛出点，这些位置程序状态相对稳定和清晰，可以安全的暂停和恢复。编译器在编译的时候会插入安全点检查代码，GC 线程需要同步的时候，先激活该线程的安全点检查，待线程执行到最近的安全点位置后，便可响应 GC 同步，改变自身的 GC 状态。
- 内存屏障：内存屏障顾名思议，它是一道屏障，就像交通信号灯，拦住一边的线程，放行另一边的线程，以此来达到可控的执行顺序。内存屏障在 GC 的场景下，用来解决 GC 线程与业务线程的数据竞争的问题。

### 减少内存碎片和削峰

- Mark-Sweep 算法：传统的标记清理算法，会造成大量的内存碎片。
- 复制算法：先对`from-space`中活对象进行标记，然后将标记的所有活对象移动到`to-space`，然后清理`from-space`中剩下的死对象。它有一个缺点就是得等所有的活对象都移动完毕才能释放`from-space`中的内存，这会造成“内存尖峰”的问题，因为在复制的过程中，可用内存并没有随着增加。
- 内存整理技术：内存整理技术在复制算法上进行改进，可以一边移动活对象，一边释放对应的内存块，因此可以消除内存尖峰的问题。

仓颉在内存整理的基础上，进一步优化，将堆内存划分为连续的小块`region`，在`from-space`中称为`from-region`，在`to-space`中称为`to-region`，当一个`from-region`完成搬移后便可立即释放。在内存压力大的时候，`from-region`和`to-region`还可以重叠，进一步降低内存峰值。

### bumping-pointer 极快的对象分配速度

通过维护一个或多个指针指向可用的内存空间，在分配对象的时候无需查找空余空间，直接分配对象。使用这项技术，在仓颉中平均每 10 个时钟周期便可完成一次内存分配。时钟周期又称 CPU 周期，指的是 CPU 执行一次指令的时间。

### 优化 GC 开销

仓颉全并发整理算法采用指针标记的算法来表示对象不同的引用状态。

- 未标记：未被 GC 跟踪引用状态，例如新创建的对象，这时的指针代表对象的真实地址。
- 在当前 GC 中被标记（current pointer）：在当前 GC 中的活对象，表示将要被移动到`to-region`，当内存屏障访问到`current-pointer`时，需要同步 GC 状态来判断该对象是否被转移，如果转移，则查询转发表获取新地址，否则，内存屏障需要将该对象转移到`to-region`，并更新被标记的指针。
- 在上次 GC 中被标记（old pointer）：在上次 GC 中标记但未被处理的对象，当内存屏障访问到`old-pointer`时，需要更新最新的引用状态。

使用指针标记，复用了原有指针的结构，没有引入新的开销。

标记阶段分为 enum 和 trace，enum 标记普通的对象，针对嵌套对象，还需要深入 trace 标记。

在仓颉的全并发整理算法中，被标记的指针只有在被内存屏障访问时，才会更新，它是 lazy 的。这个策略可以显著减少 GC 时间。

### 适配值类型

值类型不具有引用语义，没有指针的概念，它是数据本身，如果被引用类型包含，也是直接将数据嵌套在引用类型中的，访问的时候不需要通过指针，而是直接访问数据本身。由于没有指针的概念，GC 算法需要额外的数据结构和算法来处理值类型。比如在整理内存的时候，引用类型中的值类型，GC 除了需要更新对象的指针之外，还需要复制值类型的内联数据。GC 的工程复杂度大幅增加。

[上一篇： 轻松并发](./4-concurrency.md)

[下一篇： 轻量化运行时](./6-lightweight-runtime.md)
