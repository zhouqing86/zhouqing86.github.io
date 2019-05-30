---
layout: post
comments: false
categories: Java基础
date:   2019-05-28 10:30:54
title: HashMap源码阅读 Java 1.8
---

<div id="toc"></div>

市场进寒冬，危机感爆棚，温故而知新，学习压压惊。

> 本文基于Java 1.8.0_121-b13版本

因为此文较长，后续的针对成员变量和逻辑的分析需要花时间阅读。大部分人应该只是想大概了解下HashMap的使用，可以直接跳到使用章节。

## 搭建Debug环境

由于JVM本身有一些类中使用了HashMap，导致使用java.util.HashMap进行DEBUG学习时比较困难。

我这里使用的方式是将HashMap相关的代码拷贝生成自己的HashMap类。如我在自己项目的package map下创建如下文件并从java.util包中拷贝相关实现：

- HashMap

- LinkedHashMap

- AbstractMap

而后创建第一个基于我们拷贝出来的HashMap的单元测试：

```
@Test
public void testNewHashMap() throws Exception {
    HashMap hashMap = new HashMap();
    hashMap.put("key", "value");
    assertEquals(1, hashMap.size());
}
```

以此为入口我们可以愉快的进行DEBUG了。

## 一些常量和成员变量

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

## HashMap中定义的静态类

### Node类

由于Node类的定义`static class Node<K,V> implements Map.Entry<K,V>`，其实现了Map.Entry<K,V>接口，我们可以断言：HashMap中的所有键值对都是Map.Entry。

#### 成员变量

```
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  V value;
  Node<K,V> next;
}
```

其代码看出来很直观，Node类存储了键，值以及键的hash值。同时其有一个指向下一个Node元素。

#### 定义的方法

Node类实现了Map.Entry<K,V>定义的接口方法:

```
K getKey()
V getValue()
int hashCode()
V setValue(V newValue)
boolean equals(Object o)
```

这里需要略做解释的三个方法：

- hashCode方法将返回Node对象自身的HashCode，与成员变量`hash`无关，只和与`key`和`value`有关

- setValue，方法将设置新值给`value`并返回旧的值

- equals方法也是比较两个Node对象是否相等，键和值都相等就表示两个Node对象是相等的


#### 接口Map.Entry<K,V>

Map.Entry接口不仅仅提供了接口方法，其也提供了大量的default方法。

- comparingByKey()返回的是Comparator接口用来来比较键值对里的键，此接口为FunctionInterface且定义了`int compare(T o1, T o2);`方法

  ```
  public static <K extends Comparable<? super K>, V> Comparator<Map.Entry<K,V>> comparingByKey() {
       return (Comparator<Map.Entry<K, V>> & Serializable)
           (c1, c2) -> c1.getKey().compareTo(c2.getKey());
  }
  ```
- comparingByValue()返回的是Comparator接口用来来比较键值对里的值

- comparingByKey(Comparator<? super K> cmp)将基于传入的Comparator来进行键的比较，并返回封装后的Comparator接口

```
public static <K, V> Comparator<Map.Entry<K, V>> comparingByKey(Comparator<? super K> cmp) {
    Objects.requireNonNull(cmp);
    return (Comparator<Map.Entry<K, V>> & Serializable)
        (c1, c2) -> cmp.compare(c1.getKey(), c2.getKey());
}
```

> comparingByValue也同样可以传入一个Comparator参数

- getOrDefault(Object key, V defaultValue)，如果在哈希表中没有找到对象，就返回defaultValue

> 什么情况表示存在呢？`(v = get(key)) != null) || containsKey(key)`，这里看出如果containsKey方法是允许null的话，则可以为key为NULL的键值对设置value。

- forEach方法来对哈希表进行循环，并可以传入回调函数。

```
default void forEach(BiConsumer<? super K, ? super V> action) {
    Objects.requireNonNull(action);
    for (Map.Entry<K, V> entry : entrySet()) {
        K k;
        V v;
        try {
            k = entry.getKey();
            v = entry.getValue();
        } catch(IllegalStateException ise) {
            // this usually means the entry is no longer in the map.
            throw new ConcurrentModificationException(ise);
        }
        action.accept(k, v);
    }
}
```

> 这里getKey()和getValue()都可能会接收到异常，往往意味着在迭代时Key已经不在哈希表中了，应该是有其他线程并行执行了，所以抛出ConcurrentModificationException

- replaceAll方法是用传入的函数来重新计算value值，其基本逻辑与forEach类似，不过其传入的参数是`BiFunction<? super K, ? super V, ? extends V> function`。BiFunction能返回计算值。

  ```
  v = function.apply(k, v);

  try {
      entry.setValue(v);
  } catch(IllegalStateException ise) {
      // this usually means the entry is no longer in the map.
      throw new ConcurrentModificationException(ise);
  }
  ```

- putIfAbsent方法，此接口中实现也是没有考虑同步的

  ```
  default V putIfAbsent(K key, V value) {
      V v = get(key);
      if (v == null) {
          v = put(key, value);
      }

      return v;
  }
  ```

- remove方法注意传入的是两个参数，key和value

- replace(K key, V oldValue, V newValue)需要先找到相应key下的键值对，如果找不到返回false，找到了就`put(key, newValue)`

- computeIfAbsent函数有两个重载方法，返回的是Value，此方法是当key值如果不存在或者value为空时，利用传入的`Function<? super K, ? extends V> mappingFunction`函数来初始化value。
  如使用的例子: `map.computeIfAbsent(key, k -> new Value(f(k)));`或`map.computeIfAbsent(key, k -> new HashSet<V>()).add(v);`

> 如果参数是`BiFunction<? super K, ? super V, ? extends V> remappingFunction`,则可以使用Key, Value来计算并初始化键值对里的value.

- compute函数将根据参数中的函数来计算，如果结算的结果为null，则删除相应键值对，否则覆盖为新值。
  `V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction)`

- merge方法定义了只是key获取老值，并与传入的新值根据传入的函数进行合并。如果获取老值为NULL，就直接设置为新值。
  `V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction)`

### TreeNode类

关于红黑树的实现这里不做详细介绍。可另开一篇来介绍TreeNode的算法实现。

### 抽象类HashIterator

#### 成员变量

- 成员变量`next`和`current`为Node类型

- 成员变量expectedModCount是int型，其是为了fail-fast存在的

- index为int型，其指向的是当前的slot，即当前在HashMap的node数组的位置


#### 类构造函数

HashIterator构造函数完成了以下的初始化

- expectedModCount的值会初始化为HashMap类的modCount

- current的值为空，如果HashMap的Node数组table不为空，则将next赋值为第一个元素

- index初始值为0，如果HashMap的Node数组table不为空，则index值为1

#### 类方法

nextNode方法:

- 会先将HashMap的modCount与expectedModCount做比较，如果不相等，则应该有多线程操作此HashMap, fail-fast

- 然后更新成员变量`current`和`next`。经典的语句，两句话实现多个功能：

  ```
  if ((next = (current = e).next) == null && (t = table) != null) {
      do {} while (index < t.length && (next = t[index++]) == null);
  }
  ```

remove方法：

- 与`nextNode`方法类似，会先检查是不是有多线程操作对HashMap的修改，fail-fast

- 将`current`置为空

- 调用HashMap的`removeNode`方法

#### 子类

`KeyIterator`，`ValueIterator`以及`EntryIterator`都是HashIterator的子类，其区别只是对next()的实现不一样，
分别返回`nextNode().key`, `nextNode().value`以及`nextNode()`。

#### 使用类

`KeySet`,`Values`以及`EntrySet`分别使用了`KeyIterator`，`ValueIterator`以及`EntryIterator`作为其`iterator`方法的返回值。

而HashMap的方法`keySet`, `values`以及`entrySet`分别返回的是`KeySet`,`Values`以及`EntrySet`对象。

### HashMapSpliterator

#### Spliterator接口

Spliterator是一个可分割迭代器(splitable iterator)，Spliterator就是为了并行遍历元素而设计的一个迭代器，jdk1.8中的集合框架中的数据结构都默认实现了spliterator。

#### 子类

`KeySpliterator`,`ValueSpliterator`以及`EntrySpliterator`都是HashMapSpliterator的子类。
其区别是对trySplit和forEachRemaining方法的实现不一样。

## HashMap中定义的重要方法

### put

键值对插入`V put(K key, V value)`，如果键已存在则用新的值覆盖旧的值，返回旧值;如果键不存在，则存入并返回NULL。

其调用了`putVal(hash(key), key, value, false, true)`，第四个参数为onlyIfAbsent，第五个参数为evict(暂且不管它的用途)。

#### putVal

这个函数的详细处理过程为：

- 如果table(Node[])为空或者长度为0，先调用resize()方法来初始化table。

- 根据`(table.length - 1) & hash`找到目前key在table中的位置

- 如果table中对应位置的Node为NULL，则调用newNode方法来创建新Node, `new Node<>(hash, key, value, next)`, 这里next为NULL

- 如果table中对应位置的Node不为空，则需要找到键相匹配的键值对，首先判断第一个Node是不是正好是键值匹配的节点，如果不是则判断Node节点是TreeNode还是普通Node分别进行处理。

  ```
  // Node<K,V> p = tab[i = (table.length - 1) & hash]
  Node<K,V> e; K k;
  if (p.hash == hash &&
          ((k = p.key) == key || (key != null && key.equals(k))))
      e = p;
  else if (p instanceof TreeNode)
      e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
  else {
      for (int binCount = 0; ; ++binCount) {
          if ((e = p.next) == null) {
              p.next = newNode(hash, key, value, null);
              if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                  treeifyBin(tab, hash);
              break;
          }
          if (e.hash == hash &&
                  ((k = e.key) == key || (key != null && key.equals(k))))
              break;
          p = e;
      }
  }

  ```
- 如果是哈希表中已经存在的键值对，则用新值替换旧值，并注意调用了回调函数afterNodeAccess(e)

- 将modCount的值加一，表示哈希表的结构的变化次数加一

- 将size加一，如果size大于threshold，则需要进行resize()

- 调用回调函数`afterNodeInsertion(evict)`

### get

根据键获取值`V get(Object key)`,在方法中首先计算key的hash值，而后调用`getNode`方法。

#### getNode

- 先根据key的hash值查找其在Node数组中的位置`(tab.length - 1) & hash`

- 判断Node的第一个位置是否就是要查找的`key`值，如果是就直接返回

- 判断当前Node元素的`next`节点的类型，如果为TreeNode则调用TreeNode的`getTreeNode`方法
  如果为Node则循环遍历`next`节点查找与Key值相等的节点。

### resize

首先将重新计算threshold的值和新Node数组的长度：

- 如果HashMap内部Node数组的长度大于MAXIMUM_CAPACITY，即`1 << 30`，则值重新设置threshold的值为`Integer.MAX_VALUE`,resize函数返回不再后续处理。

- 如果内部Node数组的长度还可以翻倍扩展，则将threshold值设置为其当前threshold的两倍，且计算新Node数组的长度为老数组长度的两倍。

- 如果Node数组的长度为0，但threshold的值不为0， 则其应为类似于`new HashMap(123)`构造的新鲜HashMap，新Node数组的长度将为当前threshold, 且重新计算threshold值为当前threshold值*`loadFactor`

- 如果Node数组的长度为0，则设置threshold值为`DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY`，新Node数组的长度将为`DEFAULT_INITIAL_CAPACITY`

其次创建Node数组：

- 根据前面计算的新Node数组的长度创建新的Node数组

  ```
  Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
  ```

最后将目前Node数组中的每个值放入新的Node数组中：

- for循环遍历目前的Node数组，每个Node进行处理

- 如果当前Node的next为空，则计算器在新Node数组中的位置为`e.hash & (newCap - 1)`，由于newCap都是2的N次方，即等于`e.hash % newCap`，, 赋值并将老的Node数组中相应位置置为null

- 如果当前的Node为`TreeNode`，则调用方法`((TreeNode<K,V>)e).split(this, newTab, j, oldCap)`

- 如果当前的Node不是`TreeNode`，则进行两个分类，一个分类是Node在当前Node数组和老Node数组所处位置相同；另一个分类是Node在新的数组的位置是当前数组的位置+当前数组的长度。
  如何做分类呢，代码中用的比较巧，由于oldCap是2的N次方，`(e.hash & oldCap) == 0`就意味着e.hash值比当前Node数组长度`oldCap`小，那么意味着其在新老Node数组中此Node所处位置是相同的。
  `(e.hash & oldCap) != 0`的情况则意味着e.hash值比`oldCap`大，其在新的Node数组的位置就需要+oldCap。


## 使用

### HashMap的创建

```
new HashMap();
new HashMap(1000);
new HashMap(1000, 0.8f);
```

可以针对其写测试如下去测试`size()`，`capacity()`, `threshold`, `table`, `loadFactor`值初始化以及插入数据后的值的变化：

```
HashMap hashMap = new HashMap();
assertEquals(0, hashMap.size());
assertEquals(0.75, hashMap.loadFactor, 0.000001);
assertEquals(0, hashMap.threshold);
assertEquals(16, hashMap.capacity());
assertNull(hashMap.table);
hashMap.put(1,1);
assertEquals(1, hashMap.size());
assertEquals(16, hashMap.table.length);


hashMap = new HashMap(1000);
assertEquals(0, hashMap.size());
assertEquals(0.75, hashMap.loadFactor, 0.000001);
assertEquals(1024, hashMap.capacity());
assertEquals(1024, hashMap.threshold);
hashMap.put(1,1);
assertEquals(1, hashMap.size());
assertEquals(1024*0.75, hashMap.threshold, 0.000001);
assertEquals(1024, hashMap.capacity());
assertEquals(1024, hashMap.table.length);

hashMap = new HashMap(1000, 0.8f);
assertEquals(0, hashMap.size());
assertEquals(0.8, hashMap.loadFactor, 0.000001);
assertEquals(1024, hashMap.capacity());
assertEquals(1024, hashMap.threshold);
hashMap.put(1,1);
assertEquals(1, hashMap.size());
assertEquals(Math.floor(1024*0.8), hashMap.threshold, 0.000001);
assertEquals(1024, hashMap.capacity());
assertEquals(1024, hashMap.table.length);
```

关于设置初始化容量为1000，但threshold却初始化为1024，参考`tableSizeFor`方法详解。

而为什么threshold初始化为1024，插入数据后却变成了`1024*0.75`，参考`resize`方法详解。

另创建HashMap时候如果没有精确定义键值的类型，则key和value就可以为任意的Object对象，如:

```
HashMap map = new HashMap();
map.put("a", 1);
map.put(1, "a");
map.put(0, true);
map.put(true, 1.2);
map.put(false, "b");
```

更建议的使用方法是精确定义key,value的类型：

```
HashMap<String, Integer> map = new HashMap<>();
map.put("a", 1);
assertEquals(1, map.size());
```

注意不要使用`HashMap map=new HashMap<String, Integer>()`的方式，没有意义，并不会约束住Key和Value的类型。

### HashMap的自动扩容

当HashMap的size超过`threshold`时，将对HashMap进行扩容：即重新创建新的Node数组，数组的长度`一般`将为过去的两倍。也有不为两倍的情况，具体看`resize`函数详解。

```
HashMap map = new HashMap(2);
assertEquals(0, map.size());
assertEquals(0.75, map.loadFactor, 0.000001);
assertEquals(2, map.threshold);
map.put(1, 1);
assertEquals(1, map.threshold);
assertEquals(2, map.table.length);
map.put(2, 2);
assertEquals(3, map.threshold);
assertEquals(4, map.table.length);
map.put(3, 3);
map.put(4, 4);
assertEquals(6, map.threshold);
assertEquals(8, map.table.length);
```

### HashMap的遍历

有四种常用的遍历方式。我们先创建一个HashMap:

```
HashMap<Integer, String> idNameMap = new HashMap<>();
idNameMap.put(1, "Zhang San");
idNameMap.put(2, "Li Si");
```

- keySet

```
for (Integer key : idNameMap.keySet()) {
  ....
}
```

- entrySet

```
for (Map.Entry<Integer,String> entry : idNameMap.entrySet()) {
  ....
}
```

- values

```
for (String value : idNameMap.values()) {
  .....
}
```

- 使用迭代器来遍历

```
Iterator<Integer> keyIterator = idNameMap.keySet().iterator();
while (keyIterator.hasNext()) {
    Integer key = (Integer) keyIterator.next();
    .....
}

Iterator<String> valueIterator = idNameMap.values().iterator();
while (valueIterator.hasNext()) {
    String value = (String) valueIterator.next();
    .....
}

Iterator<String> entryIterator = idNameMap.entrySet().iterator();
while (entryIterator.hasNext()) {
    Map.Entry<Integer,String> entry = (Map.Entry<Integer,String>) entryIterator.next();
    .....
}
```


<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
