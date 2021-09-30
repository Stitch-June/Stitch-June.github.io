---
title: Laravel使用全文检索引擎TNTSearch和中文分词jieba-php实现中文全文搜索
tags: []
id: '9'
categories:
  - - uncategorized
date: 2019-10-27 00:16:39
---

搜索基本就是每个网站必备的虽然是搜索 但是去搜索我是一个php程序员，我是一个java程序员完全是两个概念一般搜索都是 sql 的 like 去查询比如 php是世界上最好的语言 这句话如果我们用 sql 的 like 的 % 模糊查询的话搜索的词要连贯并且一字不差；

可以搜 `like 'php是%'` 也可以搜 `like '%最好的语言'` 或者 like `'%世界上最好%'` ；

但你要是搜索 世界最好语言 还能搜索到吗？？像 百度谷歌 可能会让用户一字不差的输入进去吗这时候就会用到我们的全文搜索简单的来说全文搜索的原理就是把内容按关键字给拆分了比如说上面这句话拆成 php 、世界 、最好 、 语言

也就是php不用依赖第三方实现全文搜索的TNTSearch

英文句子实现比较简单可以按空格去拆分而中文 它并不懂 世界 、最好 、 语言 这些是词语会把它给拆成单个字这时候就需要中文分词了中文分词就是会智能按中文的词语来拆分成关键字

作为 TNTSearch 中文分词的jieba-php

为了方便 先创建一个测试表和测试模型

`php artisan make:model Models/Test -m`

添加字段  
/database/migrations/2019\_01\_16\_055730\_create\_tests\_table.php

1.  `/**`
2.  `* Run the migrations.`
3.  `*`
4.  `* @return void`
5.  `*/`
6.  `public function up()`
7.  `{`
8.  `Schema::create('tests', function (Blueprint $table) {`
9.  `$table->increments('id');`
10.  `$table->string('title')->default('')->comment('测试标题');`
11.  `$table->mediumText('content')->comment('测试内容');`
12.  `$table->timestamps();`
13.  `});`
14.  `}`

.env文件数据库配置 就不用多说了吧

运行迁移命令生成数据表

`php artisan migrate`

创建数据填充文件

`php artisan make:seed TestsTableSeeder`

生成测试数据  
/database/seeds/TestsTableSeeder.php

1.  `public function run()`
2.  `{`
3.  `DB::table('tests')->insert([`
4.  `[`
5.  `'title' => 'TNTSearch',`
6.  `'content' => 'PHP编写的全文搜索引擎,非常强大'`
7.  `],`
8.  `[`
9.  `'title' => 'jieba-php',`
10.  `'content' => '强大的中文分词，最好的php中文分词，中文拆分成关键字'`
11.  `]`
12.  `]);`
13.  `}`

运行填充

`php artisan db:seed --class=TestsTableSeeder`

这时候可以看到数据表里面有数据了  
![](http://blog.gaobinzhan.com/uploads/article/20190116/c338522447c77fac5ff37befbb2f336d.png)

接下来测试

/routes/web.php

1.  `use App\Models\Test;`
2.  `Route::get('search', function () {`
3.  `dump(Test::all()->toArray());`
4.  `});`

进入web页面 显示以下数据  
![](http://blog.gaobinzhan.com/uploads/article/20190116/fd1f2e0093ad3628c94408bb557c1b7c.png)

依赖 SQLite 存储索引  
再确认下自己的 php 是否开启以下扩展：

`pdo_sqlite`

`sqlite3`

`mbstring`

vanry为我们造的轮子 laravel-scout-tntsearch

直接通过composer安装

`composer require vanry/laravel-scout-tntsearch`

config/app.php

1.  `// config/app.php`
2.  `'providers' => [`
3.  `// ...`
4.  `Laravel\Scout\ScoutServiceProvider::class,`
5.  `Vanry\Scout\TNTSearchScoutServiceProvider::class,`
6.  `],`

在 .env 文件中添加 SCOUT\_DRIVER=tntsearch  
然后就可以将 scout.php 配置文件发布到 config 目录。

`php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"`

在 config/scout.php 中添加:

1.  `'tntsearch' => [`
2.  `'storage' => storage_path('indexes'), //必须有可写权限`
3.  `'fuzziness' => env('TNTSEARCH_FUZZINESS', false),`
4.  `'searchBoolean' => env('TNTSEARCH_BOOLEAN', false),`
5.  `'asYouType' => false,`
6.  `'fuzzy' => [`
7.  `'prefix_length' => 2,`
8.  `'max_expansions' => 50,`
9.  `'distance' => 2,`
10.  `],`
11.  `'tokenizer' => [`
12.  `'driver' => env('TNTSEARCH_TOKENIZER', 'default'),`
13.  `'jieba' => [`
14.  `'dict' => 'small',`
15.  `//'user_dict' => resource_path('dicts/mydict.txt'), //自定义词典路径`
16.  `],`
17.  `'analysis' => [`
18.  `'result_type' => 2,`
19.  `'unit_word' => true,`
20.  `'differ_max' => true,`
21.  `],`
22.  `'scws' => [`
23.  `'charset' => 'utf-8',`
24.  `'dict' => '/usr/local/scws/etc/dict.utf8.xdb',`
25.  `'rule' => '/usr/local/scws/etc/rules.utf8.ini',`
26.  `'multi' => 1,`
27.  `'ignore' => true,`
28.  `'duality' => false,`
29.  `],`
30.  `],`
31.  `'stopwords' => [`
32.  `'的',`
33.  `'了',`
34.  `'而是',`
35.  `],`
36.  `],`

目前支持 jieba, phpanalysis 和 scws 中文分词，在 .env 文件中配置 TNTSEARCH\_TOKENIZER 可选值 为 jieba, analysis, scws, default， 其中 default 为 TNTSearch 自带的分词器。  
考虑到性能问题，建议生产环境使用由C语言编写的scws分词扩展。

使用 jieba 分词器，需安装 fukuball/jieba-php

`composer require fukuball/jieba-php`

使用 phpanalysis 分词器，需安装 lmz/phpanalysis

`composer require lmz/phpanalysis`

使用 scws 分词器，需安装 vanry/scws

`composer require vanry/scws`

分别在 config/scout.php 中的 jieba, analysis 和 scws 中修改配置。

这里我使用的jieba 先安装 然后在.env文件配置TNTSEARCH\_TOKENIZER=jieba

模型中定义全文搜索；  
/app/Models/Test.php