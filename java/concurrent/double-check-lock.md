**引子** 排查一个问题的时候，涉及到了DCL，之前就看到过关于DCL失效的问题，但是细节没记住，准确说没有真正理解，于是又翻阅了下资料，顺道梳理下。  

**DCL idiom**  
```java 
private LazyClass lazyInsntance = null ;

public LazyClass getLazyInstance(){
  if(lazyInstance == null){
    synchronized(this){
      if(lazyInstance == null){
          lazyInstance= new LazyClass();
      }
    }
  }
  return lazyInstance ;
}
```  

**出现的意义**  

**失效根源**  
这篇《[The "Double-Checked Locking is Broken" Declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)》做了比较深入的探讨。结合文章按照我的理解阐述下。   
这里如果有T1 T2两个线程同时访问getLazyInstance,就有可能其中一个线程获取到了未初始化完成的LazyClass实例！  
展开成字节码更容易理解(字节码有删节)：  
```java
// TODO
```
关键在于new 和<init>这里，当发生指令重排序（reordering）时，写入字段lazyInstance先于执行构造方法初始化lazyInstance。如果恰好在写入lazyInstance之后，初始化lazyInstance之前，T2执行第一个if(lazyInstance==null)判断，则为false，于是拿着一个为初始化完成的对象进行了操作，后果可想而知。。。  
插句题外话，第一次接触reordering概念时，三观都毁掉啦，就像脚下踩得坚实的土地变成了沙滩，请不要笑话我浅薄的知识orz...  
那么这里什么情况下会出现重排序呢? 这篇[Synchronization and the Java Memory Model](http://gee.cs.oswego.edu/dl/cpj/jmm.html)有提到指令重排的各种情况。  
具体到DCL这里，比较简单的一种情况是编译重排序。 如果编译器推断，构造方法不会抛出异常以及没有做同步，就可能将构造函数内联调用，进一步对指令reorder。这里就发生了，lazyInstance字段先被赋值，然后再初始化的情况产生。但是synchronized是获取this的锁，而lazyInstance本身没法加锁，于是T2可以顺利的读取到lazyInstance 非null，于是就欢乐地去使用lazyInstance了，一个半成品的lazyInstance对象，于是整个线程就不好了。。。  
另外，即便没有做compile recorder， 在多处理器机器上，内存模块也有可能会重排写指令，导致先赋值lazyInstance字段后初始化赋值lazyInstance字段的字段。  

代码界有很多聪明虫，提出了改进，试图解决这个问题，他们的代码如下，看上去确实很聪明，以至于我看了几遍才搞明白他用意何在：  
```java 
private LazyClass lazyInstance ; 
public LazyClass getLazyInstance(){
  if(lazyInstance == null){
    LazyClass tmp = null ; 
    synchronized(this){
      if(lazyInstance == null)
        synchronized(this){
          tmp = new LazyClass();
        }
      lazyInstance = tmp ;
    }
  }
  return lazyInstance ;
}
```  
这里又创建了一个局部变量tmp，并且在内部同步块之外将tmp赋值给lazyInstance，用意在于，同步块会保证，在出同步块时，同步块之前的命令都是执行完毕的。如此，赋予lazyInstance的对象应该是已经初始化过的，因为tmp实例实在同步块里创建的。但是，但是，同步块的语义并不保证，同步块之后的指令会在同步块之后执行，也就是说，lazyInstance=tmp这句，有可能因为reorder被order到第二个同步块里面去，于是这个就退化成了第一种情况。。。。  

还有其他的尝试，


