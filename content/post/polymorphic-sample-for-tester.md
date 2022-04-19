---
title: 面向对象的多态
date: 2012-06-22 16:26:02
tags: ["OOD"]
---

本文以自动化测试为例子解释面向对象中的多态概念。 

从一个最简单的test case开始，打开一个网站，然后点击后退。这是一个国外的网站，所以要求同时兼容IE和Firefox。

首先，我们把测试脚本写出来。脚本很简单，描述每一个测试步骤。

```CSharp
void SimpleTestCase()
{
     Navigate();
     ValidateNavigate(); 

     Back();

     ValidateBack(); 
}
```

<!-- more -->

接下去要考虑的事情，就是看怎么实现具体的方法了。很幸运地，VS开始支持录制脚本了（类似QTP），所以我们不需要太担心如何调用浏览器的navigate和back操作，录制就好。针对navigate这个动作，我们针对IE和Firefox分别录了两次。

```CSharp
void NavigateInIE()
{
    //在IE打开网站
}

void NavidgateInFF()
{
    //在Firefox打开网站
} 
```

但是在试图把这两个函数统一起来的时候碰到了点问题，VS能抓到IE的对象，所以NavigateInIE里面是对象调用。但VS抓不到FF的对象，所以NavigateInFF是以模拟鼠标点击的方式实现的。（以上纯属虚构，主要想表达的是，NavigateInIE和NavigateInFF这两个方法区别很大，没有任何共同点可以抽取，或者说抽取公有函数的价值不高。）同样的BackInIE()和BackInFF()也会碰到类似的问题。

那么我们的test script会变成什么样呢？

```CSharp
void SimpleTestCase(string browser)
{
    switch (browser)
    {
        case "IE":
            NavigateInIE();
        case "FF":
            NavigateInFF();
    }    
    ValidateNavigate(); 
    switch (browser)
    {
        case "IE":
            BackInIE();
        case "FF":
            BackInIE();
    }
    ValidateBack(); 
}
```

test script的可读性瞬间变差了N个级别，test script里面我们应该只关心测试需要做的事情（打开网站），而不要关心事情怎么做（怎么能让IE打开一个网站）。我们试着把事情怎么做这些细节封装起来。

```CSharp
void SimpleTestCase(string browser)
{
    Navigate(browser);    
    ValidateNavigate(); 
    Back(browser);
    ValidateBack(); 
}

void Navigate(string browser)
{
    switch (browser)
    {
        case "IE":
            NavigateInIE();
        case "FF":
            NavigateInFF();
    }    
}

void Back(string browser)
{
    switch (browser)
    {
        case "IE":
            BackInIE();
        case "FF":
            BackInFF();
    }
}
```

我们可爱的test script又回来了，很好维护，测试要调用的时候也很方便。在两个浏览器里面测试，只要调用下面两句代码即可。

```CSharp
SimpleTestCase("IE");
SimpleTestCase("FF");
```

整个世界干净了，但是似乎发现了那么点坏味道。我们说，代码尽量不要copy & paste，这会影响代码维护。但是回顾Navigate和Back这两个方法，会发现它们的共同点非常多，这样的代码有问题吗？

如果不发生变化，代码其实无所谓可维护性，反正是给机器看的。但据说不会变化的只有变化本身，于是...

市场部经理提出，现在Chrome的占有率比Firefox高了，我们应该把它列入到测试范围。看看我们需要做什么：

1. 增加（录制）NavigateInChrome方法
2. 增加（录制）BackInChrome方法
3. 修改Navigate方法，为Chrome增加一条选择支
4. 修改Back方法，为Chrome增加一条选择支
5. 增加test script的调用，SimpleTestCase("Chrome")

其中3和4，它们的修改会非常相似，都是需要增加类似下面的代码。

```CSharp
case "Chrome":
     //Do something In Chrome
```

想像一下，如果我们在测试过程中，用到的浏览器功能不只是navigate和back，还有其他的100个浏览器功能，那增加对Chrome支持的时候，我们不是要把上面的代码复制102次？？有什么办法能解决这个问题呢？

我们先回顾一下上一篇文章里面出现问题的代码。

```CSharp
void Navigate(string browser)
{
    switch (browser)
    {
        case "IE":
            NavigateInIE();
        case "FF":
            NavigateInFF();
    }    
}

void Back(string browser)
{
    switch (browser)
    {
        case "IE":
            BackInIE();
        case "FF":
            BackInFF();
    }
}
```

在尝试解决重复代码时，我们会把相同的代码放到一个公共函数，把不同的代码以参数形式传递过去。在上面的代码中，不同的代码，其实就是4个方法，NavigateInXxx和BackInXxx。但是在试图抽取公共函数的时候，碰到了点麻烦。我们说了，不同的内容会作为参数，但是，现在的情况下，不同的内容是函数，函数怎么传递呢？ 貌似我们要开始考虑面向对象了，因为：1、对象可以作为参数传递。2、对象里面可以定义函数（方法）。

我们现在知道的信息是，各种NavigateInXxx和BackInXxx函数是需要传递的，那么这些函数都应该封装到对象里面，因为这些都是IE/Friefox的行为，我们很容易想到，要定义IE/Firefox类，因为方法被封装在了各自的类里面，InXxx的后缀就可以不要了。

```CSharp
class IE
{
    void Navigate();    //原来的NavigateInIE();
    void Back();        //原来的BackInIE();
}

class Firefox
{
    void Navigate();    //原来的NavigateInFF();
    void Back();        //原来的BackInFF();
}
```

我们又看到各种重复的代码，根据面向对象的思想，我们会试着为两个很类似的类建立父类Browser，然后让IE和Firefox都继承于Browser。

```CSharp
class Browser
{
    //声明为抽象函数，因为Browser并不知道IE/Firefox自己的具体实现
    abstract void Navigate();  
    abstract void Back();
}

class IE : Browser
{
    override void Navigate(){/*impl*/}    //原来的NavigateInIE()
    override void Back(){/*impl*/}        //原来的BackInIE()
}

class Firefox : Browser
{
    override void Navigate(){/*impl*/}    //原来的NavigateInFF()
    override void Back(){/*impl*/}        //原来的BackInFF()
}
```

至此，我们可以说，我的代码是面向对象的。但是，似乎除了多写一个Browser类，我们并没有得到任何的好处，IE和Firefox有各自的Navigate和Back的实现，没法通过继承父类得到，还是得自己重现一遍。那么，Browser类创建出来有什么用呢？

我们回到Navigate函数，现在因为把NavigateInIE和NavigateInFF分别封装到了IE和Firefox对象里面，我的代码可以写成这样：

```CSharp
void Navigate(string browser, Browser browserObj)
{
    switch(browser)
    {
        case "IE":
            (IE)browserObj.Navigate();            //把类型由Browser转义成IE
        case "FF":
            (Firefox)browserObj.Navigate();       //把类型由Browser转义成Firefox
    }
} 
```

这样稍微能看出来一点Browser类的作用了。 Navigate的第一个参数browser，可以接收"IE"和"FF"字符串，第二个参数，也应该可以接收IE对象和Firefox对象。怎么做到这点呢？用IE类和Firefox类的父类Browser作为参数类型即可。

我们已经成功地把一个函数封装在对象里，通过这个对象传递给了另一个函数。看看还有没有优化的余地。在调用browserObj.Navigate()的时候，我先把Browser类型转换成了具体的子类（IE/Firefox），但实际上并不需要这样。由于面向对象多态的支持，在代码运行时，运行环境有办法知道browserObj的真实类型，并不需要显示地转义。继续改进Navigate函数。

```CSharp
void Navigate(string browser, Browser browserObj)
{
    switch(browser)
    {
        case "IE":
            browserObj.Navigate();
        case "FF":
            browserObj.Navigate();
    }
}
```

这时候，会有点点凌乱，无论传递过去的brower是什么，走哪条选择支，最后被执行的语句都是browserObj.Navigate(); 那我为什么还要把browser传递过去呢？进一步地说，我是不是连选择判断都可以去掉呢？继续修改Navigate函数。

```CSharp
void Navigate(Browser browserObj)
{
    browserObj.Navigate();
}
```

神奇的事情发生了，之前引起维护困难的switch...case...语句消失了。 这时候，要测试IE的Navigate功能，代码如下。

```CSharp
Navigate(new IE());
```

我把IE对象传递过去，真正想传递的其实是把IE的Navigate方法。 当browserObj.Navigate()执行的时候，真正执行的是ie.Navigate()而不是browser.Navigate()，这就是传说中的多态...

而至此，Navigate函数也已经没有存在的必要了：我要调用Navigate函数的时候，要传过去一个browser对象，那我为什么不直接调用browser.Navigate()呢？Back同理。

说了那么多，把老板交下来的任务都忘了。回到我们的自动化测试，因为上面做的各种封装，我们的test script会变成这个样子。

```CSharp
void SimpleTestCase(Browser browser)
{
    browser.Navigate();   
    ValidateNavigate(); 
    browser.Back();
    ValidateBack(); 
}

SimpleTestCase(new IE());       //在IE下测试
SimpleTestCase(new Firefox());  //在Firefox下测试 
```

和之前其实没什么两样， 但一旦需求变更来到，我们就能看到不同了。还是那个问题，现在我们的脚本要支持在Chrome下测试，还是把要做的事情列出来：

1. 增加Chrome类。
2. 为Chrome类实现（录制）Navigate方法。
3. 为Chrome类实现（录制）Back方法。
4. 增加test script的调用。 SimpleTestCase(new Chrome()); 

我们把在前一种实现中，为了应对变化所需要做的事情也复制过来：

1. 增加（录制）NavigateInChrome方法
2. 增加（录制）BackInChrome方法
3. 修改Navigate方法，为Chrome增加一条选择支
4. 修改Back方法，为Chrome增加一条选择支
5. 增加test script的调用，SimpleTestCase("Chrome") 

有什么区别呢？

从工作量的角度看，第二种方式要多写一点搭建类框架的代码，但是不用修改switch case选择支，如前文说的，当我们实现的浏览器功能有100个的时候，可以发现比起省下来修改switch case的时间，多出来的搭建类框架的代码，工作量不值一提。

更重要的是，有没有发现第二种方式要做的，都是“增加”代码，而不是“修改”现有代码？这就是所谓的[开放封闭原则](https://baike.baidu.com/item/%E5%BC%80%E6%94%BE%E5%B0%81%E9%97%AD%E5%8E%9F%E5%88%99?fromTaglist=)。 