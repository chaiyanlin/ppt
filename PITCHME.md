## 环境要求

`swoole`是一个PHP 异步网络通信引擎，其核心就是PHP扩展`swoole.so`文件。

- Swoole-2.x
- PHP-7.1.x
- 依赖hiredis库（如果需要async-redis特性）

---

## 编译安装

```
wget https://github.com/swoole/swoole-src/archive/v2.2.0.zip
unzip v2.2.0.zip
cd swoole-src-2.2.0
phpize
./configure --enable-openssl --enable-async-redis --enable-timewheel --enable-mysqlnd --enable-coroutine
```

> `php.ini`添加`extension=swoole.so`

---


## 原生HTTPServer

```php
<?php
$http = new swoole_http_server("10.222.147.99", 9501);
$http->set([
    'worker_num' => 100,
]);
$ch = curl_init();
curl_setopt($ch,CURLOPT_URL,"http://apps.gzh.qq.com/");
curl_setopt($ch,CURLOPT_RETURNTRANSFER,1);
curl_setopt($ch,CURLOPT_HEADER,0);
$http->on('request', function ($request, $response) use ($ch) {
    //$response->end("<h1>Hello Swoole. #" . rand(1000, 9999) . "</h1>");
    $output = curl_exec($ch);
    //var_dump($output);
});
$http->start();
```

---

## EasySwoole

EasySwoole 是一款基于Swoole Server 开发的常驻内存型的分布式PHP框架，专为API而生，摆脱传统PHP运行模式在进程唤起和文件加载上带来的性能损失。EasySwoole 高度封装了 Swoole Server 而依旧维持 Swoole Server 原有特性，支持同时混合监听HTTP、自定义TCP、UDP协议，让开发者以最低的学习成本和精力编写出多进程，可异步，高可用的应用服务

---

## 优势

- 简单易用开发效率高
- 并发百万TCP连接
- TCP/UDP/UnixSock
- 支持异步/同步/协程
- 支持多进程/多线程
- CPU亲和性/守护进程

---

## 常用功能与组件

- HTTP控制器与自定义路由
- TCP、UDP、WEB_SOCKET控制器
- 多种混合协议通讯
- 异步客户端与协程对象池
- 异步进程、自定义进程、定时器
- 集群分布式支持，例如集群节点通讯，服务发现，RPC
- 全开放系统事件注册器与EventHook

---

## 框架安装

```
# 创建项目
composer create-project easyswoole/app easyswoole

# 进入项目目录并启动
cd easyswoole
php easyswoole start
```
---

## 目录结构

```
project                   项目部署目录
├─App                     应用目录(可以有多个)
│  ├─HttpController       控制器目录
│  │  └─Index.php         默认控制器
│  └─Model                模型文件目录
├─Log                     日志文件目录
├─Temp                    临时文件目录
├─vendor                  第三方类库目录
├─composer.json           Composer架构
├─composer.lock           Composer锁定
├─Config.php              框架全局配置
├─EasySwooleEvent.php     框架全局事件
├─easyswoole              框架管理脚本
├─easyswoole.install      框架安装锁定文件
```

---

## Config.php

```php
return [
    'SERVER_NAME' => "EasySwoole",
    'MAIN_SERVER' => [
        'HOST' => '0.0.0.0',
        'PORT' => 9501,
        'SERVER_TYPE' => \EasySwoole\Core\Swoole\ServerManager::TYPE_WEB_SERVER,
        'SOCK_TYPE' => SWOOLE_TCP, //该配置项当为SERVER_TYPE值为TYPE_SERVER时有效
        'RUN_MODEL' => SWOOLE_PROCESS,
        'SETTING' => [
            'task_worker_num' => 8, //异步任务进程
            'task_max_request' => 100,
            'max_request' => 5000, //强烈建议设置此配置项
            'worker_num' => 100,
        ],
    ],
    'DEBUG' => true,
    'TEMP_DIR' => null, //若不配置，则默认框架初始化
    'LOG_DIR' => null, //若不配置，则默认框架初始化
    'EASY_CACHE' => [
        'PROCESS_NUM' => 1, //若不希望开启，则设置为0
        'PERSISTENT_TIME' => 0, //如果需要定时数据落地，请设置对应的时间周期，单位为秒
    ]
```
[详细说明](https://www.easyswoole.com/Manual/2.x/Cn/_book/Introduction/config.html)

---

## 服务管理脚本

```
使用:
  easyswoole [操作] [选项]

操作:
  install       安装easySwoole
  start         启动easySwoole
  stop          停止easySwoole
  reload        重启easySwoole
  help          查看命令的帮助信息
```

---

## Nginx反向代理配置

访问：http://test.gzh.qq.com/moss-gateway/wechat/callback

```
location /moss-gateway/ {
        rewrite /moss-gateway/(.*) /$1 break;
        proxy_http_version 1.1;
        proxy_set_header Connection "keep-alive";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://127.0.0.1:9501;
    }
```

---

## 连接池

连接池不是跨进程的，也就是说一个进程有一个连接池，配置中的MAX为100，开了4个worker，最大连接数可能达到400。

---

## 数据库协程连接池

```
PoolManager::getInstance()->registerPool(MysqlPool2::class, 3, 10);
$pool = PoolManager::getInstance()->getPool('App\Utility\MysqlPool');// 获取连接池对象
$db = $pool->getObj();
```

---

## 连接池基本方法

**getObj 从连接池中取得对象：**

```
public function getObj($timeOut = 0.1) {}
```

**freeObj 释放对象：**
```
public function freeObj($obj) {}
```

---

## 协程curl

```
function concurrent()
    {
        //以下流程网络IO的时间就接近于 MAX(q1网络IO时间, q2网络IO时间)。
        $micro = microtime(true);
        $q1 = new Http('http://127.0.0.1:9501/curl/sleep/index.html?time=1');
        $c1 = $q1->exec(true);

        $q2 = new Http('http://127.0.0.1:9501/curl/sleep/index.html?time=4');
        $c2 = $q2->exec(true);

        $c1->recv();
        $c1->close();
        $c2->recv();
        $c2->close();

        $time = round(microtime(true) - $micro,3);
        $this->response()->write($time);

    }
```

---

## 异步worker

> 直接投递闭包

```
function index()
{
    \EasySwoole\Core\Swoole\Task\TaskManager::async(function () {
        echo "执行异步任务...\n";
        return true;
    }, function () {
        echo "异步任务执行完毕...\n";
    });
}
```

--- 

> 投递任务模板类

```
// 在控制器中投递的例子
function index()
{
    // 实例化任务模板类 并将数据带进去 可以在任务类$taskData参数拿到数据
      $taskClass = new Task('taskData');
    \EasySwoole\Core\Swoole\Task\TaskManager::async($taskClass);
}
```

`Task`类需要继承`AbstractAsyncTask`；

---

## 定时器

```
$register->add($register::onWorkerStart, function (\swoole_server $server, $workerId) {
            //为第一个进程添加定时器
            if ($workerId == 0) {
                # 启动定时器
                Timer::loop(10000, function () {
                    Logger::getInstance()->console('timer run');  # 写日志到控制台
                    ProcessManager::getInstance()->writeByProcessName('test', time());  # 向自定义进程发消息
                });
            }
        });
```
