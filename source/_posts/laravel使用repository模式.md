---
title: Laravel使用repository模式
tags: []
id: '66'
categories:
  - - uncategorized
date: 2020-09-23 07:14:02
---

> **什么是Repository模式？**Repository 模式是架构模式，在设计架构时，才有参考价值； Repository 模式主要是封装数据查询和存储逻辑； Repository 模式实际用途：更换、升级 ORM 引擎，不影响业务逻辑； Repository 模式能提高测试效率，单元测试时，用 Mock 对象代替实际的数据库存取，可以成倍地提高测试用例运行速度。
> 
> 详细了解 https://blog.csdn.net/ZuoAnYinXiang/article/details/80711936
> 
> **REPOSITORY模式是怎样工作的呢？**Repository 是一个独立的层，介于领域层与数据映射层（数据访问层）之间。它的存在让领域层感觉不到数据访问层的存在，它提供一个类似集合的接口提供给领域层进行领域对象的访问。Repository 是仓库管理员，领域层需要什么东西只需告诉仓库管理员，由仓库管理员把东西拿给它，并不需要知道东西实际放在哪。
> 
> 详细了解 http://www.jquerycn.cn/a\_17077

当controller不使用Repository模式 ，在controller的各个方法中存在花式的数据库操作(这是非常糟糕的)，如果需求变更，重写将变得非常困难。

## Laravel如何部署

Laravel 5 Repositories用于抽象数据层，使我们的应用程序更易于维护。

### 安装

`composer require prettus/l5-repository`

### laravel部署

laravel>=5.5  
在框架的`config/app.php`中的`providers`数组添加如下代码：‘

`Prettus\Repository\Providers\RepositoryServiceProvider::class,`

发布配置

`php artisan vendor:publish --provider "Prettus\Repository\Providers\RepositoryServiceProvider"`

### 命令

要生成模型所需的所有内容，请运行以下命令：

`php artisan make:entity Post`

这将创建Controller，Validator，Model，Repository，Presenter和Transformer类。它还将创建一个新的服务提供程序，用于将Eloquent Repository与其相应的Repository Interface绑定。要加载它，只需将其添加到AppServiceProvider @ register方法：

`$this->app->register(RepositoryServiceProvider::class);`

### 自定义使用方法

在你的控制器中

```
namespace App\Http\Controllers;

use App\PostRepository;

class PostsController extends Controller {

    /**
     * @var PostRepository
     */
    protected $repository;

    public function __construct(PostRepository $repository){
        $this->repository = $repository;
    }
    public function index(){
        return $this->repository->all();
    }
}
```

更多操作：GitHub : https://github.com/andersao/l5-repository/tree/3.0-develop