# Kotlin学习摘要


##协程相关
###概念
特征：协程是运行在单线程中的并发程序
优点：省去了传统 Thread 多线程并发机制中切换线程时带来的线程上下文切换、线程状态切换、Thread 初始化上的性能损耗，能大幅度唐提高并发性能
简单理解：在单线程上由程序员自己调度运行的并行计算

```
##lauch 与 runBlocking
launch (CoroutineScope.lauch / GlobalScope.lauch) 是非阻塞的 而 runBlocking 是阻塞的

runblocking 的线程会阻塞直到内部的协程执行完毕
```

```
##withContext 与 async 
多个 withContext 任务是串行的
场景：withConext可以用在一个请求结果依赖另一个请求结果的这种情况

多个 async 任务是并行的，async 返回的是一个Deferred<T>，需要调用其await()方法获取结果

async 后面使用 await() 就和 withContext 一样
await() 只有在 async 未执行完成返回结果时，才会挂起协程。若 async 已经有结果了，await() 则直接获取其结果并赋值给变量，此时不会挂起协程。	
```
```
coroutineScope 是挂起函数。
挂起函数在执行完成之后，协程会重新切回它原先的线程
```

```
Activity或Fragment中使用协程时，要尽量避免使用GlobalScope
使用androidx.lifecycle:lifecycle-runtime-ktx:2.2.0-alpha01或更高版本，会用LifecycleScope代替GlobalScope

又或者

onCreate()方法里面创建CoroutineScope对象
onDestroy()方法里面取消CoroutineScope对象
```

##关键字
```
lateinit 只用于变量 var，而 lazy 只用于常量 val
lazy 应用于单例模式(if-null-then-init-else-return)，而且当且仅当变量被第一次调用的时候，委托方法才会执行。
```
```
val + get() 每次都会执行get的语法
```
```
@JvmField 注解的作用是 "指示 Kotlin 编译器不要为这个属性生成 getter 和 setter 方法，并将其作为一个成员变量允许其被公开访问"
```
