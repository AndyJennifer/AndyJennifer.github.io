---
title: 'volatile修饰数组,那么数组元素可见吗?'
tags:
- volatile
categories:
- 奇思妙想
---

[](https://stackoverflow.com/questions/53753792/java-volatile-array-my-test-results-do-not-match-the-expectations)


### CopyOnWriteArrayList


```   

  private transient volatile Object[] elements;

    /**
     * Gets the array.  Non-private so as to also be accessible
     * from CopyOnWriteArraySet class.
     */
    final Object[] getArray() {
        return elements;
    }

 public E get(int index) {
        return get(getArray(), index);
    }

```

### ConcurrentHashMap

```
    transient volatile Node<K,V>[] table;

    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```


### 数组中在内存中的存储结构

难道是因为数组的连续存储的结构？


### 奇怪的地方


```
public class Test {
    public static volatile long[] arr = new long[20];
    public static void main(String[] args) throws Exception {
        new Thread(new Thread(){
            @Override
            public void run() {
                //Thread A
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                arr[19] = 2;
            }
        }).start();
        new Thread(new Thread(){
            @Override
            public void run() {
                //Thread B
                while (arr[19] != 2) {
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();
    }
}
```

```
public class Test {
    public static long[] arr = new long[20]; // Make this non-volatile now
    public static volatile int vol = 0; // Make another volatile variable

    public static void main(String[] args) throws Exception {
        new Thread(new Thread(){
            @Override
            public void run() {
                //Thread A
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);    
                    arr[19] = 2;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        new Thread(new Thread(){
            @Override
            public void run() {
                //Thread B
                while (true) {
                    int i = vol;

                    if (arr[19] == 2) {
                        break;
                    }
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();
    }
}
```

这里也居然退出了


```
    public static void main(String[] args) throws Exception {
        new Thread(new Thread() {
            @Override
            public void run() {
                //Thread A
                try {
                    TimeUnit.MILLISECONDS.sleep(1000);
                    arr[19] = 2;
                    flag = true;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        new Thread(new Thread() {
            @Override
            public void run() {
                //Thread B
                while (true) {
//                    int i = vol;
                    System.out.println("flag--->" + flag);
                    if (arr[19] == 2) {
                        break;
                    }
                }
                System.out.println("Jump out of the loop!");
            }
        }).start();
    }
```
这里居然也退出了