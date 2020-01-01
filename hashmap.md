# map

## hashmap

### 1、简介

1. 哈希值的桶和链表

   1. hash表：Node<K,V>[] table;

   2. 链表：

      ```java
      static class Node<K,V> implements Map.Entry<K,V> {
              final int hash;
              final K key;
              V value;
              Node<K,V> next;
      
              Node(int hash, K key, V value, Node<K,V> next) {
                  this.hash = hash;
                  this.key = key;
                  this.value = value;
                  this.next = next;
              }
          ......
      ```

      3、key和value可以空

### 2、问题

1. 为什么hashMap的初始容量必须是2的幂次方?**（jdk1.7）**

   原因：1、元素放置的位置为：**hashCode & (length - 1)**，如果不是2的指数幂，则length-1的二进制值不是全1，则hash表上会有位置是空的

   ​			2、jdk1.8在扩容的时候，不用rehash，而是通过hash & oldLength将原位置的链表拆分成高低两个链表，分别放在原位置和原位置+oldLength【感觉没啥用啊，都进行了&运算】

2. 死锁

   1. jdk1.7
      1. **put**操作的时候，冲突的时候，是将**新元素加入到链表的头部**【因为加入链表头部是最快的】
      2. 扩容**resize**的时候，会把链表的顺序颠倒，导致死锁
   2. **jdk1.8** 
      1. put的时候，将**新元素加入到链表的尾部**

3. 冲突

   1. jdk1.7出现的问题

      1. key的hashCode相同时会发生冲突，发生hash冲突时，hashMap使用链表的方式将相同的hashCode元素以链表的方式存储
      2. 如果发生冲突的元素过多，查找效率会降低，因为链表的查询效率是o(n)

   2. jdk1.8解决的冲突带来的性能问题

      1. 如果发生冲突的元素过多（阈值：8），这链表会转换成红黑树

      2. 阈值：TREEIFY_THRESHOLD=8的原因：泊松分布

      3. put方法：

         ```java
         final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                            boolean evict) {
                 Node<K,V>[] tab; Node<K,V> p; int n, i;
                 if ((tab = table) == null || (n = tab.length) == 0)
                     n = (tab = resize()).length;
                 if ((p = tab[i = (n - 1) & hash]) == null)
                     tab[i] = newNode(hash, key, value, null);
                 else {
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
                     if (e != null) { // existing mapping for key
                         V oldValue = e.value;
                         if (!onlyIfAbsent || oldValue == null)
                             e.value = value;
                         afterNodeAccess(e);
                         return oldValue;
                     }
                 }
                 ++modCount;
                 if (++size > threshold)
                     resize();
                 afterNodeInsertion(evict);
                 return null;
             }
         ```

      4. get方法：

         ```java
         final Node<K,V> getNode(int hash, Object key) {
                 Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
                 if ((tab = table) != null && (n = tab.length) > 0 &&
                     (first = tab[(n - 1) & hash]) != null) {
                     if (first.hash == hash && // always check first node
                         ((k = first.key) == key || (key != null && key.equals(k))))
                         return first;
                     if ((e = first.next) != null) {
                         if (first instanceof TreeNode)
                             return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                         do {
                             if (e.hash == hash &&
                                 ((k = e.key) == key || (key != null && key.equals(k))))
                                 return e;
                         } while ((e = e.next) != null);
                     }
                 }
                 return null;
             }
         ```

         

## hashTable

1、线程安全，但是每个方法都加上了synchronized，**性能差**

2、key和value都不能为空



## concurrentHashMap

1、线程安全，使用分段锁，锁的范围小，性能好

jdk1.7：Segment

jdk1.8：使用table的第一个元素作为锁



