# Evenloop

  std::unique_ptr<Poller> poller_;

  std::unique_ptr<TimerQueue> timerQueue_;

  ChannelList activeChannels_;

  Channel* currentActiveChannel_;

---

Loop(): 从poll中获取activeChannels_ 然后遍历activeChannels_来调用Channel的handleEvent

runInLoop(): 看不懂



# TimeQueue

  Channel timerfdChannel_; 与Channel绑定，回调函数是handleRead()

---

handleRead(): 从getExpired中获取过期的Timer，然后执行他们

