---
layout: post
comments: false
categories: Java基础
date:   2019-06-01 00:30:54
title: ConcurrentHashMap源码阅读 Java 1.8
---

<div id="toc"></div>

市场进寒冬，危机感爆棚，温故而知新，学习压压惊。

> 本文基于Java 1.8.0_121-b13版本

ConcurrentHashMap在JDK8中彻底抛弃了JDK7的分段锁的机制，新的版本主要使用了Unsafe类的CAS自旋赋值+synchronized同步+LockSupport阻塞等手段实现的高效并发，代码可读性稍差。理想情况下table数组元素的大小就是其支持并发的最大个数。

## 搭建Debug环境

由于JVM本身有一些类中使用了ConcurrentHashMap，导致使用java.util.ConcurrentHashMap进行DEBUG学习时比较困难。

我这里使用的方式是将ConcurrentHashMap相关的代码拷贝生成自己的ConcurrentHashMap类。如我在自己项目的package map下创建如下文件并从java.util包中拷贝相关实现：

- ConcurrentHashMap

- ThreadLocalRandom

而后创建第一个基于我们拷贝出来的HashMap的单元测试：

```
@Test
public void testNewCurrentHashMap() throws Exception {
    ConcurrentHashMap hashMap = new ConcurrentHashMap();
    hashMap.put("key", "value");
    assertEquals(1, hashMap.size());
}
```

以此为入口我们可以愉快的进行DEBUG了。

## 一些常量和成员变量

### ConcurrentHashMap和HashMap都包含的成员变量

- MAXIMUM_CAPACITY，最大容量为`1 << 30`，其16进制表示`0x40000000`，10进制表示`1073741824`。注意Integer.MAX_VALUE的值是`0x7fffffff`, 10进制为`2147483647`。而`1 << 31`，即16进制`0x80000000`则变成了`-2147483648`，这也就可以理解MAXIMUM_CAPACITY为什么不是`1 << 31`,可是为什么不是`Integer.MAX_VALUE`呢？

- Node<K,V>[] table， 最底层的存储Key, Value的对象数据。

- entrySet是一个集合，其将存储所有的键值对

- DEFAULT_INITIAL_CAPACITY=16， HashMap的默认大小为16, 但是并不意味着new HashMap()会`new Node[16]()`，对于new HashMap()需要put第一个元素时才创建Node数组。

- DEFAULT_LOAD_FACTOR = 0.75f，这是影响HashMap性能的另外一个参数，加载因子(loadFactor)是为了衡量到底哈希表装的有多满。0.75是比较好的一个使得HashMap能够在时间空间花销上较平衡。

> 如果哈希表中的Entry超过了`当前容量*加载加载因子`。哈希表将会被`rehashed`.

- threshold，如果创建哈希表时传入了初始容量值，如`new HashMap(17)`，则会将threshold初始化为其后最近的2的指数，这里为32。
  HashMap提供了一个实现巧妙的方法`tableSizeFor(int cap)`来获取这个最近的2的指数。
  如果是`new HashMap()`则初始化时并不设置threshold值，直到put第一个元素时才根据计算threshold值，为`DEFAULT_INITIAL_CAPACITY*DEFAULT_LOAD_FACTOR=12`.

- size，记录的是HashMap中放入了多少键值对。

- modCount，记录HashMap结构发生变化的次数，结构发生变化的定义是`键值对数发生变化或发生了rehash`。利用这个值，当HashMap在程序中被并发访问时能够快速失败(fail-fast)。

- TREEIFY_THRESHOLD=8， Node[]中的Node其实可能是`Node`或`TreeNode`，默认如果当put的`键不相等但hashcode值相等`等于8个时，
  这个对应的Node将被转换为TreeNode（红黑树)。

> TreeNode是Node的子类，不过HashMap中的实现略有点绕，其继承自LinkedHashMap.Entry<K,V>，后者继承自HashMap.Node<K,V>

- UNTREEIFY_THRESHOLD=6，默认如果某个Node为TreeNode, remove某个元素时导致TreeNode的键值对等于6时，将转换为普通Node。

### ConcurrentHashMap独有的成员变量

- sizeCtl, 用来做`table`的初始化和扩/减容操作。

  ```
  -1表示table正在初始化
  -(1+正在进行扩减容的线程数)
  初始化前，0表示没有指定初始容量，其他值时表示初始容量
  正常状态时，其为扩容阈值
  ```

- Node<K,V>[] nextTable, 扩容期间，将table数组中的元素迁移到nextTable。

- DEFAULT_CONCURRENCY_LEVEL， 默认值为16，`1.8.0_121-b13`版本中并没有使用此变量，只是为了兼容以前的版本

- MIN_TRANSFER_STRIDE，默认值为16, 扩容线程每次最少要迁移的Node节点个数

- RESIZE_STAMP_BITS，默认为16，sizeCtl中用多少位来存储`generation stamp`

- MAX_RESIZERS，最大的扩容线程的数量，默认为`(1 << (32 - RESIZE_STAMP_BITS)) - 1`

- MOVED，TREEBIN，RESERVED定义了可用的常量-1, -2, -3

## HashMap中定义的静态类

### Node类

由于Node类的定义`static class Node<K,V> implements Map.Entry<K,V>`，其实现了Map.Entry<K,V>接口，我们可以断言：HashMap中的所有键值对都是Map.Entry。

#### 成员变量

其定义的成员变量与HashMap.Node是基本类似的，不过其用volatile关键字修饰了`val`和`next`。

```
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  volatile V val;
  volatile ConcurrentHashMap.Node<K,V> next;
}
```

#### 定义的方法

Node类实现了Map.Entry<K,V>定义的接口方法:

```
K getKey()
V getValue()
int hashCode()
V setValue(V newValue)
boolean equals(Object o)
```

注意的是这里实现的`V setValue(V newValue)`其实只是`throw new UnsupportedOperationException();`。

另外，相比HashMap.Node，其还定义了方法`ConcurrentHashMap.Node<K,V> find(int h, Object k)`。


### BaseIterator

BaseIterator是`KeyIterator`,`ValueIterator`和`EntryIterator`的基类。这几个类都各自定义了`next`方法。

BaseIterator实现的方法有`hasNext`，`hasMoreElements`和`remove`。BaseIterator继承了`Traverser`类。


#### Traverser

成员变量：

- Node数组`tab`，表示当前的table

- Node节点`next`

- TableStack的`stack`和`spare`，用来存储和恢复`ForwardingNodes`

方法：

`advance`方法是Traverser类的主要方法，其返回下一个可用的`Node`,如果没有则为null。

> ConcurrentHashMap中的`table`数组是由volatile关键字修饰，组引用是强可见的，但是其元素却不一定，并不能准确反映最新的实际情况。所以iterator不能反映最新的实际情况

## ConcurrentHashMap中定义的重要方法

### spread

ConcurrentHashMap中没有使用HashMap里的hash方法，而是自己定义了一个spread方法来计算哈希值，`HASH_BITS`的作用是屏蔽哈希值的第一位即符号位，spread返回的值为非负数：

```
static final int HASH_BITS = 0x7fffffff;
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

个人觉得ConcurrentHashMap的spread方法比起HasMap的hash方法要好，好在可测性方面。

```
@Test
public void testSpread() throws Exception {
    assertEquals(0xffff, ConcurrentHashMap.spread(0xffff));
    assertEquals(0x00000fff, 0x0fff0000 >> 16);
    assertEquals(0x0fff0fff, 0x0fff0000 ^ 0x00000fff);
    assertEquals(0x0fff0fff, ConcurrentHashMap.spread(0x0fff0000));
}
```

### put

键值对插入`V put(K key, V value)`，如果键已存在则用新的值覆盖旧的值，返回旧值;如果键不存在，则存入并返回NULL。

其调用了`putVal(key, value, false)`，第三个参数为onlyIfAbsent，`put`中调用`putVal`时直接设置第三个参数为false, 而`putIfAbsent`设置为true。

#### putVal

putVal中对多线程写入的并发处理使用了`CAS`和`synchronized`；对于`数据写入时正在扩容`的情况，通过for死循环，先帮助扩容，然后再根据情况安全写入数据。

- 如果key或value为空，则抛出`NullPointerException`异常。意味着ConcurrentHashMap不允许key或者value为null，这么设计的原因是在ConcurrentHashMap中，一旦value出现null，则代表HashEntry的key/value没有映射完成就被其他线程所见，需要特殊处理。

- 调用spread方法如`spread(key.hashCode())`来处理key值，以便后续确定其存在Node数组中的位置。

- 将节点写入的逻辑放到一个for死循环中

- 如果table(Node[])为空或者长度为0，先调用`initTable()`方法来初始化table。

- 根据`(table.length - 1) & hash`找到目前key在table中的位置，但是这里注意的是不是直接通过`table[(table.length - 1) & hash]`去获取，而是通过Unsafe去获取。
  在java内存模型中，我们已经知道每个线程都有一个工作内存，里面存储着table的副本，虽然table是volatile修饰的，但不能保证线程每次都拿到table中的最新元素，Unsafe.getObjectVolatile可以直接获取指定内存的数据，保证了每次拿到数据都是最新的。

- 如果其获取的Node节点为空，则直接创建新的Node节点，同时通过`CAS`（compareAndSwap）的方式将其赋值到`table`数组中。

- 如果获取的Node节点不为空，但是其Node的`hash`值为`MOVED`，意味着有其他线程正在进行扩减容，则其调用`helpTransfer`方法来先帮助进行扩减容

- 如果获取的Node节点不为空其Node的`hash`值不为`MOVED`，则将当前的Node节点用`synchronized`关键字锁起来，而后进行写入操作。
  写入时和HashMap类似，如果是Node节点，则写入到列表队尾，如果队列长度大于`TREEIFY_THRESHOLD`即默认为8，则调用`treeifyBin`将这个Node节点从列表转换为红黑树。否则如果是`TreeBin`，则调用TreeBin的`putTreeVal`方法插入到红黑树里。

  这里判断节点是不是Node节点有点特殊，其不是通过`instanceOf Node`来判断，而是通过Node的`hash`值如果大于等于0，则判断其为正常的Node节点。Node节点的`hash`值为-1表示`MOVED`，而如果为-1则表示`TREEBIN`，意味着当前的Node为TreeBin。TreeBin的实例本身不存储Key或Value，但是其成员变量是为TreeNode的`root`和`first`。其也定义了读写锁来控制对`写等待读`。


关于TreeBin里的`putTreeVal`中是如果保证数据插入安全的:

- 其会在插入数据前进行`lockRoot`方法调用，其先使用的是`CAS`机制`U.compareAndSwapInt(this, LOCKSTATE, 0, WRITER)`

- 如果上面的`CAS`机制没有成功，则调用`contendedLock`方法

- contendedLock方法中是一个for死循环，循环中对`lockState`进行判断和`CAS`操作，如从`WAITER`到`WRITER`的转换。

最后，putVal方法中当插入的是新的元素（oldVal == null)时，调用addCount方法`addCount(1L, binCount)`。

#### initTable

initTable方法初始化Node数组`table`变量，当前table为空。

- sizeCtl小于0时，意味着有其他的线程在做初始化，所以当前线程自旋直到`sizeCtl>0`

- 对sizeCtl进行CAS设置为-1，如果本线程设置值成功，则初始化`table`，初始化创建Node数组，如果sizeCtl大于0，则初始化的数组大小为sizeCtl，否则为DEFAULT_CAPACITY，即16

- 初始化Node数组后，将sizeCtl赋值为`table.length * 3/4`，即为`table.length - (table.length >>> 2)`

#### addCount

addCount方法主要做的事:

- 利用CAS来并发修改`baseCount`，竞争修改失败的线程将调用`fullAddCount`方法

- 如果已经有其他线程在执行扩容操作，则协助进行扩容

- 当前线程是唯一的或是第一个发起扩容的线程，则开始扩容

### get

get方法看起来并没有使用任何的同步机制，如锁或者CAS。但是使用了Unsafe使得能够从内存里获取最新的值。

- 先根据key的hash值查找其在Node数组中的位置`(tab.length - 1) & hash`

- 判断Node的第一个位置是否就是要查找的`key`值，如果是就直接返回

- 如果Node的`hash`值小于0，则调用Node的`find`方法来获取节点并返回，这里的Node可能是普通Node，TreeBin， ForwardingNode，ReservationNode

- 如果Node的第一个元素并不是要找的元素，且`hash`值大于等于0，就通过while循环来查找。

  ```
  while ((e = e.next) != null) {
      if (e.hash == h &&
              ((ek = e.key) == key || (ek != null && key.equals(ek))))
          return e.val;
  }
  ```

Node类的find方法通过简单的while循环来查找节点。

TreeBin的find方法：

- 从TreeBin的`e=first`节点出发，开始循环遍历`next`节点

- TreeBin的`lockState`值为`WAITER`（2）或者`WRITER`，则直接判断next节点是否为要找的节点，如果是则返回，否则`e=e.next`

- 如果TreeBin的`lockState`不为`WAITER`或者`WRITER`，则通过Unsafe的`compareAndSwapInt`来修改`lockState`的值为`lockState+4`。
  而后通过调用`root`成员变量的`findTreeNode`方法来查找相应节点。

- 如果TreeBin的`lockState`不为`WAITER`或者`WRITER`，定义了finnaly代码块，来修改`lockState`的值为`lockState-4`，如果lockState的值为`READER|WAITER`,则调用`LockSupport.unpark(w)`，这里的w为Thread成员变量`waiter`。

ForwardingNode的find方法:

- 使用了双层for循环死循环来查找节点

- 第一层for循环是对ForwardingNode的成员变量Node数组`tab = nextTable`进行循环来获取`Key`应该所处的Node槽位置

- 第二层for循环是对查找到的Node槽进行循环遍历来查找节点，在此循环内，如果节点的类型是`ForwardingNode`，将修改`tab`值为ForwardingNode的`nextTable`，返回第一层循环。

### transfer扩容

什么时候进行扩容呢？

- 如果新增节点之后，所在的链表的元素个数大于等于8，则会调用treeifyBin把链表转换为红黑树。在转换结构时，若tab的长度小于MIN_TREEIFY_CAPACITY，默认值为64，则会将数组长度扩大到原来的两倍，并触发transfer，重新调整节点位置。（只有当`tab.length >= 64`, ConcurrentHashMap才会使用红黑树。）

- 新增节点后，addCount统计tab中的节点个数大于阈值（sizeCtl），会触发transfer，重新调整节点位置。

如何进行扩容呢？

- 构建一个nextTable，大小为table两倍。这个操作是单线程完成的。这个单线程的保证是通过RESIZE_STAMP_SHIFT这个常量经过一次运算来保证的

- 把table的数据复制到nextTable中，这里允许多线程进行操作

- 在扩容过程中，依然支持并发更新操作；也支持并发插入


ConcurrentHashMap需要等扩容完之后，所有的读写操作才能进行，所以扩容的效率就成为了整个并发的一个瓶颈点，好在Doug lea大神对扩容做了优化，本来在一个线程扩容的时候，如果影响了其他线程的数据，那么其他的线程的读写操作都应该阻塞，其他线程闲着也是闲着，不如来一起参与扩容任务，这样人多力量大，办完事你们该干啥干啥，别浪费时间，于是在JDK8的源码里面就引入了一个ForwardingNode类，在一个线程发起扩容的时候，就会改变sizeCtl这个值。扩容时候会判断这个值，如果超过阈值就要扩容。

扩容时读写操作：

- 对于get读操作，如果当前节点有数据，还没迁移完成，此时不影响读，能够正常进行。 如果当前链表已经迁移完成，那么头节点会被设置成fwd节点，此时get线程会帮助扩容。

- 对于put/remove写操作，如果当前链表已经迁移完成，那么头节点会被设置成fwd节点，此时写线程会帮助扩容，如果扩容没有完成，当前链表的头节点会被锁住，所以写线程会被阻塞，直到扩容完成。

### replaceNode

这个方法的定义是`final V replaceNode(Object key, V value, Object cv)`，定义为final意味着不能被子类重写。

这个方法中考虑了多线程访问的安全性。 这个方法的逻辑是：

- 首先根据key来计算当前`key`所在`table`的位置，计算索引的方式`i = (n - 1) & spread(key.hashCode())`

- 如果当前节点为空则直接返回

- 如果当前节点的值为`moved`(-1)，则调用`helpTransfer`来辅助其他节点进行扩容

- 如果为正常的节点，则使用`synchronized`锁住当前节点，然后进行节点的替换或删除。特别注意点是`replaceNode`也可以用来做删除。

ConcurrentHashMap中有很多方法调用了`replaceNode`方法

- `remove`方法调用了`replaceNode(key, null, null)`

- `replace`方法调用了`replaceNode(key, newValue, oldValue)`

- `replaceAll`方法也调用了`replaceNode(key, newValue, oldValue)`

- `BaseIterator.remove`方法调用了`replaceNode(p.key, null, null)`

### size

`size`方法调用的是`sumCount`方法，sumCount方法中基于`baseCount`，然后对`CounterCell`数组`counterCells`进行累加。由于counterCells的元素并不是强可见的，所以size的计算并不完全准确。

size方法其可能会是大于int的最大值。所以建议使用的是`mappingCount`方法，因为其返回`long`类型的值：

```
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}
```

关于counterCells，其是个全局的变量，表示的是CounterCell类数组，其由`volatile`关键字修饰。如果存在两个线程同时执行CAS修改baseCount值，则失败的线程会继续尝试插入，并且会将计数存入`counterCells`中。

```
@sun.misc.Contended
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

`@sun.misc.Contended` 注解标识，这个注解是防止伪共享的。是 1.8 新增的。使用时，需要加上`-XX:-RestrictContended` 参数。

关于什么是伪共享：

- 缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。

- 多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

关于CPU的三级缓存：

- CPU Cache分成了三个级别：L1，L2，L3。越靠近CPU的缓存越快也越小

- L1缓存很小但很快，并且紧靠着在使用它的CPU内核。

- L2大一些，也慢一些，并且仍然只能被一个单独的 CPU 核使用。

- L3在现代多核机器中更普遍，仍然更大，更慢，并且被单个插槽上的所有 CPU 核共享。

- 最后，你拥有一块主存，由全部插槽上的所有 CPU 核共享。

- 当CPU执行运算的时候，它先去L1查找所需的数据，再去L2，然后是L3，最后如果这些缓存中都没有，所需的数据就要去主内存拿。

## 参考文章

- [ConcurrentHashMap源码分析（JDK8） 扩容实现机制](https://www.jianshu.com/p/487d00afe6ca)

- [深入浅出ConcurrentHashMap1.8](https://www.jianshu.com/p/c0642afe03e0)

- [JDK8的ConcurrentHashMap扩容 ](https://www.cnblogs.com/lfs2640666960/p/9621461.html)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
