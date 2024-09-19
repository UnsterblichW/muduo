# Muduo 学习笔记
<!-- TOC -->
- [Muduo 学习笔记](#muduo-学习笔记)
  - [定时器 `TimerQueue` 的原理](#定时器-timerqueue-的原理)
<!-- TOC -->

## 定时器 `TimerQueue` 的原理
* 用来区分每个定时器对象的类 TimerId，每次调用`TimerQueue::addTimer`就会返回一个独一无二的TimerId
    ```C++
    class TimerId : public muduo::copyable
    {
    public:
        TimerId()
            : timer_(nullptr),
            sequence_(0)
        {
        }

        TimerId(Timer* timer, int64_t seq)
            : timer_(timer),
            sequence_(seq)
        {
        }

        // default copy-ctor, dtor and assignment are okay

        friend class TimerQueue;

    private:
        Timer* timer_;
        int64_t sequence_;
    };

    ```
* 获取当前时间用的是 gettimeofday ，这个接口的粒度是 1us
  ```C++
    Timestamp Timestamp::now()
    {
        struct timeval tv;
        gettimeofday(&tv, NULL);
        int64_t seconds = tv.tv_sec;
        return Timestamp(seconds * kMicroSecondsPerSecond + tv.tv_usec);
    }
  ``` 
> muduo的定时任务没有使用sleep()，因为muduo是个非阻塞的服务端库，sleep()或类似的让程序原地停留等待的做法，会让服务端程序失去响应，因为主事件循环被挂起了，无法处理IO事件。
**muduo使用 `timefd_***()` 系列接口来处理定时任务**
1. 用`timerfd_create`创建`timerfd`，详见`createTimerfd()`
2. 将这个`timerfd`注册到主事件循环中，`TimerQueue`会记录下这个`timerfd`，并且绑定读回调`TimerQueue::handleRead()`，详见`TimerQueue`的构造函数；
3. `resetTimerfd()`之中会调用`timerfd_settime`将`timerfd`描述的定时器启动，当定时器到期后，会触发`timerfd`的可读事件，因为`timerfd`已经被主事件循环关注了，将会通知`TimerQueue::handleRead()`，此时会调用所有到期的`Timer`，并且重置曾记录的Timer列表

需要着重理解一下`timers_`和`activeTimers_`的作用
1. 这两者都是set，都是用来管理Timer的，但是timers_是按照Timestamp来排序（就是按照定时器的过期时间来排序），activeTimers_是按照Timer*对象地址来排序的
2. activeTimers_的主要是用于让使用者可以主动取消某个定时器
3. 模拟一下场景：
   * `timerfd`的读回调触发了，走到`TimerQueue::handleRead()`里面，走到L174的循环里面，此时已经通过`getExpired()`找出了now时刻所有到期的Timer，而且也已经从`timers_`和`activeTimers_`中erase掉了这些Timer
   * 假设`TimerQueue::handleRead()`还处于L174的循环之中时，外部使用者想要删掉某个Timer，也就是调用了`TimerQueue::cancel`，最终会走到`TimerQueue::cancelInLoop`里面，此时删除操作和到期触发的操作发生在同一个loop里面。此时又有两种情况了
     1. 外部使用者想要删除的这个**Timer还没到期**，那么`TimerQueue::cancelInLoop`就会走到L151里面去
     2. 外部使用者想要删除的这个**Timer已经到期了**，那么由于`getExpired()`里面已经把到期的Timer都erase掉了，那么，那么`TimerQueue::cancelInLoop`就会走到L158里面去，记录下在本次调用到期Timer期间，有哪些需要cancel掉的Timer，存到`cancelingTimers_`这个set里面，`cancelingTimers_`和`activeTimers_`是同一个set结构
     3. 最后当`TimerQueue::handleRead()`运行到`TimerQueue::reset`里面的时候，会根据当前这批已经到期的Timer（参数表expired传递的这些Timer）是否在`cancelingTimers_`里面，来判断是否要重新把Timer添加到`timers_`和`activeTimers_`里面，因为有些Timer并不是只调用了一次就失效的，比如说由`EventLoop::runEvery`创建的Timer需要每隔一段时间触发，所以才需要在`TimerQueue::reset`里面重新过期的Timer加回来