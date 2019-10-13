---
layout: post
title: "Associated Objects关联对象"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 


在之前文章说过。category 可以添加方法，可以添加协议实现，可以添加属性，但是却不能添加实例变量。那么如果在 category 需要添加实例变量的情况下怎么办呢？关联对象来帮你。

Associated Objects（关联对象）可以给类动态的添加实例变量。可以使我们增强类结构的灵活性，在 AFNetworking，SDWebimage 中得到广泛的应用。

# 使用
Associated Objects 使用简单，
`objc_setAssociatedObject`可以为一个对象设置一个关联对象。
`objc_getAssociatedObject`可以获取一个对象的关联对象。
`objc_removeAssociatedObjects`可以移除一个关联对象。

# 源码中的 Associated Objects
首先我们先看 `objc_setAssociatedObject`

## objc_setAssociatedObject
`objc_setAssociatedObject` 底层调用的是 `objc_setAssociatedObject_non_gc`，而 `objc_setAssociatedObject_non_gc`调用了`_object_set_associative_reference`

其实现代码如下

```
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}

```

这里碰到几个数据结构简单介绍一下
### AssociationsManager
一个管理对象。维护了一个带锁的`AssociationsHashMap`，构造函数加锁，析构函数解锁

### AssociationsHashMap
无序 hash 表，key 是对象的关联对象所在对象的地址。value 是 `ObjectAssociationMap`
### ObjectAssociationMap
也是一个无序 hash 表，key 是关联对象设置的 key， value 是 `ObjcAssociation`
### ObjcAssociation
关联对象的封装，里面维护了关联对象的 value 和 关联策略

### 关联对象的添加流程


首先使用`acquireValue`来判断关联策略

```
static id acquireValue(id value, uintptr_t policy) {
    switch (policy & 0xFF) {
    case OBJC_ASSOCIATION_SETTER_RETAIN:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
    case OBJC_ASSOCIATION_SETTER_COPY:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_copy);
    }
    return value;
}

```
如果是 OBJC_ASSOCIATION_SETTER_RETAIN 则 objc_msgSend 调用 retain ，如果是 OBJC_ASSOCIATION_SETTER_COPY 则调用 copy


```
 AssociationsManager manager;
 AssociationsHashMap &associations(manager.associations());
 disguised_ptr_t disguised_object = DISGUISE(object);
```
获取由 runtime 维护的 `AssociationsHashMap`


```
 AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }

```

如果设置新的 value 不是 nil，首先寻找被关联对象的 AssociationsHashMap 是否存在，如果存在，应该去更新这个 key 的值。如果不存在则 创建 ObjectAssociationMap ，并且建立 key value 映射。完成后将对象的 instancesHaveAssociatedObjects 设置为YES, 释放的时候使用。


```
 AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }

```

当设置的 value 是 nil，那么相当于移除这个关联对象。并且把 old_association 指向 被移除的对象。


```
  // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);

```

检测到有需要释放的对象，那么使用`ReleaseValue`来释放旧的关联对象。

## objc_getAssociatedObject
和 `objc_setAssociatedObject` 相同。`objc_getAssociatedObject` 最终会调用 `_object_get_associative_reference`来获取关联对象。


```
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) ((id(*)(id, SEL))objc_msgSend)(value, SEL_retain);
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        ((id(*)(id, SEL))objc_msgSend)(value, SEL_autorelease);
    }
    return value;
}
```

和建立关联对象相似，首先获取被关联对象的 AssociationsHashMap ，之后获取相应的 key 所映射的 value 根据不同的策略来调用 autorelase 或者 SEL_retain。

## objc_removeAssociatedObjects
`objc_removeAssociatedObjects`底层调用了`_object_remove_assocations`


```
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
```
实现原理比较简单，遍历擦除类中所有的 AssociationsHashMap 对象即可。

# 释放的时机
关联对象是在什么时候释放的呢？我们可以从 delloc 方法实现看出端倪。


```
// Replaced by NSZombies
- (void)dealloc {
    _objc_rootDealloc(self);
}
```
可以看出 dealloc 调用了 _objc_rootDealloc


```
void
_objc_rootDealloc(id obj)
{
    assert(obj);

    obj->rootDealloc();
}
```

继续调用了对象的 rootDealloc


```
inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;
    object_dispose((id)this);
}
```
rootDealloc 调用了 object_dispose


```
id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);
    
#if SUPPORT_GC
    if (UseGC) {
        auto_zone_retain(gc_zone, obj); // gc free expects rc==1
    }
#endif

    free(obj);

    return nil;
}
```
在调用的 objc_destructInstance 我们找到了最终的答案。

```
void *objc_destructInstance(id obj) 
{
    if (obj) {
        Class isa = obj->getIsa();

        if (isa->hasCxxDtor()) {
            object_cxxDestruct(obj);
        }

        if (isa->instancesHaveAssociatedObjects()) {
            _object_remove_assocations(obj);
        }

        if (!UseGC) objc_clear_deallocating(obj);
    }

    return obj;
}
```

调用 `_object_remove_assocations`来删除关联对象
