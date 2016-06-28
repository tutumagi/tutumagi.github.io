---
title: Runtime之Associative References
date: 2016-06-28 15:11:45
tags:
- iOS
- Runtime
category:
- iOS
---

Runtime中跟`AssociateObject`相关的接口如下

```objc
void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy)

id objc_getAssociatedObject(id object, void *key)

void objc_removeAssociatedObjects(id object)
```

我们来看下[苹果开源中的实现](http://opensource.apple.com/tarballs/objc4/)  
我这里下载的是[objc4-680.tar.gz](http://opensource.apple.com/tarballs/objc4/objc4-680.tar.gz)  
下载源码后从`objc-runtime.mm`里面找到对应方法，会发现实现在文件`objc-references.mm`

先看一下相关的结构：  


```c++
// class AssociationsManager manages a lock / hash table singleton pair.
// Allocating an instance acquires the lock, and calling its assocations() method
// lazily allocates it.

class AssociationsManager {
    static spinlock_t _lock;
    static AssociationsHashMap *_map;               // associative references:  object pointer -> PtrPtrHashMap.
public:
    AssociationsManager()   { _lock.lock(); }
    ~AssociationsManager()  { _lock.unlock(); }
    //Lazy allocate，这里返回引用应该是为了避免返回指针时被不小心修改  
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};
```
这样我们就可以通过

```c++
    AssociationsManager manager;
    AssociationsHashMap &associations(manager.associations());
```
来获取对应的`HashMap`

`AssociationsHashMap`的结构

```c++
    //存储AssociationValue时，包括我们传入的policy和value
    class ObjcAssociation {
        uintptr_t _policy;
        id _value;
    public:
        ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
        ObjcAssociation() : _policy(0), _value(nil) {}

        uintptr_t policy() const { return _policy; }
        id value() const { return _value; }
        
        bool hasValue() { return _value != nil; }
    };
    
    //存储的是<AssociationKey, AssociationValue>
    typedef ObjcAllocator<std::pair<void * const, ObjcAssociation> > ObjectAssociationMapAllocator;
    class ObjectAssociationMap : public std::map<void *, ObjcAssociation, ObjectPointerLess, ObjectAssociationMapAllocator> {
    public:
        void *operator new(size_t n) { return ::malloc(n); }
        void operator delete(void *ptr) { ::free(ptr); }
    };
    typedef ObjcAllocator<std::pair<const disguised_ptr_t, ObjectAssociationMap*> > AssociationsHashMapAllocator;
    //主要关注模板的前两个模板实例化参数
    //可以知道该结构是 <disguised_ptr_t, ObjectAssociationMap>
    class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap *, DisguisedPointerHash, DisguisedPointerEqual, AssociationsHashMapAllocator> {
    public:
        void *operator new(size_t n) { return ::malloc(n); }
        void operator delete(void *ptr) { ::free(ptr); }
    };
```
通过上面我们可以看到我们绑定在OC对象的"属性"是通过一个全局的`HashMap<ObjKey, <AssociateObjKey, AssociateObjValue> >`来实现的，其实看到这个结构大概能猜到我们对绑定"属性"的get，set以及remove的操作是如何实现的了，不过我们还是看下，**特别是对于我们绑定到对象上的值的释放的细节**。

主要看下`objc_setAssociatedObject`，不是很复杂，对应写上注释了。

```c++
void objc_setAssociatedObject(id object, const void *key, id value, 
                         objc_AssociationPolicy policy) 
{
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    // 1. 这里根据传入的policy预先对value做一些操作：是否retain，是否copy
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        //2. 得到全局的HashMap
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        //3. 不直接存储指向该OC对象的指针，而是存储经过转换的指针值，目前的实现是对指针去翻
        disguised_ptr_t disguised_object = DISGUISE(object);
        //4.1 如果value不为空
        if (new_value) {
            // break any existing association.
            //5. 先查找HashMap中是否有对应的OC对象的AssociationMap
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            //5.1 如果有，而且对应的key有值，则临时保存旧值，并赋值新值
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
            } 
            //5.2 如果没有，这新建一个AssociationMap，并添加<key, value>
            else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        }
        //4.2 如果value为空，则在对应的AssociationsMap（如果有）中取出旧值，然后删除对应key的节点
        else {
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
    // 5. 如果旧值不为空，释放旧值
    if (old_association.hasValue()) ReleaseValue()(old_association);

}
```

`objc_getAssociatedObject`和`void objc_removeAssociatedObjects`比较简单，主要注意一下关于释放的地方

```c++
id objc_getAssociatedObject(id object, const void *key) 
{
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
    //如果有 GETTER_AUTORELEASE的flag，则给它发送一个autorelease消息
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        ((id(*)(id, SEL))objc_msgSend)(value, SEL_autorelease);
    }
    return value;
}

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

整体来看，实现不是很复杂，不过可以看到两个关于C++的"技巧"，我也不知道能不能称为技巧～  

1. 是获取HashMap时返回的是引用
2. 是在存储指针时不直接存储指针的值，而是存储变换后的值（变换操作必须可逆），在取的时候逆向运算一下  