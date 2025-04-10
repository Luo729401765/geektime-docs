你好，我是郑雨迪。

在 JVM 中，每个 Java 对象都包含一个对象头，用于存储多种元数据，例如锁信息、哈希码、垃圾回收状态及指向其类的指针等。

这些数据主要是为了维护 JVM 的正常运行，并不包含任何应用程序相关的数据，因此，通常被视作运行 JVM 所需的额外内存开销。

目前，64 位 JVM 的默认对象头大小为 12 字节，其中包括 8 字节的标记字段（mark word）以及 4 字节的压缩类指针。据调研[\[1\]](https://wiki.openjdk.org/display/lilliput/Lilliput+Experiment+Results)，Java 对象在大多数场景下的平均大小约为 32 至 64 字节，其中对象头所占的比例大致在 37.5% 至 18.75% 之间。

这一显著的内存开销使得 JVM 饱受诟病，也因此催生了名为 Project Lilliput 的项目。

**Project Lilliput 的理念相对简单，核心在于压缩对象头的大小。**

为了说明这一点，我们可以假设Java对象的平均大小为 32 字节。根据这个假设，原本 12 字节的对象头若被压缩为 8 字节，便可实现 12.5% 的内存节省；若能够进一步将对象头压缩至 4 字节，内存节省则可达到 25%。

在 Lilliput 的相关介绍中[\[2\]](https://openjdk.org/jeps/450)提到，实际生产环境中，通过压缩对象头大小所节省的内存通常在 10% 到 20% 之间。这项技术的应用，不仅能够降低内存占用，还可以提高 JVM 的效率，进而优化 Java 应用程序的性能。

## 标记字段的结构

那么，Lilliput 是如何实现对象头的压缩呢？

要解答这个问题，我们需要先了解标记字段的结构及用途。在 JDK 24 之前，64 位 JVM 中标记字段的结构如下：

```
-------- -------- -------- -HHHHHHH HHHHHHHH HHHHHHHH HHHHHHHH -AAAALLL
```

这里的每个字母表示一个比特位。其中，前 25 位未被使用，接下来的 31 位用于存储本地哈希码（identity hash code）。需要注意的是，这个本地哈希码不同于 Java 类中可以覆写的 hashCode 方法所返回的哈希值。

当首次调用 System.identityHashCode 或 Object.hashCode 的默认本地实现时，JVM 会基于该 Java 对象的内存地址计算出一个 31 位整数，并将其缓存到该区域。从这次调用开始以及之后的所有调用，都会返回这个缓存值。

**这种机制确保了 Java 对象在其整个生命周期内始终返回相同的本地哈希码。**

紧接着这 31 位后，有一个未使用的比特位。再往后的 4 位用于存储该 Java 对象的 GC 年龄值，即对象经历的 YGC（Young Generation Collection）次数。当该值达到一定阈值时，JVM 会将该对象晋升至老年代。

最后的三个位则用于表示锁状态。其中，倒数第三位为偏向锁标志位，但自 JDK 18起已被废弃[\[3\]](https://bugs.openjdk.org/browse/JDK-8256425)。最低的两位则有以下四种状态组合：

- 00：表示对象已使用轻量级锁且处于加锁状态。
- 01：表示对象未上锁，这是 Java 对象的默认状态。
- 10：表示对象已使用重量级锁。是否上锁需访问对应的重量级锁。
- 11：表示 GC 标记，通常意味着对象已在 GC 过程中被迁移到其他内存位置。

除了 01（未上锁）的情况外，标记字段的高位在其他状态下都会存储一个指针：

```
PPPPPPPP PPPPPPPP PPPPPPPP PPPPPPPP PPPPPPPP PPPPPPPP PPPPPPPP PPPPPLLL
```

当对象使用轻量级锁时（LL=00），指针指向 Java 线程的栈帧中的锁记录，因此轻量级锁也被称为栈锁（stack locking）。

当使用重量级锁时（LL=10），指针则指向本地的重量级锁。当 GC 在垃圾回收过程中将对象复制到其他位置时（LL=11），指针会指向 Java 堆中的新位置。此外，在 JDK 18 之前，当对象使用偏向锁时（LLL=100），指针指向持有该锁的线程。

这里有两个关键点需要注意：

1. **指针对齐要求**。由于指针的低三位被锁状态位占用，因此指针所指向的内存地址必须是 8 字节对齐，即对齐到 2³ 字节边界。这也与 Java 对象需要按 8 字节对齐的要求相吻合（可通过 -XX:ObjectAlignmentInBytes=8 参数调整，取值范围为\[8, 256]）。
2. **替代标记字段（Displaced Mark Word）**。当标记字段的高位已经存储了非零值时，将其覆盖为指针会丢失原本携带的信息。例如已缓存的本地哈希码，如果对象的内存地址发生变化，直接重新计算哈希码将返回不同的值。
   
   为了避免这种情况，JVM 会将原始的标记字段复制到指针所指向的内存区域，称为替代标记字段。每当需要读取或写入原标记字段的内容时，JVM 必须先通过指针访问替代标记字段。这一额外的内存读取操作在某些应用场景下可能会影响到 Java 应用的性能。

可以看到，标记字段中有 25+1+1（偏向锁标记位）共 27 位未使用。Lilliput 的核心思路是将类指针整合到标记字段中，从而将对象头的大小从 12 字节压缩至 8 字节。

**然而，这一方案同样面临 2 个问题。**

#### 问题 1：未使用位不足32位。

针对这一问题，我们可以有两种思路：压缩本地哈希码或者压缩类指针。

压缩本地哈希码的思路相对简单。哈希码本质上是一串随机数，从理论上讲，即便始终返回 0 也符合哈希码的定义。然而，哈希码的质量直接影响许多算法的效率。

例如，HashMap 依赖哈希码（注意不一定是本地哈希码）来将对象分布到不同的桶中，以提升查找效率。均匀的哈希分布能够有效减少碰撞，从而使查找速度更快。因此，压缩哈希码可能影响实际性能。

另一种方案是进一步压缩类指针。目前，压缩类指针技术已经将 64 位的引用压缩为 32 位的数组索引，通过该索引定位到类数组中的元素。

进一步压缩类指针仅是减少索引的取值范围。例如，假设只使用前 25 位索引，JVM 仍然可以支持最多 2²⁵ = 33,554,432 个不同的 Java 类。这一数量通常已经足够。

同时，并非所有 Java 类都需要存储在类数组中。因为 Java 对象始终是某个类的直接实例，反过来，那些没有直接实例的类（例如抽象类、接口类）不会被 Java 对象引用，因此也不需要包含在类数组中。

JDK 24 已经引入了这一优化，将抽象类和接口类移出了元空间，单独存储[\[4\]](https://bugs.openjdk.org/browse/JDK-8338526)。

#### 问题 2：获取类信息的流程变化

在获取 Java 对象的类时，JVM 需要首先判断对象是否使用了替代标记字段。如果使用了替代标记字段，则需要解引用，并从替代标记字段中提取压缩后的类指针。

然而，由于获取对象类信息是一个非常高频的操作，即使上锁的对象占比很低，这种额外的判断也会带来开销。

另外一个更加严重的问题则是当并发 GC 尝试获取标记字段时，JVM 恰好将标记字段替换为替代标记字段。如果此时没有进行同步操作，并发 GC 很可能读到错误的标记字段。

为避免上述问题，**Lilliput 的设计者更倾向于避免依赖指针来访问替代标记字段。**这需要对 JVM 的轻量级锁和重量级锁机制进行重新设计，使其在不依赖标记字段指针的情况下，也能快速定位到相应的锁结构。

## 新轻量级锁

新轻量级锁（lightweight locking，-XX:LockingMode=2）并非 JDK 24 的全新特性。早在 JDK 21 的开发阶段，该特性便已引入，并一度作为默认选项[\[5\]](https://bugs.openjdk.org/browse/JDK-8291555)。

然而，在 JDK 21 正式发布前，出于稳定性考虑，这一选项被重置为非默认值。在后续的 JDK 版本中，新轻量级锁再次成为默认选项，并不断得到优化，例如引入了递归锁的优化[\[6\]](https://bugs.openjdk.org/browse/JDK-8319796)。

新轻量级锁与传统的旧轻量级锁在设计思路上类似，都基于无冲突假设，即假设大多数锁不会发生竞争冲突。如果锁出现冲突，JVM 仍会将新轻量级锁膨胀为重量级锁。

因此，新轻量级锁并不需要实现重量级锁的那些复杂机制，如线程的休眠、唤醒以及等待队列的管理。

新轻量级锁的核心任务是实现一个从 Java 对象到 Java 线程的映射关系，以便判定当前锁是否由执行线程持有。

传统的旧轻量级锁通过在对象头中存储一个指向 Java 栈帧特定位置的指针来实现这种映射。由于每个线程的内存空间是已知的，JVM 可以通过指针快速判断哪个线程持有锁。

相比之下，新轻量级锁采用了不同的实现方式。JVM 在线程本地空间中维护了一个锁栈（lock stack），用于记录当前线程持有的所有锁对应的 Java 对象。当线程获取锁时，JVM 会将对应的 Java 对象压入锁栈；当锁释放时，则将其从锁栈弹出。

具体来说，当需要判断某个锁的持有者时：

- 如果锁栈中包含该 Java 对象，说明锁由当前线程持有；
- 如果锁栈中不包含该 Java 对象，则说明该锁的持有者是其他线程，此时 JVM 需要将轻量级锁膨胀为重量级锁。

JVM 在线程本地空间中维护了以下数据结构，用于管理新轻量级锁的状态：

```
class LockStack {
  uint32_t _top;
  const uintptr_t _bad_oop_sentinel = -1;
  oop _base[8];
}
```

其中，\_bad\_oop\_sentinel 用于检测解锁越界的情况，\_top 可以视为 Java 对象引用数组 \_base 的索引，并始终指向当前栈顶空闲位置。基于此，锁的加锁和解锁操作可以通过以下伪代码来描述：

```
def lightweight_lock(obj):
  if 0<= _top < 8 and CAS(obj._mark, LOCKED):
    _base[_top++] = obj
  else: inflate to heavy monitor
  
def lightweight_unlock(obj):
  _top--
  CAS(obj._mark, UNLOCKED)
```

其中，CAS 为比较并交换原子操作，当交换成功时返回 true。

在递归锁的情况下，即：

```
synchronized (obj) {    // monitor_enter obj
  synchronized (obj) {  // monitor_enter obj
    ...
  }                     // monitor_exit  obj
}                       // monitor_exit  obj
```

中的第二行，旧轻量级锁可以通过对象头中的指针快速判断锁的持有者是否为当前线程。

因此，JVM 只需将该锁标记为递归锁，避免在解锁时（第四行）执行真正的解锁操作，从而防止将对应的 Java 对象状态设置为未上锁。

然而，对于新轻量级锁来说，加锁或解锁时的递归锁判断要复杂得多。

- 加锁时：当遇到已上锁的 Java 对象时，JVM 需要扫描整个锁栈来确定当前线程是否已持有该锁。这一步主要是为了判断锁是否属于递归锁。由于轻量级锁本身假定不存在锁冲突，而递归锁可以通过源代码规避，因而，加锁时遇到已上锁的对象并不常见，由其带来的全锁栈扫描的额外开销通常可以接受。
- 解锁时：为了避免错误地解锁递归锁，JVM 同样需要扫描整个锁栈，以确认该锁是否为递归锁。该操作是无条件的，即任何解锁操作均需要全锁栈扫描，这对 Java 应用的性能影响非常大。

目前，新轻量级锁采取了一种折中的递归锁处理办法——只检查锁栈的栈顶元素。当遇到已上锁的 Java 对象时，如果栈顶的 Java 对象与目标对象匹配，则认定为递归锁；如果不匹配， JVM 会直接膨胀该轻量级锁为重量级锁。

因此，对于如下递归锁的代码，新轻量级锁 a 会发生膨胀，导致执行效率远低于旧轻量级锁的实现。

```
synchronized (a) {      // monitor_enter a
  synchronized (b) {    // monitor_enter b
    synchronized (a) {  // monitor_enter a
      ...
    }                   // monitor_exit  a
  }                     // monitor_enit  b
}                       // monitor_exit  a
```

除了递归锁优化有所弱化外，新轻量级锁在即时编译的支持上还存在以下 2 个主要缺陷。

#### 缺陷1：解锁必须严格遵循加锁顺序

新轻量级锁要求解锁操作必须按照先加锁后解锁的顺序执行，类似于栈的“先入后出”原则。因此，即时编译器不能生成如下的解锁代码：

```
monitor_enter a
  monitor_enter b
  ...
  monitor_exit a
monitor_exit b
```

在旧轻量级锁的机制中，只要解锁之间没有其他逻辑插入，解锁顺序并不会影响程序的正确性。

而在新轻量级锁中，由于加锁和解锁操作被转化为栈的压栈与弹栈操作，一旦解锁顺序不对，将会导致错误解锁：栈中的某个 Java 对象的对象头处于未上锁状态，而栈外的某个 Java 对象仍然处于已上锁状态。

因此，**新轻量级锁的解锁顺序必须严格按照先入后出的顺序，否则会出现逻辑错误。**

#### 缺陷2：基于逃逸分析的锁消除更加复杂

考虑如下代码：

```
obj = new Obj
synchronized (obj） {  // moniter_exter obj
  ...                  // may trigger deoptimization
}                      // monitor_exit  obj
```

假设 obj 在方法中不发生逃逸，理论上即时编译器可以对对象分配和加锁、解锁操作进行优化，从而直接消除这些操作。

然而，在发生退优化（deoptimization）时，旧轻量级锁只需在退优化时重新分配对象，并简单地为其上锁；而新轻量级锁在重新分配对象后，不能直接加锁，而是需要判断该锁是否为栈顶锁。

如果在同步代码块中已经对其他 Java 对象加过锁，那么新对象的锁无法放置在栈顶位置，否则会出现与解锁顺序错误相同的问题。为避免出错，最稳妥的做法是直接为新对象使用重量级锁。

综上，作为旧轻量级锁的替代方案，新轻量级锁的主要优势在于移除了对象头中指向 Java 栈帧的引用，从而简化了对象头的结构。但它仍然存在一些明显的局限：

- 数量限制：每个线程最多支持 8 个轻量级锁，更改这一限制需要重新编译 JDK。
- 递归锁优化不足：新轻量级锁在处理递归锁时效率较低，容易导致锁膨胀。
- 对即时编译器不友好：新锁机制对解锁顺序有严格要求，并增加了锁消除的实现复杂度。

但是，在大多数实际应用中，复杂的锁场景并不常见，因此这些缺陷对整体应用性能的影响并不明显。

## 重量级锁的新映射方式

为了移除存储在 Java 对象头中的重量级锁引用，JDK 24 引入了一个全局并发哈希表来实现从 Java 对象到重量级锁的映射机制（通过参数 -XX:+UseObjectMonitorTable启用）[\[7\]](https://bugs.openjdk.org/browse/JDK-8315884)。

虽然并发哈希表的具体实现细节不在本文讨论范围内，但这种新的映射方式存在以下几个显著缺陷：

1. **内存开销增加。**这是最直接的缺点。在旧机制中，重量级锁的引用直接存储在 Java 对象头中，占用的是对象自身的内存空间。而在新的映射机制下，JVM 需要额外维护一对引用——一个指向 Java 对象，另一个指向重量级锁。此外，哈希表本身也会带来额外的内存开销。
2. **映射表的增删操作需要同步。**为了确保同一个 Java 对象始终映射到同一个重量级锁，任何对全局哈希表的插入或删除操作都必须是线程安全的，因此需要同步机制。  
   不过，这一缺点的影响相对较小，主要原因是锁膨胀和 GC 回收曾经加锁的对象这两种场景本身就很少出现，且操作过程本身也较为耗时，因此同步开销不算显著。
3. **查询映射时的内存访问成本更高。**在旧机制中，获取重量级锁的引用仅需一次内存访问，而且访问的内存通常已经缓存在当前 CPU 中。这种方式在性能上非常高效。
   
   而新机制需要在全局哈希表中查找目标对象的映射项，这个过程涉及多次内存访问。最坏情况下，可能需要遍历整个哈希表。如果在查找过程中其他线程对哈希表进行了插入或删除操作，当前 CPU 的缓存会失效，这将进一步增加了内存访问的延迟。

考虑到查询操作在应用程序运行时可能并不罕见，新的映射机制引入的额外内存访问和缓存失效会严重拖慢加锁效率。

此外，哈希表查询逻辑较为复杂，如果即时编译器将这部分逻辑内联到编译后的代码中，将显著增加生成代码的体积，从而对性能和内存占用造成不小的影响。

为缓解这些问题，JVM 在新的映射机制中引入了线程本地缓存。当 JVM 首次查询全局哈希表时，会将映射结果缓存到线程本地空间。

接下来，如果同一线程再次尝试对同一 Java 对象加锁，JVM 会优先查询线程本地缓存，直接返回对应的重量级锁，从而避免频繁访问全局哈希表，提升加锁性能。

具体数据结构如下：

```
class OMCache {
  struct OMCacheEntry {
    oop _oop = nullptr;
    ObjectMonitor* _monitor = nullptr;
  } _entries[8];
  const oop _null_sentinel = nullptr;
}
```

对应的查询伪代码如下：

```
def get_monitor(obj):
  int i = 0
  while _entries[i]._oop != null:
    if _entries[i]._oop == obj:
      return _entries[i]._monitor
    else: i++
  call slow_path_lookup
```

即时编译器会将上述查找映射的逻辑内联至编译后的代码中，作为快速路径来提高锁查询效率。如果在线程本地缓存中找不到对应的映射，编译代码会通过性能开销较大的 JNI 调用进入 HotSpot，执行全局哈希表查找并重填线程本地缓存。

然而，这一缓存机制存在一个关键缺陷：由于线程本地缓存中的任何 Java 对象引用都会被识别为 GC 根，垃圾回收时会将这些引用的对象无条件标记为存活，并进一步标记其直接或间接引用的对象。这会导致无法及时回收相关对象，从而增加内存占用。

为了解决这一问题，每次安全点触发时，JVM 都会清空线程本地缓存，以防止不必要的对象保留。但这样一来，当程序再次对同一个 Java 对象进行重量级锁加锁时，原本应当快速完成的编译代码将不得不再次进行 JNI 调用，重新进入 HotSpot 查找映射并填充缓存，导致性能开销增加。

综上，重量级锁的新映射方式以极大的查找代价移除了存储在 Java 对象头中的重量级锁引用。其导致的性能下降是 Lilliput 实际投产的瓶颈之一。

## Lilliput 1

在 JDK 24 中，Lilliput 的实现方案可以通过启用 -XX:+UseCompactObjectHeaders 来激活[\[8\]](https://bugs.openjdk.org/browse/JDK-8294992)。该方案依赖于之前提到的两种锁机制：新轻量级锁和重量级锁的新映射方式。启用 Lilliput 时，这两种机制将被强制启用。此外，Lilliput 对标记字段的结构进行了如下调整：

```
CCCCCCCC CCCCCCCC CCCCCCHH HHHHHHHH HHHHHHHH HHHHHHHH HHHHH--- -AAAASLL
```

压缩后的类指针仅占用 22 位，这使得可以索引的类的数量减少到 2²² =4,194,304 个。紧随其后的 31 位用于哈希码，接下来有 4 位为预留位（用于 Project Valhalla），4 位用于 GC 年龄，1 位为 GC 使用的 self-forwarded 位，以及 2 位用于锁状态。

除了节省对象头的内存开销，将类指针并入标记字段对 Java 应用的影响主要体现在以下几个方面：

1. **在对象分配时，写入对象头的操作从原本的两个写操作减少为一个写操作，并且伴随少量的计算操作。**

原本对象头的写入过程如下所示：

```
obj._mark     = 0b01                      // 写操作
obj._metadata = COMPRESSED_CLASS_POINTER  // 写操作
```

当启用 Lilliput 时，对象头写入则变更为：

```
tmp0          = COMPRESSED_CLASS_POINTER << 42
tmp1          = 0b01 | tmp0
obj._mark     = tmp1                      // 写操作
```

对于已知类的对象分配来说，COMPRESSED\_CLASS\_POINTER 是常量，因此，在即时编译中这些计算操作通常会被常量折叠，因而上述伪代码可以进一步精简为：

```
obj._mark = SHIFTED_CLASS_POINTER_OR_0B01 // 写操作
```

2. **读取 Java 类由原本的单个读操作转化为一个读操作及数个计算操作。**

为了说明这一点，我们以 instanceof 检查 final 类为例：由于 final 类没有子类，instanceof 判断会被优化为直接读取类指针并与目标类进行比较。原本该操作对应的伪代码如下所示：

```
tmp0          = obj._metadata             // 读操作
isInstance    = tmp0 == COMPRESSED_CLASS_POINTER
```

当启用 Lilliput 时，instanceof 判断则变更为：

```
tmp0          = obj._mark                 // 读操作
tmp1          = tmp0 >>> 42
isInstance    = tmp1 == COMPRESSED_CLASS_POINTER
```

上述伪代码等价于：

```
tmp0          = obj._mark                 // 读操作
tmp1          = tmp0 & 0xFFFF_FC00_0000_0000
isInstance    = tmp1 == SHIFTED_CLASS_POINTER
```

这是因为前者在第二行的位移操作中，实际上将低位全部清除，从而避免了额外的掩码操作。同时，由于应用代码通常会对对象的类进行进一步操作，因此即时编译器通常选择前者。

3. **读取 Java 对象的本地哈希码时，需要额外的掩码操作。**

原本标记字段的高位未被使用，默认值为 0，因此即时编译器可以省略读取本地哈希码后的掩码操作。然而，当启用 Lilliput 后，类指针的最低位可能不再为 0，因此在读取本地哈希码之后，必须进行额外的掩码操作：

```
tmp0          = obj._mark                 // 读操作
tmp1          = tmp0 >>> 11
hash          = tmp1 & 0x7FFF_FFFF
```

4. **不同类型的数组元素在内存中的起始位置有所不同。**

根据 Java 数组对象的内存布局，数组元素需要按照其类型宽度进行对齐。例如，long、double 类型的数组以及在 -XX:-UseCompressedOops 下的 Java 对象数组，其元素必须对齐到 8 字节。

由于数组对象需要额外的 4 字节来存储数组的长度，因此，上述数组的首个元素从 16 字节位置开始。相对而言，其他类型的数组，其首个元素通常从 12 字节位置开始。

可以看出，Lilliput 对 Java 应用的影响有好有坏，虽然它显著减少了 Java 对象头的大小，但其前置条件，特别是重量级锁的新映射方式，对存在锁冲突的 Java 应用并不友好。

Lilliput 的设计者希望在 JDK 25 解决新映射方式带来的性能问题，否则 -XX:+UseCompactObjectHeaders 将无法成为默认选项。同时，他们希望能在 JDK 26 废弃旧轻量级锁，使新轻量级锁成为唯一的选项。

然而，尽管 Lilliput 降低了Java对象头的大小，它并不一定能减少 Java 对象的总体大小。这是因为 Java 对象需要按 8 字节对齐。例如，对于一个空数组对象，原本对象头占据 12 字节，数组长度占据 4 字节，总大小为 16 字节，正好满足 8 字节对齐。

当启用 Lilliput 后，对象头的大小变为 8 字节，而数组长度仍然占据 4 字节，因此需要额外的 4 字节填充，使得总大小对齐到 8 字节。**因此，Lilliput 并不能优化空数组对象的内存开销。**

考虑到许多 Java 类库的重点类已经根据 8 字节对齐进行了优化，启用 Lilliput 很可能会导致生成带有 4 字节填充的 Java 对象，而无法有效优化内存开销。

接下来，我们将通过具体的案例来进一步说明这一现象。

## Java 对象的内存布局

我们之前提到过 OpenJDK 中的 Code Tools 项目中的一个小工具 jol[\[9\]](https://openjdk.org/projects/code-tools/jol/)，它能够分析 Java 对象在内存中的布局。举个例子，对于 java.lang.String 类，jol 可以根据用户的配置生成对应的内存布局：

```
$ java -jar jol-cli.jar internals java.lang.String
java.lang.String object internals:
OFF  SZ      TYPE DESCRIPTION               VALUE
  0   8           (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4           (object header: class)    0x00182b08
 12   4       int String.hash               0
 16   1      byte String.coder              0
 17   1   boolean String.hashIsZero         false
 18   2           (alignment/padding gap)   
 20   4    byte[] String.value              []
Instance size: 24 bytes
Space losses: 2 bytes internal + 0 bytes external = 2 bytes total
```

在这个输出中，第一列表示偏移量，第二列表示大小（单位为字节），第三列是字段类型，第四列是内容说明，第五列则给出具体数值。

例如，第 5 行：

```
  8   4           (object header: class)    0x00182b08
```

表示该项的偏移量为 8 字节，大小为 4 字节，类型为无，内容为压缩类指针，具体值为 0x00182b08。

第 10 行：

```
 20   4    byte[] String.value              []
```

表示该项的偏移量为 20 字节，大小为 4 字节，类型为 byte\[]，内容为字段 String.value，具体值为\[]，即空数组。

对于非空字符串，具体值将是一个非空数组。需要注意的是，由于 jol 的实现方式，它只会调用目标类的默认构造器，因此无法显示非空字符串的具体内容。

从这个示例可以看出，对于一个空字符串，JVM 需要创建两个对象：一个是 String 对象，另一个是 byte 数组对象，每个对象的对象头占用 12 字节。这两个对象的总内存占用为 24+16 字节，意味着对象头的内存开销占到了 60%。

当启用 Lilliput 时，String 的内存布局如下所示：

```
$ java -XX:+UnlockExperimentalVMOptions -XX:+UseCompactObjectHeaders \
  -jar jol-cli.jar internals java.lang.String
java.lang.String object internals:
OFF  SZ      TYPE DESCRIPTION               VALUE
  0   8           (object header: mark)     0x0008500000000001 (Lilliput)
  8   4       int String.hash               0
 12   1      byte String.coder              0
 13   1   boolean String.hashIsZero         false
 14   2           (alignment/padding gap)   
 16   4    byte[] String.value              []
 20   4           (object alignment gap)    
Instance size: 24 bytes
Space losses: 2 bytes internal + 4 bytes external = 6 bytes total
```

可以观察到，String 对象的对象头大小已经减少至 8 字节，但 JVM 在其末尾添加了 4 字节的填充，使得对象的总大小仍然为 24 字节。同样地，byte 数组对象的对象头也被压缩至 8 字节，并且末尾增加了 4 字节的填充。

综合来看，**Lilliput 通过压缩对象头节省的内存空间被用于对齐Java对象所产生的填充所抵消。**

当然，我们可以根据 Lilliput 的实现调整当前 Java 类库中关键类的设计。通过牺牲在未启用 Lilliput 时的内存开销，确保这些类的字段总大小是 8 字节的倍数。另一种可能的优化则是进一步将对象头的大小压缩至 4 字节，这也是 Lilliput 2 计划中的改进方向。

## Lilliput 2

首先需要明确的是，Lilliput 2 并不包含在 JDK 24 中，目前所有已知的信息均来自 OpenJDK Lilliput 项目的 wiki[\[10\]](https://wiki.openjdk.org/display/lilliput/Lilliput+2+-+4-byte+headers)。

Lilliput 2 的目标是将对象头大小进一步压缩至 4 字节，其中最重要的一项改进是压缩占据 31 位的本地哈希码。为此，Lilliput 2 提出了一种巧妙的本地哈希码获取方案，既确保了一致性，又尽可能节省空间。

具体来说，Lilliput 2 将 Java 对象分为三种状态：

- 未计算本地哈希码：对于这类 Java 对象，JVM 无需为其额外分配空间存储本地哈希码。
- 已计算本地哈希码且对象末尾有 4 字节填充：此时，Lilliput 2 将本地哈希码存储在对象的末尾填充区域。
- 已计算本地哈希码且对象末尾没有 4 字节填充：对于这类对象，Lilliput 2 要求，当对象未被 GC 移动时，通过对象的地址始终返回相同的哈希值。如果对象被 GC 移动，需要额外分配 8 字节的空间，其中 4 字节用于缓存基于对象之前内存地址计算得到的本地哈希码。此时，对象的状态相当于第二种情况，且可以通过访问对象末尾的 4 字节来获取缓存的本地哈希码。

由于这种本地哈希码缓存机制要求将 Java 对象分为三种状态，因此需要额外的两个状态位。这样，Java 对象的对象头布局将发生以下变化：

```
CCCCCCCC CCCCCCCC CCCHH--- -AAAASLL
```

在 Lilliput 2 中，压缩类指针的大小被压缩为 19 位，这使得可索引的类的数量进一步减少至 2¹⁹=524,288 个。紧接着，后面有两位用于表示哈希码状态，4 位预留给 Project Valhalla 的未使用位，4 位用于 GC 年龄，1 位为 GC 使用的 self-forwarded 标志位，另外还有两个锁状态位。

由于可索引的类的数量减少，我们需要额外的应变机制来处理当 Java 应用加载的类超过这一限制时的情形。Lilliput 2 提出的解决方案是，当前 8 位全为 1 时，剩余的压缩类指针将作为偏移量，JVM 将使用这个偏移量在 Java 对象对应位置存储类指针。

具体而言，在类加载过程中，JVM 会检查元空间是否有足够的空间来存储该类。如果没有足够空间，JVM 将在创建该类对象时，在该类字段之前以及该类的超类字段之后插入一个额外的字段，用来存储类指针。这个额外字段的偏移量将被存储在剩余的 11 位中，如下所示：

```
11111111 FIELDOFFSET HH--- -AAAASLL
```

由此，我们便解决了可索引的类的数目不足的问题。

## 总结

JDK 24 中的 Lilliput 1 实现方案存在一定的不完善之处，尤其是其依赖的新轻量级锁和重量级锁的新映射方式，后者对存在严重锁冲突的 Java 应用性能有较大影响。

Lilliput 1 在加载 Java 对象的类时需要执行额外的移位计算操作，但这种影响对 Java 应用的性能影响非常小。由于 JVM 的内存对齐机制，Lilliput 1 对现有 Java 类库的优化效果可能不显著，但这一问题预计将在未来的 Lilliput 2 版本中得到改进。

有兴趣尝试 Lilliput 1 的同学，可以通过使用 -XX:+UnlockExperimentalVMOptions -XX:+UseCompactObjectHeaders 选项进行实验。
<div><strong>精选留言（2）</strong></div><ul>
<li><span>Geek_f24e8e</span> 👍（0） 💬（0）<p>这个项目似乎有点得不偿失</p>2025-02-21</li><br/><li><span>陈时锐</span> 👍（0） 💬（0）<p>昨天刚翻lilliput的相关文档，今天就看到这篇文章</p>2025-02-14</li><br/>
</ul>