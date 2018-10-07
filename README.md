# super

有2个类,`TYPerson`.及其子类`TYStudent`.

在子类`TYStudent`的`init`方法中打印如下信息,看结果是什么?

```objc
// TYStudent.m

- (instancetype)init {
    if (self = [super init]) {
        NSLog(@"[self class] = %@",[self class]);
        NSLog(@"[self superclass] = %@",[self superclass]);
        
        NSLog(@"[super class] = %@",[super class]);
        NSLog(@"[super superclass] = %@",[super superclass]);
    }
    return self;
}
```

- 其打印结果为:

```c
TYStudent
TYPerson

TYStudent
TYPerson
```

### 验证下打印结果.

为什么 `[super class]` 和 `[self class]` 的结果一样那?

这里为了搞清楚`super`到底干了什么事情,我们要重写下父类的方法,通过分析其底层`runtime`源码,看`super`到底做了什么.

通过将`TYStudent.m`文件编译为`C++`代码后,看其底层实现.编译语句:

```c
xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc TYStudent.m
```

得到`TYStudent.cpp` 文件,在其中找到 `- (void)run` 方法的实现,主要看`[super run]`做了什么.得到如下:

```c++
static void _I_TYStudent_run(TYStudent * self, SEL _cmd) {

    ((void (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("TYStudent"))}, sel_registerName("run"));

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_w8_wnywnfxn7zldh13vnt816cmm0000gn_T_TYStudent_0a61d4_mi_0,__func__);

}
```

其中 `[super run]` 对应如下代码:

```c++
((void (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("TYStudent"))}, sel_registerName("run"));

// 精简一下,大致如下:
// __rw_objc_super 是个结构体
struct __rw_objc_super arg = {
    self,
    class_getSuperclass(objc_getClass("TYStudent"))
};

// sel_registerName("run") == @selector(run)
objc_msgSendSuper(arg, @selector(run));
```

那么看下`__rw_objc_super`这个结构体

```c++
/// Specifies the superclass of an instance. 
struct objc_super {
    /// Specifies an instance of a class.
    // 消息接收者
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    // 旧的
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
```

对于我们来说,这个结构体就是下面这个:

```c++
struct objc_super {
    // 消息接收者
    __unsafe_unretained _Nonnull id receiver;
    // 消息接收者的父类
    __unsafe_unretained _Nonnull Class super_class;
};
```

接下来看下 `objc_msgSendSuper`这个方法做了什么事情:

```c++
/** 
 * Sends a message with a simple return value to the superclass of an instance of a class.
 * 
 * @param super A pointer to an \c objc_super data structure. Pass values identifying the
 *  context the message was sent to, including the instance of the class that is to receive the
 *  message and the superclass at which to start searching for the method implementation.
 * @param op A pointer of type SEL. Pass the selector of the method that will handle the message.
 * @param ...
 *   A variable argument list containing the arguments to the method.
 * 
 * @return The return value of the method identified by \e op.
 * 
 * @see objc_msgSend
 */
OBJC_EXPORT id _Nullable
objc_msgSendSuper(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
```

从`objc_super`这个结构体可以看出,调用`[super run]` 这个方法后,当前的消息接收者还是`self`,即`TYStudent`.

然后根据上面的解释 `the superclass at which to start searching for the method implementation`. 是从父类的方法列表里开始找这个方法(找不到就继续往上找)来调用.

那么回到最开始的`init`方法.

- `[self class]` 和`[super class]`

每个 oc 对象都可以调用`class`方法,说明这个`class`方法的实现是在`NSObject`里的.

`[self class]` 只不过是从当前对象的方法列表里开始找,如果找不到,通过`superclass`指针,去其父类的方法列表里找,还找不到,继续通过`superclass`指针,找到`NSObject`类.找到了,调用.

而`[super class]`,它不是从当前对象开始找,因为上面的结构体第2个成员是传进来的当前对象的父类,所以它直接从当前对象的父类开始查找,即`TYPerson`找,找不到,通过`superclass`指针,找到`NSObject`,调用.

**所以说,`[self class]` 和 `[super class]` 他们两个都是调用的 NSObject 的 class 方法.至于返回的都是 TYStudent, 看下面的说明:**

这就涉及到`Class`的底层结构.

```objc
- (Class)class {
   return object_getClass(self);
}
```

所以,调用 `class`方法,它的返回值取决于`self`是谁.即消息接收者是谁.

而`superclass`的底层实现:

```objc
- (Class)superclass {
   return class_getSuperclass(object_getClass(self));
}
```

还是取决于 self. 消息接收者是谁,这里的消息接收者是 `TYStudent`.所以返回其父类TYPerson.

