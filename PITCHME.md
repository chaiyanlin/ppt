# swoole环境部署

## 环境要求

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

# HelloWorld

`swoole`是一个PHP 异步网络通信引擎，其核心就是PHP扩展`swoole.so`文件。

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
