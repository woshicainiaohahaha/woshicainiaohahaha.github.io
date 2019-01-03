---
layout: article
title: 从iOS中的SEL和IMP看runtime
article_header:
type: overlay
theme: dark
background_color: '#123'
background_image: false
---

**关于iOS中的SEL和IMP：**很多时候我们在阅读一些封装的比较好的框架的时候，会经常遇到这两个关键字。那么，它们到底是什么？有什么区别呢？我也是在查阅大量资料，来对这两个关键字做一个解释。转载请注明出处（[wienli](https://tree.wienli.club)）。

> **SEL：**类成员方法的指针，但是不同于C语言的函数指针，函数指针直接保存了方法的地址，但是SEL只是方法编号；
> - 这里有大神做了一个总结：对于C中被称为函数（function）和函数调用（function call）的地方，在OBJC中被叫做方法（method）和发送消息（send message）。试图调用未定义的方法会导致编译错误，而发送一条消息，即使没有任何类定义了响应该消息的方法，编译时也不会报错。从语义上理解这也是对的，发送一条消息不要求一定会有人作出响应。**但是：**如果执行到发送消息的代码时真的没有类作出响应的话，是会发生runtime error，为了避免这样的错误发生，我们在发送消息的时候先进行检测：
> ```
> if ([object respondsToSelector:@selector(method:)]) {
>     [object method:(id)obj];
> }
> ```
> 
> **IMP：**IMP是"implementation"的缩写，它是objective-C方法（method）实现代码块的地址，可以像C函数一样直接调用。通常情况下我们是通过[对象 方法]或objc_msgSend()的方式向对象发送消息，然后objective-C运行时（objective-C runtime）寻找匹配此消息的IMP，然后调用它。但有些时候我们希望获取到IMP进行直接调用。

**SEL和IMP的关系：**这里把SEL和IMP放在一起介绍是因为两个关键字很多人都容易弄混（包括我自己），他们大概都是指"方法"，此时可达鸭眉头一皱觉得事情并不简单，下面就解释一下二者的区别；

在介绍两者之间的关系之前，我们先来复习一下objective-C中的method**结构**。
> 在objective-C中，在类中对每一个方法有一个在运行时构建的数据结构，在objective-C 2.0中，此结构对用户不可见，但仍在内部存在。
> ```
> struct objc_method
> {
>     SEL method_name;
>     char * method_types;
>     IMP method_imp;
> };
>  
>  typedef objc_method Method;
> ```
>  
>  每个方法有三个属性
>  - 方法名：方法名为此方法的签名，有着相同函数名和参数名的方法；
>  - 方法类型： 方法类型描述了参数的类型；
>  - IMP： IMP即函数指针，此方法具体实现代码块的地址，可像普通C函数调用一样使用IMP;

由于Method的内部结构不可见，所以不能通过method->method_name的方式访问其内部属性，只能由objective-C运行时提供的函数获取。

下面对objective-C的runtime提供的方法做一个总结：
> 1. 获取方法编号（SEL）
> ```
> SEL methodID = @selector(method);
> ```
> 2. 编号（SEL）获取后怎么执行对应方法呢
> ```
> [self performSelector:methodID(SEL) withObject:nil];
> ```
> 3. 通过编号（SEL）获取方法名字
> ```
> NSString *methodName = NSStringFromSelector(methodID(SEL));
> ```
> 4. 函数指针（IMP）怎么获得和使用
> ```
> IMP methodPoint = [self methodForSelector:methodID(SEL)];
> methodPoint();
> ```

扯了这么多，可能有的人有些不耐烦了，不是说介绍关系吗？怎么弄那么多没用的。别急，上面的只是铺垫，为了让大家对objective-C里面的方法这个概念有个大概的认识，更好的理解下面的内容；

**关系**
> 每个继承于NSObject的类都能自动获得runtime的支持，这个是苹果爸爸在设计这门语言的时候自动支持的。在这样的类中，有一个isa指针，指向该类定义的数据结构体，这个结构体是编译器编译时为类创建的。在这个结构体中包括了指向起父类类定义的指针及Dispatch table。Dispatch table是一张SEL和IMP的对应表；也就是说SEL只是runtime内部的一个函数编号，最后还要通过Dispatch table表找到对应的IMP。IMP是一个函数指针，然后去执行这个方法。

**下面详细的介绍一下SEL消息机制的工作原理是什么？**

我们在之前有提到，一个类就像一个C结构。NSObject声明了一个成员变量：isa。由于NSObject是所有类的基类，所以所有的对象都会有一个isa的成员变量【公共继承】。而该isa变量指向该对象的类【类在objective-c中也是一个实体，由于存在objective-c运行环境，所有的类将有自己的存储空间。这里所说的isa正是指向这样一个类的空间。从而建立类和对象之间的对应关系。**类空间** 包含了该类定义的成员变量，以及实现方法，还包含了指向自己父类空间的指针】
![](http://blog.chinaunix.net/attachment/201302/19/28653839_1361282266vI7Q.png)

方法以**selector**作为索引，**selector**的数据类型是SEL。虽然SEL定义成**char**，我们可以暂时把它想象为int。每个方法的名字对应一个唯一的int值。比如：方法**addObject：**可能对应的SEL为12.寻找该方法时，我们使用的是selector，而不是名字**addObject：**

Objective-C数据结构中，存在一个name-selector的映射表：
![](http://blog.chinaunix.net/attachment/201302/19/28653839_1361282379yvAk.png)
也就是SEL-方法名

在编译的时候，只要有方法的调用，编译器都会通过selector来查找，所以（假设addObject：的SEL为12），我们在写入下面的代码时：
```
[object addObject:(id)item];
```
将会编译成：
```
objc_msgSend(object,12,item);
```

在这里，objc_msgSend()函数将会使用object的isa指针来找到object的类空间结构，并在类空间结构中查找selector的值为12所对应的方法。如果没有找到，那么将使用父类的isa指针找到父类的空间结构，然后根据值为12的selector所对应的方法。若没有找到将继续向父类传递事件，如果到了基类NSObjec还找不到，那么程序将会抛出异常。（**假如未声明该方法，llvm编译时就会报错；若声明该方法未实现，则会在runtime运行时报错**）

我们在这里可以很清楚的看到，这是一个动态的查找过程，类的结构可以在运行时改变。这样一来，我们就很容易来进行功能扩展。这就是objective-C的动态特性之一。

**关于父类和子类方法的调用：**

上面我们说到，每个类都有自己的类空间，并有isa指向。每个类空间都存在一张Dispatch table和指向父类的isa，Dispatch table是一张SEL和IMP的对应表。

对于名称相同的方法，他们都有相同的SEL，方法的名称不包括类名称，所以子类和父类中的同名方法拥有相同的SEL，但是他们的实现可以各不相同，因此他们在各自的Dispatch table中SEL所对应的IMP是各不相同的。IMP是一个函数指针，NSObject存在一个name-selector的映射表。有了这样的一个结构，我们就可以很轻松的实现多态。

例如：
```
[theClass foo:10];
```

是向theClass发送了foo:消息,那么首先在theClass的类结构的Dispatch table里找有没有对应的SEL，如果有的话，就表示theClass有响应该消息的方法，程序就跳到该方法的代码地址头（由IMP指定），开始执行。 如果在theClass的Dispatch table找不到对应的SEL,那么就会通过isa所指的结构体中包含的父类指针，到父类里面去寻找，如果到最后还是没有找到，就会出现runtime error.所以说，即使theClass以及它的父类都没有定义-(void) foo:(int)a方法，程序还是可以通过编译，但如果是用xcode的话，编译器会有警告，告知theClass可能无法响应该消息。不会报错的原因 是类的方法也可以在执行时刻创建！

**Objective-C中方法的重载和重写**

在解释这个概念之前我们先了解一下什么是重载和重写？
> **方法重载：**就是函数名相同，函数的参数列表不同（包括参数个数和参数类型），至于返回值类型可以相同也可以不同。重载既可以发生在同一个类的不同函数之间，也可以发生在父类子类的继承关系之间。当发生在父类和子类之间时要注意和重写区分开。
> 
> **方法重写：**发生在父类和子类之间，指的是子类不想继承使用父类的方法，通过重写同一个函数实现对父类中函数的覆盖。注意：重写函数必须和父类一模一样，包括函数名、参数个数、参数类型以及返回值类型。只是重写函数的实现，这里是区分重写和重载的关键。

了解完概念之后我们发现objective-c是支持重写的，那么重载支持吗？

我们可以写一段代码测试一下：
```
- (void)addObject:(id)object;
- (void)addObject:(NSString *)string;
- (void)addObject:(int)a withLocation:(int)loc;
- (int)addObject:(id)object;
```

我们会发现方法2和方法4会直接在编译阶段就报错，为什么objective-c不支持返回值类型不同和参数类型不同的重载呢？

>  我们在上面讲到过SEL消息机制的工作原理，每一个SEL都对应一个方法名字，也就是name-selector的映射表。这里我们假设addObject：的SEL为12，则可以得到12->addObject:。当我们通过objc_msgSend(object,12,item)时，runtime会发现多个值为12的SEL，那么此时根本无法判断到底要向对象发送什么消息。

**若类方法和实例方法相同时**

类方法和实例方法可以有相同的名字，而又有不同类型的参数和返回类型,因为它们不是处在同一张dispatch table中。

**分类的方法**

这里先介绍一下分类的概念：
>  **分类：**分类（category）是OC特有语法，它是表示一个指向分类的结构体的指针。原则上他只能增加方法，不能增加成员变量和属性。
>  
>  Category 是表示一个指向分类的结构体的指针，其定义如下：
>  ```
> typedef struct objc_category *Category;
> struct objc_category {
> char *category_name                          OBJC2_UNAVAILABLE; // 分类名
>  char *class_name                             OBJC2_UNAVAILABLE; // 分类所属的类名
>  struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE; // 实例方法列表
>  struct objc_method_list *class_methods       OBJC2_UNAVAILABLE; // 类方法列表
>  struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE; // 分类所实现的协议列表
> }
>  ```

这里可以清楚的看到它和类一样有自己的类空间，但是少了自己的isa和指向父类的isa和属性列表，当然苹果这样设计就是指明它只是一个可以扩展本类的工具。那么接下来我们暂时只针对其方法扩展做下了解。

分类中的对象方法依然是存储在类对象中的，同对象方法在同一个地方，那么调用步骤也同调用对象方法一样。如果是类方法的话，也同样是存储在元类对象中。

通过源码我们发现，分类的方法，协议，属性等好像确实是存放在categroy结构体里面的，那么他又是如何存储在类对象中的呢？这里需要对大量源码进行分析才能找到答案，本人水平有限贴一篇大神的文章[iOS分类底层原理](http://www.cocoachina.com/ios/20180514/23364.html)，感兴趣的小伙伴可以阅读源码。

**分类方法与类方法重复：**
我们上面讲到分类的方法确实存放在类对象中的，那么如果分类中存在一个和原类里面一个一样的方法的时候该怎么办？会不会像继承那样被分类所覆盖？

这里我们贴一段官方对此的说明，[Customizing Existing Classes](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html)：
> Avoid Category Method Name Clashes
> Because the methods declared in a category are added to an existing class, you need to be very careful about method names.

> If the name of a method declared in a category is the same as a method in > the original class, or a method in another category on the same class (or even a superclass), the behavior is undefined as to which method implementation is used at runtime. This is less likely to be an issue if you’re using categories with your own classes, but can cause problems when using categories to add methods to standard Cocoa or Cocoa Touch classes.
> 避免使用类别方法名称冲突
> 因为在类别中声明的方法被添加到现有类中，所以您需要非常小心方法名称。

> 如果在类别中声明的方法的名称与原始类中的方法相同，或者在同一个类（或甚至是超类）上的另一个类别中的方法相同，则行为未定义为在哪个方法实现中使用运行。如果您使用具有自己类的类别，则不太可能成为问题，但在使用类别向标准Cocoa或Cocoa Touch类添加方法时可能会出现问题。

也就是说：**重写之后，会导致在Runtime的时候，只有一个方法会被执行，而哪个会被执行是undefined。**并且，官方这里特别指出了，千万不要在分类里重写Cocoa或Cocoa Touch类的方法，这样会导致程序变为极不可控状态。

官方给出的做法是：为了避免未定义的行为，最佳做法是在框架类的类别中为方法名称添加前缀，就像您应该为自己的类的名称添加前缀一样。您可以选择使用用于类前缀的相同三个字母，但小写字母按照通常的方法名称约定，然后使用下划线，在方法名称的其余部分之前。对于NSSortDescriptor例如，你自己的类别可能是这样的：
```
@interface NSSortDescriptor（XYZAdditions）
+（id）xyz_sortDescriptorWithKey :( NSString *）键升序：（BOOL）升序;
@结束
```

这意味着您可以确保在运行时使用您的方法。删除歧义，因为您的代码现在看起来像这样：
```
NSSortDescriptor * descriptor = [NSSortDescriptor xyz_sortDescriptorWithKey：@“name”升序：YES];
```

**多个分类重写同一个方法：**

假如，我们在项目里引用了第三方库，而且不小心写了跟库内类别同样的类别方法，会发生什么事情呢？

我们根据源码可以很清楚的看出分类的对象方法依然存在类对象里，假如此时类对象存在两个相同的方法，则会根据编译先后顺序而进行覆盖，也就是后人栈的方法会覆盖之前的方法。

这里我们要区分上面的方法重载，不同的分类存在于不同的类空间里面，故不会出现多个SEL指向。

**结束语：**本文只是我对SEL和IMP的理解，为了确保文章的准确性参考了大量的别人的文章，然后加入了自己的理解。总之，站在巨人的肩膀上我们才能看的更远。文章只是具有参考性，请读者自行分辨，谢谢。
