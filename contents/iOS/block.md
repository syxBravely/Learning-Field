# Block 底层分析
## 一、Block简介
**Block**：带有==自动变量==的==匿名函数==。（开发过程中我们常用便捷的回调方式）
**匿名函数**：没有函数名的函数，一对{}包裹的内容是匿名函数的作用域。
**自动变量**：那些在Block的作用域里使用到但却是在作用域以外声明的变量，就是Block截获的自动变量。（栈上声明的一个变量既不是*静态变量*也不是*全局变量*，是不能在这个栈内声明的匿名函数里使用，但在Block中可以）
## 二、Block的本质
* block本质上也是一个OC对象，它内部也有个isa指针
* block是封装了函数调用以及函数调用环境的OC对象
* block是封装函数及其上下文的OC对象

我们先定义一个Block（如下代码）：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int age = 20;
        void (^customBlock1)(int, int) =  ^(int a , int b){
            NSLog(@"this is a block! -- %d", age);
        };
        
        customBlock1(1, 1);
    }
    return 0;
}
```
我们使用 clang 将 OC 代码转换为 C 文件：
`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m`

```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

int age = 20;

void (*customBlock1)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));

((void (*)(__block_impl *, int, int))((__block_impl *)customBlock1)->FuncPtr)((__block_impl *)customBlock1, 1, 1);
        
    return 0;
}

//我们分析一下上面的两段代码，去掉一些强制转换的代码，我们可以看得更加清晰
//定义Block变量
void (*customBlock1)(int, int) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA, age));
//调用Block
((customBlock1)->FuncPtr)(customBlock1, 1, 1);

```

我们在转换的C文件里，搜索**__main_block_impl_0**，**__main_block_func_0**，**__main_block_desc_0**，**__block_impl**我们会找到以下代码

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;//描述Block大小、版本等信息
  int age;
    //结构体构造函数（类似与OC的init方法）
  __main_block_impl_0(void *fp,//block函数实现
                      struct __main_block_desc_0 *desc,
                      int _age,
                      int flags=0) : age(_age) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

//定义的Block描述信息
static struct __main_block_desc_0 {
  size_t reserved; //值为0
  size_t Block_size;//__main_block_impl_0的内存大小
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

//Block的真面目
struct __block_impl {
  void *isa;//isa指针指向当前block对象类对象
  int Flags;
  int Reserved;
  void *FuncPtr;//指向block保存的函数指针
};

//封装了我们定义的Block内部实现的函数
static void __main_block_func_0(
   struct __main_block_impl_0 *__cself, 
   int a, 
   int b) {
int age = __cself->age; // bound by copy

NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_a264ff_mi_0, age);
}

```
我们画图来直观的显示以上结构关系：
![Block底层结构图](https://github.com/syxBravely/BlogImages/raw/master/iOS/block_c.jpeg)

由以上分析，我们来简略的总结一下一个简单的`Block`的定义以及调用转换过程：

1. 定义`Block`变量相当于调用`__main_block_impl_0`构造函数，向其构造函数里传入`&__main_block_desc_0_DATA`（当前定义的block描述信息结构体地址）和 `__main_block_func_0`（封装的block函数代码块指针），最后得到`__main_block_impl_0`实例。

2. 在`__main_block_impl_0`构造函数内部，将外部传进来的`__main_block_func_0`函数指针，设置内部实际的block变量（`__block_impl类型的结构体`）的函数指针。

3. 调用block时，取出`__main_block_impl_0结构体`中的`__block_impl结构体`中的函数指针（`__main_block_func_0`）并调用。

## 三、Block变量捕获
> 为了保证block内部能够正常访问外部的变量，block有个变量捕获机制

```
int gloableVar = 10;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        auto   int autoVar   = 20;
        static int staticVar = 30;
        
        void (^customBlock1)(int, int) =  ^(int a , int b){
            NSLog(@"GlobleVar:%d autoVar:%d staticVar:%d", gloableVar,autoVar,staticVar);
        };
      
        
        customBlock1(1, 1);
        
    }
    return 0;
}
```
使用`clang`将上述代码转换为C文件

```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        auto int autoVar = 20;
        static int staticVar = 30;

        void (*customBlock1)(int, int) = ((void (*)(int, int))&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, autoVar, &staticVar));


        ((void (*)(__block_impl *, int, int))((__block_impl *)customBlock1)->FuncPtr)((__block_impl *)customBlock1, 1, 1);

    }
    return 0;
}

static void __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
  int autoVar = __cself->autoVar; // bound by copy
  int *staticVar = __cself->staticVar; // bound by copy

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_d71298_mi_0, gloableVar,autoVar,(*staticVar));
        }
```
从上述转化过得代码中我们可以看出：
1. `static`局部变量的访问方式是指针传递，`auto`局部变量的访问方式是值传递，全局变量是直接访问。
2. 局部变量会被block捕获到其内部，全局变量不会，直接在函数代码块里调用。

block的变量捕获机制如下图：
![Block变量捕获机制图](https://github.com/syxBravely/BlogImages/raw/master/iOS/block_cap.png)

但是我们会有以下的疑问：
**1. 为什么会有变量捕获？**
   ==->==为了保证block内部能够正常访问外部的变量，block有个变量捕获机制。
**2. 为什么block对auto和static变量捕获有差异？**
   ==->==auto自动变量可能会销毁的，内存可能会消失，不采用指针访问；static变量一直保存在内存中，指针访问即可。
## 四、block类型
 block的类型，取决于isa指针，可以通过调用class方法或者isa指针查看具体类型，最终都是继承自NSBlock类型

* __NSGlobalBlock __  （存在于全局内存中, 相当于单例）
* __NSStackBlock __   （存在于栈内存中, 超出其作用域则马上被销毁）
* __NSMallocBlock __  （存在于堆内存中, 是一个带引用计数的对象, 需要自行管理其内存）

但是我们怎么去判断一个人block的类型了？
1. 没有访问auto变量的block是__NSGlobalBlock __ ，放在数据段。
2. 访问了auto变量的block是__NSStackBlock __。
3. `[__NSStackBlock __ copy]`操作就变成了__NSMallocBlock __。
接下来，我们运行一下下面的代码(在ARC环境下)，看看定义的Block的类型：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int age = 1;
    
        void (^gloableBlock)(void) = ^{
            NSLog(@"gloableBlock");
        };
        
        void (^mallocBlock)(void) = ^{
            NSLog(@"mallocBlock:%d",age);
        };
        
        NSLog(@"%@/%@/%@",[gloableBlock class],[mallocBlock class],[^{
            NSLog(@"block3:%d",age);
        } class]);
      
    }
    return 0;
}
```
打印结果：

```
__NSGlobalBlock__/__NSMallocBlock__/__NSStackBlock__
```
按照上面的结论`mallocBlock`的类型应该是`__NSStackBlock__`的，但是打印结果却是`__NSMallocBlock__`。这是为什么?上面的结论是不是错了？

这是因为在ARC环境下，编译器会根据情况自动将栈上的block复制到堆上。

在以下几种情况下，在ARC环境下编译器会根据情况自动将栈上的block复制到堆上：
1. block作为函数返回值时
2. 将block赋值给__strong指针时
3. block作为Cocoa API中方法名含有usingBlock的方法参数时
4. block作为GCD API的方法参数时

## 五、Block捕获对象类型变量时强引用问题
**在栈Block里**
如果block是在栈上，将不会对auto变量产生强引用，因为栈上的block随时会被销毁，也没必要去强引用其他对象。
**在堆Block里**
1.如果block被拷贝到堆上：
a) 会调用block内部的`copy函数`
b) `copy函数`内部会调用`_Block_object_assign`函数
c) `_Block_object_assign`函数会根据auto变量的修饰符（`__strong`、`__weak`、`__unsafe_unretained`）做出相应的操作，形成强引用（`retain`）或者弱引用

## 六、__block关键字
我们都知道Block可以捕获自动变量，保证了栈上的自动变量被销毁后，Block内仍可使用该变量，以至于在Block里我们无法修改外部变量的值，只能读取，无法写入。

**__block**保证了栈上和Block内（通常在堆上）可以访问和修改“同一个变量”。

**__block**发挥作用的原理：将栈上用__block修饰的自动变量封装成一个结构体，让其在堆上创建，以方便从栈上或堆上访问和修改同一份数据。
看下面代码：

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        __block int age = 10;
        
        void (^block1)(void) = ^{
            age = 20;
        };
        
        block1();
        NSLog(@"age：%d",age);//age：20
        
    }
    return 0;
}

```

运行后，我们可以看得到`age`的值被改成20了。`__block`使我们可以修改外部的变量。

接下来我们分析一下其原理。
首先，我们先将上面的代码Clang一下，得到以下代码:


```
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        __attribute__((__blocks__(byref))) __Block_byref_age_0 age = {(void*)0,(__Block_byref_age_0 *)&age, 0, sizeof(__Block_byref_age_0), 10};

        void (*block1)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_age_0 *)&age, 570425344));

        ((void (*)(__block_impl *))((__block_impl *)block1)->FuncPtr)((__block_impl *)block1);
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_9b9c74_mi_0,(age.__forwarding->age));

    }
    return 0;
}
```

从上面，我们发现我们声明的变量`age`变成了一个`__Block_byref_age_0`结构体对象，我们在文件里找到`__Block_byref_objc_1`。

`__Block_byref_objc_1`定义如下：

```
struct __Block_byref_age_0 {
  void *__isa;  //0
__Block_byref_age_0 *__forwarding;//age的地址
 int __flags;//0
 int __size;//当前结构体的size
 int age;//定义的变量age的值10
};
```

* __isa指针 ：__Block_byref_age_0中也有isa指针也就是说__Block_byref_age_0本质也一个对象。
* __forwarding ：__forwarding是__Block_byref_age_0结构体类型的，并且__forwarding存储的值为(__Block_byref_age_0 *)&age，即结构体自己的内存地址。
* __flags ：0
* __size ：sizeof(__Block_byref_age_0)即__Block_byref_age_0所占用的内存空间。
* age ：真正存储变量的地方，这里存储局部变量10。

**接下来，**我们发现`__Block_byref_age_0`结构体`age`被放入入`__main_block_impl_0`结构体中，并赋值给`__Block_byref_age_0 *age`。

之后，在函数代码块里修改外部变量age的值过程如下：
1. 首先，取出`__main_block_impl_0`中的age
2. 然后，通过age结构体拿到`__forwarding`指针
3. 最后，通过`__forwarding`拿到结构体中的age(10)变量并修改其值
（后续NSLog中使用age时也通过同样的方式获取age的值。）

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_age_0 *age = __cself->age; // bound by ref

            (age->__forwarding->age) = 20;
        }
```

**但是，**这里我有一个小疑问：`__forwarding`是指向自己的指针，为什么我们要通过`__forwarding`而不是直接使用封装的`__Block_byref_age_0`去对变量age进行操作？
这样的做法其实是为了方便内存管理，在下面内存管理章节我们在详细解释。

到此为止，__block为什么能修改变量的值已经很清晰了。__block将变量包装成对象，然后在把age封装在结构体里面，block内部存储的变量为结构体指针，也就可以通过指针找到内存地址进而修改变量的值。

## 七、Block内存管理
`__block`修饰的变量或捕获的变量是对象类型在block结构体中一直都是强引用，而其他类型的是由传入的对象指针类型决定。
我们看看下面的代码来深入观察一下：

```
typedef void(^SYBlock)(int,int);
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        int n1 = 1;
        __block int n_block = 10;
        
        NSObject *objc1 = [[NSObject alloc] init];
        __weak NSObject *weakObjc = objc1;
        __block NSObject *objc_block = [[NSObject alloc] init];
        
        Person *p = [Person new];
        __block __weak Person *weakP_block = p;
                
        SYBlock block1 = ^(int a, int b) {
            NSLog(@"%d", n1);//基本数据类型
            NSLog(@"%d", n_block);//__block修饰的基本类型
            NSLog(@"%@", objc1);//对象类型
            NSLog(@"%@", weakObjc);//__weak修饰的对象类型
            NSLog(@"%@", objc_block);//__block修饰的对象类型
            NSLog(@"%@", weakP_block);//__block,__weak修饰的对象类型
        };
        block1(1,1);
   
    }
    return 0;
}
```

将上述文件Clang转换为C文件,并找到结构体`__main_block_impl_0`如下:

```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int n1;
  NSObject *__strong objc1;
  NSObject *__weak weakObjc;
  __Block_byref_n_block_0 *n_block; // by ref
  __Block_byref_objc_block_1 *objc_block; // by ref
  __Block_byref_weakP_block_2 *weakP_block; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _n1, NSObject *__strong _objc1, NSObject *__weak _weakObjc, __Block_byref_n_block_0 *_n_block, __Block_byref_objc_block_1 *_objc_block, __Block_byref_weakP_block_2 *_weakP_block, int flags=0) : n1(_n1), objc1(_objc1), weakObjc(_weakObjc), n_block(_n_block->__forwarding), objc_block(_objc_block->__forwarding), weakP_block(_weakP_block->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
在`__main_block_impl_0`结构体中，我们可以看出没有被`__block`修饰的变量（objc1，weakObjc），根据其声明时是否指定强引用或弱引用，来确定其在`__main_block_impl_0`结构体中的持有关系。而一旦使用`__block`修饰的变量，`__main_block_impl_0`结构体内一律使用强指针引用生成的结构体。

接下来，我们找到结构体`__main_block_desc_0`，我们发现相比较以前，它多了两个函数`copy`和`dispose`，其值分别对应`__main_block_copy_0`，`__main_block_copy_0`，我们在文件中找到他们，代码如下：

```
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
//copy
 static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src){
    _Block_object_assign((void*)&dst->n_block, (void*)src->n_block, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_assign((void*)&dst->objc1, (void*)src->objc1, 3/*BLOCK_FIELD_IS_OBJECT*/);
    _Block_object_assign((void*)&dst->weakObjc, (void*)src->weakObjc, 3/*BLOCK_FIELD_IS_OBJECT*/);
    _Block_object_assign((void*)&dst->objc_block, (void*)src->objc_block, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_assign((void*)&dst->weakP_block, (void*)src->weakP_block, 8/*BLOCK_FIELD_IS_BYREF*/);}
    
//dispose
static void __main_block_dispose_0(struct __main_block_impl_0*src) {
    _Block_object_dispose((void*)src->n_block, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_dispose((void*)src->objc1, 3/*BLOCK_FIELD_IS_OBJECT*/);
    _Block_object_dispose((void*)src->weakObjc, 3/*BLOCK_FIELD_IS_OBJECT*/);
    _Block_object_dispose((void*)src->objc_block, 8/*BLOCK_FIELD_IS_BYREF*/);
    _Block_object_dispose((void*)src->weakP_block, 8/*BLOCK_FIELD_IS_BYREF*/);}
```

其实，只有当结构体`__main_block_impl_0`中捕获的变量是对象（包含被__block修饰封装的对象），结构体`__main_block_desc_0`里就会有多了两个函数`copy`和`dispose`，因为需要进行内存管理。

`__main_block_copy_0`函数中会根据变量是**强弱指针**及**有没有被__block修饰**做出不同的处理，**强指针**在Block内部产生强引用，**弱指针**在Block内部产生弱引用。被**__block**修饰的变量最后的参数传入的是8，没有被__block修饰的变量最后的参数传入的是3。

当Block从堆中移除时，会通过dispose函数来释放他们。

接下来，我们来看看`__block`修饰的变量生成的结构体是怎么样的，代码如下:

```
struct __Block_byref_n_block_0 {
  void *__isa;
__Block_byref_n_block_0 *__forwarding;
 int __flags;
 int __size;
 int n_block;
};

struct __Block_byref_objc_block_1 {
  void *__isa;
__Block_byref_objc_block_1 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 NSObject *__strong objc_block;
};

struct __Block_byref_weakP_block_2 {
  void *__isa;
__Block_byref_weakP_block_2 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 Person *__weak weakP_block;
};
```
从面代码我们可以看出：
`__block`修饰对象类型的变量生成的结构体内部多了`__Block_byref_id_object_copy`和`__Block_byref_id_object_dispose`两个函数，用来对对象类型的变量进行内存管理的操作。而结构体对对象的引用类型，则取决于block捕获的对象类型的变量。`weakP_block`是弱指针，所以`__Block_byref_weakP_block_2`对`weakP_block`就是弱引用，`objc_block`是强指针，所以`__Block_byref_objc_block_1`对`objc_block`就是强引用。

**__forwarding指针**
我们可以看到`__block`修饰的变量生成的结构体里都有`__forwarding指针`，从上面的分析中，我们知道其值为指向的是结构体自己的指针。通过结构体找到`__forwarding指针`，在通过`__forwarding指针`找到相应的变量。这样设计的目的是为了方便内存管理。

接下来我们看看，在函数代码块里是怎么调用__block修饰的变量的：

```
static void __main_block_func_0(struct __main_block_impl_0 *__cself, int a, int b) {
  __Block_byref_n_block_0 *n_block = __cself->n_block; // bound by ref
  __Block_byref_objc_block_1 *objc_block = __cself->objc_block; // bound by ref
  __Block_byref_weakP_block_2 *weakP_block = __cself->weakP_block; // bound by ref
  int n1 = __cself->n1; // bound by copy
  NSObject *__strong objc1 = __cself->objc1; // bound by copy
  NSObject *__weak weakObjc = __cself->weakObjc; // bound by copy

            (n_block->__forwarding->n_block) = 20;
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_85936c_mi_0, n1);
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_85936c_mi_1, (n_block->__forwarding->n_block));
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_85936c_mi_2, objc1);
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_85936c_mi_3, weakObjc);
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_85936c_mi_4, (objc_block->__forwarding->objc_block));
            NSLog((NSString *)&__NSConstantStringImpl__var_folders_zk_73gwkj_x5sscfthd5y578bx00000gn_T_main_85936c_mi_5, (weakP_block->__forwarding->weakP_block));
        }
```
通过源码可以知道：

* 当修改__block修饰的变量时，是根据变量生成的结构体这里是__Block_byref_n_block_0找到其中__forwarding指针，__forwarding指针指向的是结构体自己因此可以找到n_block变量进行修改。

* 当block在栈中时，__Block_byref_n_block_0结构体内的__forwarding指针指向结构体自己。

* 而当block被复制到堆中时，栈中的__Block_byref_n_block_0结构体也会被复制到堆中一份，而此时栈中的__Block_byref_n_block_0结构体中的__forwarding指针指向的就是堆中的__Block_byref_n_block_0结构体，堆中__Block_byref_n_block_0结构体内的__forwarding指针依然指向自己。

从上面代码我们可以看出，当修改变量`n_block`时：

* `__Block_byref_n_block_0 *n_block = __cself->n_block;`获取封装的结构体
* `n_block->__forwarding`获取堆中的n_block结构体
* `(n_block->__forwarding->n_block) = 20` 修改堆中n_block结构体的n_block变量
* 
我们通过下图展示__forwarding指针的作用
![__forwarding图](https://github.com/syxBravely/BlogImages/raw/master/iOS/forwarding.png)


现在，让我们来分析一下被__block修饰的对象类型的内存管理,在OC里变量的声明为`__block NSObject *objc_block = [[NSObject alloc] init];`

```
struct __Block_byref_objc_block_1 {
  void *__isa; //8
__Block_byref_objc_block_1 *__forwarding;//8
 int __flags;//4
 int __size;//4
 void (*__Block_byref_id_object_copy)(void*, void*);//8
 void (*__Block_byref_id_object_dispose)(void*);//8
 NSObject *__strong objc_block;
};

static void __Block_byref_id_object_copy_131(void *dst, void *src) {//8+8+4+4+8+8 = 40
 _Block_object_assign((char*)dst + 40, *(void * *) ((char*)src + 40), 131);
}

static void __Block_byref_id_object_dispose_131(void *src) {
 _Block_object_dispose(*(void * *) ((char*)src + 40), 131);
}
//变量声明
__attribute__((__blocks__(byref))) __Block_byref_objc_block_1 objc_block = {
            (void*)0,
            (__Block_byref_objc_block_1 *)&objc_block,
            33554432,
            sizeof(__Block_byref_objc_block_1),
            __Block_byref_id_object_copy_131,
            __Block_byref_id_object_dispose_131,
            ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"))};
```
从上面，可以看出__block修饰的对象类型生成的结构体中新增加了两个函数`void (*__Block_byref_id_object_copy)(void*, void*);`和
`void (*__Block_byref_id_object_dispose)(void*);`。
这两个函数为__block修饰的对象提供了内存管理的操作。赋值的分别为`__Block_byref_id_object_copy_131`和`__Block_byref_id_object_dispose_131`。

`__Block_byref_id_object_copy_131`函数中调用了`_Block_object_assign`函数，而`_Block_object_assign`函数内部拿到`dst指针`即block对象自己的地址值加上40个字节。并且`_Block_object_assign`最后传入的参数是131，同block直接对对象进行内存管理传入的参数3，8都不同。可以猜想`_Block_object_assign`内部根据传入的参数不同进行不同的操作的。

通过对上面`__Block_byref_objc_block_1`结构体占用空间计算发现`__Block_byref_objc_block_1`结构体占用的空间为48个字节。而加40恰好指向的就为`objc_block`指针。
![__block+strong](https://github.com/syxBravely/BlogImages/raw/master/iOS/__block%2BStrong.png)


接下来，我们看看`__weak`修饰对象的实现原理，在OC里变量的声明为
`Person *p = [Person new];__block __weak Person *weakP_block = p;`

```
struct __Block_byref_weakP_block_2 {
  void *__isa;
__Block_byref_weakP_block_2 *__forwarding;
 int __flags;
 int __size;
 void (*__Block_byref_id_object_copy)(void*, void*);
 void (*__Block_byref_id_object_dispose)(void*);
 Person *__weak weakP_block;
};
```
我们发现有变化的是`__Block_byref_weakP_block_2`结构体里对'weakP_block'的引用变成`__weak`。
![__block+__weak](https://github.com/syxBravely/BlogImages/raw/master/iOS/__block%2Bweak.png)

最后，我们来总结一下Block的内存管理机制：（ARC模式下）

1. 当捕获的变量被`__blcok`修饰时，变量包会被封装成一个`__Block_byref_`结构体并将其地址传入`__main_block_impl_0`，`__main_block_impl_0`会对`__Block_byref_`结构体产生强引用，并且`__main_block_desc_0`结构体里会加入`copy`和`dispose`两个函数对`__Block_byref_`进行内存管理。

2. 如果捕获的变量是对象类型，`__Block_byref_`结构体里会加入`__Block_byref_id_object_copy`和`__Block_byref_id_object_dispose`负责对我们实际捕获的变量进行内存管理。如果捕获的变量是对象类型，其被__weak 修饰，则`__Block_byref_`结构体会对其弱引用；其被strong 修饰，则`__Block_byref_`结构体会对其强引用。


3. 当捕获的变量是对象类型且没有被`__blcok`修饰时，会将其地址传入`__main_block_impl_0`，`__main_block_desc_0`结构体里会加入`copy`和`dispose`两个函数对其进行内存管理。

4. 如果捕获的变量被__weak 修饰则`__main_block_desc_0`结构体会对其弱引用，反之，强引用。


**当block从堆中移除的时候。会调用`dispose`函数，block块中去除对`__Block_byref_person_0 *person;`的引用，`__Block_byref_person_0`结构体中也会调用`dispose`操作去除对`Person *person;`的引用。以保证结构体和结构体内部的对象可以正常释放。**











