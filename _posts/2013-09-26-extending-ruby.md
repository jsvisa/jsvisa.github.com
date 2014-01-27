---
layout: post
title: "如何用 C 语言扩展 Ruby"
description: "扩展 Ruby"
category: "Ruby Rails"
tags: [Ruby, cRuby]

---

#### 声明: 本文是 [Programming Ruby 手册中关于扩展Ruby](http://www.ruby-doc.org/docs/ProgrammingRuby/html/ext_ruby.html)
的中文翻译篇，水平有限，翻译不到位的地方还望大家多多指正.

使用 Ruby 写代码来扩展 Ruby 的功能是比较容易的，一旦你开始用 C
来写一些底层的代码，扩展 Ruby 的功能将会无止尽。我们选择用 C 来
扩展 Ruby，假定我们在为 *Sunset Diner and Grill* 公司构建一个在线的播放器，
它可以从硬盘，CD或者DVD 中读取音乐播放，我们希望通过 Ruby 来控制播放器。
播放器产家给我们提供了一个 C 头文件和一个二进制库文件, 我们的工作就是构建一个
让相应C文件调用的 Ruby 对象。

在 Ruby 和 C 能协同工作之前，我们需要在 C 层面看看 Ruby 世界是个什么样子的。

# C 中的 Ruby 对象
第一件要做的事情是如何用 C 中表示和访问 Ruby 中的数据结构。 Ruby
中所有的事物都是对象，并且所有的变量都指向对象，这意味着在 C 语言中所有 Ruby
变量的类型都是 `VALUE`, 要么是指向 Ruby
对象的指针，要么是一个即时变量(如：`Fixnum`).

这是 Ruby 以 C 语言代码实现面向对象的机制，一个 Ruby
对象是内存中包含了一张表的实例变量和类信息的指定结构;
类本身是包含一张表的关于这个类的方法的另一个对象(内存中分配的结构).

## VALUE 当作指针值
当 `VALUE` 当作指针时，它指向一个已定义的 Ruby 对象结构---不能把 `VALUE`
指向不确定的对象结构。内建的 Ruby 类结构定义在 *ruby.h* 文件中，以 `RClassname`
命名，例如**RArray**, **RString**.

有很多种方法可以查看一个指定的 `VALUE` 所使用的数据类型，宏 `TYPE(obj)`
可以返回一个表示 C 中给定对象类型的常量，例如 `T_OBJECT, T_STRING`...
内建类的常量定义在 *ruby.h* 文件中.
请注意，我们在这里指的类型是一个实现细节---它和作为一个对象的类是不一样的。

可以使用宏 `Check_Type` 来确保一个 value 指针指向一个特定的结构，如果 value 没有指向指定的对象，这个宏发出一个
**TypeError**异常。

    Check_Type(VALUE value, int type)
对于即时变量 `Fixnum` 和 `nil` 有几个快速判断的宏：

    FIXNUM_P(value) -> non-zero if value is a Fixnum
    NIL_P(value)    -> non-zero if value is nil
    RTEST(value)    -> non-zero if value is neither nil nor false
再次提醒下，这里提到的类型代表了 C 结构中的特殊内建类型,
这和一个对象的类是完全另一回事。内建类的类对象在 C 语言中
以命名为 `rb_cClassname` 的全局变量的形式保存(例如 `rb_cObject`), 模块以 `rb_mModulename`命名。

直接弄乱这些结构中的数据是不明智的，然而你可以去查看下，但不要去触碰这些结构，除非你想要调试它。
正常情况下，你只能通过提供的 C 函数去操作 Ruby 数据(等下我们再来讨论这个话题).

然而，为了效率起见，你需要进入这些结构里面去挖掘数据，为了引用 C 结构中的成员，需要构造 `VALUE`
到合适的结构类型。*ruby.h* 中定义了一系列宏用来执行这个任务，允许你方便引用结构成员。这些宏以 `RCLASSNAME` 为名，
如 `RSTRING`, `RARRAY`.

    VALUE str, arr;
    RSTRING(str)->len -> length of the Ruby string
    RSTRING(str)->ptr -> pointer to string storage
    RARRAY(arr)->len  -> length of the Ruby array
    RARRAY(arr)->capa -> capacity of the Ruby array
    RARRAY(arr)->ptr  -> pointer to array storage

##  VALUE 当作即时对象
上面说到了，限时变量不是指针，`Fixnum, Symbol, true, false`, 和 `nil` 都是直接存储在 `VALUE` 中。
`Fixnum` 以31位数(或者63位在更宽的 CPU 架构上)，将原始数向左移动一位，然后将最低有效位(位0)为“1”。
当 `VALUE` 作为指向一个特定结构的指针使用时，最低有效位为"0"; 对于其它即时变量, 所有的最低有效位为"0",
通过一个简单的位测试，可以分辨是否是一个 `Fixnum`. 表17.1 中列出了数字和其它标准的数据结构的有用的转换宏。

其它的即时变量(`true`, `false`, `nil`)以 C 中的常量代替(`Qtrue`, `Qfalse`, `nil`), 可以直接将 `VALUE` 和这些常量
测试，或者通过转换宏来实现。

## 在 C 中书写 Ruby
Ruby的乐趣之一是，你几乎可以直接用 C 语言编写 Ruby 程序.
也就是说你可以使用相同的逻辑和稍微有点不一样的语法, 以适应 C
语言.下面是一个用 Ruby 写的小的简单的测试类：

    class Test
      def initialize
        @arr = Array.new
      end
      def add(anObject)
        @arr.push(anObject)
      end
    end
用 C 语言写的相同的代码是:

    #include "ruby.h"

    static VALUE t_init(VALUE self)
    {
      VALUE arr;

      arr = rb_ary_new();
      rb_iv_set(self, "@arr", arr);
      return self;
    }

    static VALUE t_add(VALUE self, VALUE anObject)
    {
      VALUE arr;

      arr = rb_iv_get(self, "@arr");
      rb_ary_push(arr, anObject);
      return arr;
    }

    VALUE cTest;

    void Init_Test() {
      cTest = rb_define_class("Test", rb_cObject);
      rb_define_method(cTest, "initialize", t_init, 0);
      rb_define_method(cTest, "add", t_add, 1);
    }
让我们来好好研究下这个例子，这里用到了这一章节中很多重要知识。首先我们需要包含头文件
*ruby.h* 来获取一些必要的定义。

现在看下最后一个函数 `Init_Test()`, 每一个类或模块定义了一个 C 全局函数，命名为
`Init_`*Name*。当编译器第一次载入扩展 *Name*
或者在静态链接扩展的时候会调用这个函数。这个函数是用来初始化扩展，并将其载入到
Ruby 环境中。在这个例子中，我们定义了一个叫做 **Test** 的类，这人
**Object**(符号为: `rb_cObject`，查看 *ruby.h* 作进一步了解)
的一个子类。

接下来我们设置了 `initialize`, `add` 两个实例方法。 `rb_define_method()`
用来在 Ruby 实例方法和对应的 C 函数之间做一个绑定，因此调用 Ruby 的 `add`
方法会调用到 C 的 `t_add` 函数(有一个参数)。

相似地，当这个类中 `new` 方法被调用到的时候，Ruby 会构成一个基本的对象，然后调用
`initialize` 方法(没有参数)，在这里会调用到 C 的 `t_init()` 方法。

现在回过来看 `initialize` 方法的定义，前面说调用这个方法时不需要 Ruby
参数，但是在这里确实看到了参数。事实是对于任意的 Ruby
参数，每一个方法都会传递一个最初的 `VALUE`
参数，这个参数中包含了这个方法的调用者(和 Ruby 的 `self` 相同)。

在 `initialize` 方法中的第一件事是创建一个 Ruby 数组，并且将实例变量 `@arr`
指向它。正如你所期待的那样，假定你在写 Ruby
源代码，如果一个实例变量不存在的话就创建一个。

最后，`t_add()` 函数从当前对象中得到实例变量 `@arr`，调用 `Array#push`
方法将传递进来的值压入这个数组中，当以这种方法访问 Ruby
中的实例变量时，**@**前缀是强制需要的，否则这个变量创建了，但是无法在 Ruby
中被引用到。

尽管额外的、繁重的 C 语法规定，你还是可以用 Ruby
来书写，你可以用你熟知的并且喜爱的方法来操纵对象，以写出紧凑的高效的代码。

**WARNING**: Ruby 中调用的每一个 C 函数必须要返回一个 `VALUE` , 即使只是
`Qnil`，否则会发生**Core dump** 或者**GFP** 错误。

要想在 Ruby 代码中调用 C 版本的代码, 在运行时可通过动态的 `require`-ing
来实现。

    require "code/ext/Test"
    t = Test.new
    t.add("Bill Chase")

###C 和 Ruby 数据转换函数和宏

C 数据类型======>       |   Ruby 对象
----------------------- | -------------------
INT2NUM(int)            | -> Fixnum or Bignum
INT2FIX(int)            | -> Fixnum (faster)
INT2NUM(long or int)    | -> Fixnum or Bignum
INT2FIX(long or int)    | -> Fixnum (faster)
CHR2FIX(char)           | -> Fixnum
rb_str_new2(char *)     | -> String
rb_float_new(double)    | -> Float


Ruby 对象 | ======> | C 数据结构
-------- | --------------- | -------------------
int |  NUM2INT(Numeric) |     (Includes type check)
int |  FIX2INT(Fixnum) |  (Faster)
unsigned int |     NUM2UINT(Numeric) |    (Includes type check)
unsigned int |     FIX2UINT(Fixnum) |     (Includes type check)
long |  NUM2LONG(Numeric) |    (Includes type check)
long |     FIX2LONG(Fixnum) |     (Faster)
unsigned long |    NUM2ULONG(Numeric) |   (Includes type check)
char   |  NUM2CHR(Numeric or String) |   (Includes type check)
char * |  STR2CSTR(String) |
char * | rb_str2cstr(String, int *length) |     Returns length as well
double | NUM2DBL(Numeric) |

### C 中使用 Ruby 表达式
当在 C 代码中间，但同时想要使用 Ruby 代码来替代一系列的 Ruby C 代码，可使用 C 版本的 `eval` 方法书写 Ruby 代码。
假设你要清除一系列的对象中的一个标记位：`rb_eval_string("anObject.each{|x| x.clearFlag }");`.
如果你要调用一个特殊的方法，可使用如下的使用方法(比使用 `eval` 要轻量级)：

    rb_funcall(receiver, method_id, argc, ...)

## 在 C 和 Ruby 中共享数据
我们已经谈论了足够多的基础知识了，现在回到我们的音乐播放器中来--包含 Ruby 代码的
C 接口，能在这两个世界中共享数据和行为。

###直接共享数据
你可以指定 C 版本的变量，再指定一些 Ruby
版本的变量，并使得他们保持同步，[这违反了 **DRY--Don't repeat yourself** 原则]。
有一种简单的方法可以使得两者之间共享数据: 在 C
层面上创建一个 Ruby 对象，并把他的地址绑定到 Ruby
全局变量中。在这个例子中，**$**前缀是可选的，但是它帮助指明了这是个全局变量。

    VALUE hardware_list;
    hardware_list = rb_ary_new();
    rb_define_variable("$hardware", &hardware_list);
    ...
    rb_ary_push(hardware_list, rb_str_new2("DVD"));
    rb_ary_push(hardware_list, rb_str_new2("CDPlayer1"));
    rb_ary_push(hardware_list, rb_str_new2("CDPlayer2"));

Ruby 层面上可以访问 C 变量 `hardware_list` 作为 `$hardware`

    $hardware   »   ["DVD", "CDPlayer1", "CDPlayer2"]
可以创建一个钩子变量，当这个变量被访问到的时候，会调用一个指定的函数；
虚变量--只用来调用钩子变量，没有实际变量被关联到。查看 **API** 作进一步了解。
如果在 C 层面上创建了一个 Ruby 对象，并且将它存储在一个 C
的全局变量中，但是没有将之导出到 Ruby
层面上，那么至少要告诉垃圾回收器这个变量，否则，^_^内存泄漏。。。

    VALUE obj;
    obj = rb_ary_new();
    rb_global_variable(obj);

### 封装 C 结构
现在开始真正有趣的事情，我们得到了产家用来控制音乐播放器的为库文件，我们要将之
Ruby 化，产家的头文件如下：

    typedef struct _cdjb {
      int statusf;
      int request;
      void *data;
      char pending;
      int unit_id;
      void *stats;
    } CDJukebox;

    // Allocate a new CDPlayer structure and bring it online
    CDJukebox *CDPlayerNew(int unit_id);

    // Deallocate when done (and take offline)
    void CDPlayerDispose(CDJukebox *rec);

    // Seek to a disc, track and notify progress
    void CDPlayerSeek(CDJukebox *rec,
                      int disc,
                      int track,
                      void (*done)(CDJukebox *rec, int percent));
    // ... others...
    // Report a statistic
    double CDPlayerAvgSeekTime(CDJukebox *rec);

产家有他自己的行为，然而产家或许不允许这样子，代码以面向对象来书写，我们并不确切知道结构体`CDJukebox`
中各字段的含义，不过这不要紧的，我们就将它当作黑盒子来看待，产家自己知道如何对待这些代码，
我们只要将它带出来就行了。

不管什么时候，如果你想将 C 结构体当作 Ruby 对象来处理的时候都需要将结构体封装成
一个特殊的内部的 Ruby 类
`Data`(类型为**T_DATA**)。有两个宏可用来完成这个封装，还有一个宏可将类恢复加结构体。

    VALUE Data_Wrap_Struct(VALUE class, void (*mark)(), void (*free)(), void *ptr);
    VALUE Data_Make_Struct(VALUE class, c-type, void (*mark)(), void (*free)(), c-type *")
    Data_Get_Struct(VALUE obj,c-type,c-type *")

`Data_Wrap_Struct` 所创建的对象是一个普通的 Ruby 对象，只是这个对象有一个额外的 Ruby 中不能访问到的 C 数据类型。
这个 C 数据类型和这个对象所包含的任意实例变量是分开的。既然是分开的，
那么垃圾回收器在销毁这个对象时该如何释放这个变量？当你要释放一些资源的时候(比如关闭一些文件，释放一些锁或
IPC 结构，等等)又该如何做？

为了参与到 Ruby 的 **mark-and-sweep**
垃圾回收程序中来，你需要自己定义一个固定的流程来释放你的结构，
有可能有另一个固定流程用来标识对这个结构的引用转移到
另一个结构中。这两个流程都携带着一个 `void` 指针，对结构体的一个引用。
在 **mark** 阶段，标识流程会被垃圾回收器调用到，如果结构体引用了其它 Ruby
对象，那么标识流程需要调用 `rb_gc_mark(value)`
来确认这些对象；如果结构体没有引用到其它的 Ruby
对象，这时传递一个空函数指针即可。

当对象需要被拆解掉的时候，垃圾回收器会调用释放函数; 对于你自己申请的内存(`Data_Wrap_Struct()`
申请的内存)需要你主动传递释放函数，即使是像标准 C 的 `free()`
函数；对于一个复杂的数据结构，相应的释放函数需要遍历所有申请到的内存并逐一释放。

首先来个简单的例子，没有复杂的处理机制。这里给出结构体定义：

    typedef struct mp3info {
      char *title;
      char *artist;
      int  genre;
    } MP3Info;
我们可以创建一个结构，并把它封装成 Ruby 对象[这个例子中的 `Mp3Info` 结构使用了 `char *` 指针，
但在实际使用过程中我们将这两个字段初始化成两个静态字符串，如果使用动态申请的话就需要释放这两个动态
申请过来的指针, 否则就不需要].

    MP3Info *p;
    VALUE info;

    p = ALLOC(MP3Info);
    p->artist = "Maynard Ferguson";
    p->title = "Chameleon";
    ...
    info = Data_Wrap_Struct(cTest, 0, free, p);
为了方便起见，我们还需要再做几件事情：提供一个 `initialize` 方法，一个 C
构造函数。在编写 Ruby 代码的时候，`new` 方法可分配并初始化一个对象, 在 C
扩展中，相对应的调用是 `Data_Make_Struct()`,
虽然为这个对象分配了内存，但实际上并没有自动调用 `initialize`
方法，如果执行类似如下的代码片段：

    info = Data_Make_Struct(cTest, MP3Info, 0, free, one);
    rb_obj_call_init(info, argc, argv);
这有利用 Ruby 中的子类重写，或者在我们的类中扩展基础 `initialize` ，
在 `initialize` 内部允许更改存在的数据指针
(但这必不是推荐做法)，这个数据指针可以直接通过宏 `DATA_PTR(obj)` 来访问到。

最后，我们需要定义一个 C 构造函数，一个全局可用的 C 函数可通过一种方便的方式创建对象，
可以在自己的代码中或者外围扩展模块中使用这个函数，全部的内建函数支持这种类型的函数思想：
`rb_str_new, rb_ary_new`。我们可以定义自己的：

    VALUE mp3_info_new() {
      VALUE info;
      MP3Info *one;
      info = Data_Make_Struct(cTest, MP3Info, 0, free, one);
      ...
      rb_obj_call_init(info, 0, 0);
      return info;
    }

## 一个完整例子
现在可以给定一个完整的例子了，给定产家的头文件，接下来开始我们的代码：

    #include "ruby.h"
    #include "cdjukebox.h"

    VALUE cCDPlayer;

    static void cd_free(void *p) {
      CDPlayerDispose(p);
    }

    static void progress(CDJukebox *rec, int percent)
    {
      if (rb_block_given_p()) {
        percent = (percent > 100) ? 100 : (percent < 0) ? 0 : percent;
        rb_yield(INT2FIX(percent));
      }
    }

    static VALUE
    cd_seek(VALUE self, VALUE disc, VALUE track)
    {
      CDJukebox *ptr;
      Data_Get_Struct(self, CDJukebox, ptr);

      CDPlayerSeek(ptr,
                   NUM2INT(disc),
                   NUM2INT(track),
                   progress);
      return Qnil;
    }

    static VALUE
    cd_seekTime(VALUE self)
    {
      double tm;
      CDJukebox *ptr;
      Data_Get_Struct(self, CDJukebox, ptr);
      tm = CDPlayerAvgSeekTime(ptr);
      return rb_float_new(tm);
    }

    static VALUE
    cd_unit(VALUE self)
    {
      return rb_iv_get(self, "@unit");
    }

    static VALUE
    cd_init(VALUE self, VALUE unit)
    {
      rb_iv_set(self, "@unit", unit);
      return self;
    }

    VALUE cd_new(VALUE class, VALUE unit)
    {
      VALUE argv[1];
      CDJukebox *ptr = CDPlayerNew(NUM2INT(unit));
      VALUE tdata = Data_Wrap_Struct(class, 0, cd_free, ptr);
      argv[0] = unit;
      rb_obj_call_init(tdata, 1, argv);
      return tdata;
    }

    void Init_CDJukebox() {
      cCDPlayer = rb_define_class("CDPlayer", rb_cObject);
      rb_define_singleton_method(cCDPlayer, "new", cd_new, 1);
      rb_define_method(cCDPlayer, "initialize", cd_init, 1);
      rb_define_method(cCDPlayer, "seek", cd_seek, 2);
      rb_define_method(cCDPlayer, "seekTime", cd_seekTime, 0);
      rb_define_method(cCDPlayer, "unit", cd_unit, 0);
    }

现在我们可以用 Ruby 以简单的、面向对象的方式控制音乐播放器：

    require "code/ext/CDJukebox"
    p = CDPlayer.new(1)
    puts "Unit is #{p.unit}"
    p.seek(3, 16) {|x| puts "#{x}% done" }
    puts "Avg. time was #{p.seekTime} seconds"
输出：

    Unit is 1
    26% done
    79% done
    100% done
    Avg. time was 1.2 seconds

这个例子总结了我们前面讲的大部分知识，还给出了个额外的有用功能。
产家的库文件提供了一个回调函数---当硬件转向下一张光碟的时候被调用到。
在这里我们给 `seek` 传递了一个块代码作为参数，在 `progress` 函数中，
我们查看当前上下中是否有有迭代器，如果存在的话以当前完成百分比来运行它。

## 内存分配
有些时候，我们必不只为了 Ruby 对象才分配内存，比如你需要为 *Boolm* 过滤器分配一个超大的位图，
或者一张图片，或者是 Ruby 用不到的一些小结构的集合。

为了配合垃圾回收器能正确的工作，你需要使用以下一些内存分配函数，这些函数比 `malloc()` 完成更多的工作，
比如 `ALLOC_N()` 在分配内存的时, 如果当前不能分配所需要大小的内存，会告知垃圾回收器去释放一些空间;
如果未能发出请求或者请求无效，会产生一个 `NoMemError` 错误码。

    type * ALLOC_N(c-type, n)
         Allocates n c-type objects, where c-type is the literal name of the C type,
         not a variable of that type.
    type * ALLOC(c-type")
         Allocates a c-type and casts the result to a pointer of that type.

    REALLOC_N(var, c-type, n)
         Reallocates n c-types and assigns the result to var, a pointer to a c-type.
    type * ALLOCA_N(c-type, n)
         Allocates memory for n objects of c-type on the stack---this memory will be
         automatically freed when the function that invokes ALLOCA_N returns.

## 创建扩展模块
写好扩展模块的代码之后，需要编译代码，那样 Ruby 才能运行。我们可以编译成运行时加载的动态库，
或者是加载到解释器 `main` 中的静态链接库。这个处理流程是一样的：

* 在一个目录中创建 C 源代码
* 创建 *extconf.rb* 文件
* 运行 *extconf.rb* 来生成 **Makefile** 文件
* 运行 `make`
* 运行 `make install`

### 利用 *extconf.rb* 创建 *Makefile* 文件
在 *extconf.rb* 文件中，你需要写一个简单的程序来指定有用记的系统上哪些功能是可用的，
并且需要将这些功能安装到哪里。执行 *extconf.rb* 来生成定制化的 **Makefile** 文件，
这个文件同时兼容当前系统和你的应用程序。

最简单的 *extconf.rb* 配置只有两行，且对于大多数的扩展模块来说这是够用了:

```
require 'mkmf'
create_makefile("Test")
```
第一行加载了 *mkmf* 模块，这个模块中包含了我们所有用到的指令。第二行为 **"Test"**
模块创建了一个 *Makefile* 文件(**"Test"** 是扩展模块名)。**"Test"** 模块会根据当前目录下所有的 C 文件生成。

在一个只包含一个源文件 *main.c* 的目录下执行 *extconf.rb* 程序，生成一个可生成扩展模块的 *Makefile*。
在我的系统上生成的代码如下：

```
gcc -fPIC -I/usr/local/lib/ruby/1.6/i686-linux -g -O2  \
  -c main.c -o main.o
gcc -shared -o Test.so main.o -lc
```
这个编译结果是 **Test.so**，在 Ruby 应用程序中使用 `require` 可加载这个动态库。
看到了吗，这里 `mkmf` 定位到当前平台的库文件，并且自动使用编译器指定的选项。

虽然这个基本程序对于大多数应用程序来说是够用了，但如果你的程序中使用的库文件和头文件不在编译
环境默认路径，或者你想根据所使用的库文件和函数有选择地编译，那么你就要学习以下更多知识了。

一种普通的情况是，你的库文件和头文件放在了一个非标准的目录中。这个在 *extconf.rb* 中只需要两步就能完成：
1. *extconf.rb* 文件中需要包含一个或者多个 `dir_config` 指令, 这个指定了一系列目录的标志；
2. 运行 *extconf.rb*, 告知 `mkmf` 当前系统上相应的代码路径；

如果 *extconf.rb* 中包含了 `dir_config(name)`, 在命令行上需要给相对应的目录一个指定的位置:

    --with-name-include=directory
    *添加include目录到编译命令中
    --with-name-lib=directory
    *添加lib目录到链接命令中
通常情况下，你的 *include* 和 *librara* 文件都是某个目录的子目录，并且他们分别命名为 *include* 和 *lib*,
那么可以简写为如下形式：

    --with-name-dir=directory
    *include 和 lib 目录是同一个目录的子目录，同时添加
在运行 *extconf.rb* 时需要指定 `--with` 选项，当 Ruby 是为你自己的机器编译时也可以指定 `--with` 选项,
也就是说你可以找到 Ruby 自己使用的库文件。

这里我们使用的 *extconf.rb* 可以包含以下内容：

```
require 'mkmf'
dir_config('cdjukebox')
# .. more stuff
create_makefile("CDJukeBox")
```
执行 *extcon.rb* 如下：

    % ruby extconf.rb --with-cdjukebox-dir=/usr/local/cdjb
生成的 *Makefile* 文件会假定所使用的库文件在 */usr/local/cdjb/lib*, 头文件在 */usr/local/cdjb/include* 中。

`dir_config()` 添加了搜索头文件和库文件的位置，但是这并没有将库文件链接到应用程序中来，可以使用 `have_library`
或者 `find_library` 指令来完成这个功能。

`have_library` 在一个给定的库文件中搜索一个给定的入口点。如果找到了这个入口点，
将这个库添加到待链接到应用程序库链表中; `find_library` 的作用是相似的，只是它可以指定一个搜索目录。

```
require 'mkmf'
dir_config('cdjukebox')
have_library('cdjb', 'CDPlayerNew')
create_makefile("CDJukeBox")
```
有一些平台上，一个流行的库文件可能在好几个不同的地方。举个例子来说，
X Windows 系统是一个臭名昭著的在不同系统上有不同的文件目录。
`find_library` 会搜索一列指定的目录，以找到正确的那个(这点和 `have_library` 是不同的，
后者只使用搜索指定的配置)。创建一个在 X Windoes 系统下使用jpeg 库的 *Makefile*
文件所使用的 *extconf.rb* 文件如下：

```
require 'mkmf'

if have_library("jpeg", "jpeg_mem_init") and
   find_library("X11", "XOpenDisplay", "/usr/X11/lib",
                "/usr/X11R6/lib", "/usr/openwin/lib")
then
    create_makefile("XThing")
else
    puts "No X/JPEG support available"
end
```
有上面这个例子中，我们还加入了一些额外的功能，如果执行失败的话，所有的 `mkmf` 指令将返回 `false`。
这也意味着我们所写的 *extconf.rb* 只有在所有依赖资准备好之后才能生成 *Makefile* 文件。
由于 Ruby 版本的原因，这只会编译出系统所指定的扩展模块。

你当然希望你的代码能在目标平台上编译出扩展模块来，比如用户安装了一个高性能的音乐解码器，
可通过搜索头文件来检查是否安装了：

```
require 'mkmf'
dir_config('cdjukebox')
have_library('cdjb', 'CDPlayerNew')
have_header('hp_mp3.h')
create_makefile("CDJukeBox")
```
我们也要检查下目标机器上是否有各个库都可用的特殊函数，比如说 `setpriority` 函数非常有用，但并不都是可用的，
可通过如下方法检查：

```
require 'mkmf'
dir_config('cdjukebox')
have_func('setpriority')
create_makefile("CDJukeBox")
```

函数 `have_func()` 和 `have_header()` 如果找到了目标都会定义预编译常量，通过将目标名转换成大写字母，
并加入前导的 **HAVE_**， 你的 C 代码利用了这个优点，看起来像是：

```
#if defined(HAVE_HP_MP3_H)
#  include <hp_mp3.h>
#endif

#if defined(HAVE_SETPRIORITY)
  err = setpriority(PRIOR_PROCESS, 0, -10)
#endif
```
如果你还有一些特殊的需求是 `mkmf` 命令中不支持的，可直接通过添加到全局变量 `$CFLAGS` 和 `$LFLAGS` 中，
这两个变量是直接传递给编译器和链接器的。

### 静态链接
如果你的系统不支持动态，那就只能将扩展模块静态链接到 Ruby 自身上去了，
首先编辑文件 *ext/Setup*, 将你自己的目录添加到模块链表上，然后重新编译 Ruby. *Setup* 中指定的扩展模块
可以静态链接到 Ruby 可执行程序中。如果你不想使用所有的动态库，并且静态链接所有的扩展模块，在 *ext/Setup*
中添加下面一个选项：

    option nodynamic

## 嵌入 Ruby 编译器
可通过添加 C 代码来扩展 Ruby, 你也可以将这个问题反过来，将 Ruby 嵌入到你自己的程序中，示例如下：

```
#include "ruby.h"

main() {
  /* ... our own application stuff ... */
  ruby_init();
  ruby_script("embedded");
  rb_load_file("start.rb");
  while (1) {
    if (need_to_do_ruby) {
      ruby_run();
    }
    /* ... run our app stuff */
  }
}
```
需要调用 `ruby_init()` 来初始化 Ruby 解释器，但在一些平台上或许需要做特殊的几个步骤。

```
#if defined(NT)
  NtInitialize(&argc, &argv);
#endif
#if defined(__MACOS__) && defined(__MWERKS__)
  argc = ccommand(&argv);
#endif
```
一起看下 *main.c* 中用到的几个 Ruby 相关的函数：

```
void ruby_init()
     Sets up and initializes the interpreter.
     This function should be called before any other Ruby-related functions.
void ruby_options(int argc, char **argv)
     Gives the Ruby interpreter the command-line options.
void ruby_script(char *name)
     Sets the name of the Ruby script (and $0) to name.
void rb_load_file(char *file)
     Loads the given file into the interpreter.
void ruby_run()
     Runs the interpreter.
```
需要对异常处理特别关心一下，在顶级所调用的任何 Ruby 调用需要能够捕捉到异常并将异常完整地处理掉。
`rb_protect()`, `rb_rescue()` 和相关一些函数请自行查阅手册。

嵌入式 Ruby 的另一个程序，也就是 **eruby** 请查阅另外的资料。

## 将 Ruby 桥接到其它语言中
到目前为止，我们已经谈论了用 C 语言来扩展 Ruby, 然而你可以书写 Ruby 扩展模块用任意一种语言，只要你能将
这两种语言用 C 语言桥接起来。几乎所有的事情都是可能的，例如 Ruby 和 C++结合，Ruby 和 Java 结合，等等。
但是你也可以不通过 C 语言来完成这整个过程，比如你使用了中间件来完成桥接到其它语言上。

## Ruby C 语言 API 接口
最后，但不是最终，这里提供了一些有用的 C 级别的函数，用于书写扩展模块。
一些函数需要 **ID:**， 你可以通过 `rb_intern` 指定 **ID** 为字符串，
或者可以使用 `rb_id2name` 从 **ID** 中重建名字。

这些具体的接口可查看 *ruby.h* 或者 *intern.h* 作详细审核，这里不详述了。

