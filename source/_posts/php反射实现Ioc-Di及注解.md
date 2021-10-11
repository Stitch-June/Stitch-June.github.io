---
title: php反射实现Ioc-Di及注解
tags: []
id: '44'
categories:
  - - Php
date: 2020-06-08 23:47:49
---

PHP5之后提供了完整的反射API，添加了对类、接口、函数、方法和扩展进行反向工程的能力。此外，反射API提供了方法来取出函数、类和方法的文档注释。

**Ioc/Di**大家应该都不陌生，但是对小白来说呢听起来就挺高大上的，下面就用代码来实现：

```php
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */

class Foo
{
    public function getClassName()
    {
        return 'this is Foo';
    }
}

class Bar
{
    public function getClassName()
    {
        return 'this is Bar';
    }
}

class Test
{
    public function __construct(Foo $foo, Bar $bar)
    {
        var_dump($foo->getClassName());
        var_dump($bar->getClassName());
    }

    public function index(Foo $foo)
    {
        var_dump($foo->getClassName());
    }
}

// 反射Test
$reflect = new ReflectionClass(Test::class);
// 获取是否有构造函数
$constructor = $reflect->getConstructor();
if ($constructor) {
    // 如果存在构造 获取参数
    $constructorParams = $constructor->getParameters();
    // 初始化注入的参数
    $args = [];
    // 循环去判断参数
    foreach ($constructorParams as $param) {
        // 如果为class 就进行实例化
        if ($param->getClass()) {
            $args[] = $param->getClass()->newInstance();
        } else {
            $args[] = $param->getName();
        }
    }
    // 实例化注入参数
    $class = $reflect->newInstanceArgs($args);
} else {
    $class = $reflect->newInstance();
}


// 假设我们要调用index方法 在此之前自己判断下方法是否存在 我省略了
$reflectMethod = new ReflectionMethod($class, 'index');
// 判断方法修饰符是否为public
if ($reflectMethod->isPublic()) {
    // 以下代码同等上面
    $args = [];
    $methodParams = $reflectMethod->getParameters();
    foreach ($methodParams as $param) {
        if ($param->getClass()) {
            $args[] = $param->getClass()->newInstance();
        } else {
            $args[] = $param->getName();
        }
    }
    $reflectMethod->invokeArgs($class, $args);
}
```

> 以上就简单的实现了依赖注入，下面我们接着封装一下。

`Ioc.php`

```php
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */

class Ioc
{
    public static function getInstance($className)
    {
        $args = self::getMethodParams($className);
        return (new ReflectionClass($className))->newInstanceArgs($args);
    }

    public static function make($className, $methodName, $params = []) {

        // 获取类的实例
        $instance = self::getInstance($className);

        // 获取该方法所需要依赖注入的参数
        $args = self::getMethodParams($className, $methodName);

        return $instance->{$methodName}(...array_merge($args, $params));
    }

    protected static function getMethodParams($className, $methodsName = '__construct')
    {

        // 通过反射获得该类
        $class = new ReflectionClass($className);
        $args = []; // 记录注入的参数

        // 判断该类是否有构造函数
        if ($class->hasMethod($methodsName)) {
            // 构造函数存在 进行获取
            $construct = $class->getMethod($methodsName);

            // 获取构造函数的参数
            $params = $construct->getParameters();

            // 构造函数无参数 直接返回
            if (!$params) return $args;

            // 判断参数类型
            foreach ($params as $param) {

                // 假设参数为类
                if ($paramClass = $param->getClass()) {

                    // 获得参数类型名称
                    $paramClassName = $paramClass->getName();
                    // 如果注入的这个参数也是个类 就要继续判断是否存在构造函数
                    $methodArgs = self::getMethodParams($paramClassName);

                    // 存入数组中
                    $args[] = (new ReflectionClass($paramClass->getName()))->newInstanceArgs($methodArgs);
                }
            }
        }
        // 返回参数
        return $args;
    }
}
```

以上代码实现了构造函数的依赖注入及方法的依赖注入，下面进行测试。

```php
<?php
class Bar
{
    public function getClassName()
    {
        return 'this is Bar';
    }
}

class Test
{
    public function __construct(Foo $foo, Bar $bar)
    {
        var_dump($foo->getClassName());
        var_dump($bar->getClassName());
    }

    public function index(Foo $foo)
    {
        var_dump($foo->getClassName());
    }
}
Ioc::getInstance(Test::class);
Ioc::make(Test::class,'index');
```

以上呢，就简单的通过php的反射机制实现了依赖注入。



继基于`swoole`的微服务框架出现，注解呢就开始慢慢出现在我们的视角里。据说php8也加入了注解支持：

```php
use \Support\Attributes\ListensTo;

class ProductSubscriber
{
    <<ListensTo(ProductCreated::class)>>
    public function onProductCreated(ProductCreated $event) { /* … */ }

    <<ListensTo(ProductDeleted::class)>>
    public function onProductDeleted(ProductDeleted $event) { /* … */ }
}
```

就类似这样的，哈哈哈。

而我们现在的注解则是通过反射拿到注释去做到的解析。

接下来我们去用别人写好的组件去实现[annotations](https://github.com/doctrine/annotations)

编写我们的**composer.json**：

```json
{
    "require": {
        "doctrine/annotations": "^1.8"
    },
    "autoload": {
        "psr-4": {
            "app\\": "app/",
            "library\\": "library/"
        }
    }
}
```

接下来要执行啥？？？这个你要是再不会，真的我劝你回家种地吧！！哈哈哈 闹着玩呢！

`composer install`

然后我们接下来去创建目录：

```bash
- app // app目录
	- Http
		- HomeController.php
- library // 核心注解库
	- annotation
		- Mapping
			- Controller.php
			- RequestMapping.php
		- Parser
- vendor
- composer.json
- index.php // 测试文件
```

![php-annotation目录](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9xaW5pdS5nYW9iaW56aGFuLmNvbS8yMDIwLzA2LzA4LzU0MTFjNTlkYjZmZWIucG5n?x-oss-process=image/format,png)

害 图片有点大。。。。。咋整。。。算了，就这样吧！！！

温馨提示：在phpstrom里面，安装插件**PHP Annotation**写代码会更友好啊！！

创建`library\Mapping\Controller.php`

```php
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */


namespace library\annotation\Mapping;


use Doctrine\Common\Annotations\Annotation\Attribute;
use Doctrine\Common\Annotations\Annotation\Attributes;
use Doctrine\Common\Annotations\Annotation\Required;
use Doctrine\Common\Annotations\Annotation\Target;

/**
 * Class Controller
 * @package library\annotation\Mapping
 * @Attributes({
 *     @Attribute("prefix", type="string"),
 * })
 * @Annotation
 * @Target("CLASS")
 */
final class Controller
{
    /**
     * @Required()
     * @var string
     */
    private $prefix = '';
    public function __construct(array $value)
    {
        if (isset($value['value'])) $this->prefix = $value['value'];
        if (isset($value['prefix'])) $this->prefix = $value['prefix'];
    }

    public function getPrefix(){
        return $this->prefix;
    }
}
```

> **@Annotation**表示这是个注解类，让IDE提示更加友好！
>
> **@Target**表示这个注解类只能被类使用！
>
> **@Required**表示这个属性是必须填写的！
>
> **@Attributes**表示这个注解类有多少个属性！

创建`library\Mapping\RequestMapping.php`

```php
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */


namespace library\annotation\Mapping;


use Doctrine\Common\Annotations\Annotation\Required;
use Doctrine\Common\Annotations\Annotation\Target;

/**
 * Class RequestMapping
 * @package library\annotation\Mapping
 * @Annotation
 * @Attributes({
 *     @Attribute("route", type="string"),
 * })
 * @Target("METHOD")
 */
final class RequestMapping
{
    /**
     * @Required()
     */
    private $route;
    public function __construct(array $value)
    {
        if (isset($value['value'])) $this->route = $value['value'];
        if (isset($value['route'])) $this->route = $value['route'];
    }

    public function getRoute(){
        return $this->route;
    }
}
```

> 这里的代码就不用再重复解释了吧！我偏不解释了！哈哈哈

创建`app\Http\HomeController.php`

```php
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */


namespace app\Http;


use library\annotation\Mapping\Controller;
use library\annotation\Mapping\RequestMapping;

/**
 * Class HomeController
 * @package app\Http
 * @Controller(prefix="/home")
 */
class HomeController
{
    /**
     * @RequestMapping(route="/test")
     */
    public function test(){
        echo 111;
    }
}
```

> 这里代码没啥解释的。。。

哈哈，下面来测试我们的结果！

`index.php`

```php
<?php
/**
 * @author gaobinzhan <gaobinzhan@gmail.com>
 */

$loader = require './vendor/autoload.php';

// 获取一个反射类
$refClass = new \ReflectionClass('\app\Http\HomeController');

// 注册load
\Doctrine\Common\Annotations\AnnotationRegistry::registerLoader([$loader, 'loadClass']);

// new 我们的注解读取器
$reader = new \Doctrine\Common\Annotations\AnnotationReader();

// 获取该类上的所有注解
$classAnnotations = $reader->getClassAnnotations($refClass);

// 这是个循环 说明$classAnnotations是个数组
foreach ($classAnnotations as $annotation){
    // 因为我们定义了Controller注解类 要判断好啊
    if ($annotation instanceof \library\annotation\Mapping\Controller){
        // 获取我们的 prefix 这地方能看懂吧。。。
        echo $routePrefix = $annotation->getPrefix().PHP_EOL;
        // 获取类中所有方法
        $refMethods = $refClass->getMethods();
    }

    // 进行循环
    foreach ($refMethods as $method){
        // 获取方法上的所有注解
        $methodAnnotations = $reader->getMethodAnnotations($method);
        // 循环
        foreach ($methodAnnotations as $methodAnnotation){
            if ($methodAnnotation instanceof \library\annotation\Mapping\RequestMapping){
                // 输出我们的route
                echo $methodAnnotation->getRoute().PHP_EOL;
            }
        }
    }
}
```

执行结果：

```bash
/home
/test
```

之前我们不是建立了个**Parser**的目录嘛，可以在里面创建对应的解析类，然后去解析，把它们封装一下子！！


