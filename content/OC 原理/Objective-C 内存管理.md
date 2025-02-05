# Objective-C 内存管理
### 程序的内存结构

### 错误的内存管理会带来的问题：
1. 使用已经释放或重写过的内存数据。
    会造成数据混乱，通常的结果是crash，设置损坏用户的数据。
2. 没有释放不再使用的数据，造成的内存泄漏。
    应用程序在使用过程中不断增加的内存量，这可能导致系统性能不佳或内存占用过多而被终止。

## 二、引用计数管理内存
iOS中使用了引用计数来管理内存，引用计数中包含两种方式：MRC 和 ARC。这里假设读者使用过MRC并且有一定了解。

## 三、内存管理
在MRC中有四个法则知道程序员手动管理内存：
* 自己生成的对象，自己持有。
* 非自己生成的对象，自己也能持有。
* 不在需要自己持有对象的时候，释放。
* 非自己持有的对象无需释放。

四个法则对应的代码：

```objective-c
/*
 * 自己生成并持有该对象
 */
 id obj0 = [[NSObeject alloc] init];
 id obj1 = [NSObeject new];
```

```
/*
 * 持有非自己生成的对象
 */
id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
[obj retain]; // 自己持有对象
```

```
/*
 * 不在需要自己持有的对象的时候，释放
 */
id obj = [[NSObeject alloc] init]; // 此时持有对象
[obj release]; // 释放对象
/*
 * 指向对象的指针仍就被保留在obj这个变量中
 * 但对象已经释放，不可访问
 */
```

```
/*
 * 非自己持有的对象无法释放
 */
id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
[obj release]; // ~~~此时将运行时crash 或编译器报error~~~ 非 ARC 下，调用该方法会导致编译器报 issues。此操作的行为是未定义的，可能会导致运行时 crash 或者其它未知行为
```
**非自己持有的对象无法释放**，这些对象什么时候释放呢？这就要利用**autorelease**对象来实现的，**autorelease**对象不会在作用域结束时立即释放，而是会加入**autoreleasepool**释放池中，应用程序在事件循环的每个**循环开始**时在主线程上创建一个**autoreleasepool**，并在**循环最后**调用**drain**将其排出，这时会调用**autoreleasepool**中的每一个对象的**release**方法。

### 修饰符
#### 属性修饰符
```objective-c
@property (assign/retain/strong/weak/unsafe_unretained/copy) NSArray *array;
```

`assign`: 表明`setter` 仅仅是一个简单的赋值操作，通常用于基本的数值类型，例如CGFloat和NSInteger。
`retain`: 引用计数加1。
`strong`: 和`retain`类似，引用计数加1。对象类型时默认就是`strong`。
`weak`: 和`assign`类似，当对象释放时，会自动设置为`nil`。
`unsafe_unretained`: 的语义和`assign`类似，不过是用于对象类型的，表示一个非拥有(unretained)的，同时也不会在对象被销毁时置为`nil`的(unsafe)关系。性能优于`weak`参照`weak`实现原理。
`copy`: 类似`storng`，不过在赋值时进行`copy`操作而不是`retain`，在`setter`中比较明显的会发现一个`copy`行为。 常在model或者赋值时使用，防止外部修改或者多线程中修改。

#### 变量修饰符
`__strong`: 对象默认使用标识符，`retain`+1。只要存在`strong`指针指向一个对象那他就会一直保存存活。
`__weak`: 弱引用对象，引用计数不会增加。对象被销毁时自己被置为 nil 。
`__unsafe_unretained`: 不会持有对象，引用计数不会增加，但是在对象被销毁时不会自动置为nil，该指针依旧指向原来的地址，这就变成一个悬垂指针了。
`__autoreleasing`: 用来表示`d *`修饰的参数，并且在返回时被自动释放掉。

``` objctive-c
// ClassName * qualifier variableName;
// for example:
MyClass * __weak myWeakReference;
MyClass * __unsafe_unretained myUnsafeReference;
```
> `qualifier`只能放在 * 和 变量名 之间，但是放到其他位置也不会报错，编译器对此做过优化。

在使用引用地址传值时需要特别注意，比如以下代码能正常工作：
```objective-c
NSError *error;
BOOL OK = [myObject performOperationWithError:&error];
if (!OK) {
    // Report the error.
    // ...
}
```
但是，这里有一个错误的隐式声明：
`NSError * __strong error;`
而方法的声明是：
`-(BOOL)performOperationWithError:(NSError * __autoreleasing *)error;`

因此编译器会重写：
```objective-c
NSError * __strong error;
NSError * __autoreleasing tmp = error;
BOOL OK = [myObject performOperationWithError:&tmp];
error = tmp;
if (!OK) {
    // Report the error.
    // ...
}
```
当然你也可以创建`-(BOOL)performOperationWithError:(NSError * __strong *)error;`方法，也可以创建`NSError * __autoreleasing error;`使他们的类型一致，采用何种方式视具体上下文逻辑而定。

### 循环引用问题
使用引用计数管理内存时，不可避免的会遇到循环引用问题。 产生原因是多个对象间存在相互引用，其中某个对象的释放都依赖于另一个对象的释放，形成了一个独立的环状结构。

为了打破这个循环引用关系，有以下两种办法：
1. 手动将其中的一条强引用置为nil。
2. 使用`weak`弱引用的方式，修饰对象。

### AutoreleasePool
上面也有提到了**AutoreleasePool**，这在整个内存管理中扮演了非常重要的角色。
在[NSAutoreleasePool](https://developer.apple.com/documentation/foundation/nsautoreleasepool?language=occ)文档中：
> In a reference counted environment, Cocoa expects there to be an autorelease pool always available. If a pool is not available, autoreleased objects do not get released and you leak memory. In this situation, your program will typically log suitable warning messages.

> The Application Kit creates an autorelease pool on the main thread at the beginning of every cycle of the event loop, and drains it at the end, thereby releasing any autoreleased objects generated while processing an event. If you use the Application Kit, you therefore typically don’t have to create your own pools. If your application creates a lot of temporary autoreleased objects within the event loop, however, it may be beneficial to create “local” autorelease pools to help to minimize the peak memory footprint.

>  **autoreleasepool** 和线程的关系
> Each thread (including the main thread) maintains its own stack of NSAutoreleasePool objects (see Threads). As new pools are created, they get added to the top of the stack. When pools are deallocated, they are removed from the stack. Autoreleased objects are placed into the top autorelease pool for the current thread. When a thread terminates, it automatically drains all of the autorelease pools associated with itself.
> Threads
If you are making Cocoa calls outside of the Application Kit’s main thread—for example if you create a Foundation-only application or if you detach a thread—you need to create your own autorelease pool.

>If your application or thread is long-lived and potentially generates a lot of autoreleased objects, you should periodically drain and create autorelease pools (like the Application Kit does on the main thread); otherwise, autoreleased objects accumulate and your memory footprint grows. If, however, your detached thread does not make Cocoa calls, you do not need to create an autorelease pool.

文中提到了**autoreleasepool**4个比较特别的情况：
1. 如果**autoreleasepool**无效时，autorelease对象是无法收到release消息，从而造成内存泄漏。在ARC情况下很少会出现**autoreleasepool**无效的情况下。

2. 对于需要产生大量临时**autorelease**对象的逻辑，需要使用**@autoreleasepool{}**来立即释放来降低内存的峰值。

3. 关于autoreleasepool在线程中的线程的布局，官方文档说每一个线程都会在栈中维护创建的NSAutoreleasePool 对象，并且会把这个对象放到栈的顶部，从而确保在线程结束时autoreleasepool能在最后drain并且dealloc后从栈中移除。

4. autoreleasepool与线程的关系，除了**main thread**外其他线程都没有自动生成的autoreleasepool。如果你的线程需要长时间存活或者会有大量autorelease对象生成，就得自己创建autoreleasepool了。尤其是长时间存活的线程，你还需要像主线程在runloop末尾定时 的去drain。

## 四、内存泄漏检测
 1. 观察内存增长减少情况，在退出界面时，观察内存增长情况。 
 2. 查看对象`delloc`方法调用情况。
 3. Xcode提供的Debug Memory Graph。
 4. Xcode提供的instruments。
 5. [MLeaksFinder](https://github.com/Tencent/MLeaksFinder)



## 参考资料
[《Apple About Memory Management》](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html#//apple_ref/doc/uid/10000011-SW1)
[《Clang中ARC的实现》](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)
[《黑幕背后的 Autorelease》](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)