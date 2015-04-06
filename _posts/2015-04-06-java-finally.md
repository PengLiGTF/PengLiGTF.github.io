---
layout: post
title: Java finally block
excerpt: "Just about everything you'll need to style in the theme: headings, paragraphs, blockquotes, tables, code blocks, and more."
modified: 2015-054-06
tags: [java, try, finally, jvm]
comments: true
image:
  feature: sample-image-5.jpg
  credit: WeGraphics
  creditlink: 
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

##finally语句块

在java中，finally必须结合try语句块或try{}catch(e)语句块一起使用，用法有try{......}finally{......}
和try{......}catch(e){}finally{......}两种用法。根据java 8语言规范14.20 "try statement"章节中的描述，
当try语句块后面有跟一个finally语句块时，不管当前的try块和所有的catch块（如果有）是否正常或异常结束，
fianlly块中的程序逻辑都将得到执行。

为此，在使用jdk7（jdk7+中的try-with-resources会对实现了java.lang.AutoCloseable接口的资源进行自动关闭）
以前的版本进行java程序编写时，当程序中有涉及到资源（如流）释放操作时，程序员往往被要求一定将资源释放的
处理逻辑放到finally块中进行，确保资源得到释放。如：

流的关闭：
```java
   InputStream in = null;
   try{
       in = new FileInputstream("/home/xxx.txt");
   }catch(Exception e){
   }finally{
       if(in != null){
           in.close();
       }
   }
```
锁的释放：
```java
   try{
       lock.lock();   
   }finally{
       lock.unlock();
   }
```
由于finally块最后一定被执行，因此将资源的释放操作放在finally块中无疑是安全的做法，可以减少犯错的可能，
既然java虚拟机（JVM）给出保证一定执行在try块后面的finally块，那么问题来了，在下面的情况中程序会怎样执行呢？

情况一：
```java
    static int test()
  {
    int i = 10;
    try
    {
      return i;
    } finally
    {
      i = 100;
    }
  }
```
情况二：
```java
    static int test()
  {
    int i = 10;
    try
    {
      return i;
    } finally
    {
      i = 100;
      return 200;
    }
  }
```
情况三：
```java
    static boolean trueOrFlase(boolean flag)
    {
    while(true)
    {
      try{
        if(flag){
          return true;
          }
      }finally{
        break;
      }
    }
    return false;
  }
```
情况四：
```java
    static int test()
    {
    int i = 0;
    while (true)
    {
      try
      {
        try
        {
          i = 1;
        } finally
        {
          i = 2;
        }
        i = 3;
        return i;
      } finally
      {
        if (i == 3)
        {
          continue;
        }
      }
    }
  }
```
情况五：
```java
    static boolean trueOrFlase(boolean flag)
    {
    while (true)
    {
      try
      {
        if (flag)
        {
          return true;
        }
      } finally
      {
        throw new RuntimeException("");
      }
    }
  }
```
差不多了，先列出这五种情况，当下很多面试题中都有出现类似的题目，不要着急上机测试，先想想，科学证实 ,
人类对大脑的实际使用率只是10%左右，还有很多潜能没被开发出来，看了几期的“最强大脑”，我个人觉得
大家对大脑的使用率估计不到10%，你怎么看？

但是，坊间又有说法了，说可能存在的超级厉害的远古人类之所以消失了就是因为他们对大脑开发过度，所以现在大家
可以时不时的听到当下一些比较厉害的人Duang~的一下就挂了的新闻事件。然，知识诚可贵，小命价更高。想不出来还
是去上机吧，目的达到即可。


然，机上得来终觉浅，绝知此事要躬行。

首先这里有一个概念需要知道，finally块的正常结束和异常结束，先说异常结束，java规范中指出，如果在finally块中
有诸如：break,continue,return或是抛出异常的情况，则表明这个finally块属于异常结束一类的情况。反之，则表示该
finally块是正常结束。

在带有finally语句块的try语句块生成的java字节码中，方法中的finally块的表现为一个“微型子例程”（miniature subroutines），
并有一条相应的指令（jsr或jsr_w(jdk7以下版本中有，我用的是jdk8,生成的字节码中没有这个指令)）与之对应，当JVM在执行到
该指令时（表示方法正准备进入到finally块），它会将“return adddress”存放到局部变量栈中，然后JVM跳转到“微型子例程”开始处继续执行。
注意这里的”return address“（偏移量或本地指针）指的是紧跟着 jsr或jsr_w操作码和其操作数 之后的字节码的地址，其类型
为 ”returnAddress“。
在正常执行完finally语句块（微型子例程）后，JVM执行从”微型子例程序“中返回的'ret'指令，这个ret指令包含一个”index“操作数，"index"表示执行
这个“微型子例程“之前存储在局部变量栈中的”return address“的存放位置。执行完"ret"指令之后，jvm跳转到”jsr或jsr_w“之后的指令处继续执行。
注意这里的“ret"要和方法的返回区别开来，在字节码中，这里的“返回”对应的操作指令为“ret”，作用仅仅是用于在“微型子例程”
正常结束之后跳转到方法内部开始执行“微型子例程”的指令的下一个指令所处的位置。而上面提到了，存放到局部变量的”return address“表示的是”紧跟着 jsr或jsr_w操作码和其操作数 之后的字节码的地址“，
所以执行完”ret“指令之后接着就会执行jsr或jsr_w操作码和其操作数 之后的字节码的地址所表示的指令。

而在”微型子例程“异常结束时，将不会执行”ret“，也即JVM不会返回到执行”微型子例程“的开始处执行，而是直接走finally后面的代码块逻辑。


好了，现在让我们回答上面提到的几种情况程序会有什么样的输出：
情况一：
```java
    static int test()
    {
    int i = 10;
    try
    {
      return i;
    } finally
    {
      i = 100;
    }
  }
```
fianlly块是正常结束，JVM在执行到finally时，先将第6行的指令地址放到局部变量中进行了存储，执行完第9行中的指令之后，从
局部变量栈中取出之前存放的地址，然后jvm继续执行这条指令：return i,之前存放时i的值是10,所以取出来后仍然为10，所以
程序返回10

在来看情况二：
```java
    static int test()
    {
    int i = 10;
    try
    {
      return i;
    } finally
    {
      i = 100;
      return 200;
    }
  }
```
finally块中有return语句，属于异常结束，JVM将不会执行”ret“指令，不会执行第6行的 return i,因为finally中有返回操作，
故程序直接执行finally块中的返回操作，所以程序返回200.

而在情况三：
```java
    static boolean trueOrFlase(boolean flag)
    {
      while(true)
    {
      try{
        if(flag){
          return true;
          }
      }finally{
        break;
      }
    }
    return false;
  }
```
finally块中存在有break语句，finally块也是异常结束，同理，JVM也不会执行”ret“指令，而是直接跳出循环体执行 return false
对应的指令，所以程序返回false而不论参数为何。

情况四：
```java
    static int test()
    {
      int i = 0;
    while (true)
    {
      try
      {
        try
        {
          i = 1;
        } finally
        {
          i = 2;
        }
        i = 3;
        return i;
      } finally
      {
        if (i == 3)
        {
          continue;
        }
      }
    }
  }
```
代码原指望循环3此之后退出循环体程序返回3，但是，finally块中有continue语句，fianlly异常结束，JVM也将不执行”ret“指令，
而会继续循环体，导致出现死循环。

情况五：
```java
    static boolean trueOrFlase(boolean flag)
    {
      while (true)
    {
      try
      {
        if (flag)
        {
          return true;
        }
      } finally
      {
        throw new RuntimeException("");
      }
    }
  }
```
在finally块中抛出异常，finally块异常结束，JVM不会执行”ret“指令，因finally中抛出异常，故程序会异常终止。



##fianlly块总结

* finally块分为正常结束和异常结束
* finally块异常结束指在finally块中包含
    * break
    * continue
    * return
    * throw exception
* 当finally块正常结束时，程序会执行位于try或catchs中的返回操作，结果不会受到finally块中对待返回变量赋值操作的影响。
* finally块因return语句导致异常结束时，则程序将执行fianlly块中的返回操作。
* finally块因包含包含break或continue异常结束时，程序不会执行位于try或catchs中的返回操作，而会继续执行break和continue该有的逻辑。
* finally块因包抛出异常导致异常结束时，程序会异常终止。


##交流
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(lipeng82@126.com)



##参考
感谢以下的项目,排名不分先后

* [Inside the Java virtual machine]()
* [Java Language Specification]()


##关于作者
```java
public static void main
```
```javascript
  var ihubo = {
    nickName  : "Peng LI",
    site : "http://pengligtf.github.io"
  }
```



## Code Snippets

{% highlight css %}
#container {
  float: left;
  margin: 0 -240px 0 0;
  width: 100%;
}
{% endhighlight %}

## Buttons

Make any link standout more when applying the `.btn` class.

{% highlight html %}
<a href="#" class="btn btn-success">Success Button</a>
{% endhighlight %}

<div markdown="0"><a href="#" class="btn">Primary Button</a></div>
<div markdown="0"><a href="#" class="btn btn-success">Success Button</a></div>
<div markdown="0"><a href="#" class="btn btn-warning">Warning Button</a></div>
<div markdown="0"><a href="#" class="btn btn-danger">Danger Button</a></div>
<div markdown="0"><a href="#" class="btn btn-info">Info Button</a></div>

## Notices

**Watch out!** You can also add notices by appending `{: .notice}` to a paragraph.
{: .notice}
