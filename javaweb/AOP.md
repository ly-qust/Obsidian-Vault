![[Pasted image 20251128084052.png]]![[Pasted image 20251128091232.png]]![[Pasted image 20251128091912.png]]
![[Pasted image 20251128092348.png]]![[Pasted image 20251128110517.png]]

@PointCut可以提供抽取公共表达式，提高代码复用性
```java
@Pointcut("execution(* com.itheima.service.*.*(..))") 
private void pt(){}
```


![[Pasted image 20251128111707.png]]

如果有的类没有Order(),先执行有的，再执行没有的



![[Pasted image 20251128112501.png]]![[Pasted image 20251128112920.png]]
![[Pasted image 20251128125538.png]]
- 优先考虑execution


![[Pasted image 20251128130416.png]]


![[Pasted image 20251128140730.png]]
## 基于ThreadLocal来完成从token中取数据的操作，适用情况是在同一个线程/请求中共享数据
![[Pasted image 20251128141754.png]]