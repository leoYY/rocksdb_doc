#Rocksdb MemTable

分析Rocksdb的Memtable相比leveldb方面的优化

##相关类成员

#### class MemtableRep
由facebook重新定义的Memtable基类，提供实现Memtable内部存储结构的标准定义接口。并且对memtable的底层存储提出了几条标准，值得注意的一个是不能存储相同key的数据，对比leveldb中的skiplist实现，Insert接口有注释，不能出现相同Key的结果，由于本身leveldb有Seq保证多次插入相同的数据会变为不同的版本，从外部避开了这个问题。

#### class MemTableAllocator

相比原版本leveldb直接使用Arena，rocksdb封装了新的MemTableAllocator。该类接口基本涵盖了Arena的借口。

成员：

	Arena* arena_;
	WriteBuffer* write_buffer_;   // 仅用于记录实际需要写出的buffer大小，以及通过设置限制Buffersize来判断是否需要flush
	size_t bytes_allocated_;
	
#### class Memtable

相比leveldb的Memtable添加了很多其他成员

	...// 与leveldb相同无变化部分省略
	MemTableAllocator allocator_;
	unique_ptr<MemTableRep> table_;
	
	atomic<uint64_t> data_size_;
	atomic<uint64_t> num_entries_;
	uint64_t num_deletes_;
	
	bool flush_in_progress_;
	bool flush_completed_;
	uint64_t file_number_;
	
	VersionEdit edit_;
	
	SequenceNumber first_seqno_;
	
	SequenceNumber earliest_seqno_;
	
	uint64_t mem_next_logfile_number_;
	
	vector<port::RWMutex> locks_;
	
	const SliceTransform* const prefix_extractor_;
	unique_ptr<DynamicBloom> prefix_bloom_;
	
	bool should_flush_;
	
	bool flush_scheduled_;
	Env* env_;
	
###### 关注点 prefix_extractor_

prefix\_extractor\_ 优化主要应用在hash_skiplist_rep／hash_linklist_rep两种数据结构中，通过上层hash索引prefix_extractor作为hash索引可以保证在想通prefix下的桶内使用seek，降低seek复杂度。另外dynamic_bloom也有应用于prefix_extractor来加key。  

**hash\_skiplist\_rep**中getIterator会新建一个skiplist，返回普通iterator，因此对于hash\_skiplist\_rep并不适合跨range查询，效率会比较低，而对于查询指定前缀（或匹配一定规则）hash\_skiplist\_rep的优势是根据prefix进行hash分bucket，并在bucket中的skiplist进行遍历，减少了普通skiplist中访问的数据量。

hash\_skiplist\_rep有两个Iterator， DynamicIterator和Iterator，分别对应上面的支持分桶skiplist查询和重建全量skiplist查询的迭代器。

**hash\_linklist\_rep**中的情况类似，包含LinkListIterator，EmptyIterator，	DynamicIterator，FullListIterator。其中hash\_linklist\_rep中提供两个获取借口GetIterator，和GetDynamicIterator。其中GetIterator，会申请一个新的skiplist，并会根据bucket的具体为linklist／skiplist来进行遍历，返回fulllistIterator，GetDynamicIterator，如果bucket中的list为linklist则会编程线性查找。

###### 关注点 hashCuckooRep 数据结构

此处主要关注Cuckoo hashing的原理， 最早是从bloomfilter的替代品中了解到Cuckoo filter。

#### class DynamicBloom

dynamicbloom 应该是rocksdb对应bloom fliter类似

成员：

	uint32_t kTotalBits;	   // bit数
	uint32_t kNumBlocks;      // block nums
	cnost uint32_t kNumProbes;// hash次数
	
	uint32_t (*hash_func_)(const Slice& key);
	unsigned char* data_;		// 数据起始地址
	unsigned char* raw_;		// 实际申请内存起始地址（区别主要是在于内存对齐）
	
###### 关注点在于内存对齐以及cpu cache对齐方面的考虑

dynamicBloom 

###### \_\_builtn_prefetch
	
该方法支持通过对数据预取到cpu cache中，提升访问效率

#### class MemTableIterator 

rocksdb中的MemTableIterator把bloom filter机制从sstable中提前到了memtable中，一方面可以节省tableBuilder中重新遍历一遍数据构建bloomfilter block，而是随着memtable的构建，因此




