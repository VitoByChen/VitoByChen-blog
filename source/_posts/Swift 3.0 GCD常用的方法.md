---
title:  Swift 3.0 GCD常用的方法
---
####导语:

>随着最近苹果推出`Swift 3.0` 觉得不能把Swift丢下了,就赶在最近学了一下Swift, 然后把`Swift 3.0` GCD 部分稍微的总结一下


为了方便我定义了几个属性

```
class ViewController:UIViewController{  
    /**队列*/  
  var myQueue:DispatchQueue?   
  var myQueueTimer:DispatchQueue?  
  var mnytimer:DispatchSourceTimer?   
  var myGroup:DispatchGroup?   
  var mySource:DispatchSource?  
  override fun viewDidLoad() {  
     super.viewDidLoad()       
    GCDTest1()      
    GCDTest8()    
  }
}
```


####定义队列
```
//MARK: - 创建队列
        //1.
        myQueue = DispatchQueue(label: "第一条线程")
        
/*
 - parameter  qos:DispatchQoS
         线程的策略
        case background  //后台
        
        case utility     //公共的
        
        case `default`    //默认的
        
        case userInitiated  //用户期望优先级（不要放太耗时的操作）
        
        case userInteractive  //用户交互(跟主线程一样)
        
        case unspecified    //不指定
 */
         //2.
        myQueue = DispatchQueue(label: "第二条线程", qos: .default, attributes: .concurrent, autoreleaseFrequency: .workItem, target: nil)
   ```     
创建好的队列执行任务
```
        myQueue?.sync(execute: { 
            print("执行同步任务")
        })
        
        myQueue?.async(execute: { 
            print("执行异步任务")
        })
```
   串行执行队列
   
```
         myQueue?.async {
            for _ in 0...10 {
                print("aaaaaaa");
            }
            
        }
        myQueue?.async {
            for _ in 0...10 {
                print("bbbbbbb");
            }
        }
        ```
接下来是GCD常用的用法
1. 开线程异步执行完耗时代码返回主线程刷新UI
```
 func GCDTest2() {
    /**1. 开线程异步执行完耗时代码返回主线程刷新UI*/
        DispatchQueue.global().async {
            print("开一条全局队列异步执行任务")
            
            DispatchQueue.main.async {
                print("在主队列执行任务")
            }
        }
    }
```
```
2. 等待异步执行多个任务后, 再执行下一个任务

等待异步执行多个任务后, 再执行下一个任务*/
    func GCDTest3() {
        
        myQueue?.async {//任务一
            for _ in 0...10 {
                print("......")
            }
        }
        myQueue?.async {//任务二
            for _ in 0...10 {
                print("++++++");
            }
        }
        
        // barrier 会等待上面执行完毕再执行下面的，会阻塞当前线程
        //        myQueue?.async(flags:.barrier ,execute: {//1.
        //            print("000000")
        //        })
        
        myQueue?.async(group: nil, qos: .default, flags: .barrier, execute: {//2.
            print("000000")
        })
        
        myQueue?.async {
            print("111111")
        }
        /* 打印的结果
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         ++++++
         ......
         000000
         111111
         */
    }

```

 延时提交任务 
func GCDTest4() {
        //主队列
       DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 10) {
        print("延时提交的任务")
        }
    
        //指定队列
        myQueue?.asyncAfter(deadline: DispatchTime.now() + 10, execute: {
            print("延时提交的任务")
        })
    }
    
```
```
信号量:

  func GCDTest5() {
        //初始化信号量, 计数为三
    let mySemaphore = DispatchSemaphore(value: 3)
        for i in 0...10 {
            print(i)
//            let _ = mySemaphore.wait()  //获取信号量，信号量减1，为0时候就等待,会阻碍当前线程
            let _ = mySemaphore.wait(timeout: DispatchTime.now() + 2.0) //阻碍时等两秒信号量还是为0时将不再等待, 继续执行下面的代码
            myQueue?.async {
                for j in 0...4 {
                 print("有限资源\(j)")
                sleep(UInt32(3.0))
                }
                print("-------------------")
            mySemaphore.signal()
            }
            
        }
    
    }
```
信号量的说明:
GCD 信号量控制并发 （dispatch_semaphore）
     当我们在处理一系列线程的时候，当数量达到一定量，在以前我们可能会选择使用NSOperationQueue来处理并发控制，但如何在GCD中快速的控制并发呢？答案就是dispatch_semaphore。
     信号量是一个整形值并且具有一个初始计数值，并且支持两个操作：信号通知和等待。当一个信号量被信号通知，其计数会被增加。当一个线程在一个信号量上等待时，线程会被阻塞（如果有必要的话），直至计数器大于零，然后线程会减少这个计数。
     在GCD中有三个函数是semaphore的操作，分别是：
     1、dispatch_semaphore_create　　　创建一个semaphore
     2、dispatch_semaphore_signal　　　发送一个信号
     3、dispatch_semaphore_wait　　　　等待信号
     
     下面我们逐一介绍三个函数：
     
     （1）dispatch_semaphore_create的声明为：
     　　dispatch_semaphore_t dispatch_semaphore_create(long value);
     　　传入的参数为long，输出一个dispatch_semaphore_t类型且值为value的信号量。值得注意的是，这里的传入的参数value必须大于或等于0，否则dispatch_semaphore_create会返回NULL。
     
     （2）dispatch_semaphore_signal的声明为：
     　　long dispatch_semaphore_signal(dispatch_semaphore_t dsema)这个函数会使传入的信号量dsema的值加1；（至于返回值，待会儿再讲）
     
     (3) dispatch_semaphore_wait的声明为：
     　　long dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout)；
     这个函数会使传入的信号量dsema的值减1。这个函数的作用是这样的，如果dsema信号量的值大于0，该函数所处线程就继续执行下面的语句，并且将信号量的值减1；如果desema的值为0，那么这个函数就阻塞当前线程等待timeout（注意timeout的类型为dispatch_time_t，不能直接传入整形或float型数），如果等待的期间desema的值被dispatch_semaphore_signal函数加1了，且该函数（即dispatch_semaphore_wait）所处线程获得了信号量，那么就继续向下执行并将信号量减1。如果等待期间没有获取到信号量或者信号量的值一直为0，那么等到timeout时，其所处线程自动执行其后语句。
     
     （4）dispatch_semaphore_signal的返回值为long类型，当返回值为0时表示当前并没有线程等待其处理的信号量，其处理的信号量的值加1即可。当返回值不为0时，表示其当前有（一个或多个）线程等待其处理的信号量，并且该函数唤醒了一个等待的线程（当线程有优先级时，唤醒优先级最高的线程；否则随机唤醒）。
     　　dispatch_semaphore_wait的返回值也为long型。当其返回0时表示在timeout之前，该函数所处的线程被成功唤醒。当其返回不为0时，表示timeout发生。
     
     （5）关于信号量，一般可以用停车来比喻。
     　　停车场剩余4个车位，那么即使同时来了四辆车也能停的下。如果此时来了五辆车，那么就有一辆需要等待。信号量的值就相当于剩余车位的数目，dispatch_semaphore_wait函数就相当于来了一辆车，dispatch_semaphore_signal就相当于走了一辆车。停车位的剩余数目在初始化的时候就已经指明了（dispatch_semaphore_create（long value）），调用一次dispatch_semaphore_signal，剩余的车位就增加一个；调用一次dispatch_semaphore_wait剩余车位就减少一个；当剩余车位为0时，再来车（即调用dispatch_semaphore_wait）就只能等待。有可能同时有几辆车等待一个停车位。有些车主没有耐心，给自己设定了一段等待时间，这段时间内等不到停车位就走了，如果等到了就开进去停车。而有些车主就像把车停在这，所以就一直等下去。

重复提交任务(定时器)

```
func GCDTest6() {
        
        //      秒               毫秒                      微秒                      纳秒
        //  1 seconds = 1000 milliseconds = 1000,000 microseconds = 1000,000,000 nanoseconds
        myTimer = DispatchSource.makeTimerSource(flags: [], queue: myQueue)
        myTimer?.scheduleRepeating(deadline: .now(), interval: .seconds(1) ,leeway:.milliseconds(100))
        myTimer?.setEventHandler {
            print("fff")
        }
        myTimer?.resume()
//        myTimer?.cancel()
//        myTimer?.activate()
       
    }
```

####接下来就是Group的用法
```

1. notify(依赖任务)
  //MARK: - notify(依赖任务)
    func GCDTest7() {
      let group = DispatchGroup()
        myQueue?.async(group: group, qos: .default, flags: [], execute: { 
            for _ in 0...10 {
            
            print("耗时任务一")
            }
        })
        myQueue?.async(group: group, qos: .default, flags: [], execute: {
            for _ in 0...10 {
                
                print("耗时任务二")
            }
        })
        //执行完上面的两个耗时操作, 回到myQueue队列中执行下一步的任务
        group.notify(queue: myQueue!) {
            print("回到该队列中执行")
        }
       
    }
```

2. wait(任务等待)

```
func GCDTest8() {
        let group = DispatchGroup()
        myQueue?.async(group: group, qos: .default, flags: [], execute: {
            for _ in 0...10 {
                
                print("耗时任务一")
            }
        })
        myQueue?.async(group: group, qos: .default, flags: [], execute: {
            for _ in 0...10 {
                
                print("耗时任务二")
                sleep(UInt32(3))
            }
        })
         //等待上面任务执行，会阻塞当前线程，超时就执行下面的，上面的继续执行。可以无限等待 .distantFuture
        let result = group.wait(timeout: .now() + 2.0)
        switch result {
        case .success:
            print("不超时, 上面的两个任务都执行完")
        case .timedOut:
            print("超时了, 上面的任务还没执行完执行这了")
        }
        
        print("接下来的操作")
        
    }
```

3. enter leave 手动管理group计数,enter和leave必须配对

```
func GCDTest9() {
    
        let group = DispatchGroup()
        group.enter()//把该任务添加到组队列中执行
        myQueue?.async(group: group, qos: .default, flags: [], execute: {
            for _ in 0...10 {
                
                print("耗时任务一")
                group.leave()//执行完之后从组队列中移除
            }
        })
         group.enter()//把该任务添加到组队列中执行
        myQueue?.async(group: group, qos: .default, flags: [], execute: {
            for _ in 0...10 {
                
                print("耗时任务二")
                group.leave()//执行完之后从组队列中移除
            }
        })
        
        //当上面所有的任务执行完之后通知
        group.notify(queue: .main) { 
            print("所有的任务执行完了")
        }
    
    }
```
以上就是个人对swift 3.o GCD 的理解