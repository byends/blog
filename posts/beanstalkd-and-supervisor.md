---
type: post
category: php
date: 2018-06-14
title: Beanstalkd 搭配 Supervisor 的安装与使用
meta:
  - name: description
    content: Beanstalkd 没有提供主备同步以及故障切换机制，在应用中有成为单点的风险
---

前面有说到延迟消息队列的选择，可移步查看：[谁才是PHP实现延迟消息队列的最佳CP？](./php-delay-queue.md)

::: tip
Beanstalkd 没有提供主备同步以及故障切换机制，在应用中有成为单点的风险。实际应用中，可以用数据库为任务 (job) 提供持久化存储。另外，和 memcached 类似，Beanstalkd 依赖 libevent 的单线程事件分发机制，不能有效利用多核 cpu 的性能，这一点可以通过单机部署多个实例克服。  
:::


## 安装 

``` sh
git clone https://github.com/kr/beanstalkd.git
cd beanstalkd
make
mv beanstalkd /usr/local/bin/
```

## 运行

``` bash
beanstalkd -l 127.0.0.1 -p 11300 port -c -b /data/logs/beanstalkd/
```
查看支持的命令  
![查看支持的命令](~@img/beanstalkd-and-supervisor_1.jpg)


### 简单的生产者：`producer.php`

推荐PHP beanstalkd 客户端：[davidpersson/beanstalk](https://github.com/davidpersson/beanstalk)  

```php
require_once 'src/Client.php'

use Beanstalk\Client;

$beanstalk = new Client();
$beanstalkd->useTube('default');

// 一次压入多条消息
$n = 10;
while ($n) {
    $delay = mt_rand(0, 30);
    $beanstalkd->put(
        2, // priority.
        $delay,  //  delay. 秒数
        3, // run time
        "beanstalkd $n delay $delay" // The job's body.
    );
    $n --;
}

echo 'done!';
```

可用 PHP 内置 Webserver 启动：
``` bash
php -S localhost:8080 producer.php
```

### 简易的消费者：`consumer.php`

``` php
$beanstalk = new Client();

$beanstalk->connect();
$beanstalk->watch('default');

// 监听一定数目后退出
$limit = 20;
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
```

``` bash
/usr/local/php7/bin/php consumer.php
```

运行后命令行会输出相应的结果。

### 日志

Beanstalkd 可以开启binlog，如果遇到 Benstalkd 进程因为某些原因挂了，或者机器需要重启时，Beanstalkd 都能轻松地从 binlog 恢复这些消息。
beanstalkd 的 binlog 位于 -b 选项指定的文件夹中，名字就叫 binlog.$order。binlog 的序号 $order 从 1 开始，逐一递增。
binlog 文件大小是固定的，可以通过 -s 选项指定，默认为 10M。


### 查看消息列队

Beanstalkd 和 memcached 一样，采用 TCP 通信，想要进入它的命令行模式，需要使用 telnet
``` bash
telnet 127.0.0.1 11300
```
常用的一些命令：
- stats
- list-tubes
- list-tube-used
- list-tubes-watched

切换到某个队列查看：
- use default
- stats-tube default
- stats-job 2

更多命令查看这里：[Beanstalkd中文协议](https://github.com/kr/beanstalkd/blob/master/doc/protocol.zh-CN.md)  
推荐一个网页版的管理控制台：[beanstalk_console](https://github.com/ptrofimov/beanstalk_console)


## 安装 supervisor

``` bash
yum -y install python-setuptools
easy_install supervisor

# 或 pip install supervisor

mkdir /etc/supervisor/
mkdir /etc/supervisor/conf.d
echo_supervisord_conf > /etc/supervisor/supervisord.conf
```


编辑配置文件:
``` bash
vi /etc/supervisor/supervisord.conf
```

在最后面添加 :
[include]
files = /etc/supervisor/conf.d/*.conf

然后添加 beanstalkd 服务:
``` bash
vi etc/supervisor/conf.d/beanstalkd.conf
```

输入:
[program:beanstalkd]
command=/usr/local/bin/beanstalkd -l 127.0.0.1 -p 11300 -c -b /data/logs/beanstalkd/

添加服务
``` bash
vi /etc/supervisor/conf.d/queue.conf
```

输入
[program:queue]
user=nobody
command=/usr/local/php7/bin/php /usr/local/project/htdocs/consumer.php
process_name=%(program_name)s_%(process_num)02d
numprocs=2
numprocs_start=1

启动服务:
``` bash
supervisord -c /etc/supervisor/supervisord.conf
```

supervisorctl 是 supervisor 管理工具，可启动、关闭、查看由 supervisor 维护的进程

它有两种运行方式：
- 默认使用 tcp 连接，即在配置中开启 inet_http_server 配置
- 使用 unix 域套接字，即开启 unix_http_server 配置

查看状态：
``` bash
supervisorctl -c /etc/supervisor/supervisord.conf status
```

查看/管理所有服务的状态:  

![查看支持的命令](~@img/beanstalkd-and-supervisor_2.jpg)


这里有 supervisor 现成的 *nux [服务脚本](https://github.com/Supervisor/initscripts)
centos 的机器可以选择 redhat-init-mingalevme，下载后修改里面的配置文件路径，改成：`/etc/supervisor/supervisord.conf`  


参考资料：  
[Supervisor Configuration File](http://supervisord.org/configuration.html)  