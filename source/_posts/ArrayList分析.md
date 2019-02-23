![超杀女.jpg](http://upload-images.jianshu.io/upload_images/2824145-5dda6fec988df5dd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 对于集合的源码分析，一般我会采用这几种方式
1.  怎么添加元素？
2.  怎么获取元素？
3.  怎么删除元素？
5.  内部数据结构实现？

话不多说，直接走起。

## 一.怎么添加元素？
一般我们通过ArrayList添加元素。一般会调用其构造方法,然后调用其对象的add方法

### 查看空参构造函数

```
//Constructs an empty list with an initial capacity of ten.
 public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
通过构造函数可以发现。ArrayList在调用无参的构造函数时，会构造一个长度为10的缓存数组

### 查看add方法
```
public boolean add(E e) {
        ensureCapacityInternal(size + 1); 
        elementData[size++] = e;
        return true;
    }
```
通过该方法发现 ArralyList内部的数据结构其实是一个**数组**（elementData[size++] = e;）并且在添加时会先判断当前容器在添加了一个对象之后该对象的容纳能力（主要为了，在下一次添加元素的时候，缓存数组能够有足够的空间添加元素)。之后将元素添加到数组末尾。

### 继续查看ensureCapacityInternal（）方法

```
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
```
这里我们发现，如果当前elementData为空的话，minCapacity=DEFAULT_CAPACITY,同时DEFAULT_CAPACITY的默认值是**10**,从这我们可以看出，在第一次初始化的时候，ArrayList内部会默认**创建一个内部长度为10的数组。**


### 继续点击ensureExplicitCapacity（）方法

```
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        //判断添加元素后，缓存数组时候需要扩展
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
```
该方法会记录当前数组的更改次数，并且判断当前数组添加后，是否需要进行增长，

### 继续走grow方法（重点的来了！！！）

```
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //扩展数组的长度，
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
该方法会扩展缓存为当前数组的长度为 **原数组长度+原数组长度的二分之一**，也就是按照原数组的**50%**进行增长，同时该数组最大的扩展长度是**Integer.MAX_VALUE - 8**。也就是ArrayList最多能存储的数据长度，通过扩展数组长度以后，在下一次添加数据的时候，ArrayList就有足够的空间去添加新的元素了。

## 二.怎么获取元素

其实ArrayList获取其中的元素很简单，根据角标获取对应数组中的元素，具体代码如下：
```
 public E get(int index) {
        if (index >= size)//判断当前角标长度是否超过数组长度，如果是抛出异常，反之返回数据
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
        return (E) elementData[index];
    }

```
##三.怎么删除元素
在ArrayList中，有两个关于删除元素的方法，一个是**remove(int)**,另一个是**remove(Object)**

### 1.remove(int)方法
```
public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // 将数组最后一位置null

        return oldValue;
    }
```
这里先判断删除角标是否超过数组长度，然后通过System.arrayCopty()方法将角标对应的元素删除。

这里对System.arrayCopty()方法解释一下。该方法的第一个参数是**源数组**，第二个参数是复制的**开始角标**，第二个参数是**目标数组**。第三个参数是**目标数组**与**源数组**的复制数据开始角标。最后一个参数是复制的长度。(注意：！！！复制的长度不能大于**目标数组减去开始角标的长度**或**源数组减去开始角标的长度**)
#### eg:
```
  int[] a = {0, 1, 2, 3, 4};
  int[] b = {5, 6, 7, 8, 9};
  System.arraycopy(a, 0, b, 1, 3);
 // 则进行操作后 b = {5，0,1,2,9} 
```
#### 2.remove(Object)方法
```
 public boolean remove(Object o) {
        if (o == null) {//判断当前元素是否为空，遍历数组，获取其角标
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

  private void fastRemove(int index) {//根据角标，删除相应元素
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
remove(Object)根据object在数组的角标，执行fastRemove(index)方法。删除方法与remoIndex(int)一样。这里就不在分析了。

### 总结
- ArrayList内部实现是数组，且当数组长度不够时，数组的会进行原数组长度的**1.5倍扩容**。
- ArrayList内部元素是**可以重复的**。且**有序**的，因为是按照数组一个一个进行添加的。
- ArrayList是**线程不安全的**，因为其内部添加、删除、等操作，没有进行同步操作。
- ArrayList**增删元素速度较慢**，因为内部实现是数组，每次操作都会对数组进行复制操作，复制操作是比较耗时的


最后，附上我写的一个基于Kotlin 仿开眼的项目[SimpleEyes](https://github.com/AndyJennifer/SimpleEyes)(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start


