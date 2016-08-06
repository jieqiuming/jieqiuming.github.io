---
layout: post
title: JavaScript高级程序设计阅读笔记
category: 前端
tags: JavaScript,总结
date: 2014-11-02
---



## 一、可维护性：可理解性、直观性、可适应性、可扩展性、可调试性

### 代码约定：
#### 可读性
1. 格式化：建议缩进大小为4个空格
2. 注释：函数和方法、大段代码、复杂的算法、hack
#### 变量和函数命名
1. 变量名为名词
2. 函数名为动词开始
3. 变量和函数使用合符逻辑的名字，不要担心长度。
#### 变量类型透明：表示变量类型的三种方式
1. 初始化： var found = false; //布尔型
2. 使用匈牙利标记法来指定变量类型：o代表对象，s代表字符串，i代表整数，f代表浮点数，b代表布尔型
3. 使用类型注释： var found /*:Boolen*/ = false;

### 松散耦合：
#### 解耦HTML/JavaScript
1. HTML中包含JavaScript，示例：

    <script type="text/javascript">document.write("hello world!")</script>; // <script>标签紧密耦合
    <input type="button" value="Click me " onclick="doSomething();"/> //事件属性值紧密耦合 
   
   
2. 理想情况：HTML和JavaScript应该完全分离，并通过外部文件和使用DOM附加行为来包含JavaScript。
问题：出现JavaScript错误就要判断是在HTML中还是在JavaScript中，且在doSomething()可用之前就按下button，也会引发JavaScript错误。
3. JavaScript中包含HTML，JavaScript生成HTML，这个应该避免，保持层次的分离有助于很容易的确定错误来源。
4. 理想情况：JavaScript用于插入数据时，尽量不直接插入标记，可以控制标记的显示和隐藏，而非生成它。另一种方法是进行Ajax请求并获取更多要显示的HTML，这个方法可以让同样的渲染层（PHP、JSP、Ruby）来输出标记。
#### 解耦CSS/JavaScript
1. 利用JavaScript修改样式时，应该通过动态修改样式类而非特定样式来实现。
2. 显示问题的唯一来源应该是CSS，行为问题的唯一来源应该是JavaScript。
#### 解耦应用逻辑/事件处理程序
1. 应用逻辑和事件处理程序相分离，一个事件处理程序应该从事件对象中获取相关信息，并将这些信息传送到处理应用程序的某个方法中。
2. 好处：可以更容易更改触发特定过程的事件；其次可以在不附加到事件的情况下测试代码，使其更易创建单元测试或者是自动化应用流程。
3. 应用和业务逻辑之间松散耦合的几条原则：
  勿将event对象传给其他方法；只传来自event对象中所需的数据；
  任何在应用层面的动作都应该可以在不执行任何事件处理程序的情况下进行。
  任何事件处理程序都应该处理事件，然后将处理转交给应用逻辑。
### 编程实践：
#### 尊重对象所有权：
如果你不负责创建和维护某个对象、它的对象或者它的方法，那么你就不能对它们进行修改。
1. 不要为实例或者原型添加属性；
2. 不要为实例或者原型添加方法；
3. 不要重定义已存在的方法。
#### 避免全局变量：
最多创建一个全局量，让其他对象和函数存在其中。
1. 避免与null进行比较
2. 如果值应为一个引用类型，使用instanceof操作符检查其构造函数。
3. 如果值应为一个基本类型，使用typeof检查其类型。
4. 如果是希望对象包含某个特定的方法名，则使用typeof操作符确保指定名称的方法存在于对象上。
#### 使用常量
1. 关键在于将数据和使用它的逻辑进行分离
2. 重复值：任何在多处用到的值都应抽取为一个常量，也包含css类名，这就限制了当一个值变了而另一个没变的时候会造成的错误。
3. 用户界面字符串：方便国际化
4. URLs：在web应用中，资源位置很容易变更，所以推荐用一个公共地方存放所有的URL
5. 任意可能会更改的值


## 二、保证代码性能

### 注意作用域：
#### 避免全局查找：使用全局变量和函数肯定要比局部的开销更大，因为涉及作用域链上的查找。
1. 示例代码：

    function updateUI(){
        var imgs = document.getElementsByTagName("img");
        for(var i=0,len=imgs.length;i<len;i++)
        {
            imgs[i].title = document.title + " image " + i;
        }
        var msg = document.getElementById("msg");
        msg.innerHTML = "Update complete.";
    }

2. 优化后的代码


    function updateUI(){
        var doc = document;
        var imgs = doc .getElementsByTagName("img");
        
        for(var i=0,len=imgs.length;i<len;i++)
        {
            imgs[i].title = doc .title + " image " + i;
        }
        var msg = doc .getElementById("msg");
        msg.innerHTML = "Update complete.";
    }

#### 避免with语句：在性能非常重要的地方必须避免使用with语句。
1. 和函数类似，with语句会创建自己的作用域，肯定会增加其中执行的代码的作用域链的长度。
2. 必须使用with语句的情况很少，它主要用于消除额外的字符。在大多数情况下，可以用局部变量完成相同的事情而不用引入新的作用域。
3. 实例代码：

    function updateBody(){
        with(document.body){
            alert(tagName);
            innerHTML = "hello world!";
        }
    }


4. 改进后的代码：

    function updateBody(){
        var body = document.body;
        alert(body.tagName);
        body.innerHTML = "hello world!";
    }

### 选择正确方法
#### 避免不必要的属性查找
1. 常数值O(1)：指代字面值和存储在变量中的值，访问数组元素
2. 对数值O(log n)：
3. 线性O(n)：访问对象，对象上的任何属性查找都要比访问变量或者数组花费更长时间，因为必须在原型链中对拥有该名称的属性进行一次搜索，属性查找越多，执行时间久越长。
一旦多次用到对象属性，应该将其存储在局部变量中。
4. 平方O(n²)：
#### 优化循环：基本步骤如下：
1. 减值迭代：在很多情况下，从最大值开始，在循环中不断减值的迭代器更加高效。
2. 简化终止条件：由于每次循环过程都会计算终止条件，所以必须保证它尽可能快。也就是说避免属性查找或者其他O(n)的操作。
3. 简化循环体：循环体是执行最多的，所以要确保其被最大限度的优化。确保没有某些可以被很容易移除循环的密集计算。
4. 使用后测试循环：最常用的for循环和while循环都是前测试循环，而如do-while这种后测试循环，可以避免最初终止条件的计算，因此运行更快。
5. 示例代码：

    for(var i=0; i < values.length; i++){
        process(value[i]);
    }

6. 减值迭代优化：

    for(var i=values.length; i >= 0 ; i--){
        process(value[i]);
    }


7. 后测试循环优化：
记住使用后测试循环时必须确保要处理的值至少有一个，空数组会导致多余的一次循环而前测试循环则可以避免。

    var i = values.length - 1;
    if(i > -1){
        do{
            process(values[i]);
        }while(--i > 0);
    }


#### 展开循环
1. 当循环的次数是确定的，消除循环并使用多次函数调用往往更快。
2. 如果循环中的迭代次数不能事先确定，可以使用duff装置技术，它以创建者Tom Duff命名，并最早在C语言中使用这项技术。Jeff Greenberg 用JavaScript实现了Duff装置，基本概念是通过计算迭代的次数是否为8的倍数将一个循环展开为一系列语句。
3. Jeff Greenberg的Duff装置技术代码：通过将values数组中元素个数除以8来计算出循环需要进行多次迭代的。

    //credit: Jeff Greenberg for JS implementation of Duff's Device
    //假设values.length > 0
    var iterations = Math.ceil(values.length / 8);
    var startAt = values.length % 8;
    var i = 0;

    do{
        switch(startAt){
            case 0: process(values[i++]);
            case 1: process(values[i++]);
            case 2: process(values[i++]);
            case 3: process(values[i++]);
            case 4: process(values[i++]);
            case 5: process(values[i++]);
     
            case 6: process(values[i++]);
            case 7: process(values[i++]);
        }
     
        startAt = 0;
    } while (--iterations > 0);


4. 由Andrew B.King 所著的Speed Up your Site（New Riders，2003），提出了一个更快的Duff装置技术，将do-while循环分成2个单独的循环。一下是例子：

    //credit: Speed Up your Site（New Riders，2003）
    var iterations = Math.floor(values.length / 8);
    var leftover = values.length % 8;

    if(leftover > 0){
        do{
            process(values[i++]);
        }while(--leftover > 0);
    }
    do{
        process(values[i++]);
        process(values[i++]);
        process(values[i++]);
        process(values[i++]);
        process(values[i++]);
        process(values[i++]);
        process(values[i++]);
        process(values[i++]);
    }while(--iterations > 0);



5. 在这个实现中，剩余的计算部分不会再实际循环中处理，而是在一个初始化循环中进行除以8的操作。当处理掉额外元素，继续执行每次调用8次process()的主循环，这个方法几乎比原始的Duff装置实现快上40%。
6. 针对大数据集使用展开循环可以节省很多时间，不过对于小数据集，额外的开销则可能得不偿失。

#### 避免双重解释：
当JavaScript代码想解析JavaScript的时候就会存在双重解释惩罚。当使用eval函数或者是Function构造函数以及使用setTimeout()传一个字符串参数时都会发生这种情况。

1. 实例代码：

    //某些代码求值---避免
    eval("alert('hello world!')");

    //创建新函数---避免
    var sayHi = new Function("alert('hello world!')");

    //设置超时---避免
    setTimeout("alert('hello world!')",500);

2. 分析：以上代码中都要解析包含了JavaScript代码的字符串，这个操作是不能再初始的解析过程中完成的，因为代码是包含在字符串中的，也就是说在JavaScript代码运行的同时必须新启动一个解析器来解析新的代码。

3. 修正后的例子：

    //已修正
    alert('hello world!');

    //创建新函数---已修正
    var sayHi = function(){
        alert('hello world!');
    };

    //设置一个超时---已修正
    setTimeout(function(){
        alert('hello world!');
    },500);

#### 性能的其他注意事项：

1. 原生方法较快：原生方法是用诸如C/C++之类的编译型语言写出来的，所以要比JavaScript快的很多很多。JavaScript最容易被忘记的就是可以在Math对象中找到的复杂的数学运算，这些方法要比任何用JavaScript的同样方法如正弦、余弦快的多。
2. Switch语句较快
3. 位运算符较快：取模、逻辑与和逻辑或

### 最小化语句数：
JavaScript代码中的语句数量也影响所执行的操作的速度。完成多个操作的单个语句要比完成单个操作的多个语句快。
1. 多个变量声明
2. 插入迭代值
3. 使用素组和对象字面量

### 优化DOM交互
#### 最小化现场更新
1. 现场更新：需要立即（现场）对页面对用户的显示进行更新。每一个更改，不管是插入单个字符，还是移除整个片段，都有一个性能惩罚，因为浏览器要重新计算无数尺寸以进行更新。
2. 实例代码：

    var list = document.getElementById("myList"),
                    item,
                    i;
    for(i = 0; i < 10;i ++){
         item = document.createElement("li");
         list = appendChild(item);
         item.append(document.createTextNode("Item " + i));
    }

3. 分析：该代码添加每个项目时都有2个现场更新：一个添加<li>元素，另一个给它添加文本节点。总共需要20个现场更新。两种优化方法：第一种将列表从页面上移除，最后进行更新，最后再将列表插回到同样的位置，这个方法不理想，因为每次页面更新的时候会不必要的闪烁。第二个方法是使用文档片段来构建DOM结构，接着将其添加到List元素中，这种方法避免了现场更新和页面闪烁问题。
4. 优化后的代码：

    var list = document.getElementById("myList"),
                    fragment.document.createDocumentFragment(),
                    item,
                    i;
    for(i = 0; i < 10;i ++){
         item = document.createElement("li");
         fragment.appendChild(item);
         item.appendChild(document.createTextNode("Item " + i));
    }

    list.appendChild(fragment);

#### 使用innerHTML:
1. 页面中创建DOM节点的方法有：使用诸如createElement()和appendChild()之类的DOM方法，以及使用innerHTML。对于小的DOM更改而言，两种方法效率都差不多。然而对于大的DOM更改，使用innerHTML要比使用标准DOM方法创建同样的DOM结构快得多。
2. 因为当把innerHTML设置为某个值时，后台会创建一个HTML解析器，然后使用内部的DOM调用来创建DOM结构，而非基于JavaScript的DOM调用。由于内部方法是编译好的而非解释执行的，所以执行快的多。
3. 调用innerHTML（和其他DOM操作一样）关键在于最小化调用它的次数。
#### 使用事件代理
1. 页面上的事件处理程序的数量和页面响应用户交互的速度之间有个负相关，为了减轻这种惩罚，最好使用事件代理。
2. 事件代理用到了事件冒泡，任何可以冒泡的事件都不仅仅可以在事件目标上进行处理，目标的任何祖先节点上也能处理，因此可以将事件处理程序附加到更高层的地方负责多个目标的事件处理，如果在文档级别附加事件处理程序，就可以处理整个页面的事件。
#### 注意HTMLCollection
1. 访问HTMLCollection，不管它是一个属性还是一个方法，都是在文档上进行一个查询，而且这个查询开销很昂贵，最小化访问HTMLColletcion的次数可以极大地改进脚本性能。
2. 优化HTMLCollection访问最重要的地方在于循环
实例代码：

    var images = document.getElementsByTagName("img"),image,i,len;

    for(i=0,len=images.length;i < len;i++){
         image = images[i];
        //处理
    }

3. 何时会返回HTMLCollection对象：
  进行了对getElementsByTagName()的调用
  获取了元素的childNodes属性
  获取了元素的attributes属性
  访问了特殊的集合，如document.forms、document.images等。