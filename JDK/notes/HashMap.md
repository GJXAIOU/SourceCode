<!-- GFM-TOC -->

* [HashMap](#hashmap)
    * [1. HashMap 简介](#1-hashmap-简介)
       * [1.1 类定义](#11-类定义)
         * [1.1.1 实现和继承关系](#111-实现和继承关系) 
         * [1.1.2 HashMap引入](#112-hashmap引入)
           * [1.1.2.1 哈希表原理](#1121-哈希表原理)
           * [1.1.2.2 底层实现](#1122-底层实现)
             * [1.1.2.2.1 基本面貌](#11221-基本面貌)
             * [1.1.2.2.2 基本组成成员](#11222-基本组成成员)
             * [1.1.2.2.3 构造器](#11223-构造器)
           * [1.1.2.3 基本操作](#1123-基本操作)
             * [1.1.2.3.1 如何获取Hash值](#11231-如何获取hash值)
             * [1.1.2.3.2 获取桶索引位置](#11232-获取桶索引位置)
             * [1.1.2.3.3 put()方法](#11233-put方法)
             * [1.1.2.3.4 get()方法](#11234-get方法)
    * [2. HashMap 高级特性](#2-hashmap-高级特性)
      * [2.1 扩容机制](#21-扩容机制)
        * [2.1.1 扩容后位置变化](#211-扩容后位置变化)
        * [2.1.2 HashMap扩容为啥是2倍扩展](#212-hashmap扩容为啥是2倍扩展) 
      * [2.2 负载因子为什么是0.75](#22-负载因子为什么是0.75)  
      * [2.3 线程安全问题](#23-线程安全问题)
        * [2.3.1 线程不安全的表现](#231-线程不安全的表现)
        * [2.3.2 源码分析](#232-源码分析)
          * [2.3.2.1 新增节点顺序](#2321-新增节点顺序)
          * [2.3.2.2 JDK1.7死环现象分析](#2322-jdk1.7死环现象分析)
          * [2.3.2.3 JDK1.8扩容过程分析](#2323-jdk1.8扩容过程分析)
      * [2.4 树的特性](#24-树的特性)
<!-- GFM-TOC -->

# HashMap
## 1. HashMap 简介
### 1.1 类定义
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable 
```
### 1.1.1 实现和继承关系
   -  **继承**  抽象类**AbstractMap**,提供了 Map 的基本实现，使得我们以后要实现一个 Map 不用从头开始，只需要继承 AbstractMap, 然后按需求实现/重写对应方法即可;
   -  **实现**  Map接口,Map是"key-value键值对"接口;
   -  **实现**  Cloneable接口,即覆盖了clone()方法,能被克隆;
   -  **实现**  java.io.Serializable 接口,支持序列化,能通过序列化去传输;


### 1.1.2 HashMap引入
#### 1.1.2.1 哈希表原理
 -  **哈希表** 

>  [1]我们知道在数组中根据下标查找某个元素,一次定位就可以达到,哈希表利用了这个特性,哈希表的主干就是数组;</br>
  [2]比如我们要新增或查找某个元素,我们通过把当前元素的关键字通过某个函数映射到数组的某一个位置,通过数组下标一次定位就可完成操作;

>    存储位置 = f(关键字)
>        其中这个函数f()一般就称之为【哈希函数】,这个函数的设计好坏会直接影响到哈希表的优劣.举个栗子,比如我们要在哈希表中执行插入操作:
>    ![hash](HashMap.resource/hash.png)
>        查找操作:同理,先通过哈希函数计算 出实际存储地址,然后从数组中对应位置取出即可;
>
>     -  **哈希冲突** 

> 如果两个不同的元素，通过哈希函数得出的实际存储地址相同怎么办？
  也就是说，当我们对某个元素进行哈希运算，得到一个存储地址，然后要进行插入的时候，发现已经被其他元素占用了，
  其实这就是所谓的哈希冲突，也叫哈希碰撞。

 > 解决hash冲突的方法:</br>
  [1]开放地址法(发生冲突，继续寻找下一块未被占用的存储地址)</br>
  [2]再散列函数法</br>
  [3]链地址法(HashMap采用的就是这种方法)

#### 1.1.2.2 底层实现
##### 1.1.2.2.1 基本面貌
```
  HashMap采用链地址法,大概长下面的样子
```
 - 横着看

    ![hashmap-横着看](HashMap.resource/hashmap-%E6%A8%AA%E7%9D%80%E7%9C%8B.jpg)

 - 竖着看

![hashMap-竖着看](HashMap.resource/hashMap-%E7%AB%96%E7%9D%80%E7%9C%8B.png)

 > 简单来说，HashMap由数组+链表组成的，数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的，
  如果定位到的数组位置不含链表（当前entry的next指向null）,那么对于查找，添加等操作很快，仅需一次寻址即可；如果定位到的数组包含链表，
  对于添加操作，其时间复杂度依然为O(1)，因为最新的Entry会插入链表头部，因为最近被添加的元素被查找的可能性更高，
  而对于查找操作来讲，此时就需要遍历链表，然后通过key对象的equals方法逐一比对查找。
  所以，性能考虑，HashMap中的链表出现越少，性能才会越好；

##### 1.1.2.2.2 基本组成成员
 - table
```java
  // 我们称之为 桶(数组),初始化为16个;
  transient Node<K,V>[] table;
```
 - Node
```java
    //基本节点(构成链表的基本元素)
    // hash 当前节点 hash值;
    // key 当前节点的 key;
    // value 当前节点 value;
    // next 当前节点的下一个指向;
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
```
 - 重要属性:
```java
 // 初始化HashMap数组的长度
 static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
 // 最大数组的长度
 static final int MAXIMUM_CAPACITY = 1 << 30;
 // 默认负载因子为0.75,代表了table的填充度有多少;
 static final float DEFAULT_LOAD_FACTOR = 0.75f;
 //用于快速失败，由于HashMap非线程安全，在对HashMap进行迭代时，
 //如果期间其他线程的参与导致HashMap的结构发生变化了（比如put，remove等操作），
 //需要抛出异常ConcurrentModificationException
 transient int modCount;
 //实际存储的key-value键值对的个数
 transient int size;
 //当add一个元素到某一个桶位置时,若链表长度达到8就会将链表转化为红黑树;
  static final int TREEIFY_THRESHOLD = 8;
  //当某一个桶位置上的链表长度退减为6时,由红黑树转化为链表;
  static final int UNTREEIFY_THRESHOLD = 6;
 //阈值，当table == {}时，该值为初始容量（初始容量默认为16）；
 //当table被填充了，也就是为table分配内存空间后，threshold一般为 capacity*loadFactory。
 //HashMap在进行扩容时需要参考threshold，后面会详细谈到
  int threshold;
```
<table frame="hsides" rules="groups" cellspacing=0 cellpadding=0>
<!-- 表头部分 -->
<thead align=center style="font-weight:bolder; background-color:#cccccc">
     <tr>
          <td>变量名</td>
          <td>变量初始值</td>
          <td>变量值含义</td>
     </tr>
</thead>

<tbody>
    <tr>
        <td>DEFAULT_INITIAL_CAPACITY </td>
        <td>2^4</td>
        <td>HashMap默认初始化桶(数组)数量</td>
    </tr>
    <tr>
        <td>MAXIMUM_CAPACITY </td>
        <td>2^30</td>
        <td>HashMap最大桶(数组)数量</td>
    </tr>
    <tr>
        <td>DEFAULT_LOAD_FACTOR</td>
        <td>0.75</td>
        <td>负载因子(影响扩容机制)</td>
    </tr>
    <tr>
        <td>size</td>
        <td>无初始化值</td>
        <td>实际存储的key-value键值对的个数</td>
    </tr>
    <tr>
        <td>TREEIFY_THRESHOLD</td>
        <td>8</td>
        <td>链表长度>8时变化数据结构为红黑树</td>
    </tr>
    <tr>
        <td>UNTREEIFY_THRESHOLD</td>
        <td>6</td>
        <td>当桶位置上元素个数<=6时 退化数据结构由红黑树变为链表</td>
    </tr>

</tbody>
</table>

##### 1.1.2.2.3 构造器
 - 没有传入参数
```java
    // 默认构造器,负载因子为0.75;
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
 - 传入参数: 指定数组长度,指定负载因子
```java
  /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                             initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
    
```
   <font color="red"> **tableSizeFor方法** </font>
```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    功能: tableSizeFor的功能（不考虑大于最大容量的情况）
    是返回大于输入参数且最近的2的整数次幂的数;
    
    为什么上述的位运算能得到大于参数且最近的2的整数次幂的数呢?
    答:| 为或操作符, >>>y为无符号右移y位 ,空位补0,
       n |= n >>> y会导致它的高n+1位全为1,
       最终n将为111111..1 加1,即为100000..00,即2的整次幂;
```

  <font color="red">对于这个构造函数,主要讲两点:</font>

    [1]虽然说是自定义initialCapacity,但是实际上并不是你所想的那样的任意数组长度;
    经过tableSizeFor方法的处理之后,会得到一个大于输入参数且最近的2的整数次幂的数;
    比如你输入一个10,得到的是初始化数组长度为16的HashMap;
    
    [2]在常规构造器中，没有为数组table分配内存空间(有一个入参为指定Map的构造器例外),
    而是在执行put操作的时候才真正构建table数组;

#### 1.1.2.3 基本操作
##### 1.1.2.3.1 如何获取Hash值
 - 扰动函数</br>
   

  **[1]获取hashCode值** 
```java
   public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
```
  **[2]获取key的hash值** 
```java
  static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```
  <font color="red"> **扰动函数解析:** </font>
```
  上述代码中key.hashCode()函数调用的是超类Object类中的哈希函数,JVM进而调用本地方法,返回一个int值;
  理论上是一个int类型的值,范围为-2147483648-2147483647,
  前后加起来大概有40亿的映射空间,的确这么大的空间,映射会很均匀
  但是这么大的数组,内存中是放不下的。HashMap的初始化数组只有16
  ,所以这个散列值是无法直接拿来用的,所以要对hashCode做一些处理
  ,具体在下面小节中解答;
  
  扰动函数的结果: key的hash是hashCode中高位和低位的复合产物;
```
##### 1.1.2.3.2 获取桶索引位置
```
  JDK 1.8中是直接通过p = tab[i = (n - 1) & hash这个方法来确定桶索引位置的;
  
  [1]hash为扰动函数的产物;
  [2]n-1为当前hashmap的数组个数len-1;
  
  此时,我们会有诸多疑问,比如:
  【1】为什么要做&运算呢?
    
   答: 与操作的结果就是高位全部归零,只保留低位值,用来做数组下标访问。
   举一个例子说明,初始化长度为16,16-1=15 . 
   2进制(4字节,32位)表示就是 00000000 00000000 00000000 00001111 
   和 某散列值做与操作结果就是
   00000000 10100101 11000100 00100101
   00000000 00000000 00000000 00001111
 & -------------------------------------
   00000000 00000000 00000000 00000101 //高位全部归零,只保留尾部四位
   
  【2】为什么是 n-1 呢? 
   答: 源头就是HashMap的数组长度为什么要取2的整数幂?
   因为这样(n-1)
   相当于一个"低位掩码"; 
   这样能结合下面的&运算使散列更为均匀,减少冲突
   (相对其他数字来说,比如10的话,下面我们以00100101和00110001这两个数做一个例子说明)
    
    和15&      和10&       和15&    和10&
   00100101  00100101    00110001  00110001
   00001111  00001010    00001111  00001010
   &-------  --------    --------  --------
   00000101  00000000    00000001  00000000
   
   我们发现两个不同的数00100101 和0011001 分别和15以及10做与运算,
   和15做&运算会得到两个不同的结果(即会被散列到不同的桶位置上),而和10做与运算得到的是相同的结果(散列到同一个位置上),
   这就会出现冲突!!!!!
   
  【3】hash&(len-1) 和 hash%len 的效果是相同的;
   
   例如 x=1<<4,即2^4=16;
   x:   00010000
   x-1: 00001111
   我们假设一个数M=01011011 (十进制为:Integer.valueOf("01011011",2) = 91)
   M :  01011011
   x-1: 00001111
   &-------------
        00001011(十进制数为:11)
   
   91 % 16 = 11
   证明两者效果相同,位运算的性能更高在操作系统级别来分析;
  【4】为什么扰动函数是那样子的呢?
   如下图所示,h>>>16 之后和h 做异或运算得到的hash前半部分是h的高8位,
   后半部分是hash的高16位和低16位的复合产物;
```
![扰动函数](HashMap.resource/%E6%89%B0%E5%8A%A8%E5%87%BD%E6%95%B0.png)
##### 1.1.2.3.3 put()方法
 - put()方法
```java
    [1]调用put方法放入key-value键值对;
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
```
   [2] putVal()过程
```
![putVal](HashMap.resource/putVal.png)
  - 分析put过程:
    - [1]首先判断table是否为空或者为null,如果是,则初始化数组table;
    - [2]根据键值key计算hash值 并得到桶索引位置((n-1)& hash),如果table[i]=null,直接新建节点添加,转向[6],如果table[i]不为空,则转向[3];
    - [3]判断table的首个元素是否和key相同,如果相同的话直接覆盖value,否则直接转向[4],这里的相同指的是hashCode和equals
    - [4]判断table[i]是否为treeNode,即table[i]是否为红黑树,如果是红黑树,则直接在树中插入键值对,否则转向[5];
    - [5]遍历table[i](第i+1个桶),判断链表长度是否大于8,大于8的话将链表转换为红黑树,在红黑树中执行插入操作,否则进行链表的插入操作;遍历过程中若发现key已经存在直接覆盖value即可;
    - [6]插入成功,判断实际存在的键值对size是否超过了最大容量(阈值),如果超过,进行扩容;
>
   [1]这里重点说明下,在JDK1.6中index = hash % len 做与运算 ,在JDK 1.7/1.8中 变为了 i =(len-1) & hash,两者的作用是相同的;</br>
   [2]Put时如果key为null，存储位置为table[0]或table[0]的冲突链上(table为HashMap中存的数组),</br>
   如果该对应数据已存在，执行覆盖操作。用新value替换旧value，并返回旧value，如果对应数据不存在,则添加到链表的头上(保证插入O(1));
    put：首先判断key是否为null，若为null，则直接调用putForNullKey方法。若不为空则先计算key的hash值，
    然后根据hash值搜索在table数组中的索引位置，如果table数组在该位置处有元素，循环遍历链表，比较是否存在相同的key，
    若存在则覆盖原来key的value，否则将该元素保存在链头（最先保存的元素放在链尾）。若table在该处没有元素，则直接保存。

  - 拉链法的工作原理
```java
    HashMap<String, String> map = new HashMap<>();
    map.put("K1","V1");
    map.put("K2","V2");
    map.put("K3","V3");
    
    [1]新建一个HashMap,默认大小为1<<4(16);
    [2]插入Node<K1,V1> ,先计算K1的hash值为115,使用和(len-1)做&运算得到所在的桶下标为115&15=3;
    [3]插入Node<K2,V2>,先计算K2的hash值为118,使用和(len-1)做&运算得到所在的桶下标为118&15=6;
    [4]插入Node<K3,V3>,先计算K3的hash值为118,使用和(len-1)做&运算得到所在的桶下标为118&15=6,插在<K2,V2>后面.
```
![hashmap-put](HashMap.resource/hashmap-put.png)

##### 1.1.2.3.4 get()方法
![get](HashMap.resource/get.png)
 - 过程分析:
   - 根据Key计算出hash值;
   - 桶位置index =hash & (len-1)
     - 如果要找的key就是上述数组index位置的元素,直接返回该元素的值;
     - 如果该数组index位置元素是TreeNode类型,则按照红黑树的查询方式来进行查询;
     - 如果该数组index位置元素非TreeNode类型,则按照链表的方式来进行遍历查询;
## 2. HashMap 高级特性
### 2.1 扩容机制
```
 resize() 方法用于初始化数组(初始化数组过程由于涉及到类加载过程,将会放到JVM模块中进一步解释)或数组扩容，每次扩容后，容量为原来的 2 倍,并进行数据迁移。
```
#### 2.1.1 扩容后位置变化
```
  每次扩容，新容量为旧容量的2倍;

  扩容之后元素的位置
  [1]要么在原来位置上
  
  [2]要么在原位置再移动2^n的位置上;
```
 - **解释过程**
 ```
hashmap的原来桶数量为16,即n-1=15(二进制表示:1111)
[1]扩容前
n-1 :  0000 0000 0000 0000 0000 0000 0000 1111
hash1: 1111 1111 1111 1111 0000 1111 0000 0101
     &----------------------------------------
       0000 0000 0000 0000 0000 0000 0000 0101 (桶位置为5)
       
       
n-1 :  0000 0000 0000 0000 0000 0000 0000 1111       
hash2: 1111 1111 1111 1111 0000 1111 0001 0101
     &----------------------------------------
       0000 0000 0000 0000 0000 0000 0000 0101 (桶位置也是5)
两个hash值不同的对象,和n-1做&运算之后,得到相同的结果;
[2]扩容后
hashmap此时桶的数量变为32,即n-1=31(二进制表示:11111)
n-1:   0000 0000 0000 0000 0000 0000 0001 1111
hash1: 1111 1111 1111 1111 0000 1111 0000 0101
      &---------------------------------------
       0000 0000 0000 0000 0000 0000 0000 0101 (扩容后,桶位置依旧是5)
       
n-1:   0000 0000 0000 0000 0000 0000 0001 1111
hash2: 1111 1111 1111 1111 0000 1111 0001 0101
      &---------------------------------------
       0000 0000 0000 0000 0000 0000 0001 0101 (扩容后,桶位置变为之前5+2^4=21)
 ```
```
  元素在hashmap扩容之后,会重新计算桶下标,从上面的例子中可以看出来,hash1的桶位置在扩容前后没有发生变化,hash2的桶位置在扩容前后发生了变化;
```

>上述扩容过程中&运算的关键点就在于</font>
  扩容之后新的长度(n-1)转化为2进制之后新增的bit为1,</br>
  而key的hash 值所对应位置的bit是1 还是0,</br>
  如果是0的话那么桶位置就不会变化,依旧为index;</br>
  是1 的话桶位置就会变成index+oldCap;</br>
  通过if((e.hash&oldCap)==0)来判断hash的新增判断bit是1还是0;

#### 2.1.2 HashMap扩容为啥是2倍扩展
```
 综上所述，根本原因为2进制数字;
```

### 2.2 负载因子为什么是0.75
```
  通俗地说默认负载因子(0.75)在时间和空间成本上提供了很好的折中;
```
```
   JDK1.8-HashMap注释:
   * Because TreeNodes are about twice the size of regular nodes, we
     * use them only when bins contain enough nodes to warrant use
     * (see TREEIFY_THRESHOLD). And when they become too small (due to
     * removal or resizing) they are converted back to plain bins.  In
     * usages with well-distributed user hashCodes, tree bins are
     * rarely used.  Ideally, under random hashCodes, the frequency of
     * nodes in bins follows a Poisson distribution
     * (http://en.wikipedia.org/wiki/Poisson_distribution) with a
     * parameter of about 0.5 on average for the default resizing
     * threshold of 0.75, although with a large variance because of
     * resizing granularity. Ignoring variance, the expected
     * occurrences of list size k are (exp(-0.5) * pow(0.5, k) /
     * factorial(k)). The first values are:
     *
     * 0:    0.60653066
     * 1:    0.30326533
     * 2:    0.07581633
     * 3:    0.01263606
     * 4:    0.00157952
     * 5:    0.00015795
     * 6:    0.00001316
     * 7:    0.00000094
     * 8:    0.00000006
     * more: less than 1 in ten million

```
[泊松分布](http://www.ruanyifeng.com/blog/2015/06/poisson-distribution.html#comment-356111)</br>
[Stackoverflow-老外的牛顿二项式](https://stackoverflow.com/questions/10901752/what-is-the-significance-of-load-factor-in-hashmap)
![](https://upload-images.jianshu.io/upload_images/9402357-7a38574a79fa9c28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
```
    理想状态下,在随机哈希值的情况，对于loadfactor = 0.75 ,
  虽然由于粒度调整会产生较大的方差,桶中的Node的分布频率服从参数为0.5的泊松分布;

    一个bucket空和非空的概率为0.5，通过牛顿二项式等数学计算，得到这个loadfactor的值为log（2），约等于0.693.
  同回答者所说，可能小于0.75 大于等于log（2）的factor都能提供更好的性能，0.75这个数说不定是 pulled out of a hat
```

### 2.3 线程安全问题
#### 2.3.1 线程不安全的表现
- JDK 1.7: **并发死环，丢数据**
- JDK 1.8: **并发丢数据**

#### 2.3.2 源码分析
##### 2.3.2.1 新增节点顺序
 - JDK 1.7
   - **<font color='red'>头插法</font>** : 新增节点插入链表头部
 - JDK 1.8
   - **<font color='red'>尾插法</font>** : 新增节点插入链表尾部
- <b>源码分析与参考</b>:
   - [HashMap到底是插入链表头部还是尾部](https://blog.csdn.net/qq_33256688/article/details/79938886)
##### 2.3.2.2 JDK1.7死环源码分析
```
   说到根本就是扩容之后,链表元素顺序发生变化导致的;
   JDK 1.7 中扩容之后 链表顺序会倒置（采用头插法）;
```

``` java
    JDK 1.7 扩容中重新安排Entry操作:

    void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}              
```
###### 2.3.2.2.1 扩容后节点顺序反向存储

```
   在分析HashMap的线程不安全之前,我们先看一下上面扩容重组过程中的一段扩容后重新分配bucket的代码,并结合一小段自定义代码来理解这个过程
```
``` 
   我最开始看上面那段代码的时候，死活看不懂,最后自己模拟了这个过程才明白。咱们先看下 我的模拟代码:
```
  - Entry :
```
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Data
public class Entry<T> {
    int val;
    Entry next;

    public Entry(int val) {
        this.next = null;
        this.val = val;
    }
}

 public static void main(String[] args) {
        Entry e = new Entry(1);
        e.next = new Entry(2);
        e.next.next = new Entry(3);
        e.next.next.next = new Entry(4);
        Entry[] newTable = new Entry[6];
        while (e != null) {
            // [1]
            Entry next = e.next;
            // [2]
            e.next = newTable[5];
            // [3]
            newTable[5] = e;
            // [4]
            e = next;
        }
    }
```
![](/about/media/pic/hashmap-1.7-resize-01.png)
![](/about/media/pic/hashmap-1.7-resize-02.png)
![](/about/media/pic/hashmap-1.7-resize-03.png)
![](/about/media/pic/hashmap-1.7-resize-04.png)
![](/about/media/pic/hashmap-1.7-resize-05.png)
![](/about/media/pic/hashmap-1.7-resize-06.png)
![](/about/media/pic/hashmap-1.7-resize-07.png)
![](/about/media/pic/hashmap-1.7-resize-08.png)

###### 2.3.2.2.2 并发请求

##### 2.3.2.3 JDK1.8扩容过程分析
``` 
  JDK 1.8扩容时保持链表顺序不变,避免了死环现象的发生;
```
### 2.4 树状化存储结构
#### 2.4.1 HashMap中引用
