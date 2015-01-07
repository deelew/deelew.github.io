**Object.wait**  
- object.wait() 将当前线程放置到一个wait set中  
- 调用该方法前，当前线程需要成功获取该对象的锁  
- wait调用的第二个语义是，释放线程对该对象持有的锁。仅仅释放该对象的锁
- 这里为了防止极少出现的 ‘伪唤醒(spurious wakeup)’ 需要在wait返回处再次判断wait的逻辑是否达到，示例代码：  
```java 
synchronized (obj) {
    while (<condition does not hold>)
        obj.wait(timeout);
         //...Perform action appropriate to condition
     }
```  

---

**Object.notify** 
- object.notify 将该对象wait set里的线程选择一个恢复到待调度队列，选择算法由语言实现定义，标准不做约束，具有随意性  
- 调用该方法前，当前线程需要成功获取该对象的锁  
- 被唤醒的线程，将重新竞争wait时释放掉的锁，竞争锁成功后，线程状态恢复到wait方法返回处，继续执行剩余逻辑  
**Object.notifyAll**  
类似Object.notify， 它将唤醒目标对象wait set里的所有线程  
