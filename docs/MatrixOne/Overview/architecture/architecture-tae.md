# 存储引擎架构详解

MatrixOne 的存储引擎称为事务分析引擎（Transactional Analytical Engine，TAE）。

## 存储引擎架构

TAE 的最小 IO 单位被称为列块（Column Block），目前固定行数进行组织，对于 Blob 类型的列，我们进行了特别的处理。

TAE 以表的形式组织数据，每个表的数据结构化为一个 LSM（Log-structured Merge-tree）树。当前 TAE 的实现是一个三层 LSM 树，包括 L0、L1 和 L2。其中，L0 规模较小，全部存储在内存中，而 L1 和 L2 则持久化存储在硬盘上。

在 TAE 中，L0 由被称为临时块（Transient Block）的组件组成，数据在此并未进行排序。而 L1 由已排序块（Sorted Block）组成，包含了已经排序的数据。新的数据总是被插入到最新的临时块中。当插入的数据使得该块的行数超过限制时，该块将进行主键排序后作为一个已排序块刷新到 L1。如果已排序的块数量超过一个段（Segment）的最大数量，会使用合并排序（Merge Sort）方法按主键排序并写入 L2。

L1 和 L2 存储的都是按主键排序的数据。这两层的主要区别在于，L1 确保块内部的数据按主键排序，而 L2 确保每个段内的数据按主键排序。这里的段是一个逻辑概念，等同于其他实现中的行组（Row Group）或行集（Row Set）等。一个段可以根据更新（删除）操作的多少进行压缩，生成一个新的段。同时，也可以将多个段合并为一个新的段。这些操作都是由后台的异步任务完成，其调度策略主要权衡写放大和读放大之间的关系。

## 功能特性

- 适应 AP 场景：TAE 具备高效的数据压缩、查询效率高、聚合操作快、并发处理能力强等特性。因此，TAE 在处理大规模数据时具有更好的性能和扩展性，更适合分析处理和数据仓库等场景。
- 灵活的负载适配：通过引入 Column Family 的概念，可以灵活地适配负载。例如，如果所有列都属于同一个 Column Family（即所有列数据存储在一起），这将与数据库的 HEAP 文件非常类似，可以表现出类似于行存储的行为。如果每列都是独立的 Column Family（即每一列都独立存储），那就类似于列存储。用户只需在 DDL 表定义中指定，即可方便地在行存储和列存储之间切换。

## 索引与元数据管理

TAE，如传统列存储引擎一样，在块（Block）和段（Segment）级别引入了 Zonemap（最小/最大数据）信息。作为一个支持事务处理（TP）的存储引擎，TAE 实现了完整的主键约束功能，包括支持多列主键和全局自增 ID。每个表的主键都会默认创建一个主键索引，主要用于插入数据时去重以满足主键约束，以及根据主键进行过滤。

主键去重是数据插入的关键步骤，TAE 在以下三个方面达到平衡效果：

- 查询性能
- 内存使用
- 匹配数据布局

从索引粒度来看，TAE 可以分为两类：表级索引和段级索引。例如，可以有一个表级索引，或者每个段都有一个索引。由于 TAE 的表数据由多个段组成，每个段的数据都会经历从 L1 到 L3，从无序到有序的压缩/合并过程，这对表级索引不友好，因此，TAE 的索引建立在段级。

在段级索引中，有两种类型的段，一种可以进行追加修改，另一种不可修改。对于不可修改的段，段级索引是一个包含 Bloomfilter 和 Zonemap 的两级结构。对于可追加修改的段，它至少由一个可追加的块和多个不可追加的块组成。可追加的块索引是一个常驻内存的 ART-tree（Adaptive Radix Tree）结构和 Zonemap，而不可追加的则是 Bloomfilter 和 Zonemap。

![](https://community-shared-data-1308875761.cos.ap-beijing.myqcloud.com/artwork/docs/Tae/index-metadata.png?raw=true)

## 缓存区管理

在稳定的存储引擎中，精细的内存管理离不开缓存区管理。虽然缓存区管理在理论上只是一个最近最少使用（LRU）缓存的概念，但数据库不会直接使用操作系统的页缓存（Page Cache）来代替缓存管理器，尤其是对于事务处理（TP）类型的数据库。

在 MatrixOne 中，使用缓存管理器对内存缓冲区进行管理。每个缓冲区节点（buffer node）具有固定的大小，并分配给以下四个区域：

1. Mutable：用于存储 L0 的临时列块的固定大小缓冲区。
2. SST：用于存储 L1 和 L2 的块。
3. Index：存放索引信息的区域。
4. Redo log：存储事务未提交的数据，每个事务至少需要一个缓冲区。

每个缓冲区节点都有 Loaded 和 Unloaded 两种状态。当用户要求缓存管理器对节点进行固定（Pin）操作时，如果节点处于 Loaded 状态，引用计数会增加 1；如果节点处于 Unloaded 状态，它会从硬盘或远程存储中读取数据，并增加节点的引用计数。当内存空间不足时，系统会按照 LRU 策略从内存中移除一些节点，以释放空间。当用户取消对节点的固定（Unpin）时，只需关闭节点句柄。如果引用计数为 0，该节点将成为可能被移除的候选节点；如果引用计数大于 0，该节点将不会被移除。

## 预写日志与日志回放

在 TAE 中，我们优化了 redo log 的处理方式，使得列存储引擎的预写日志（WAL，Write Ahead Log）设计更加高效。相比行存储引擎，TAE 只需要在事务提交时记录 redo log，而不是每次写操作都记录。我们利用缓存管理器降低了输入输出（IO）的使用，特别是对于生命周期短且可能需要回滚的事务，我们避免了 IO 事件的发生。此外，TAE 还能够支持大型或长时间的事务。

TAE 的预写日志采用以下格式的日志条目头部（Log Entry Header）：

|项目|字节大小|
|---|---|
|GroupID|4|
|LSN|8|
|长度|8|
|类型|1|

事务日志条目包括以下类型：

|类型|数据类型|值|说明|
|---|---|---|---|
|AC|int8|0x10|已提交事务的完整写操作|
|PC|int8|0x11|已提交事务的部分写操作|
|UC|int8|0x12|未提交事务的部分写操作|
|RB|int8|0x13|事务回滚|
|CKP|int8|0x40|检查点|

大部分事务只有一个日志条目，但更大或更长的事务可能需要记录多个日志条目。因此，一个事务的日志可能包含一个或多个 UC 类型的日志条目和一个 PC 类型的日志条目，或者只有一个 AC 类型的日志条目。UC 类型的日志条目被分配到一个专门的组。

在 TAE 中，事务日志条目的有效载荷（Payload）包含多个事务节点。事务节点包括数据操作语言（DML）的删除、添加、更新操作，以及数据定义语言（DDL）的创建/删除表、创建/删除数据库等。每个事务节点都是已提交日志条目的子项，可以理解为事务日志的一部分。活动事务的节点共享固定大小的内存空间，并由缓存管理器进行管理。当剩余空间不足时，部分事务节点会被卸载（Unload），并将相应的日志条目保存在 Redo Log 中。在加载时，这些日志条目会被回放，即应用到相应的事务节点中。

Checkpoint 是一种安全点，在系统重启期间，状态机可以从该点开始应用日志条目。Checkpoint 之前的日志条目不再需要，并会在适当的时机被物理销毁。TAE 通过组（Group）来记录最后一个 Checkpoint 的信息，使得在系统重启时可以从最后一个 Checkpoint 开始进行日志回放。

TAE 的 WAL 和日志回放部分已被独立抽象为一个名为 logstore 的代码模块。它提供了对底层日志存取操作的抽象，可以适配不同的实现，从单机到分布式。在物理层面，logstore 的行为类似于消息队列。从 MatrixOne 0.6 版本开始，我们将演进为云原生版本，并使用独立的 shared log 服务作为日志服务。因此，在将来的版本中，TAE 的 logstore 将进行适当的修改，直接访问外部的 shared log 服务，而不依赖于任何本地存储。

## 事务处理

TAE 通过采用多版本并发控制（MVCC）机制来保证事务的隔离性。每个事务都配备一个由事务启动时间决定的一致性读取视图（读视图），因此事务内部读取的数据永不会反映其他并发事务所做的修改。TAE 提供了细粒度的乐观并发控制，在对同一行和同一列进行更新操作时才可能发生冲突。事务使用的是在事务开始时已存在的值版本，并在读取数据时不对其进行加锁。当两个事务试图更新同一个值时，第二个事务会因为写-写冲突而失败。

在 TAE 的架构中，一个表由多个段组成，每个段是多个事务共同产生的结果。因此，一个段可以表示为 `$[T{start}, T{end}]$`。`$[T{start}` 是最早事务的提交时间，`T{end}]$` 是最新事务的提交时间。为了能够压缩段并进行合并，我们需要在段的表示中增加一个维度以区分版本，即 `$([T{start} T{end}]`，`[T{create}, T{drop}])$`。`$T{create}$` 是段的创建时间，`$T{drop}$` 是段的删除时间。当 `$T{drop} = 0$` 时表示该段未被丢弃。块的表示方式与段 `$([T{start}, T{end}]`，`[T{create}, T{drop}])$` 相同。事务提交时，需要根据提交时间来获取它的读视图：`$(Txn{commit} \geqslant T{create})\bigcap（(T{drop}= 0)\bigcup（T{drop} > Txn{commit}))$`。

段的生成和变化由后台异步任务处理，为确保数据读取的一致性，TAE 将这些异步任务纳入到事务处理框架中，如下例所示：

![](https://community-shared-data-1308875761.cos.ap-beijing.myqcloud.com/artwork/docs/Tae/segment.png?raw=true)

L0 层的块 `$Block1 {L0}$` 在 `$t1$` 时创建，它包含了来自 `$Txn1`、`Txn2`、`Txn3`、`Txn4$` 的数据。`$Block1 {L0}$` 在 `$t11$` 开始排序，它的读视图是基线加上一个未提交的更新节点。排序和持久化块可能需要较长时间。在提交排序的 `$Block2 {L1}$` 之前，存在两个已提交事务 `$Txn5`、`Txn6$` 和一个未提交事务 `$Txn7$`。当 `$Txn7$` 在 `$t16$` 提交时，将失败，因为 `$Block1 {L0}$` 已经被终止。在 `$(t11, t16)$` 期间提交的更新节点 `$Txn5`、`Txn6$` 将被合并为一个新的更新节点，它将与 `$Block2 {L1}$` 在 `$t16$` 时一同提交。

![](https://community-shared-data-1308875761.cos.ap-beijing.myqcloud.com/artwork/docs/Tae/compaction.png?raw=true)

压缩过程会终止一系列块或段，并原子性地创建一个新的块或段（或建立索引）。与常规事务相比，此过程通常需要更长的时间，而我们不希望阻止涉及的块或段的更新或删除事务。因此，我们扩展了读视图，将块和段的元数据纳入其中。在提交常规事务时，一旦检测到写操作对应的块（或段）的元数据已更改（提交），该事务将失败。对于压缩事务，其写操作包括块（或段）的软删除和添加。在事务执行期间，每次写入操作都会检测是否存在写-写冲突。一旦冲突发生，事务将被提前终止。

## MVCC（多版本并发控制）

数据库的版本存储机制决定了系统如何保存不同版本的元组以及每个版本所包含的信息。TAE 创建了一个称为版本链的无锁链表，它基于数据元组的指针字段。版本链使得数据库能够准确地定位到所需元组的版本。因此，版本数据的存储机制是数据库存储引擎设计中的一个重要考虑因素。

一种解决方案是采用追加模式（Append Only）机制，将一个表的所有元组版本存储在同一个存储空间中。由于无法维护一个无锁的双向链表，版本链只能指向一个方向，要么是从旧到新（O2N），要么是从新到旧（N2O）。

另一种类似的方案被称为时光旅行（Time-Travel），它将版本链的信息单独存放，而主表则维护主版本的数据。

第三种方案是在主表中维护元组的主版本，并在单独的增量存储中维护一系列增量版本。在更新现有元组时，数据库从增量存储中获取一段连续的空间，用于创建一个新的增量版本，该增量版本仅包含被修改属性的原始值，而不是整个元组。然后数据库直接对主表中的主版本进行原地更新。

以上各种方案都有其特点，并对它们在 OLTP 工作负载中的表现产生影响。对于 LSM 树来说，由于其天然的追加模式结构，因此更接近第一种方案。但是版本链的链表可能不会显现。

TAE 目前选择了第三种方案的变体：

![](https://community-shared-data-1308875761.cos.ap-beijing.myqcloud.com/artwork/docs/Tae/mvcc.png?raw=true)

在大量更新的情况下，LSM 树结构的旧版本数据会导致大量的读放大。而 TAE 的版本链由缓冲区管理器维护，当需要被替换时，它会与主表数据合并，重新生成新的块。因此，在语义上，它是原地更新，但在实现上是写时复制，这对于云存储来说是必需的。重新生成的新块的读放大较少，这对于频繁更新后的 AP 查询更有利。
