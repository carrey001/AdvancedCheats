# RxJava
---

RxJava 操作符
以下运算符是distinct rxjava-async模块的一部分。它们用于将同步方法转换为Observable。

- start( ) - 创建一个发射函数的返回值的Observable
- toAsync( )或asyncAction( )或asyncFunc( ) -转换功能或操作成一个可观察的执行函数，并将其返回值
- deferFuture( ) - 将一个返回一个Observable的Future转换为一个Observable，但不要尝试获取Future将返回的Observable，直到订阅者订阅
- fromAction( ) - 将一个Action转换为一个Observable，当订阅者订阅时调用该操作并发出其结果
- fromCallable( ) - 将一个Callable转换为一个Observable，调用该可调用，并在订阅者订阅时发出其结果或异常
- fromRunnable( ) - 将Runnable转换为调用runable的Observable，并在订阅订阅时发出其结果
- runAsync( )- 返回一个StoppableObservable发出由调度程序上指定的Action生成的多个操作



本节介绍可用于合并多个Observable的运算符。

- startWith( ) - 在开始从Observable中发出项目之前，发出指定的项目序列
- merge( ) - 将多个Observable组合成一个
- mergeDelayError( ) - 将多个Observables组合成一个，允许无错误的Observable在传播错误之前继续
- zip( ) - 通过指定的函数将由两个或多个Observable一起发送的项组合在一起，并根据此函数的结果发出项
- （rxjava-joins）and( )，then( )和when( ) - 组合由两个或更多个Observables通过Pattern和Plan中间发射的项目集
- combineLatest( ) - 当项由两个Observable中的任一个发出时，通过指定的函数组合每个Observable发出的最新项，并基于此函数的结果发出项
- join( )以及groupJoin( ) - 当来自一个观测器的一个项目落入由另一个观测器发射的项目所指定的持续时间的窗口内时，组合由两个观测器发射的项目
- switchOnNext( ) - 将一个发出Observable的Observable转换为一个Observable，发出由最近发出的那些Observable发射的项目

条件运算符

- amb( ) - 给定两个或多个源Observable，从这些Observable的第一个发出所有项目以发射项目
- defaultIfEmpty( ) - 从源Observable中发射项目，或者如果源Observable在没有发射项目之后完成，则发射默认项目
- （rxjava-computation-expressions）doWhile( )- 发出源Observable的序列，然后只要条件保持为真，重复序列
-（rxjava-computation-expressions）ifThen( )- 如果条件为真，则只发出源Observable的序列，否则发出空或默认序列
- skipUntil( ) - 丢弃源发射的项Observable直到第二个Observable发射一个项，然后发射源的其余部分Observable的项
- skipWhile( ) - 丢弃Observable发出的项，直到指定的条件为false，然后发出余数
- （rxjava-computation-expressions）switchCase( )- 基于评估的结果从特定的Observable发出序列
- takeUntil( ) - 从源Observable发出项目，直到第二个Observable发出项目或发出通知
- takeWhile( )和takeWhileWithIndex( ) - 只要指定的条件为真，就发出Observable发出的项目，然后跳过剩余部分
- （rxjava-computation-expressions）whileDo( )- 如果条件为真，发出源Observable的序列，然后只要条件保持为真，重复序列


布尔运算符

- all( ) - 确定Observable发出的所有项目是否满足一些标准
- contains( ) - 确定Observable是否发出特定项
- exists( )和isEmpty( ) - 确定观测器是否发射任何项目
- sequenceEqual( ) - 测试两个Observables发出的序列的相等性


此页面显示可用于过滤和选择由Observable发出的项目的运算符。

-    filter( ) - 过滤由Observable发出的项目
-    takeLast( )- 只发射由Observable 发射的最后n个项
-    last( ) - 只发射一个Observable发出的最后一个项目
-    lastOrDefault( ) - 只发出由Observable发出的最后一个项目，如果源Observable为空，则为默认值
-    takeLastBuffer( )- 将由Observable 发出的最后n个项目作为单个列表项发出
-    skip( )- 忽略Observable发出的前n个项
-    skipLast( )- 忽略Observable发出的最后n个项
-    take( )- 只发出由Observable 发出的前n个项
-    first( )以及takeFirst( ) - 仅发射由观察者发射的第一项目或满足一些条件的第一项目
-    firstOrDefault( ) - 仅发出由Observable发出的第一个项目，或者满足某个条件的第一个项目，或者如果源Observable为空，则为默认值
-    elementAt( )- emit 由source Observable 发出的项目n
-    elementAtOrDefault( )- emit 由Source Observable 发出的项目n，或者如果源Observable发出少于n个项目，则为默认项目
-    sample( )或throttleLast( ) - 在周期性时间间隔内发射由Observable发射的最近的项目
-    throttleFirst( ) - 在周期性时间间隔内发射由Observable发射的第一项
-    throttleWithTimeout( )或者debounce( ) - 只有在特定时间跨度已经过去之后才从源Observable发出一个项，而Observable不发出任何其他项
-    timeout( ) - 从源Observable发射项目，但如果在指定的时间范围内没有发射项目，则发出异常
-    distinct( ) - 抑制源Observable发出的重复项目
-    distinctUntilChanged( ) - 抑制源Observable发出的重复连续项
-    ofType( ) - 只从那些属于特定类的源Observable中发出那些项
-    ignoreElements( ) - 丢弃源Observable发出的项目，并且只传递错误或完成的通知
