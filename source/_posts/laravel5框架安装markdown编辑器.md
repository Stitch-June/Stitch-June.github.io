---
title: Laravel5框架安装Markdown编辑器
tags: []
id: '47'
categories:
  - - uncategorized
date: 2020-09-20 18:57:00
---

`composer require chenhua/laravel5-markdown-editor`

##### 2.完成上面操作，修改 config/app.php 中 providers 数组

`Chenhua\MarkdownEditor\MarkdownEditorServiceProvider::class,`

##### 3\. 修改 config/app.php 中 aliases 数组

`'MarkdownEditor' => Chenhua\MarkdownEditor\Facades\MarkdownEditor::class,`

##### 4\. 执行 artisan 命令，生成 config/markdowneditor.php 配置文件

`php artisan vendor:publish --tag=markdown`

##### 5\. 根据自己所需修改 config/markdowneditor.php 配置文件

#### 使用方法

##### 在 xxx.blade.php 相应位置添加代码

```

    


@include('markdown::encode',['editors'=>['test-editormd']])
```

#### 解析 markdown 格式文本为 html 格式

① 前端js方式解析 markdown 文本

```

    
# 这是一个h1标签
## 这是一个h2标签
    

@include('markdown::decode',['editors'=>['doc-content']])
```

② PHP方式解析 markdown 文本

```
echo MarkdownEditor::parse("#中间填写markdown格式的文本");
```

## 效果图：

![](http://blog.gaobinzhan.com/uploads/article/20190106/2dee9273094eaabd676948a333cfbada.png)