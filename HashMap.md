
HashMap
=======

数据结构
========
数组 + 链表/红黑树,数组作为Hash容器,链表/红黑树消除不同key值Hash冲突。

put函数
=======
1. 初始化Hash容器数组
2. 判key为空，则添加到数组第一个元素的链表中
3. key已经存在，则更新原有的值
4. key不存在，若超容量，resize和rehash
5. 创建包含key和value的Entry，将Entry加入到链表的尾部
6. 若数组长度大于64，并且链表的长度大于8，将链表转化成红黑树。

get函数
=======
1. 计算Hash，确定在Hash容器中的位置
2. 链表遍历查询或者红黑树遍历查询

remove 函数
==========
根据key找到对应的节点，删除，(子类)善后处理，例如对LinkedHashMap需要将双向链表中的节点删除。

LinkedHashMap
=============
保存插入/访问的顺序

数据结构
=======
继承了HashMap，内部用双向链表保存插入的顺序。
accessOrder = true时，最近访问的Entry被置于双向链表的尾部。

TreeMap
=======
自定义compare函数，保存key值排序的顺序

数据结构
========
红黑树



LinkedHashMap 和 TreeMap的区别
==============================
LinkedHashMap保留节点插入的顺序，TreeMap保留节点key值比较后的大小顺序，使用场景不同。LinkedHashMap继承了HashMap，允许key值为null，TreeMap使用红黑树，需要比较key值，不能为null。

