## ArrayList源码解析

ArrayList使用方法有三种。

```java
public class DemoUtils {
    //不指定初始化大小
    List<String> list = new ArrayList<>();
  
    //指定初始化大小
    List<String> list = new ArrayList<>(1<<4);

    //指定collection为构造参数
    List<String> list = new ArrayList<>(oneCollection);
}
```

三种不同的方式将会调用不同的构造方法，在看构造方法前首先看一下ArrayList中的成员变量。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    /**
     * 默认初始容量
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 用于空实例的共享空数组实例
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 用于默认大小的空实例的共享空数组实例。我们将其与EMPTY_ELEMENTDATA区分开来，以了解添加第一个元素时要膨胀多少。
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 存储ArrayList元素的数组缓冲区。ArrayList的容量是此数组缓冲区的长度。任何具有elementData==DEFAULTCAPACITY_empty_elementData的空ArrayList将在添加第一个元素时扩展为DEFAULT_CAPACITY。
     */
    transient Object[] elementData; // 非私有以简化嵌套类访问

    /**
     * ArrayList的大小（它包含的元素数）
     */
    private int size;
}
```

这些变量具体哪里使用先不用着急，从构造方法再到增删改查过一遍就能大致看遍整个ArrayList。

### ArrayList中三个常用的构造方法

```java
public class ArrayList {
    /**
     * 构造具有指定初始容量的空列表。
     * 一般代码中都会要求按照预估容量在创建的时候进行初始化容量赋值，所以这也是最常用的构造方法。
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            //上面我们知道elementData就是我们底层存放数据的数组，这里只要传入初始化容量值合理就会创建对应大小的数组。
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            //空实例
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                    initialCapacity);
        }
    }

    /**
     * 上面我们知道这里也是个空的数组，源码中这里的注释是构造一个初始容量为10的空列表。涉及到DEFAULT_CAPACITY这个成员变量，后面在add的时候就能用到。
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

  /**
   * 进来没有判断直接调用集合的toArray方法进行底层数组赋值，所以说不能传入null，否则会空指针
   * 
   */
  public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    //判断的同时完成size的赋值
    if ((size = elementData.length) != 0) {
      //非空情况下还需要判断下c.toArray()赋值的是不是Object[].class
      if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
      //传入的是空集合的效果等同于自己赋值初始化容量为0
      this.elementData = EMPTY_ELEMENTDATA;
    }
  }
}
```

### ArrayList中Add方法（扩容相关）

整个ArrayList大部分的核心都在Add方法中，Add分为直接尾部添加元素以及指定index添加元素

**尾部添加元素**

```java
public class ArrayList {
    /**
     * add方法会将元素添加到list的尾部，里面同时包含了初始化及扩容的一些列操作.
     */
    public boolean add(E e) {
        //这个作为核心，主要用于保证容量适配等内容
        ensureCapacityInternal(size + 1);
        //数组中下一个空元素赋值为e，同时也增加了size的计数
        elementData[size++] = e;
        return true;
    }

    private void ensureCapacityInternal(int minCapacity) {
        //内部进行调用ensureExplicitCapacity方法
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    /**
     * 首先看看获取minCapacity的calculateCapacity方法
     */
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        //在初始化创建之后第一次调用时，由于size为0，所以这里传入的minCapacity为1
        //这里就是为什么区分了两个为空的数组，对于缺省初始化容量指定的数组，使用的容量就是DEFAULT_CAPACITY为10
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        //不是的话就为1了（首次调用add的情况）
        return minCapacity;
    }
  
    private void ensureExplicitCapacity(int minCapacity) {
        //对于modCount就不多说了，在介绍ArrayList的快速失败机制时有说道过，想了解的可以看看那个文章
        modCount++;

        //比较下是否需要扩容，由minCapacity和当前数组长度比较。具体的几种情况看下面分析
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    /**
     * 扩容，保证能够容纳元素
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        //oldCapacity 被指定为 底层数组的长度（注意包含了空元素）
        int oldCapacity = elementData.length;
        //newCapacity 其实也就是扩容之后的容量，对于2的N次幂容量来说就是扩容为1.5倍，其他的容量只能是近似1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //这里主要针对于容量配置为0的时候进行基础性设置，因为这种情况newCapacity为0
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        //MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8，主要是做避免扩容后溢出处理
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        //这里传入的minCapacity是当前底层数组size+1，一般非首次添加都是当前数组的长度+1，当本身已经是Integer.MAX_VALUE长度时+1就溢出了
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        //没有溢出的话判断是否大于MAX_ARRAY_SIZE从而赋值
        return (minCapacity > MAX_ARRAY_SIZE) ?
                Integer.MAX_VALUE :
                MAX_ARRAY_SIZE;
    }
}

```

**对于初始化及扩容补充**

对于ensureExplicitCapacity方法，有以下几种情况

- 创建List时指定初始化容量为0，首次调用add时
  - 首先自己指定为0的时候，赋予的空数组是EMPTY_ELEMENTDATA，因此无法触发minCapacity被设置为默认值10，因此minCapacity仅为1。
  - elementData是个空数组，所以elementData.length为0，minCapacity - elementData.length > 0就被满足了，需要调用grow方法扩容。
  - 扩容后数据容量为1，所以一般自己指定为0的情况下表明结果为空不会往内部添加元素，否则后续触发扩容会十分频繁。
- 创建List时缺省初始化容量，首次调用add时
  - 缺省时赋予的空数组是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，会触发赋值为DEFAULT_CAPACITY也就是10
  - elementData是个空数组，所以elementData.length为0，minCapacity - elementData.length > 0成立，因此需要调用grow进行扩容操作(实际上是初始化操作)
- 创建List时指定了初始化容量并且大于0，首次调用add时
  - elementData会被初始化为指定容量长度的空数组，而minCapacity还是为1，所以最低情况下指定初始化容量为1也不会触发扩容
- 其它非首次情况其实就是根据minCapacity和长度做比较
  - minCapacity就是当前size+1，所以一句话说就是List是满了才扩容

**指定index写入元素**

```java
public class ArrayList{
    public void add(int index, E element) {
        //指定的index不能超出现有元素的size
        rangeCheckForAdd(index);
        //同样的由于插入了一个元素，所以需要保证容量
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //使用arraycopy进行index后元素复制操作，近似于插入后后移操作
        System.arraycopy(elementData, index, elementData, index + 1,
            size - index);
        //替换index位置元素
        elementData[index] = element;
        size++;
    }
}

```

### ArrayList中Remove方法

ArrayList分别有两个remove方法，分别为移除指定元素以及指定index上的元素

**移除指定元素**

```java
public class ArrayList 
{
  
    public boolean remove(Object o) {
        if (o == null) {
            //由于可以添加null，所以还是要遍历找一下第一个null值元素进行删除
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                //移除首个匹配的元素
                fastRemove(index);
                return true;
                }
        } else {
            //和null一样的操作，区分是因为避免直接使用下面这种写法造成空指针
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
                }
        }
        return false;
    }

    private void fastRemove(int index) {
        //快速失败机制保证
        modCount++;
        //删除后需要往前移动的元素个数
        int numMoved = size - index - 1;
        if (numMoved > 0)
        //直接使用arraycopy进行复制操作时将后面需要移动的元素直接复制到新数组的index位置开始，完成了快速删除  
        System.arraycopy(elementData, index + 1, elementData, index,
                numMoved);
        //size减少同时对于多余的最后一个元素进行置空便于gc
        elementData[--size] = null; // clear to let GC do its work
    }
  
}

```

**移除指定index上的元素**

```java
public class ArrayList
{
    public E remove(int index) {
        //一样的避免调用方瞎填写index
        rangeCheck(index);
        modCount++;
        //返回被删除的元素
        E oldValue = elementData(index);
        //一样的快速删除方式
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                    numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
}
```

### ArrayList中get方法

get方法算是极其简单的方法了，直接获取index的元素返回即可

```java
public class ArrayList
{
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
}
```

### ArrayList中set方法

set方法就是替换指定index位置元素，没有什么内容

```java
public class ArrayList
{
    public E set(int index, E element) {
        rangeCheck(index);
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
}
```


### ArrayList中其它方法

**获取元素索引位置方法**
```java
public class ArrayList
{
    /**
    * 其实就是个遍历，返回首个查到该元素的位置
    */
    public int indexOf(Object o) {
        if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
            return i;
        } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
            return i;
        }
        return -1;
    }

    /**
    * 还是遍历，倒着查最后一个
    */
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                return i;
        }
        return -1;
    }
}
```

**扩容方法**

ensureCapacity方法在ArrayList内部并没有被调用，主要提供给外部调用，保证在放入数据前先进行调用以避免多次扩容。比较常见的情况就是不知道数据量大小，先创建一个空ArrayList之后拿到了要放入的数据时调用ensureCapacity方法避免放入大量数据多次扩容。

```java
public class ArrayList
{
    public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }
}
```