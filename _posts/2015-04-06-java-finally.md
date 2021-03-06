---
layout: post
title: Java finally block 原理,结合实例从字节码层面深入理解Java finally机制
excerpt: "java finally block实现原理探究,结合实例从字节码层面深入理解Java finally机制"
modified: 2015-04-06
tags: [java, try, finally, jvm, 面试, 原理, 机制,字节码,实例]
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

##概述
当下，很多程序员对java finally的了解来自网上的一些面试题目，对其机制的了解仅仅停留在简单的概念层面上，稍微换个方式提问就“死机”了。
本文将从JVM底层机制结合字节码的方式分析研究java finally的运行机制，彻底掌握其运行机制。

##finally作用和常用表现形式

根据java language specification中描述：当try语句块后面有跟一个finally语句块时，
不管当前的try块和所有的catch块（如果有）是否正常或异常结束，fianlly块中的程序逻辑都将得到执行。这就能够确保放在finally块中的资源
释放逻辑得到执行，避免了因资源忘记释放而导致程序BUG的可能。常见的在finally块中进行资源释放的处理逻辑有：

流的关闭：
{%  highlight java linenos %}
   InputStream in = null;
   try{
       in = new FileInputstream("/home/xxx.txt");
   }catch(Exception e){
   }finally{
       if(in != null){
           in.close();
       }
   }
{% endhighlight %}
锁的释放：
{%  highlight java linenos %}
   try{
       lock.lock();   
   }finally{
       lock.unlock();
   }
{% endhighlight %}

在java中，finally必须结合try语句块或try{}catch(e)语句块一起使用，常用方式有：
{% highlight java linenos %}
    try{
      //do something
  }finally{
    //do something
  }

{% endhighlight %}
和：
{% highlight java linenos %}

      try{
      //do something
    }catch(Exception e){
      //do something
    }
    finally{
      //do something
    }
{% endhighlight %}

在jdk7+中，由于引入了“try-with-resources”机制，finally块在某些情况下可以不再需要（jdk7+中的try-with-resources会对实现了java.lang.AutoCloseable接口的资源进行自动关闭）。
但是，这不能成为我们不对finally运行机制理解的理由。

##问题

既然java虚拟机（JVM）给出保证一定执行在try块后面的finally块，且在finally中可以有任何符合java语法的java代码，那么问题来了，在下面的情况中程序会怎样执行呢？

情况一：
{%  highlight java linenos %}
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
{% endhighlight %}
情况二：
{%  highlight java linenos %}
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
{% endhighlight %}
情况三：
{%  highlight java linenos %}
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
{% endhighlight %}
情况四：
{%  highlight java linenos %}
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
{% endhighlight %}
情况五：

{%  highlight java linenos %}
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
{% endhighlight %}

差不多了，先列出这五种情况，当下很多面试题中都有出现类似的题目，如果你能很快得出正确的答案，表明你已经理解了finally的运行机制，如果你还犹豫，那就一起再研究研究。
如果你直接copy代码去机上运行快速地得出结论，表明你是个快枪手。要知道：机上得来终觉浅，绝知此事要躬行。

##分析探讨

首先这里有两个重要的概念需要知道，finally块的正常结束和异常结束，先说异常结束，java规范中指出，如果在finally块中
有诸如：break,continue,return或是抛出异常的情况，则表明这个finally块属于异常结束一类的情况。反之，则表示该
finally块是正常结束。



简单来说，当JVM执行带有finally块的try块时，在执行完try块准备执行finally块之前，先将try块中的返回值（如果有）存放到栈中局部变量区中，然后执行finally块，

* 如果finally块正常结束，则从局部变量区中取出之前存放的值进行返回或者是覆盖掉之前存放的值，继续执行finally块后面的语句。
* 如果finally块异常结束:
   * 如果因为finally块中包含了return语句,则，jvm会直接从finally块中进行返回，而会抛弃掉之前在try块中存放到局部变量区中的值。
   * 如果因为finally块中包含了break语句,则JVM会跳出finally块所属的循坏块，继续执行循坏块后续的语句，放弃try块中的返回操作（如果有）
   * 如果因为finally块中包含了break语句包含了continue语句,则JVM会放弃try块中的返回操作，继续进行循坏操作，这种情况下有可能导致死循坏。
   * 如果因为finally块中因为抛出异常，则真个程序会异常终止。

更专业的（来自Inside into Java virtual machine)阐述如下：

在带有finally语句块的try语句块生成的java字节码中，方法中的finally块的表现为一个“微型子例程”（miniature subroutines），
并有一条相应的指令（jsr或jsr_w(jdk5以下版本中有)）与之对应，当JVM在执行到该jsr或jsr_w指令时，它会先将“return adddress”存放到局部变量栈中，然后JVM跳转到“微型子例程”
也即是finally块开始处继续执行。

注意这里的”return address“（偏移量或本地指针）指的是紧跟着 jsr或jsr_w操作码和其操作数 之后的字节码的地址，其类型为 ”returnAddress“。

在正常执行完finally语句块（微型子例程）后，JVM执行从”微型子例程序“中返回的'ret'指令，这个ret指令包含一个”index“操作数，"index"表示执行
这个“微型子例程“之前存储在局部变量栈中的”return address“的存放位置。执行完"ret"指令之后，jvm跳转到”jsr或jsr_w“之后的指令处（也即之前保存的“return address”表示的指令）继续执行。

注意这里的“ret"要和方法的返回区别开来，在字节码中，这里的“返回”对应的操作指令为“ret”，作用仅仅是用于在“微型子例程”
正常结束之后跳转到方法内部开始执行“微型子例程”的指令的下一个指令所处的位置。而上面提到了，存放到局部变量的”return address“表示的是”紧跟着 jsr或jsr_w操作码和其操作数 之后的字节码的地址“，
所以执行完”ret“指令之后接着就会执行jsr或jsr_w操作码和其操作数 之后的字节码的地址所表示的指令。

而在”微型子例程“异常结束时，将不会执行”ret“，也即JVM不会返回到执行”微型子例程“的开始处执行，而是直接走finally后面的代码块逻辑。


##实例解析

了解了finally的运行机制之后，现在让我们回答上面提到的几种情况程序会有什么样的输出：
情况一：
{%  highlight java linenos %}
    static int test()
    {
    int i = 10;
    try
    {
      return i;
    } finally
    {
      System.out.println("this is finally");
      i = 100;
    }
  }
  
  字节码：
     static int test();
    Code:
       0: bipush        10
       2: istore_0
       3: iload_0
       4: istore_1
       5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: ldc           #3                  // String this is finally
      10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: bipush        100
      15: istore_0
      16: iload_1
      17: ireturn
      //jvm为执行try块中发生异常时的异常处理
      18: astore_2
      19: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      22: ldc           #3                  // String this is finally
      24: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: bipush        100
      29: istore_0
      30: aload_2
      31: athrow
    Exception table:
       from    to  target type
           3     5    18   any

  
{% endhighlight %}
因为fianlly块是正常结束，JVM在执行到finally时（见上字节码），在第4行（4： istore_1）将要返回的值放到局部变量区中的第一个位置进行了存储，然后执行
finall块中的逻辑，执行完输出语句后，将100压入栈中，随即弹出栈存放到局部变量区中的第0个位置，至此finally块执行完成，finally块执行完成之后，从
局部变量区中的第一个位置取出之前存放的地址（16: iload_1），然后jvm继续执行这条指令：ireturn(17：)完成返回操作,而之前存放时i的值是10,所以取出来后仍然为10，所以
程序返回10

再来看情况二：
{%  highlight java linenos %}
    static int test()
    {
    int i = 10;
    try
    {
      return i;
    } finally
    {
      System.out.println("this is finally");
      i = 100;
      return 200;
    }
  }
  字节码：
   static int test();
    Code:
       0: bipush        10
       2: istore_0
       3: iload_0
       4: istore_1     //将待返回的值存放到局部变量区中的第一个位置
       5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       8: ldc           #3                  // String this is finally
      10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      13: bipush        100
      15: istore_0
      16: sipush        200       //将200压入栈
      19: ireturn                 //紧接着执行返回操作将200作为方法的返回
      //jvm为执行try块中发生异常时的异常处理
      20: astore_2
      21: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      24: ldc           #3                  // String this is finally
      26: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      29: bipush        100
      31: istore_0
      32: sipush        200
      35: ireturn
    Exception table:
       from    to  target type
           3     5    20   any

  
{% endhighlight %}
finally块中有return语句，属于异常结束，JVM将不会执行try块中的return语句,因为finally中有返回操作，故程序直接执行finally块中的返回操作，所以程序返回200.

而在情况三：
{%  highlight java linenos %}
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
  字节码：
  static boolean trueOrFlase(boolean);
    Code:
       0: iload_0
       1: ifeq          9
       4: iconst_1
       5: istore_1
       6: goto          16
       9: goto          16
       
      12: astore_2
      13: goto          16
      16: iconst_0     //将0压人栈 jvm中boolean类型用0表示false,大于0的整数表示true
      17: ireturn      //直接返回栈顶元素0
    Exception table:
       from    to  target type
           0     6    12   any

{% endhighlight %}
finally块中存在有break语句，finally块是异常结束，JVM也不会执行try块中的return指令，而是直接跳出循环体执行”return false“对应的指令，所以程序返回false而不论参数为何。

情况四：
{%  highlight java linenos %}
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
          System.out.println("this is inner finally block");
          i = 2;
        }
        i = 3;
        return i;
      } finally
      {
        System.out.println("this is outter finally block");
        if (i == 3)
        {
          continue;
        }
      }
    }
  }
  字节码：
  static int test();
    Code:
       0: iconst_0
       1: istore_0
       2: iconst_1
       3: istore_0
       4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       7: ldc           #3                  // String this is inner finally block
       9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      12: iconst_2
      13: istore_0       //第一个finally块正常结束，会覆盖掉try块中存放在局部变量区中的值
      14: goto          30
      17: astore_1
      18: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      21: ldc           #3                  // String this is inner finally block
      23: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      26: iconst_2
      27: istore_0
      28: aload_1
      29: athrow
      30: iconst_3            //将3压入栈
      31: istore_0            //将3存放到局部变量区中的第0个位置，覆盖掉原来的值2
      32: iload_0
      33: istore_1            //将3存放到局部变量区中的第1个位置
      34: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      37: ldc           #5                  // String this is outter finally block
      39: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      42: iload_0
      43: iconst_3
      44: if_icmpne     50  //执行if判断
      47: goto          2   //if判断条件为ture，跳转到第二行指令处执行
      50: iload_1           //if判断条件为false，取出之前存放到局部变量区位置为1的变量
      51: ireturn           //执行返回
      52: astore_2
      53: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      56: ldc           #5                  // String this is outter finally block
      58: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      61: iload_0
      62: iconst_3
      63: if_icmpne     69
      66: goto          2
      69: aload_2
      70: athrow
    Exception table:
       from    to  target type
           2     4    17   any
           2    34    52   any

{% endhighlight %}
代码原指望循环3此之后退出循环体程序返回3，但是，finally块中有continue语句，fianlly异常结束，JVM也将不执行try块中的return指令，
而会继续循环体，在本例中将导致出现死循环。

情况五：

{%  highlight java linenos %}
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
        System.out.println("this is finally block");
        throw new RuntimeException("");
      }
    }
  }
  字节码：
   static boolean trueOrFlase(boolean);
    Code:
       0: iload_0
       1: ifeq          24
       4: iconst_1
       5: istore_1
       6: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       9: ldc           #6                  // String this is finally block
      11: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      14: new           #7                  // class java/lang/RuntimeException
      17: dup
      18: ldc           #8                  // String
      20: invokespecial #9                  // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
      23: athrow                   //抛出异常
      24: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      27: ldc           #6                  // String this is finally block
      29: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      32: new           #7                  // class java/lang/RuntimeException
      35: dup
      36: ldc           #8                  // String
      38: invokespecial #9                  // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
      41: athrow
      //jvm为执行try块中发生异常时的异常处理
      42: astore_2
      43: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
      46: ldc           #6                  // String this is finally block
      48: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      51: new           #7                  // class java/lang/RuntimeException
      54: dup
      55: ldc           #8                  // String
      57: invokespecial #9                  // Method java/lang/RuntimeException."<init>":(Ljava/lang/String;)V
      60: athrow
    Exception table:
       from    to  target type
           0     6    42   any

  
{% endhighlight %}

在finally块中抛出异常，finally块异常结束，JVM不会执行try语句块中的return指令，因finally中抛出异常，故程序会异常终止。



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

* [Inside the Java virtual machine]()
* [Java Language Specification]()


##关于作者
{%  highlight java linenos %}
public static void main
{% endhighlight %}
{%  highlight java linenos %}script
  var ihubo = {
    nickName  : "Peng LI",
    site : "http://pengligtf.github.io",
    email: "lipeng82@126.com"
  }
{% endhighlight %}


