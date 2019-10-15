# objectmonitor

## 关于R大的一些回答

### 轻量级锁为什么要膨胀?

因为thin-lock没有足够空间来存储额外状态，所以在遇到某个锁实际上有contention的情况下，如果不膨胀，
实质上就要迫使所有在等待这把锁的线程都要spin-wait，有可能会非常非常低效。thin-lock膨胀到
fat-lock（HotSpot VM里的ObjectMonitor）就有了更多存储空间，可以存下诸如native mutex之类的OS
提供的同步原语对象的指针，可以在contended的情况下更高效地实现线程等待。
 


