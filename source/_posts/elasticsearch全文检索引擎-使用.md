---
title: ElasticSearch全文检索引擎-使用
tags: []
id: '64'
categories:
  - - uncategorized
date: 2020-09-23 06:52:22
---

\[TOC\]

#### 手册官网

es-docs  
es-php-docs

#### 索引

1.  **创建索引**
    
    `vim createindex.json`
    
    ```
     {
         "settings": {
             "refresh_interval": "5s",
             "number_of_shards": 1,
             "number_of_replicas": 0
         },
         "mappings": {
             "_default_": {
                 "_all": {
                     "enabled": true
                 }
             }
         },
         "products": {
             "dynamic": false,
             "properties": {
                 "productid": {
                     "type": "long"
                 },
                 "title": {
                     "type": "string",
                     "index": "analyzed",
                     "analyzer": "ik"
                 },
                 "descr": {
                     "type": "string",
                     "index": "analyzed",
                     "analyzer": "ik"
                 }
             }
         }
     }
    ```
    
    具体的字段意思手册上都有
    
    创建
    
    `curl -XPOST "http://ip:port/shop" -d'@createindex.json`
    
    这里的shop就是我们要设置的索引名称 可以自定义  
    products 是 索引的type类型 可以自定义
    
2.  **添加数据**
    
    `curl -XPUT "http://119.23.51.33:9200/shop/products/1" -d'{"productid":1,"title":"这是一个商品的标题","descr":"这是一个商品的描述"}'`
    
    `curl -XPUT "http://ip:port/shop/products/2?pretty" -d'{"productid":2,"title":"这是一部手机","descr":"这是一个苹果手机的描述信息"}'`
    
    url地址后面的整值要跟文章id一致
    
    pretty就是 Elasticsearch美化输出(pretty-print)JSON响应以便更加容易阅读。
    
3.  **搜索数据**
    
    `vim search.json`
    
    ```
    {
         "query": {
             "multi_match": {
                 "query": "苹果",
                 "fields": [
                     "title",
                     "descr"
                 ]
             }
         },
         "highlight": {
             "pre_tags": [
                 ""
             ],
             "post_tags": [
                 ""
             ],
             "fields": {
                 "title": {},
                 "descr": {}
             }
         }
     }
    ```
    
    query 要查询的关键字 fields 哪些字段需要被查询 highlight 被查询出来的关键字 进行一些改变 相当于高亮显示
    
    pre\_tags 标签开始 post\_tags标签结束
    

测试

`curl -XGET "http://ip:port/shop/_search?pretty" -d"@search.json"`

#### elasticsearch-jdbc

将数据库的数据同步到elasticsearch

1.  **下载**
    
    `wget http://xbib.org/repository/org/xbib/elasticsearch/importer/elasticsearch-jdbc/2.3.4.0/elasticsearch-jdbc-2.3.4.0-dist.zip`
    
    `unzip elasticsearch-jdbc-2.3.4.0-dist.zip`
    
2.  **修改配置**
    
    `cd elasticsearch-jdbc-2.3.4.0/bin`
    
    为了防止出错 先备份一下
    
    `cp mysql-blog.sh mysql-blog.sh.bak`
    
    `vim mysql-blog.sh`
    
    将 echo 下的内容替换成以下内容
    
    ```
       {
           "type" : "jdbc",
           "jdbc" : {
               "url" : "jdbc:mysql://localhost:3306/good",
               "user" : "root",
               "password" : "1C292567e208",
               "sql" : "select productid,title,descr,productid as _id from test",
               "index" : "shop",
               "type" : "products",
               "elasticsearch" : {
                    "cluster" : "search",
                    "host" : "ip",
                    "port" : 9300
               }   
           }
       }
    ```
    
    url 是我们的数据库 good是库名 对哪个库进行操作 user password 用户名和密码 sql语句 就是 执行后 把查询出来的数据添加到es中 这里呢productid as \_id 是因为es的唯一标识是根据\_id生成的 index 我们刚刚创建的索引名称 products 是索引type类型 cluster 是我们es配置文件中的项目名称
    
    host port 分别为ip和端口号 port为9300 这个地方不可以改
    
    1.  **运行**
        
        `./mysql-blog.sh`
        
        `curl -XGET "http://119.23.51.33:9200/shop/_search?pretty"`
        
        可以看到数据库的内容被添加进去了
        
        但是修改数据库内容 或者 添加内容 es却不会改变 下面怎么改变呢
        
3.  **监控 实时同步**
    
    ```
       {
           "type" : "jdbc",
           "jdbc" : {
               "url" : "jdbc:mysql://localhost:3306/good",
               "schedule" : "0 0-59 0-23 ? * *",
               "user" : "root",
               "password" : "1C292567e208",
               "sql" : [{
                       "statement": "select productid,title,descr,productid as _id from test where updatetime > unix_timestamp(?) ",
                       "parameter": ["$metrics.lastexecutionstart"]}
                   ],
               "index" : "shop",
               "type" : "products",
               "metrics": {
                   "enabled" : true
               },
               "elasticsearch" : {
                    "cluster" : "search",
                    "host" : "ip",
                    "port" : 9300
               }   
           }
       }
       
    ```
    
    这时需要一个字段去判断 那就是我们数据的修改时间 parameter 里面的内容 就是获取上次脚本执行时间
    
    sql 的where条件 就是判断 修改时间大于上次脚本执行时间 就属于新数据
    
    重新执行下脚本 就可以了
    
    怎么用php操作呢
    
    现在有好多框架的轮子
    
    或者直接看es-php-docs
    
    php框架：
    
    laravel-elasticsearch  
    yii2-elasticsearch  
    去GitHub上查找就行