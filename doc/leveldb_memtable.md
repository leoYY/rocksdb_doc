#Leveldb模块之Memtable

##相关类介绍

####class Arena

类成员：
	
	char* alloc_ptr_;      				// 内存块可用地址指针
	size_t alloc_bytes_remaining_;	// 剩余可用内存大小
	
	std::vector<char*> blocks_;		// 记录所有申请过的block地址，包含已分配和未分配的	
	size_t blocks_memory_;           // 记录所有block的大小，包含已分配和未分配的

类方法：

	public:
		char* Allocate(size_t bytes);		
		char* AllocateAligned(size_t bytes);
		size_t MemoryUsage() const;
		
	private:
		char* AllocateFallback(size_t bytes);
		char* AllocateNewBlock(size_t block_bytes);
		
该类主要为内存池类，负责申请释放并统计内存使用

其中主要两个接口为Allocate／AllocateAligned， 通过alloc\_ptr\_和alloc\_bytes\_remaining_记录当前block的使用情况。  

######关注点1，AllocateFallback
当当前block剩余空间不足需要的bytes，触发AllocateFallback，**判断当前剩余空间是否超过1/4BlockSize，如果超过则直接分配新的block，大小为需求bytes，否则舍弃当前block的剩余空间，申请固定大小的新Block进行分配**  


######关注点2， AllocateAligned
该接口主要申请满足固定大小内存分配（低位地址不变），由于block本身为固定BlockSize的，因此为规整的内存分配，**首先会根据当前计算机指针大小来确定，并计算当前分配指针与预期的地址相差内存空间大小，通过bytes+slop的方式分配大小**
		
#### class SkipList
该类为模版类实现线程安全的skiplist  
成员：

	Comparator const compare_； // key的比较接口
	Arena* const arena_;		 // 内存分配器
	
	Node* const head_;			 // skiplist通过二维链表实现，表头
	
	port::AtomicPointer max_height_;  // 
	
	Random rnd_;					 // 随机函数
	
接口方法：

	public:
		void Insert(const Key& key);
		bool Contains(const Key& key);
	
	private:
		Node* NewNode(const Key& key, int height);
		
		int RandomHeight();
		
		bool Equal(const Key& a, const Key& b);
		
		bool KeyIsAfterNode(const Key& key, Node* n) const;
		
		Node* FindGreaterOrEqual(const Key& key, Node** prev) const;
		Node* FindLessThan(const Key& key) const;
		Node* FindLast() const;
		
		
######关注点1 Node的实现

结构体Node，成员主要是Key，而对于Next，setNext，通过Barrier的方式提供了相对线程安全的读取和修改操作，barrier接口类似提供底层的同步元语。此处借助SkipList高并发的优势，因为对于SkipList并发操作主要考虑对单个节点的锁，锁的粒度更小。  

之前有些错误理解，认为threadsafety就是完全线程安全的，结合注释会发现，skiplist实现依赖外部需要单线程顺序写的前提下，保证读的并发，且这里面一方面依赖memory barrier， 一方面因为skip为只增不删，保证一些多读的情况下，不需要加读锁保证读取的安全性。内存屏障可以保证在执行的顺序性，避免cpu乱序导致的异常读区问题。类似于保证了内存强一致性。保证了多线程读／单线程写模型下的一致性问题。

结合c++11种的atomic memory order flag，skiplist的node中主要应用了三种   

1. memory_order_acquire
2. memory_order_store
3. memory_order_relaxed

之所以出现memory order的问题，在于多核cpu下内存一致性模型的问题。acquire保证实时获取最新的store

结合skiplist代码来看，1个线程写，多个线程读的方式。对于写会引入max_height_和prev[level]->next的修改，而读会涉及到max_height_的读和node[level]->next的读。  

1. 无memory ordering 控制机制下，会出现max_height_更新后还未更新prev节点时，读线程读到max_height_，不过注释也说到通过NULL值可以降低level访问无影响，但是对于prev[level]->next的修改代码是.
	
		x = NewNode(key, height);
		for (int i = 0; i < height; ++i) {
			x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));				// (2)
			prev[i]->SetNext(i, x); // (3)
		}
	
	如果无内存屏障等顺序性问题，(2), (3)满足乱序优化，可以先执行(3),再执行（2），那么对于读进程会导致读到了x，单无next值，到支持出现非预期现象，SetNext使用release store／memory barrier，保证代码的顺序性。
2. 对于Next()的读取，如果不使用acquire load／memory barrier，可能会导致数据一致性的问题，其他线程已经setNext，但读线程并未读取到最新结果，memory barrier会保证在Next之前获取到其他cpu下修改操作的最新结果。

从这种角度来看acqure load／release store，保证了执行顺序一致性与数据同步的一致性。

#### MemtableIterator

该类主要依赖skiplist提供的FindLessThan等接口提供访问查询。

#### MemTable
除了以上类作为成员外，主要是方面层面

	public:
		void Add(SequenceNumber seq, ValueType type, const Slice& key, const Slice& value);
		
		bool Get(const LookupKey& key, std::string* value, Status* s);
		
######关注点 LookupKey

此处有三种key类型，MemtableKey， InternalKey， UserKey  


