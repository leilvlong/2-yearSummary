谈谈java数据容器的使用

​	该文理解基于java8

​	java中的数据容器主要分为两类，单列数据容器List与双列数据容器Map；单列数据容器单纯的持有大量对象，而双列数据容器则是以key：value的形式存储，并且可以通过一些技巧与java提供的API去获取keys、values。至于set，严格意义上来讲并不属于单列。

​	单列中最常用的是ArrayList，其底层由数组实现，关于ArrayList的实现只需理清三个点即可对整体有直观的认识。其一：elementData，是一个Object数组，作为ArrayList的成员而存在，用于持有大量对象；其二：size，是一个int，作为ArrayList的成员而存在，用来记录容器持有的对象数量；其三：System. arraycopy方法，它不属于ArrayList，但是对ArrayList的增删都是由它来完成的，例如add方法，添加对象时，会对应的增加size，并将size与elementData的length做比较，如果length不够，将会创建一个新的数组，通过System. arraycopy方法将所有对象从原elementData复制到新elementData中，并将add方法传入的对象添加到新elementData中；add的重载方法支持向指定索引添加元素，其实现逻辑也是通过System. arraycopy方法实现，依然是先确定elementData数组容量是否足够，若不够将先进行扩容操作（创建一个容量更大的新数组，将原数组的元素都赋值复制到新数组中，elementData引用指向新数组），然后将指定索引位置及以后的元素复制到elementData中，但是这些对象在数组中的整体索引会整体位移增大一位，这样指定的索引就会空出来，然后将传入的对象添加到指定索引上；在ArrayList中的增删只需理解之前分析的三点就很容易阅读ArrayList整体的源码，因为整体的逻辑都是差不多的，都是操作elementData、索引、size。同时为什么ArrayList增删的开销很大的原因解开了，因为他在不断的创建新数组，并将旧数组中的元素给复制到新数组中。ArrayList允许指定初始容器大小，这可以缓解增删的开销，但会增加内存空间的负担。因此，如果要持有大量对象进行操作，使用ArrayList是合适的，但是要考虑合理优化增删的开销，倘若频繁的对容器对象进行增删操作，LinkedList会更加合适。LinkedList作为链表结构，通过指向关联对象，因此查询较ArrayList会慢上不少，但是增删效率远远高于ArrayList，只需将指向这一属性指向到新对象上即可，至于LinkedList的实现只有一个核心，它的静态内部类Node具有三个属性，next指向下一个对象，prev指向上一个对象，item则表示当前节点指向的对象，因此LinkedList是双向的，对LinkedList的所有增删查改操作最终都转化为对Node的操作。最后提一下，使用System. arraycopy方法可以实现很多弱类型语言中对list的切片操作。此处给出一个此操作的例子：

```java
/**
 * 关于更完善的逻辑可以自行调整,不必对一个小示例钻牛角尖,更重要的是思路
 */
public class ListTest{
    public static void main(String[] args) throws IOException {
        list(3,7);
        list(3,-1);
    }

    public static void list(int start, int end){
        int[] ints = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
		
        int newEnd = end > 0 ? end : ints.length + end;
        int newLength = (newEnd - start)+1;
        int[] newInts = new int[newLength];
        System.arraycopy(ints, start, newInts, 0, newLength);

        System.out.println(Arrays.toString(newInts));
    }

}
```



​	双列中最常用的HashMap，该容器命名很具有实用性，一目了然：散列映射；通过计算key的hash存储value，因此它也是无序的，事实上，只要是通过hash存储的，如果没有提供额外的开销，那么都是无序的。关于HashMap的使用与源码，依然有两个核心点：table，是一个Node数组；Node，HashMap的静态内部类，具有四个属性：hash，key的hash；key，存储时的key；value，存储时的value；next，在HashMap中存储时没有使用。事实上这个Node不仅仅是HashMap的核心，它也让HashMap成为了好几个数据容器的核心，稍后再说。基于这两个核心点，以put与get方法为例；put方法：进入put方法后，会直接调用putVal方法，将key做hash后传入参数hash、key、value、onlyIfAbsent（默认为false）、evict（默认为true），关于这些参数，官方文档有详细的介绍，接下来是围绕这些参数的逻辑；进入putVal方法后首先会判断table是否为空，如果为空，则创建长度为16的数组，接下来继续判断，将key的hash值与table的长度减一（也就是table的索引范围）进行&的位运算，该运算表示在取二进制，双方都为1时则返回1产生一个新的十进制的值，该值为table中的存储索引，判断该索引在table中是否为null，若为null，则直接存储，存储的方式是创建一个Node，设置Node的属性hash为传入的hash，key为传入的key，value为传入的value，next不使用；然后将table的存储索引指向该Node对象，从而完成键值对的存储；倘若进行&运算后得到的索引指向的不为null，则表示此处有Node对象，此时会进入else，在else里面会对传入的key的hash和key本身与Node对象的hash与key进行比较，若返回true，则覆盖value。值得一提的是，HashMap也会记录存储的键值对的数量，即size，每次添加完成时会与table数组的length比较，然后判断是否对table扩容，如果需要扩容，将创建一个新table,对原table进行遍历获取Node对象的hash属性与next，只有当next为null时，则代表它是属于Hash方式存储的，此时与新table的length-1（即新table的索引范围）进行&运算并存储。get方法：该方法的逻辑很简单，将传入的key计算hash然后调用getNode方法，参数是key的hash与key，将hash与table的length-1进行位运算，得到索引后取出该索引对应的Node，若为Null则返回Null，若不为Null，则将Node对象的key和hash与传入的key和hash做比较，若返回true，则返回Node的value属性，若为false，返回null。关于HashMap的方法举例就到此为此，HashMap的源码相对于LinkedList与ArrayList要难阅读很多，因为其中充斥着各种功能的内部类以及数学计算公式，更重要的是HashSet、LinkedHashSet、LinkedHashMap都是围绕它实现的，HashMap中的很多逻辑与自身的使用其实并无关系，都是给它们使用的。这里顺便再提一下，hash值如果冲突（意思是如果key的hash与equals比较都为true，但是不是同一个对象），那么value就会被覆盖，因此，使用自定义类的对象作为key时，确保自定义类重写过equals方法与hashCode方法。以上解释都基于存储索引不冲突的大前提下，文末将解释HashMap的存储索引冲突。以下附上一个简单的hashMap实现案例对hashMap的散列查找方式做示例：

```java
// 自定义hashMap
class MyHash<K,V>{
    private Object[] table;
    public MyHash() {
        table = new Object[10000];
    }
    private int hash(K key){
        return key.hashCode() & (table.length-1);
    }
    public V put(K key, V value){
        int hash = hash(key);
        System.out.println(hash);
        Object o = table[hash];
        table[hash] = value;
        return (V) o;
    }
    public V get(K key){
        int hash = hash(key);
        return (V) table[hash];
    }
}

// 自定义对象key
class TestMyHashKey{
    private String name;
    private Integer age;
    public TestMyHashKey(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
   
    @Override
    public int hashCode() {
        return Objects.hash(name, age,System.identityHashCode(this));
    }
}

//测试效果
public class TestMyHash{
    public static void main(String[] args) {
        MyHash<TestMyHashKey, Integer> myHash = new MyHash<>();
         //属性相同对象不相同
        TestMyHashKey tets1 = new TestMyHashKey("张三",18);
        TestMyHashKey tets2 = new TestMyHashKey("张三",18);
        TestMyHashKey tets3 = new TestMyHashKey("张三",18);
        myHash.put(tets1, 1);
        myHash.put(tets2, 2);
        myHash.put(tets3, 3);

        System.out.println(myHash.get(tets1));
        System.out.println(myHash.get(tets2));
        System.out.println(myHash.get(tets3));

    }
}
```

​	HashSet、LinkedHashSet、LinkedHashMap的实现都是围绕着HashMap的源码实现的。HashSet：里面有个成员变量，就是HashMap；对HashSet的add操作实际就是调用了HashMap的add，将你传入的元素做为key传入HashMap的add方法中然后存储，因此，它也能确保元素唯一；HashSet并不提供任何形式的get方法，想想就能明白，对一个无序的集合get毫无意义且愚蠢；通过一些特殊的技巧运用HashSet是很不错的，比如说，需要对ArrayList去重，HashSet的构造方法就可以完成此操作，ArrayList的构造方法也允许HashSet。LinkedHashMap：同样是围绕HashMap的源代码实现，更确切的说是继承了HashMap，其内部类Entry也继承了HashMap的内部类Node，并添加了额外的属性before、after记录上一个与下一个元素，因此LinkedHashMap它是双向的，LinkedHashMap的put方法是完全用HashMap的put方法实现的，而get方法一半是用HashMap的get方法实现的，但是额外多加了一层判断，它会判断存取顺序，若为true，则在内部进行一些操作，就是Entry的before、after的一些指向引用转换，并不会影响此次获取的value。LinkedHashSet：继承自HashSet，近乎完全操作HashSet的API，但是奇怪之处就在于没有任何的链表指向操作，LinkedHashSet是如何保持有序的？事实上通过DEBUG调试可以看到是通过LinkedHashMap的Entry实现的；LinkedHashSet的默认构造器调用了父类的构造器，在父类中构造了LinkedHashMap，LinkedHashSet的实现精髓全在构造器中。

​	关于HashMap最后一点就是Node的next属性，在存储key、value时完全没有使用，那么该属性到底是何时使用的？这里所涉及的是另外一个知识点，即hash计算table存储索引冲突，一旦发生这种冲突，整个HashMap的结构会发生根本性的变化。前文提到过使用put方法时：若table的length-1与key的hash进行&运算得出的索引处不为null，则会执行else，在else里面会对传入的key的hash和key本身与Node对象的hash与key进行比较，此处便是索引冲突的发生点，若该判断为false，则表示发生冲突。为了解决这种冲突，它会将整个HashMap的存储结构额外在添加一层链表结构，当链表过长时不易查找，此时会将链表通过红黑树转换。之所以说是额外添加一层链表结构是因为原有的Hash与table.length-1的查找方式依然可以使用，若是该方法找不到，则会判断HashMap是否发生过存储索引冲突，判断方式很简单，next属性是否为null，如果不为null，则代表存储索引冲突过，通过next属性查找，如果next也找不到，则会判断是否发生过红黑树转换，如果发生过，则通过红黑树查找，若以上都找不到，则返回null。通过上文结合getNode方法源码解释此时会更直观：

```java
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                //此处返回值表示未发生过存储索引冲突
                return first;
            
            //此处判断是否发生过存储索引冲突,给HashMap额外添加链表结构
            if ((e = first.next) != null) {
                
                //此处判断是否发生过红黑树转换
                if (first instanceof TreeNode)
                    //如果发生了红黑树转换则通过红黑树增加查找效率
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                
                //通过链表查找元素
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







