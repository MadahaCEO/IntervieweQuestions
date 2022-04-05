#Block


| 变量  | 是否捕获  | 访问方式 |
|:------------- |:---------------:|:-------------:|
| 局部auto变量 | 是 | 值传递 |
| 局部static变量 | 是 | 指针传递 |
| 全局变量 | 否 | 直接访问 |

为什么局部变量会被block捕获？而全局变量不会？


```
- (void)localTest {
    
    int num = 1;
    static int hands = 2;
    
    void(^TestBlock)(void) = ^{
        NSLog(@"2 num[%d]【%p】 hands[%d]【%p】",num,num,hands,hands);
    };
    
    num = 11;
    hands = 22;
    
    NSLog(@"1 num[%d]【%p】 hands[%d]【%p】",num,num,hands,hands);
    
    TestBlock();
}

！！！！！！num打印的内存地址不一样，而hands的内存地址是一样的！！！！！！

2022-04-05 10:59:18.779085+0800 CE[3188:95855] 1 num[11]【0xb】 hands[22]【0x16】
2022-04-05 10:59:18.779272+0800 CE[3188:95855] 2 num[1]【0x1】 hands[22]【0x16】

```

编译成c++文件

```
对应 - (void)localTest { }

static void _I_BlockTest_localTest(BlockTest * self, SEL _cmd) {

    int num = 1;
    static int hands = 2;

    void(*TestBlock)(void) = ((void (*)())&__BlockTest__localTest_block_impl_0((void *)__BlockTest__localTest_block_func_0, &__BlockTest__localTest_block_desc_0_DATA, num, &hands));

    num = 11;
    hands = 22;

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_7df74b_mi_1,num,hands);

    ((void (*)(__block_impl *))((__block_impl *)TestBlock)->FuncPtr)((__block_impl *)TestBlock);

}

------------------------------------------------------------------
Block的C++实现，是一个结构体.

struct __BlockTest__localTest_block_impl_0 {
  struct __block_impl impl;
  struct __BlockTest__localTest_block_desc_0* Desc;
  int num;
  int *hands;
  __BlockTest__localTest_block_impl_0(void *fp, struct __BlockTest__localTest_block_desc_0 *desc, int _num, int *_hands, int flags=0) : num(_num), hands(_hands) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

------------------------------------------------------------------
struct __block_impl {
  void *isa; // isa指针，指向一个类对象
  int Flags; // block 的负载信息（引用计数和类型信息）
  int Reserved; // 保留变量
  void *FuncPtr; // 一个指针，指向Block执行时调用的函数 (__blockTest_block_func_0)
};

------------------------------------------------------------------

static struct __BlockTest__localTest_block_desc_0 {
  size_t reserved; // Block版本升级所需的预留区空间
  size_t Block_size; // Block大小
} __BlockTest__localTest_block_desc_0_DATA = { 0, sizeof(struct __BlockTest__localTest_block_impl_0)};

__BlockTest__localTest_block_desc_0_DATA 是 __BlockTest__localTest_block_desc_0的是一个实例
------------------------------------------------------------------
Block的执行时调用的函数，这个函数就可以准确的看出局部变量的访问方式
【局部自动变量】自动保存在函数的每次执行的【栈帧】中，并随着函数结束后自动释放，另外，函数每次执行则保存在【栈】中。作用域是函数localTest内，那么有可能变量比block先销毁，这时候block再通过指针去访问变量就会有问题，所以做了拷贝操作，从打印可以看出是做了内存拷贝（内存地址是不一样的）
【局部静态变量】既非堆，也非栈，而是专门的【全局（静态）存储区static】，即并不会被销毁，所以仍然可以通过指针进行访问

static void __BlockTest__localTest_block_func_0(struct __BlockTest__localTest_block_impl_0 *__cself) {
  int num = __cself->num; // bound by copy
  int *hands = __cself->hands; // bound by copy

/**
从NSLog可以看出，打印的hands变量是 (*hands) 一个指针
再向上看看，重新定义了一个num变量来接收cself中的num
*/
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_7df74b_mi_0,num,(*hands));
       
    }

```


```
- (void)globalTest {
    age = 1;
    self.sex = @"f";

    void(^TestBlock)(void) = ^{
        NSLog(@"2 age[%d]【%p】 sex[%@]【%p】",age,age,self.sex,self.sex);
    };
        
    age = 11;
    self.sex = @"m";

    NSLog(@"1 age[%d]【%p】 sex[%@]【%p】",age,age,self.sex,self.sex);
    
    TestBlock();
}
打印的内存地址都是一样的
2022-04-05 11:26:03.895029+0800 CE[3342:109314] 1 age[11]【0xb】 sex[m]【0x1005351e0】
2022-04-05 11:26:03.895156+0800 CE[3342:109314] 2 age[11]【0xb】 sex[m]【0x1005351e0】
```
编译成c++文件

```
对应 - (void)globalTest { } , 注意参数中包含了 “ BlockTest * self ”

static void _I_BlockTest_globalTest(BlockTest * self, SEL _cmd) {

    (*(int *)((char *)self + OBJC_IVAR_$_BlockTest$age)) = 1;
    ((void (*)(id, SEL, NSString *))(void *)objc_msgSend)((id)self, sel_registerName("setSex:"), (NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_7df74b_mi_2);

    void(*TestBlock)(void) = ((void (*)())&__BlockTest__globalTest_block_impl_0((void *)__BlockTest__globalTest_block_func_0, &__BlockTest__globalTest_block_desc_0_DATA, self, 570425344));

    (*(int *)((char *)self + OBJC_IVAR_$_BlockTest$age)) = 11;
    ((void (*)(id, SEL, NSString *))(void *)objc_msgSend)((id)self, sel_registerName("setSex:"), (NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_7df74b_mi_4);

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_7df74b_mi_5,(*(int *)((char *)self + OBJC_IVAR_$_BlockTest$age)),((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("sex")));

    ((void (*)(__block_impl *))((__block_impl *)TestBlock)->FuncPtr)((__block_impl *)TestBlock);

}

------------------------------------------------------------------

static void __BlockTest__globalTest_block_func_0(struct __BlockTest__globalTest_block_impl_0 *__cself) {
  BlockTest *self = __cself->self; // bound by copy

/**
全局变量都在静态数据区，不会在当前函数执行完被销毁，
所以block直接访问了对应的变量，而没有在_block_impl_0结构体中给变量预留位置。
*/
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_7df74b_mi_3,(*(int *)((char *)self + OBJC_IVAR_$_BlockTest$age)),((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("sex")));
    }
static void __BlockTest__globalTest_block_copy_0(struct __BlockTest__globalTest_block_impl_0*dst, struct __BlockTest__globalTest_block_impl_0*src) {_Block_object_assign((void*)&dst->self, (void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __BlockTest__globalTest_block_dispose_0(struct __BlockTest__globalTest_block_impl_0*src) {_Block_object_dispose((void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}

------------------------------------------------------------------

static NSString * _I_BlockTest_sex(BlockTest * self, SEL _cmd) { return (*(NSString **)((char *)self + OBJC_IVAR_$_BlockTest$_sex)); }
extern "C" __declspec(dllimport) void objc_setProperty (id, SEL, long, id, bool, bool);

static void _I_BlockTest_setSex_(BlockTest * self, SEL _cmd, NSString *sex) { objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct BlockTest, _sex), (id)sex, 0, 1); }
// @end

```
Block内部修改局部变量

```
- (void)changeLocalTest {
   __block int apple = 1;
    void(^TestBlock)(void) = ^{
        apple++;  // 如果apple不添加 __block 修饰，报错信息Variable is not assignable (missing __block type specifier)
        NSLog(@"2 apple[%d]【%p】",apple,apple);
    };
    
    apple = 11;
    NSLog(@"1 apple[%d]【%p】",apple,apple);
    
    TestBlock();
}
```
编译成c++

```
static void _I_BlockTest_changeLocalTest(BlockTest * self, SEL _cmd) {

/**
apple 被封装成了对象，对象类型 __Block_byref_apple_0，此结构体中包含了isa的指针

它所捕获的由 __block 修饰的局部基本类型也会被拷贝到堆内（拷贝的是封装后的对象），从而会有 copy 和 dispose处理函数。__block修饰的基本类型会被包装为对象，并且只在最初block拷贝时复制一次
*/ 
    __attribute__((__blocks__(byref))) __Block_byref_apple_0 apple = {(void*)0,(__Block_byref_apple_0 *)&apple, 0, sizeof(__Block_byref_apple_0), 1};

    void(*TestBlock)(void) = ((void (*)())&__BlockTest__changeLocalTest_block_impl_0((void *)__BlockTest__changeLocalTest_block_func_0, &__BlockTest__changeLocalTest_block_desc_0_DATA, (__Block_byref_apple_0 *)&apple, 570425344));
/**
通过 _forwarding 访问 自身
*/
    (apple.__forwarding->apple) = 11;

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_3__4qk0t2014_d9vnrgd3jnnpg40000gn_T_BlockTest_06fc68_mi_7,(apple.__forwarding->apple),(apple.__forwarding->apple));

    ((void (*)(__block_impl *))((__block_impl *)TestBlock)->FuncPtr)((__block_impl *)TestBlock);
}

------------------------------------------------------------------
struct __Block_byref_apple_0 {
  void *__isa; // 指向所属类的指针
__Block_byref_apple_0 *__forwarding; // 指向自身的指针 或 指向对象在堆中的拷贝 【！！！重要！！！】
 int __flags; // 标志变量，在实现block的内部操作时会用到
 int __size; // 对象的内存大小
 int apple; // 原始类型的变量
};

------------------------------------------------------------------

static void __BlockTest__changeLocalTest_block_copy_0(struct __BlockTest__changeLocalTest_block_impl_0*dst, struct __BlockTest__changeLocalTest_block_impl_0*src) {_Block_object_assign((void*)&dst->apple, (void*)src->apple, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __BlockTest__changeLocalTest_block_dispose_0(struct __BlockTest__changeLocalTest_block_impl_0*src) {_Block_object_dispose((void*)src->apple, 8/*BLOCK_FIELD_IS_BYREF*/);}

```

## 循环引用

```
@interface Person : NSObject

@property (nonatomic, strong) NSString *name;
@property (nonatomic, copy) void (^block)(void);

- (void)testReferenceSelf;

@end

@implementation Person

- (void)testReferenceSelf {
    self.block = ^ {
        NSLog(@"self.name = %s", self.name.UTF8String);
    };
    self.block();
}

- (void)dealloc {
    NSLog(@"-------dealloc-------");
}
@end


int main(int argc, char * argv[]) {
    Person *person = [[Person alloc] init];
    person.name = @"roy";
    [person testReferenceSelf];
}

```

编译c++

```
struct __Person__testReferenceSelf_block_impl_0 {
  struct __block_impl impl;
  struct __Person__testReferenceSelf_block_desc_0* Desc;
  Person *const __strong self; // 【强引用了self】 
  __Person__testReferenceSelf_block_impl_0(void *fp, struct __Person__testReferenceSelf_block_desc_0 *desc, Person *const __strong _self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

static void _I_Person_testReferenceSelf(Person * self, SEL _cmd) {
  
    ((void (*)(id, SEL, void (*)()))(void *)objc_msgSend)((id)self, sel_registerName("setBlock:"), ((void (*)())&__Person__testReferenceSelf_block_impl_0((void *)__Person__testReferenceSelf_block_func_0, &__Person__testReferenceSelf_block_desc_0_DATA, self, 570425344)));
  对应 self.block(); 即self强引用了block，
    ((void (*(*)(id, SEL))())(void *)objc_msgSend)((id)self, sel_registerName("block"))();
}

这就可以看出 self.block(); 即self强引用了block， __Person__testReferenceSelf_block_impl_0 中 Person *const __strong self; Person 强拥有self
```
使用weak修饰
```
@implementation Person

- (void)testReferenceSelf {
    __weak typeof(self) weakself = self;
    self.block = ^ {
        __strong typeof(self) strongself = weakself;
        NSLog(@"self.name = %s", strongself.name.UTF8String);
    };
    self.block();
}

- (void)dealloc {
    NSLog(@"-------dealloc-------");
}

@end

```
编译c++
```
struct __Person__testReferenceSelf_block_impl_0 {
  struct __block_impl impl;
  struct __Person__testReferenceSelf_block_desc_0* Desc;
  Person *const __weak weakself; // 【重要】
  __Person__testReferenceSelf_block_impl_0(void *fp, struct __Person__testReferenceSelf_block_desc_0 *desc, Person *const __weak _weakself, int flags=0) : weakself(_weakself) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};


static void _I_Person_testReferenceSelf(Person * self, SEL _cmd) {
    __attribute__((objc_ownership(weak))) typeof(self) weakself = self;
    ((void (*)(id, SEL, void (*)()))(void *)objc_msgSend)((id)self, sel_registerName("setBlock:"), ((void (*)())&__Person__testReferenceSelf_block_impl_0((void *)__Person__testReferenceSelf_block_func_0, &__Person__testReferenceSelf_block_desc_0_DATA, weakself, 570425344)));
   
    ((void (*(*)(id, SEL))())(void *)objc_msgSend)((id)self, sel_registerName("block"))();
}

```




##内存

在ARC环境下，有哪些情况编译器会自动将栈上的把Block从栈上复制到堆上呢？

* Block从栈中复制到堆
* 调用Block的copy实例方法时
* Block作为函数返回值返回时
* 在带有usingBlock的Cocoa方法或者GCD的API中传递Block时候
* 将block赋给带有__strong修饰符的id类型或者Block类型时


1.当Block在栈上时，__block的存储域是栈，__block变量被栈上的Block持有。

2.当Block被复制到堆上时，会通过调用Block内部的copy函数，copy函数内部会调用_Block_object_assign函数。此时__block变量的存储域是堆，__block变量被堆上的Block持有。

3.当堆上的Block被释放，会调用Block内部的dispose，dispose函数内部会调用_Block_object_dispose，堆上的__block被释放。
