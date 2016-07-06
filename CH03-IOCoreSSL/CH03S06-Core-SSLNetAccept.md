# 核心组件 SSLNetAccept

SSLNetAccept继承自NetAccept，它对NetAccept的部分方法进行了重载，实现了SSLNetAccept的功能。

前面讲到NetVC的创建有两种方式：

  - 一种通过NetAccept
  - 一种通过connectUp

因此对于每一种NetVC及其继承子类，都有对应的NetAccept和connectUp的实现。

而SSLNetAccept则是对应NetAccept，用于创建SSLNetVConnection

下面的讲解将着重介绍被重载的方法，因此请对照NetAccept的章节一起理解SSLNetAccept的实现

## 定义

```
//
// NetAccept
// Handles accepting connections.
//
struct SSLNetAccept : public NetAccept {
  virtual NetProcessor *getNetProcessor() const;
  virtual EventType getEtype() const;
  virtual void init_accept_per_thread(bool isTransparent);
  virtual NetAccept *clone() const;

  SSLNetAccept(){};

  virtual ~SSLNetAccept(){};
};

typedef int (SSLNetAccept::*SSLNetAcceptHandler)(int, void *);
```

## 方法

### SSLNetAccept::getNetProcessor()

此方法用于返回NetProcessor的全局类型实例。

```
NetProcessor *
SSLNetAccept::getNetProcessor() const
{
  return &sslNetProcessor;
}
```

### SSLNetAccept::getEtype()

在NetAccept中，此方法用于返回 etype 成员，etype成员用于表示此NetAccept创建的NetVC后，会把此NetVC交由哪一个线程组负责。

在ATS的设计中，存在一个独立的线程组 ET_SSL，专门处理 SSLNetVConnection 类型的连接。

但是当 ET_SSL 专用的线程组在配置文件中被设置为 0 时：

  - 那么 ET_SSL 就被设置为与 ET_NET 相同的值
  - 因此 UnixNetVConnection 和 SSLNetVConnection 将由同一个线程组处理

此方法固定返回 ET_SSL。

为何不是返回 etype 成员呢？？？

```
// Virtual function allows the correct
// etype to be used in NetAccept functions (ET_SSL
// or ET_NET).
EventType
SSLNetAccept::getEtype() const
{
  return SSLNetProcessor::ET_SSL;
}
```

### SSLNetAccept::init_accept_per_thread(bool isTransparent)

为 ET_SSL 线程组创建并初始化SSLNetAccept状态机。

前面讲过 NetAccept 有两种运行模式，独立线程模式和状态机模式，这个就是采用状态机模式，要在线程组中创建对应的状态机。

period 是负数，因此会进入隐性队列（负队列）。

```
void
SSLNetAccept::init_accept_per_thread(bool isTransparent)
{
  int i, n;
  NetAccept *a;

  if (do_listen(NON_BLOCKING, isTransparent))
    return;
  if (accept_fn == net_accept)
    SET_HANDLER((SSLNetAcceptHandler)&SSLNetAccept::acceptFastEvent);
  else
    SET_HANDLER((SSLNetAcceptHandler)&SSLNetAccept::acceptEvent);
  period = -HRTIME_MSECONDS(net_accept_period);
  n = eventProcessor.n_threads_for_type[SSLNetProcessor::ET_SSL];
  for (i = 0; i < n; i++) {
    if (i < n - 1)
      a = clone();
    else
      a = this;
    EThread *t = eventProcessor.eventthread[SSLNetProcessor::ET_SSL][i];

    PollDescriptor *pd = get_PollDescriptor(t);
    if (ep.start(pd, this, EVENTIO_READ) < 0)
      Debug("iocore_net", "error starting EventIO");
    a->mutex = get_NetHandler(t)->mutex;
    t->schedule_every(a, period, etype);
  }
}
```

### SSLNetAccept::clone()

这个方法用于上面的SSLNetAccept::init_accept_per_thread()，与NetAccept::clone()的不同之处就是：

  - new SSLNetAccept
  - 创建的是SSLNetAccept对象。

```
NetAccept *
SSLNetAccept::clone() const
{
  NetAccept *na;
  na = new SSLNetAccept;
  *na = *this;
  return na;
}
```

## 参考资料

- [P_SSLNetAccept.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNetAccept.h)
- [SSLNetAccept.cc](https://github.com/apache/trafficserver/tree/master/iocore/net/SSLNetAccept.cc)
