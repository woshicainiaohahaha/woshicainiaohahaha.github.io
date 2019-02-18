---
layout: article
title: iOS开发之从__NSArray0、__NSArrayI、__NSArrayM和__NSPlaceholderArray里面审查类簇
article_header:
type: overlay
theme: dark
background_color: '#123'
background_image: false
---

我们经常在写代码的时候遇到下面的崩溃信息：

> -[__NSArrayI rectValue]: unrecognized selector sent to instance 0x608000252060。

大部分情况下是因为对象被提前release了，在你心里不希望它release的情况下，指针还在，对象已经不在了。当然，出现这种情况的原因有很多种，解决办法要根据具体情况来进行。

这里暂时不提解决方案，但是这个__NSArrayI是个什么鬼？转载请注明出处（[wienli](https://tree.wienli.club)）。

### Class Clusters

首先说一下Class Clusters（类簇）是抽象工厂模式下在iOS下的一种实现，iOS中如NSString、NSSArray、NSDictionary以及NSNumber都运作在这一模式下。在我们完全不知情的情况下，隐藏了很多具体实现的类，只是暴露简单的接口。

### 类簇的基本概念和实现思路

为了举例说明类簇的结构体和好处，**我们先想想如何构建一个类的结构体系，然后用这个类指定一个对象存储不同数据类型的变量（如：char、int、float、double）？**因为不同数据类型的变量在使用的时候可以互相转换类型或用字符串标识，所以我们可以用一个简单的类来管理他们。可是无论怎样，它们的存储形式都是不同的，所以使用一个类来管理他们的效率很低下。考虑到这个问题，我们设计了类Number，结构如下图所示：
![](https://upload-images.jianshu.io/upload_images/2280423-44d529203e8bbbcb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/292/format/webp)

使用了类簇以后，我们只能看到公开的类Number，那么该如何创建（alloc）这些子类的实例对象呢？解决思路是利用抽象类Number来处理这些实例对象。

- **创建实例**
	在类簇设计模式中抽象类Number必须声明一个方法可以创建它的私有子类的实例对象。这个抽象类Number的主要职责就是当你在调用方法的时候，负责分发不同子类的创建实例的方法来帮助你返回正确的实例对象。（**注：创建对象的类我们没有办法手动选择**）
	在**Foundation**框架中，一般使用类方法、或者alloc和init发送消息创建一个实例对象。例如NSNumber使用如下方法创建对象：

```
NSNumber *aChar = [NSNumber numberWithChar:’a’];
NSNumber *anInt = [NSNumber numberWithInt:1];
NSNumber *aFloat = [NSNumber numberWithFloat:1.0];
NSNumber *aDouble = [NSNumber numberWithDouble:1.0];
```

使用上面的类方法创建对象，可以不用release对象，因为上面的类方法本质是便利构造器，在创建对象的时候已经加入了autorelease机制。当然许多类仍然保留了最基本的alloc和init来创建对象，以防止你需要管理他们的dealloc实现。（注：autorelease在将来的某个时刻自动释放池清理的时候才释放。dealloc在RC=0的时候马上调用进行清理）。

回到正题，使用上面的创建方法返回的对象aChar、anInt、aFloat、aDouble可能由不同的私有子类创建。即使每一个对象的类从属关系被隐藏了，但是它的接口是公开的，我们可以通过抽象类NSNumber声明的接口来访问。当然这种做法是极其不严谨的，某种意义上也是不正确的，因为NSNumber返回的对象并不是一个NSNumber的成员对象，而是一个被隐藏了的我们不知道的成员对象（例如：aChar的成员关系从属于私有类char，而不是number）。尽管，这种关系极其不严谨，我们却可以很方便的使用抽象类NSNumber接口中声明的方法来实例化这些对象以及操作它们的实例方法。

**拥有多个抽象公共类的类簇**

在上面的例子，使用的是一个公共的抽象方法父类来管理很多个私有类，公共类声明接口来供外部进行调用。但是在Foundation框架中也有很多使用了两个或者两个以上的公共抽象父类来管理很多个私有子类的例子，如：

|类簇|公共抽象父类|
|----|----|----|
|NSData|NSData、NSMutableData|
|NSArray|NSArray、NSMutableArray|
|NSDictionary|NSDictionary、NSMutableDictionary|
|NSString|NSString、NSMutableString|

**由此可知：**

在Cocoa中，许多类实际上是以类簇的方式实现的，即他们是一群隐藏在通用接口之下的与实现相关的类。例如，创建NSString对象时，实际上获得的可能是NSLiteralString、NSCFString、NSSimpleString、NSBallOfString或者其他未写入文档的与实现相关的对象。所以，请不要尝试去创建NSString、NSArray、NSDictionary的子类。如果必须添加或者修改某个方法，可以使用类目（类别）的方式。

注：对于类簇使用isMemberOfClass和isKindOfClass是不允许的，因为类簇是由抽象公共类管理的一组私有类，抽象公共类并不是真正的实例的父类，类簇中真正的类从属关系被隐藏了，所以使用isMemberOfClass和isKindOfClass结果可能不准确。

### NSArray的类簇

在《effective objective-c 2.0编写高质量iOS与OS X代码的52个有效方法》中这样写道：系统框架中有许多类簇，大部分collection类都是类簇。例如NSArray与其可变版本NSMutableArray。这样看起来实际上有两个抽象基类，一个用于不可变数组，一个用于可变数组。尽管具备公共接口的类有两个，担任然可以合起来算一个类簇。不可变的类定义了对所有数组都通用的方法，而可变类则定义了那些只适用于可变数组的方法。两个类共同属于同一个类簇，这意味着二者在实现各自类型数组时可以共用实现代码，此外还能把可变数组复制成不可变数组，反之亦然。

在使用NSArray的alloc方法来获取实例的时候，该方法首先会分配一个属于某类的实例，此实例充当“占位数组”。该数组稍后会转为另一个类的实例，而那个类则是NSArray的实体子类。

下面我们可以用一个具体的例子来说明，比如：

```
NSArray *placeholder = [NSArray alloc];
NSArray *arr1 = [placeholder init];
NSArray *arr2 = [placeholder initWithObjects:@0,nil];
NSArray *arr3 = [placeholder initWithObjects:@0, @1, nil];
NSArray *arr4 = [placeholder initWithObjects:@0, @1, @2, nil];

NSLog(@"placeholder: %s", object_getClassName(placeholder));    // placeholder: __NSPlaceholderArray
NSLog(@"arr1: %s", object_getClassName(arr1));                  // arr1: __NSArray0
NSLog(@"arr2: %s", object_getClassName(arr2));                  // arr2: __NSSingleObjectArrayI
NSLog(@"arr3: %s", object_getClassName(arr3));                  // arr3: __NSArrayI
NSLog(@"arr4: %s", object_getClassName(arr4));                  // arr4: __NSArrayI
```

可以看到，alloc后所得到的类为 **__NSPlaceholderArray。**而当init为一个空数组后，变成了 **__NSArray0**。如果有且仅有一个元素，那么为**__NSSingleObjectArrayI。**如果数组大于一个元素，那么为**_NSArrayI。**这里暂时不去讨论为什么arr1-4有所区别----先来关心一下为什么alloc和init前后转化为了不同的类。

从名字上很容易知道**__NSPlaceholderArray**作用为占位，我们可以尝试打印几个地址：

```
NSArray *placeholder = [NSArray alloc];
NSArray *placeholder2 = [NSArray alloc];
NSArray *arr1 = [placeholder init];
NSArray *arr2 = [placeholder initWithObjects:@0, nil];

NSLog(@"placeholder: %p", placeholder);     // placeholder: 0x618000013b10
NSLog(@"placeholder2: %p", placeholder2);   // placeholder2: 0x618000013b10
NSLog(@"arr1: %p", arr1);                   // arr1: 0x618000013b30
NSLog(@"arr2: %p", arr2);                   // arr2: 0x608000014050
```

可以看到[NSArray alloc]产生的实例为一个单例，而在init或者其他初始化方法后，地址发生了变化，也就是说，placeholder目前看来只是一个占位用的单例，在init之后就被新的实例给替换掉了。那么，这个placeholder真的只用作占位吗？

**__NSPlaceholderArray**
我们可以参考另一个开源实现GNUstep一瞥究竟。根据GUNstep的代码，可知NSObject的alloc是直接返回的[self allocWithZone:NSDefaultZone()]，也就是说调用了对应类实现的此方法。我们来看看GUNstep中的NSArray的allocWithZone：是如何实现的：

```
+ (id) allocWithZone: (NSZone*)z
{
  if (self == NSArrayClass)
  {
    /*
    * For a constant array, we return a placeholder object that can
    * be converted to a real object when its initialisation method
    * is called.
    */
    if (z == NSDefaultMallocZone() || z == 0)
    {
      /*
      * As a special case, we can return a placeholder for an array
      * in the default malloc zone extremely efficiently.
      */
      return defaultPlaceholderArray;
    }
    else
    {
      // 此处省略
    }
  }
  else
  {
    return NSAllocateObject(self, 0, z);
  }
}
```

可以看到NSArray此时会返回defaultPlaceholderArray。在GUNstep的实现中，defaultPlaceholderArray实例所对应的类为GSPlaceholderArray。所以alloc完成后的init消息是发送给GSPlaceholderArray实例的。而init恰恰调用的是initWithObjects:count:-----这个方法其实就是NSArray的指定初始化方法。我们继续看看GNUstep实现：

```
// GSPlaceholderArray
- (id) initWithObjects: (const id[])objects count: (NSUInteger)count
{
  self = (id)NSAllocateObject(GSInlineArrayClass, sizeof(id)*count, [self zone]);
  return [self initWithObjects: objects count: count];
}

// GSInlineArray
- (id) initWithObjects: (const id[])objects count: (NSUInteger)count
{
  _contents_array = (id*)(((void*)self) + class_getInstanceSize([self class]));

  if (count > 0)
  {
    NSUInteger  i;

    for (i = 0; i < count; i++)
    {
      if ((_contents_array[i] = RETAIN(objects[i])) == nil)
      {
        _count = i;
        DESTROY(self);
        [NSException raise: NSInvalidArgumentException format: @"Tried to init array with nil object"];
      }
    }
    _count = count;
  }
  return self;
}
```

可以看到在GSPlaceholderArray的initWithObjects:count:方法中，通过NSAllocateObject给GSInlineArray实例分配空间，包括所含元素的空间。并且在GSInlineArray的initWithObjects:count:方法中，对分配的元素的空间进行初始化。自此就返回了一个类型为GSInlineArray实例。

CoreFoundation中NSArray的相关实现会比GNUstep中的实现复杂些，但通过汇编代码来看可以知道基本逻辑是类似的，在此不再赘述，有几点可以提下：
- 1、当元素为空时，返回的是**__NSArray0**的单例
- 2、当元素有且仅有一个时，返回的是**__NSSingleObjectArrayI**的实例
- 3、当元素大于一个的时候，返回的是**__NSArrayI**的实例

同样的，对于NSMutableArray、NSNumber、NSString等也是有相同的placeholder机制的。
