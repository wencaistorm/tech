---
title: Promise的顺序执行和并行执行
date: 2016-01-23 22:52:26
tags: [Promise, parallel, sequential, 顺序执行，并行执行]
categories: JS
---
并行和顺序执行的前提当然是，有一堆promise等着你去执行……
通常我们把这“一堆”promise对象放到一个数组里，`[promise1, promise2, promise3, ...]`

我们都知道想要让promise按顺序执行，那就是一个接一个的then。然而手写很多then太累了，而嵌套的promise又是反模式，我之前蠢蠢的[用递归解决了顺序执行的问题](tech/2016/01/18/sequentialize-promise-by-recursion/)，后来终于在udacity习得了顺序promise的正确打开方式！

### 场景
假如我们要去拿github按关键字搜索google, amazon, facebook，每个关键字搜出的第一个用户的第一个repo的名字。
单独拿一个，比如google，过程是这样的：
1. 请求https://api.github.com/search/users?q=google ，得到用户列表，读取第一个用户的repos_url
2. 请求上一步读取的repous_url，获得repo列表，读取第一个repo的名字
显然，第二步是依赖于第一步的执行结果的。

因此，很容易想到下面几种情况：
1. 等第一步全部执行完，再执行第二步（并行执行）
```
|--------------------|
google第一步          |------------------|
amazon第一步        google第二步
facebook第一步      amazon第二步 
                   facebook第二步
```
2. 对第一步的顺序有要求，必须严格按照google, amazon, facebook的顺序执行（顺序执行）
```
|---------|
google1   |---------|
            google2 |---------|
                     amazon1  |---------|
                              amazon2   |---------|
                                        facebook1 |--------|
                                                  facebook2
```
3. google1执行完就执行google1，amazon1执行完就执行amazon2，但对google, amazon, facebook的顺序没有要求
```
|------------------|
google1            |-------------|
                   google2
|---------|
amazon1   |-----------|
          amazon2
|------------|
facebook1    |-------------|  
             facebook2        
```

### Promise并行执行
对于场景1，主要就是用到`Promise.all`，因为是数组，所以在处理的过程中通常会用到`.map`或`.forEach`
{% jsfiddle HiiTea/zfjvr4pz/1 %}
![promise例1网络请求时间线](http://7xow88.com1.z0.glb.clouddn.com/tech-promise1.png)
可以看到3个user请求是同时进行（并行），3个repos请求也是同时进行（并行），由于使用了`Promise.all`，所以repos请求等待users请求全部完成才开始。

### promise顺序执行的正确打开方式
对于场景2，有一个小技巧，我第一次看到的时候感受是…惊为天人
按照顺序执行，容易想到的有以下几种方法：
1. then.then.then，从头then到尾…
2. then(then(then()))，then的嵌套…

promise链的本质其实就是从头then到尾，但是第一种方法怎么用程序来实现，也就是上面提到的小技巧，就是值得学习的地方了。（反正我觉得太巧妙了！！！我自己就想不到…）

大概跟在做求和运算时候的思想一样：
```
//求和的时候通常这么做，先定义一个sum，然后依次往里做加法
var sum = 0;
array.forEach(function(item) {
  sum = sum + item;
});
```
```
//要得到一个then then then的promise链，先定义一个已经resolve了的promise，然后依次往后then…
var sequence = Promise.resolve();
array.forEach(function(item) {
  sequence = sequence.then(...)
});
```
下面给出场景2的代码：
{% jsfiddle HiiTea/sq2aga08 %}
![promise例2网络请求时间线](http://7xow88.com1.z0.glb.clouddn.com/tech-promise2.png)
可以看到请求是按顺序依次发出

### 并行执行和顺序执行混用
场景3:
{% jsfiddle HiiTea/z09xjowq/1 %}
![promise例3网络请求时间线](http://7xow88.com1.z0.glb.clouddn.com/tech-promise3.png)
可以看到users请求是并行发出，但完成时间不一样，而repos请求是在**对应的**users请求完成后就立即执行

### 混用的另一种情况
有了以上的知识，很容易写出最后一种混用的情况，users按顺序执行，等users全部执行完之后并发执行repos。我比较懒…这个我就不写了…

### 补充知识：
#### 浏览器的fetch API
本文跟fetch不是充分必要关系，只是我太懒了不想写太多的代码来举例。你可以把它理解为是用Promise包住的`$.ajax`，也就是fetch返回一个promise对象。关于fetch的详细用法请参考[MDN Fetch_ API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)

#### 如何查看请求的发送是并行还是顺序
打开chrome，按F12，选中Network选项卡，在No throttling这个下拉列表选GPRS，再运行代码，然后就能清楚的看到各个请求的时间线了。

如果看到的区别不是很明显，很可能是已经缓存了，清空浏览器的缓存在重复上面的步骤就能看到比较明显的区别了。

### 参考：
[1] https://www.udacity.com/course/viewer#!/c-ud898
