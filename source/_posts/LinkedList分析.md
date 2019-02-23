![模仿游戏.jpeg](http://upload-images.jianshu.io/upload_images/2824145-09c6c13397af88ee.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上次我们分析了ArrayList,大家都已经了解了分析一个集合的步骤。那接下来，我们继续分析LinkedList。废话不不多说，直接整。

#查看LinkedLis成员

```
	/**
     * 指针指向第一个节点
     * 初始化: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * 指针指向最后一个节点
     * 初始化: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
```
从LinkedList成员中，可以看出Lined内部有两个指针，first(指向第一个节点)与Last(指向最后一个节点)。查看相关Node类声明：
###Node类声明

```
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
Node类中，保存了当前数据元素，上一个节点，及下一个节点。从这里，我们大概了解到了LinkedList的内部结构是链表。

##一、添加元素
###add(e)方法

```
 public boolean add(E e) {
        linkLast(e);
        return true;
    }

```
add方法内部调用了linkLast(e),继续走linkLast(e)。

####查看linkLast（e)方法

```
void linkLast(E e) {
        final Node<E> l = last;//last指向最后一个节点。
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;//frist指向第一个添加元素
        else
            l.next = newNode;//不是第一次添加，上一个节点的next指向当前节点
        size++;
        modCount++;
    }
```
当该方法执行是，会初始化一个Node节点保存当前添加元素及上一个节点。如果是第一次执行，该方法，first与Last都会指向该节点。如果不是第一次执行。上个节点的next会指向新添加的节点，且last指向新添加的节点。
###addFist（e)方法
addFist（e)中方法直接调用了linkFrist(e)方法，我们直接查看LinkFirst方法:

```
  private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```
linkFirst方法内部创建了一个新的节点。如果是第一次添加。新节点上个节点为null。如果不是,则新的节点的上个的节点为first原来指向的节点，first指向新添加的节点。

###addLast(e)方法原理与addFirst（e)原理差不多，这里就直接跳过了

## 二、获取元素

###get(Index)方法
```
public E get(int index) {
        checkElementIndex(index);//判断是否操作存储的长度
        return node(index).item;
    }
```
get方法先判断时候在有效范围类，如果调用了node(index）方法返回相应元素，继续走node方法。

```
Node<E> node(int index) {
        // assert isElementIndex(index);
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

```
该方法内部先判断index的位置是否小于**总长度的一半**，如果是，则从链表前方遍历，如果不是，则从链表最末尾进行遍历。

##三、删除元素

###removeFirst()方法

``` public E removeFirst() {
        final Node<E> f = first;
        if (f == null)//如果first没有指向元素，抛出异常
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
```
removeFirst()方法，先判断当前frist时候为null,如果不是,将first作为参数传入unLinkFirst()方法，查看unLinkFirst方法。

```
 private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```
unLinkFirst方法将器node中数据置为null,且将frist节点，指向f的下一个节点。并将f的下一个节点的上个节点(也就是f)至为null。

###removeLast()方法

```
public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }
```
removeLast方法内把Last指向的节点，传入unLikeLast()方法，继续走unLinkLast方法。

```
 private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
```
unLinkLast方法内部获取last重写指向Last原节点的上一个节点，同时将Last原节点至为null.

总结

* LinkedList方法内部实现是链表，且内部有fist与last指针控制数据的增加与删除等操作
* LinkedList内部元素是可以重复，且有序的。因为是按照链表进行存储元素的。
* LinkedList线程不安全的，因为其内部添加、删除、等操作，没有进行同步操作。
* LinkedList增删元素速度较快。

最后，附上我写的一个基于Kotlin 仿开眼的项目SimpleEyes(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start




