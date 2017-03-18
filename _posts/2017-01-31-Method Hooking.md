---
layout: post
title:  "Method Hooking"
date:   2017-01-31 12:00:00
categories: blog
---

So you're writing tweaks for iOS and wanted to understand more what is being done? What you're about to find is a concise, detailed explanation about what is going on under the hood when hooking into software during runtime. Previous experience with Objective C is required.

## The Objective C Runtime
Objective C is the primary language that Apple uses for application development in the userspace. It is an object oriented language and runtime written in C. It has been used for years ever since the inception of `NextStep`, later acquired by (you'd never figure this out) Apple, where the standard frameworks originated from. <br />

Although Swift is growing more popular, its runtime libraries are still not locally stored on the filesystem and end up having to be packaged into the app bundle (remaining dynamically linked) of an application that depends on code written in Swift. I'm not sure why Apple does this, my original assumption is that the ABI support is not fully complete. In addition, Swift has to rely on the frameworks and libraries that were written in Objective C. So least for now, Objective C remains strong. <br />

Anyway, the magical part about Objective C seems to be that it supports something called `reflection`, which is the ability for computer program to introspect, examine, and even modify the internal structures and behavior during runtime. Although one might feel this is a super large strength of Objective C, a person, given the security mitigations within iOS/macOS, can not take realistically take advantage of such language features effectively outside the scope of an application's address space without a security flaw in the OS. These vulnerabilities described are for another post. <br />

As far as reflection goes, let's take a look at the runtime headers of the Objective C runtime library, which is dynamically linked into a `Mach-O` binary everytime an app depends on Objective C code. In `objc/runtime.h` and `objc/objc.h`,

```c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class
    const char *name
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;
```

We find that an `objc_class` is a structure of many different fields that basically define the abstract concept of a class. From basic programming experience, a `Class` conceptually should have a `super class`, a way to identify the type of class (in this case it is by string), `instance variables`, `methods`, etc. `Protocols` may not be familiar to non-objective c developers, but they are just an added language feature that is well documented.

```c
#if !OBJC_TYPES_DEFINED
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
/// Represents an instance of a class.
```

If you happen to be confused as to what the predefined type `Class` was, it is typedef to a pointer to an `objc_class` type. This would be important since we don't have to copy data over and over if it happens to change during runtime (which would be the case since objc supports reflection) or when passing objects as parameters to methods/functions.

```c
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
/// A pointer to an instance of a class.
typedef struct objc_object *id;
#endif
/* Use `Class` instead of `struct objc_class *` */
```

And astoundingly, the only field of the `objc_object` structure is just a pointer to an instance of a `Class`. We also find that `id`, a widely used keyword, is also defined which is just typedef to a pointer to a `objc_object` (or `nil`). <br />

Objective C happens to use something called a metaclass system, which, in short, basically defines a mutual relationship between objects and classes. Classes are objects, and objects are instances of classes. This means that we can send messages to both a class and an object.

To better explain this, a good example would be the `NSString` class. <br />

`[NSString stringWithFormat:]` is a class method call that returns a pointer to an `NSString`. `NSString` is of type `Class`, and is an object too because a message can be passed to it (the selector `stringWithFormat:`). Conversely, the pointer to an NSString* object that is returned from that class method call at a lower level is an instance of a `Class`. This is the case because a pointer to an `id` (pointer to any Objective C object) has only one member, what is known as the `isa` pointer (of type `Class` or `objc_class*`).


```c
struct objc_category {
    char *category_name                                      OBJC2_UNAVAILABLE;
    char *class_name                                         OBJC2_UNAVAILABLE;
    struct objc_method_list *instance_methods                OBJC2_UNAVAILABLE;
    struct objc_method_list *class_methods                   OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;

struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;

struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}

/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;

/// A pointer to the function of a method implementation.
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ );
#else
typedef id (*IMP)(id, SEL, ...);
#endif
```

And here we find more definitions of `objc_category`, `objc_ivar`, `objc_methods`, `SEL` and `IMP` (implementation [basically a function pointer]). <br />

You might ask, "why is knowing this important?". Well firstly, this sets up your knowledge of the Objective C runtime. After understanding the basic types within the runtime, I can then explain how reflection works. <br />

So now you know the underlying data structures that make up the runtime. But how are they used?
The runtime includes many functions that create and modify these intrinsic structures. A list is shown below. <br />

```
Classes
class_getName
class_getSuperclass
class_isMetaClass
class_getInstanceSize
class_getInstanceVariable
class_getClassVariable
class_addIvar
class_copyIvarList
class_getIvarLayout
class_setIvarLayout
class_getWeakIvarLayout
class_setWeakIvarLayout
class_getProperty
class_copyPropertyList
class_addMethod
class_getInstanceMethod
class_getClassMethod
class_copyMethodList
class_replaceMethod
class_getMethodImplementation
class_getMethodImplementation_stret
class_respondsToSelector
class_addProtocol
class_addProperty
class_replaceProperty
class_conformsToProtocol
class_copyProtocolList

Adding Classes during Runtime
objc_allocateClassPair
objc_disposeClassPair
objc_registerClassPair
objc_duplicateClass

Creating Instances of Classes
class_createInstance
objc_constructInstance.
objc_destructInstance

Instances
object_copy
object_dispose
object_setInstanceVariable
object_getInstanceVariable
object_getIndexedIvars
object_getIvar
object_setIvar
object_getClassName
object_getClass
object_setClass

Classes (cont’d)
objc_getClassList
objc_copyClassList
objc_lookUpClass
objc_getClass.
objc_getRequiredClass
objc_getMetaClass

Message Sending
objc_msgSend
objc_msgSend_fpret
objc_msgSend_stret
objc_msgSendSuper
objc_msgSendSuper_stret

Methods
method_invoke
method_invoke_stret
method_getName
method_getImplementation
method_getTypeEncoding
method_copyReturnType
method_copyArgumentType
method_getReturnType
method_getNumberOfArguments
method_getArgumentType
method_getDescription
method_setImplementation
method_exchangeImplementations
```
With any of these functions, the runtime is able to perform the underlying behavior that the normal developer doesn't have to worry about in their process. There are more functions to the runtime not documented in this post, but these are the most relevant to reflection and method hooking. Now let's talk about some noteworthy functions that are important to us. <br />

`id objc_msgSend(id self, SEL op, ...)` and its counterparts send a message to an instance of a class. Method dispatching works a little differently in Objective C in comparison to other languages such as C++, Java and even Swift, which use virtual function tables to find the right method to jump to. The way that Objective C handles method dispatching is by using the meta-class structure (the `isa` pointer) to find the right selector and arguments. `objc_msgSend` starts where self points to (an id.. i.e. a pointer to an `objc_object`) and looks inside the isa pointer if it contains the right method in the method list. If it doesn't, it looks within the `superclass`, which is another type of `Class` in memory. If it goes through the whole class hierarchy and does not find a method that is dispatchable, the program aborts (usually an `EXC_CRASH SIGABRT` [unix signal abort]) and the operating system is forced to recover from it via an `exception`. When I discuss exceptions, traps, and interrupts in `XNU`, you'd able to know how that works in a much deeper level. But for now, let's stay on topic.

 The functions prefixed with class generally perform operations on classes or retrieves data from them. They are self explanatory and will be implemented in code samples in a future portion of this post, and this should clarify in detail about how they work. <br />

During runtime, you can even manually create classes and construct instances of them without the formal syntax that Objective C uses. Those are depicted by the sections "Adding Classes during Runtime" and "Creating Instances of Classes". <br />

`objc_getClass` returns the type of class that you would like during runtime. If the class you're querying is local to the process you're trying to use, then this syntax generally isn't needed. In special cases like writing tweaks and hooking into classes in a separate process via MobileSubstrate, you'll find a need for these functions. Below is an example of basic retrieval of `Classes` during runtime. 

```c
// retrieval of Classes via objc_getClassList
int numClasses;
Class *classes = NULL;
// first get the number of classes so we can dynamically allocate a pointer to them
numClasses = objc_getClassList(NULL, 0);

if (numClasses > 0 ){
    // allocate a pointer to all the Classes using the numClasses variable
    classes = (__unsafe_unretained Class *)malloc(sizeof(Class) * numClasses);
    // get them and store them into the classes pointer
    numClasses = objc_getClassList(classes, numClasses);
    for (int i = 0; i < numClasses; i++) {
        // print the name of the class via class_getName
        NSLog(@"Class name: %s", class_getName(classes[i]));
    }
    free(classes);
}
```
As shown in this simple example, this is one of the super flexible things about Objective C. The runtime is composed of fancy functions and data structures that happen to be super useful for developers. <br />
As far as the Objective C runtime goes, this is as in detail I'm going to go into for length purposes. Explaining every single crevice and crack of the runtime would be much too long for one post. Most of the intricacies that are outside the scope of this document are most likely on [Apple's developer portal]("http://developer.apple.com"). I managed to hit the high notes of what a person should know if they are interested in reverse engineering iOS. 


## Method Swizzling and Mobile Substrate
`MobileSubstrate` is a framework provided on jailbroken devices that allows developers to provide patches to system functions at the assembly level and to the internal structures of classes and method implementations in Objective C applications. It was written originally by saurik, a familiar member of the jailbreak community that created `Cydia`, the alternative AppStore which actually happens to be a frontend to `apt`, a package manager that ships on some Linux distributions. <br />

With regards to hooking methods and providing on the fly patches during runtime, `MobileSubstrate` takes advantage of something that the objective c runtime allows developers to perform, something called `method swizzling`. Method swizzling is a process of manipulating the implementation of an existing selector in a class. Based on the `objc_class` structure that we reviewed before, we can see that method swizzling basically modifies the dispatch table (the methodLists field). An example of method swizzling within a class is shown below. 

```c
@implementation UIViewController (Swizzling)

+(void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
    
        SEL originalSelector = @selector(viewDidLoad:);
        SEL swizzledSelector = @selector(swizzled_viewDidLoad:);
    
        Method originalMethod = class_getInstanceMethod(class,originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class,swizzledSelector);

        // This is for swizzling class methods
        // Class class = object_getClass((id)self);
        // Method originalMethod = class_getClassMethod(class,originalSelector);
        // Method swizzledMethod = class_getClassMethod(class,swizzledSelector);

        BOOL didAddMethod = class_addMethod(class,
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod,swizzledMethod);
        }
    });
}

-(void)swizzled_viewDidLoad {
    [self swizzled_viewDidLoad];
    NSLog(@"viewDidLoad %@", self);
}

@end
```

One thing you might notice in `swizzled_viewDidLoad` is that it seemingly makes a recursive call to itself. Actually, it doesn't (mind-blown). `+[UIViewController load]` is called when the class is loaded into the process' address space, which means that the swizzling happens before the class is even used. What actually happens during the swizzling process is that the `swizzled_viewDidLoad` becomes added as a new instance method in the class. Then, the implementations of `viewDidLoad` and `swizzled_viewDidLoad` get exchanged. This means that the `viewDidLoad` method contains the implementation of `swizzled_viewDidLoad`. So when we call `[self swizzled_viewDidLoad]`, what we're actually doing is calling the original implementation. This is the magic of the Objective C runtime. <br />

While this particular implementation works, let's take a step back and recall the objc_method structure. It contains `SEL method_name`, which points to the name of the selector and `IMP method_imp`, which is a pointer to the start of a method implementation. What you probably didn't realize is that Objective C `IMP`'s are actually just `functions pointers`. What we can do instead of internally in the class define the method is use the layout that Objective C uses for functions and perform method swizzling that way as well. Let's show an example.

```c
static IMP _original_viewDidLoad;

void _swizzled_viewDidLoad(id self, SEL _cmd){
    // calls the original implementation of viewDidLoad
    ((void(id,SEL))_original_viewDidLoad)(self, _cmd);
    NSLog(@"This is the swizzled implementation for -[UIView viewDidLoad]");
}

void swizzleFunctions(){
    // gets the original method to swizzle
    Method method = class_getInstanceMethod([UIViewController class],@selector(viewDidLoad));
    // sets the IMP method_imp field of objc_method to the swizzled implementation
    // this function returns the original implementation
    // we can use it to call the original if we need its behavior
    _original_viewDidLoad = method_setImplementation(method,(IMP)_swizzled_viewDidLoad);
}
```
MobileSubstrate does all the work for you to perform method swizzling during runtime. There are functions within the library that allow you to change the implementation of a class. They will be shown below after I explain how MobileSubstrate works from a bird's eye view. <br />

Firstly, what needs to be done is force a dynamic library to load into an application. The first component to MobileSubstrate is `MobileLoader`, which loads itself into an application using the `DYLD_INSERT_LIBRARIES` variable. The command line equivalent of doing so is `DYLD_INSERT_LIBRARIES=/PATH_TO/DYLIB.dylib /Applications/APP_NAME.app &;`. The dynamic loader for iOS and macOS, stored at `/usr/lib/dyld`, which happens to manage the loading and linking of dynamic libraries (also known as shared objects `.so` in Linux, and dynamically linked libraries `.dll` in Windows). When I explain the `Mach O` binary format I will explain this more in detail. But once `MobileLoader` is injected into the application, it looks into a directory named `/Library/MobileSubstrate/DynamicLibraries`, iterates through every dynamic library in the directory and checks if their respective plist, known as a filter, contains the same bundle/process name as the application. If they match, then `MobileLoader` performs a `dlopen` (dynamic library open) on the dynamic library matched. The entry points for the libraries should then perform the swizzling of implementations for Objective C classes or hooking of functions at the assembly level based on whatever the tweak developer decides. `MobileHooker` is responsible for this because the tweak developers utilize the following functions to achieve hooking.

```c
IMP MSHookMessage(Class class, SEL selector, IMP replacement, const char* prefix); // prefix should be NULL.
void MSHookMessageEx(Class class, SEL selector, IMP replacement, IMP *result);
void MSHookFunction(void* function, void* replacement, void** p_original);
```

`MSHookMessage()` replaces the `IMP` of the `SEL` passed to the functions, and return the original implementation. In order for a class method to be hooked, the `metaclass` should be used. It can be retrieved through the runtime function named `objc_getMetaClass`. <br />

`MSHookMessageEx()` does the same exact thing as `MSHookMessage()` but a placeholder to the pointer to the original `IMP` is passed instead of returned. <br />

`MSHookFunction()` works for C/C++ functions. This is performed at the assembly level. It writes instructions that `branch` to the replacement function. If the original function is called, the hooked function branches to a custom memory location where the instructions for the original function are stored as well as a branch back to the hooked function. <br />

