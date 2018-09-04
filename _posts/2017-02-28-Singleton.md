---
title: Singleton
description: This is an introduction to singleton in ios.
header: singleton
---

昨天去面试遇到了一些平时很常见但是从没花过心思的题，然后回来整理了一下，没想到万恶的csdn把我写了一下午的心得根本没有保存，只留下一行寥寥无几的文字在那赤裸裸的嘲讽我。

言归正传，下面首先讲一讲singleton的写法：

OC版：

1.线程锁方式

	@implementation Singleton
	static Singleton *instance = nil;
	
	+(instancetype) sharedInstance {
	    @synchronized (self) {
	        if (instance == nil) {
	            instance = [[Singleton alloc] init];
	            return instance;
	        }
	    }
	    return instance;
	}
	
	+(instancetype) allocWithZone:(struct _NSZone *)zone {
	    @synchronized(self) {
	        if (instance == nil) {
	            instance = [super allocWithZone: zone];
	            return instance;
	        }
	    }
	    return instance;
	}
	
	-(id)copyWithZone:(NSZone *)zone {
	    return self;
	}

2.dispatch_once

	@implementation Singleton
	static Singleton *instance = nil;
	
	+(instancetype)sharedInstance {
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        //if (instance == nil) {
	    	instance = [[Singleton alloc] init];
	        //}
	    });
	    return instance;
	}
	
	+(instancetype)allocWithZone:(struct _NSZone *)zone {
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	        //if (instance == nil) {
	        instance = [super allocWithZone:zone];
	        //}
	    });
	    return instance;
	}
	
	-(id)copyWithZone:(NSZone *)zone {
	    return instance;
	}
	
	-(id)mutableCopyWithZone:(NSZone *)zone {
	    return instance;
	}

看见这些我心里有很多疑问，@synchronized是个嘛东西，原理？dispatch_once又是干嘛的，allocWithZone又是什么？dispatch_once_t？

![???](https://Jeremy1221.github.io/img/doubt.gif)

首先我们来看看@synchronized和dispatch\_once，先举个例子，比如现在同时有两个线程(A和B)要使用sharedInstance创建实例，当两个线程同事运行到@synchronized的代码块时，其中一个线程(A)会上锁，然后线程B会进入睡眠直到线程A运行结束，当线程A运行结束时instance已经不是nil了，所以线程B不会再创建。同理，dispatch\_once会保证这段代码只运行一次，所以线程A先运行后，线程B就不会再运行了。[原文戳这里](http://www.cocoachina.com/ios/20160613/16661.html)

下面我们来看看dispatch\_once\_t与dispatch\_once:

![dispatch_once_t](https://jeremy1221.github.io/img/dispatch_once_t.png)

typedef long dispatch_once_t;
自己领悟吧。

再来看看allocWithZone,原来这是个历史遗留问题，当我们调用alloc时最终还是会调用allocWithZone，所以当某人直接调用allocWithZone(谁会装逼这么用？)来创建实例时还是会创建出一个新的实例从而不能保证唯一性。[了解更多点击我](http://blog.csdn.net/jiajiayouba/article/details/44306679)

###2018.09.04 有新的发现

在复习单利的时候发现了一些洗的问题

在上面使用dispatch\_once\_t的代码注释了两行，在这个锁(暂且这么叫吧)里面不用再判断instance是否为空了，因为只会运行一次。

而且还发现了新的简化代码的写法:

	@implementation Singleton
	static Singleton *instance = nil;
	
	+(instancetype)sharedInstance {
	    static dispatch_once_t onceToken;
	    dispatch_once(&onceToken, ^{
	    	instance = [[super allocWithZone:NULL] init];
	    });
	    return instance;
	}
	
	+(instancetype)allocWithZone:(struct _NSZone *)zone {
	    return [Singleton sharedInstance];
	    //如果想一劳永逸的话，或许可以这么写
	    //return [[self class] sharedInstance];
	}
One demo run everywhere.
这样我们就可以使用一个模板来写单例了。

接下来我们看swift版的单例，swift有四种写法：

1.OC翻译版(太丑)

	class Singleton {
	    class var sharedInstance: Singleton {
	        struct Static {
	            static var onceToken: dispatch_once_t = 0
	            static var instance: Singleton? = nil
	        }
	        dispatch_once(&Static.onceToken) {
	            Static.instance = Singleton()
	        }
	        return Static.instance!
	    }
	}

swift3.0已经废弃了dispatch_once了好像，所以上述方法已经不能通过编译了，不过我还是觉得可以拿出来说一下

2.结构体方法(1的优化)

	class Singleton {
	    class var sharedInstance: Singleton {
	        struct Static {
	            static let instance = Singleton()
	        }
	        return Static.instance
	    }
	}

3.全局变量方法

	private let instance = Singleton()
	class Singleton {
	    class var sharedInstance: Singleton {
	        return instance
	    }
	}
	
现在，你可能会有疑问：为何看不到dispatch_once？根据Apple Swift博客中的说法，以上方法都自动满足dispatch_once规则。这里有个帖子可以证明dispatch_once规则一直在起作用。

4.最终方法

	class Singleton {
	    static let sharedInstance = Singleton()
	}

最后不要忘记设置初始化方法为私有，防止别人直接调用init方法来创建实例。

	class Singleton {
	    static let sharedInstance = Singleton()
	    private init() {}
	}

看到这里我又有了疑问，这个class和static修饰词是干嘛的呢(基础太差😂)

![static&class](https://jeremy1221.github.io/img/static%26class.png)

在类(class)中class和static是用来修饰computed property和stored property的，在我的测试中，两个效果是一样的，用class或static修饰后只能通过类名来访问，不能通过实例（对象）来访问。而在结构体或者枚举中只能用static。protocol中也可以用class。但是static修饰的属性不支持子类进行重定义，class修饰的属性支持子类重定义。至于computed property和stored property的作用，前者是用来计算的不直接存储值，而后者是用来存储值的。

类型属性语法

在 C 或 Objective-C 中，与某个类型关联的静态常量和静态变量，是作为全局（global）静态变量定义的。但是在 Swift 中，类型属性是作为类型定义的一部分写在类型最外层的花括号内，因此它的作用范围也就在类型支持的范围内。

使用关键字 static 来定义类型属性。在为类定义计算型类型属性时，可以改用关键字 class 来支持子类对父类的实现进行重写。下面的例子演示了存储型和计算型类型属性的语法：

	struct SomeStructure {
	    static var storedTypeProperty = "Some value."
	    static var computedTypeProperty: Int {
	        return 1
	    }
	}
	enum SomeEnumeration {
	    static var storedTypeProperty = "Some value."
	    static var computedTypeProperty: Int {
	        return 6
	    }
	}
	class SomeClass {
	    static var storedTypeProperty = "Some value."
	    static var computedTypeProperty: Int {
	        return 27
	    }
	    class var overrideableComputedTypeProperty: Int {
	        return 107
	    }
	}