
# <p align='center'>NSObject本质探究</p>

## OC对象的本质
我们平时编写的Objective-C代码，底层实现其实都是C\C++代码。

![OC->C](https://github.com/syxBravely/BlogImages/raw/master/iOS/OC-%3EC.png)

接下来，我们着重分析一下，OC转换为C/C++到底是怎么样的。
OC代码如下：

```
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        NSObject *objc = [[NSObject alloc] init];
        
    }
    return 0;
}
```

首先，我们先点击**NSObjcet**进入发现**NSObject**的内部实现如下：

```
@interface NSObject <NSObject> {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
#pragma clang diagnostic pop
}
```

**NSObjcet**对象只有一个属性**isa**。


接下来，我们使用以下命令行将OC代码转化为C++文件
> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main-arm64.cpp 

我们在`main-arm64.cpp `文件中搜索`NSObject_IMPL`，我们或发现以下代码：

```
struct NSObject_IMPL {
	Class isa;
};
```
上述代码可能就是`NSObject`转化的C++代码。

现在，我们再定义一个`Person`对象，如下

```
@interface Person : NSObject {
    NSString *_name;
    int       _age;
}

@end
```

在`main`函数里初始化，并转化为C++文件，接着在`main-arm64.cpp `文件中搜索`Person_IMPL`，我们发现以下代码：

```
struct Person_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	NSString *_name;
	int _age;
};

```
我们发现，`Person_IMPL`结构体里除了我们自己定义的变量外，还有一个`NSObject_IMPL`类型的变量`NSObject_IVARS`。

最后，我们再定义一个`Student`对象并继承`Person`，去看看其转换成C++是怎样的？

最终是这样的：

```
struct Student_IMPL {
	struct Person_IMPL Person_IVARS;
	int _number;
};
```

我们发现，`Student_IMPL`结构体里除了我们自己定义的变量外，还有一个`Person_IMPL`类型的变量`Person_IVARS`。

通过上面的代码分析，我们大概可以得出这样的结论：类对象实质上是以结构体的形式存储在内存中。
下图可以较直观的反应上述对象在内存中的分布：
![](https://github.com/syxBravely/BlogImages/raw/master/iOS/OC_Struct.png)

## OC对象的分类
通过上面的代码，我们并没有发现类信息（类的属性信息，类的对象方法信息，类的协议信息等）的描述。但是发现只要是继承自NSObject的对象，那么底层结构体内一定有一个isa指针，除此之外为自定义的变量。
那么我们可以猜想，类信息可能放在`isa指针`指向的内存中。

首先，我们先说明一下，OC对象主要可以分为三种：
1. instance对象（实例对象）
2. class对象（类对象）
3. meta-class对象（元类对象）

关于三者之前的关系，我们可以看一下下面的这张经典图：
![isa](https://github.com/syxBravely/BlogImages/raw/master/iOS/isa.png)

对上图的分析说明：
1. instance的isa指向class
2. class的isa指向meta-class
3. meta-class的isa指向基类的meta-class，基类的isa指向自己
4. class的superclass指向父类的class，如果没有父类，superclass指针为nil
5. meta-class的superclass指向父类的meta-class，基类的meta-class的superclass指向基类的class
6. instance调用对象方法的轨迹，isa找到class，方法不存在，就通过superclass找父类
7. class调用类方法的轨迹，isa找meta-class，方法不存在，就通过superclass找父类

接下来，我们来验证一下上面**isa**的指向是否真确。

代码如下:

```
NSObject *object = [[NSObject alloc] init];
Class objectClass = [NSObject class];
Class objectMetaClass = object_getClass([NSObject class]);
        
NSLog(@"\n object %p \n objectClass %p \n objectMetaClass %p", object, objectClass, objectMetaClass);

```
打印结果如下

```
object          0x1005344a0 
objectClass     0x7fff930aa140 
objectMetaClass 0x7fff930aa0f0
```
接下来，我们通过断点的方式来获取实例对象的isa指针的值

```
(lldb) p/x object->isa
(Class) $0 = 0x001dffff930aa141 NSObject
(lldb) p/x objectClass
(Class) $1 = 0x00007fff930aa140 NSObject
```
从上面我们发现，实例对象的**isa**和我们得到的值竟然不一样😱。我们上面的结论错了吗？
这是因为在arm64架构之前，**isa**就是一个普通的指针，存储了Class,metal-class的地址。
从arm64架构开始，苹果对**isa**进行了优化，变成了一个公用体(union)结构，还使用位域来存储更多信息。（大家可以找一下相关资料了解一下）。
isa需要进行一次位运算，才能计算出真实地址。即 isa & ISA_MASK 。
而位运算的值我们可以通过下载[objc](https://opensource.apple.com/tarballs/objc4/)源代码找到

```
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
# elif __x86_64__
#   define ISA_MASK        0x00007ffffffffff8ULL

```
我们来计算一下

```
p/x 0x001dffff930aa141 & 0x00007ffffffffff8
$3 = 0x00007fff930aa140
```
我们发现，object-isa指针地址0x001dffff96537141经过同0x00007ffffffffff8位运算，得出objectClass的地址0x00007fff96537140。

接着我们来验证class对象的isa指针是否同样需要位运算计算出meta-class对象的地址。 当我们以同样的方式打印objectClass->isa指针时，发现无法打印。

那么，我们自己创建一个同样的结构体并通过强制转化拿到isa指针。

```
struct NSObject_IMPL {
    Class isa;
};


struct NSObject_IMPL *objc_impl =  (__bridge struct NSObject_IMPL *)(objectClass);
```

现在，我们重新验证一下

```
(lldb) p/x objectMetaClass
(Class) $0 = 0x00007fff930aa0f0
(lldb) p/x objc_impl->isa
(Class) $1 = 0x001dffff930aa0f1
(lldb) p/x  0x001dffff930aa0f1 & 0x00007ffffffffff8
(long) $2 = 0x00007fff930aa0f0
```

证实了objectClass2的isa指针经过位运算之后的地址是meta-class的地址。

**总结:**
1. OC对象本质上是一个C++结构体，内部包含了一个isa指针和自定义的变量。
2. OC对象分为3种:instance对象（实例对象），class对象（类对象），meta-class对象（元类对象）。
3. 成员变量的值存储在instance对象中，对象方法，协议，属性，成员变量信息存放在class对象。类方法信息存放在meta-class对象。



