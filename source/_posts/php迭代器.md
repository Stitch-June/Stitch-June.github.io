---
title: php迭代器
tags: []
id: '42'
categories:
  - - uncategorized
date: 2020-09-20 18:39:38
---

*   博主： gaobinzhan
*   发布时间：2018 年 12 月 25 日
*   701次浏览
*   暂无评论
*   945字数
*   分类： PHP

```
class Season implements Iterator{
    private $position = 0;//指针指向0
    private $arr = array('一','二','三','四');
    //重置迭代器
    public function rewind(){
        return $this -> position = 0;
    }
    //验证迭代器是否有数据
    public function valid() {
//        return isset($this -> arr[$this -> position]);
        return $this->positionarr);
    }
    //获取当前内容
    public function current(){
        return $this -> arr[$this -> position];
    }
    //获取迭代器位置key
    public function key(){
        return $this -> position;
    }
    //移动key到下一个
    public function next() {
        ++$this -> position;
    }
}
$obj = new Season();
foreach ($obj as $k => $v){
    echo $k.':'.$v.'';
}
```

结果：

0:一1:二2:三

3:四

##### php有对数组指针的操作，可不用定义$position

1.key();从关联数组中取得键名，没有取到返回NULL2.current();返回数组中当前单元3.next();将数组中的内部指针向前移动一位4.prev();将数组的内部指针倒回一位5.reset();将数组的内部指针指向第一个单元

6.end();将数组的内部指针指向最后一个单元

```
class Season implements Iterator{
    private $position = 0;//指针指向0
    private $arr = array('一','二','三','四');
    //重置迭代器
    public function rewind(){
        return $this -> position = 0;
    }
    //验证迭代器是否有数据
    public function valid() {
//        return isset($this -> arr[$this -> position]);
        return $this->positionarr);
    }
    //获取当前内容
    public function current(){
        return $this -> arr[$this -> position];
    }
    //获取迭代器位置key
    public function key(){
        return $this -> position;
    }
    //移动key到下一个
    public function next() {
        ++$this -> position;
    }
}
$obj = new Season();
foreach ($obj as $k => $v){
    echo $k.':'.$v.'';
}
```

结果：

0:一1:二2:三

3:四

##### php有对数组指针的操作，可不用定义$position

1.key();从关联数组中取得键名，没有取到返回NULL2.current();返回数组中当前单元3.next();将数组中的内部指针向前移动一位4.prev();将数组的内部指针倒回一位5.reset();将数组的内部指针指向第一个单元

6.end();将数组的内部指针指向最后一个单元