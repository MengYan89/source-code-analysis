# ArrayList源码解析

## ArrayList是什么
ArrayList是Java集合框架中比较常用的数据结构。是一个容量能够动态增长的数组。它继承了AbstractList抽象类，
并实现了List,RandomAccess,Cloneable,Serializable四个接口，所以ArrayList支持快速访问，克隆并且支持序列化。
## 继承关系
上面介绍了ArrayList都继承了什么实现了什么，现在我们通过一张图来了解一下ArrayList的继承关系。
![uml](img/uml.png)
## 成员变量
```java
    /**
    * 用于序列化
    */
    private static final long serialVersionUID = 8683452581122892189L;
    /**
     * 默认初始容量.
     */
    private static final int DEFAULT_CAPACITY = 10;
    /**
     * 用于空实例的共享空数组实例
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};
    /**
     *
     * 用于默认大小的空实例的共享空数组实例。我们将其与空的元素数据区分开来，以了解添加第一个元素时要膨胀多少。
     * 缺省空对象数组
     * 用于无参构造方法，判断第一次添加元素时判断最小容量
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    /**
     *
     * 存储ArrayList元素的数组缓冲区。
     ArrayList的容量是这个数组缓冲区的长度。添加第一个元素时，任何elementData==DEFAULTCAPACITY_empty_elementData的空ArrayList都将扩展为DEFAULT_CAPACITY大小。
     */
    transient Object[] elementData; 
    /**
     * ArrayList的大小（它包含的元素数）。
     * @serial
     */
    private int size;
    // ArrayList的最大大小
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    
```
### DEFAULT_CAPACITY
ArrayList默认的初始大小，如果创建实例时使用无参构造函数就会初始化一个DEFAULT_CAPACITY(10)大小的ArrayList。  
### EMPTY_ELEMENTDATA
是个空的Object数组，用于初始化一个空的elementData(ArrayList实际储存数据的数组)
### DEFAULTCAPACITY_EMPTY_ELEMENTDATA
也是一个空的Object数组，用于判断创建ArrayList时是否给出初始大小，如何使用会在下面解释。  
也叫缺省空对象数组。  
### elementData
实际存储元素的Object数组，需要注意的是它被transient修饰，在序列化反序列化时默认将忽视被transient修饰的成员变量。  
在之前我们说过ArrayList支持序列化，其实这个ArrayList为了节约资源做出的优化，ArrayList如果序列化会在下面序列化与反序列化的章节说明。 
### size
ArrayList的大小，这个大小是实际保存的元素的数量，而不是ArrayList的大小。
### MAX_ARRAY_SIZE
java官方描述是ArrayList的最大大小也就是Integer的最大值-8，但实际上最大并不是，最大应该是等于Integer的最大值(2的31次幂-1 = 2147483647)。
为什么扩容时会讲。
### serialVersionUID
用于序列化不多做阐述
## 构造函数
ArrayList有三个构造函数,分别是:
```java
    /**
     * 将缺省空对象数组给elementData
     */
    public ArrayList();   
    /**
     * 构造具有初始容量的空列表
     */
    public ArrayList(int initialCapacity);
    /**
     *  根据一个集合来生成一个ArrayList
     */
    public ArrayList(Collection<? extends E> c);
```
首先来说这个无参的构造函数:
```java
    /**
     * 将缺省空对象数组给elementData
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
这个可以说是很简单了，就是将之前讲过的成员变量DEFAULTCAPACITY_EMPTY_ELEMENTDATA赋值给elementData。
之后会在第一次判断扩容的时候判断elementData是不是DEFAULTCAPACITY_EMPTY_ELEMENTDATA来决定容量大小。  
```java
    public ArrayList(int initialCapacity) {
        // 如果initialCapacity大于0就创建一个对应大小的Object数组给elementData，如果等于0就吧空数组EMPTY_ELEMENTDATA给elementData
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }
```
第二个构造方法有一个initialCapacity参数，这个参数就决定了ArrayList容器的大小，如果initialCapacity大于0就new一个同样大小的
Object数组赋值给elementData。如果等于0就将空数组EMPTY_ELEMENTDATA赋值给elementData。小于0就会抛出IllegalArgumentException。
```java
    /**
     *  根据一个集合来生成一个ArrayList
     */
    public ArrayList(Collection<? extends E> c) {
        // 返回包含c集合中所有元素的数组。
        elementData = c.toArray();
        // 如果数组大小为0就初始化一个空的ArrayList
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            //c.toArray有可能返回的不是Object[]
            if (elementData.getClass() != Object[].class)
                // 如果不是Object[]则生成一个新的Object[]将元素放进去
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
第三个构造方法的参数是一个接口Collection，也就是说它可以根据任意一个继承了Collection接口的类生成一个ArrayList。
首先调用Collection#toArray获取这个集合元素对应的Object数组。  
但通过官方的注释可知toArray返回也许不是Object数组所以我们在判断数组不为空之后，进行了一次类型判断，如果不为Object[].class就重新
拷贝一个Object[]赋值给elementData。
如果数组为空就直接将EMPTY_ELEMENTDATA(空数组)赋值给elementData。
## 添加数据
这个环境主要是讲解ArrayList自己的添加元素的方法，不包括ListIterator的添加方法。  
共有5个方法。
### add
add方法进行了一次重载所以有两个add方法，参数与返回值都不相同。
```java
    /**
     * 将指定元素e添加至末尾
     */
    public boolean add(E e);
    /**
     * 在指定位置插入一个元素，如果当前位置和后续有任何元素，则向右移动一个元素
     */
    public void add(int index, E element);
```
咱们先看第一个也是项目里最常用的只有一个泛型参数的add方法。
```java
    /**
     * 将指定元素e添加至末尾
     */
    public boolean add(E e) {
        // 第一次add elementData无参初始化，判断是否需要扩容，如果需要扩容就进行扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 将元素e放至末尾
        elementData[size++] = e;
        return true;
    }
```
我们可以看到add方法进来就直接调用了一个私有方法ensureCapacityInternal，我们进去看看它都做了些什么。
```java
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
```
在调用ensureExplicitCapacity之前先用elementData与minCapacity调用了私有方法calculateCapacity。
这个方法实际上就是用到之前所说的DEFAULTCAPACITY_EMPTY_ELEMENTDATA(缺省空数组)的地方
```java
    // 根据elementData获得最小容量
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        // 如果elementData是缺省空数组DEFAULTCAPACITY_EMPTY_ELEMENTDATA,则判断minCapacity与默认大小10谁大返回大的一方
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        // 如果不是则直接返回minCapacity
        return minCapacity;
    }
```
其实代码很简单就是判断elementData是不是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，结合之前无参构造函数的代码。
这样就可以判断ArrayList的创建方式是不是无参构造函数了，如果是第一次添加元素时minCapacity应该为1，所以在与
DEFAULT_CAPACITY(10)比大小的时候自然会返回10，如果不是就直接返回minCapacity的大小。  
确认了calculateCapacity时做什么的之后，让我们看看ensureExplicitCapacity吧。  
这个函数主要是为了确保minCapacity比现在的容器大小更大才去扩容
```java
    // 在扩容前确保minCapacity大于现在elementData的大小
    private void ensureExplicitCapacity(int minCapacity) {
        // 操作次数增加
        modCount++;
        // overflow-conscious code
        // 判断minCapacity是否大于elementData.length,大于才去扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
代码里首先让modCount++，这条代码的意义是记录操作数组的次数，有点类似于乐观锁的作用，在后面序列化和Iterator的时候会体现出他的作用。  
在下面就是判断判断minCapacity是否大于elementData.length了，如果大于才去执行grow,现在让我们来看看grow:
```java
    /**
     * 增加容量，以确保至少可以容纳minCapacity个元素
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        // 获取现在Array的容量
        int oldCapacity = elementData.length;
        // 得到默认扩容大小 也就是原来容量的1.5倍,抹去小数
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果minCapacity比默认扩容大，则扩容至minCapacity
        // 如果默认扩容更大，则扩容至默认大小
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果newCapacity大于最大值，则再去判断minCapacity是否小于最大值
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // 生成一个有这原数据大小为newCapacity的数组给elementData
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
这个就是ArrayList的扩容方法了，首先获取现在的容量。
然后计算oldCapacity + (oldCapacity >> 1)，得到的结果就是默认扩容的大小，ArrayList默认扩容为原本的1.5倍就是从这里得来的。  
然后判断传入的minCapacity与默认扩容哪个更大取最大的。
之后在与MAX_ARRAY_SIZE(ArrayList最大容量)判断，如果比最大容量大,则再进一段特殊的判断hugeCapacity。
```java
    private static int hugeCapacity(int minCapacity) {
        //如果扩容大小为负数，则抛出OutOfMemoryError
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        //如果minCapacity大于最大值，则返回Integer最大值，如果不大于最大值则返回最大值
        return (minCapacity > MAX_ARRAY_SIZE) ?
                Integer.MAX_VALUE :
                MAX_ARRAY_SIZE;
    }
```
在hugeCapacity中首先判断minCapacity是否为负数如果是负数则抛出OutOfMemoryError。  
然后再次判断minCapacity是否大于MAX_ARRAY_SIZE(ArrayList最大容量),如果大于就返回Integer.MAX_VALUE。  
这之后grow里就只剩下一行代码了
```java
elementData = Arrays.copyOf(elementData, newCapacity);
```
Arrays.copyOf这个方法我觉得大家都应该很熟悉了，生成一个新的长度为newCapacity的数组并且类型与elementData相同，
再将elementData原本的元素拷贝进去，然后重新赋值给elementData。  
由于之前的hugeCapacity方法中如果大于MAX_ARRAY_SIZE会返回Integer.MAX_VALUE,所以newCapacity的最大值是Integer.MAX_VALUE，
这样也解释了之前说的ArrayList的最大容量为什么不是代码中所写的MAX_ARRAY_SIZE而是Integer.MAX_VALUE。  
讲完了扩容现在让我们回到add方法中剩下的代码：
```java
    public boolean add(E e) {
        // 第一次add elementData无参初始化，判断是否需要扩容，如果需要扩容就进行扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 将元素e放至末尾
        elementData[size++] = e;
        return true;
    }
```
在执行完ensureCapacityInternal后，就将元素放在了现在元素列表的最末尾，返回true这个add方法就结束了。

下面来看第二个add方法，有两个参数一个是index要插入的索引与element元素本身:
```java
    /**
     * 在指定位置插入一个元素，如果当前位置和后续有任何元素，则向右移动一个元素
     */
    public void add(int index, E element) {
        // 判断index是否小于size大于0
        rangeCheckForAdd(index);
        // 判断是否需要扩容，如果需要扩容就进行扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 将index索引开始以后的元素拷贝至index+1索引之后
        System.arraycopy(elementData, index, elementData, index + 1,
                size - index);
        // 将element放至index位置
        elementData[index] = element;
        // 元素数量+1
        size++;
    }
```
首先调用rangeCheckForAdd判断index是否合规：
```java
    /**
     * 判断index是否大于元素数量或者小于0
     */
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```
然后与第一个add方法一样调用ensureCapacityInternal判断是否进行扩容并扩容  
经过ensureCapacityInternal后，调用System.arraycopy将index的位置空出来，后面的元素右移一位。  
![arraycopy](img/arraycopy.gif)
最后将element放到index位置，元素数量+1。
### addAll
addAll同样也进行了重载一共有2个方法，一个是插入到末尾的，一个是插入到指定位置的
```java
    /**
     * 将指定集合添加进列表
     */
    public boolean addAll(Collection<? extends E> c);
        /**
         * 将指定集合添加到index位置，后面的元素右移相应的数量
         */
    public boolean addAll(int index, Collection<? extends E> c)
```
先看第一个的源码吧，参数只有一个集合：
```java
    /**
     * 将指定集合添加进列表
     */
    public boolean addAll(Collection<? extends E> c) {
        // 将集合转为Object[]
        Object[] a = c.toArray();
        // 获得数组的数量
        int numNew = a.length;
        // 判断现在的元素数量+新的数组长度是否需要扩容，如果需要就进行
        ensureCapacityInternal(size + numNew);  // Increments modCount
        // 将a数组拷贝至elementData的末尾
        System.arraycopy(a, 0, elementData, size, numNew);
        // 修改元素数量
        size += numNew;
        return numNew != 0;
    }
```
由于解释过扩容了所以代码也很简单，将集合转为Object数组获取数组大小，然后给ensureCapacityInternal判断是否需要扩容，如需要则扩容。
最后直接将新数组拷贝至elementData的元素末尾，修改元素数量就结束了。  
然后看看第二个addAll：
```java
    /*
     * 将指定集合添加到index位置，后面的元素右移相应的数量
     */
    public boolean addAll(int index, Collection<? extends E> c) {
        // 判断index是否大于size或者小于0
        rangeCheckForAdd(index);
        // 获取Object[]数组
        Object[] a = c.toArray();
        // 获取数组大小
        int numNew = a.length;

        // 判断是否需要扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount

        // 判断index位置后是否有元素
        int numMoved = size - index;
        if (numMoved > 0)
            // 如果有元素则后移，为新数组腾出位置
            System.arraycopy(elementData, index, elementData, index + numNew,
                    numMoved);
        // 将新数组拷贝至elementData
        System.arraycopy(a, 0, elementData, index, numNew);
        // 修改元素数量
        size += numNew;
        return numNew != 0;
    }
```
与上面指定位置的add大同小异,先判断index是否合规，然后给ensureCapacityInternal判断是否需要扩容，如需要则扩容。
然后根据size计算插入的位置是否有元素，如果有先将这个位置和后面的元素右移numNew个位置，为新元素腾出空间，然后将
新元素拷贝进elementData，修改元素数量。
### set
set方法只有一个：
```java
    /**
     * 替换指定位置元素，返回这个位置替换前的元素
     */
    public E set(int index, E element) {
        // 判断index是否大于等于元素数量
        rangeCheck(index);
        // 通过elementData获取替换前的元素
        E oldValue = elementData(index);
        // 替换元素
        elementData[index] = element;
        // 返回替换前元素
        return oldValue;
    }
```
代码基本也很简单，就简单赘述一下，rangeCheck是用来判断index是否大于等于size的方法，如果大于等于则抛出IndexOutOfBoundsException。  
检测通过之后取出当前index位置的元素，将新元素替换上去后，返回原本的元素。