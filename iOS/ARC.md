### ARC

#### ARC基本实现

使用ARC，开发者不再需要手动的retain/release/autoreleae 编译器会自动插入对应的代码，再结合Objective-C的runtime，实现自动引用计数。

```objc
NSObject *obj;
{
	obj = [[NSObject alloc] init];
}
NSLog(@"%@",obj);
```

```objc
NSObject *obj;
{
	obj = [[NSObject alloc] init];
	[obj release];
}
NSLog(@"%@",obj);
```

其中，retain/release的方法实现原理：
```objc
- (id)retain {
	return ((id)self )->rootRetain();
}
inline id objc_object::rootRetain() {
	if(isTaggedPointer()) return (id) this;
	return sidetable_retain();
}

#if SUPPORT_MSB_TAGGED_POINTERS
#   def TAG_MASK (1ULL<<63>)
#else
#   define TAG_MASK 1

inline bool
objc_object::isTaggedPointer()
{
#if SUPPORT_MSB_TAGGED_POINTERS
	return ((uintptr_t)this & TAG_MASK);
#else
	return false;
#endif
}
```

由retain方法的实现可知，有些对象如果支持使用TaggedPointer，苹果会直接将其指针作为引用计数返回；如果当前设备是64位环境并且使用Objective-C 2.0，那么“一些”对象会使用其isa指针的一部分空间来存储它的引用计数；否则Runtime会使用一张散列表来管理引用计数。
```objc
id objc_object::sidetable_retain() {
	//获取table
	SideTable& table = SideTable*()[this];
	//加锁
	table.lock();
	//获取引用计数
	size_t& refcntStorage = table.refcnts[this];
	if( ! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
		//引用计数器增加
		refcnStorage += SIDE_TABLE_RC_ONE;
	}
	//解锁
	table.unlock();
	return (id)this;
}
```

与retain类似，release方法也是如此的逻辑，只是增加了弱引用计数位0，则销毁对象：
```objc
SideTable& table = SideTables()[this];
bool do_dealloc = false;
table.lock()
//找到对应地址
RefcountMap::iterator it = table.refcnts.find(this);
if(it == table.refcnts.end()) {
	//找不到的话，执行dealloc
	do_dealloc = true;
	table.refcnts[this] = SIDE_TABLEA_DEALLOCATING;
} else if (it->second < SIDE_TABLEA_DEALLOCATING) {//引用计数器小于阈值，dealloc
	do_dealloc = true;
	it->second |= SIDE_TABLEA_DEALLOCATING;
} else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
	//引用计数减1
	it->second -= SIDE_TABLE_RC+ONE;
}
table.unlock();
if (do_dealloc && performDealloc) {
	//执行dealloc
	((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
}
return do_dealloc;
```

再介绍下SideTable这个类，它用于管理引用计数器和weak表，并使用spinlock_lock自旋锁来繁殖操作表结构时有可能的竞态条件。它用一个64*128大小的uint8_t静态数组作为buffer来保存所有的SideTable实例。并提供三个公有属性。
```
spinlock_t slock;//保证原子操作的自旋锁
RefcountMap refcnts;//保存引用计数器的散列表
weak_table_t weak_table;//保存weak引用的全局散列表
```

weak表的作用是在对象执行dealloc的时候将所有指向该对象的weak指针值设为nil，避免悬空指针。这事weak表的结构：

```objc
struct weak_table_t {
	weak_entry_t *weak_entries;
	size_t    num+entries;
	uintptr_t mask;
	uintptr_t max_hash_displacement;
}
```

苹果使用一个全局weak表来保存所有的weak引用。并将对象作为键，weak_entery_t作为值。weak_entry_t中保存了所有指向该对象的weak指针。

#### autoreleasepool原理

Autorelease 对象是在当前的runloop迭代结束时释放的，而它讷讷够释放的原因是系统在每个runloop迭代中都加入了自动释放池的Push和Pop

Autorelease对象的各种方法都是在类AutoreleasePoolPage的封装，其中AutoreleasePoolPage类结构如下：

```objc
magic_t const magic;
id *next;
pthread_t const thread;
AutoreleasePoolPage *const parent;
AutoreleasePoolPage *child;
uint32_t const depth;
uint32_t hiwat;
```

- AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的结构组合而成（分别对应结构中的parent指针和child指针）

- AutoreleasePool 是按线程一一对应的（结构中的thread指针指向当前线程）

- AutoreleasePoolPage 每个对象会避开4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占的空间，剩下的空间全部用来储存autorelease对象的地址

- 上面的id *next指针作为游标指向栈顶最新add进来的autorelease对象的下一个位置

- 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，链接链表，后来的autorelase对象在新的page加入

所以，若当前线程中只有一个AutoreleasePoolPage对象，并记录了很多autorelease对象地址时呢诶村如下图：

图中的情况，这一页再加入一个autorelease对象就要满了，（也就是next指针马上指向栈顶），这时就要执行上面所说的操作，建立下一个page对象，这一页链表链接完成后，新page的next指针被初始化在栈底（begin的位置），然后继续向栈顶添加新对象。

所以，每一个对象发送- autorelease消息，就是讲这个对象加入到当前AutoreleasePoolPage的栈顶next指针指向

每当进行一次objc_autoreleasePoolPush调用时，runtime向当前的AutoreleasePoolPage中add进一个哨兵对象，值为0（也就是个nil），那么这一个page就变成了下面这样

objc_autoreleasePoolpush的返回值正事这个哨兵对象的地址，被objc_autoreleasePoolPop（哨兵对象）作为入参，于是：

- 根据传入的哨兵对象地址找到哨兵对象所处的page
- 在当前page中，将晚于哨兵对象插入的所有autorelease对象都发送一次 -relaese消息，并向回移动next指针到正确位置
- 从最新加入的对象一直向前清理，可以向前跨越若干个page，知道哨兵所在的page
