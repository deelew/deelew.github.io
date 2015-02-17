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
简单来讲，是由于指令重排（reordering）导致instance字段先赋值然后再执行行初始化构造方法。当赋值完成，且初始化未完成时，并发线程执行初次instance==null的判断为false，于是获取到了未初始化完成的LazyClass实例。  
通过展开字节码仔细查看下这个问题：  

```java
public LazyClass getLazyInstance();
  Code:
   Stack=3, Locals=3, Args_size=1
   0:   aload_0
   1:   getfield        #2; //Field instace:LLazyClass;
   4:   ifnonnull       39
   7:   aload_0
   8:   dup
   9:   astore_1
   10:  monitorenter
   11:  aload_0
   12:  getfield        #2; //Field instace:LLazyClass;
   15:  ifnonnull       29
   18:  aload_0
   19:  new     #3; //class LazyClass
   22:  dup
   23:  invokespecial   #4; //Method LazyClass."<init>":()V
   26:  putfield        #2; //Field instace:LLazyClass;
   29:  aload_1
   30:  monitorexit
   31:  goto    39
   34:  astore_2
   35:  aload_1
   36:  monitorexit
   37:  aload_2
   38:  athrow
   39:  aload_0
   40:  getfield        #2; //Field instace:LLazyClass;
   43:  areturn
```
从编码18到26的5行bytecode对应的是 lazyInstance= new LazyClass(); 这一句java instrument  
当乱序产生时，如果变成了  

```java 
1: aload_0
2: new     #3; //class LazyClass
3: astore_2 
4: aload_2
5: putfield        #2; //Field instace:LLazyClass;
6: aload_2
7: invokespecial   #4; //Method LazyClass."<init>":()V
```  

当然，这是人肉码的字节码，看上去指令更多了，简直就是劣化orz... 为了理解问题方便嘛。。 当执行完第5行指令时，字段instance 就!= null了，注意这时还没有调用<init>构造方法，如果并发线程执行第一个instance == null 结果为false，拿到了一个未正确初始化完成的实例。  

我们进一步展开讨论几个问题：为啥会发生TMD reordering ? 有没有改进版的DCL ? 有没有其他方法实现DCL的延迟初始化目的?   

**关于reordering**  
博主半道失足，基础知识甚不扎实，初次看到reordering的时候理解不能，感觉脚下坚实的大地突然变得柔软了，这TMD还怎么写代码？指令不按照寡人写的代码顺序来执行！实际上，细细看来，没有那么糟糕，软是软了点，但是还是可以找到正确的姿势奔跑的。主要分为三个层面：  

- compiler reordering  
  编译器有可能会对指令进行重排序，另外如果将方法调用内联了，那么内联方法的内容进一步增加了重排序的范围。
- machine instruction process reordering 
- cache/momery flush recordering  

reordering的底线在哪里？  
编译器、处理器、存储都确保as-if-serial 语义级别的有序，可以理解为，从执行结果看，与代码语义一致，并且是单线程执行的情况。

如果编译器推断，构造方法不会抛出异常以及没有做同步，就可能将构造函数内联调用，进一步对指令reorder。这里就发生了，lazyInstance字段先被赋值，然后再初始化的情况产生。但是synchronized是获取this的锁，而lazyInstance本身没法加锁，于是T2可以顺利的读取到lazyInstance 非null，于是就欢乐地去使用lazyInstance了，一个半成品的lazyInstance对象，于是整个线程就不好了。。。  
另外，即便没有做compile recorder， 在多处理器机器上，内存模块也有可能会重排写指令，导致先赋值lazyInstance字段后初始化赋值lazyInstance字段的字段。  

拉回到Java代码。。。   


代码界有很多聪明虫，提出了改进，试图解决这个问题，他们的代码如下，看上去确实很聪明，以至于我看了几遍才搞明白他用意何在：  
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
这里又创建了一个局部变量tmp，并且在内部同步块之外将tmp赋值给lazyInstance，用意在于，同步块会保证，在出同步块时，同步块之前的命令都是执行完毕的。如此，赋予lazyInstance的对象应该是已经初始化过的，因为tmp实例实在同步块里创建的。但是，但是，同步块的语义并不保证，同步块之后的指令会在同步块之后执行，也就是说，lazyInstance=tmp这句，有可能因为reorder被order到第二个同步块里面去，于是这个就退化成了第一种情况。。。。  

还有其他的尝试，

**参考资料**  
[The "Double-Checked Locking is Broken" Declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html) 对DCL失效的全面阐释  
[Synchronization and the Java Memory Model](http://gee.cs.oswego.edu/dl/cpj/jmm.html)关于同步、可见性  


