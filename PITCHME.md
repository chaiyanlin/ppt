# 项目规范

---

## Laravel 版本选择

**使用Laravel/Lumen5.5 LTS 版本**

```
composer create-project laravel/laravel project-name --prefer-dist "5.5.*"

composer create-project laravel/lumen project-name --prefer-dist "5.5.*"
```

---

## 开发和线上环境

一个项目至少应该有以下三个基本的项目环境：

- `local` 开发环境
- `test` 测试环境
- `prod` 正式生产环境

因此项目根目录下的`.env`环境变量配置需要有三份：

- `.env` 开发者本地开发，应加入`.gitingore`不提交至版本库
- `.env.test` 测试环境
- `.env.prod` 正式生产环境

---

## 配置信息与环境变量

针对不同环境的公共配置信息，如Mysql，Redis，CDN等，应**存储在 .env 和 config/app.php 文件中，然后使用 config() 函数来读取。**

---

e.g.

**.env文件存储Mysql用户信息**

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=ieg_briefs
DB_USERNAME=root
DB_PASSWORD=
```

---

**config/database.php文件配置Mysql信息**


```php
<?php

return [
    'connections' => [
        ...
        'mysql' => [
            'driver'    => 'mysql',
            'host'      => env('DB_HOST', 'localhost'), // 从环境变量中提取信息
            'port'      => env('DB_PORT', 3306), // 从环境变量中提取信息
            'database'  => env('DB_DATABASE', 'forge'), // 从环境变量中提取信息
            'username'  => env('DB_USERNAME', 'forge'), // 从环境变量中提取信息
            'password'  => env('DB_PASSWORD', ''), // 从环境变量中提取信息
            'charset'   => env('DB_CHARSET', 'utf8mb4'),
            'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
            'prefix'    => env('DB_PREFIX', ''),
            'timezone'  => env('DB_TIMEZONE', '+00:00'),
            'strict'    => env('DB_STRICT_MODE', false),
        ],
];

```

---

## 辅助函数

必须 把所有的 **自定义辅助函数** 存放于 bootstrap 文件夹中。

并在 `bootstrap/app.php` 文件的最顶部进行加载：

```php
<?php

require __DIR__ . '/helpers.php';

...
```

---

## 缓存策略规范

---

**缓存驱动的选择需要与业务团队事先沟通清楚，究竟是使用Memcached，Redis，或者其他。**

缓存驱动默认使用**Redis**，不要使用**Memcached**。

缓存key **必须** 配置项目前缀作为区分：

```php
'redis' => [

        'client' => 'predis',

        'options' => [
            'prefix' => env('NAMESAPCE', '').'_moss_laravel:', // 项目前缀
        ],

        'default' => [
            'host' => env('REDIS_HOST', '127.0.0.1'),
            'password' => env('REDIS_PASSWORD', null),
            'port' => env('REDIS_PORT', 6379),
            'database' => 0,
        ],
```

---

## 项目文档编写规范

---

每一个项目都 **必须** 包含一个 `readme.md` 文件，readme 里书写这个项目的简单信息。作用主要有两个，一个是团队新成员可从此文件中快速获悉项目大致情况，另一个是部署项目时可以作为参考。

---


`readme.md` 文档 **应该** 包含以下内容：

-「项目概述」- 介绍说明项目的一些情况，类似于简单的产品说明，简单的功能描述，项目相关链接等，500 字以内；

-「运行环境」- 运行环境说明，系统要求等信息；

-「开发环境部署/安装」- 一步一步引导说明，保证项目新成员能最快速的，没有歧义的部署好开发环境；

-「服务器架构说明」- 最好能有服务器架构图，从用户浏览器请求开始，包括后端缓存服务使用等都描述清楚（主要体现为软件的使用），配合「运行环境」区块内容，可作为线上环境部署的依据；

-「代码上线」- 介绍代码上线流程，需要执行哪些步骤；

-「扩展包说明」- 表格列出所有使用的扩展包，还有在哪些业务逻辑或者用例中使用了此扩展包；

-「自定义 Artisan 命令列表」- 以表格形式罗列出所有自定义的命令，说明用途，指出调用场景；

-「队列列表」- 以表格形式罗列出项目所有队列接口，说明用途，指出调用场景。

---



# 编码规范

---

## 代码风格

代码风格 **必须** 严格遵循 [PSR-2](https://www.kancloud.cn/thinkphp/php-fig-psr/3141) 规范。

---

## 路由器

---

### 路由闭包

---

**绝不** 在路由配置文件里书写『闭包路由』或者其他业务逻辑代码，因为一旦使用将无法使用 [路由缓存](https://laravel-china.org/docs/laravel/5.5/controllers/1296#route-caching) 。

路由器要保持干净整洁，**绝不** 放置除路由配置以外的其他程序逻辑。

---

### Restful 路由

**必须** 优先使用 Restful 路由，配合资源控制器使用，见 [文档](https://laravel-china.org/docs/laravel/5.5/controllers/1296#RESTful-%E8%B5%84%E6%BA%90%E6%8E%A7%E5%88%B6%E5%99%A8)。

**由资源控制器处理的行为**

动词 | 路径 | 行为（方法）| 路由名称
--|--|--|--|
GET | /photos | index | photos.index
GET | /photos/create | create | photos.create
POST | /photos | store | photos.store
GET | /photos/{id} | show | photos.show
PUT/PATCH | /photos/{id} | update | photos.update
DELETE | /photos/{id} | destory | photos.destory

**超出 Restful 路由的，应该 模仿上表的方式来定义路由。**

---

### resource 方法正确使用

一般资源路由定义：

```php
Route::resource('photos', 'PhotosController');
```

---

等于以下路由定义：

```php
Route::get('/photos', 'PhotosController@index')->name('photos.index');
Route::get('/photos/create', 'PhotosController@create')->name('photos.create');
Route::post('/photos', 'PhotosController@store')->name('photos.store');
Route::get('/photos/{photo}', 'PhotosController@show')->name('photos.show');
Route::put('/photos/{photo}', 'PhotosController@update')->name('photos.update');
Route::delete('/photos/{photo}', 'PhotosController@destroy')->name('photos.destroy');
```

---

使用 `resource` 方法时，如果仅使用到部分路由，必须 使用 `only` 列出所有可用路由：

```php
Route::resource('photos', 'PhotosController', ['only' => ['index', 'show']]);
```

**绝不** 使用 `except`，因为 `only` 相当于白名单，相对于 `except` 更加直观。路由使用白名单有利于养成『安全习惯』。

---

### 单数 or 复数？

资源路由路由 URI 必须 使用复数形式，如：

```
/photos/create
/photos/{photo}
```

错误的例子如：

```
/photo/create
/photo/{photo}
```

---

## 数据模型

---

### 放置位置

所有的数据模型文件，都 必须 存放在：`app/Models/` 文件夹中。

命名空间：

```
namespace App\Models;
```
---

### User.php

Laravel 5.1 默认安装会把 `User` 模型存放在 `app/User.php`，必须 移动到 `app/Models` 文件夹中，并修改命名空间声明为 `App/Models`，同上。

为了不破坏原有的逻辑点，必须 全局搜索 `App/User` 并替换为 `App/Models/User`。

---


### 使用基类

所有的 **Eloquent 数据模型** 都 必须 继承统一的基类 `App/Models/Model`，此基类存放位置为 `/app/Models/Model.php`，内容参考以下：

---

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model as EloquentModel;

class Model extends EloquentModel
{
    public function scopeRecent($query)
    {
        return $query->orderBy('created_at', 'desc');
    }
}
```

---

以 Photo 数据模型作为例子继承 Model 基类：

```php
<?php

namespace App\Models;

class Photo extends Model
{
    protected $fillable = ['id', 'user_id'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

---

### 命名规范

数据模型相关的命名规范：

- 数据模型类名 **必须** 为「单数」, 如：App\Models\Photo
- 类文件名 **必须** 为「单数」，如：app/Models/Photo.php
- 数据库表名字 **必须** 为「复数」，多个单词情况下使用[Snake Case](https://en.wikipedia.org/wiki/Snake_case) 如：photos, my_photos
- 数据库表迁移名字 **必须** 为「复数」，如：2014_08_08_234417_create_photos_table.php
- 数据填充文件名 **必须** 为「复数」，如：PhotosTableSeeder.php
- 数据库字段名 **必须** 为[Snake Case](https://en.wikipedia.org/wiki/Snake_case)，如：view_count, is_vip
- 数据库表主键 **必须** 为「id」
- 数据库表外键 **必须** 为「resource_id」，如：user_id, post_id
- 数据模型变量 **必须** 为「resource_id」，如：$user_id, $post_id

---

## 控制器

### 资源控制器

**必须** 优先使用 Restful [资源控制器](http://d.laravel-china.org/docs/5.5/controllers#restful-resource-controllers) 。

### 单数 or 复数？

**必须** 使用资源的复数形式，如：
```
类名：PhotosController
文件名：PhotosController.php
```

错误的例子：
```
类名：PhotoController
文件名：PhotoController.php
```

### 保持短小精炼

**必须** 保持控制器文件代码行数最小化，还有可读性。

- **不应该** 为「方法」书写注释，这要求方法取名要足够合理，不需要过多注释；
- **应该** 为一些复杂的逻辑代码块书写注释，主要介绍产品逻辑 - 为什么要这么做。；
- **不应该** 在控制器中书写「私有方法」，控制器里 应该 只存放「路由动作方法」；
- **绝不** 遗留「死方法」，就是没有用到的方法，控制器里的所有方法，都应该被使用到，否则应该删除；
- **绝不** 在控制器里批量注释掉代码，无用的逻辑代码就必须清除掉。

## 表单验证

### 表单请求验证类

**应该** 使用 表单请求 - [FormRequest 类](https://laravel-china.org/docs/laravel/5.5/validation/1302#form-request-validation) 来处理控制器里的表单验证。

所有 FormRequest 表验证类 必须 继承 `app/Http/Requests/Request.php` 基类。基类文件如下：

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class Request extends FormRequest
{
    public function authorize()
    {
        // Using policy for Authorization
        return true;
    }
}
```

### 验证类命名

FormRequest 表验证类 必须 遵循 资源路由 方式进行命名，`photos` 对应 `app/Http/Requests/PhotoRequest.php` 。

### 类文件参考

FormRequest 表验证类文件请参考以下：

```php
<?php

namespace App\Http\Requests;

class PhotoRequest extends Request
{
    public function rules()
    {
        switch($this->method())
        {
            // CREATE
            case 'POST':
            {
                return [
                    // CREATE ROLES
                ];
            }
            // UPDATE
            case 'PUT':
            case 'PATCH':
            {
                return [
                    // UPDATE ROLES
                ];
            }
            case 'GET':
            case 'DELETE':
            default:
            {
                return [];
            };
        }
    }

    public function messages()
    {
        return [
            // Validation messages
        ];
    }
}
```


## Artisan 命令行

所有的自定义命令，都 **必须** 有项目的命名空间。

如：

```
php artisan wechat:set-menu
php artisan wechat:send-template-message
```

错误的例子为：

```
php artisan clear-token
php artisan send-status-email
```

## 中间件

**鉴权** 类型的事务 **必须** 以 middleware中间件的形式实现。

Auth 中间件 **必须** 书写在控制器的 __construct 方法中，并且 **必须** 使用 except 黑名单进行过滤，这样当你新增控制器方法时，默认是安全的。

```php
public function __construct()
{
    $this->middleware('auth', [            
        'except' => ['show', 'index']
    ]);
}
```

# 安全

## 关闭 DEBUG

Laravel Debug 开启时，会暴露很多能被黑客利用的服务器信息，所以，生产环境下请 必须 确保：

```
APP_DEBUG=false
```

## XSS

跨站脚本攻击（cross-site scripting，简称 XSS），具体危害体现在黑客能控制你网站页面，包括使用 JS 盗取 Cookie 等，关于 [XSS 的介绍请前往 IBM 文档库：跨站点脚本攻击深入解析](https://www.ibm.com/developerworks/cn/rational/08/0325_segal/) 。

默认情况下，在无法保证用户提交内容是 100% 安全的情况下，必须 使用 Blade 模板引擎的 {{ $content }} 语法会对用户内容进行转义。

Blade 的 {!! $content !!} 语法会直接对内容进行 非转义 输出，使用此语法时，必须 使用 [HTMLPurifier for Laravel 5](https://github.com/mewebstudio/Purifier) 来为用户输入内容进行过滤。使用方法参见： [使用 HTMLPurifier 来解决 Laravel 5 中的 XSS 跨站脚本攻击安全问题](https://laravel-china.org/articles/4798/the-use-of-htmlpurifier-to-solve-the-xss-xss-attacks-of-security-problems-in-laravel)

## SQL 注入

__除非特殊情况，否则不要使用raw()来编写sql语句。__

Laravel 的 [查询构造器](http://d.laravel-china.org/docs/5.3/queries) 和 [Eloquent](https://laravel-china.org/docs/laravel/5.3/eloquent/1192) 是基于 PHP 的 PDO，PDO 使用 prepared 来准备查询语句，保障了安全性。

在使用 raw() 来编写复杂查询语句时，必须 使用数据绑定。


错误的做法：

```php
Route::get('sql-injection', function() {
    $name = "admin"; // 假设用户提交
    $password = "xx' OR 1='1"; // // 假设用户提交
    $result = DB::select(DB::raw("SELECT * FROM users WHERE name ='$name' and password = '$password'"));
    dd($result);
});
```

以下是正确的做法，利用 select 方法 的第二个参数做数据绑定：
```php
Route::get('sql-injection', function() {
    $name = "admin"; // 假设用户提交
    $password = "xx' OR 1='1"; // // 假设用户提交
    $result = DB::select(
        DB::raw("SELECT * FROM users WHERE name =:name and password = :password"),
        [
            'name' => $name,
            'password' => $password,
        ]
    );
    dd($result);
});
```

**再次强调，Eloquent 能够覆盖“绝大多数”的业务场景，我们应该严格遵守官方提供接口使用规范**


## CSRF

CSRF 跨站请求伪造是 Web 应用中最常见的安全威胁之一，具体请见 [Wiki - 跨站请求伪造](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0) 或者 Web [应用程序常见漏洞 CSRF 的入侵检测与防范](https://www.ibm.com/developerworks/cn/rational/r-cn-webcsrf/)。

Laravel 默认对所有『非幂等的请求』强制使用 VerifyCsrfToken 中间件防护，需要开发者做的，是区分清楚什么时候该使用『非幂等的请求』。

> 幂等请求指的是：'HEAD', 'GET', 'OPTIONS'，既无论你执行多少次重复的操作都不会给资源造成变更。

- 所有删除的动作，必须 使用 DELETE 作为请求方法；
- 所有对数据更新的动作，必须 使用 POST、PUT 或者 PATCH 请求方法。
