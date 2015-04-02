在JDK中 java.lang.String 是一个再平常不了的类，也几乎是每个应用中实例占比最大的一个类了。从JDK1.1开始，这个类在使用上基本没有什么变化，可是其内部实现却经历了好几次不算很小的变化，本文尝试介绍一下这个变化的过程，以及相关的原因。这个过程也反映了Java自身的不断优化，乃至于试错的一个过程。

从JDK1.1一直到JDK1.6，String对象的结构没有变化，都是
```java
public final class String  {
    private final char[] value;  // 存储实际字符数组
    private final int offset;    // 在数组中的偏移量
    private final int count;     // 字符串长度
    private int hash;            // Hash值
}
```

为什么不直接存储为如下形式呢：
```java
public final class String  {
    private final char[] value;  // 存储实际字符数组
    private int hash;            // Hash值
}
```
由于String实际上是JVM中最为常用的对象，如果我们使用 MAT（Memory Analysis Tool)等内存分析工具来查看一个JVM实例的内存时，我们可以看到在大部分情况下，String是数量最多的对象之一。如果每个对象都节省2个字段，那么，相当于每个对象都节省了8个字节，这个还是相当之可关的。那么，为什么从1.1到1.6，JDK的设计者都要新增offset、count字段呢？

阅读以下代码，基本上可以从 String.substring方法看出作者的设计意图：
```java
public String substring(int beginIndex, int endIndex) {
    return beginIndex == 0 && endIndex == this.count?this:new String(this.offset + beginIndex, endIndex - beginIndex, this.value);
}
```
在这里，如果字符串a是从字符串b使用substring方法创建出来，那么，a和b会共享value这个字符串数组，只是更新以下offset、count字段而已。可以想象，这可以避免新分配一个char[]的开销，而且，如果字符串很长的话，这个节省还是相当可观的。因此，JDK的设计者正是基于更有效的使用利用内存这一原则，在每个String实例中增加了offset和count这一字段，从而实现不同的String可以共享同一份字符数组。

**多么关注内存利用率的程序员！**

简单的说一下，为什么要加上 hash 这个字段？由于String经常需要进行Hash运算（例如作为HashMap的key），如果每次都重复这个hash计算过程，那么，会产生较大的计算成本的。因此，String会将hash值缓存起来。（不过，如果这个Hash值正好是0，而且字符串的长度又比较的长，那么，这个悲剧的过程还是会每次重演，比如，这个字符串```"\u00037=1;6#"```的hash值就正好为0，hashCode这个方法每次调用都会重复这个计算过程，**黑，说一定又可以诞生一种新的Java攻击方式，不要说是我这里传出去的....**）。

可是到了JDK1.7，突然变了：
```java
    private final char value[];
    private int hash; // Default to 0
    /**
     * Cached value of the alternative hashing algorithm result
     */
    private transient int hash32 = 0;
```
offset和count字段不见了，看看源代码，substring不再共享底层的字符数组了。为什么要作这个转变呢？难道内存利用率不重要吗？

故事说来话长，最终的结论是：substring的共享机制，导致了由于长度较小的字符串一旦被引用，那么原来那个长度更大的 value[] 数组就会被无限期的被引用到，从而无法进行垃圾回收（本来值需要占用一个较小的value[]即可）。得出的结论是：substring得不偿失。不过我个人比较怀疑这个推导过程，这种几率应该存在，但有多大呢？很有可能只是个案。相反，如果分析一下Java的对象，看看采用共享机制，共享带来的节省（我的估计非常有限），以及带来的开销（每个对象增加8个字节），我估计后者其实更高得多，所以我个人其实还是认为取消共享应该是得大于失的。（没有证明过，不过，这个证明并不困难）。

如此说来，从JDK1.1到JDK1.6所采用的那个很有技术特色的设计，居然是错误的？这个居然会出现在JDK的String这个最简单、最常用的类中？一点也不出奇。

问题来了，怎么多了一个hash32字段？原来，Java的世界中多了一个因为HashCode相同导致的Hash漏洞攻击（感谢我党的GFW，我想google一下资料来作为本文的参考，不过，失败了，因此，我无法向你提供这个漏洞的更详细情况），原理其实很简单，找到一批hashCode相同的字符串，然后不断的访问Tomcat，Tomcat会将其作为HashMap的key，进行存储，以及进行查询。现在问题来了，由于Hash值相同，这个搜索将会退化成为线性搜索，从而导致性能急剧下降，从而产生DDos攻击。

估计JDK的设计者为了解决这个问题，尝试采用了一种新的hash算法，这个值就存储在hash32中。可是，这个新的算法怎么保证就不易出现重复呢？这个我倒真没想过？但为此在String中增加一个hash32字段，感觉不算是高明啊。成本可真不小。而且，HashMap也针对String特殊计算Hash值，不再使用之前的hashcode，感觉也是怪怪的。

来到JDK1.8，哈哈，又发生变化了，share机制还是去掉了，hash32也去掉了，现在的String开始瘦身了，只剩下2个字段：
```java
    private final char value[];
    private int hash; // Default to 0
```
那么，JDK1.8怎么来解决hash漏洞攻击呢？话说回来，Hash漏洞攻击和String有什么关系呢？有哪一个hash算法能够保证不重复呢？所以JDK1.8干脆回归到优化HashMap上来，原来当hash值重复的时候，HashMap会退化成为链表，有O(N)的时间复杂性，那么，我们将其转换为平衡树，这样，在搜索时就只有O(log N)的复杂性了。这样不是更简单有效得多吗？

其实，在String中，还有一个intern方法，也牵涉到JVM的内存管理策略，在多个JVM中有不同的实现方式，这个可以参考：http://java-performance.info/string-intern-in-java-6-7-8/ 也是一个很有意思的变迁过程。



