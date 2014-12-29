**enum 变量赋任意值/enum “匿名”**
```java
#include<stdio.h>
enum {a,b,c} etest;
//enum etype {a,b,c} 
/** 
TestEnum.c:3: error: redeclaration of enumerator `a'
TestEnum.c:2: error: previous definition of 'a' was here
TestEnum.c:3: error: redeclaration of enumerator `b'
TestEnum.c:2: error: previous definition of 'b' was here
TestEnum.c:3: error: redeclaration of enumerator `c'
TestEnum.c:2: error: previous definition of 'c' was here
**/
enum etype {aa, bb, cc};
int main(){
enum etype evar = aa; 
evar = 10 ; 
printf("%d",evar);
}
$ gcc ok 
//output: 10 
```
