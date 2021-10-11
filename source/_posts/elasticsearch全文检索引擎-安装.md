---
title: ElasticSearch全文检索引擎-安装
tags: []
id: '15'
categories:
  - - uncategorized
date: 2019-10-27 15:30:51
---

# ElasticSearch安装
#### 安装JDK

1. **下载rpm包**

    `wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "https://download.oracle.com/otn-pub/java/jdk/8u201-b09/42970487e3af4f5aa5bca3f542482c60/jdk-8u201-linux-x64.rpm"`

2. **rpm安装**

    `rpm -ivh jdk-8u201-linux-x64.rpm`
    安装成功后测试 java javac java -version 命令都能使用
    
    
 #### 安装ElasticSearch
 
 1. **官网**
    [ElasticSearch](https://www.elastic.co/)
   
 2. **下载**
 这里我下载的2.4.6版本
 
    `wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.4.6/elasticsearch-2.4.6.rpm`
    
3. **安装ElasticSearch**

    `yum install elasticsearch-2.4.6.rpm`
    
 4. **修改配置文件**
 
    `vim /etc/elasticsearch/elasticsearch.yml`
    
    `cluster.name: my-application`这行取消注释 并且改为自己的应用名称
    `node.name: node-1`  这行也一样 改为自己的节点名称
    为了方便操作 应用名称改为 `search`  节点名称为 `master`
    `network.host: 192.168.0.1` 取消注释 改为 `0.0.0.0`
    `http.port: 9200` 取消注释就行 可以自行修改端口号 
    切记 取消注释后 首字母前面不能有空格 不然启动会报错
    
5. **运行ElasticSearch**

    `service elasticsearch start`
    浏览器访问ip+端口号 如果出现界面证明 成功

#### 安装ik中文分词

1. **ik分词地址**

    [ik](https://github.com/medcl/elasticsearch-analysis-ik)
    
    
2. **安装git和maven**

    如果有的话跳过此步
    
    安装git
    `yum install -y git`
    
    下载maven包
    `wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz`
    
    解压到指定目录
`tar -xzvf apache-maven-3.5.4-bin.tar.gz -C /usr/local`

   进入指定目录
   `cd /usr/local/`
    
   创建软连接
   `ln -s apache-maven-3.5.4 maven`
   
   编辑文件 添加maven环境变量
   `vim /etc/profile`
   
   在文件末尾添加以下代码
   `MAVEN_HOME=//usr/local/maven`
   `export MAVEN_HOME`
   `export PATH=${PATH}:${MAVEN_HOME}/bin`

   运行命令
   `source /etc/profile`

    ` mvn -v `
   出现版本 安装maven成功
 
 
3. **安装ik分词**
  克隆
  `git clone https://github.com/medcl/elasticsearch-analysis-ik.git`
   
   根据elasticsearch版本号切换相对应的ik分词版本号 这里es是2.4.6 相对应是1.10.6
   
   克隆好之后进入该目录 
   `cd elasticsearch-analysis-ik/`
   
   切换版本
   `git checkout tags/v1.10.6`
    
   执行
  `mvn package`
  
   执行成功后会在 `target/releases/`  目录下生成一个插件包 进入该目录
  `cd target/releases/`
  
   复制到es的插件目录下解压
   `cp elasticsearch-analysis-ik-1.10.6.zip /usr/share/elasticsearch/plugins/`
   
   进入目录
   `cd /usr/share/elasticsearch/plugins/`
   
   解压
   `unzip elasticsearch-analysis-ik-1.10.6.zip`

   为了方便管理 新建个文件夹 把刚刚解压出来的文件移动进去
   `mkdir ik`
   `mv ./* ik`

   重启elasticsearch
   `service elasticsearch restart`

   测试下
   `curl -XPOST "http://ip:port/_analyze?analyzer=ik&pretty" -d '这是一个商品的标题'`


* * *


### ElasticSearch和ik分词安装成功