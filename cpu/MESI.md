## MESI协议

### MESI简介

不同CPU使用的缓存一致性协议是不同的，**x86处理器所使用的缓存一致性协议是基于MESI协议的**

MESI协议对内存数据访问的控制类似于读写锁，同一地址的读操作时并发的，写操作时独占的。

+ **Invalid(无效的，记为I)**:表示相应缓存行中不包含任何内存地址对应的**有效**副本数据。
+ **Shared(共享的，记为S)**:该状态表示相应缓存行包含相应内存地址所对应的副本数据。并且，其他处理器上的高速缓存中也可能包含相同内存地址对应的副本数据。**处于该状态的缓存条目，其缓存行中包含的数据与主内存中包含的数据一致。**
+ **Exclusive(独占的，记为E)**:该状态表示相应缓存行包含相应内存地址所对应的副本数据。**处于该状态的缓存条目，其缓存行中包含的数据与主内存中包含的数据一致。**
+ **Modified(更改过的，记为M)**:该状态表示相应缓存行包含对相应内存地址所做的更新结果数据。**处于该状态的缓存条目，其缓存行中包含的数据与主内存中包含的数据不一致。**

处理器在执行内存读、写操作时在必要的情况下会往总线（Bus）中发送特定的请求消息，同时每个处理器还**嗅探**（Snoop，也称拦截）总线中由其他处理器发出的请求消息并在一定条件下往总线中回复相应的响应消息

| 消息名                 | 消息类型 | 描述                                                         |
| ---------------------- | -------- | ------------------------------------------------------------ |
| Read                   | 请求     | 通知其他处理器、主内存当前处理器准备读取某个数据。该消息包含带读取数据的内存地址 |
| Read Response          | 响应     | 该消息包含被请求读取的数据。该消息可能是主内存提供的，也可能是嗅探Read消息的其他高速缓存提供的 |
| Invalidate             | 请求     | 通知其他处理器将其高速缓存中指定内存地址对应的缓存条目状态置为I，即通知这些处理器删除指定内存地址的副本数据 |
| Invalidate Acknowledge | 响应     | 接收到Invalidate消息的处理器必须回复该消息，以表示删除了其高速缓存上的相应副本数据 |
| Read Invalidate        | 请求     | 相当于Read和Invalidate合起来发送。通知其他处理器当前处理器准备更新（Read-Modify-Write，读后写更新）一个数据，并请求其他处理器删除其高速缓存中相应的副本数据。接受到该消息的处理器必须回复Read Response消息和Invalidate Acknowledge |
| Writeback              | 请求     | 包含写入主内存的数据及其对应的内存地址                       |

每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里

### 读取数据

+ Processor 0 找到的缓存条目的状态如果为M、E、S，那么该处理器可以直接从相应的缓存行中读取地址A所对应，而无需往总线中发送任何消息

+ Processor 0 找到的缓存条目状态如果为I，则说明该处理器的高速缓存中并不包含S的有效副本数据，此时Processor 0 需要往总线发送Read消息以读取地址A对应的数据，而其他处理器Processor 1(或主内存)则需要回复Read Response以提供相应的数据。如下表所示

  <table>
     <tr>
        <td colspan=2></td>
        <td>Processor 0</td>
        <td>Processor 1</td>
     </tr>
     <tr>
        <td rowspan=2>缓存条目</td>
        <td>当前状态</td>
        <td>I</td>
        <td>M/E/S</td>
     </tr>
     <tr>
        <td>目标状态</td>
        <td>S</td>
        <td>S</td>
     </tr>
     <tr>
        <td rowspan=2>消息</td>
        <td>发送</td>
        <td>Read 消息</td>
        <td>Read Response</td>
     </tr>
     <tr>
        <td>接受</td>
        <td>Read Response消息</td>
        <td>Read消息</td>
     </tr>
  </table>

Processor0 接受到Read Response消息时，会将其中携带的数据（包含数据S的数据块）存入相应的缓存行并将相应缓存条目的状态更新为S。Processor 0 接收到的Read Response消息可能来自主内存也可能来自其他处理器（Processor 1）

Processor1 嗅探到Read消息的时候，会从该消息中取出待读取的内存地址，并根据该地址在其高速缓存中查找对应的缓存条目。

+ 如果Processor 1 找到的缓存条目不为I,则说明该处理器的高速缓存中有待读取数据的副本。此时Processor 1会构造相应的Read Response消息并将相应缓存行所存储的**整块**数据（而不仅仅是Processor 0 所请求的数据S）"塞入"该消息。
+ 如果Processor 1找到的相应缓存条目的状态为M，那么Processor 1可能在往总线发送Read Response消息前将相应缓存行数据写入主内存。往总线发送Read Response之后相应缓存条目状态会被更新成S。
+ 如果Processor 1找到的高速缓存条目状态为I，那么Processor 0所接收到的Read Response消息就来自于主存

可见：在Processor 0读取内存的时候，即便Processor 1对相应的内存数据进行了更新并且这种更新停留在Processor 1的高速缓存中而造成**高速缓存与主内存中的数据不一致，在MESI消息的协调下，这种不一致不会导致Processor读取到的是一个旧值**

### **写入数据**

**任何一个处理器执行内存写操作时必须拥有相应数据的所有权**

在执行写操作时，Processor 0 会先根据内存地址A找到相应的缓存条目。

+ Processor 0的缓存条目状态为E或者M说明该处理器已经拥有相应数据的所有权，此时该处理器可以直接将数据写入相应的缓存行并将相应缓存条目的状态更新为M
+ Processor 0的缓存条目状态不为E、M，则该处理器需要往总线发送Invalidate消息以获得数据的所有权。其他处理器接受到Invalidate消息会将其高速缓存中相应的缓存条目更新为I并回复Invalidate Acknowledge消息。发送Invalidate消息的处理器必须在接收到其他处理器所回复**所有Invalidate Acknowledge消息之后**再将数据更新到相应缓存行中

<table>
   <tr>
      <td colspan=2></td>
      <td colspan=2>Processor 0</td>
      <td colspan=2>Processor 1</td>
   </tr>
     <tr>
      <td colspan=2></td>
      <td>场景1</td>
      <td>场景2</td>
      <td>场景1</td>
      <td>场景2</td>
   </tr>
   <tr>
      <td rowspan=2>缓存条目</td>
      <td>当前状态</td>
      <td>S</td>
      <td>I</td>
      <td>I</td>
      <td>M/E/S</td>
   </tr>
   <tr>
      <td>目标状态</td>
      <td>M</td>
      <td>M</td>     
      <td>I</td>
      <td>I</td>     
   </tr>
   <tr>
      <td rowspan=2>消息</td>
      <td>发送</td>
      <td>Invalidate</td>
      <td>Read Invalidate</td>
      <td>Invalidate Acknowledge</td>
      <td>Read Response 和 Invalidate Acknowledge</td>     
   </tr>
   <tr>
      <td>接受</td>
      <td>Invalidate Acknowledge</td>
      <td>Read Response消息 和 Invalidate Acknowledge</td>     
      <td>Invalidate</td>
      <td>Read Invalidate</td>     
   </tr>
</table>


+ Processor 0所找到的状态若为S，则说明Processor 1上的高速缓存可能保留了地址A对应的数据副本（场景1），此时Processor 0 需要往总线发送Invalidate消息。Processor 0 在接收到所有处理器所回复的Invalidate Acknowledge消息之后会将相应缓存条目的状态更新为E。此时Processor 0 获得地址A上的数据所有权。接着Processor 0便可以将数据写入相应的缓存行。并将相应的缓存条目的状态更新为M

+ Processor 0所找到的状态若为I，则表示该处理器不包含地址A对应的有效副本数据（场景2）。此时Processor 0 需要往总线发送Read Invalidate消息。Processor 0在接收到Read Response消息以及其他所有处理器所回复的Invalidate Acknowledge消息之后，会将缓存条目状态改为E,表示该处理器已经获得了相应数据的所有权。接着Processor 0可以往相应缓存行写入数据了并将缓存条目状态置为M

可见：**Invalidate消息和Invalidate Acknowledge消息使得针对同一个内存地址的写操作在任意一个时刻只能由一个处理器执行**，从而避免了多个处理器同时更新同一数据可能导致数据不一致的问题

### Writeback

发起 `Writeback` 一般是因为某个 CPU 的 Cache 不够了，比如需要新 Load 数据进来，但是 Cache 已经满了。就需要找一个 Cache Line 丢弃掉，如果这个被丢弃的 Cache Line 处于 `modified` 状态，就需要触发一次 `WriteBack`，可能是把数据写去主存，也可能写入同一个 CPU 的更高级缓存，还有可能直接写去别的 CPU。比如之前这个 CPU 读过这个数据，可能对这个数据有兴趣，为了保证数据还在缓存中，可能就触发一次 `Writeback` 把数据发到读过该数据的 CPU 上。

### 缓存一致性协议能够解决可见性问题吗？

假设 CPU 0 要写数据到某个地址，有两种情况：

1. CPU 0 已经读取了目标数据所在 Cache Line，处于 `Shared` 状态；
2. CPU 0 的 Cache 中还没有目标数据所在 Cache Line；

第一种情况下，CPU 0 只要发送 `Invalidate` 给其它 CPU 即可。收到所有 CPU 的 `Invalidate Ack` 后，这块 Cache Line 可以转换为 `Exclusive` 状态。第二种情况下，CPU 0 需要发送 `Read Invalidate` 到所有 CPU，拥有最新目标数据的 CPU 会把最新数据发给 CPU 0，并且会标记自己的这块 Cache Line 为无效。

无论是 `Invalidate` 还是 `Read Invalidate`，CPU 0 都得等其他所有 CPU 返回 `Invalidate Ack` 后才能安全操作数据，这个等待时间可能会很长。

**MESI协议缺点：处理器执行写内存操作时，必须等待其他所有处理器将高速缓存的相应副本数据删除并接收到这些处理器所回复的Invalidate Acknowledge/Read Invalidate消息之后才能将数据写入高速缓存**

为规避和减少这种等待造成的写操作延迟（Latency）硬件设计者引入了**写缓冲器和无效化队列**

### 写缓冲器

因为 CPU 0 这里只是想写数据到目标内存地址，它根本不关心目标数据在别的 CPU 上当前值是什么，所以这个等待是可以优化的，办法就是用 Store Buffer:

![preview](https://raw.githubusercontent.com/ppb2/note/main/imgs/StoreBuffer1.png)

引入写缓存器后，如果相应的缓存条目状态为E或者M，那么处理器可能会直接将数据写入相应的缓存行而无须发送任何消息。
如果相应的缓存条目为S，那么处理器会先将写操作的相关数据存入写缓冲器的条目中，并发送Invalidate消息。
等到所有 CPU 都回复 `Invalidate Ack` 后，再将对应 Cache Line 数据从 Store Buffer 移除，写入 CPU 实际 Cache Line。

除了避免等待 `Invalidate Ack` 外，Store Buffer 还能优化 `Write Miss` 的情况。比如即使只用一个 CPU，如果目标待写内存不在 Cache，正常来说需要等待数据从 Memory 加载到 Cache 后 CPU 才能开始写，那有了 Store Buffer 的存在，如果待写内存现在不在 Cache 里可以不用等待数据从 Memory 加载，而是把新写数据放入 Store Buffer，接着去执行别的操作，等数据加载到 Cache 后再把 Store Buffer 内的新写数据写入 Cache。

另外对于 `Invalidate` 操作，有没有可能两个 CPU 并发的去 `Invalidate` 某个相同的 Cache Line？

这种冲突主要靠 Bus 解决，可以从前面 MESI 的可视化工具看到，所有操作都得先访问 Address Bus，访问时会锁住 Address Bus，所以一段时间内只有一个 CPU 会操作 Bus，会操作某个 Cache Line。但是两个 CPU 可以不断相互修改同一个内存数据，导致同一个 Cache Line 在两个 CPU 上来回切换。

### 存储转发

比如现在有这个代码，a 一开始不在 CPU 0 内，在 CPU 1 内，值为 0。b 在 CPU 0 内：

```
a = 1;
b = a + 1;
assert(b == 2); 
```

CPU 0 因为没缓存 a，写 a 为 1 的操作要放入 Store Buffer，之后需要发送 `Read Invalidate` 去 CPU 1。等 CPU 1 发来 a 的数据后 a 的值为 0，如果 CPU 0 在执行 `a + 1` 的时候不去读取 Store Buffer，则执行完 b 的值会是 1，而不是 2，导致 assert 出错。

一个处理器在更新一个变量后紧接着又读取该变量的值的时候，由于该处理器先前对该变量的更新结果仍然停留在写缓冲器中，因此该变量相应的内存地址所对应的缓存行中存储的值是该变量的旧值。所以需要先从StoreBuffer中读取数据，如果StoreBuffer不存在才从高速缓存中读取数据

<img src="D:\陶志明\note-main\note-main\多线程\volatile\StoreForwarding.png" alt="CA88D5E5-F7E8-474C-9D14-D1761DA38863.png" style="zoom:120%;" />

### write barrier

```java
// CPU 0 执行 foo(), 拥有 b 的 Cache Line
void foo(void) { 
    a = 1; 
    b = 1; 
} 
// CPU 1 执行 bar()，拥有 a 的 Cache Line
void bar(void) {
    while (b == 0) continue; 
    assert(a == 1);
} 
```

对 CPU 0 来说，一开始 Cache 内没有 a，于是发送 `Read Invalidate` 去获取 a 所在 Cache Line 的修改权。a 写入的新值存在 Store Buffer。之后 CPU 0 就可以立即写 `b = 1` 因为 b 的 Cache Line 就在 CPU 0 上，处于 `Exclusive` 状态。

对 CPU 1 来说，它没有 b 的 Cache Line 所以需要先发送 `Read` 读 b 的值，如果此时 CPU 0 刚好写完了 `b = 1`，CPU 1 读到的 b 的值就是 1，就能跳出循环，此时如果还未收到 CPU 0 发来的 `Read Invalidate`，或者说收到了 CPU 0 的 `Read Invalidate` 但是只处理完 `Read` 部分给 CPU 0 发回去 a 的值即 `Read Response` 但还未处理完 `Invalidate`，也即 CPU 1 还拥有 a 的 Cache Line，CPU 0 还是不能将 a 的写入从 Store Buffer 写到 CPU 0 的 Cache Line 上。这样 CPU 1 上 a 读到的值就是 0，从而触发 assert 失败。

上述问题原因就是 Store Buffer 的存在，如果没有 Write Barrier，写入操作可能会乱序，导致后一个写入提前被其它 CPU 看到。

上面问题解决办法就是 Write Barrier，其作用是将 Write Barrier 之前所有操作的 Cache Line 都打上标记，Barrier 之后的写入操作不能直接操作 Cache Line 而也要先写 Store Buffer 去，只是这种拥有 Cache Line 但因为 Barrier 关系也写入 Store Buffer 的 Cache Line 不用打特殊标记。等 Store Buffer 内带着标记的写入因为收到 `Invalidate Ack` 而能写 Cache Line 后，这些没有打标记的写入操作才能写入 Cache Line。

```java
// CPU 0 执行 foo(), 拥有 b 的 Cache Line
void foo(void) { 
    a = 1; 
    smp_wmb();
    b = 1; 
} 
// CPU 1 执行 bar()，拥有 a 的 Cache Line
void bar(void){
    while (b == 0) continue; 
    assert(a == 1);
} 
```

此时对 CPU 0 来说，a 写入 Store Buffer 后带着特殊标记，b 的写入也得放入 Store Buffer。这样如果 CPU 1 还未返回 `Invalidate Ack`，CPU 0 对 b 的写入在 CPU 1 上就不可见。CPU 1 发来的 `Read` 读取 b 拿到的一直是 0。等 CPU 1 回复 `Invalidate Ack` 后，Ack 的是 a 所在 Cache Line，于是 CPU 0 将 Store Buffer 内 `a = 1` 的写入写到自己的 Cache Line，在从 Store Buffer 内找到所有排在 a 后面不带特殊标记的写入，即 `b = 1` 写入自己的 Cache Line。这样 CPU 1 再读 b 就会拿到新值 1，而此时 a 在 CPU 1 上因为回复过 `Invalidate Ack`，所以 a 会是 `Invalidate` 状态，重新读 a 后得到 a 值为 1。assert 成功。

### 无效化队列

写缓冲器的引入使得处理器在执行写操作的时候可以不等待Invalidate Acknowledge消息，从而减少了写操作延迟，使得写操作的**执行处理器**在其他处理器回复Invalidate Acknowledge消息的这段时间可以执行其他指令。从而减少了处理器的执行效率。

但是每个CPU的Store Buffer都是有限的，当Store Buffer被写满时，后续写入就必须等 Store Buffer 有位置后才能再写。就导致了性能问题。

之前提到 Store Buffer 存在原因就是等待 `Invalidate Ack` 可能较长，那缩短 Store Buffer 排队时间办法就是尽快回复 `Invalidate Ack`。于是解决办法就是为每个 CPU 再增加一个 p Queue。收到 `Invalidate` 请求后将请求放入队列，并立即回复 `Ack`。

<img src="D:\陶志明\note-main\note-main\多线程\volatile\无效化队列.png" alt="E5B30CFB-09FE-4873-81CF-D7553F0596C4.png" style="zoom:120%;" />

这么做会导致一些可见性问题。一个被 Invalidate 的 Cache Line 本来应该处于 `Invalidate` 状态，CPU 不该读、写里面数据的，但因为 `Invalidate` 请求被放入队列，CPU 还认为自己可以读写这个 Cache Line 而在操作老旧数据。从上图能看到 CPU 和 Invalidate Queue 在 Cache 两端，所以跟 Store Buffer 不同，CPU 不能去 Invalidate Queue 里查一个 Cache Line 是否被 Invalidate，这也是为什么 CPU 会读到无效数据的原因。

另一方面，Invalidate Queue 的存在导致如果要 Invalidate 一个 Cache Line，得先把 CPU 自己的 Invalidate Queue 清理干净，或者至少有办法让 Cache 确认一个 Cache Line 在自己这里状态是非 Invalidate 的。

### Read Barrier

```java
// CPU 0 执行 foo(), a 处于 Shared，b 处于 Exclusive
void foo(void) { 
    a = 1; 
    smp_wmb();
    b = 1; 
} 
// CPU 1 执行 bar()，a 处于 Shared 状态
void bar(void){
    while (b == 0) continue; 
    assert(a == 1);
} 
```

CPU 0 将 `a = 1`写入 Store Buffer，发送 `Invalidate` (不是 `Read Invalidate`，因为 a 是 `Shared` 状态) 给 CPU 1。CPU 1 将 `Invalidate` 请求放入队列后立即返回了，所以 CPU 0 很快能将 1 写入 a、b 所在 Cache Line。CPU 1 再去读 b 的时候拿到 b 的新值 1，读 a 的时候认为 a 处于 `Shared` 状态于是直接读 a，拿到 a 的旧值比如 0，导致 assert 失败。最后，即使程序运行失败了，CPU 1 还需要继续处理 Invalidate Queue，把 a 的 Cache Line 设置为无效。

解决办法是加 Read Barrier。Read Barrier 起作用不是说 CPU 看到 Read Barrier 后就立即去处理 Invalidate Queue，把它处理完了再接着执行剩下东西，而只是标记 Invalidate Queue 上的 Cache Line，之后继续执行别的指令，直到看到下一个 Load 操作要从 Cache Line 里读数据了，CPU 才会等待 Invalidate Queue 内所有刚才被标记的 Cache Line 都处理完才继续执行下一个 Load。比如标记完 Cache Line 后，又有新的 `Invalidate` 请求进来，因为这些请求没有标记，所以下一次 Load 操作是不会等他们的。

```java
// CPU 0 执行 foo(), a 处于 Shared，b 处于 Exclusive
void foo(void) { 
    a = 1; 
    smp_wmb();
    b = 1; 
} 
// CPU 1 执行 bar()，a 处于 Shared 状态
void bar(void) {
    while (b == 0) continue; 
    smp_rmb();
    assert(a == 1);
} 
```

有了 Read Barrier 后，CPU 1 读到 b 为 0后，标记所有 Invalidate Queue 上的 Cache Line 继续运行。下一个操作是读 a 当前值，于是开始等所有被标记的 Cache Line 真的被 Invalidate 掉，此时再读 a 发现 a 是 `Invalidate` 状态，于是发送 `Read` 到 CPU 0，拿到 a 所在 Cache Line 最新值，assert 成功。

除了 Read Barrier 和 Write Barrier 外还有合二为一的 Barrier。作用是让后续写操作全部先去 Store Buffer 排队，让后续读操作都得先等 Invalidate Queue 处理完。