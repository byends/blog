---
type: post
category: php
date: 2018-06-13
title: 谁才是PHP实现延迟消息队列的最佳CP？
meta:
  - name: description
    content: PHP 延迟消息队列的选择
---

注：本文首发于[知乎](https://zhuanlan.zhihu.com/p/32149460)

### 延迟消息
大白话来说，就是实现一种计划任务的定时机制，希望在设定的时间到达后才触发发送消息。

### 使用场景
举两个使用场景：  
- 人为的控制，比如订单系统里，用户下单后规定，30分钟内没有支付，则自动取消该订单。  
- 程序处理比较耗时，比如发邮件，当邮件内容比较大、收件人多或者网络不好等，都有可能导致比较耗时，无法立即返回发送状态。

<!-- more -->

### 初步的解决方案
Linux 里有 Crontab 定时任务，Windows 也有计划任务。  
- 使用定时任务每分钟执行一次PHP脚本；  
- 该脚本根据当前时间去查询数据表，把符合条件的记录（即时间已到的记录）查出来发送即可。

那么，这里要思考的问题是，如果每条记录因业务场景不同可能会比较耗时，如果不做处理则会阻塞后面的消息送达，还有可能因为脚本中断导致后续消息记录无法发送，轻则影响后续消息的发送时间，重则导致大量消息记录积压。那么，此时需要做进一步处理，把查出来的消息记录扔进 Redis 队列，需要另起PHP进程去轮询 Redis 队列，取出消息来发送。

### 新的问题
一、由于PHP无法实现定时器功能，什么时候启动PHP进程合适？是使用长驻的PHP进程还是使用定时任务每分种查询一次队列？  
二、启动多少PHP进程合适？  
三、如果一条消息因PHP进程意外退出导致没有发送成功，如何回滚？  

其实，如果没有严格时效要求，我们可以这样。可以想像，最坏的情况是消息延迟2分钟（即从数据表里查出来最多延迟1分钟，然后再从Redis队列拿出来最多延迟1分钟）发送，如此的话使用定时任务每分种启动PHP进程来查询Redis 队列即可。当然，这并不是最好的方式。

带着上面的问题，我们再来看看需求方更细化的需求：  
- 要求可以在添加任务时任意指定延迟时间触发任务。比如精确到 30 秒种以后，或者几分钟、几小时、几天以后；  
- 这种任务会出现比较多，有些消息重要且时效性要求高。  

**这个时候，单单使用 Crontab 定时任务已经无法满足需求，需要寻找更好的解决方案。既然想到了消息队列，那么，我们是否可以从这方面切入，找到一个可以实现定时器功能的消息队列，取代 Crontab 这种无法精确到秒的定时任务机制？**



### 常见的两种消息队列

其实说到消息队列，可能大家都会想到比较常见的 Rabbitmq、Redis。好，来看看是否是我们想要的。  
- Rabbitmq，原生不支持消息延迟，需要通过其它方式模拟。比如，使用 Time To Live (TTL)  + Dead Letter Exchanges(DLX)。即进入这种队列的消息在一定时间内超时会进入 exchange，然后再使用定时器，定时从 exchange 捞出来。 也可以使用插件 [rabbitmq-delayed-message-exchange](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange) 来实现。遗憾的是，我们最需要的是定时器，因为PHP很难去实现一个定时触发器。

- Redis，原生不支持延迟消息队列，可以通过设置过期时间，定时去队列里捞过期的消息，但是存在过期消息被回收的风险。

### 更好的解决方案

以上两种中间件都没有集成我们最需要的定时器，而PHP这方面确实比较弱，没有办法去实现一个友好的定时器。那业界有没有其它的解决办法呢？  
有的，那就是 Beanstalkd，轻量级消息中间件，原生支持延迟消息队列，延迟时间精确到秒，绝对是PHP实现延迟消息队列的最佳CP。  

Beanstalkd，一个高性能、轻量级的分布式内存队列系统，最初设计的目的是想通过后台异步执行耗时的任务来降低高容量Web应用系统的页面访问延迟，支持过有9.5 million用户的Facebook Causes应用。其内部实现采用 libevent，服务器-客户端之间类似 memcached 轻量级 tcp 通讯协议，因此有很高的性能，这里有个外国人做的测试对比：

![测试对比](~@img/php_delay_queue_1.jpg)

Beanstalkd 利用任务（job） 代替消息（message） 的概念，每一个任务都有以下几种状态：  
- READY：需要立即处理的任务，当延时 (DELAYED) 任务到期后会自动成为当前任务；  
- DELAYED： 延迟执行的任务, 当消费者处理任务后， 可以用将消息再次放回 DELAYED 队列延迟执行；  
- RESERVED：已经被消费者获取, 正在执行的任务。Beanstalkd 负责检查任务是否在 TTR(time-to-run) 内完成；  
- BURIED：保留的任务: 任务不会被执行，也不会消失，除非有人把它 “踢” 回队列；  
- DELETED：消息被彻底删除。  


从生产者 - 消费者的角度去看状态流转：  

![状态流转](~@img/php_delay_queue_2.jpg)

从开发者开发的角度去看状态流转：  

![状态流转](~@img/php_delay_queue_3.jpg)

Beanstalkd 最大特点是基于 管道（tube）和 任务 （job）的工作队列（work-queue），支持以下特性：  

### 任务优先级 (priority):  
任务 (job) 可以有 0~2^32 个优先级，0 代表最高优先级。 beanstalkd 采用最大最小堆 (Min-max heap) 处理任务优先级排序， 任何时刻调用 reserve 命令的消费者总是能拿到当前优先级最高的任务， 时间复杂度为 O(logn)。
 
### 延时任务 (delay):  

有两种方式可以延时执行任务 (job)：  
- 生产者发布任务时指定延时；  
- 当任务处理完毕后, 消费者再次将任务放入队列延时执行 (`RELEASE with <delay>`)。这种机制可以实现分布式的 `java.util.Timer`，这种分布式定时任务的优势是：如果某个消费者节点故障，任务超时重发 (time-to-run) 能够保证任务转移到另外的节点执行。
 
### 任务超时重发 (time-to-run):  

Beanstalkd 把任务返回给消费者以后：消费者必须在预设的 TTR (time-to-run) 时间内发送 delete / release/ bury 改变任务状态，否则 Beanstalkd 会认为消息处理失败，然后把任务交给另外的消费者节点执行。如果消费者预计在 TTR (time-to-run) 时间内无法完成任务，也可以发送 touch 命令，它的作用是让 Beanstalkd 从系统时间重新计算 TTR (time-to-run)。
 
### 任务预留 (buried):  
如果任务因为某些原因无法执行，消费者可以把任务置为 buried 状态让 Beanstalkd 保留这些任务。管理员可以通过 peek buried 命令查询被保留的任务，并且进行人工干预。简单的, `kick <n>` 能够一次性把 n 条被保留的任务踢回队列。


下面来看看如何与 PHP 结合使用，解决前面提到的问题。  

这里推荐个简洁的 PHP 客户端库：[davidpersson/beanstalk](https://github.com/davidpersson/beanstalk)  
我们需要一个生产者和一个消费者，把消息扔进消息队列即为生产者，取出消息来处理即为消费者。  

一个简易的生产者：

``` php
public function producer()
{
    $this->beanstalkd->useTube('default');
    $n = 1;
    while ($n) {
        $delay = mt_rand(0, 30);
        $this->beanstalkd->put(
            2, // priority.
            $delay,  //  delay. 秒数
            3, // run time
            "beanstalkd $n delay $delay" // The job's body.
        );
        $n --;
    }
}

```


一个简易的消费者：

``` php
public function consumer()
{
    $this->beanstalkd->watch('default');
    $limit = 10;
    echo 'start consumer' .chr(10);
    while ($limit) {
            // reserve 会阻塞进程，适当设置超时时间，比如 5 秒超时后进入下一次等待
            $job = $this->beanstalkd->reserve(5); 
            var_dump($job);
            if ($job) {
                //$jobStats = $this->beanstalkd->statsJob($job['id']);
                $this->beanstalkd->delete($job['id']);
                sleep(5);
                // if ($jobStats['reserves'] > 8) {
                //     $this->beanstalkd->bury($jobStats['id'], $jobStats['pri']);
                // }
                cilog($job);
                echo chr(10) . $limit . chr(10);
                $limit --;
            }
    }
    echo 'end' .chr(10);
}
```


从代码中可以看到，其实消费者进程是一个阻塞进程，使用一个循环去监听行等待 beanstalkd 返回消息，拿到消息后再进程处理。  

### 那为什么是阻塞进程呢？  

这是因为连接 Beanstalkd 服务端的客户端是用 `fsockopen/pfsockopen` 去连接通信的，默认情况下采用阻塞模式开启套接字连接，发送请求指令后将阻塞程序以等待响应。另外一个原因，这也是业务的需要，我们总是希望有一个进程去监听服务端给我们返回的消息，以便拿到消息后进行处理，然后进入下一次等待，而不是执行一次就退出，或者说在服务端没有返回消息时我们的消息处理程序还在不断的循环执行，浪费资源。  

在实际应用中，可能会产生各种类型的消息，消费者也会存在多个进程。所以我们还要考虑更为复杂的情况，比如：
#### 1、一个消息执行超时了我们应该如何处理（包括消息发送失败或PHP进程意外退出的情况）？  
由于Beanstalk的运行机制，一个job，即一条消息取出后如果不手动删掉或置为其它状态，则该消息将重回消息队列，由其它消费者程序处理。所以，为了避免一条消息重复处理，取出一条消息后，需要判断是否已经处理过，以及处理完一条消息后应该删除或置为其它状态。  

Beanstalkd 的每个 job 都有记录被消费者读取的次数，以及超时的次数，更多信息如下图：  

![job 包含的字段](~@img/php_delay_queue_4.jpg)

#### 2、一个消费者进程每次启动后执行多少条消息合适？或者说一个消费者进程持续运行多长时间比较合适？  
这里主要是为了PHP进程能在执行一段时间后自动退出，因为PHP不适合做一个常驻进程，PHP的设计目的也并非是后台服务，所以，更好的办法是在跑了一段时间后自动退出，新起一个进程。我们可以通过设置执行消息的次数以及持续运行的时间来让进程自动退出。你可能会说因为阻塞所以根本没法实现让程序在执行过程中判断次数和运行时间。放心，Beanstalkd 可以在监听服务端的时候设置超时间，即使用 reserve with timeout 来预订 job，设定后，在监听超时后将会进入下一次循环。  

另外，Beanstalkd 也可以开启binlog，如果遇到 Benstalkd 进程因为某些原因挂了，或者机器需要重启时，Beanstalkd 都能轻松地从 binlog 恢复这些消息。然而，总有一些消息是比较重要的，我们需要详细记录这个消息的发送情况，这就需要我们把消息落地，记录到数据库中，下图是一个记录消息状态的架构图。  

![消息状态架构图](~@img/php_delay_queue_5.jpg)

最后，我们需要根据业务量增加或减少消息处理进程。为了更好地管理这些处理进程，推荐使用 supervisor 进程管理器，可以轻松地解决下面几点：  
- 把不同类型的消息处理进程分组
- 方便的设定启动进程的数量
- 自动维护每个进程

### 附两个术语的解释：

::: tip TTR是如何工作的？
TTR仅用于一个job变成reserved状态的时候。在那个时候，一个计时器（在job状态中称为“time-left”）开始从job的TTR开始倒计时。
如果计时器变成0，这个job被放回ready队列。
计时器到期之前，如果这个job变成"buried"、“deleted”或"released"状态，计时器将停止并退出。
计时器变成0之前，如果这个工作变成"touched"，计时器从TTR重新开始倒计时。
（非"reserved"状态的job仍然包含一个"time-left"条目，但它的值是无意义的。）
:::

::: tip DEADLINE_SOON是什么意思?
DEADLINE_SOON是一个reserve命令的响应，它表明一个“reserved”状态的job的最后期限（deadline）马上要到期（目前的安全边际大约是1秒）。
如果你执行reserve命令时，频繁地收到DEADLINE_SOON错误，你可能应该考虑对你的job增加TTR，因为它表示你没有安时完成你的job。这也可能是在完成了你的job，却没有删除它们。
:::

参考资料：  
[Beanstalkd中文协议](https://github.com/kr/beanstalkd/blob/master/doc/protocol.zh-CN.md)  
[Beanstalkd 介绍](http://in355hz.iteye.com/blog/1395727)   
[rabbitmq 实现延迟队列的两种方式](http://blog.csdn.net/u014308482/article/details/53036770)  