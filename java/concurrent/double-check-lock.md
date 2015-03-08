**引子** 
排查一个问题的时候，涉及到了DCL，之前就看到过关于‘double-chek-lock’(DCL)失效的问题，但是细节没记住，准确说没有真正理解，于是又翻阅了下资料，顺道梳理下。  

**DCL idiom**  
DCL通常是用来延迟对象初始化的，大概是这个样子：  
```java 
public class DCLTest {
    private LazyClass instace = null;
    public LazyClass getLazyInstance() {
        if (instace == null) {
            synchronized (this) {
                if (instace == null) {
                    instace = new LazyClass();
                }
            }
        }
        return instace;
    }
}
class LazyClass {
}
```  
**失效根源**   
简单来讲，是由于指令重排（reordering）导致instance字段先赋值然后再执行行初始化构造方法。用伪指令表示下第二层check：
```
0: load instace
1: cmp instance , null
2: allocate LazyInstance 
3: invoke LazyInstance.<init>
4: store instance 
```
编译重排序可以把3、4条指令调换顺序，重排后变成：
```
0: load instace
1: cmp instance , null
2: allocate LazyInstance 
3: store instance 
4: invoke LazyInstance.<init>
```
这时，如果threadA 执行完毕指令3，待执行4，threadB执行第一个if(instance==null)判断不成立，于是读取instance对其进行访问。但是由于instance指向的是尚未执行指令4的一个实例，因此数据是非法的，就出问题了。  

编译器为什么要做这种重排序呢？   

**关于reordering**  
重排序的根本目的在于提高代码执行速度。随着cpu、cache的复杂度越来越高，一些指令顺序的改变对指令的执行效率有明显的影响，因而发展出了多种指令重排序技术。
包括编译期重排序，重排序主要分为三个层面：  
- **compiler reordering**  
  编译器有可能会对指令进行重排序，另外如果将方法调用内联了，那么内联方法的内容进一步增加了重排序的范围。指令重排只是[Optimizing compiler](http://en.wikipedia.org/wiki/Optimizing_compiler#Data-flow_optimizations)的优化策略之一。对读写指令重排序主要是增强数据局部性（data locality），从而提高缓存命中率。从上述伪指令可以看出，通过调换3/4条指令，使得store instance 与0指令更加接近
- **machine instruction process reordering**  
  通过重排执行指令，提高管线（pipeline)执行的效率
- **cache/memory flush recordering**  
  多层、大容量cache/memory也可能通过批量、重排读写操作提高读写效率、缓存命中率  

如此复杂的重排序，岂不是把代码逻辑都搞乱了？重排序的底线在哪里？  
虽然存在多个重排序的方面，但是所有的重排序都确保as-if-serial 语义级别的有序，可以理解为，从执行结果看，重排序的代码执行结果与单线程执行未重排序代码代码语义一致。也就是说，对于单线程顺序执行的代码，重排序完全不会有影响，但是多线程情况下就不同了！  

___了解了重排序以及其边界，再拉回到Java代码___  

**改进版DCL**  
代码界有很多聪明虫，提出了改进，试图解决这个问题，一种版本的代码如下：  
```java 
public class DCLTest {
    private LazyClass instace = null;
    public LazyClass getLazyInstance() {
        if (instace == null) {
            synchronized (this) {
                LazyClass tmp = instance ;
                if (tmp == null) {
                    synchronized(this){
                        tmp = new LazyClass();
                    }
                }
                instance = tmp ;
            }
        }
        return instace;
    }
}
class LazyClass {
}
```  
这里又创建了一个局部变量tmp，并且在内部同步块之外将tmp赋值给lazyInstance，用意在于，JLS规定同步块会保证，在代码出同步块时，同步块之前的命令都是执行完毕的。如此，赋予lazyInstance的对象应该是已经初始化过的，因为tmp实例是在同步块里创建的。但是，但是，同步块的语义并不保证，同步块之后的指令会在同步块之后执行，也就是说，lazyInstance=tmp这句，有可能因为reorder被order到第二个同步块里面去，于是这个就退化成了第一种情况。。。。  

在Java代码里，if(instance==null)判断失效的原因在于!=null并不等价于值就是正确的，但是当instance是一个32bit的原生数据类型时，这种情况就不存在了，也就是说，对于32bit原生类型，赋值操作不会被重排序，因此DCL是正确地：
```
class Foo { 
  private int cachedHashCode = 0;
  public int hashCode() {
    int h = cachedHashCode;
    if (h == 0) 
    synchronized(this) {
      if (cachedHashCode != 0) return cachedHashCode;
      h = computeHashCode();
      cachedHashCode = h;
      }
    return h;
    }
```  

另外，在jdk5之后，jls进一步定义了内存模型，也就是jsr-133，其中增强了volatile的语义，规定读写由volatile修饰的指令，不得与其前面的任何读写指令重排序，也不可与其后的任何读写指令重排序。于是只要加上volatile修饰，问题就解决了：
```
class Foo {
    private volatile Helper helper = null;
    public Helper getHelper() {
        if (helper == null) {
            synchronized(this) {
                if (helper == null)
                    helper = new Helper();
            }
        }
        return helper;
    }
}
```  

在JSR133之前，为了实现lazy initialize的效果，有种结合ThreadLocal guard variable的实现方法：
```
class Foo {
    /** If perThreadInstance.get() returns a non-null value, 
    this thread has done synchronization needed to see initialization of helper 
	**/
    private final ThreadLocal perThreadInstance = new ThreadLocal();
    private Helper helper = null;
    public Helper getHelper() {
        if (perThreadInstance.get() == null) createHelper();
        return helper;
    }
    private final void createHelper() {
        synchronized(this) {
            if (helper == null)
                helper = new Helper();
        }
        perThreadInstance.set(perThreadInstance);
    }
}
```  
helper == null 的判断以及赋值在同一个guard block里，每个线程通过perThreadInstance 开关减少执行createHelper的可能，减少了同步块被执行的次数，很聪明。。。


**参考资料**  
[The "Double-Checked Locking is Broken" Declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html) 对DCL失效的全面阐释  
[Synchronization and the Java Memory Model](http://gee.cs.oswego.edu/dl/cpj/jmm.html)关于同步、可见性  
[Memory Ordering at Compile Time](http://preshing.com/20120625/memory-ordering-at-compile-time)

