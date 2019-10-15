# pthread_cond_wait的整体运转过程

参考: https://www.zhihu.com/question/24116967。 

主要看3个核心的函数。

- pthread_mutex_lock
- pthread_cond_wait
- pthread_mutex_unlock

简单的伪代码实现如下

```c++
    int status = pthread_mutex_lock(_mutex);                                   
    while (_Event < 0) {
      status = pthread_cond_wait(_cond, _mutex);                               
    } 
    status = pthread_mutex_unlock(_mutex);  
```

## _mutex到底是在保护谁?
首先不是在保护_cond的内部状态.假如说_cond真的需要保护, 完全可以在_cond内部实现一个锁来完成这个事情。 
用上边的例子来举例就是发现遍历_Event不满足要求, 所以现在调用pthread_cond_wait把当前线程挂起。期待
别的线程修改_Event的值。此时变量_Event自然是多个线程之间共享的。所以本意是使用_mutex来保护_Event. 

## _mutex的其他作用?
pthread_cond_wait内部其实可以分为5部分。

- 把线程放到阻塞队列中
- _mutex.unlock()
- 阻塞
- 唤醒
- _mutex.lock()

由于pthread_cond_wait可能会阻塞。当即将阻塞时, _mutex必须先释放掉(否则一直不释放对于其他线程来说就
是个悲剧)。然后当重新被唤醒再重新lock。

进一步来说, 如果没有这个_mutex, 那么pthread_cond_wait()第一步还没放到阻塞队列另外一个线程就修改条件
进行唤醒, 次数当前线程由于还没到阻塞队列, 就不能够被唤醒了。
 
## example

这个小例子可能与本文关系不是很密切。暂时只是为了加深对于这几个函数的理解。

```c++
    #include <pthread.h>
    #include <stdio.h>
    #include <unistd.h>
    
    pthread_mutex_t mutex ;
    void *print_msg(void *arg){
            int i=0;
            pthread_mutex_lock(&mutex);
            for(i=0;i<15;i++){
                    printf("output : %d\n",i);
                    usleep(1500000);
            }
            pthread_mutex_unlock(&mutex);
    }
    int main(int argc,char** argv){
            pthread_t id1;
            pthread_t id2;
            pthread_mutex_init(&mutex,NULL);
            pthread_create(&id1,NULL,print_msg,NULL);
            pthread_create(&id2,NULL,print_msg,NULL);
            pthread_join(id1,NULL);
            pthread_join(id2,NULL);
            pthread_mutex_destroy(&mutex);
            return 1;
    }
```
 
