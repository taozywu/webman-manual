# 生命周期

## 进程生命周期
- 每个进程都有很长的生命周期
- 每个进程在其生命周期内可以处理多个请求
- 进程在收到`stop` `reload` `restart`命令时会执行退出，结束本次生命周期

## 请求生命周期
- 每个请求会产生一个`$request`对象
- `$request`对象在请求处理完毕后会被回收

## 控制器生命周期
- 控制器每个进程只会实例化一次
- 控制器实例会被多个请求共享
- 如果控制器在路有由定义，则在进程启动时被实例化
- 如果控制器在路由里没有定义，则在需要时被实例化
- 控制器生命周期在进程退出后结束

## 关于变量生命周期
webman是基于php开发的，所以它完全遵循php的变量回收机制。业务逻辑里产生的临时变量包括new关键字创建的类的实例，在函数或者方法结束后自动回收，无需手动`unset`释放。也就是说webman开发与传统框架开发体验基本一致。例如下面例子中`$foo`实例会随着index方法执行完毕而自动释放：
```php
<?php

namespace app\controller;

use app\service\Foo;
use support\Request;

class Index
{
    public function index(Request $request)
    {
        $foo = new Foo(); // 这里假设有一个Foo类
        return response($foo->sayHello());
    }
}
```
如果你想某个类的实例被复用，则可以将类保存到类的静态属性中或长生命周期对象(如控制器)的属性中，也可以使用Container容器的get方法来初始化类的实例，例如：
```php
<?php

namespace app\controller;

use app\service\Foo;
use support\Container;
use support\Request;

class Index
{
    public function index(Request $request)
    {
        $foo = Container::get(Foo::class);
        return response($foo->sayHello());
    }
}
```

`Container::get()`方法用于创建并保存类的实例，下次再次以同样的参数再次调用时将返回之前创建的类实例。

> **注意**
> `Container::get()`只能初始化没有构造参数的实例。`Container::make()`可以创建带构造函数参数的实例，但是与`Container::get()`不同的是，`Container::make()`并不会复用实例，也就是说即使以同样的参数`Container::make()`始终返回一个新的实例。

# 关于内存泄漏
绝大部分情况下，我们的业务代码并不会发生内存泄漏(极少有用户反馈发生内存泄漏)，我们只要稍微注意下长生命周期的数组数据不要无限扩张即可。请看以下代码：
```php
<?php
namespace app\controller;

use support\Request;

class Foo
{
    // 数组属性
    public $data = [];
    
    public function index(Request $request)
    {
        $this->data[] = time();
        return response('hello index');
    }

    public function hello(Request $request)
    {
        return response('hello webman');
    }
}
```
控制器是长生命周期的，同样的控制器的`$data`数组属性也是长周期的，随着`foo/index`请求不断增加，`$data`数组元素越来越多导致内存泄漏。

> **提示**
> webman的monitor进程会监控webman内存占用，如果某个进程占用内存即将超过php.ini中`memory_limit`设定的值，webman会安全重启这个进程，达到释放内存的作用。所以即使业务代码出现内存泄漏，也不会对业务造成影响。

> **提示**
> 随着请求增加，webman会载入一些类文件到内存，内存会有一定增长，这种内存增长是正常现象，只要不是内存无限增长就不是内存泄漏。