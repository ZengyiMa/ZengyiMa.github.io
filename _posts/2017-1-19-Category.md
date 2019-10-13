---
layout: post
title: "理解Category"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 


category 类似于设计模式中的装饰器模式，可以在不改变当前类的情况下，给当前类扩展功能。category 可以添加方法和属性，但是不能添加实例变量，而在 runtime 层面是怎么处理 category 的呢，我们一步步往下探究。

# category 数据结构
和类，方法等 runtime 对象一样，category 在底层一样是结构体，它的结构如下所示

```
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};
```

可以看出 category 有方法列表，类方法列表，协议列表，属性列表。这里可以看出，category的可以做的工作，添加实例方法、添加类方法、实现协议、添加属性，不能做的工作是添加实例变量，因为类对象在编译时候内存布局就已经确定，而 category 是工作在 runtime 时刻，这时候如果添加实例方法会破坏内存布局。

谈到内存模型，这里引用唐巧的一幅图片
![](http://upload-images.jianshu.io/upload_images/809311-c56811862bcc0d5b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好了回到正文，理解了 category 的结构我们回头看看 runtime 对 category 如何处理。这里分2部分介绍，首先从编译器的工作，到 runtime 的流程说开。

# 编译器的工作

在编译时刻编译器会把 OC 对象转换成 runtime 需要的结构体，category 也不例外。我们来做个测试，创建一个自定义类。


```
.h

@interface DLClass : NSObject
@end
@interface DLClass (DLCategory)<NSCopying>

@property (nonatomic, strong) DLClass *prop;
- (void)hello;
@end

.m

@implementation DLClass
@end

@implementation DLClass (DLCategory)
- (void)hello
{
}
@end

```

使用 `clang -rewrite-objc DLClass.m`使 OC 代码编译到 C++ 格式。我们会得到一个 cpp 文件。打开，搜索`_category_t`，在最后面我们看到这样的代码

```
static struct _category_t _OBJC_$_CATEGORY_DLClass_$_DLCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"DLClass",
	0, // &OBJC_CLASS_$_DLClass,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_DLClass_$_DLCategory,
	0,
	(const struct _protocol_list_t *)&_OBJC_CATEGORY_PROTOCOLS_$_DLClass_$_DLCategory,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_DLClass_$_DLCategory,
};
```
看到这里，我们会奇怪我们的方法、属性和协议去哪了呢，我们来搜索看看，搜索`_OBJC_$_CATEGORY_INSTANCE_METHODS_DLClass_$_DLCategory`可以看到这么一段。

```
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_DLClass_$_DLCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"hello", "v16@0:8", (void *)_I_DLClass_DLCategory_hello}}
};
```
这里就是我们的方法啦，可以看到`hello`已经在里面了，
相应的属性和协议也可以这么找，

```
//协议
static struct /*_protocol_list_t*/ {
	long protocol_count;  // Note, this is 32/64 bit
	struct _protocol_t *super_protocols[1];
} _OBJC_CATEGORY_PROTOCOLS_$_DLClass_$_DLCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	1,
	&_OBJC_PROTOCOL_NSCopying
};

//属性
static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_DLClass_$_DLCategory __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"prop","T@\"DLClass\",&,N"}}
};
```
可以看出我们遵守了 NSCopying 协议，和定义了 DLClass 的属性。
至此编译器的工作也就完成啦。接下去交给 runtime

# Runtime 的处理
在 dyld 启动了 APP 之后，会调用 runtime 的 `_objc_init` 来初始化 runtime。


```
#if !__OBJC2__
static __attribute__((constructor))
#endif
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();
        
    // Register for unmap first, in case some +load unmaps something
    _dyld_register_func_for_remove_image(&unmap_image);
    dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_2_images);
    dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}
```

首先进行一些初始化工作，如环境变量的初始化，锁的初始化。接下去看最后2行。绑定了2个回调方法，而 category 就在 map_2_images 里面中处理

```
const char *
map_2_images(enum dyld_image_states state, uint32_t infoCount,
             const struct dyld_image_info infoList[])
{
    recursive_mutex_locker_t lock(loadMethodLock);
    return map_images_nolock(state, infoCount, infoList);
}
```
`map_2_images`比较简单，只是调用了 `map_images_nolock` ，`map_images_nolock` 这个方法比较长， 首先从 dyld_image_info 也就是可执行文件中获取 Objective-C 结构体元信息，如类的个数，SEL的个数等，之后调用 `_read_images`来读取这些东西。

`_read_images`这个方法也是非常长，这里贴一个简化版的代码


```
void _read_images(header_info **hList, uint32_t hCount)
{
   
    // 1、初始化工作,判断 GC, 是否要 禁用TaggedPointers等操作
    // 2、获取类的统计，命名类
    // 3、读取所有的类
    // 4、建立hash建立类名和类的映射
    // 5、注册 SEL
    // 6、加载 protocol 
    // 7、实现 class 在runtime 生成类对象 使用 realizeClass 函数

    // 8、处理 category 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            //寻找 category的类
            Class cls = remapClass(cat->cls);
            
            //没有，就忽略
            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                //把 category 映射到相应的 class
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    //重新生成method
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  
                /* ||  cat->classProperties */) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }

    //9、打印log
}

```

这里我忽略了其他的代码，把 category 部分保留了。
这里说一下 runtime 如何处理 category 的流程

## 获取所有的 category 

使用 `_getObjc2CategoryList` 可以获取编译时生成的 category 的集合。

```
  category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
```

## 判断 category 的类对应的类是否存在

使用 `remapClass` 在类的获取 hash 表中是否有相应的类，如果没有直接不处理此 category

```
 category_t *cat = catlist[i];
            //寻找 category的类
            Class cls = remapClass(cat->cls);
            
            //没有，就忽略
            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

```

## 处理实例方法，协议，属性

判断 category 的实例方法，协议，属性是否有存在，如果有存在，则去绑定这个 category 到类中。绑定之后，重新生成类的信息，将方法列表合并，把协议列表合并，把属性合并，然后输出 log

```
 bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                //把 category 绑定相应的 class
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    //重新生成 class 的信息
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

```

`addUnattachedCategoryForClass`函数的功能是将未绑定的 category 映射到 class 上，

```
static void addUnattachedCategoryForClass(category_t *cat, Class cls, 
                                          header_info *catHeader)
{
    runtimeLock.assertWriting();

    // DO NOT use cat->cls! cls may be cat->cls->isa instead
    NXMapTable *cats = unattachedCategories();
    category_list *list;

    list = (category_list *)NXMapGet(cats, cls);
    if (!list) {
        list = (category_list *)
            calloc(sizeof(*list) + sizeof(list->list[0]), 1);
    } else {
        list = (category_list *)
            realloc(list, sizeof(*list) + sizeof(list->list[0]) * (list->count + 1));
    }
    list->list[list->count++] = (locstamped_category_t){cat, catHeader};
    NXMapInsert(cats, cls, list);
}
```
原理比较简单。在 map 中查找指定 class 的 category list，如果没找到，则创建一个。如果找到，扩容一个，之后把 category 加到这个 class 的category list 下面，之后插入 map

`remethodizeClass`函数则是用来重新生成类的实例方法列表，协议列表，和协议列表。


```
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertWriting();

    isMeta = cls->isMetaClass();
    
    //取出class 的category的list，然后绑定
    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        //绑定
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}
```

实现思路为获取 class 的 category list ,`addUnattachedCategoryForClass`添加的，之后调用 `attachCategories`来完成 method list 等的合并。

```
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    
    //添加所有的方法列表
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate
    //生成 category 的 method_list_t， property_list_t， protocol_list_t
    
     allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];
	
		//获取 category 的 method_list_t 
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
		//获取 category 的 property_list_t 

        property_list_t *proplist = entry.cat->propertiesForMeta(isMeta);
        if (proplist) {
            proplists[propcount++] = proplist;
        }
	
			//获取 category 的 protocol_list_t 

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

//合并列表
    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}

```

原理比较简单，首先获取 category 的方法列表，属性列表，协议列表，之后使用 attachLists 来合并列表。
我们瞥一眼 attachLists

```
   void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount * sizeof(array()->lists[0]));
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 0 lists -> 1 list
            list = addedLists[0];
        } 
        else {
            // 1 list -> many lists
            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList;
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
    }
```

添加列表的时候，先扩容一片容纳旧的和新的内存空间，使用 memmove 将旧的 list 放到列表后边，将新的放到前面，所以这里也解释了，category 方法会覆盖了类的方法，因为 category 的方法总是在前面，会最先找到， runtime 找到就不继续找了。

## 处理类方法和协议
处理手法和上一步是一样的。不过这里是针对类方法，也就是 class 的 meta class。所以调用 `addUnattachedCategoryForClass`和
 `remethodizeClass`使用的是 `cls->ISA()`，类的 isa 就是指向 meta class
 
```
 if (cat->classMethods  ||  cat->protocols  
                /* ||  cat->classProperties */) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }

```


# 总结
1. category 会在编译阶段编译成`category_t` 结构体。 
2. runtime 会将 class 和 category 做一个映射。
3. category 的方法总会在 class 的方法之前，所以会造成`覆盖`，其实方法是保留的。
4. category 可以处理实例方法，协议，属性，类方法，但是不能处理实例变量。