**引子  ** 排查一个问题的时候，涉及到了DCL，然后很久之前就看到过关于DCL失效的问题，但是细节没记住，准确说没有真正理解，于是又翻阅了下资料，顺道梳理下。
**DCL idiom**  
```java 
private LazyClass lazyInsntance = null ;

public LazyClass getLazyClass(){
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

